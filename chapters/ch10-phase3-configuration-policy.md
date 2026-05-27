## Chapter 10: Phase 3 -- Configuration and Policy

Phase 3 is where the image transforms from "tools installed" to "enterprise-ready." This phase configures Claude Code enterprise policy, creates the OpenClaw configuration template, registers Active Setup for first-login hydration, sets Teams optimisation prerequisites, generates the SBOM, and cleans up the image.

> **💡 Note:** In earlier versions of this solution, a separate build phase installed OpenSpec and MCP server npm packages into the image. These have been moved to **post-provisioning** and are delivered via Intune:
>
> - **AI agents** (OpenClaw, Claude Code, Codex CLI) and **OpenSpec** are deployed as **required** Win32 apps targeted per-user, ensuring they are present on every developer Cloud PC automatically.
> - **MCP servers** are deployed as **available** Win32 apps targeted per-user, allowing developers to install them on demand from the Intune Company Portal. This ensures agents are fully installed before MCP server configuration occurs.
>
> This keeps the image focused on runtimes, developer tools, and enterprise policy. See Chapter 22 and Chapter 25 for the post-provisioning delivery model.

### Claude Code Enterprise Managed Settings

The `managed-settings.json` file enforces enterprise policy at the machine level. Developers cannot override these settings:

```powershell
$claudeConfigDir = "C:\ProgramData\ClaudeCode"
New-Item -ItemType Directory -Path $claudeConfigDir -Force | Out-Null

$managedSettings = @{
    autoUpdatesChannel = "stable"
    permissions = @{
        defaultMode = "allowWithPermission"
    }
} | ConvertTo-Json -Depth 5

$managedSettingsPath = "$claudeConfigDir\managed-settings.json"
Set-Content -Path $managedSettingsPath -Value $managedSettings -Encoding UTF8
```

The default `defaultMode` is `"allowWithPermission"`, which allows Claude Code to execute commands but prompts the user for explicit approval on operations that require elevated permissions. This strikes a balance between developer productivity and security: routine operations proceed without friction, while sensitive actions still require confirmation. For teams that want maximum control, this can be changed to `"ask"` (requires approval for all operations) in the `managed-settings.json` template. See Chapter 28 for a deep dive on all available settings, including deny-listing high-risk commands.

> **💡 Tip:** The `"allowWithPermission"` default assumes your network segmentation (Chapter 30) and agent identity isolation (Chapter 27) are in place. If those controls are not yet deployed, consider starting with `"ask"` until your containment architecture is verified.

### OpenClaw Configuration Template

OpenClaw expects configuration in `~/.openclaw/openclaw.json`. Since `~` resolves to the Local System profile during the build, we create a **template** in ProgramData and copy it to the user's profile at first login:

```powershell
$openclawTemplateDir = "C:\ProgramData\OpenClaw"
New-Item -ItemType Directory -Path $openclawTemplateDir -Force | Out-Null

$openclawConfig = @{
    agent = @{
        model = "${var.openclaw_default_model}"
        defaults = @{
            workspace = "~/Documents/OpenClawWorkspace"
        }
    }
    gateway = @{
        mode = "local"
        port = ${var.openclaw_gateway_port}
    }
    channels = @{
        web = @{ enabled = $true }
    }
} | ConvertTo-Json -Depth 5

$templatePath = "$openclawTemplateDir\template-config.json"
Set-Content -Path $templatePath -Value $openclawConfig -Encoding UTF8
```

### Agent Skills (Curated)

OpenClaw and Claude Code both support **skills**, bundled instruction sets that teach the agent how to perform specific tasks (Terraform workflows, Jira integration, coding standards, etc.). Skills are directories containing a `SKILL.md` file and optional scripts, references, and assets.

During the image build, curated skills are pre-installed to a machine-wide location. At first login, they're copied into the user's profile.

