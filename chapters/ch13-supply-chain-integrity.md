## Chapter 13: Supply Chain Integrity

### The Problem

The image build process downloads software installers and npm packages from the public internet. Without integrity verification, a compromised upstream release (supply chain attack) would be silently baked into every provisioned Windows 365 developer image.

This is not a theoretical risk. Based on community-reported incidents, the "ClawHavoc" campaign indicated that approximately 12% of skills in the OpenClaw ecosystem's ClawHub registry were malicious (see Chapter 29). While independent verification of the exact figure varies by source, the pattern is consistent with broader supply chain attacks on npm packages and binary installers, which are well-documented and increasing in frequency.

### SHA256 Checksum Verification

A `Test-InstallerHash` function verifies every binary installer before execution:

```powershell
function Test-InstallerHash {
    param([string]$FilePath, [string]$ExpectedHash)
    if ([string]::IsNullOrWhiteSpace($ExpectedHash)) {
        Write-Host "[INTEGRITY] No SHA256 provided for $(Split-Path $FilePath -Leaf) -- skipping verification"
        return
    }
    $actual = (Get-FileHash -Path $FilePath -Algorithm SHA256).Hash
    if ($actual -ne $ExpectedHash.ToUpper()) {
        Write-Error "[INTEGRITY] SHA256 MISMATCH for $(Split-Path $FilePath -Leaf)! Expected: $ExpectedHash Got: $actual"
        exit 1
    }
    Write-Host "[INTEGRITY] SHA256 verified for $(Split-Path $FilePath -Leaf)"
}
```

The behaviour is opt-in: if a SHA256 hash is provided, verification is mandatory and a mismatch **fails the build**. If no hash is provided (empty string), the function logs a warning and continues.

Five Terraform variables control the checksums:

| Variable | Description |
|----------|-------------|
| `node_sha256` | SHA256 for Node.js MSI installer |
| `python_sha256` | SHA256 for Python installer |
| `pwsh_sha256` | SHA256 for PowerShell 7 MSI installer |
| `git_sha256` | SHA256 for Git for Windows installer |
| `azure_cli_sha256` | SHA256 for Azure CLI MSI installer |

### How to Obtain Checksums

When bumping a software version, obtain the SHA256 from the official release:

| Tool | Checksum Source |
|------|----------------|
| Node.js | `https://nodejs.org/dist/v24.13.1/SHASUMS256.txt` |
| Python | Release page -> Files -> SHA256 column |
| PowerShell 7 | GitHub release -> `hashes.sha256` asset |
| Git | GitHub release notes or compute from download |
| Azure CLI | Microsoft docs for MSI releases |

Example `terraform.tfvars`:

```hcl
node_sha256      = "abc123..."
python_sha256    = "def456..."
pwsh_sha256      = "789ghi..."
git_sha256       = "jkl012..."
azure_cli_sha256 = "mno345..."
```

### Intentional Exceptions

Two tools are intentionally excluded from SHA256 verification:

| Tool | Reason |
|------|--------|
| VS Code | Microsoft auto-update redirector; no stable versioned URL with checksums |
| GitHub Desktop | GitHub's CDN always serves latest; no versioned download with checksums |

Both are signed binaries from Microsoft/GitHub. The risk is accepted because they are user-facing GUI tools (not security-critical infrastructure), pinning them would require maintaining a mirror, and their auto-update mechanisms supersede the installed version on first login.

### npm Package Integrity

npm packages use a different integrity mechanism:

1. **Version pinning**: All packages pinned to exact versions (no `latest`, `^`, or `~`)
2. **SBOM generation**: Full package tree recorded at `C:\ProgramData\ImageBuild\sbom-npm-global.json`
3. **Build audit**: `npm list -g --depth=0` output logged for every build

npm's built-in integrity checking (via `package-lock.json` SHA512 hashes) does not apply to global installs. The version pin + SBOM is the primary control.

### Pinned Versions

Software versions are pinned at two levels: **Terraform variables** for image-build components, and **Intune script versions** for post-provisioning agents.

**Image-build pins (Terraform variables):**

| Package | Version | Pin Method |
|---------|---------|-----------|
| Node.js | v24.13.1 | MSI URL with version in path |
| Python | 3.14.3 | EXE URL with version in path |
| PowerShell 7 | 7.4.13 | MSI URL with version in path |
| Git | 2.53.0 | EXE URL with version in path |
| Azure CLI | 2.83.0 | MSI URL with version in path |

**Post-provisioning pins (Intune script versions):**

| Package | Version | Pin Method |
|---------|---------|-----------|
| OpenSpec | 0.9.1 | `npm install -g @fission-ai/openspec@0.9.1` |
| OpenClaw | 2026.2.14 | `npm install -g openclaw@2026.2.14` |
| Claude Code | 2.1.42 | `npm install -g @anthropic-ai/claude-code@2.1.42` |
| Codex CLI | 0.101.0 | `npm install -g @openai/codex@0.101.0` |

Post-provisioning pins are managed outside Terraform, in Intune platform scripts. This decouples agent and tooling release cycles from image builds — agents can be updated on running Cloud PCs without reprovisioning (see Chapter 25). Always pin to exact versions; never use `latest`.

---

