# Chapter 26 — Agent Threat Model Research
**Date:** 2026-05-28  
**Purpose:** Deep research to expand Chapter 26 from ~35 lines to a full anchor chapter

---

## Key Frameworks and Sources

### OWASP Two-Layer Model
- **LLM Top 10 (2025):** genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/ — model manipulation
- **Agentic Top 10 (2026):** genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/ — autonomy amplification
- Released Dec 2025 by 100+ industry experts
- ASI01–ASI10 covers Goal Hijack, Tool Misuse, Identity Abuse, Supply Chain, Code Execution, Memory Poisoning, Cascading Trust, Data Boundary Violations, Audit Gaps, Rogue Agents

### MITRE ATLAS
- v5.4.0 (Feb 2026): 16 tactics, 84 techniques, 56 sub-techniques
- Oct 2025: 14 new agent-specific techniques added (with Zenity Labs)
- Feb 2026 additions: "Publish Poisoned AI Agent Tool", "Escape to Host"
- Key techniques: AML.T0054 (Prompt Injection), AML.T0057 (Context Poisoning), AML.T0058 (Memory Manipulation), AML.T0060 (Poisoned Tool), AML.T0061 (Escape to Host)
- Source: armosec.io/blog/mitre-atlas-for-ai-agent-attack-detection/

### ASTRIDE Extension
- Extends STRIDE with "A" category for Agent-Specific Attacks
- Covers prompt injection, unsafe tool invocation, reasoning subversion
- Source: arxiv.org/pdf/2512.04785

### MCP Vulnerabilities
- CVE-2025-49596 (CVSS 9.4): Arbitrary command execution via unauthenticated MCP Inspector
- CVE-2025-68145 (CVSS 6.4): Path validation bypass
- First malicious MCP package: September 2025
- April 2026: Anthropic MCP spec design flaw affecting LettaAI, LangFlow, Windsurf
- NSA advisory on MCP security: nsa.gov/Portals/75/documents/Cybersecurity/CSI_MCP_SECURITY.pdf
- Attack classes: Tool Poisoning, Puppet Attacks, Rug Pull Attacks, Path Validation Bypasses
- Source: darkreading.com/application-security/microsoft-anthropic-mcp-servers-risk-takeovers

### Documented CVEs / Real-World Incidents
- **CVE-2025-53773** — GitHub Copilot configuration manipulation via code comment injection; autoApprove flag abuse
- **CVE-2025-54135** — Cursor AI MCP config attack → RCE without user approval
- **Q4 2025 Inter-Agent Escalation** — GitHub Copilot + Claude rewriting each other's config files
- **ServiceNow Now Assist** — Low-privilege agent delegated to high-privilege peer; exported case file to external URL
- **AIShellJack (2025)** — 314 payloads, 70 MITRE ATT&CK techniques, 55.6–93.3% success rates
- **Cline npm Publish (Feb 2026)** — Untrusted issue input → CI shell access → cache poisoning → credential abuse
- Source: microsoft.com/en-us/security/blog/2026/05/07/prompts-become-shells-rce-vulnerabilities-ai-agent-frameworks/

### Memory Poisoning
- MINJA (NeurIPS 2025, Dong et al.): Query-only memory injection without direct store access
- Email assistant scenario: Malicious "meeting notes" → months of silent data exfiltration
- Payment routing attack: False "approved vendor" in memory → payment redirect
- Plan injections: 3× higher success rate than direct injection; context-chained injections +17.7%
- Source: christian-schneider.net/blog/persistent-memory-poisoning-in-ai-agents/

### Industry Statistics
- 48% of cybersecurity professionals: agentic AI = top 2026 attack vector (Kiteworks)
- 1 in 8 AI breaches linked to agentic systems (HiddenLayer 2026 AI Threat Report)
- 89% YoY surge in AI-enabled attacks; eCrime breakout time: 29 minutes (CrowdStrike / Barracuda)
- 40% of enterprise apps will use AI agents by 2026 (Gartner), up from <5% in 2025
- Only 29% of organizations feel ready to deploy agentic AI securely
- 39% of firms: AI agents reached unauthorized systems in 2025

