## Chapter 35: PowerShell Scripts

### Initialize-BuildWorkstation.ps1

```powershell
<#
.SYNOPSIS
    Prepares a Windows workstation to run the W365Claw Terraform image build.

.DESCRIPTION
    Checks, installs, and configures all prerequisites for building Windows 365
    developer images using the W365Claw Terraform solution:
      - Terraform CLI (>= 1.5.0)
      - Azure CLI (>= 2.60)
      - Git (>= 2.40)
      - Az PowerShell module (>= 12.0)
      - Azure authentication
      - Azure resource provider registration
      - Terraform initialization

    The script is idempotent -- running it on an already-configured machine is a no-op.

.PARAMETER Force
    Skip confirmation prompts and install all missing prerequisites automatically.

.PARAMETER TerraformDir
    Path to the terraform/ directory. Defaults to ..\terraform relative to this script.

.EXAMPLE
    .\Initialize-BuildWorkstation.ps1
    # Interactive mode -- prompts before each installation

.EXAMPLE
    .\Initialize-BuildWorkstation.ps1 -Force
    # Non-interactive -- installs everything without prompting
#>

[CmdletBinding()]
param(
    [switch]$Force,
    [string]$TerraformDir = (Join-Path $PSScriptRoot "..\terraform")
)

$ErrorActionPreference = "Stop"
$ProgressPreference = "SilentlyContinue"

$MinVersions = @{
    Terraform = [version]"1.5.0"
    AzureCLI  = [version]"2.60.0"
    Git       = [version]"2.40.0"
    AzModule  = [version]"12.0.0"
}

$RequiredProviders = @(
    "Microsoft.Compute",
    "Microsoft.VirtualMachineImages",
    "Microsoft.Network",
    "Microsoft.ManagedIdentity"
)

$ProviderRegistrationTimeoutSeconds = 300
$ProviderRegistrationPollSeconds = 10

function Update-SessionPath {
    $machinePath = [Environment]::GetEnvironmentVariable("Path", "Machine")
    $userPath = [Environment]::GetEnvironmentVariable("Path", "User")
    $env:Path = "$machinePath;$userPath"
}

function Get-ParsedVersion {
    param([string]$VersionString)
    if ($VersionString -match '(\d+\.\d+\.\d+)') {
        return [version]$Matches[1]
    }
    return $null
}

function Test-CommandExists {
    param([string]$Command)
    $null -ne (Get-Command $Command -ErrorAction SilentlyContinue)
}

function Test-WingetAvailable {
    Test-CommandExists "winget"
}

function Confirm-Action {
    param([string]$Message)
    if ($Force) { return $true }
    $response = Read-Host "$Message [Y/n]"
    return ($response -eq "" -or $response -eq "Y" -or $response -eq "y")
}

function Test-IsAdmin {
    $identity = [Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = New-Object Security.Principal.WindowsPrincipal($identity)
    return $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}

function Test-IsWindowsDesktop {
    $os = Get-CimInstance Win32_OperatingSystem
    return ($os.Caption -match "Windows 1[01]" -or $os.Caption -match "Windows 11")
}

function Get-TerraformVersion {
    if (-not (Test-CommandExists "terraform")) { return $null }
    $output = terraform version 2>&1 | Select-Object -First 1
    return Get-ParsedVersion $output
}

function Get-AzureCLIVersion {
    if (-not (Test-CommandExists "az")) { return $null }
    try {
        $output = az version 2>&1 | ConvertFrom-Json
        return Get-ParsedVersion $output.'azure-cli'
    } catch { return $null }
}

function Get-GitVersion {
    if (-not (Test-CommandExists "git")) { return $null }
    $output = git --version 2>&1
    return Get-ParsedVersion $output
}

function Get-AzModuleVersion {
    $mod = Get-Module -ListAvailable Az -ErrorAction SilentlyContinue |
           Sort-Object Version -Descending | Select-Object -First 1
    if ($mod) { return $mod.Version }
    return $null
}

function Test-AzureLogin {
    if (-not (Test-CommandExists "az")) { return @{ LoggedIn = $false; Subscription = $null } }
    try {
        $account = az account show 2>&1 | ConvertFrom-Json
        return @{ LoggedIn = $true; Subscription = $account.name }
    } catch {
        return @{ LoggedIn = $false; Subscription = $null }
    }
}

function Get-ProviderStatus {
    param([string]$ProviderNamespace)
    try {
        $result = az provider show --namespace $ProviderNamespace 2>&1 | ConvertFrom-Json
        return $result.registrationState
    } catch { return "Unknown" }
}

function Test-TerraformInitialized {
    $tfDir = Join-Path $TerraformDir ".terraform"
    return (Test-Path $tfDir -PathType Container)
}

# --- Pre-Flight Checks ----------------------------------------------------

function Invoke-PreFlightChecks {
    Write-Host ""
    Write-Host "=======================================================" -ForegroundColor Cyan
    Write-Host "  W365Claw Build Prerequisites -- Pre-Flight Check" -ForegroundColor Cyan
    Write-Host "=======================================================" -ForegroundColor Cyan

    $results = [ordered]@{}

    if (Test-IsWindowsDesktop) {
        $results["OS"] = @{ Status = $true; Detail = "Windows Desktop (x64)" }
    } else {
        $results["OS"] = @{ Status = $false; Detail = "Not Windows 10/11 Desktop" }
    }

    if (Test-IsAdmin) {
        $results["Administrator"] = @{ Status = $true; Detail = "Running elevated" }
    } else {
        $results["Administrator"] = @{ Status = $false; Detail = "NOT ELEVATED" }
    }

    # Check each tool...
    $tfVer = Get-TerraformVersion
    if ($null -eq $tfVer) {
        $results["Terraform"] = @{ Status = $false; Detail = "MISSING"; NeedsInstall = $true }
    } elseif ($tfVer -lt $MinVersions.Terraform) {
        $results["Terraform"] = @{ Status = $false; Detail = "$tfVer (need >= $($MinVersions.Terraform))"; NeedsInstall = $true }
    } else {
        $results["Terraform"] = @{ Status = $true; Detail = "$tfVer" }
    }

    # ... (Azure CLI, Git, Az Module, Azure Login, Resource Providers, terraform init)
    # Full implementation checks all prerequisites and prints a summary table

    foreach ($key in $results.Keys) {
        $r = $results[$key]
        $icon = if ($r.Status) { "✅" } else { "❌" }
        $paddedKey = $key.PadRight(20)
        if ($r.Status) {
            Write-Host "  $icon $paddedKey $($r.Detail)" -ForegroundColor Green
        } else {
            Write-Host "  $icon $paddedKey $($r.Detail)" -ForegroundColor Red
        }
    }

    return $results
}

# --- Installation Phase ---------------------------------------------------

function Install-MissingPrerequisites {
    param([System.Collections.Specialized.OrderedDictionary]$Results)

    $hasWinget = Test-WingetAvailable

    if ($Results["Terraform"].NeedsInstall) {
        if (Confirm-Action "Install Terraform via winget?") {
            if ($hasWinget) {
                winget install Hashicorp.Terraform --silent --accept-package-agreements --accept-source-agreements
            }
            Update-SessionPath
        }
    }

    # ... (Azure CLI, Git, Az Module, Azure Login, Resource Provider registration, terraform init)
    # Each missing prerequisite is installed with confirmation (or automatically with -Force)
}

# --- Main Execution -------------------------------------------------------

$TerraformDir = (Resolve-Path $TerraformDir -ErrorAction SilentlyContinue).Path
if (-not $TerraformDir) { $TerraformDir = Join-Path $PSScriptRoot "..\terraform" }

$results = Invoke-PreFlightChecks

if (-not $results["Administrator"].Status) {
    Write-Host "ERROR: This script must run as Administrator." -ForegroundColor Red
    exit 1
}

$failures = $results.Keys | Where-Object { -not $results[$_].Status }
if (-not $failures) {
    Write-Host "All prerequisites met. Ready to build! 🚀" -ForegroundColor Green
    exit 0
}

$didInstall = Install-MissingPrerequisites $results

if ($didInstall) {
    Update-SessionPath
    $finalResults = Invoke-PreFlightChecks
    $finalFailures = $finalResults.Keys | Where-Object { -not $finalResults[$_].Status }
    if (-not $finalFailures) {
        Write-Host "All prerequisites met. Ready to build! 🚀" -ForegroundColor Green
        exit 0
    } else {
        Write-Host "Some prerequisites could not be resolved." -ForegroundColor Red
        exit 1
    }
}
```

