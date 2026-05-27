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

The Terraform module currently uses single-region replication only (one replica in `var.location`).

> **Future enhancement:** Native Terraform support for multi-region replication (`target_regions` and per-region replica counts) is planned but not yet implemented.

Until this is implemented, replicate image versions to additional regions with manual Azure CLI commands (for example, `az sig image-version update` with `--target-regions`).

Each additional region replica incurs storage costs (proportional to the image size, typically a few dollars per month per region). Replication is asynchronous; the build completes in the primary region first, then replicas propagate to target regions over the next 15--30 minutes.

---

