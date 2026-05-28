## Chapter 29: Hardening OpenClaw

OpenClaw requires a "Presume Breach" mentality due to its susceptibility to supply chain attacks.

### The ClawHavoc Campaign

The "ClawHavoc" campaign, based on community-reported incidents and security researcher analysis, illustrated the fragility of the OpenClaw skills ecosystem:

- Approximately **12%** of skills in the ClawHub marketplace were reported as malicious (exact figures vary by source)
- Attackers published skills with innocuous names ("Solana Wallet Tracker", "PDF Tools")
- Post-install scripts downloaded the **Atomic Stealer** malware
- The malware gained immediate access to browser data, crypto wallets, and SSH keys

While the precise scope of ClawHavoc is difficult to verify independently, the pattern it represents (supply chain poisoning through community marketplaces) is well-established across the broader software ecosystem (npm, PyPI, VS Code extensions).

### Mitigation: Skill Vetting

1. **Prohibit** installation of skills from the public ClawHub marketplace
2. **Establish** an internal, curated Git repository for approved skills
3. **Integrate** the [Cisco Skill Scanner](https://github.com/cisco-ai-defense/skill-scanner) for static analysis:

```bash
skill-scanner scan --path ./new-skill --output-format json
```

The scanner detects:
- Data exfiltration patterns (`curl`, `requests.post` to unknown domains)
- Prompt injection in markdown descriptions
- Obfuscation (Base64 encoded payloads)
- Suspicious complexity

**Internal Registry Configuration:** Configure OpenClaw to point to your internal skill registry by setting the `skillRegistry` property in the Gateway configuration to your internal registry URL.

**Preventing Registry Changes:** Lock the `skillRegistry` setting using the same managed-settings.json mechanism described in Chapter 28 to prevent developers from switching to the public ClawHub marketplace.

**CI/CD Integration:** Cisco Skill Scanner can be integrated as a CI/CD gate in your internal skill registry pipeline — any skill that fails the scanner is blocked from publishing to the internal registry.

### Persistent Memory Poisoning

OpenClaw maintains persistent memory in files like `SOUL.md` and local databases. An attacker can send an email or commit a repository file containing hidden instructions. When OpenClaw processes this content, the malicious instruction enters its memory and may execute hours or days later, a "sleeper agent" capability that makes forensic attribution extremely difficult.

**Mitigation:**
- Restrict OpenClaw's memory files to read-only access from external sources
- Monitor memory file modifications via Sysmon Event ID 11 (File Create)
- Implement input sanitization for all external content ingested by OpenClaw

**Windows-Specific Mitigations:**

On Windows, `SOUL.md` and `MEMORY.md` live under `C:\ProgramData\OpenClaw\` (or the equivalent ProgramData path for your deployment). Restrict write access using NTFS permissions: grant the OpenClaw service account read access, and reserve write access to the local Administrators group. Verify whether OpenClaw requires write access to these files at runtime — if so, scope the write permission narrowly to the specific paths it modifies.

### WebSocket Security (Architectural Concern)

Prior to OpenClaw version 2026.1.29, the local web dashboard lacked CSRF protection and WebSocket origin validation (CVE-2026-25253). This allowed a malicious website to open a WebSocket connection to OpenClaw's control port on localhost, potentially achieving remote code execution without penetrating the corporate firewall.

**OpenClaw 2026.1.29 and later** include origin validation on WebSocket endpoints and fix the unauthenticated `gatewayUrl` parameter issue. However, defence-in-depth remains essential, as not all deployments will be on the latest version, and similar architectural concerns may arise in future features.

**Mitigation (regardless of version):**
- **Keep OpenClaw updated**: ensure your pinned version is >= 2026.1.29
- Configure OpenClaw to bind strictly to `127.0.0.1`, never `0.0.0.0`
- Use NSGs to block inbound connections to OpenClaw's port from any external source
- Monitor for new security advisories in the OpenClaw changelog

### Entra Agent ID Integration (Forward-Looking)

When Microsoft Entra Agent ID reaches GA, it provides a purpose-built identity model that supersedes the secondary Entra ID user approach described in Chapter 27. This section identifies the architectural questions that determine whether and how OpenClaw can adopt Agent ID in your environment.

> **Note:** Entra Agent ID remains in preview as of this writing. Do not adopt this section's guidance in production until GA is confirmed and the feature carries a Microsoft SLA. Use the secondary Entra ID user model from Chapter 27 until then.

For full Entra Agent ID configuration details, see Chapter 27 (Identity Architecture).

---

