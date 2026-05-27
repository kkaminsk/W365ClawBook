## Chapter 32: Intune Configuration Profiles

### OMA-URI Settings

Deploy hardening configurations via Intune Custom OMA-URI profiles:

| Setting | OMA-URI | Value |
|---------|---------|-------|
| Disable WebClient | PowerShell script | `Set-Service -Name WebClient -StartupType Disabled; Stop-Service -Name WebClient -Force -ErrorAction SilentlyContinue` |
| Windows Sandbox (no audio) | `./Device/Vendor/MSFT/Policy/Config/WindowsSandbox/AllowAudioInput` | `0` |
| Windows Sandbox (no network) | `./Device/Vendor/MSFT/Policy/Config/WindowsSandbox/AllowNetworking` | `0` |

### Conditional Access for Agent Identity

When using a Secondary User (Agent Identity):

- **Device Compliance:** Require the Cloud PC to be marked as "Compliant" (BitLocker, Secure Boot, Defender active)
- **Location Fencing:** Restrict the Agent Identity's login to Windows 365 gateway IPs or corporate VPN
- **MFA:** Require phishing-resistant MFA (FIDO2) for agent identity authentication

### Data Loss Prevention via Purview

Configure Endpoint DLP policies targeting the agent identity group:

- **Condition:** File contains Sensitive Info Types (source code patterns, API keys, PII)
- **Action:** Block upload to unapproved domains; allow only authorized endpoints (e.g., `github.com/my-org`)
- **Sensitivity Labels:** Auto-apply "Internal Only" label to files created/modified by the agent identity

---

