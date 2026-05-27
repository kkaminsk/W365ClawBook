## Chapter 31: Endpoint Protection

### Attack Surface Reduction (ASR) Rules

Deploy via Intune Endpoint Protection profiles:

| ASR Rule | GUID | Purpose |
|----------|------|---------|
| Block process creation from PSExec/WMI | `d1e49aac-8f56-4280-b9ba-993a6d77406c` | Break exploit kill chains |
| Block untrusted/unsigned executables | `01443614-cd74-433a-b99e-2ecdc07bfc25` | Prevent malware execution |
| Block JS/VBS launching downloaded content | `d3e037e1-3eb8-44c8-a917-57927947596d` | Prevent script-based attacks |

### Application Control: WDAC and AppLocker

Application control is the **primary preventive layer** against agent runtimes on Cloud PCs. The goal is to stop execution before it reaches the network — blocking `node.exe` at the firewall is useful; stopping it from running at all is stronger.

Microsoft's current guidance is to prefer **App Control for Business (WDAC)** over AppLocker. WDAC is more actively developed, enforces at the kernel level, and supports signer-based deny rules that survive binary updates. Use AppLocker only to complement WDAC where user- or group-specific rules are needed, or in environments that cannot yet support a full WDAC rollout.

#### What to Deny

Microsoft publishes a recommended bypass block list that covers Windows binaries commonly abused by attackers and agent bootstrap chains: `mshta.exe`, `wscript.exe`, `cscript.exe`, `wmic.exe`, `msbuild.exe`, `dotnet.exe`, and WSL-related binaries. On **standard productivity Cloud PCs** (not developer or admin), additionally deny `node.exe` and package-manager binaries, since they are the shortest path from "a user installed an agent runtime" to "the runtime can reach the internet and persist."

| Binary | Why deny on standard Cloud PCs |
|---|---|
| `node.exe` | OpenClaw's canonical runtime; required to install, update, and run the agent |
| `mshta.exe` | Script host for HTA files; frequently abused for initial access and agent bootstrap |
| `wscript.exe` / `cscript.exe` | Windows Script Host; common in skill-delivered malware |
| `msbuild.exe` | Trusted builder abused to run arbitrary C# inline tasks |
| `dotnet.exe` | Can execute arbitrary assemblies; alternate runtime if Node is blocked |
| `wmic.exe` | WMI command-line tool; often used for reconnaissance and lateral movement |
| WSL binaries | WSL2 provides a full Linux environment that bypasses Windows-level controls |

#### WDAC Policy Design

A practical WDAC design for Windows 365:

1. Start from a Microsoft example base policy.
2. Add a managed installer trust path for Intune-delivered software if you rely on Intune Win32 app deployment.
3. Add explicit deny rules for Microsoft's recommended bypass tools.
4. Add explicit deny rules for `node.exe` and package-manager binaries on non-developer Cloud PC groups.
5. Pilot in **Audit** mode first; move to **Enforce** only after validating no legitimate workflows are blocked.

```powershell
$denyRules = @()

# Third-party runtime: Node.js — deny by publisher, fall back to hash
$denyRules += New-CIPolicyRule -Level FilePublisher `
  -DriverFilePath "C:\Program Files\nodejs\node.exe" `
  -Fallback SignedVersion,Publisher,Hash -Deny

# Microsoft LOLBins commonly abused for agent bootstrap or post-exploit use
foreach ($bin in @("mshta.exe","wscript.exe","cscript.exe")) {
    $denyRules += New-CIPolicyRule -Level FileName `
      -DriverFilePath "$env:WINDIR\System32\$bin" `
      -Fallback Hash -Deny
}

# PowerShell (deny outright on standard Cloud PCs; see note below for developer PCs)
$denyRules += New-CIPolicyRule -Level FileName `
  -DriverFilePath "$env:WINDIR\System32\WindowsPowerShell\v1.0\powershell.exe" `
  -Fallback Hash -Deny
```

> **⚠️ PowerShell nuance:** Denied PowerShell scripts that bypass WDAC run in **Constrained Language Mode** rather than being stopped absolutely. CLM prevents `.NET` API calls and limits what scripts can do, but it is not a hard block. If PowerShell must be allowed (developer and admin Cloud PCs), combine CLM, outbound firewall rules, and EDR monitoring rather than treating WDAC alone as sufficient.

#### AppLocker for Organizations Not Yet on WDAC

If WDAC is not yet feasible, AppLocker provides a meaningful control layer for Entra-joined Cloud PCs managed with Intune or GPO. Key rule shapes:

- **Executable rules:** Publisher deny for signed runtimes like Node; file-hash deny for unsigned agent wrappers.
- **Script rules:** Deny execution from user-writable folders and the Downloads directory.
- **MSI/script collections:** Control installers and bootstrap scripts explicitly.
- **Exceptions:** Per-role or per-group exceptions for developer or admin Cloud PCs.

> **AppLocker path-rule caveat:** AppLocker's default rules are path-based. If Node is installed in `Program Files`, the default allow rule may permit it. Explicitly add a publisher or hash deny rule for Node rather than relying on the absence of an allow rule.

