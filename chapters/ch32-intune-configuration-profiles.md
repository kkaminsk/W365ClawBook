## Chapter 32: Intune Configuration Profiles

### OMA-URI Settings

Deploy hardening configurations via Intune Custom OMA-URI profiles:

| Setting | OMA-URI | Value |
|---------|---------|-------|
| Windows Sandbox (no audio) | `./Device/Vendor/MSFT/Policy/Config/WindowsSandbox/AllowAudioInput` | `0` |
| Windows Sandbox (no network) | `./Device/Vendor/MSFT/Policy/Config/WindowsSandbox/AllowNetworking` | `0` |

> **Note:** Disabling Windows services via Intune requires a Platform Script (PowerShell), not an OMA-URI. See the Platform Scripts section below.

### Conditional Access for Agent Accounts

Conditional Access policies for agent accounts — including device compliance requirements, location-based fencing, and phishing-resistant MFA — are covered in Chapter 27 (Identity Architecture).

### Data Loss Prevention via Purview

DLP policies for agent environments are covered in Chapter 38 (Data Protection with Microsoft Purview).

---

### Configuration Profile Sets

The following profile sets are organized by the three Cloud PC policy rings established in Chapter 31: Standard, Developer, and Admin. Deploy each ring's profiles to the corresponding Intune device group.

#### Standard Ring

| Profile | Type | Deployment Group | Key Settings |
|---|---|---|---|
| Windows Update for Business | Settings Catalog | Standard Cloud PCs | Quality updates deferral: 7 days; Feature updates deferral: 30 days; Active hours: 8am–6pm |
| Telemetry Restriction | Settings Catalog | Standard Cloud PCs | Allow Telemetry: Security (0) or Basic (1) only |
| Script Execution Policy | OMA-URI | Standard Cloud PCs | `./Device/Vendor/MSFT/Policy/Config/TextInput/AllowIMENetworkAccess` — restrict as appropriate |
| WebClient Service Disable | Platform Script | Standard Cloud PCs | PowerShell: `Set-Service -Name WebClient -StartupType Disabled; Stop-Service -Name WebClient -Force` |

#### Developer Ring

| Profile | Type | Deployment Group | Key Settings |
|---|---|---|---|
| Developer Mode | Settings Catalog | Developer Cloud PCs | Allow Developer Unlock: Allowed |
| PowerShell Execution Policy | OMA-URI | Developer Cloud PCs | `./Device/Vendor/MSFT/Policy/Config/TextInput/...` or managed via GPO-equivalent |
| Script Signing | Settings Catalog | Developer Cloud PCs | PowerShell: RemoteSigned minimum |
| Node.js and npm Allowance | App Control | Developer Cloud PCs | Add Node.js and npm to the WDAC supplemental policy (see Chapter 31) |

#### Admin Ring

| Profile | Type | Deployment Group | Key Settings |
|---|---|---|---|
| WDAC Audit Mode | App Control | Admin Cloud PCs | Deploy base policy in Audit mode rather than Enforcement for troubleshooting |
| Local Admin Rights | Settings Catalog | Admin Cloud PCs | Enable standard local admin account for break-glass scenarios |

---

### Platform Scripts

Platform Scripts (Devices > Scripts > Windows > Add in the Intune admin center) allow you to deploy PowerShell scripts to managed devices in System or User context. Use Platform Scripts for tasks that cannot be expressed as OMA-URI values — such as disabling Windows services, configuring registry keys with complex logic, or installing software that does not have a Win32 app package.

**Execution context:** Choose System context when the action requires elevated privileges (e.g., stopping a service). Choose User context when the action must run as the signed-in user (e.g., configuring a user-profile setting).

**Script requirements:** Scripts may be signed or unsigned. When deploying unsigned scripts, Intune applies `-ExecutionPolicy Bypass` automatically for the script run; this does not change the device's execution policy for interactive sessions.

**Example — WebClient service disable script:**

```powershell
# Disable-WebClient.ps1
# Disables the WebClient service to neutralize the WebDAV/NTLM attack surface.
# Deploy via Intune Platform Script, System context, run as 64-bit PowerShell.

Set-Service -Name WebClient -StartupType Disabled -ErrorAction SilentlyContinue
Stop-Service -Name WebClient -Force -ErrorAction SilentlyContinue
```

Deploy this script targeting the Standard Cloud PCs device group. After deployment, verify using the remediation steps in the Verifying Profile Deployment section below.

---

### Verifying Profile Deployment

After deploying configuration profiles, confirm settings applied correctly before expanding to production rings.

**Intune portal:** In the Intune admin center, navigate to Devices > Configuration profiles, select the profile, and review the Device and user check-in status blade. Profiles show per-device status of Succeeded, Failed, Conflict, or Not applicable.

**Verify Defender settings on a Cloud PC:**

```powershell
Get-MpComputerStatus | Select-Object AMServiceEnabled, AntispywareEnabled, AntivirusEnabled, RealTimeProtectionEnabled, BehaviorMonitorEnabled
```

**Verify disabled services:**

```powershell
Get-Service -Name WebClient | Select-Object Name, Status, StartType
```

**Verify Windows Firewall profile state:**

```powershell
Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

For automated verification of the full build and configuration baseline, see the checklist in Chapter 15.

---

### Agent 365 Policy Integration

Microsoft Agent 365 (GA May 1, 2026) exposes Intune-compatible policies for governing locally running AI agents. For organizations licensed for Agent 365, these policies extend the manual Intune profiles in this chapter:

- **Agent deployment controls:** Agent 365 can block npm-based agent installations via Intune policy, supplementing the WDAC/AppLocker execution controls
- **MCP server allowlist:** Agent 365 integrates with the VS Code MCP allowlist controls (public preview, November 2025) for centralized MCP governance
- **Conditional Access integration:** Agent 365 contributes agent-specific risk signals to Entra Conditional Access, enabling access decisions based on detected agent behavior rather than only device compliance state

Configure Agent 365 policies in **Endpoint security → Agent governance** within the Intune admin center, alongside the existing profiles described in this chapter.

---

