# Part VII: Reference

---

## Chapter 34: Troubleshooting Reference

This chapter is organized by pipeline phase. Before scanning troubleshooting items, identify which phase your issue occurred in, then go directly to that section. For pre-written diagnostic commands and scripts, see Chapter 35 (PowerShell Scripts).

### Build and Image Pipeline

- AIB build fails during provisioning: confirm all required resource providers are registered and the managed identity has Contributor on the resource group. Check registration status with `az provider show --namespace Microsoft.VirtualMachineImages --query registrationState`.
- Terraform apply hangs on image build: check AIB run status in the Azure Portal (Azure Image Builder > Image templates > select template > Runs) and validate network egress to installer sources. Run `az image builder show-runs --name <template-name> --resource-group <rg> --output table` to get run status from the CLI.
- Version drift: ensure `terraform.tfvars` values match the versions pinned in the build scripts and the component matrix. See Chapter 35's Initialize-BuildWorkstation.ps1 for prerequisite validation.

### Build Pipeline Failures

| Symptom | Cause | Resolution |
|---|---|---|
| **AIB build times out** (120+ min) | Windows Update taking too long, or large cumulative update | Increase `build_timeout_minutes` to 150--180; exclude problematic KBs |
| **npm install fails with EACCES** | PATH not refreshed after Node.js install | Ensure `Update-SessionEnvironment` is called after MSI install |
| **`openclaw: command not found`** after install | npm global bin not in PATH | Run `Update-SessionEnvironment`; verify `C:\Program Files\nodejs` is in system PATH |
| **Sysprep fails** | Machine was domain-joined, or recovery partition exists | Verify source image is clean marketplace image; check for provisioning packages |
| **Image import fails in Intune** | Missing ACG feature flags | Verify all five features (SecurityType, Hibernate, DiskController, AccelNet, SecureBoot) |
| **Active Setup (see Chapter 10) doesn't run** | Registry key malformed or version not bumped | Check `HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\OpenClaw-ConfigHydration`; verify `StubPath` and `Version` |
| **VS Code extensions missing** after login | Extensions installed into System profile during build | Install only Copilot during build; deliver others via post-provisioning Intune script |
| **OpenClaw gateway won't start** | Port conflict or missing config | Check if port 18789 is in use; verify `template-config.json` exists in ProgramData |
| **Terraform plan shows perpetual diff** | Using `timestamp()` instead of `time_static` | Use the `time_static` resource pattern |

**Diagnostic commands for build pipeline failures:**

```powershell
# Verify all tools are in PATH
node --version; python --version; git --version; openclaw --version; claude --version

# Check Active Setup registry
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\OpenClaw-ConfigHydration"

# Check OpenClaw gateway status
Invoke-WebRequest -Uri "http://127.0.0.1:18789/health" -UseBasicParsing

# Verify SBOM exists
Get-ChildItem "C:\ProgramData\ImageBuild\sbom-*.json"

# Check image build log (during build)
az image builder show-runs --name "aib-w365-dev-ai-1-0-0" --resource-group "rg-w365-images" --output table
```

### Windows 365 Import

- Import fails with image not found: verify the image version exists in the Azure Compute Gallery and is replicated to the target region. Check replication status in the Azure Portal (Azure Compute Gallery > Image definitions > select version > Replication).
- Provisioning policy cannot see the image: confirm the Intune image import succeeded and the image is assigned to the correct region. Navigate to Intune admin center > Devices > Windows 365 > Azure network connections to verify import status.
- DR region provisioning fails: replicate the ACG image version to the failover region and re-import if needed. Run `az sig image-version create` with the failover region included in `--target-regions`.

### First Login Experience

- PATH changes not visible: restart the session after MSI installs or trigger a logoff/logon cycle. Verify the current PATH with `$env:PATH -split ';'` in a new PowerShell window.
- GitHub Desktop does not appear: ensure the hydration (see Chapter 22) step runs on first login and that the user profile is not roaming. Check Active Setup (see Chapter 10) ran by verifying `HKCU:\SOFTWARE\Microsoft\Active Setup\Installed Components\OpenClaw-ConfigHydration` exists and its version matches the HKLM source key.
- VS Code extensions missing: validate the Intune script ran in user context and that the extension IDs are correct. Navigate to Intune admin center > Devices > Scripts > select the script > Device status to check per-device execution results.

### Agent Runtime

- Agent cannot access API keys: verify Intune settings catalog or environment variable deployment for the agent account. Check the deployed environment variables with `[System.Environment]::GetEnvironmentVariable('ANTHROPIC_API_KEY', 'Machine')`.
- CLI commands fail: confirm Node.js and npm are installed system-wide and that global npm paths are in `PATH`. Run `node --version; npm --version; openclaw --version` in a new session to verify availability.
- MCP servers not available: confirm global npm installs completed and the config template was hydrated (see Chapter 22) into the user profile. Check `%USERPROFILE%\.openclaw\workspace\config\mcporter.json` exists and contains the expected MCP server entries. Note: `mcporter.json` is OpenClaw's MCP server registry — it is distinct from `%APPDATA%\Claude\claude_desktop_config.json`, which is the standalone Claude Desktop application config and is not used in this deployment.
- OpenClaw gateway fails to start after hydration or after reprovisioning: run `openclaw onboard` manually. This command is normally handled automatically by the Active Setup hydration script, but it is idempotent and safe to run again as a manual recovery step. It detects the existing configuration and re-initialises only missing components.

### Operations

- Reprovisioning resets user config: store everything critical in repo or OneDrive and rely on Active Setup (see Chapter 10) to rehydrate config. Verify the Active Setup key is intact after reprovisioning with `Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\OpenClaw-ConfigHydration"`.
- Updates on running Cloud PCs fail: use an Intune platform script in user context and avoid running update commands during image build. Navigate to Intune admin center > Devices > Scripts to deploy and monitor update scripts.

---

