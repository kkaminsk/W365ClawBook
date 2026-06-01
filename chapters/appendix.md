## Appendix

### Operational Quick Reference

### Build a New Image

```powershell
cd terraform
terraform init
terraform plan -var-file="terraform.tfvars" -out tfplan
terraform apply tfplan
# Wait 60-90 minutes
.\scripts\Teardown-BuildResources.ps1
```

### Version Bump

```powershell
terraform apply -var='image_version=1.1.0' -var='exclude_from_latest=true'
```

After the first apply completes, test the new version with your pilot provisioning group before proceeding to production rollout. Do not run the second apply until pilot testing is complete and the build has passed the Chapter 15 verification checklist.

```powershell
terraform apply -var='image_version=1.1.0' -var='exclude_from_latest=false'
.\scripts\Teardown-BuildResources.ps1
```

### Hotfix

```powershell
terraform apply -var='image_version=1.0.1' -var='exclude_from_latest=false'
.\scripts\Teardown-BuildResources.ps1
```

### Update Agents on Running Cloud PCs

These commands are the payload of an Intune Platform Script (PowerShell) running in System context, not interactive commands to execute on individual Cloud PCs. Deploy them through Intune for organization-wide rollout, or run them manually on a specific machine for targeted troubleshooting only.

```powershell
# Deploy via Intune platform script
npm install -g openclaw@2026.3.1
npm install -g @anthropic-ai/claude-code@2.2.0
```

### Verify Image

```powershell
Get-AzGalleryImageVersion `
  -ResourceGroupName "rg-w365-images" `
  -GalleryName "acgW365Dev" `
  -GalleryImageDefinitionName "W365-W11-25H2-ENU" |
  Format-Table Name, ProvisioningState, PublishingProfile
```

### Cleanup Old Versions

```powershell
Remove-AzGalleryImageVersion `
  -ResourceGroupName "rg-w365-images" `
  -GalleryName "acgW365Dev" `
  -GalleryImageDefinitionName "W365-W11-25H2-ENU" `
  -Name "1.0.0" -Force