### Windows Defender Firewall: Host-Level Process Blocking

Azure NSG rules operate at the subnet IP/port level — they cannot distinguish which process on the Cloud PC is making an outbound connection. Windows Defender Firewall can, because it supports application rules scoped to a **full executable path** or an **App Control PolicyAppId tag**. This makes it a critical complement to the NSG: even if HTTPS to a broad destination is allowed at the subnet level, a specific binary like `node.exe` can be denied egress at the host.

The recommended design is: **keep the Windows Firewall default outbound action at Allow**, then add explicit outbound Block rules for high-risk binaries. This is operationally safer than outbound-default-block for Windows 365 because the platform requires outbound connectivity to many Microsoft services (see Chapter 30) and an improperly configured outbound-block policy will disrupt provisioning, Intune enrollment, and RDP connectivity.

#### Deploy via Intune Firewall Rules Policy

```powershell
$blocked = @(
    @{ Name = "CloudPC Block Node";       Path = "C:\Program Files\nodejs\node.exe" },
    @{ Name = "CloudPC Block PowerShell"; Path = "$env:WINDIR\System32\WindowsPowerShell\v1.0\powershell.exe" },
    @{ Name = "CloudPC Block Pwsh";       Path = "C:\Program Files\PowerShell\7\pwsh.exe" },
    @{ Name = "CloudPC Block WScript";    Path = "$env:WINDIR\System32\wscript.exe" },
    @{ Name = "CloudPC Block CScript";    Path = "$env:WINDIR\System32\cscript.exe" },
    @{ Name = "CloudPC Block MSHTA";      Path = "$env:WINDIR\System32\mshta.exe" },
    @{ Name = "CloudPC Block Rundll32";   Path = "$env:WINDIR\System32\rundll32.exe" },
    @{ Name = "CloudPC Block MSBuild";    Path = "$env:WINDIR\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe" },
    @{ Name = "CloudPC Block DotNet";     Path = "C:\Program Files\dotnet\dotnet.exe" }
)

foreach ($app in $blocked) {
    New-NetFirewallRule `
      -DisplayName $app.Name `
      -Direction Outbound `
      -Program $app.Path `
      -Action Block `
      -Profile Domain,Private,Public
}
```

Deploy this through Intune **Endpoint security → Firewall rules**, not as local administrator scripts. Centralized deployment prevents a local admin or attacker from quietly adding an allow rule that overrides the block.

#### Disable Local Policy Merge

Windows Firewall's local policy merge setting controls whether a local administrator can add rules that override Intune-deployed policy. In high-security environments, disable merge to prevent quiet exception-adding:

```text
Intune: Endpoint security → Firewall
  "Disable local address processing for inbound rules" → Yes
  "Disable local address processing for outbound rules" → Yes
  "Apply local connection security rules" → No
```

#### PolicyAppId Tagging (Preferred for Robustness)

Path-based firewall rules are brittle: renaming or relocating a binary bypasses them. The more robust approach is to use App Control PolicyAppId tags and scope firewall rules to the tag rather than the path. If you already have a WDAC policy tagging agent runtime processes, add the network control:

```powershell
New-NetFirewallRule `
  -DisplayName "CloudPC Block Agent Runtime Tagged Processes" `
  -Direction Outbound `
  -PolicyAppId "BHG.CloudPC.AgentRuntime" `
  -Action Block `
  -Profile Domain,Private,Public
```

#### Intune Profile Set

| Intune policy | Recommended configuration | Why |
|---|---|---|
| **Endpoint security → Firewall** | Enable on Domain/Private/Public; default inbound **Block**; default outbound **Allow**; disable inbound notifications; enable dropped and successful connection logging; disable local policy merge | Aligns with Windows 365 baseline; preserves platform compatibility; prevents quiet local exceptions |
| **Endpoint security → Firewall rules** | Outbound **Block** for `node.exe`, `powershell.exe`, `pwsh.exe`, `wscript.exe`, `cscript.exe`, `mshta.exe`, `rundll32.exe`, `msbuild.exe`, `dotnet.exe` | Targeted containment without the blast radius of default outbound block |
| **Endpoint security → App Control for Business** | Base allow + managed installer + deny rules for LOLBins and unapproved runtimes | Stops execution, not just networking; integrates with Intune-delivered software |
| **Compliance + Conditional Access** | Require Defender for Endpoint threat level ≤ Low; apply CA to Azure Virtual Desktop / Windows Cloud Login app set | Turns endpoint detections into access revocation for Cloud PCs |

### Inbound Port Hardening

Windows 365's cloud-native RDP model is **outbound-only by design**. The platform uses outbound TCP 443 and UDP 3478 for the RDP session, not inbound TCP 3389. This means you can be aggressive about closing inbound listeners — any new inbound port on a Cloud PC is a deviation from the intended platform model and should be investigated.

