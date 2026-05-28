# Part V: Post-Provisioning

---

## Chapter 22: First Login Experience

When a developer signs into their newly provisioned Cloud PC for the first time, several things happen automatically:

### Active Setup Hydration (see Chapter 10 for the mechanism)

The Active Setup entry registered during the image build executes `hydrate-config.ps1` in the user's security context. This:

1. Creates `%USERPROFILE%\.openclaw\` directory
2. Creates `%USERPROFILE%\Documents\OpenClawWorkspace\` directory
3. Copies the OpenClaw configuration template to `%USERPROFILE%\.openclaw\openclaw.json`
4. Copies curated agent skills to `%USERPROFILE%\.agents\skills\`
5. Copies MCP server configuration to `%USERPROFILE%\.openclaw\workspace\config\mcporter.json` — OpenClaw's MCP server connection registry, pre-populated with internal MCP server endpoints during image build

The developer's OpenClaw configuration, skills, and MCP server integrations are all in place before they open their first terminal.

### GitHub Desktop Hydration

The machine-wide GitHub Desktop provisioner detects the new user login and installs the GitHub Desktop application into `%LocalAppData%\GitHubDesktop\`. The user sees GitHub Desktop in their Start menu without any admin action.

### What the Developer Sees

The developer lands on a Windows 11 desktop with the following tools in two states:

**Ready to use** — these items are fully functional immediately after first login:

- Node.js, Python, PowerShell 7, Git, all available from any terminal
- VS Code with GitHub Copilot, ready to open and use
- GitHub Desktop, in the Start menu, ready for repository cloning
- MCP integrations: Microsoft Docs (ready out of the box)

**Requires setup before use** — these items need an API key or additional configuration before they can be used:

- OpenClaw: installed post-provisioning (user context), configuration pre-seeded, curated skills installed, MCP servers configured — waiting for an Anthropic API key
- Claude Code: managed settings enforced — waiting for an Anthropic API key
- MCP integrations: Perplexity and other API-backed servers — waiting for provider keys

The only things missing are authentication tokens, and that's intentional. API keys and authentication tokens are intentionally absent from the shared image because they are per-developer credentials that must not be embedded in infrastructure accessible to other users — see Chapter 23 for the delivery pattern.

### OpenClaw First-Login Walkthrough

Because the image includes a pre-seeded `openclaw.json` configuration template (copied by Active Setup), the developer does **not** need to run `openclaw onboard`. The onboarding wizard is designed for first-time setup on a clean machine; it creates the config file, selects a model, and configures the gateway. All of that is already done by the hydration script.

Instead, the developer's first-login workflow is:

1. **Open a terminal** (PowerShell 7 or Windows Terminal).
2. **Set the Anthropic API key:**
   ```powershell
   [Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "<your-key>", "User")
   ```
3. **Start the OpenClaw gateway:**
   ```powershell
   openclaw gateway start
   ```
4. **Verify:** Open `http://127.0.0.1:18789` in a browser to confirm the gateway is running.
5. **Begin working.** The gateway is connected, skills are loaded, and MCP servers are configured.

If the developer runs `openclaw onboard` anyway, it will detect the existing configuration and offer to reconfigure; this is harmless but unnecessary. The profile initialization via Active Setup already contains the enterprise-standard model, gateway port, and workspace path.

For **Claude Code**, the developer runs `claude login` to authenticate via Anthropic's browser-based flow, or sets `ANTHROPIC_API_KEY` as described above. No additional onboarding is needed; the managed-settings.json enforces enterprise policy automatically.

API keys and MCP server credentials are configured manually, as discussed in Chapter 23.

---

