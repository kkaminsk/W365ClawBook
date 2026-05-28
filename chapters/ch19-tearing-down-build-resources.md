## Chapter 19: Tearing Down Build Resources

### Why Not `terraform destroy`?

The gallery and image definition have `lifecycle { prevent_destroy = true }` to prevent accidental deletion. Running `terraform destroy` would fail on these protected resources.

### The Teardown-BuildResources.ps1 Script

The solution includes a targeted teardown script that removes only the build-time resources:

```powershell
<#
.SYNOPSIS
    Removes AIB build resources while preserving the gallery and image versions.
#>

[CmdletBinding()]
param(
    [switch]$Force,
    [string]$TerraformDir = (Join-Path $PSScriptRoot "..\terraform")
)

$ErrorActionPreference = "Stop"
$TerraformDir = (Resolve-Path $TerraformDir -ErrorAction Stop).Path

Write-Host "This will remove:" -ForegroundColor Yellow
Write-Host "  -- AIB image template" -ForegroundColor Yellow
Write-Host "  -- Build action" -ForegroundColor Yellow
Write-Host "  -- Build timestamp" -ForegroundColor Yellow
Write-Host ""
Write-Host "This will PRESERVE:" -ForegroundColor Green
Write-Host "  -- Azure Compute Gallery" -ForegroundColor Green
Write-Host "  -- Image definition" -ForegroundColor Green
Write-Host "  -- All image versions" -ForegroundColor Green
Write-Host "  -- Resource group" -ForegroundColor Green
Write-Host "  -- Managed identity + RBAC" -ForegroundColor Green

if (-not $Force) {
    $response = Read-Host "Continue? [Y/n]"
    if ($response -ne "" -and $response -ne "Y" -and $response -ne "y") {
        Write-Host "Aborted." -ForegroundColor Red
        exit 0
    }
}

Push-Location $TerraformDir
try {
    terraform destroy `
        -target="module.image_builder" `
        -var-file="terraform.tfvars" `
        -auto-approve
} finally {
    Pop-Location
}
```

The `-target` flag limits Terraform to the `image_builder` module only, leaving the Azure Compute Gallery (ACG), managed identity, and resource groups untouched — exactly the scope needed to release build compute costs without affecting the image gallery.

### Cost Optimization Workflow

> **Warning:** Do not run teardown if the build failed. The `IT_rg-w365-images_*` staging resource group contains the AIB (Azure VM Image Builder) build log and the partially-built VM needed for diagnosis. Run teardown only after successful build verification. Deleting the staging group prematurely destroys the diagnostic evidence.

```text
terraform apply -> wait for build (~60-90 min) -> verify -> Teardown-BuildResources.ps1
```

After teardown, only these resources remain (and incur cost):

- Azure Compute Gallery (ACG) (minimal cost)
- Image definition (no cost)
- Image versions (storage cost per version per replica)
- Resource group (no cost)
- Managed identity (no cost)

Image versions cost approximately $0.05 per GB per month in Standard LRS storage; see Chapter 20 for a complete retention cost model.

---

