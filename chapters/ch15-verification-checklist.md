## Chapter 15: Verification Checklist

Use this checklist after every ACG (Azure Compute Gallery) build, before importing the image into Intune. The checklist is organized into three tiers: Tier 1 covers build-time verification, Tiers 2 and 3 cover post-provisioning checks.

After the image build completes and before importing into Windows 365, verify every component:

### Image Version in ACG

```powershell
Get-AzGalleryImageVersion `
  -ResourceGroupName "rg-w365-images" `
  -GalleryName "acgW365Dev" `
  -GalleryImageDefinitionName "W365-W11-25H2-ENU" |
  Format-Table Name, ProvisioningState, PublishingProfile
```

### Software Verification (Create a Test VM)

Create a temporary VM from the image and verify the **build-time installed** components:

- [ ] `node --version` returns v24+
- [ ] `python --version` returns 3.14+
- [ ] `pwsh --version` returns PowerShell 7.4+
- [ ] `git --version` returns expected version
- [ ] `az --version` returns expected Azure CLI version
- [ ] VS Code installed in `C:\Program Files\Microsoft VS Code`
- [ ] `code --list-extensions` includes `GitHub.copilot` and `GitHub.copilot-chat`

After post-provisioning agent delivery (via Intune), additionally verify:

- [ ] `openclaw --version` returns expected version
- [ ] `claude --version` returns expected version
- [ ] `codex --version` returns expected version

If any software verification check fails, access the AIB (Azure VM Image Builder) build log from the staging resource group before running the teardown procedure in Chapter 19.

### Configuration Verification

- [ ] `C:\ProgramData\ClaudeCode\managed-settings.json` exists with correct policy (schema details in Chapter 28)
- [ ] `C:\ProgramData\OpenClaw\template-config.json` exists with correct model
- [ ] Active Setup registry key exists for profile initialization via Active Setup (see Chapter 10 for the mechanism)
- [ ] Teams `IsWVDEnvironment` registry key is set to 1
- [ ] SBOM files exist in `C:\ProgramData\ImageBuild\`

### Image Compliance

- [ ] No recovery partition present
- [ ] Image is generalized (Sysprep completed successfully)
- [ ] Image was never Entra/AD joined or Intune enrolled (failure: Windows 365 custom image import validation rejects previously Entra-joined or Intune-enrolled images — this is one of the most common import rejection causes)
- [ ] `end_of_life_date` is set to 90 days from build date

---

