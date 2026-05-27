## Chapter 12: Windows Update and Sysprep

### Why Windows Update in the Image

The AIB template includes a Windows Update customizer to apply cumulative updates during the build:

```json
{
  "type": "WindowsUpdate",
  "searchCriteria": "IsInstalled=0",
  "filters": [
    "exclude:$_.Title -like '*Preview*'",
    "include:$true"
  ],
  "updateLimit": 40
}
```

Without this, newly provisioned Cloud PCs start with a stale image and depend on Windows Update post-provisioning, increasing first-sign-in time by 30--60 minutes and leaving a security window during which the machine is vulnerable to patched exploits.

The `exclude:Preview` filter prevents preview/beta updates from being installed, which could introduce instability.

### Windows Update Retry Guidance

Windows Update within AIB can be unpredictable. Cumulative updates on fresh images occasionally fail on the first attempt, particularly when multiple updates compete for restart requirements. If your build fails during the Windows Update phase:

1. **Check the build log** for specific KB failure codes. Common culprits are timeout (the update took longer than the customizer's internal limit) and transient download failures.
2. **Increase the build timeout**: set `build_timeout_minutes = 150` (or higher) in `terraform.tfvars` to accommodate large cumulative updates.
3. **Re-run the build.** AIB builds are idempotent; a fresh build VM starts clean. Transient failures often succeed on retry.
4. **If a specific update consistently fails**, add it to the `filters` exclusion list (e.g., `"exclude:$_.Title -like '*KB5034567*'"`) and apply it post-provisioning via Windows Update for Business instead.

> **💡 Tip:** Monthly cumulative updates released on Patch Tuesday can take 45--60+ minutes on fresh images. Plan your build windows accordingly and don't schedule builds on Patch Tuesday itself; wait 2--3 days for the update CDN to stabilize.

### Sysprep Constraints

Sysprep runs automatically at the end of the AIB build. There are three non-negotiable constraints:

1. **The image must be generalized.** Sysprep removes machine-specific information (SID, computer name, etc.) so each Cloud PC gets a unique identity.
2. **The image must never have been Entra joined, AD joined, or Intune enrolled.** Windows 365 handles domain join and Intune enrollment during provisioning. A pre-joined image will fail.
3. **No recovery partition.** Windows 365 manages its own recovery mechanisms.

These constraints are enforced by Windows 365 and are non-negotiable. The marketplace source image satisfies all of them, and as long as your build scripts don't join a domain or enroll in Intune, the resulting image will be compliant.

### The Final Restart

After Windows Update, the AIB template includes a final restart to ensure all updates are fully applied before Sysprep:

```json
{
  "type": "WindowsRestart",
  "restartTimeout": "10m"
}
```

---

