## Chapter 27: Identity Architecture -- The Secondary User Imperative

### Why Primary User Identity Fails

Running autonomous agents under the primary user's interactive identity (e.g., `CORP\JSmith`) introduces three fatal security flaws:

**1. Identity Conflation and Non-Repudiation.** In all Windows Event Logs, Entra ID sign-in logs, and audit trails, actions performed by the agent are attributed to the human user. If OpenClaw executes a destructive command, the logs show JSmith as the perpetrator. Security teams cannot distinguish between a malicious insider, a compromised account, or a rogue AI agent.

**2. Excessive Privilege Inheritance.** A human user accumulates privileges over time: access to legacy file shares, HR portals, financial systems. An agent running as that user inherits this entire accumulated privilege set. An AI coding assistant needs access to a specific Git repository; it does not need access to the user's email, payroll data, or the CEO's calendar.

**3. SSO Exploitation.** If an agent running in the primary user's session is compromised via prompt injection, the attacker gains access to the user's active SSO tokens (Primary Refresh Token). The attacker can access other corporate resources (SharePoint, Teams, Salesforce) without triggering MFA, as the session is already trusted.

### The Recommended Architecture

![Identity Architecture](../Graphics/Chapter27.png)

```mermaid
graph LR
    subgraph Identities
        USER["kevin@bighatgroup.com\nFull M365 License"]
        AGENT["agent-claude-devteam@\nEntra ID P1 Only"]
    end

    subgraph Cloud PC
        LOGIN["Developer Session\n(Email, Teams, SharePoint)"]
        RUNAS["Agent Process\n(Git Repos Only)"]
    end

    USER -->|signs in| LOGIN
    AGENT -->|RunAs| RUNAS
    LOGIN -->|launches| RUNAS
```

### Identity Options

| Feature | Secondary Entra ID User | Entra Agent ID (Preview) | Local Standard User |
|---|---|---|---|
| **Description** | Standard cloud-only Entra ID account | Specialized identity for AI agents | Local Windows account |
| **Use Case** | Interactive & Cloud access | Autonomous background agents | Strictly local operations |
| **Authentication** | Password + MFA (FIDO2) | Certificate-based | Local password |
| **Auditability** | High -- Entra ID sign-in logs | Very High -- explicit Agent ID logs | Low -- local only |
| **Intune Management** | Full | Full | Limited |
| **Cost** | Requires licence (M365 F3 or Entra P1) | Usage-based (preview) | Free |

### Microsoft Entra Agent ID

Microsoft is developing **Entra Agent ID** to formalize agent identity as a first-class concept in Entra. This section explains the technical architecture so you can evaluate it for your environment and plan a migration from secondary users when it reaches GA.

**Recommendation:** Entra Agent ID is a compelling long-term solution, but its **preview status** should give production-oriented teams pause. Preview features carry no SLA, may introduce breaking changes, and can be deprecated before reaching GA. Organizations with strict change-management or compliance requirements will find it difficult to justify a preview dependency in their identity architecture. Until Entra Agent ID reaches GA with stable APIs, SLA coverage, and a clear licensing model, provision **Secondary Entra ID Users** for agents. This provides immediate segregation and auditability using GA-supported primitives, and the identity is not tied to the Cloud PC. A developer can use the same secondary agent account from their local development machine to authenticate to Azure CLI, Azure DevOps, or any Entra-integrated service, running agent workloads locally while retaining the audit separation and scoped permissions of the dedicated identity. This makes the secondary user approach useful even without Windows 365: it works anywhere `az login` does. When Entra Agent ID reaches GA, migrating from secondary users to Agent IDs should be straightforward — the scoping and Conditional Access patterns are architecturally aligned.

#### The Four-Object Data Model

Agent ID is not a parallel directory — it is an agent-aware specialization of existing Entra object families. Microsoft Graph exposes four core resource types:

| Object | Inherits from | Role |
|--------|--------------|------|
| `agentIdentityBlueprint` | `application` | Template and credential anchor; holds managed identity/FIC/client credentials |
| `agentIdentityBlueprintPrincipal` | `servicePrincipal` | Tenant-local record of the blueprint; exposes app roles and delegated scopes |
| `agentIdentity` | `servicePrincipal` | Primary identity for the running agent; holds no credentials itself |
| `agentUser` | `user` | Optional 1:1 user object for systems that require a user, not a service, identity |

