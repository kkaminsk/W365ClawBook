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

### Updating Skills

Curated skills can be updated on running Cloud PCs without reprovisioning. Deploy a user-context Intune script that pulls from your internal skills repository:

### Git Credential Management for Private Repos

If your skills repository or development repos are private (GitHub, Azure DevOps), the Cloud PC needs Git credentials. The recommended approach is **Personal Access Tokens (PATs)** stored in Windows Credential Manager via Git Credential Manager (GCM), which is included with Git for Windows:

1. The developer generates a PAT in GitHub or Azure DevOps with the minimum required scopes (typically `repo` read-only for skills, `repo` read-write for development).
2. On first `git clone` or `git pull`, GCM prompts for credentials and stores them securely in Windows Credential Manager.
3. Subsequent operations use the cached credential automatically.

For automated skill updates via Intune scripts, the PAT can be passed inline (though this is less secure):

```powershell
git clone "https://<PAT>@github.com/org/approved-agent-skills.git" $tempDir
```

For Azure DevOps, GCM supports Azure AD-backed authentication natively, so no PAT is required if the developer (or agent identity) has appropriate project access.

> **⚠️ Warning:** Never bake PATs into the image. They are user-specific, time-limited credentials that must be managed per-developer.

```powershell
# Runs in user context via Intune
$skillsDest = "$env:USERPROFILE\.agents\skills"
$tempDir = "$env:TEMP\skill-update-$(Get-Date -Format 'yyyyMMddHHmmss')"

git clone --depth 1 "https://github.com/bighatgroup/approved-agent-skills.git" $tempDir 2>&1 | Out-Null
if ($LASTEXITCODE -eq 0) {
    Copy-Item -Path "$tempDir\*" -Destination $skillsDest -Recurse -Force
    Remove-Item -Path $tempDir -Recurse -Force
}
```

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