> **💡 Tip:** The full script is available in the W365Claw repository at `scripts/Initialize-BuildWorkstation.ps1`. The version above is abbreviated for readability; the complete implementation includes Azure CLI, Git, Az Module checks, Azure login with subscription selection, resource provider registration with polling, and `terraform init`.

### Initialize-TerraformVars.ps1

Manually populating `terraform.tfvars` with 30+ variables (subscription IDs, software versions, SHA256 checksums, source image versions) is tedious and error-prone. `Initialize-TerraformVars.ps1` automates this by querying live APIs, detecting the current Azure context, and prompting for overrides.

```powershell
<#
.SYNOPSIS
    Interactively populates terraform/terraform.tfvars for the W365Claw project.

.DESCRIPTION
    Auto-detects latest software versions, SHA256 checksums, Azure subscription,
    and source image versions. Prompts for all values with sensible defaults.
    Parses existing terraform.tfvars for idempotent re-runs.

.PARAMETER Force
    Skip confirmation prompts; use detected/existing defaults for all values.

.PARAMETER TerraformDir
    Path to the terraform/ directory. Defaults to ..\terraform relative to this script.
#>

[CmdletBinding()]
param(
    [switch]$Force,
    [string]$TerraformDir = (Join-Path $PSScriptRoot "..\terraform")
)
```

