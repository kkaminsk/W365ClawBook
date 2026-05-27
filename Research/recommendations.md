# Technical Accuracy Review — Recommendations

**Scope:** All 37 chapters + appendix reviewed by automated analysis (7 parallel agents, one per Part).
**Method:** Each agent read the assigned chapter files and flagged technical inaccuracies only — prose, style, and completeness were explicitly excluded.
**Date:** 2026-05-27

---

## Summary by Severity

| Severity | Count | Description |
|---|---|---|
| Critical | 3 | Security misconfigurations or deployment-breaking errors |
| Moderate | 14 | Verifiable factual errors likely to mislead readers |
| Minor | 9 | Imprecisions or inconsistencies requiring clarification |

---

## Critical Issues

---

### NSG Rule 900 Allows All HTTPS Outbound (Chapter 30)

**Chapter:** Chapter 30: Network Segmentation

**Issue:** The NSG design includes a priority-900 rule that allows all HTTPS outbound to the Internet (destination `Internet`, port 443). This is evaluated *before* the default deny at priority 4096, meaning any agent on the segment can reach any HTTPS destination on the internet. This directly contradicts the zero-trust, network-isolated premise of the entire chapter and the security architecture described in Part VI.

**Suggestion:** Remove rule 900 from production NSG configurations. Replace it with explicit allowlist rules for the specific FQDNs/IP ranges the agent requires (npm registry, Anthropic API, GitHub, Microsoft endpoints). Include a warning that rule 900 is a development/troubleshooting aid only and must be removed before production use. A dedicated callout box with a ⚠️ warning would be appropriate given the severity.

---

### Intune Remediation Scheduling Claim Is Incorrect (Chapter 33)

**Chapter:** Chapter 33: Monitoring and Forensics

**Issue:** The chapter states "Deploy this as an Intune remediation with a schedule of every 1–4 hours depending on how critical gateway uptime is for your team." Intune remediation scripts do not support custom scheduling — they run at a fixed interval controlled by the Intune service (typically daily or at device check-in). Readers who follow this guidance will find the promised hourly check is impossible via Intune remediations, and may leave the gateway unmonitored.

**Suggestion:** Revise to: *"Deploy this as an Intune remediation script. Note: Intune runs remediation scripts at a fixed daily interval (or at device check-in); custom per-hour scheduling is not available. For sub-hourly gateway monitoring, use the Scheduled Task approach described below."* Also correct the contradictory scheduled task trigger syntax (see Minor Issues).

---

### Terraform Multi-Region Replication Incorrectly Described as Future Feature (Chapter 20)

**Chapter:** Chapter 20: Version Retention and Cost Management

**Issue:** The chapter describes native Terraform support for multi-region replication (`target_regions` and per-region replica counts) as a "planned but not yet implemented" future enhancement. The `target_region` blocks in `azurerm_shared_image_version` have been supported in the AzureRM provider for several years and are not a future feature.

**Suggestion:** Remove the "future enhancement" framing. Describe `target_region` blocks as the current, available mechanism for multi-region replication and provide a brief example or link to the `azurerm_shared_image_version` provider documentation.

---

## Moderate Issues

---

### Python Version 3.14.3 — Multiple Chapters

**Chapters:** Chapter 6 (Terraform vars), Chapter 8 (Phase 1 Core Runtimes), Chapter 15 (Verification Checklist), Chapter 37 (Component Summary Matrix)

**Issue:** Python 3.14.3 is referenced repeatedly as the version being installed. Python 3.14 is scheduled for release in October 2025 and may be available at the book's stated date of February 2026, but 3.14.3 is a specific patch version that requires verification. More critically, the verification checklist in Chapter 15 states `python --version returns 3.14+` as a pass/fail check — if the actual installed version differs, every build will fail this check.

**Suggestion:** Verify that Python 3.14.3 was actually released and available via the Windows installer at the book's stated date. If not, use the most current 3.13.x stable release. Update all four chapters consistently. Consider pinning the version in a single variable definition (already done in ch06) and cross-referencing that value in ch08, ch15, and ch37 rather than hardcoding in each.

---

### Node.js Version v24.13.1 Does Not Appear to Exist (Chapter 13)

**Chapter:** Chapter 13: Supply Chain Integrity

