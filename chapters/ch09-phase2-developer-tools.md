## Chapter 9: Phase 2 -- Developer Tools

Phase 2 installs Visual Studio Code, Git, GitHub Desktop, Azure CLI, and the GitHub Copilot extensions.

### Visual Studio Code: System Installer

VS Code **must** use the **System Installer**, not the User Installer. The User Installer places binaries in `%LocalAppData%`, which in the Local System context means `C:\Windows\System32\config\systemprofile`, invisible to the actual developer.

The Inno Setup installer includes a "Launch VS Code" task that causes the script to hang in a headless build. The `!runcode` merge task explicitly disables this:

```powershell
$VSCodeUrl = "https://update.code.visualstudio.com/latest/win32-x64-system/stable"
$VSCodeInstaller = "$env:TEMP\VSCodeSetup-x64.exe"
Get-InstallerWithRetry -Uri $VSCodeUrl -OutFile $VSCodeInstaller

$proc = Start-Process -FilePath $VSCodeInstaller `
    -ArgumentList "/VERYSILENT /NORESTART /MERGETASKS=`"!runcode,addcontextmenufiles,addcontextmenufolders,addtopath`"" `
    -Wait -PassThru
if ($proc.ExitCode -ne 0) {
    Write-Error "VS Code installation failed ($($proc.ExitCode))"
    exit 1
}
```

The critical argument is `/MERGETASKS="!runcode,addcontextmenufiles,addcontextmenufolders,addtopath"`:

| Task | Purpose |
|------|---------|
| `!runcode` | The `!` negates the task -- prevents VS Code from launching after install |
| `addcontextmenufiles` | Adds "Open with Code" to file context menus |
| `addcontextmenufolders` | Adds "Open with Code" to folder context menus |
| `addtopath` | Adds `code` CLI to system PATH |

> **⚠️ Warning:** VS Code is downloaded from the `/latest/` URL and is intentionally not version-pinned. This is a documented exception; Microsoft's auto-update redirector doesn't provide stable versioned URLs with published checksums. VS Code's own auto-update mechanism will supersede the installed version on first login anyway. See Chapter 13 for the full rationale.

### Git for Windows

```powershell
$GitVersion = "${var.git_version}"
$GitUrl = "https://github.com/git-for-windows/git/releases/download/v${GitVersion}.windows.1/Git-${GitVersion}-64-bit.exe"
$GitInstaller = "$env:TEMP\Git-${GitVersion}-64-bit.exe"
Get-InstallerWithRetry -Uri $GitUrl -OutFile $GitInstaller

$proc = Start-Process -FilePath $GitInstaller `
    -ArgumentList "/VERYSILENT /NORESTART /PathOption=Cmd /NoAutoCrlf /SetupType=default" `
    -Wait -PassThru
if ($proc.ExitCode -ne 0) {
    Write-Error "Git installation failed ($($proc.ExitCode))"
    exit 1
}
Update-SessionEnvironment
```

Key flags:

| Flag | Purpose |
|------|---------|
| `/VERYSILENT` | No UI -- headless install |
| `/PathOption=Cmd` | Add `git.exe` to system PATH |
| `/NoAutoCrlf` | Don't set `core.autocrlf` -- let developers choose |

### GitHub Desktop: Per-User Install Activation

GitHub Desktop presents a unique installation architecture that's worth understanding in detail. The standard `.exe` installer is designed for per-user installation (ClickOnce-style) into AppData. For enterprise/image deployment, GitHub provides a **Machine-Wide MSI Installer**.

It's vital to understand that this MSI does **not** install the application into Program Files. Instead, it installs a **provisioner** into `C:\Program Files (x86)\GitHub Desktop Deployment`. When a user logs in, the provisioner detects the login and activates (installs) the actual GitHub Desktop application into the user's `%LocalAppData%` folder via the per-user install activation mechanism.

This is exactly the behaviour you want for a Cloud PC image:

- The MSI runs during the image build as Local System ✅
- The actual application installs into the correct user profile at first login ✅
- Each user gets a clean, updateable copy without admin rights ✅

```powershell
$GHDesktopUrl = "https://central.github.com/deployments/desktop/desktop/latest/GitHubDesktopSetup-x64.msi"
$GHDesktopInstaller = "$env:TEMP\GitHubDesktop-x64.msi"
Get-InstallerWithRetry -Uri $GHDesktopUrl -OutFile $GHDesktopInstaller

$proc = Start-Process -FilePath "msiexec.exe" `
    -ArgumentList "/i `"$GHDesktopInstaller`" /qn /norestart ALLUSERS=1" `
    -Wait -PassThru
if ($proc.ExitCode -ne 0) {
    Write-Error "GitHub Desktop installation failed ($($proc.ExitCode))"
    exit 1
}
```

> **💡 Tip:** Like VS Code, GitHub Desktop is intentionally not version-pinned. GitHub's CDN always serves the latest version, and the auto-update mechanism supersedes the installed version immediately. This is a documented and accepted exception.

### Azure CLI

The Azure CLI is baked into the image rather than delivered post-provisioning because build-time and post-provisioning automation scripts require it, and waiting for per-user installation would block early automation steps.

```powershell
$AzCliVersion = "${var.azure_cli_version}"
$AzCliUrl = "https://azcliprod.blob.core.windows.net/msi/azure-cli-$AzCliVersion-x64.msi"
$AzCliInstaller = "$env:TEMP\azure-cli-$AzCliVersion-x64.msi"
Get-InstallerWithRetry -Uri $AzCliUrl -OutFile $AzCliInstaller

$proc = Start-Process -FilePath "msiexec.exe" `
    -ArgumentList "/i `"$AzCliInstaller`" /qn /norestart ALLUSERS=1" `
    -Wait -PassThru
if ($proc.ExitCode -ne 0) {
    Write-Error "Azure CLI installation failed ($($proc.ExitCode))"
    exit 1
}
Update-SessionEnvironment
```

### GitHub Copilot Extensions

With VS Code installed, the Copilot extensions can be installed machine-wide using the `code.cmd` CLI:

```powershell
$codeBin = "C:\Program Files\Microsoft VS Code\bin\code.cmd"
if (Test-Path $codeBin) {
    & $codeBin --install-extension GitHub.copilot --force 2>&1 | Write-Host
    & $codeBin --install-extension GitHub.copilot-chat --force 2>&1 | Write-Host
    Write-Host "[VERIFY] GitHub Copilot extensions installed"
} else {
    Write-Error "VS Code not found at expected path"
    exit 1
}
```

> **⚠️ Warning:** VS Code extension installation during the image build installs into the **default extensions directory**. When VS Code is installed using the System installer, extensions installed via `code.cmd --install-extension` are placed in the shared system-level extensions directory (`C:\Program Files\Microsoft VS Code\extensions` or its equivalent), which VS Code makes available to all users on the machine. This is the correct and expected behaviour for system-wide extension deployment. For user-specific or optional extensions that should not be pre-installed for all users, use post-provisioning delivery as discussed in Chapter 24.

---

