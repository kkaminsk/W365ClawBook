## Chapter 24: VS Code Extensions

### Decision Framework: Image vs Post-Provisioning

The general rule is: **put the minimum into the image, let the developer customize afterward.** Specifically:

| In the Image | Post-Provisioning | Developer Choice |
|---|---|---|
| GitHub Copilot (core to the workflow) | Language-specific extensions (Python, C#) | Theme, font, keybinding extensions |
| | Claude Code VS Code extension | Productivity tools (GitLens, TODO Tree) |
| | Linting / formatting extensions | Personal preference extensions |

Extensions baked into the image are frozen at build time and update only via reprovisioning. Extensions installed post-provisioning update automatically via VS Code's built-in mechanism. Keep the image lean; developers are best positioned to choose their own tooling beyond the baseline.

### Why Not Bake Most Extensions

VS Code extensions install into `%USERPROFILE%\.vscode\extensions`. During the image build (running as Local System), this resolves to the system profile, invisible to the actual developer. While you can install some extensions via `code.cmd --install-extension` during the build (as we do for GitHub Copilot), this is not reliable for all extensions and doesn't support user-context extensions.

### Post-Provisioning Delivery

Deploy extensions via an Intune user-context PowerShell script:

```powershell
# Runs in the user's context after provisioning
code --install-extension anthropic.claude-code --force
code --install-extension ms-python.python --force
code --install-extension github.copilot --force
code --install-extension github.copilot-chat --force
```

This ensures extensions are installed into the correct user profile and can be updated independently via VS Code's built-in extension update mechanism.

---