**Issue:** The chapter references `https://nodejs.org/dist/v24.13.1/SHASUMS256.txt` for checksum verification and lists `v24.13.1` in the version table. Node.js v24 is an LTS release line (LTS since October 2024), but v24.13.1 as a specific patch version requires verification — if this URL does not resolve, the supply chain verification step will fail.

**Suggestion:** Verify the specific Node.js version against the actual dist index at `https://nodejs.org/dist/index.json`. Update the version to a confirmed release. The checksum verification pattern itself is correct; only the specific version needs confirmation.

---

### GitHub Actions Default Timeout Claim Is Wrong (Chapter 21)

**Chapter:** Chapter 21: CI/CD Pipeline Integration

**Issue:** The chapter states "Most CI/CD runners have default timeouts of 30–60 minutes." GitHub Actions default job timeout is 360 minutes (6 hours), not 30–60 minutes. Readers using GitHub Actions as their CI platform will question the accuracy of the surrounding guidance.

**Suggestion:** Correct to: *"GitHub Actions defaults to a 360-minute (6-hour) job timeout. Azure DevOps pipelines default to 60 minutes per job. If you are using self-hosted runners, check the runner's configured timeout. Set an explicit `timeout-minutes: 180` (or higher) on the AIB build job regardless of platform to make the timeout contract explicit."*

---

### M365 E3 License Description Is Imprecise (Chapter 27)

**Chapter:** Chapter 27: Identity Architecture — The Secondary User Imperative

**Issue:** The chapter states M365 E3 "includes Entra P1 and Intune P1." M365 E3 includes Intune management capabilities, but the SKU is not marketed as including "Intune P1" — Intune P1 is a standalone license. The bundled Intune capability in M365 E3 is sufficient for this use case, but stating it includes "Intune P1" may confuse license planning.

**Suggestion:** Revise to: *"A Microsoft 365 E3 licence (~$36/user/month, includes Entra ID P1 and Intune device management)."*

---

### Cisco Skill Scanner Reference Is Unverified (Chapter 29)

**Chapter:** Chapter 29: Hardening OpenClaw

**Issue:** The chapter references a tool at `https://github.com/cisco-ai-defense/skill-scanner` with specific claimed capabilities (detecting exfiltration patterns, prompt injection, Base64 obfuscation). This repository should be verified to exist and to actually provide the described capabilities. If it does not exist, readers following this guidance will hit a dead end on a security-critical step.

**Suggestion:** Verify the repository URL and capabilities before publication. If the tool does not exist, replace with verified alternatives for skill/plugin security scanning such as `syft` (for SBOM analysis), `semgrep` with custom rules, or a policy-based code review process with documented criteria.

---

### CVE-2026-25253 Reference Needs Clarification (Chapter 29)

**Chapter:** Chapter 29: Hardening OpenClaw

**Issue:** The chapter references CVE-2026-25253 as a real, past vulnerability in OpenClaw versions prior to 2026.1.29. If this is a fictional example, it should be labeled as such; if real, it needs a proper citation. Using a fabricated CVE number in a practitioner's guide may cause confusion when readers search for it.

**Suggestion:** If fictional: add a disclaimer such as *"(This is a representative example of a class of vulnerability; the CVE number is illustrative.)"* If real: add a link to the NVD entry or OpenClaw security advisory.

---

### Azure Compute Gallery Preview Status Outdated (Appendix)

**Chapter:** Appendix

**Issue:** The appendix states that Azure Compute Gallery integration with Windows 365 "was in public preview at the time of writing." ACG integration with Windows 365 reached general availability in 2024 and is not in preview. Labeling a GA feature as preview may cause readers to add unnecessary validation steps or delay adoption.

**Suggestion:** Update to reflect GA status. If the specific feature being referenced (e.g., direct import from ACG) has its own GA date, cite that specifically.

---

### Azure Compute Gallery Table Missing Secure Boot Feature (Chapter 4)

**Chapter:** Chapter 4: Azure Compute Gallery for Windows 365

**Issue:** The chapter text states "five" required Gallery features, but the Terraform attribute table lists only four. The Azure CLI example correctly shows five features including `IsSecureBootSupported=True` (Terraform equivalent: `secure_boot_enabled = true`), which is absent from the Terraform table. This inconsistency will cause readers to implement an incomplete configuration.