```mermaid
flowchart TD
    B["agentIdentityBlueprint
    inherits application
    owns credentials / FIC / scopes"] --> BP["agentIdentityBlueprintPrincipal
    inherits servicePrincipal
    tenant-local record of blueprint"]
    B --> AI["agentIdentity
    inherits servicePrincipal
    primary identity for agent"]
    AI --> AU["agentUser
    inherits user
    optional 1:1 user object"]

    MI["Managed Identity or Certificate or FIC"] --> B
    BP --> Logs["Audit logs / sign-in logs"]
    AI --> Tokens["Access tokens with appid/azp, oid, tid, xms_* claims"]
    AU --> Tokens
    CA["Conditional Access / custom security attributes / risk policies"] --> AI
    CA --> B
```

The administrative model adds **owners** (technical administrators), **sponsors** (business accountability), and optionally **managers** (for agent user scenarios). Microsoft explicitly positions these as part of agent lifecycle governance, which is why Agent ID is materially different from a plain service principal.

#### Identifier Architecture

A critical detail: for an `agentIdentity`, Microsoft documents that **`id` and `appId` are the same value**. This is a deliberate departure from the classic application/service-principal model, where the application object's `id` (tenant-scoped) differs from its `appId` (globally unique client ID). For agent identities specifically, Microsoft collapses them to simplify agent attribution.

A second subtlety: the `agentIdentityBlueprintId` property on an `agentIdentity` stores the parent blueprint's **`appId`**, not the blueprint object's `id`. If you store only object IDs and later try to correlate child agent identities by `agentIdentityBlueprintId`, you will get mismatches. Always translate to `appId` when navigating the blueprint-to-identity relationship.

| Identifier | Property / claim | Scope | Notes |
|-----------|-----------------|-------|-------|
| Agent identity "ID" | `agentIdentity.id` = `agentIdentity.appId` | Tenant | The closest thing to an "agent ID" — but both properties hold the same value for `agentIdentity` objects specifically |
| Object ID | `id` | Tenant | For classic apps/SPs, differs from `appId`; for `agentIdentity`, equals `appId` |
| Application / client ID | `appId` | Global | The globally unique client identifier; used in token `appid`/`azp` claims |
| Tenant ID | `tenantId` / token `tid` | Tenant | Identifies the Entra tenant, not the agent |
| Principal ID | `properties.principalId` | ARM | Service-principal object ID for managed identities; use for RBAC; use `clientId` in app code |
| Blueprint parent ref | `agentIdentityBlueprintId` on `agentIdentity` | — | Stores the blueprint's **`appId`**, not its `id` |
| Agent-user parent ref | `identityParentId` on `agentUser` | — | Stores the parent agent identity's **object ID** |

#### Token Claims

Agent tokens use standard Entra claims plus agent-specific extension claims. There is no dedicated `agent_id` JWT claim — agent semantics are carried in `xms_*` fact claims:

| Claim | Value | Meaning |
|-------|-------|---------|
| `xms_sub_fct` | `11` | Subject is an `agentIdentity` (autonomous flow) |
| `xms_sub_fct` | `13` | Subject is an `agentUser` (delegated/user-required flow) |
| `xms_act_fct` | `11` | Actor is an `agentIdentity` |
| `xms_par_app_azp` | blueprint `appId` | Parent application attribution — useful for **audit logging only**, not for authorization decisions |
| `idtyp` | `app` | Autonomous agent identity token (app-only) |
| `idtyp` | `user` | Agent user token (user-like subject) |

A token for an autonomous agent identity looks like:

```json
{
  "tid": "00000001-0000-0ff1-ce00-000000000000",
  "idtyp": "app",
  "appid": "aaaaaaaa-1111-2222-3333-444444444444",
  "oid":   "aaaaaaaa-1111-2222-3333-444444444444",
  "xms_act_fct": "11",
  "xms_sub_fct": "11",
  "aud": "f2510d34-8dca-4ab8-a0bc-aaec4d3a3e36"
}
```

Note that `appid` and `oid` are the same value — consistent with the data model above. When the subject switches to an agent user, `idtyp` becomes `user`, `oid` becomes the agent user's object ID, and `xms_sub_fct` shifts to `13`.

**Authorization guidance:** Validate `aud`, `tid`, `iss`, signature, and expiry as normal. Authorize on `oid`, `appid`/`azp`, and Graph relationships. Do not use `xms_par_app_azp` for authorization — Microsoft explicitly says this could inadvertently grant broad access to all agents under the same parent blueprint.

#### Credential Architecture

