# Part III: The Build Pipeline

---

## Chapter 7: Preparing the Build Workstation

Part III covers the three build phases that produce the Windows 365 custom image, beginning with preparing the environment that drives the Azure VM Image Builder pipeline.

### Prerequisites

Before running `terraform apply`, your workstation needs:

The build workstation is a trusted Azure VM or operator-managed jump server — not a personal laptop — with outbound internet access for installer downloads.

| Tool | Minimum Version | Installation Method | Purpose |
|------|----------------|-------------------|---------|
| Terraform CLI | >= 1.5.0 | `winget install Hashicorp.Terraform` | Infrastructure as code |
| Azure CLI | >= 2.60 | `winget install Microsoft.AzureCLI` | Authentication for Terraform |
| Git | >= 2.40 | `winget install Git.Git` | Repository management |
| Az PowerShell Module | >= 12.0 | `Install-Module -Name Az` | Post-build verification |

### Azure Resource Provider Registration

Four resource providers must be registered on your subscription:

> **⚠️ Note:** `Microsoft.VirtualMachineImages` is the one that trips people up. It's not registered by default on most subscriptions, and the error message when it's missing is not always obvious.

| Resource Provider | Purpose | Default State |
|-------------------|---------|--------------|
| `Microsoft.Compute` | Gallery, image definitions, image versions | Usually registered |
| `Microsoft.VirtualMachineImages` | Azure VM Image Builder | **Often NOT registered** |
| `Microsoft.Network` | Transient networking for build VM | Usually registered |
| `Microsoft.ManagedIdentity` | User-assigned managed identity | Usually registered |

### The Initialize-BuildWorkstation.ps1 Script

The repository includes a comprehensive prerequisite installer script that automates everything:

```powershell
# Interactive -- prompts before each installation
.\scripts\Initialize-BuildWorkstation.ps1

# Non-interactive -- installs everything without prompting
.\scripts\Initialize-BuildWorkstation.ps1 -Force
```

The script performs three phases:

**Phase 1: Pre-Flight Checks**: Verifies OS, admin privileges, and each prerequisite. Outputs a summary table:

```text
=======================================================
  W365Claw Build Prerequisites -- Pre-Flight Check
=======================================================

  ✅ OS                Windows Desktop (x64)
  ✅ Administrator      Running elevated
  ✅ Terraform          1.9.5 (>= 1.5.0)
  ✅ Azure CLI          2.83.0 (>= 2.60)
  ✅ Git                2.53.0 (>= 2.40)
  ❌ Az Module          MISSING
  ✅ Azure Login        Subscription: rg-w365-images
  ✅ RP: Compute        Registered
  ❌ RP: VMImages       NotRegistered
  ✅ RP: Network        Registered
  ✅ RP: ManagedId      Registered
  ✅ terraform init     Initialized
```

**Phase 2: Installation**: For each missing prerequisite, prompts the operator and installs via `winget` (preferred), ZIP download (fallback for Terraform), or `Install-Module` (for Az).

**Phase 3: Verification**: Re-runs all pre-flight checks to confirm everything passes.

The script is fully idempotent; running it on an already-configured machine is a no-op. It never downgrades existing software and never re-registers already-registered providers.

Key implementation details:

- **PATH Propagation:** After each `winget install`, the script calls `Update-SessionPath` to refresh the PowerShell session's PATH from the registry.
- **Subscription Selection:** If multiple Azure subscriptions are detected after `az login`, the script prompts for selection with bounds validation.
- **Resource Provider Polling:** Registration is asynchronous. The script polls every 10 seconds with a 5-minute timeout.

> **💡 Tip:** The script works in both Windows PowerShell 5.1 and PowerShell 7+. It uses `winget` as the preferred package manager and falls back to direct downloads if `winget` is unavailable.

---