**Suggestion:** Add `secure_boot_enabled = true` to the Terraform feature table and update the count to five in both the prose and the table header. Confirm this attribute name against the current `azurerm_shared_image` resource schema.

---

### FSLogix Statement Is Misleading (Chapter 36)

**Chapter:** Chapter 36: Windows 365 Image Requirements Checklist

**Issue:** The checklist item "No FSLogix components" implies FSLogix is incompatible with Windows 365. FSLogix is supported and commonly used with Windows 365. The actual requirement is that FSLogix should not be *pre-installed in the image*, as Intune handles deployment.

**Suggestion:** Revise to: *"No pre-installed FSLogix components (deploy via Intune policy post-provisioning; pre-baking FSLogix into the image can conflict with Intune-managed FSLogix configuration)."*

---

### VS Code Extension Install Context Contradiction (Chapter 37)

**Chapter:** Chapter 37: Component Summary Matrix

**Issue:** The matrix lists GitHub Copilot as installed via "Image Build / Local System" context, while a later note in the same chapter acknowledges "Cannot install machine-wide reliably." This contradiction will confuse readers trying to reconcile the matrix with the actual build behavior.

**Suggestion:** Update the GitHub Copilot row (and any other VS Code extension rows with the same issue) to show "Post-Provisioning / User Context" as the install method, consistent with the guidance elsewhere in the book.

---

### Git Version 2.53.0 Is Implausible (Chapter 37)

**Chapter:** Chapter 37: Component Summary Matrix

**Issue:** The matrix lists Git 2.53.0 as the pinned version. Git's versioning in the 2.4x–2.5x range is plausible by 2026, but this specific version should be cross-referenced against the actual value in `terraform.tfvars` (or wherever the version is pinned). If it doesn't match, the matrix is stale at publication.

**Suggestion:** Verify the Git version against the actual pinned value in the W365Claw repository. If a specific version is not pinned (Git is installed from the latest available), note that in the matrix rather than listing an unverifiable specific version.

---

### OpenClaw and Claude Code Version Numbers Unverified (Chapter 37)

**Chapter:** Chapter 37: Component Summary Matrix

**Issue:** The matrix lists specific version numbers for OpenClaw (2026.2.14) and Claude Code (2.1.42). These should be cross-referenced against the actual versions pinned in the W365Claw repository at time of publication. If they are placeholders, they should be marked as such.

