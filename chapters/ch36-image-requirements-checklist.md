## Chapter 36: Windows 365 Image Requirements Checklist

Use this checklist as a pre-import gate before submitting a custom image to Windows 365 through Intune. Items are organized by requirement type; items marked [Auto] can be incorporated into a CI/CD pipeline gate, while items marked [Manual] require portal or console verification.

### General Image Requirements

- [ ] Windows 10 or Windows 11 Enterprise (supported version)
- [ ] Generation 2 (Gen2) virtual machine
- [ ] Generalized via Sysprep
- [ ] Never previously Entra joined, AD joined, or Intune enrolled
- [ ] Single-session (no multi-session)
- [ ] No recovery partitions
- [ ] No data disks
- [ ] No pre-installed FSLogix components (deploy via Intune policy post-provisioning; pre-baking into the image can conflict with Intune-managed FSLogix configuration)
- [ ] Under 3,000 Start menu apps (soft guideline; validate during testing) (Soft guideline — not enforced by Windows 365 import validation; affects first-login experience)
- [ ] Not using disk encryption sets
- [ ] Default 64 GB OS disk size (Windows 365 adjusts to the licence SKU)

### Azure Compute Gallery Requirements

- [ ] x64 architecture
- [ ] Windows OS type
- [ ] Hyper-V Generation V2
- [ ] Generalized OS state
- [ ] `SecurityType` = `TrustedLaunchSupported`
- [ ] `IsHibernateSupported` = `True`
- [ ] `DiskControllerTypes` = `SCSI,NVMe`
- [ ] `IsAcceleratedNetworkSupported` = `True`
- [ ] `IsSecureBootSupported` = `True`

### Build-Time Verification

- [ ] All software versions match pinned values in `terraform.tfvars`
- [ ] SBOM files exist at `C:\ProgramData\ImageBuild\sbom-*.json`
- [ ] `C:\ProgramData\ClaudeCode\managed-settings.json` contains correct policy (see Chapter 12 for the managed-settings.json policy schema; verify that all required enterprise governance keys are present and set to their expected values)
- [ ] `C:\ProgramData\OpenClaw\template-config.json` contains correct configuration (verify `gateway.port`, `auth.mode`, and `skills.directory` match the values defined in your Intune configuration profile; see Chapter 32)
- [ ] Active Setup registry key exists for OpenClaw config hydration (see Chapter 10)
- [ ] Curated skills exist at `C:\ProgramData\OpenClaw\skills\`
- [ ] MCP server config template exists at `C:\ProgramData\OpenClaw\mcp\mcporter.json`
- [ ] MCP server API key placeholders are present (not real keys)
- [ ] Teams `IsWVDEnvironment` = 1 (IsWVDEnvironment — a legacy registry key; WVD = Windows Virtual Desktop, the pre-AVD product name; still required by Windows 365 at time of writing)
- [ ] No high/critical npm audit findings (Run `npm audit --audit-level=high`; exit 0 required)
- [ ] `end_of_life_date` set to 90 days from build

### Post-Provisioning Verification

- [Auto] Cloud PC provisions successfully from the image
- [Manual] Developer can sign in and OpenClaw config appears in `~/.openclaw/`
- [Auto] Curated skills appear in `~/.agents/skills/`
- [Auto] MCP server config appears in `~/.openclaw/workspace/config/mcporter.json`
- [Manual] GitHub Desktop hydrates on first login
- [Manual] `claude` CLI works after user configures their API key
- [Manual] Teams media optimization is active
- [Auto] VS Code extensions install via Intune script

---