Agent identities themselves hold **no credentials**. The blueprint is the credential anchor. This shifts credential management upward to the blueprint layer, which is better for rotation, policy, and governance. The blueprint then issues tokens to child agent identities via the `fmi_path` parameter in the token request.

Microsoft's recommended credential types for the blueprint, in order of preference:

1. **Managed identity** (system- or user-assigned) — no secret rotation required
2. **Federated identity credentials (FIC)** — OIDC trust to an external identity provider
3. **Client certificate** — acceptable; requires rotation management
4. **Client secret** — explicitly not recommended for production

#### Migration Path from Secondary Users

When Agent ID reaches GA, the migration from secondary Entra ID users is architecturally straightforward because the underlying controls are the same primitives:

| Secondary user model | Agent ID equivalent |
|---------------------|-------------------|
| Cloud-only Entra ID user account | `agentIdentity` (inherits `servicePrincipal`) |
| Conditional Access policy targeting agent user account | Conditional Access policy targeting `agentIdentity` or its parent blueprint |
| RBAC / app-role assignments on the agent account | App-role assignments on the `agentIdentity` |
| `az login --username agent-claude@...` | Blueprint token → `fmi_path` exchange → agent identity token |
| Sign-in log filter by `SubjectUserName has "agent"` | Sign-in log filter by `agent/agentType eq 'AgentIdentity'` (see Chapter 33) |
| Sponsor/manager tracked out-of-band | `sponsors@odata.bind` and `owners@odata.bind` on the blueprint |

The practical migration sequence: create the blueprint and agent identity in parallel with the existing secondary user account, validate token flows and Conditional Access behavior in a test tenant, then cut over by replacing `az login` credential configuration with the blueprint-to-agent-identity exchange pattern. The secondary user account can remain as a fallback during transition.

### Operational Workflow

#### Setting Up the Agent Identity

1. **Create a cloud-only user** in Entra ID (e.g., `agent-claude-teamA@bighatgroup.com`). Use a naming convention that makes agent accounts immediately identifiable in audit logs.
2. **Assign a minimal licence.** Entra ID P1 or M365 F3, enough for Entra join and Conditional Access, but no Exchange Online, Teams, or SharePoint.
3. **Scope access.** Grant access only to specific Azure DevOps projects, GitHub repositories, or Azure resource groups. Use Entra ID security groups to manage this at scale (e.g., `SG-AgentAccess-TeamA-Repos`).
4. **Configure Conditional Access.** Create a policy targeting the agent account that restricts sign-in to compliant devices and blocks access to non-development resources.
5. **Add to the Windows 365 provisioning policy (Options 2 and 3).** Include the agent account (or its security group) in the same provisioning policy that uses your custom image. The agent account gets its own Cloud PC, provisioned from the same image.

#### The Developer Experience

**Options 2/3: Dedicated agent Cloud PC (recommended for autonomous workflows):**

The developer has two Cloud PCs in their Windows 365 portal:

| Cloud PC | Signed In As | Purpose |
|----------|-------------|---------|
| Primary | `kevin@bighatgroup.com` | Email, Teams, SharePoint, interactive development |
| Agent | `agent-kevin@bighatgroup.com` | Agent-assisted coding, autonomous tasks |

The developer connects to the agent Cloud PC when they want to run extended agent sessions. Because the agent account has no access to email, HR systems, or financial tools, a compromised agent is limited to the developer's code repositories.

### Local Administrator: The Case for Giving Developers the Keys

This will be controversial in some organizations, but for AI agent developer Cloud PCs, **granting local administrator privileges to the user account is the recommended configuration.**

Here's why: AI coding agents are, by design, tools that install packages, compile software, run build systems, and modify system-level configuration. A developer using Claude Code to scaffold a new project will need `npm install`, `pip install`, `dotnet tool install`, and occasionally `winget install`. An agent debugging a Docker issue needs to restart the Docker service. An agent setting up a development database needs to modify Windows Firewall rules.

Locking the developer (and by extension, the agent) out of administrative operations creates constant friction: the agent hits a permissions wall, the developer files a helpdesk ticket, the helpdesk escalates to IT, IT pushes an Intune script that arrives 45 minutes later. This workflow is antithetical to the entire purpose of autonomous coding agents.

**The risk calculus changes when the Cloud PC is network-isolated.** A local administrator on a Cloud PC that:

- Has **no access to the corporate LAN** (RFC 1918 ranges blocked outbound; see Chapter 30)
- Is on an **isolated Microsoft network** with no line-of-sight to on-premises resources
- Has **default deny outbound** with only allowlisted endpoints (Anthropic API, GitHub, npm registry)
- Is **enrolled in Defender for Endpoint** with Attack Surface Reduction rules active (Chapter 31)
- Uses a **dedicated agent identity** (not the user's primary corporate account)

...is a fundamentally different risk from a local administrator on a domain-joined laptop sitting on the corporate network. The blast radius is contained. A compromised agent with local admin on this Cloud PC cannot pivot to the domain controller, cannot access file shares, cannot reach the HR system. It can damage the Cloud PC itself, which can be reprovisioned from the image in minutes.

#### Configuring Local Admin via Windows 365

Windows 365 provisioning policies control whether the user is a local administrator:

1. In the **Intune admin centre**, navigate to **Devices > Windows 365 > Provisioning policies**
2. Edit the provisioning policy for your developer image
3. Under **Additional settings**, set **Local admin** to **Enabled**

This grants the provisioned user local administrator rights on their Cloud PC.

#### The Dedicated Account Makes This Safe

The critical enabler is the **dedicated agent account** from the isolated model. Consider the risk matrix:

| Scenario | Local Admin | Network Isolated | Dedicated Account | Risk Level |
|----------|------------|-----------------|-------------------|------------|
| Primary user on corporate network | ✅ | ❌ | ❌ | 🔴 **Critical** -- compromised agent has admin + corporate access + user's full identity |
| Primary user, network isolated | ✅ | ✅ | ❌ | 🟡 **Medium** -- blast radius contained, but agent actions attributed to the human |
| Dedicated account, network isolated | ✅ | ✅ | ✅ | 🟢 **Low** -- contained blast radius, scoped permissions, clean audit trail |
| Standard user, network isolated | ❌ | ✅ | ✅ | 🟢 **Low** -- maximum restriction, but constant friction for agent workflows |

The sweet spot is **dedicated account + network isolated + local admin**. The agent can do its job without friction, the network prevents lateral movement, the dedicated identity prevents privilege inheritance from the human, and reprovisioning resets the machine to a known-good state.

#### The Licensing Reality

Every user and every agent account requires a **minimum base licence stack of Entra ID P1 + Windows 365 Enterprise + Intune P1**. The developer typically holds a full **Microsoft 365 E3** (which includes Entra P1 and Intune P1) plus **Windows 365 Enterprise**. The agent account options layer on top of this.

**Option 2 (base stack, no M365 E3):** Each agent account needs:

- A standalone **Entra ID P1** licence (~$6/user/month)
- A standalone **Intune P1** licence (~$8/user/month)
- A **Windows 365 Enterprise** licence (varies by SKU, from ~$31/month for 2 vCPU/4 GB to ~$123/month for 8 vCPU/32 GB; recommended: 4 vCPU/16 GB at $66/month)

The agent account does not need M365 E3 because it has no email, Teams, SharePoint, or Office apps.

**Option 3 (full M365):** Each agent account needs:

- A **Microsoft 365 E3** licence (~$36/user/month, includes Entra P1 and Intune P1)
- A **Windows 365 Enterprise** licence for the agent's own Cloud PC

Option 3 is for scenarios where the agent needs to authenticate independently to Microsoft 365 services (Graph API, Teams channels, SharePoint document libraries). The incremental cost over Option 2 is the difference between standalone Entra P1 + Intune P1 and a full M365 E3 licence.

For a team of 10 developers, the incremental cost of Options 2 or 3 is approximately $8,000--$12,000/year depending on the SKU. This is a rounding error compared to the cost of a security incident where a compromised agent with the developer's primary identity exfiltrates source code or accesses sensitive systems.

> **💡 Tip:** The recommended Cloud PC SKU for AI agent workloads is 4 vCPU / 16 GB ($66/user/month). While agents are primarily I/O bound (API calls, file reads), the additional headroom is needed for concurrent agent processes, npm operations, and local language server indexing. A 2 vCPU / 8 GB SKU ($41/user/month) can work for lightweight, single-agent use cases but may constrain heavier workflows.

#### When Standard User Is Appropriate

Not every scenario needs local admin. If the AI agents are used purely for:

- Code review and analysis (read-only)
- Documentation generation
- Chat-based Q&A about the codebase

...then a standard user account is appropriate. The agents don't need to install packages or modify system configuration for these workflows. Reserve local administrator for **active development** scenarios where the agent is building, testing, and deploying code.

---