#### What It Does

The script proceeds through six phases:

1. **Parse existing `terraform.tfvars`**: If a file already exists, all current values become the defaults. This makes re-runs idempotent: run it again after bumping one version and everything else stays the same. The parser handles quoted strings, booleans, integers, and the `tags = { }` map block.

2. **Check Azure CLI login**: Detects whether `az` is installed and the user is authenticated. If logged in, auto-populates `subscription_id` from the active subscription. If not, offers to run `az login` interactively (skipped with `-Force`).

3. **Auto-detect latest software versions**: Queries live sources for the newest releases:

   | Package | Detection Source |
   |---------|-----------------|
   | Node.js | `nodejs.org/dist/index.json` (latest v24.x LTS) |
   | Python | `endoflife.date/api/python.json` (latest 3.x) |
   | Git | GitHub Releases API (`git-for-windows/git`) |
   | PowerShell 7 | GitHub Releases API (`PowerShell/PowerShell`) |
   | Azure CLI | GitHub Releases API (`Azure/azure-cli`) |
   | OpenClaw | `npm view openclaw version` |
   | Claude Code | `npm view @anthropic-ai/claude-code version` |
   | OpenSpec | `npm view @fission-ai/openspec version` |
   | Codex CLI | `npm view @openai/codex version` |

4. **Auto-detect SHA256 checksums**: Fetches official checksum files from release pages:
   - Node.js: `SHASUMS256.txt` from the dist URL
   - Python: `.sha256` sidecar file from `python.org/ftp`
   - PowerShell, Git, Azure CLI: `hashes.sha256` or checksums assets from GitHub Releases

5. **Auto-detect source image version**: Queries `az vm image list` for the latest Windows 11 Enterprise image matching the configured SKU and location.

6. **Interactive prompts**: For every variable, the script presents the auto-detected value (or existing value) as the default. The operator can accept with Enter or override. Validation enforces:
   - Alphanumeric-only gallery names
   - `Major.Minor.Patch` format for image versions
   - Boolean (`true`/`false`) for flags
   - Integer values where required
   - Tags: prompts for each existing tag, then offers to add more

