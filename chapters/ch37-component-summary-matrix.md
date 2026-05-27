## Chapter 37: Component Summary Matrix

| Component | Version | Install Method | Execution Context | Key Consideration |
|---|---|---|---|---|
| **Node.js** | v24.13.1 | MSI (`ALLUSERS=1`) | Image Build / Local System | Refresh session PATH after install |
| **Python** | 3.14.3 | EXE (`InstallAllUsers=1`) | Image Build / Local System | `PrependPath=1` for global access |
| **PowerShell 7** | 7.4.13 | MSI | Image Build / Local System | Pin URL to specific release (not `/latest/`) |
| **VS Code** | Latest | System Installer (EXE) | Image Build / Local System | `/MERGETASKS=!runcode` to prevent hang |
| **Git** | 2.53.0 *(verify against pinned value in terraform.tfvars)* | Inno Setup (EXE) | Image Build / Local System | `/PathOption=Cmd` for PATH registration |
| **GitHub Desktop** | Latest | Machine-Wide MSI | Image Build / Local System | Hydrates into user profile at first login |
| **Azure CLI** | 2.83.0 | MSI (`ALLUSERS=1`) | Image Build / Local System | Used for Terraform authentication |
| **GitHub Copilot** | Latest | VS Code extension | Post-Provisioning / User Context | Cannot install machine-wide reliably; deploy via Intune user-context script |
| **OpenClaw** | 2026.2.14 | `npm install -g` | Post-Provisioning / User Context | Intune Win32 required app; never run `openclaw onboard` during build |
| **Claude Code** | 2.1.42 | `npm install -g` | Post-Provisioning / User Context | Intune Win32 required app; never run `claude login` during build |
| **OpenSpec** | 0.9.1 | `npm install -g` | Post-Provisioning / User Context | Intune Win32 required app (per-user) |
| **Codex CLI** | 0.101.0 | `npm install -g` | Post-Provisioning / User Context | Intune Win32 required app; OpenAI's code generation CLI |
| **OpenClaw Config** | -- | Template + Active Setup | Image Build + First Login | Template in ProgramData, copied to user profile |
| **Agent Skills (curated)** | -- | Git clone / file copy | Image Build + First Login | Vetted via Cisco Skill Scanner; copied to `~/.agents/skills/` |
| **MCP Servers (stdio)** | -- | `npm install -g` | Post-Provisioning / User Context | Intune Win32 available app (per-user); installed from Company Portal on demand |
| **MCP Server Config** | -- | Template + Active Setup | Image Build + First Login | API key placeholders; real keys via Intune env vars or Azure Key Vault. |
| **Claude Code Policy** | -- | `managed-settings.json` | Image Build / Local System | Machine-level enterprise governance |
| **API Keys** | -- | Azure Keyvault / Intune Settings Catalog | Post-Provisioning | **Never bake secrets into the image** |
| **VS Code Extensions** | -- | `code --install-extension` | Post-Provisioning / User Context | Cannot install machine-wide reliably |
| **Agent Updates** | -- | `npm update -g` | Intune Script on Running Cloud PCs | No reprovisioning needed |
| **Purview Endpoint DLP** | -- | No separate install; rides MDE onboarding | Cloud Policy / MDE sensor on device | Requires Entra join, MDE antimalware 4.18.2110+, real-time protection, and `MpDlpService.exe` firewall allowance |
| **Purview Information Protection client** | 3.1.309+ (for advanced label protection) | `.exe` or `.msi`; silent deploy via Intune | Post-Provisioning / User Context | Windows only; ARM64 supports viewer and file labeler only; internet required to apply encryption |

---