```powershell
# -- Pre-seed curated agent skills --
$skillsSourceDir = "C:\ProgramData\OpenClaw\skills"
New-Item -ItemType Directory -Path $skillsSourceDir -Force | Out-Null

# Clone the organization's curated skills repository
# This repo has been vetted through the Cisco Skill Scanner (see Chapter 29)
Write-Host "=== Cloning curated agent skills ==="
git clone --depth 1 "https://github.com/bighatgroup/approved-agent-skills.git" "$skillsSourceDir\approved" 2>&1 | Write-Host
if ($LASTEXITCODE -ne 0) {
    Write-Warning "Skills clone failed -- skills will need to be installed post-provisioning"
}

# Alternatively, copy skills from a build artifact or Azure Blob
# azcopy copy "https://stbuildartifacts.blob.core.windows.net/skills/*" "$skillsSourceDir\" --recursive
```

**Why bake skills into the image?** Skills are static files (markdown, scripts, JSON). Unlike API keys, they don't contain secrets. Pre-installing them means developers have a curated toolkit from the moment they sign in, without needing to discover and install skills individually.

**What NOT to include:** Never include skills from the public ClawHub marketplace in the image. All skills must go through your internal vetting pipeline first (see Chapter 29).

> **⚠️ Warning:** Skills can contain executable scripts in their `scripts/` subdirectory. Every skill in the curated repository must be reviewed for data exfiltration, prompt injection, and obfuscated payloads before inclusion in the image.

### MCP Servers

The Model Context Protocol (MCP) enables Claude Code and OpenClaw to interact with external services such as Jira, Microsoft Docs, Perplexity, and internal APIs. MCP servers are either npm packages (stdio transport) or HTTP endpoints (SSE transport).

**Stdio MCP servers** (npm packages) are delivered **post-provisioning** as **available** Win32 apps in Intune, targeted per-user. Developers install them on demand from the Intune Company Portal. Deploying MCP servers as available (rather than required) ensures the AI agents and OpenSpec are fully installed first — MCP servers depend on the agents being present for configuration and runtime invocation.

**HTTP/SSE MCP servers** (remote endpoints) don't require binary installation; they're configured via the MCP configuration file.

**MCP configuration** is pre-seeded as a template in ProgramData and hydrated to the user profile at first login (alongside the OpenClaw config):

```powershell
$mcpConfigDir = "C:\ProgramData\OpenClaw\mcp"
New-Item -ItemType Directory -Path $mcpConfigDir -Force | Out-Null

# mcporter configuration template
# API keys are placeholder values -- replaced post-provisioning via Intune
$mcpConfig = @{
    servers = @{
        "perplexity" = @{
            transport = "stdio"
            command   = "npx"
            args      = @("-y", "@perplexity-ai/mcp-server")
            env       = @{
                PERPLEXITY_API_KEY = "__PERPLEXITY_API_KEY__"
            }
        }
        "microsoft-docs" = @{
            transport = "http"
            url       = "https://learn.microsoft.com/api/mcp"
        }
    }
} | ConvertTo-Json -Depth 5

Set-Content -Path "$mcpConfigDir\mcporter.json" -Value $mcpConfig -Encoding UTF8
```

> **💡 Tip:** MCP server configurations often contain API keys. Use placeholder values (e.g., `__PERPLEXITY_API_KEY__`) in the image template and replace them post-provisioning via Intune environment variables or a user-context script that reads from Azure Key Vault. The `microsoft-docs` MCP server is a notable exception; it's a public API that requires no authentication.

### Active Setup: First-Login Configuration Hydration

Active Setup is a Windows mechanism that executes a command once per user at their first login. We use it to copy the OpenClaw configuration template into the user's profile:

