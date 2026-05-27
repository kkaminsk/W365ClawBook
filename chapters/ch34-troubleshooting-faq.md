# Part VII: Reference

---

## Chapter 34: Troubleshooting and FAQ

### Build and Image Pipeline

- AIB build fails during provisioning: confirm all required resource providers are registered and the managed identity has Contributor on the resource group.
- Terraform apply hangs on image build: check AIB run status in Azure Portal and validate network egress to installer sources.
- Version drift: ensure `terraform.tfvars` values match the versions pinned in the build scripts and the component matrix.

### Windows 365 Import

- Import fails with image not found: verify the image version exists in the Azure Compute Gallery and is replicated to the target region.
- Provisioning policy cannot see the image: confirm the Intune image import succeeded and the image is assigned to the correct region.
- DR region provisioning fails: replicate the ACG image version to the failover region and re-import if needed.

### First Login Experience

- PATH changes not visible: restart the session after MSI installs or trigger a logoff/logon cycle.
- GitHub Desktop does not appear: ensure the hydration step runs on first login and that the user profile is not roaming.
- VS Code extensions missing: validate the Intune script ran in user context and that the extension IDs are correct.

### Agent Runtime

- Agent cannot access API keys: verify Intune settings catalog or environment variable deployment for the agent account.
- CLI commands fail: confirm Node.js and npm are installed system-wide and that global npm paths are in `PATH`.
- MCP servers not available: confirm global npm installs completed and the config template was hydrated into the user profile.

### Operations

- Reprovisioning resets user config: store everything critical in repo or OneDrive and rely on Active Setup to rehydrate config.
- Updates on running Cloud PCs fail: use an Intune platform script in user context and avoid running update commands during image build.

---