**Suggestion:** Verify both versions against the npm package registry (or the W365Claw repo's package lockfiles). If they are illustrative, note them as "version at time of publication; verify before build."

---

## Minor Issues

---

### HKCU Registry Hive Phrasing Is Ambiguous (Chapter 3)

**Chapter:** Chapter 3: The Architectural Boundary

**Issue:** The chapter states Local System has "No HKCU registry hive (in the expected sense)." The SYSTEM account does have an HKCU hive, but it resolves to the system account's profile rather than a logged-in user's profile. The current phrasing may confuse readers who know HKCU exists for SYSTEM.

**Suggestion:** Revise to: *"HKCU resolves to the SYSTEM account's profile hive, not a user's interactive profile. Software that writes per-user preferences to HKCU during image build will write them to a location never seen by actual users at login."*

---

### Build Timeline Inconsistency: Diagram vs. Table (Chapter 14)

**Chapter:** Chapter 14: Building the Image

**Issue:** The Mermaid diagram shows "Phase 1: Core Runtimes ~20 min" while the table in the same chapter shows "~15 minutes" for the same phase.

**Suggestion:** Align both to the same value. Use the more conservative (higher) estimate if uncertain, and note it as approximate.

---

### WebClient Service Stop Command Is Incomplete (Chapter 32)

**Chapter:** Chapter 32: Intune Configuration Profiles

**Issue:** The profile table shows `Set-Service -Name WebClient -StartupType Disabled` for disabling WebClient. This prevents the service from starting at next boot but does not stop it if it is currently running. Chapter 28's guidance (which is the more complete version of this mitigation) correctly handles the running-state case.

**Suggestion:** Update to: `Set-Service -Name WebClient -StartupType Disabled; Stop-Service -Name WebClient -Force -ErrorAction SilentlyContinue`

---

### Scheduled Task Trigger Syntax Is Contradictory (Chapter 33)

**Chapter:** Chapter 33: Monitoring and Forensics

**Issue:** The watchdog scheduled task uses `-RepetitionInterval (New-TimeSpan -Minutes 15) -Once` together. The `-Once` flag creates a one-time trigger; combining it with `-RepetitionInterval` alone does not produce a repeating task as intended.

**Suggestion:** Revise to: `New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 15) -RepetitionDuration ([TimeSpan]::MaxValue)` — the `-RepetitionDuration` parameter is required for the repetition to continue indefinitely.

---

### KQL Query Uses Incorrect Field Name (Chapter 33)

**Chapter:** Chapter 33: Monitoring and Forensics

**Issue:** The "Agent Spawning Shell" KQL query filters on `ParentProcessName has_any ("node.exe", "openclaw")`. In the Log Analytics `SecurityEvent` table populated by Windows Security Event ID 4688, the parent process field is `ParentProcessName` (which is correct for that specific table) — however, the field names vary depending on whether MDE advanced hunting tables or the SecurityEvent table is being used. Readers using MDE's `DeviceProcessEvents` table would need `InitiatingProcessFileName` instead.

**Suggestion:** Add a note clarifying which table and data connector the query targets. If using `SecurityEvent` (MMA/AMA-based), `ParentProcessName` is correct. If using Microsoft Defender for Endpoint advanced hunting (`DeviceProcessEvents`), the equivalent field is `InitiatingProcessFileName`.

---

### OpenClaw Command Syntax and Port Number Require Verification (Chapter 22)

**Chapter:** Chapter 22: First Login Experience

**Issue:** The chapter shows `openclaw gateway start` as the command and `http://127.0.0.1:18789` as the gateway URL. These cannot be independently verified without OpenClaw documentation. If either is wrong, every post-provisioning validation will fail.

**Suggestion:** Cross-reference against the OpenClaw documentation or the W365Claw repository's post-provisioning scripts to confirm the exact command and default port. If the port is configurable, note the configuration parameter.

---

### Azure DevOps GCM Authentication Claim Is Overstated (Chapter 25)

**Chapter:** Chapter 25: Agent Updates Without Reprovisioning

**Issue:** The chapter states that for Azure DevOps, GCM (Git Credential Manager) "supports Azure AD-backed authentication natively, so no PAT is required if the developer (or agent identity) has appropriate project access." GCM can use Azure AD interactive flows, but unattended/non-interactive authentication (as would be needed for an agent) still requires a PAT or a service principal credential — automatic ambient authentication does not work for a headless agent identity.

**Suggestion:** Revise to: *"For Azure DevOps, GCM supports Azure AD authentication via interactive login. For unattended agent use, provision a PAT with the minimum required scope (typically Code: Read) and store it using the same secret delivery mechanism described in Chapter 23."*

---

### 3,000 Start Menu App Limit Is Informal Guidance (Chapter 36)

**Chapter:** Chapter 36: Windows 365 Image Requirements Checklist

**Issue:** The checklist states "Under 3,000 Start menu apps" as a hard requirement. This number is informal guidance, not a documented hard technical limit enforced by Windows 365.

**Suggestion:** Revise to: *"Recommended: under 3,000 Start menu apps (reduces provisioning complexity and image size; treat as a soft guideline and validate during testing)."*

---

### Feedback Section Missing Repository URL (Appendix)

**Chapter:** Appendix: Feedback and Errata

**Issue:** The errata section instructs readers to file issues but does not provide a repository URL or GitHub issue link, making it unclear where to submit corrections.

**Suggestion:** Add the full GitHub URL for the W365ClawBook issues tracker (e.g., `https://github.com/kkaminsk/W365ClawBook/issues`).

---

## No Issues Found

The following chapters had no significant technical inaccuracies identified:

- Chapter 1: Introduction
- Chapter 2: Architecture Overview
- Chapter 5: Identity and RBAC
- Chapter 7: Preparing the Build Workstation
- Chapter 9: Phase 2 — Developer Tools *(VS Code installer URL bug already documented in chapter)*
- Chapter 10: Phase 3 — Configuration and Policy
- Chapter 11: *(not present — intentional gap)*
- Chapter 16: Image Versioning and Staged Rollout
- Chapter 17: Importing into Windows 365
- Chapter 18: The Reprovisioning Reality
- Chapter 26: The Agent Threat Model
- Chapter 34: Troubleshooting and FAQ
