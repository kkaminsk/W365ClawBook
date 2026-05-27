## Appendix: Operational Quick Reference

## Appendix: Feedback and Errata

If you find an error, version mismatch, or broken link, file an issue in the W365Claw repository with:

- The exact section or heading name
- Your environment details (region, SKU, image version)
- The command output or error text
- The date of the build

This keeps the book and the repository aligned and makes future image updates safer.

---



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
# Test with pilot group
terraform apply -var='image_version=1.1.0' -var='exclude_from_latest=false'
.\scripts\Teardown-BuildResources.ps1
```

### Hotfix

```powershell
terraform apply -var='image_version=1.0.1' -var='exclude_from_latest=false'
.\scripts\Teardown-BuildResources.ps1
```

### Update Agents on Running Cloud PCs

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

*Azure Compute Gallery integration with Windows 365 was in public preview at the time of writing. Verify the current status at [learn.microsoft.com/windows-365](https://learn.microsoft.com/windows-365/); feature behaviour may change between preview and general availability. Test thoroughly in non-production environments before adopting for production workloads.*

*(c) 2026 Kevin Kaminski. This work is licensed under a [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).*



