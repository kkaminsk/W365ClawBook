## Chapter 20: Version Retention and Cost Management

### Storage Costs

ACG image versions incur storage costs per version per region replica. For a single image definition with a few versions and one region, this is typically a few dollars per month.

### Retention Policy

Retain the last 3 versions. Remove older versions to reduce storage costs:

```powershell
# List all versions
Get-AzGalleryImageVersion `
  -ResourceGroupName "rg-w365-images" `
  -GalleryName "acgW365Dev" `
  -GalleryImageDefinitionName "W365-W11-25H2-ENU" |
  Format-Table Name, ProvisioningState, PublishingProfile

# Delete a specific old version
Remove-AzGalleryImageVersion `
  -ResourceGroupName "rg-w365-images" `
  -GalleryName "acgW365Dev" `
  -GalleryImageDefinitionName "W365-W11-25H2-ENU" `
  -Name "1.0.0" `
  -Force
```

### End-of-Life Date

Each image version has an `end_of_life_date` set to 90 days from build. This is a governance signal; it doesn't automatically delete the version, but it provides visibility into which versions are stale.

### Replication Strategy

Configure replication based on where your Azure Network Connections are, not where your users are. Windows 365 handles its own replication after ingestion. The solution defaults to a single replica in the build region, which is sufficient for single-region deployments.

The Terraform module currently uses single-region replication only (one replica in `var.location`). To replicate to additional regions, add `target_region` blocks to the `azurerm_shared_image_version` resource — this is fully supported in the AzureRM provider:

```hcl
resource "azurerm_shared_image_version" "this" {
  # ... existing configuration ...

  target_region {
    name                   = var.location   # primary region (required)
    regional_replica_count = 1
    storage_account_type   = "Standard_LRS"
  }

  target_region {
    name                   = "westus2"      # add one block per additional region
    regional_replica_count = 1
    storage_account_type   = "Standard_LRS"
  }
}
```

Add one `target_region` block per region that hosts an Azure Network Connection for your Windows 365 deployment. Each additional region replica incurs storage costs (proportional to the image size, typically a few dollars per month per region). Replication is asynchronous; the build completes in the primary region first, then replicas propagate to target regions over the next 15--30 minutes.

---

