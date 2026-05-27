## Chapter 33: Monitoring and Forensics

### Sysmon Configuration

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

---

## Troubleshooting Guide

### Common Build Failures

| Symptom | Cause | Resolution |
|---|---|---|
| **AIB build times out** (120+ min) | Windows Update taking too long, or large cumulative update | Increase `build_timeout_minutes` to 150--180; exclude problematic KBs |
| **npm install fails with EACCES** | PATH not refreshed after Node.js install | Ensure `Update-SessionEnvironment` is called after MSI install |
| **`openclaw: command not found`** after install | npm global bin not in PATH | Run `Update-SessionEnvironment`; verify `C:\Program Files\nodejs` is in system PATH |
| **Sysprep fails** | Machine was domain-joined, or recovery partition exists | Verify source image is clean marketplace image; check for provisioning packages |
| **Image import fails in Intune** | Missing ACG feature flags | Verify all five features (SecurityType, Hibernate, DiskController, AccelNet, SecureBoot) |
| **Active Setup doesn't run** | Registry key malformed or version not bumped | Check `HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\OpenClaw-ConfigHydration`; verify `StubPath` and `Version` |
| **VS Code extensions missing** after login | Extensions installed into System profile during build | Install only Copilot during build; deliver others via post-provisioning Intune script |
| **OpenClaw gateway won't start** | Port conflict or missing config | Check if port 18789 is in use; verify `template-config.json` exists in ProgramData |
| **Terraform plan shows perpetual diff** | Using `timestamp()` instead of `time_static` | Use the `time_static` resource pattern |

### Diagnostic Commands

```powershell
# Verify all tools are in PATH
node --version; python --version; git --version; openclaw --version; claude --version

# Check Active Setup registry
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\OpenClaw-ConfigHydration"

# Check OpenClaw gateway status
Invoke-WebRequest -Uri "http://127.0.0.1:18789/health" -UseBasicParsing

# Verify SBOM exists
Get-ChildItem "C:\ProgramData\ImageBuild\sbom-*.json"

# Check image build log (during build)
az image builder show-runs --name "aib-w365-dev-ai-1-0-0" --resource-group "rg-w365-images" --output table
```

---

