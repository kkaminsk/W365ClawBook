## Chapter 28: Hardening Claude Code

### managed-settings.json Deep Dive

The `managed-settings.json` file at `C:\ProgramData\ClaudeCode\managed-settings.json` enforces enterprise policy at the machine level. This file cannot be overridden by developers.

This file is baked into the image during Phase 3 configuration (Chapter 10) and deployed to `C:\ProgramData\Claude\managed-settings.json`. Verify the file exists at its deployment path after provisioning. Claude Code reads this path at startup; if the file is absent, policy is not enforced.

Settings in `managed-settings.json` take precedence over user-profile settings. A developer attempting to override a managed setting will find their change silently ignored at runtime.

**Restrictive Configuration for Security-Sensitive Environments:**

```json
{
  "autoUpdatesChannel": "stable",
  "permissions": {
    "defaultMode": "ask",
    "deny": [
      "curl *",
      "wget *",
      "Invoke-WebRequest *",
      "net use *",
      "cmdkey *",
      "rundll32 *",
      "powershell -encodedCommand *"
    ]
  },
  "disableAllHooks": true,
  "cleanupPeriodDays": 7
}
```

| Setting | Value | Purpose |
|---------|-------|---------|
| `defaultMode` | `"ask"` | Require explicit approval for all operations |
| `deny` | High-risk commands | Block download tools, credential access, encoded commands |
| `disableAllHooks` | `true` | Prevent pre/post-action scripts from malicious repos |
| `cleanupPeriodDays` | `7` | Limit chat transcript retention |

### The WebDAV Permission Bypass (Theoretical Risk)

This is a theoretical risk based on general Windows NTLM relay attack primitives, not a demonstrated Claude Code-specific exploit. However, it is worth modelling in your threat assessment.

Claude Code's permission system intercepts and gates file system and network calls. However, Windows handles UNC paths pointing to WebDAV shares (e.g., `\\attacker.com\share`) at a kernel level via the **WebClient** service. If a prompt injection tricked Claude Code into accessing a UNC path, Windows could automatically attempt to authenticate to the remote server using the current user's NTLM hash. This would bypass Claude Code's internal permission logic because it appears as a file read, not a network request.

**Mitigation -- Disable the WebClient Service:**

```powershell
Set-Service -Name WebClient -StartupType Disabled -Status Stopped
```

Deploy this via Intune as a PowerShell script targeting the Cloud PC device group.

### The dangerously-skip-permissions Risk

Claude Code includes a `--dangerously-skip-permissions` flag intended for CI/CD pipelines. Developer fatigue often leads users to enable this permanently. Once permissions are skipped, Claude Code becomes an unmediated shell for the LLM; any prompt injection results in immediate command execution.

**Mitigation:** Block this flag via managed-settings.json and monitor for its use in Sysmon process creation logs. The following JSON property disables the flag at the managed-settings level:

```json
{
  "dangerouslySkipPermissions": false
}
```

> **Note:** Consult the Claude Code documentation for the `dangerouslySkipPermissions` managed settings key and confirm the key name matches your installed version before deploying.

### Sandboxing Strategies

Claude Code's native `/sandbox` command relies on Linux-specific primitives. On Windows, use:

Sandboxing with Windows Sandbox is covered in Chapter 31 (Endpoint Protection). Network-level controls for containing agent traffic are in Chapter 30 (Network Segmentation). Chapter 31 also covers the Application Control and ASR rules that complement the Claude Code policy controls in this chapter.

---

