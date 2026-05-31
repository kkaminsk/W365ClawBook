## Chapter 37: Component Summary Matrix

This matrix lists every component installed during the build pipeline phases. The Execution Context column uses the format "Build Phase / Windows Account" — for example, "Image Build / Local System" means the component is installed during AIB (Azure VM Image Builder) execution running as NT AUTHORITY\SYSTEM. Use this matrix to cross-reference installation decisions with the corresponding chapter.

| Component | Version | Install Method | Execution Context | Key Consideration |
|---|---|---|---|---|
| **Node.js** | v24.13.1 | MSI (`ALLUSERS=1`) | Image Build / Local System | Refresh session PATH after install |
| **Python** | 3.14.3 | EXE (`InstallAllUsers=1`) | Image Build / Local System | `PrependPath=1` for global access |
| **PowerShell 7** | 7.4.13 | MSI | Image Build / Local System | Pin URL to specific release (not `/latest/`) |
| **VS Code** | Latest | System Installer (EXE) | Image Build / Local System | `/MERGETASKS=!runcode` to prevent hang |
| **Git** | 2.53.0 | Inno Setup (EXE) | Image Build / Local System | `/PathOption=Cmd` for PATH registration |
| **GitHub Desktop** | Latest | Machine-Wide MSI | Image Build / Local System | Hydrates into user profile at first login |
| **Azure CLI** | 2.83.0 | MSI (`ALLUSERS=1`) | Image Build / Local System | Used for Terraform authentication |
| **GitHub Copilot** | Latest | VS Code extension | Post-Provisioning / User Context | Cannot install machine-wide reliably; deploy via Intune user-context script |
| **OpenClaw** | 2026.x.x (verify current stable at npmjs.com/package/openclaw before build) | `npm install -g` | Post-Provisioning / User Context | Intune Win32 required app; never run `openclaw onboard` during build; as of May 2026, minimum patched version is 2026.3.28 (see Chapter 29 CVE table); Agent 365 (GA May 2026) can detect unauthorized OpenClaw instances via Shadow AI discovery |
| **Claude Code** | 2.1.x (verify current stable at npmjs.com/package/@anthropic-ai/claude-code before build) | `npm install -g` | Post-Provisioning / User Context | Intune Win32 required app; never run `claude login` during build |
| **Cline** | v3.81+ | VS Code extension or `npm install -g @cline/cline` | Post-Provisioning / User Context | Optional third agent; Enterprise edition available with remote configuration dashboard and model allowlisting; lower persistence risk than OpenClaw (VS Code-embedded, no background service) |
| **OpenSpec** | 0.9.1 | `npm install -g` | Post-Provisioning / User Context | Intune Win32 required app (per-user) |
| **Codex CLI** | 0.101.0 | `npm install -g` | Post-Provisioning / User Context | Intune Win32 required app; OpenAI's code generation CLI |
| **OpenClaw Config** | -- | Template + Active Setup | Image Build + First Login | Template in ProgramData, copied to user profile |
| **Agent Skills (curated)** | -- | Git clone / file copy | Image Build + First Login | Vetted via Cisco Skill Scanner; copied to `~/.agents/skills/` |
| **MCP Servers (stdio)** | -- | `npm install -g` | Post-Provisioning / User Context | Intune Win32 available app (per-user); installed from Company Portal on demand |
| **MCP Server Config** | -- | Template + Active Setup | Image Build + First Login | API key placeholders; real keys via Intune env vars or Azure Key Vault. |
| **Claude Code Policy** | -- | `managed-settings.json` | Image Build / Local System | Machine-level enterprise governance |
| **API Keys** | N/A (secret value) | Azure Key Vault / Intune Settings Catalog | Post-Provisioning | **Never bake secrets into the image** |
| **VS Code Extensions** | -- | `code --install-extension` | Post-Provisioning / User Context | Cannot install machine-wide reliably |
| **Agent Updates** | -- | `npm update -g` | Intune Script on Running Cloud PCs | No reprovisioning needed |
| **Purview Endpoint DLP** | -- | No separate install; rides MDE onboarding | Cloud Policy / MDE sensor on device | Requires Entra join, MDE antimalware 4.18.2110+, real-time protection, and `MpDlpService.exe` firewall allowance |
| **Purview Information Protection client** | 3.1.309+ (for advanced label protection) | `.exe` or `.msi`; silent deploy via Intune | Post-Provisioning / User Context | Windows only; ARM64 supports viewer and file labeler only; internet required to apply encryption |
| **Windows 365 (standard SKU)** | Current | Microsoft 365 admin center / Intune | Platform / Cloud service | Per-user monthly subscription; GPU-enabled SKUs available for heavy AI inference workloads; Windows 365 for Agents — new consumption-based SKU in public preview (announced January 22, 2026); purpose-built for agentic AI workloads; GA projected Q4 2026 |
| **Microsoft Agent 365** | GA May 2026 | Intune / Microsoft Security portal | Post-Provisioning / Cloud service (no agent install) | Requires licensing at $15/user/month standalone; extends Shadow AI detection and asset context mapping for all AI agents on the fleet; see Chapter 27 and Chapter 31 |

---