### Multi-Agent Trust Failures
- Horizontal topologies amplify coordination threats; no central checkpoint
- Shared memory as lateral movement vector
- CSA May 2026: "cascading trust failures" most underaddressed enterprise agent threat
- Source: lyrie.ai/research/research/2026-05-11-09-deepdive-agentic-ai-cve-cascade-trust-boundary-collapse

### Zero Trust for AI Agents
- ZTAI (Zero-Trust Agent Identity): every agent call must be authenticated, authorized, audited independently
- Microsoft "Zero Trust for AI" guidance: March 2026
- CSA Agentic Trust Framework: February 2026
- Source: cloudsecurityalliance.org/blog/2026/02/02/the-agentic-trust-framework-zero-trust-governance-for-ai-agents

### Windows 365 / Agent 365
- Microsoft Agent 365 GA: May 2026 — enterprise control plane for agents
- Windows 365 for Agents: January 2026 — dedicated Cloud PC environments for autonomous agent workflows
- Intune + Entra ID as management and identity layer
- Purview for data access control

---

## Sources List

- [Agentic AI: Biggest Enterprise Security Threat for 2026](https://www.kiteworks.com/cybersecurity-risk-management/agentic-ai-attack-surface-enterprise-security-2026/)
- [HiddenLayer 2026 AI Threat Landscape Report](https://www.hiddenlayer.com/news/hiddenlayer-releases-the-2026-ai-threat-landscape-report-spotlighting-the-rise-of-agentic-ai-and-the-expanding-attack-surface-of-autonomous-systems)
- [Barracuda: Agentic AI Threat Multiplier 2026](https://blog.barracuda.com/2026/02/27/agentic-ai--the-2026-threat-multiplier-reshaping-cyberattacks)
- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/)
- [Lakera: Progressive Breach Model Behind OWASP Agentic Top 10](https://www.lakera.ai/blog/the-progressive-breach-model-behind-the-owasp-top-10-for-agentic-applications)
- [MITRE ATLAS for AI Agent Attack Detection (ARMO)](https://www.armosec.io/blog/mitre-atlas-for-ai-agent-attack-detection/)
- [CSA ATLAS Agentic Gap Analysis (May 2026)](https://labs.cloudsecurityalliance.org/agentic/csa-research-note-atlas-agentic-gap-analysis-20260327/)
- [NSA MCP Security Advisory](https://www.nsa.gov/Portals/75/documents/Cybersecurity/CSI_MCP_SECURITY.pdf)
- [Dark Reading: Microsoft & Anthropic MCP RCE Risk](https://www.darkreading.com/application-security/microsoft-anthropic-mcp-servers-risk-takeovers)
- [MCP Timeline of Breaches (AuthZed)](https://authzed.com/blog/timeline-mcp-breaches)
- [Microsoft Security Blog: Prompts Become Shells](https://www.microsoft.com/en-us/security/blog/2026/05/07/prompts-become-shells-rce-vulnerabilities-ai-agent-frameworks/)
- [AIShellJack Research Paper](https://arxiv.org/pdf/2509.22040)
- [Prompt Injection Attacks on Agentic Coding Assistants (arXiv)](https://arxiv.org/html/2601.17548v1)
- [ASTRIDE Framework (arXiv)](https://arxiv.org/pdf/2512.04785)
- [Christian Schneider: Memory Poisoning in AI Agents](https://christian-schneider.net/blog/persistent-memory-poisoning-in-ai-agents/)
- [Lyrie Research: Trust Boundary Collapse 2026](https://lyrie.ai/research/research/2026-05-11-09-deepdive-agentic-ai-cve-cascade-trust-boundary-collapse)
- [CSA Agentic Trust Framework (Feb 2026)](https://cloudsecurityalliance.org/blog/2026/02/02/the-agentic-trust-framework-zero-trust-governance-for-ai-agents)
- [Microsoft Zero Trust for AI (March 2026)](https://www.microsoft.com/en-us/security/blog/2026/03/19/new-tools-and-guidance-announcing-zero-trust-for-ai/)
- [Windows 365 for Agents Blog](https://blogs.windows.com/windowsexperience/2026/01/22/windows-365-for-agents-the-cloud-pcs-next-chapter/)
