## Chapter 25: Agent Updates Without Reprovisioning

Since OpenClaw and Claude Code are installed post-provisioning via npm, an Intune user-context script can update them on running Cloud PCs, with no reprovisioning required:

```powershell
# Update all AI agents and tooling to latest approved versions
npm update -g openclaw
npm update -g @anthropic-ai/claude-code
npm update -g @openai/codex
npm update -g @fission-ai/openspec
npm update -g @perplexity-ai/mcp-server
```

For controlled updates with specific versions:

```powershell
npm install -g openclaw@2026.3.1
npm install -g @anthropic-ai/claude-code@2.2.0
```

Deploy this script via Intune as a platform script targeting the Cloud PC device group. This gives you the ability to push agent updates within hours, compared to the multi-day cycle of rebuilding an image, importing it, and reprovisioning.

> **💡 Tip:** This is one of the strongest arguments for installing agents via npm rather than a traditional installer: you get a zero-downtime update path that doesn't require admin portal access or user disruption.

### Delivering Agent Updates via Intune

For organizations that manage software delivery centrally, Intune provides an alternative update path that does not require developers to run update commands manually.

In the **user-installed model**, all npm-based tooling — AI agents, OpenSpec, and MCP servers — is delivered **after** the Cloud PC is provisioned via **Intune Win32 app packages** targeted per-user. This provides install state detection, retries, controlled versioning, and a clear dependency ordering.

**Delivery tiers:**

| Package | Intune Assignment | Rationale |
|---|---|---|
| OpenClaw | **Required** (per-user) | Core agent; must be present on every developer Cloud PC |
| Claude Code | **Required** (per-user) | Core agent; must be present on every developer Cloud PC |
| Codex CLI | **Required** (per-user) | Core agent; must be present on every developer Cloud PC |
| OpenSpec | **Required** (per-user) | Development workflow tooling; needed alongside agents |
| MCP servers (Perplexity, Jira, etc.) | **Available** (per-user) | Optional; developers install from Company Portal on demand |

**Why this ordering matters:** MCP servers depend on the AI agents being present — they are invoked by agents at runtime and their configuration references agent workspace paths. By making agents and OpenSpec **required** and MCP servers **available**, Intune ensures the foundation is in place before optional integrations are added. Developers choose which MCP servers they need from the Company Portal, avoiding unnecessary installs and reducing the attack surface.

**Operational guidance:**

- Target the Cloud PC device group or an agent account group.
- Run installs in **user context** so npm global packages land in the correct user profile.
- Keep the **image build clean**: only runtimes (Node.js, Python) and baseline tooling in the image.
- Pin versions in the Intune payload to keep developer environments consistent.

**Detection method guidance (Win32):**

- Prefer **version-based detection** using the CLI output:
  - `openclaw --version`
  - `claude --version`
  - `codex --version`
  - `openspec --version`
- Return **non-zero** when the version does not match the pinned version to trigger remediation.
- Avoid file-path detection alone; npm global paths can vary by user profile.
- If you must use file-based detection, use the **npm global bin path** from the user context and validate the executable exists and runs.
- Keep detection scripts **idempotent** and **fast** to avoid repeated install loops.

**Example detection script (PowerShell, user context):**

```powershell
$expected = @{
    openclaw = "2026.2.14"
    claude   = "2.1.42"
    codex    = "0.101.0"
    openspec = "0.9.1"
}

function Get-CmdVersion([string]$cmd) {
    $v = & $cmd --version 2>$null
    if (-not $v) { return $null }
    return ($v -replace '[^0-9\.]','').Trim()
}

foreach ($k in $expected.Keys) {
    $ver = Get-CmdVersion $k
    if (-not $ver -or $ver -ne $expected[$k]) { exit 1 }
}

exit 0
```

> **Note:** Machine-level environment variables set by Intune Platform Scripts are visible to all users on a given Cloud PC. For secrets that belong to a specific developer rather than the machine — such as personal API keys — use the user-context delivery pattern described in Chapter 23.

### Updating Skills

Curated skills can be updated on running Cloud PCs without reprovisioning. Deploy a user-context Intune script that pulls from your internal skills repository:

```powershell
# Runs in user context via Intune
$skillsDest = "$env:USERPROFILE\.agents\skills"
$tempDir = "$env:TEMP\skill-update-$(Get-Date -Format 'yyyyMMddHHmmss')"

git clone --depth 1 "https://your-internal-repo/agent-skills/releases/latest/download/skills.zip" $tempDir 2>&1 | Out-Null
if ($LASTEXITCODE -eq 0) {
    Copy-Item -Path "$tempDir\*" -Destination $skillsDest -Recurse -Force
    Remove-Item -Path $tempDir -Recurse -Force
}
```

### Git Credential Management for Private Repos

If your skills repository or development repos are private (GitHub, Azure DevOps), the Cloud PC needs Git credentials. The recommended approach is **Personal Access Tokens (PATs)** stored in Windows Credential Manager via Git Credential Manager (GCM), which is included with Git for Windows:

1. The developer generates a PAT in GitHub or Azure DevOps with the minimum required scopes (typically `repo` read-only for skills, `repo` read-write for development).
2. On first `git clone` or `git pull`, GCM prompts for credentials and stores them securely in Windows Credential Manager.
3. Subsequent operations use the cached credential automatically.

For automated skill updates via Intune scripts, the PAT can be passed inline (though this is less secure):

```powershell
git clone "https://<PAT>@your-internal-repo/org/approved-agent-skills.git" $tempDir
```

For Azure DevOps, GCM supports Azure AD-backed authentication via interactive browser flows. For unattended agent use — such as Intune scripts — a PAT with the minimum required scope (typically `Code: Read`) is still required; GCM's interactive flow is not available in a headless context.

> **⚠️ Warning:** Never bake PATs into the image. They are user-specific, time-limited credentials that must be managed per-developer.

### Updating MCP Server Configuration

MCP server configuration changes (adding new servers, rotating API keys, adjusting endpoints) can be pushed via an Intune script that overwrites the `mcporter.json` file:

```powershell
# Runs in user context via Intune
$mcpConfigPath = "$env:USERPROFILE\.openclaw\workspace\config\mcporter.json"
$templatePath = "\\bighatgroup.com\shares\agent-config\mcporter.json"

if (Test-Path $templatePath) {
    Copy-Item -Path $templatePath -Destination $mcpConfigPath -Force
}
```

For MCP servers that require API keys delivered as environment variables, use Intune Settings Catalog to set machine-level variables (e.g., `PERPLEXITY_API_KEY`, `JIRA_API_TOKEN`). The MCP server configuration references these variables, and they're available to any process running on the Cloud PC.

---

