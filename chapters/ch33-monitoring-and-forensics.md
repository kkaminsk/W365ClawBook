## Chapter 33: Monitoring and Forensics

### Sysmon Configuration

#### Deploying Sysmon to Cloud PCs

Sysmon (System Monitor) must be deployed before the event table below is actionable. Two deployment options:

- **Baked into the image**: Install during Phase 1 (Chapter 8) using `sysmon.exe /accepteula /i sysmonconfig.xml` in the PowerShell AIB customizer block. The config file must be present in ProgramData before the install command runs.
- **Intune Platform Script**: Deploy via an Intune Platform Script (System context) that downloads the Sysmon binary, validates its hash, and runs the install with config. This approach supports config updates without an image rebuild.

Use the [SwiftOnSecurity Sysmon config](https://github.com/SwiftOnSecurity/sysmon-config) or the [Olaf Hartong modular config](https://github.com/olafhartong/sysmon-modular) as a baseline. Both are maintained community configs optimized for detection coverage.

Deploy Sysmon to Cloud PCs with a configuration tuned for agent monitoring:

| Event ID | What to Monitor | Why |
|----------|-----------------|-----|
| **1** (Process Create) | `node.exe` spawning `cmd.exe` or `powershell.exe` with suspicious arguments (`-encodedCommand`, `DownloadString`) | Prompt injection -> shell execution |
| **3** (Network Connection) | Agent processes connecting to non-standard ports (not 80/443) or unexpected IPs | Lateral movement, C2 communication |
| **11** (File Create) | Creation of `.exe`, `.bat`, `.ps1` in agent working directory or Temp | Malware staging |
| **12/13** (Registry) | Modifications to Run keys, Scheduled Tasks | Persistence mechanisms |

### Sentinel KQL Hunting Queries

**Agent Spawning Shell:**
```kql
SecurityEvent
| where EventID == 4688
| where ParentProcessName has_any ("node.exe", "openclaw")
| where NewProcessName has_any ("cmd.exe", "powershell.exe")
| where CommandLine has_any ("-encodedCommand", "DownloadString", "Invoke-Expression")
| project TimeGenerated, Computer, Account, ParentProcessName, NewProcessName, CommandLine
```

> **Note:** `ParentProcessName` is the correct field name for the `SecurityEvent` table (populated by Windows Security Audit Event 4688 via MMA/AMA). If you are using Microsoft Defender for Endpoint advanced hunting (`DeviceProcessEvents` table), the equivalent field is `InitiatingProcessFileName`.

**Agent Accessing Credentials:**
```kql
SecurityEvent
| where EventID == 4663
| where ProcessName has_any ("node.exe", "openclaw")
| where ObjectName has_any (".ssh", "Credentials", "Vault", "Cookies")
| project TimeGenerated, Computer, Account, ProcessName, ObjectName
```

**Suspicious Network Egress:**
```kql
SysmonEvent
| where EventID == 3
| where Image has_any ("node.exe", "openclaw")
| where DestinationPort !in (80, 443)
| project TimeGenerated, Computer, User, Image, DestinationIp, DestinationPort
```

**Persistence Creation:**
```kql
SecurityEvent
| where EventID == 4698
| where SubjectUserName has "agent"
| project TimeGenerated, Computer, SubjectUserName, TaskName, TaskContent
```

### Windows Event Forwarding

Configure Cloud PCs to forward security events to Azure Log Analytics Workspace (Microsoft Sentinel) for centralized analysis. Key event channels:

- Security (4688 Process Creation, 4663 Object Access)
- Sysmon/Operational (all IDs above)
- PowerShell/Operational (Script Block Logging)

### Entra Agent ID Sign-In and Risk Signals

> **Note:** The APIs in this section target the Microsoft Graph **beta** endpoint and require Entra Agent ID to be in use. If your deployment uses secondary Entra ID users (Chapter 27's current recommendation), these queries are not yet applicable. Activate these as monitoring runbook additions when Agent ID reaches GA and your organization migrates agent accounts.

When agent identities are provisioned via Entra Agent ID (Chapter 27), Microsoft exposes two categories of agent-aware signals beyond the standard sign-in logs:

**Agent sign-in events** — Microsoft documents an `agentSignIn` event type and an `agent/agentType` filter on the Graph sign-in logs API. To query all sign-ins from agent identity objects:

```http
GET https://graph.microsoft.com/beta/auditLogs/signIns
    ?$filter=signInEventTypes/any(t: t eq 'servicePrincipal')
    and agent/agentType eq 'AgentIdentity'
```

The Entra admin center also exposes filters for **Agent ID User**, **Agent Identity**, **Agent Identity Blueprint**, and **Not Agentic** across all four sign-in log types. This is materially richer than filtering by account name — the `agent/agentType` filter is driven by object type, not naming convention, and survives account renames.

**Equivalent Sentinel KQL (working query using UPN naming convention):**

```kql
// Filters by the agent account naming convention established in Chapter 27
// (agent UPNs follow the pattern agent-<name>@<tenant>)
// Replace the prefix pattern to match your naming convention
SigninLogs
| where UserPrincipalName startswith "agent-"
| project TimeGenerated, UserPrincipalName, AppDisplayName, IPAddress, ResultType, ConditionalAccessStatus
| order by TimeGenerated desc
```

> **⚠️ Warning — Schema field not yet available:** Microsoft documents an `agent/agentType` filter on the Graph sign-in logs API and an `AgentType` field in `SigninLogs`. As of publication, **this field does not exist in the `SigninLogs` schema** in Azure Monitor / Microsoft Sentinel. A query using `| where AgentType == "AgentIdentity"` returns zero results and creates a dangerous false negative in a security monitoring context. The working query above uses UPN prefix filtering as a reliable substitute until Microsoft ships the schema update. Monitor [Microsoft Sentinel release notes](https://learn.microsoft.com/azure/sentinel/whats-new) for the `AgentType` field addition.

**Risky agent signals** — Microsoft Identity Protection exposes agent-specific risk surfaces in beta:

```http
GET https://graph.microsoft.com/beta/identityProtection/riskyAgents
GET https://graph.microsoft.com/beta/identityProtection/agentRiskDetections
```

`riskyAgents` returns agent identity objects that have elevated risk scores, analogous to `riskyUsers` for human accounts. `agentRiskDetections` returns the individual detection events that drove those scores. These endpoints enable agent-specific risk-based Conditional Access and incident response workflows that are not possible with secondary Entra ID user accounts.

**Sentinel alert for elevated agent risk:**

```kql
// Requires AADRiskyUsers or equivalent table populated from Identity Protection connector
// Replace with the appropriate table name once agent risk data is ingested
AADRiskyUsers
| where UserType == "AgentIdentity" and RiskLevel in ("high", "medium")
| project TimeGenerated, UserPrincipalName, RiskLevel, RiskDetail, RiskLastUpdatedDateTime
| order by RiskLevel desc
```

For production use, confirm the exact table name with your Sentinel connector configuration once the Agent ID Identity Protection data connector is available in your workspace.

### OpenClaw Gateway Health Monitoring

The OpenClaw gateway runs as a background Node.js process. Unlike a Windows service managed by SCM, it has no built-in restart-on-failure, no health endpoint monitored by the OS, and no event log integration. If the gateway crashes (due to an unhandled exception, memory exhaustion, or a bad MCP server interaction), it stays down until someone notices.

For enterprise deployments, this gap needs to be closed.

**Detection: Intune Remediation Script**

Deploy an Intune remediation (detection + remediation pair) that monitors the gateway process:

**Detection script** (`Detect-OpenClawGateway.ps1`):

```powershell
#Requires -Version 5.1
$gatewayProcess = Get-Process -Name "node" -ErrorAction SilentlyContinue |
    Where-Object { $_.CommandLine -match "openclaw" }

if ($gatewayProcess) {
    # Gateway is running -- check if it's responsive
    try {
        $response = Invoke-WebRequest -Uri "http://127.0.0.1:18789/health" `
            -TimeoutSec 5 -UseBasicParsing -ErrorAction Stop
        if ($response.StatusCode -eq 200) {
            Write-Output "Compliant"
            exit 0
        }
    } catch {
        # Process exists but not responding
        Write-Output "Non-Compliant: Gateway process exists but not responding"
        exit 1
    }
}

Write-Output "Non-Compliant: Gateway not running"
exit 1
```

**Remediation script** (`Remediate-OpenClawGateway.ps1`):

```powershell
#Requires -Version 5.1
$logDir = "C:\ProgramData\OpenclawRemediation"
$logPath = "$logDir\OpenClawGateway-Remediation.log"

New-Item -ItemType Directory -Path $logDir -Force | Out-Null
$timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"

# Kill any zombie gateway processes
$zombies = Get-Process -Name "node" -ErrorAction SilentlyContinue |
    Where-Object { $_.CommandLine -match "openclaw" }
if ($zombies) {
    $zombies | Stop-Process -Force
    Add-Content -Path $logPath -Value "$timestamp [REMEDIATION] Killed zombie gateway processes"
    Start-Sleep -Seconds 2
}

# Restart the gateway
try {
    Start-Process -FilePath "openclaw" -ArgumentList "gateway", "start" `
        -WindowStyle Hidden -PassThru | Out-Null
    Start-Sleep -Seconds 5

    # Verify it came up
    $response = Invoke-WebRequest -Uri "http://127.0.0.1:18789/health" `
        -TimeoutSec 10 -UseBasicParsing -ErrorAction Stop
    if ($response.StatusCode -eq 200) {
        Add-Content -Path $logPath -Value "$timestamp [REMEDIATION] Gateway restarted successfully"
        Write-Output "Remediation completed successfully"
        exit 0
    }
} catch {
    Add-Content -Path $logPath -Value "$timestamp [REMEDIATION] Failed to restart gateway: $_"
}

Write-Output "Remediation failed"
exit 1
```

Deploy this as an Intune remediation script targeting the Cloud PC device group. **Important:** Intune runs remediation scripts at a fixed interval controlled by the service (typically daily or at device check-in) — custom per-hour scheduling is not available for Intune remediations. For recovery times shorter than a day, use the Scheduled Task watchdog approach below instead of relying solely on Intune.

**Alerting: Sentinel Query**

Monitor for repeated gateway crashes across the fleet:

```kql
IntuneOperationalLogs
| where OperationName == "RemediationScriptExecution"
| where ScriptName == "Detect-OpenClawGateway.ps1"
| where ResultCode == 1  // Non-compliant
| summarize CrashCount = count() by DeviceName, bin(TimeGenerated, 1h)
| where CrashCount >= 3
| project TimeGenerated, DeviceName, CrashCount
```

Three or more non-compliant detections within an hour suggests a persistent issue (bad configuration, incompatible MCP server, memory leak) rather than a transient crash. This should trigger an alert for investigation.

**Proactive: Scheduled Task as a Watchdog**

For environments where Intune remediation frequency (minimum hourly) is too slow, deploy a Windows Scheduled Task as a lightweight watchdog during the image build:

```powershell
# In Phase 3 of the image build
$watchdogScript = @'
$gatewayRunning = Get-Process -Name "node" -ErrorAction SilentlyContinue |
    Where-Object { $_.CommandLine -match "openclaw" }
if (-not $gatewayRunning) {
    Start-Process -FilePath "openclaw" -ArgumentList "gateway", "start" -WindowStyle Hidden
}
'@

$watchdogPath = "C:\ProgramData\OpenClaw\watchdog-gateway.ps1"
Set-Content -Path $watchdogPath -Value $watchdogScript -Encoding UTF8

# Register scheduled task to run every 15 minutes
$action = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument "-ExecutionPolicy Bypass -WindowStyle Hidden -File `"$watchdogPath`""
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) `
    -RepetitionInterval (New-TimeSpan -Minutes 15) `
    -RepetitionDuration ([TimeSpan]::MaxValue)
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
    -StartWhenAvailable -RunOnlyIfNetworkAvailable
Register-ScheduledTask -TaskName "OpenClaw Gateway Watchdog" `
    -Action $action -Trigger $trigger -Settings $settings `
    -User "SYSTEM" -RunLevel Highest -Force
```

This provides sub-minute recovery from gateway crashes without depending on Intune remediation timing.

> **💡 Recommendation:** For production deployments, register the OpenClaw gateway as a **Windows service** using [nssm](https://nssm.cc/) (Non-Sucking Service Manager) or `node-windows`. A Windows service provides automatic restart on failure, event log integration, and SCM (Service Control Manager) lifecycle management, all of which the scheduled task watchdog must simulate. The separate [OpenClaw Gateway Watchdog](https://github.com/openclaw/gateway-watchdog) project implements exactly this pattern. The scheduled task approach above is acceptable for initial deployments and evaluation, but the Windows service model should be the target for any team relying on persistent gateway availability.

### Agent 365 Asset Context Mapping (June 2026)

Beginning June 2026, Microsoft Defender provides **asset context mapping** for each detected agent on the fleet — a graph of the devices the agent runs on, MCP servers it connects to, associated identities, and reachable cloud resources. This supplements the KQL queries in this chapter with a visualization layer:

- Use Agent 365's asset context map as the **baseline** when investigating a security alert — it shows the full blast radius of a potentially compromised agent without requiring manual KQL joins
- The KQL queries in this chapter remain essential for **hunting** and **automated alerting** — Agent 365's map is reactive; the KQL watchdog is proactive
- Cross-reference the asset context map against the agent account inventory from Chapter 27 to detect unauthorized identities associated with known agent processes

> **Note:** Microsoft's open-source **Agent Governance Toolkit** (released April 2026) provides structured audit logging for agent tool invocations, compatible with the Log Analytics schema queried in this chapter. Organizations using the toolkit can correlate toolkit-generated audit events with Defender process telemetry for a unified forensic trail.

### Purview Activity Explorer and DLP Forensics

Sysmon and Sentinel detect agent behavior at the process and network layer. Microsoft Purview adds a complementary audit trail at the **data layer** — specifically, which files were accessed, where they were attempted to be sent, and whether DLP policy blocked or allowed the egress.

When an agent account triggers a DLP block or a block-with-override event, the activity appears in **Purview Activity Explorer** tagged with the user, device, file, activity type, policy matched, and whether an override justification was provided. These events can be exported or streamed to Sentinel for correlation with process-level telemetry.

For incident reconstruction after a suspected agent compromise, **evidence collection** can capture the original matching file from the onboarded device to Azure Blob storage (local device cache of 7, 30, or 60 days while offline). This makes the actual file available for forensic review, not just the metadata about its attempted egress.

When the scope of a compromise is unknown, **Purview eDiscovery** can search content across Exchange, SharePoint, Teams, and OneDrive for files accessed or modified by the agent account during the compromise window. Combined with the DLP audit trail, this closes the "what data left?" question that process-level telemetry alone cannot answer.

For DLP audit integration into Sentinel, the `MicrosoftPurviewDLP` data connector (where available in your workspace) ingests DLP alert and policy match events into the `PurviewDLPAlert` table. Correlate on `DeviceName` and `ActorUserPrincipalName` to join DLP events with `SecurityEvent` and `SysmonEvent` records.

### From Threshold Alerts to Behavioral Intelligence

The Sysmon rules and KQL queries earlier in this chapter are **threshold-based**: they fire when a specific command, process relationship, or port appears. Threshold alerting is necessary, fast, and well-understood — but it has a structural limitation against AI agents: agents exhibit *goal-directed* behavior. A compromised agent pursuing a data exfiltration objective may never execute `mimikatz.exe` or open a non-standard port. It may instead read files slowly, summarize them, and quietly relay that summary through the Anthropic API on port 443 — traffic that looks identical to a normal development session.

#### Behavioral Baseline Learning

An operationally mature agent monitoring posture layers **automated behavioral baseline learning** on top of threshold alerting. Rather than defining manually what anomalous looks like, baseline learning observes normal operations and flags deviations from the learned pattern. For agent environments, the following behavioral dimensions are candidates for baseline modeling:

- **API call volume and timing:** A developer agent that normally makes 200–400 Anthropic API calls per day spiking to 4,000 calls over a two-hour window is behaviorally anomalous regardless of what any individual call contains.
- **File access breadth:** An agent scoped to a specific repository that begins reading files across unrelated directories in the user profile is deviating from its normal operational footprint.
- **Context window size distribution:** Unusually large context windows may indicate the agent is being fed large document batches — consistent with RAG poisoning or bulk exfiltration staging.
- **Output destination patterns:** An agent that begins writing to new directories, network shares, or temporary paths outside its normal write scope warrants investigation.

Microsoft Defender for Endpoint's behavioral analytics (EDR mode) provides a starting point for process-level behavioral profiling that can be tuned for agent workloads. For teams using Microsoft Sentinel, the **User and Entity Behavior Analytics (UEBA)** feature applies machine-learning anomaly detection to entities including device accounts and service principals — applicable to agent secondary accounts (Chapter 27) once those accounts accumulate a behavioral baseline over time (typically 7–14 days of observation).

Advanced implementations use **continuous ML-based behavioral refinement with drift detection** — the model's understanding of "normal" updates as agent workflows evolve, while flagging baseline shifts that occur abruptly or outside expected change windows. This is the direction the field is moving, though it requires dedicated SIEM tuning investment beyond what most teams apply on initial deployment. The threshold alerting in this chapter is the foundation; behavioral baseline is the next tier.

#### Full Provenance Chains: Reconstructing Agent Reasoning Traces

Standard audit logging records *what happened*: a file was written, a process was spawned, a network connection was made. In agent environments, forensic investigators frequently need to know *why it happened* — which prompt triggered the decision, what tools were called in sequence, and how the output of one step became the input for the next. This requires provenance chains, not just event logs.

**Full provenance chains** connect input → intermediate reasoning steps → tool calls → outputs into a single queryable trace. For Claude Code, this means correlating:

1. The user prompt that initiated the session (or the injected content if indirect injection occurred)
2. The sequence of tool calls the model made (shell commands, file reads, MCP server invocations)
3. The outputs of each tool call and how they appear in the next model turn
4. The final action taken and the model's stated rationale

Chapter 28 covers enabling OpenTelemetry export from Claude Code. When OTel tracing is configured with distributed trace context propagation — assigning a single `trace-id` to the full agent session and linking each tool invocation as a child span — the full provenance chain becomes reconstructable from the trace backend (Azure Monitor Application Insights, Jaeger, or compatible OTEL collector).

For incident response, a provenance chain transforms the forensic question from "what did the agent do?" (answerable from Sysmon + Sentinel) to "why did the agent do it, and what triggered the chain?" — the question that determines whether an incident was a misuse, a misconfiguration, or a successful attack. Prioritize provenance chain logging for agents with write access to production systems, CI/CD pipelines, or external APIs. The investment in trace infrastructure pays back disproportionately in the first major incident investigation.

---

AIB and pipeline build failure troubleshooting is in Chapter 34 (Troubleshooting Reference).

---