```

---

*Azure Compute Gallery integration with Windows 365 is generally available. Test in non-production environments before adopting for production workloads.*

---

### Required Azure Role Assignments

Minimum RBAC assignments required before running `terraform apply`. See Chapter 5 for the full provisioning walkthrough.

| Identity | Role | Scope | Why |
|---|---|---|---|
| Terraform service principal | Contributor | Subscription or resource group | Create ACG, AIB template, networking resources |
| Terraform service principal | User Access Administrator | Subscription | Assign roles to AIB managed identity |
| AIB managed identity | Contributor | Image resource group | AIB needs to read/write during build |
| AIB managed identity | Storage Blob Data Reader | Build artifact storage account | Read installer scripts and config files |
| Intune service account | Intune Administrator | Tenant | Import images, deploy Win32 apps and scripts |
| Intune service account | Cloud Device Administrator | Tenant | Import Windows 365 image from ACG |
| Agent account (secondary Entra ID) | Key Vault Secrets User | Key Vault | Read secrets at agent runtime (Ch39) |
| Agent account (secondary Entra ID) | No additional Azure roles | — | Agent should not have ARM-level access |

---

### Sysmon Event Monitoring Reference

Deploy Sysmon (see Chapter 33) and alert on these event IDs for agent-related threats.

| Event ID | Category | What to Alert On | Threat |
|---|---|---|---|
| **1** | Process Create | `node.exe` or `openclaw` spawning `cmd.exe`/`powershell.exe` with `-encodedCommand`, `DownloadString`, or `Invoke-Expression` | Prompt injection → shell execution |
| **3** | Network Connection | Agent processes connecting to ports other than 80/443 | Lateral movement, C2 communication |
| **11** | File Create | `.exe`, `.bat`, `.ps1` created in agent working dir or `%TEMP%` | Malware staging |
| **12/13** | Registry | Modifications to Run keys or Scheduled Tasks | Persistence |
| **4688** | Security (Process) | Parent: `node.exe` or `openclaw`; Child: `cmd.exe`, `powershell.exe` | Shell spawn from agent |
| **4663** | Security (Object) | Agent process accessing `.ssh`, `Credentials`, `Vault`, or `Cookies` paths | Credential access |
| **4698** | Security (Scheduled Task) | Scheduled task created by agent account UPN | Persistence |

---

### Key CVE Reference

Critical and high-severity CVEs affecting agents deployed in this guide. See Chapter 28 (Claude Code) and Chapter 29 (OpenClaw) for full tables and remediation guidance.

| CVE | Product | CVSS | Summary | Minimum Safe Version |
|---|---|---|---|---|
| CVE-2025-54794 | Claude Code | 7.7 | Path restriction bypass via project settings file | 1.0.94+ |
| CVE-2025-59536 | Claude Code | 8.7 | Hooks RCE (Check Point disclosure) via project settings file | 1.0.111+ |
| CVE-2026-21852 | Claude Code | 5.3 | API key exfiltration via `ANTHROPIC_BASE_URL` | 2.0.65+ |
| CVE-2026-25253 | OpenClaw | 8.8 | Auth token exfiltration via `gatewayUrl` parameter → RCE | 2026.1.29 |
| CVE-2026-28472 | OpenClaw | 8.5 | ClawJacked — WebSocket brute-force bypass | 2026.2.25 |
| CVE-2026-32922 | OpenClaw | 9.9 | Privilege escalation via pairing token → full RCE | 2026.3.12 |
| CVE-2026-41349 | OpenClaw | 8.3 | Agentic consent bypass via `config.patch` | 2026.3.28 |

**Minimum required versions:** Claude Code 2.1.90+; OpenClaw 2026.3.28.

---

### First Deployment Checklist

Follow this sequence on a new tenant. Each item links to its source chapter.

**Phase 1: Infrastructure (Chapters 4–6)**
- [ ] Azure Compute Gallery created with image definition (Ch4)
- [ ] Service principals and managed identities created with RBAC above (Ch5)
- [ ] Terraform backend configured; `terraform init` succeeds (Ch6)
- [ ] All `terraform.tfvars` version pins set to current approved versions (Ch13, Ch37)

**Phase 2: Build Workstation (Chapter 7)**
- [ ] Build workstation VM provisioned in the correct VNet
- [ ] Initialize-BuildWorkstation.ps1 run successfully (Ch35)

**Phase 3: Image Build (Chapters 8–13)**
- [ ] Phase 1 (core runtimes) script tested and SHA256 checksums verified (Ch8, Ch13)
- [ ] Phase 2 (developer tools) script tested; VS Code installs as System installer (Ch9)
- [ ] Phase 3 (config and policy) script tested; Active Setup key registered (Ch10)
- [ ] Windows Update and Sysprep complete without errors (Ch12)
- [ ] SBOM generated at `C:\ProgramData\ImageBuild\sbom-*.json` (Ch13)

**Phase 4: Operations Setup (Chapters 14–21)**
- [ ] Image built successfully; ACG image version at `ProvisioningState: Succeeded` (Ch14)
- [ ] Chapter 15 verification checklist passed on a test Cloud PC (Ch15)
- [ ] Image imported into Windows 365 (Intune admin center) (Ch17)
- [ ] CI/CD pipeline connected to image build trigger (Ch21)

**Phase 5: Intune Configuration (Chapters 23–25, 32)**
- [ ] All Intune policies in the Complete Policy Inventory table above deployed (Ch32)
- [ ] OneDrive Known Folder Move enforced for all device groups (Ch32, Ch41)
- [ ] API key delivery (Settings Catalog env var) deployed (Ch23)
- [ ] Agent Win32 apps (OpenClaw, Claude Code, Codex, OpenSpec) deployed (Ch25)
- [ ] Detection scripts version constants match pinned payload versions (Ch25)

**Phase 6: Security (Chapters 26–33, 38–39)**
- [ ] OpenClaw gateway binds to `127.0.0.1` only; verified with `netstat` (Ch29)
- [ ] Sysmon deployed and sending events to Log Analytics (Ch33)
- [ ] KQL alert queries activated in Sentinel (Ch33)
- [ ] Purview DLP policies active; PS7 limitation documented for automation team (Ch38)
- [ ] Key Vault provisioned; agent account granted `Key Vault Secrets User` (Ch39)
- [ ] Secondary Entra ID agent accounts created per developer (Ch27)
- [ ] Conditional Access policies applied to agent accounts (Ch27)

**Phase 7: DR Readiness (Chapter 41)**
- [ ] ACG image replicated to backup region (Ch41)
- [ ] Azure Network Connection created in backup region (Ch41)
- [ ] Failover provisioning policy tested in pilot (Ch41)
- [ ] DR test completed and documented (Ch41)

### Feedback and Errata

If you find an error, version mismatch, or broken link, file an issue at **https://github.com/kkaminsk/W365ClawBook/issues** with:

- The exact section or heading name
- Your environment details (region, SKU, image version)
- The command output or error text
- The date of the build

This keeps the book and the repository aligned and makes future image updates safer.

---

*(c) 2026 Kevin Kaminski. This work is licensed under a [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).*