#### Summary and Confirmation

Before writing, the script displays a formatted summary table of all values:

```text
=======================================================
  Summary -- terraform.tfvars
=======================================================

  subscription_id       = 12345678-abcd-1234-efgh-123456789012
  location              = eastus2
  resource_group_name   = rg-w365-images
  gallery_name          = acgW365Dev
  ...
  node_version          = v24.13.1
  node_sha256           = abc123...
  ...

  Tags:
    workload    = Windows365
    purpose     = DeveloperImages
    managed_by  = PlatformEngineering
    iac         = Terraform
    cost_center = Engineering

Write terraform.tfvars? [Y/n]
```

The `-Force` flag skips all prompts and writes immediately, which is useful for CI pipelines or scripted provisioning.

#### Usage

```powershell
# Interactive -- prompts for each value with auto-detected defaults
.\scripts\Initialize-TerraformVars.ps1

# Non-interactive -- auto-detect everything and write immediately
.\scripts\Initialize-TerraformVars.ps1 -Force

# Custom terraform directory
.\scripts\Initialize-TerraformVars.ps1 -TerraformDir C:\my-project\terraform
```

#### Design Decisions

**Why auto-detect versions?** Manually looking up the latest Node.js LTS, checking GitHub releases for PowerShell, and computing SHA256 hashes is the most common source of copy-paste errors in `terraform.tfvars`. Auto-detection eliminates this entirely while still allowing manual overrides.

**Why parse existing tfvars?** Without this, every run would require re-entering all 30+ values. By parsing the existing file, the script becomes a "bump and go" tool: change one version, accept the rest.

**Why `-Force`?** In CI/CD or automated provisioning, interactive prompts aren't viable. `-Force` makes the script pipeline-friendly: detect everything automatically, skip confirmations, write the file.

**PowerShell 5.1 compatibility.** The script avoids `&&` (PS7-only), ternary operators, and other PS7-specific syntax. It runs on both Windows PowerShell 5.1 and PowerShell 7+, matching the `Initialize-BuildWorkstation.ps1` style.

> **💡 Tip:** The full script (~750 lines) is available in the W365Claw repository at `scripts/Initialize-TerraformVars.ps1`. Run it once before your first `terraform plan` to populate all version pins and checksums automatically.

### Teardown-BuildResources.ps1

```powershell
<#
.SYNOPSIS
    Removes AIB build resources while preserving the gallery and image versions.

.DESCRIPTION
    Targeted teardown for W365Claw. Removes the AIB image template and managed
    identity via Terraform, while leaving the gallery and image definition intact
    (these have prevent_destroy = true).

.PARAMETER TerraformDir
    Path to the terraform/ directory.

.PARAMETER Force
    Skip confirmation prompts.
#>

[CmdletBinding()]
param(
    [switch]$Force,
    [string]$TerraformDir = (Join-Path $PSScriptRoot "..\terraform")
)

$ErrorActionPreference = "Stop"
$TerraformDir = (Resolve-Path $TerraformDir -ErrorAction Stop).Path

Write-Host ""
Write-Host "=======================================================" -ForegroundColor Cyan
Write-Host "  W365Claw -- Targeted Build Resource Teardown" -ForegroundColor Cyan
Write-Host "=======================================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "This will remove:" -ForegroundColor Yellow
Write-Host "  -- AIB image template (azapi_resource.image_template)" -ForegroundColor Yellow
Write-Host "  -- Build action (azapi_resource_action.run_build)" -ForegroundColor Yellow
Write-Host "  -- Build timestamp (time_static.build_time)" -ForegroundColor Yellow
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

    if ($LASTEXITCODE -ne 0) {
        Write-Error "Terraform targeted destroy failed (exit code $LASTEXITCODE)"
        exit 1
    }

    Write-Host ""
    Write-Host "✅ Build resources removed. Gallery and images preserved." -ForegroundColor Green
} finally {
    Pop-Location
}
```

---

