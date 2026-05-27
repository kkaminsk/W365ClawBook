## Chapter 18: The Reprovisioning Reality

This is the single most important operational concept in Windows 365 image management, and it shapes every decision about what goes into your image versus what you deliver via Intune.

**When you change the image in a provisioning policy, only newly provisioned Cloud PCs receive the new image. Existing Cloud PCs are completely unaffected.**

There is no mechanism to "push" a new image to running Cloud PCs. If you need an existing Cloud PC to use the updated image, you must **reprovision** it, and reprovisioning **deletes and recreates the Cloud PC**, destroying all user data, locally installed apps, and customisations.

### Implications for the Developer Image

| Decision | Reasoning |
|----------|-----------|
| Keep the image lean | Anything baked in is only updated via reprovisioning |
| Install *runtimes and binaries* in the image | These change infrequently |
| Deliver *configuration and extensions* via Intune | These can be updated on running Cloud PCs |
| Use `npm update -g` for agent version bumps | No reprovisioning needed |
| Plan for data preservation via Git (primary) and OneDrive (if licensed) | Reprovisioning destroys local data |
| Monthly rebuilds aligned with Patch Tuesday | New Cloud PCs shouldn't start with stale updates (see below) |

### Monthly Rebuild Operational Effort

The monthly rebuild cadence aligns with Patch Tuesday. In practice, the operational effort per rebuild is:

1. **Run `Initialize-TerraformVars.ps1`**: the script auto-detects the latest versions of all pinned packages, fetches SHA256 checksums, and detects the latest Windows 11 marketplace image. Review the changes and accept.
2. **Bump `image_version`**: increment the minor version (e.g., `1.1.0` -> `1.2.0`).
3. **Run `terraform apply`**: wait 75--120 minutes for the build.
4. **Verify and import**: check the ACG image version, import into Intune, test with a pilot group.
5. **Promote**: set `exclude_from_latest=false` and update the production provisioning policy.

The heavy lifting (version detection, checksum computation, and API queries) is automated by `Initialize-TerraformVars.ps1`. The full script implementation is available in the companion repository at `scripts/Initialize-TerraformVars.ps1`. The operator's job is to review the detected changes, run the build, and validate the result. Total hands-on time: approximately 30 minutes per rebuild, plus the automated build wait.

### "Rollback" Is a Redeploy

There is no in-place rollback. To "roll back" a bad image:

1. Update the provisioning policy to reference the previous known-good image version
2. Reprovision affected Cloud PCs (this destroys and recreates them)

This is why staged rollout (Chapter 16) and data preservation are essential.

### Data Preservation: Git Over OneDrive

For AI agent Cloud PCs, the primary data preservation strategy should be **Git**, not OneDrive Known Folder Move.

Agent workspaces are code repositories. Every meaningful artefact (source code, configuration, documentation, OpenSpec proposals) should be committed and pushed to a remote repository (GitHub, Azure DevOps) as part of the normal workflow. If the Cloud PC is reprovisioned, the developer clones the repository and is back to work in minutes.

OneDrive KFM may be available depending on the user's Microsoft 365 licensing entitlements (E3/E5 include OneDrive for Business; F3 and standalone Entra P1 licences do not). If the dedicated agent account model (Chapter 27) uses a minimal licence without OneDrive, KFM is not an option at all.

Even when OneDrive is available, it introduces a risk worth acknowledging: **OneDrive syncs file changes bidirectionally and continuously.** An agent that writes large build artefacts, generates extensive logs, or creates temporary files in the Documents or Desktop folders will trigger constant sync activity. This can consume bandwidth, hit OneDrive storage quotas, and create sync conflicts if the developer accesses the same OneDrive from another device.

The recommended approach:

- **Git for all code and configuration.** Commit early, commit often, push to remote. This is the canonical backup.
- **OneDrive for non-code artefacts** (if licensed), such as personal notes, downloaded references, one-off files that don't belong in a repository.
- **Treat the Cloud PC as disposable.** If reprovisioning destroys something you can't recover, it should have been in Git.

> **💡 Tip:** Configure the OpenClaw workspace directory outside of the OneDrive sync folders (e.g., `C:\Dev\` or `%USERPROFILE%\Code\` rather than `%USERPROFILE%\Documents\`). This prevents OneDrive from attempting to sync agent workspace files, which can include large `node_modules` directories and frequently-changing memory files.

---

