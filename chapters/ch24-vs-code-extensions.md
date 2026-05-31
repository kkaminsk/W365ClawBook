## Chapter 24: VS Code Extensions

### Decision Framework: Image vs Post-Provisioning

The general rule is: **put the minimum into the image, let the developer customize afterward.** Specifically:

| In the Image | Post-Provisioning | Developer Choice |
|---|---|---|
| GitHub Copilot (binary pre-positioned; post-provisioning script handles activation and user-specific configuration — see note below) | GitHub Copilot (activation and user-specific configuration) | Theme, font, keybinding extensions |
| No | Language-specific extensions (Python, C#) | Productivity tools (GitLens, TODO Tree) |
| No | Claude Code VS Code extension | Personal preference extensions |
| No | Linting / formatting extensions | No |

> **Note:** The image pre-positions the GitHub Copilot extension binary during the build step. The post-provisioning script handles activation and user-specific configuration. This is not a contradiction — baking the binary speeds up first-login readiness while still requiring user-context setup before Copilot is functional.

Extensions baked into the image are frozen at build time and update only via reprovisioning. Extensions installed post-provisioning update automatically via VS Code's built-in mechanism. Keep the image lean; developers are best positioned to choose their own tooling beyond the baseline.

### Why Not Bake Most Extensions

VS Code extensions install into `%USERPROFILE%\.vscode\extensions`. During the image build (running as Local System), this resolves to the system profile, invisible to the actual developer. While you can install some extensions via `code.cmd --install-extension` during the build (as we do for GitHub Copilot), this is not reliable for all extensions and doesn't support user-context extensions.

### Post-Provisioning Delivery

Deploy extensions via an Intune user-context PowerShell script:

```powershell
# Runs in the user's context after provisioning
code.cmd --install-extension anthropic.claude-code --force
code.cmd --install-extension ms-python.python --force
code.cmd --install-extension github.copilot --force
code.cmd --install-extension github.copilot-chat --force
```

This ensures extensions are installed into the correct user profile and can be updated independently via VS Code's built-in extension update mechanism. This script must run after VS Code is installed on the Cloud PC. For the Intune deployment pattern used to execute this script post-provisioning, see Chapter 25.

### VS Code Internal MCP Registry and Allowlist (November 2025)

VS Code shipped internal MCP registry and allowlist controls in public preview in November 2025. For enterprise Windows 365 deployments, these controls provide an additional governance layer for MCP servers used by Claude Code and other VS Code-hosted agents:

**Configuring the MCP allowlist:**
```json
// .vscode/settings.json (machine-level via Intune Settings Catalog)
{
  "mcp.allowedServers": [
    "github.com/your-org/approved-mcp-server-1",
    "github.com/your-org/approved-mcp-server-2"
  ],
  "mcp.blockUnlistedServers": true
}
```

Deploy this setting via Intune Settings Catalog to all Cloud PCs running VS Code. This prevents developers from installing unapproved MCP servers through VS Code's MCP integration, complementing the file-based MCP server allow-list controls in Chapter 28.

**Monitoring:** Any attempt to install an MCP server not in the allowlist generates a VS Code warning and should generate a Defender for Endpoint alert when combined with the Chapter 33 process monitoring rules.

---

