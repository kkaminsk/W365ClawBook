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

### OneDrive Known Folder Move

OneDrive Known Folder Move (KFM) redirects the Desktop, Documents, and Pictures folders to OneDrive, ensuring their contents survive Cloud PC reprovisioning and regional failover. **This is a mandatory configuration for any Windows 365 deployment, not an optional enhancement.** Without KFM, files saved to these locations are lost during a provisioning event or DR failover — see Chapter 41.

Configure KFM via the Intune Settings Catalog:

| Setting Path | Setting Name | Value |
|---|---|---|
| OneDrive | Silently sign in users to the OneDrive sync app with their Windows credentials | Enabled |
| OneDrive | Silently move Windows known folders to OneDrive | Enabled — provide your Azure AD tenant ID |
| OneDrive | Prevent users from redirecting their Windows known folders to their PC | Enabled |

**Why "Prevent users from redirecting" matters:** This locks KFM in place. Without it, a developer can disable KFM from the OneDrive sync app settings and unknowingly remove their DR protection.

Deploy this Settings Catalog profile to all Cloud PC device groups (Standard, Developer, and Admin rings). Verify KFM is active on a Cloud PC with:

```powershell
# Verify Desktop, Documents, Pictures are redirected to OneDrive
Get-ItemProperty "HKCU:\Software\Microsoft\OneDrive\Accounts\Business1" |
    Select-Object UserFolder, KfmFoldersProtected, KfmSilentOptInError
```

`KfmFoldersProtected` should be non-zero if KFM is active and enforced.

---

### Agent 365 Policy Integration

Microsoft Agent 365 (GA May 1, 2026) exposes Intune-compatible policies for governing locally running AI agents. For organizations licensed for Agent 365, these policies extend the manual Intune profiles in this chapter:

- **Agent deployment controls:** Agent 365 can block npm-based agent installations via Intune policy, supplementing the WDAC/AppLocker execution controls
- **MCP server allowlist:** Agent 365 integrates with the VS Code MCP allowlist controls (public preview, November 2025) for centralized MCP governance
- **Conditional Access integration:** Agent 365 contributes agent-specific risk signals to Entra Conditional Access, enabling access decisions based on detected agent behavior rather than only device compliance state

Configure Agent 365 policies in **Endpoint security → Agent governance** within the Intune admin center, alongside the existing profiles described in this chapter.

---

### Complete Policy Inventory

This table lists every Intune policy recommended across the book. Use it as a deployment checklist when configuring a new tenant from scratch. For rationale and detailed settings, follow the chapter references.

| Policy | Type | Scope | Chapter |
|---|---|---|---|
| Windows Update for Business — quality deferral (7 days) | Settings Catalog | Standard, Developer, Admin | Ch32 |
| Windows Update for Business — feature deferral (30 days) | Settings Catalog | Standard, Developer, Admin | Ch32 |
| Telemetry restriction (Security/Basic only) | Settings Catalog | Standard, Developer, Admin | Ch32 |
| WebClient service disable | Platform Script (System) | Standard, Developer, Admin | Ch32 |
| Developer Mode enable | Settings Catalog | Developer, Admin | Ch32 |
| PowerShell execution policy (RemoteSigned) | Settings Catalog | Developer, Admin | Ch32 |
| WDAC base policy (Audit mode for Admin) | App Control | Admin | Ch31, Ch32 |
| WDAC supplemental policy (Node.js / npm allowance) | App Control | Developer | Ch31, Ch32 |
| OneDrive Known Folder Move — silent opt-in | Settings Catalog | Standard, Developer, Admin | Ch32, Ch41 |
| OneDrive — prevent users from disabling KFM | Settings Catalog | Standard, Developer, Admin | Ch32, Ch41 |
| Defender Antivirus — real-time protection | Settings Catalog | Standard, Developer, Admin | Ch31 |
| Defender Antivirus — behavior monitoring | Settings Catalog | Standard, Developer, Admin | Ch31 |
| Defender Antivirus — cloud-delivered protection | Settings Catalog | Standard, Developer, Admin | Ch31 |
| Attack Surface Reduction rules | Settings Catalog | Standard, Developer, Admin | Ch31 |
| Windows Firewall — domain / private / public profiles | Settings Catalog | Standard, Developer, Admin | Ch30 |
| Windows Firewall outbound block — SMB (TCP 445) | Firewall rule | Standard, Developer, Admin | Ch30 |
| Windows Firewall outbound block — NTLM (TCP 139) | Firewall rule | Standard, Developer, Admin | Ch30 |
| DNS over HTTPS (DoH) | Settings Catalog | Standard, Developer, Admin | Ch30 |
| OpenClaw gateway port (18789) — inbound block from external | Firewall rule | Standard, Developer, Admin | Ch29, Ch30 |
| Anthropic API key delivery (ANTHROPIC_API_KEY) | Settings Catalog — env var | Standard, Developer, Admin | Ch23 |
| Sysmon deployment and configuration | Platform Script (System) | Standard, Developer, Admin | Ch33 |
| OpenClaw gateway watchdog scheduled task | Platform Script (System) | Standard, Developer, Admin | Ch33 |
| VS Code extension deployment (post-provisioning) | Platform Script (User) | Developer, Admin | Ch24 |
| OpenClaw update script | Platform Script (User) | Developer, Admin | Ch25 |
| Claude Code update script | Platform Script (User) | Developer, Admin | Ch25 |
| OpenClaw — Intune Win32 app (required) | Win32 App | Developer, Admin | Ch25 |
| Claude Code — Intune Win32 app (required) | Win32 App | Developer, Admin | Ch25 |
| Codex CLI — Intune Win32 app (required) | Win32 App | Developer, Admin | Ch25 |
| OpenSpec — Intune Win32 app (required) | Win32 App | Developer, Admin | Ch25 |
| MCP servers — Intune Win32 app (available) | Win32 App | Developer, Admin | Ch25 |
| Device compliance — require BitLocker | Compliance Policy | Standard, Developer, Admin | Ch31 |
| Device compliance — require Secure Boot | Compliance Policy | Standard, Developer, Admin | Ch31 |
| Device compliance — minimum OS version | Compliance Policy | Standard, Developer, Admin | Ch31 |
| Conditional Access — require compliant device (agent accounts) | Conditional Access | Agent account group | Ch27 |
| Conditional Access — location-based fencing (agent accounts) | Conditional Access | Agent account group | Ch27 |
| Agent 365 — agent deployment controls | Endpoint Security → Agent governance | Developer, Admin | Ch32 |
| Agent 365 — MCP server allowlist | Endpoint Security → Agent governance | Developer, Admin | Ch32 |

---

