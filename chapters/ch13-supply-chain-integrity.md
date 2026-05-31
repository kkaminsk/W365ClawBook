## Chapter 13: Supply Chain Integrity

### The Problem

The image build process downloads software installers and npm packages from the public internet. Without integrity verification, a compromised upstream release (supply chain attack) would be silently baked into every provisioned Windows 365 developer image.

This is not a theoretical risk. Community advisories have flagged multiple malicious skills in the ClawHub marketplace — a pattern consistent with supply-chain attacks documented against npm and PyPI, and the primary motivation for the controls in this chapter.

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
| Git | GitHub release notes or compute from download (Computing checksums from your own download is a last resort; pre-computed hashes for Git for Windows are published on the project's GitHub Releases page.) |
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

Add `npm audit --audit-level=high` to your build validation script and cross-reference installed global packages against your SBOM (Software Bill of Materials) after installation.

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
| OpenClaw | 2026.x.x (verify current stable at npmjs.com/package/openclaw before build) | `npm install -g openclaw@<version>` |
| Claude Code | 2.1.x (verify current stable at npmjs.com/package/@anthropic-ai/claude-code before build) | `npm install -g @anthropic-ai/claude-code@<version>` |
| Codex CLI | 0.101.0 | `npm install -g @openai/codex@0.101.0` |

Post-provisioning pins are managed outside Terraform, in Intune platform scripts. This decouples agent and tooling release cycles from image builds — agents can be updated on running Cloud PCs without reprovisioning (see Chapter 25). Always pin to exact versions; never use `latest`.

### The Second-Order Supply Chain Threat: Agent-Driven Package Installs

The controls in this chapter verify the integrity of binaries downloaded **at image build time**. A separate supply chain surface requires different controls: **npm packages installed by agents during coding tasks**.

In August 2025, eight malicious Nx and Nx Powerpack packages were published to npm with `postinstall` scripts that directly invoked Claude Code, Gemini CLI, and Amazon Q CLI using unsafe flags to scan for secrets (documented by Snyk). The packages were live for 5 hours 20 minutes. This attack bypasses all build-time SHA256 verification because the poisoned package is downloaded after provisioning, during a live agent session.

**Three mitigations for this threat:**

1. **Mandatory Azure Artifacts proxy** (Chapter 30) — route all npm, PyPI, and other registry traffic through your Azure Artifacts instance. This creates an audit trail of every package download during agent sessions and allows blocking of packages not in your approved feed.

2. **Block postinstall scripts globally** — add `.npmrc` configuration to the image and to the managed OpenClaw configuration:
   ```
   # C:\ProgramData\npm\etc\.npmrc (machine-wide)
   ignore-scripts=true
   ```
   Add an exception process for packages that legitimately require postinstall (document each exception in your internal skill registry).

3. **OWASP Agentic Skills Top 10 classification** — use the OWASP Agentic Skills Top 10 (`owasp.org/www-project-agentic-skills-top-10`) as the risk classification framework when evaluating any agent-installable package, not only OpenClaw skills.

### Skill Scanner and Vendor-Neutral Risk Classification

The Cisco AI Defense skill scanner provides automated scanning of OpenClaw marketplace skills for known malicious patterns and policy violations. In addition to the Cisco AI Defense skill scanner, the **OWASP Agentic Skills Top 10** (`owasp.org/www-project-agentic-skills-top-10`) provides a formal, vendor-neutral taxonomy for agent skill and plugin supply-chain risk. Use it as the classification framework for your internal skill vetting decisions in conjunction with automated scanning.

---