| Port | Protocol | Status on standard Cloud PCs | Notes |
|---|---|---|---|
| **3389** | TCP | **Close; keep closed** | Legacy RDP; Microsoft says it is disabled by default on new Cloud PCs and recommends keeping it closed |
| **3390** | UDP | Close unless using RDP Shortpath | Required only for RDP Shortpath on ANC deployments with direct line-of-sight; not applicable to Microsoft-hosted networking |
| **5985 / 5986** | TCP | Close unless explicitly required | WinRM / PowerShell remoting; unexpected listeners indicate drift |
| **22** | TCP | Close unless explicitly required | OpenSSH server; presence on a standard knowledge-worker Cloud PC should trigger investigation |
| **445** | TCP | Close; block inbound from all sources | SMB; Microsoft recommends blocking inbound SMB from the internet |

Confirm inbound posture using the Windows 365 security baseline in Intune, which sets default inbound action to **Block** on all profiles. Any rule opening an inbound port should be explicitly documented and deployed centrally — not added by local administrators.

### Segment by Cloud PC Role

The controls above are appropriate for standard productivity Cloud PCs. Applying them uniformly to developer and admin Cloud PCs will cause disruption — blocking `node.exe`, `dotnet.exe`, or outbound PowerShell on a developer Cloud PC is immediately breaking, not just restrictive.

Create at least three Intune policy rings:

| Ring | Cloud PC type | Control posture |
|---|---|---|
| **Standard** | Knowledge-worker, office productivity | Most restrictive: Node and other runtimes denied; LOLBins denied; outbound scripting blocked |
| **Admin** | IT operations, security analysts | PowerShell allowed with strict monitoring; no unsanctioned runtimes; enhanced EDR alerting |
| **Developer** | Software engineers, build engineers | Controlled exceptions for approved runtimes; egress restricted to approved registries and repositories; stronger monitoring rather than blanket denies |

Target each ring with a separate Intune device configuration profile and Conditional Access assignment. Developer and admin Cloud PCs should have a higher Defender for Endpoint alert threshold configured to compensate for the wider execution surface.

### Detection: KQL Hunting for Suspicious Interpreter Egress

Even with application control and firewall rules in place, renamed binaries, alternate runtimes, and policy gaps can allow an agent process to reach external destinations. Hunt proactively in Microsoft Defender XDR:

```kusto
let SuspiciousBins = dynamic([
  "node.exe","powershell.exe","pwsh.exe","wscript.exe","cscript.exe",
  "mshta.exe","rundll32.exe","dotnet.exe","msbuild.exe","wmic.exe"
]);
DeviceNetworkEvents
| where Timestamp > ago(7d)
| where InitiatingProcessFileName in~ (SuspiciousBins)
| project Timestamp, DeviceName, InitiatingProcessFileName,
          InitiatingProcessCommandLine, RemoteUrl, RemoteIP, RemotePort
| order by Timestamp desc
```

Also monitor for local firewall rule creation, which can indicate an attacker or misconfigured skill attempting to add an egress allow exception:

```kusto
DeviceRegistryEvents
| where Timestamp > ago(1d)
| where RegistryKey has "SYSTEM\\CurrentControlSet\\Services\\SharedAccess\\Parameters\\FirewallPolicy"
| where ActionType in ("RegistryValueSet", "RegistryKeyCreated")
| project Timestamp, DeviceName, InitiatingProcessFileName, RegistryKey, RegistryValueName
```

Log sources that feed these detections:

| Log source | Event IDs to collect | What they catch |
|---|---|---|
| AppLocker | 8004, 8007 (blocked EXE/DLL, script/MSI); 8003, 8006 (audit-mode would-block) | Execution attempts against deny rules |
| App Control for Business | 3076, 3077 (audit/enforce blocks); 8028, 8029 (script-host checks) | WDAC policy hits |
| Windows Firewall | `pfirewall.log` dropped and successful connections | Network-layer enforcement events |
| Defender for Endpoint | `DeviceProcessEvents`, `DeviceNetworkEvents` | Process launches and network connections across all Cloud PCs |

### Windows Sandbox

For high-risk analysis tasks, configure Cloud PCs to support Windows Sandbox:

```text
OMA-URI: ./Device/Vendor/MSFT/Policy/Config/WindowsSandbox/AllowAudioInput -> 0
OMA-URI: ./Device/Vendor/MSFT/Policy/Config/WindowsSandbox/AllowNetworking -> 0
```

This allows agents to spin up disposable, isolated environments without network access for analyzing untrusted code.

### Disable WebClient Service

As discussed in Chapter 28, disable the WebClient service to neutralize the WebDAV/NTLM vulnerability:

```powershell
Set-Service -Name WebClient -StartupType Disabled -Status Stopped
```

### Data-Layer Controls

The controls in this chapter prevent malicious execution and lateral movement, but they cannot stop sensitive data that leaves through channels a user or agent is legitimately permitted to use — copy-to-USB, browser upload, print, or clipboard. Microsoft Purview Endpoint DLP closes that gap by intercepting those operations at the moment they occur and applying policy based on the content of the file. For how to configure Endpoint DLP, sensitivity labels, and advanced label-based protection for agent workstations, see Chapter 38.

---

