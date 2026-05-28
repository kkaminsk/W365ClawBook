## Chapter 8: Phase 1 -- Core Runtimes

Phase 1 installs Node.js, Python, and PowerShell 7, the runtime foundation for everything else.

> **💡 Note:** If your development teams also require .NET runtimes, these can be added to the build pipeline using a similar pattern (silent MSI install with `ALLUSERS=1`). .NET is out of scope for this book, but the same principles apply: pin the version, verify the checksum, refresh the session PATH.

### Node.js: The Primary Execution Engine

Both OpenClaw and Claude Code are Node.js applications. OpenClaw mandates **Node.js 22 or higher**. The installation uses the MSI installer with `ALLUSERS=1` to ensure machine-wide placement in `C:\Program Files\nodejs`.

```powershell
$NodeVersion = "${var.node_version}"
$NodeMsiUrl = "https://nodejs.org/dist/$NodeVersion/node-$NodeVersion-x64.msi"
$NodeInstaller = "$env:TEMP\node-$NodeVersion-x64.msi"

Write-Host "=== Installing Node.js $NodeVersion ==="
Get-InstallerWithRetry -Uri $NodeMsiUrl -OutFile $NodeInstaller  # defined below in "Download Retry Logic"

$proc = Start-Process -FilePath "msiexec.exe" `
    -ArgumentList "/i `"$NodeInstaller`" /qn /norestart ALLUSERS=1 ADDLOCAL=ALL" `
    -Wait -PassThru
if ($proc.ExitCode -ne 0) {
    Write-Error "Node.js installation failed ($($proc.ExitCode))"
    exit 1
}
```

The critical MSI properties:

| Property / Switch | Value | Purpose |
|---|---|---|
| `/i` | `<path>` | Install mode |
| `/qn` | -- | Quiet No UI -- suppresses all dialogs |
| `/norestart` | -- | Prevents automatic reboot during build |
| `ALLUSERS` | `1` | Forces machine-wide installation to `C:\Program Files` |
| `ADDLOCAL` | `ALL` | Installs all features (npm, runtime, PATH registration) |

### The Environment Variable Propagation Problem

This is the single most important gotcha in the entire build pipeline.

When the Node.js MSI updates the System PATH in the registry (`HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment`), the change is **not** reflected in the currently running PowerShell session. If the script tries to run `npm install` immediately, it fails with `CommandNotFoundException` because `$env:Path` still reflects the pre-installation state.

The solution is to programmatically refresh the session environment:

```powershell
function Update-SessionEnvironment {
    $machinePath = [Environment]::GetEnvironmentVariable("Path", "Machine")
    $userPath = [Environment]::GetEnvironmentVariable("Path", "User")
    $env:Path = "$machinePath;$userPath"
    Write-Host "[PATH] Session environment refreshed"
}
```

This function reads the Machine and User PATH values directly from the registry and reconstructs `$env:Path`. It must be called after every MSI/EXE installation that modifies the system PATH.

> **⚠️ Warning:** Relying on a system reboot to propagate PATH changes is a common but fragile approach. Reboots within AIB builds are complex to orchestrate and add significant time. The `Update-SessionEnvironment` function gives you immediate access to newly installed binaries in the same script block.

### Python

Python supports native module compilation (via `node-gyp`) and is frequently required by OpenClaw's MCP server ecosystem. The installer uses `InstallAllUsers=1` and `PrependPath=1`:

```powershell
$PythonVersion = "${var.python_version}"
$PythonUrl = "https://www.python.org/ftp/python/$PythonVersion/python-$PythonVersion-amd64.exe"
$PythonInstaller = "$env:TEMP\python-$PythonVersion-amd64.exe"

Write-Host "=== Installing Python $PythonVersion ==="
Get-InstallerWithRetry -Uri $PythonUrl -OutFile $PythonInstaller

$proc = Start-Process -FilePath $PythonInstaller `
    -ArgumentList "/quiet InstallAllUsers=1 PrependPath=1 Include_pip=1 Include_test=0 Include_launcher=1" `
    -Wait -PassThru
