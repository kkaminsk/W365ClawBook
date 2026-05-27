## Chapter 36: Windows 365 Image Requirements Checklist

### General Image Requirements

- [ ] Windows 10 or Windows 11 Enterprise (supported version)
- [ ] Generation 2 (Gen2) virtual machine
- [ ] Generalized via Sysprep
- [ ] Never previously Entra joined, AD joined, or Intune enrolled
- [ ] Single-session (no multi-session)
- [ ] No recovery partitions
- [ ] No data disks
- [ ] No FSLogix components
- [ ] Under 3,000 Start menu apps
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
- [ ] `C:\ProgramData\ClaudeCode\managed-settings.json` contains correct policy
- [ ] `C:\ProgramData\OpenClaw\template-config.json` contains correct configuration
- [ ] Active Setup registry key exists for OpenClaw config hydration
- [ ] Curated skills exist at `C:\ProgramData\OpenClaw\skills\`
- [ ] MCP server config template exists at `C:\ProgramData\OpenClaw\mcp\mcporter.json`
- [ ] MCP server API key placeholders are present (not real keys)
- [ ] Teams `IsWVDEnvironment` = 1
- [ ] No high/critical npm audit findings
- [ ] `end_of_life_date` set to 90 days from build

### Post-Provisioning Verification

- [ ] Cloud PC provisions successfully from the image
- [ ] Developer can sign in and OpenClaw config appears in `~/.openclaw/`
- [ ] Curated skills appear in `~/.agents/skills/`
- [ ] MCP server config appears in `~/.openclaw/workspace/config/mcporter.json`
- [ ] GitHub Desktop hydrates on first login
- [ ] `claude` CLI works after user configures their API key
- [ ] Teams media optimisation is active
- [ ] VS Code extensions install via Intune script

---

