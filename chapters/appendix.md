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

### Feedback and Errata

If you find an error, version mismatch, or broken link, file an issue at **https://github.com/kkaminsk/W365ClawBook/issues** with:

- The exact section or heading name
- Your environment details (region, SKU, image version)
- The command output or error text
- The date of the build

This keeps the book and the repository aligned and makes future image updates safer.

---

*(c) 2026 Kevin Kaminski. This work is licensed under a [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).*



