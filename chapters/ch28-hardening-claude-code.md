## Chapter 28: Hardening Claude Code

### managed-settings.json Deep Dive

The `managed-settings.json` file at `C:\ProgramData\ClaudeCode\managed-settings.json` enforces enterprise policy at the machine level. This file cannot be overridden by developers.

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

**Mitigation:** Block this flag via managed-settings.json and monitor for its use in Sysmon process creation logs.

### Sandboxing Strategies

Claude Code's native `/sandbox` command relies on Linux-specific primitives. On Windows, use:

**Windows Sandbox** (covered in Chapter 31): A lightweight, disposable VM built into Windows 11 Enterprise. Ideal for analysing untrusted code or running high-risk agent tasks in a throwaway environment that is destroyed on close.

**Docker Desktop and WSL 2** are also viable sandbox options but are **out of scope for this document**. Both introduce a Linux execution environment with fundamentally different properties from the native Windows host: different filesystem semantics, different process models, different networking stacks, and different security boundaries. An agent running inside a Docker container or a WSL 2 distribution is effectively running on Linux, which changes assumptions about path handling, native toolchain availability, Windows API access, and Intune policy enforcement. These are valid deployment choices, but they warrant their own treatment rather than a brief mention here.

---