```powershell
$hydrationScript = @'
$openclawDir = "$env:USERPROFILE\.openclaw"
$configFile = "$openclawDir\openclaw.json"
$templateFile = "C:\ProgramData\OpenClaw\template-config.json"

if (Test-Path $templateFile) {
    New-Item -ItemType Directory -Path $openclawDir -Force | Out-Null
    $workspaceDir = "$env:USERPROFILE\Documents\OpenClawWorkspace"
    New-Item -ItemType Directory -Path $workspaceDir -Force | Out-Null
    Copy-Item -Path $templateFile -Destination $configFile -Force
}

# Hydrate curated agent skills
$skillsSource = "C:\ProgramData\OpenClaw\skills"
$skillsDest = "$env:USERPROFILE\.agents\skills"
if (Test-Path $skillsSource) {
    New-Item -ItemType Directory -Path $skillsDest -Force | Out-Null
    Copy-Item -Path "$skillsSource\*" -Destination $skillsDest -Recurse -Force
}

# Hydrate MCP server configuration
$mcpSource = "C:\ProgramData\OpenClaw\mcp\mcporter.json"
$mcpDest = "$openclawDir\workspace\config"
if (Test-Path $mcpSource) {
    New-Item -ItemType Directory -Path $mcpDest -Force | Out-Null
    Copy-Item -Path $mcpSource -Destination "$mcpDest\mcporter.json" -Force
}
'@

$hydrationScriptPath = "$openclawTemplateDir\hydrate-config.ps1"
Set-Content -Path $hydrationScriptPath -Value $hydrationScript -Encoding UTF8

# Register via Active Setup
$activeSetupKey = "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\OpenClaw-ConfigHydration"
New-Item -Path $activeSetupKey -Force | Out-Null
Set-ItemProperty -Path $activeSetupKey -Name "(Default)" -Value "OpenClaw Configuration Hydration"
Set-ItemProperty -Path $activeSetupKey -Name "StubPath" -Value "powershell.exe -ExecutionPolicy Bypass -WindowStyle Hidden -File `"$hydrationScriptPath`""
Set-ItemProperty -Path $activeSetupKey -Name "Version" -Value "1,0,0,0"
```

The hydration script overwrites the OpenClaw configuration, skills, and MCP configuration with `-Force` on every execution. This ensures enterprise-mandated settings are consistently applied across all Cloud PCs. The Active Setup `Version` property controls re-execution: if you bump the version in a future image, Active Setup will re-run for users who have already logged in, refreshing their configuration, skills, and MCP settings to match the latest enterprise template. Developer customizations to `openclaw.json` will be replaced; developers who need persistent customizations should manage them outside the template path or use Intune-delivered overrides.

> **💡 Tip:** Active Setup runs in the user's security context at login, which is exactly what we need. The command executes before the desktop fully loads, so the OpenClaw configuration is in place by the time the developer opens a terminal.

### Teams VDI Optimisation

If you're building from a marketplace base rather than a Windows 365 gallery image, include the Teams optimisation prerequisite:

```powershell
$teamsRegPath = "HKLM:\SOFTWARE\Microsoft\Teams"
if (-not (Test-Path $teamsRegPath)) {
    New-Item -Path $teamsRegPath -Force | Out-Null
}
Set-ItemProperty -Path $teamsRegPath -Name "IsWVDEnvironment" -Value 1 -Type DWord -Force
```

> **⚠️ Warning:** Do **not** install the Teams desktop app itself. Deliver Microsoft 365 Apps (without Teams) via Intune post-provisioning. Teams should use the new Teams app delivered through its own deployment channel with media optimisation.

### Image Cleanup and DISM

The final step reduces image size by cleaning up temporary files and the Windows component store:

```powershell
# Clean temp files
Remove-Item -Path "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue

# Clean Windows Update download cache
Stop-Service -Name wuauserv -Force -ErrorAction SilentlyContinue
Remove-Item -Path "C:\Windows\SoftwareDistribution\Download\*" -Recurse -Force -ErrorAction SilentlyContinue
Start-Service -Name wuauserv -ErrorAction SilentlyContinue

# Clean npm cache
npm cache clean --force 2>&1 | Out-Null

# DISM component store cleanup
Start-Process -FilePath "dism.exe" `
    -ArgumentList "/Online /Cleanup-Image /StartComponentCleanup /ResetBase" `
    -Wait -NoNewWindow
```

---