if ($proc.ExitCode -ne 0) {
    Write-Error "Python installation failed ($($proc.ExitCode))"
    exit 1
}
Update-SessionEnvironment
```

Key flags:

| Flag | Purpose |
|------|---------|
| `InstallAllUsers=1` | Install to `C:\Program Files\Python314` instead of `%LocalAppData%` |
| `PrependPath=1` | Add Python and Scripts directories to system PATH |
| `Include_pip=1` | Include pip package manager |
| `Include_test=0` | Exclude test suite to reduce image size |
| `Include_launcher=1` | Include the `py` launcher |

> **💡 Tip:** Python 3.14.3 is the current stable release as of this writing. Python patch versions are released regularly, so verify the latest 3.14.x patch version at [python.org/downloads](https://www.python.org/downloads/) before building your image. The installer URL structure is consistent across patch releases, so updating is a single variable change in `terraform.tfvars`.

### PowerShell 7

```powershell
$PwshVersion = "${var.pwsh_version}"
$PwshUrl = "https://github.com/PowerShell/PowerShell/releases/download/v$PwshVersion/PowerShell-$PwshVersion-win-x64.msi"
$PwshInstaller = "$env:TEMP\PowerShell-$PwshVersion-win-x64.msi"

Get-InstallerWithRetry -Uri $PwshUrl -OutFile $PwshInstaller

$proc = Start-Process -FilePath "msiexec.exe" `
    -ArgumentList "/i `"$PwshInstaller`" /qn /norestart ADD_EXPLORER_CONTEXT_MENU_OPENPOWERSHELL=1 ADD_FILE_CONTEXT_MENU_RUNPOWERSHELL=1 ENABLE_PSREMOTING=0 REGISTER_MANIFEST=1 USE_MU=0 ENABLE_MU=0 ADD_PATH=1" `
    -Wait -PassThru
if ($proc.ExitCode -ne 0) {
    Write-Error "PowerShell 7 installation failed ($($proc.ExitCode))"
    exit 1
}
```

Key MSI properties:

| MSI Property | Value | Reason |
|---|---|---|
| `ENABLE_PSREMOTING` | `0` | Disables WinRM-based remoting; not needed in image build context |
| `USE_MU` / `ENABLE_MU` | `0` | Disables Microsoft Update integration; updates are managed via image rebuild |
| `ADD_PATH` | `1` | Adds PowerShell 7 to system PATH |

> **💡 Tip:** The PowerShell 7 download URL must use a specific release path (`/download/v7.4.13/`), not the `/latest/` redirect. Using `/latest/` with a versioned filename will 404 when a newer release ships. This was identified as a High severity finding in the Terraform audit.

### The Restart

After Phase 1, the AIB template (AIB template — the Azure VM Image Builder Terraform resource) includes a `WindowsRestart` customizer to ensure all PATH changes and system state updates are fully propagated:

```json
{
  "type": "WindowsRestart",
  "restartCommand": "shutdown /r /f /t 5 /c \"Restart after runtime installation\"",
  "restartTimeout": "10m",
  "restartCheckCommand": "powershell -command \"node --version; python --version\""
}
```

The `restartCheckCommand` verifies that Node.js and Python are functional after the restart, ensuring Phase 2 has a stable foundation.

### Download Retry Logic

All installer downloads use a retry function to handle transient network failures:

```powershell
function Get-InstallerWithRetry {
    param([string]$Uri, [string]$OutFile, [int]$MaxRetries = 3)
    for ($i = 1; $i -le $MaxRetries; $i++) {
        try {
            Write-Host "[DOWNLOAD] Attempt $i of $MaxRetries : $Uri"
            Invoke-WebRequest -Uri $Uri -OutFile $OutFile -UseBasicParsing
            return
        } catch {
            if ($i -eq $MaxRetries) { throw }
            Write-Host "[DOWNLOAD] Attempt $i failed, retrying in 10 seconds..."
            Start-Sleep -Seconds 10
        }
    }
}
```

---

