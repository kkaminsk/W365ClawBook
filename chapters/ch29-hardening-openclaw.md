## Chapter 29: Hardening OpenClaw

OpenClaw requires a "Presume Breach" mentality due to its susceptibility to supply chain attacks.

### The ClawHavoc Campaign

The "ClawHavoc" campaign, based on community-reported incidents and security researcher analysis, illustrated the fragility of the OpenClaw skills ecosystem:

- Approximately **12%** of skills in the ClawHub registry were reported as malicious (exact figures vary by source)
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

### Persistent Memory Poisoning

OpenClaw maintains persistent memory in files like `SOUL.md` and local databases. An attacker can send an email or commit a repository file containing hidden instructions. When OpenClaw processes this content, the malicious instruction enters its memory and may execute hours or days later, a "sleeper agent" capability that makes forensic attribution extremely difficult.

**Mitigation:**
- Restrict OpenClaw's memory files to read-only access from external sources
- Monitor memory file modifications via Sysmon Event ID 11 (File Create)
- Implement input sanitization for all external content ingested by OpenClaw

### WebSocket Security (Architectural Concern)

Prior to OpenClaw version 2026.1.29, the local web dashboard lacked CSRF protection and WebSocket origin validation (CVE-2026-25253). This allowed a malicious website to open a WebSocket connection to OpenClaw's control port on localhost, potentially achieving remote code execution without penetrating the corporate firewall.

**OpenClaw 2026.1.29 and later** include origin validation on WebSocket endpoints and fix the unauthenticated `gatewayUrl` parameter issue. However, defence-in-depth remains essential, as not all deployments will be on the latest version, and similar architectural concerns may arise in future features.

**Mitigation (regardless of version):**
- **Keep OpenClaw updated**: ensure your pinned version is >= 2026.1.29
- Configure OpenClaw to bind strictly to `127.0.0.1`, never `0.0.0.0`
- Use NSGs to block inbound connections to OpenClaw's port from any external source
- Monitor for new security advisories in the OpenClaw changelog

---

