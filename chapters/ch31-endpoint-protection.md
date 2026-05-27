## Chapter 31: Endpoint Protection

### Attack Surface Reduction (ASR) Rules

Deploy via Intune Endpoint Protection profiles:

| ASR Rule | GUID | Purpose |
|----------|------|---------|
| Block process creation from PSExec/WMI | `d1e49aac-8f56-4280-b9ba-993a6d77406c` | Break exploit kill chains |
| Block untrusted/unsigned executables | `01443614-cd74-433a-b99e-2ecdc07bfc25` | Prevent malware execution |
| Block JS/VBS launching downloaded content | `d3e037e1-3eb8-44c8-a917-57927947596d` | Prevent script-based attacks |

### AppLocker / WDAC

> **⚠️ Note:** WDAC with Constrained Language Mode represents the **strictest security posture** and is appropriate only for environments with the highest security requirements. Enforcing CLM will break many developer PowerShell workflows, including custom modules, script-based build tools, and ad-hoc scripting all require Full Language Mode. Evaluate the developer workflow impact carefully before enabling CLM, and expect significant effort to produce a working WDAC policy that allows legitimate development activities while blocking malicious ones.

**Windows Defender Application Control (WDAC)** (optional, strict environments only):
- Enforce **Constrained Language Mode** for PowerShell (limits .NET API access from scripts)
- Limit the ability of agents or malicious skills to call dangerous .NET APIs
- Requires extensive testing and policy tuning before deployment

**AppLocker Path Rules (less restrictive alternative):**
- **Allow:** `C:\Program Files\nodejs\*`, Node.js and global npm packages
- **Deny:** `C:\Users\*\AppData\Local\Temp\*`, block execution from temp directories

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

