## Chapter 30: Network Segmentation

> **Note:** Outbound network filtering is **optional** and should be weighed against operational impact. AI coding agents routinely fetch documentation, download packages from diverse registries (PyPI, crates.io, Maven Central, NuGet), pull container images, query web APIs, and retrieve context from URLs provided in prompts. A strict allowlist will block these activities until each new endpoint is manually approved, creating ongoing maintenance overhead and slowing agent productivity. If your threat model does not require egress filtering, you can omit the NSG rules below and rely on the lateral-movement blocks and Defender for Endpoint protections described in Chapter 31 instead. The rules below are provided for organisations that need defence-in-depth at the network layer.

### Default Deny Outbound

Configure the NSG associated with the Cloud PC subnet to enforce a **default deny outbound** policy, then allowlist only necessary endpoints:

```text
Priority  Direction  Action  Destination                          Port    Protocol  Purpose
---------------------------------------------------------------------------------------------
# AI Agent API Endpoints
100       Outbound   Allow   api.anthropic.com                    443     TCP       Claude Code / OpenClaw API
110       Outbound   Allow   api.openai.com                       443     TCP       Codex CLI / OpenAI API

# Developer Tool Endpoints
120       Outbound   Allow   github.com                           443     TCP       Git operations
121       Outbound   Allow   *.githubusercontent.com              443     TCP       GitHub raw content / releases
130       Outbound   Allow   registry.npmjs.org                   443     TCP       npm package registry

# Azure Services
140       Outbound   Allow   AzureCloud (service tag)             443     TCP       Azure management plane

# Windows 365 Required Endpoints (see note below)
200       Outbound   Allow   WindowsVirtualDesktop (service tag)  443     TCP       W365 RDP gateway
210       Outbound   Allow   AzureFrontDoor.Frontend (svc tag)    443     TCP       W365 web client
220       Outbound   Allow   *.manage.microsoft.com               443     TCP       Intune management
230       Outbound   Allow   *.dm.microsoft.com                   443     TCP       Delivery Optimization
240       Outbound   Allow   login.microsoftonline.com            443     TCP       Entra ID authentication
250       Outbound   Allow   *.windowsupdate.com                  443,80  TCP       Windows Update

# Default Deny
4096      Outbound   Deny    *                                    *       *         Block all other traffic
```

> **💡 Important:** Windows 365 Cloud PCs require connectivity to specific Microsoft endpoints for RDP gateway, Intune management, Windows Update, Defender, and Entra ID authentication. The full list of required endpoints is published at [learn.microsoft.com/windows-365/enterprise/requirements-network](https://learn.microsoft.com/windows-365/enterprise/requirements-network). Review this list before deploying your NSG — missing a required endpoint will cause provisioning failures or management gaps. The rules above cover the most critical service tags; consult the published list for the complete set.

> **⚠️ No blanket internet rule:** A previous version of this configuration included a priority-900 rule allowing all outbound HTTPS to `Internet`. That rule has been removed. It defeated the purpose of the NSG by allowing any agent process to reach any HTTPS destination, which eliminates the egress control layer entirely. If your agents require endpoints not listed above (for example, PyPI, crates.io, additional documentation sites, or third-party APIs), add explicit rules at priorities 131–199 for each one. Yes, this creates maintenance overhead as agents discover new endpoints — that overhead is the cost of defence-in-depth. The opening note of this chapter describes how to evaluate whether egress filtering is right for your threat model.

### Block Lateral Movement

Explicitly deny traffic to private IP ranges (RFC 1918) to prevent a compromised agent from scanning or attacking the internal corporate network:

```text
Priority  Direction  Action  Destination       Port  Protocol
300       Outbound   Deny    10.0.0.0/8        *     *
310       Outbound   Deny    172.16.0.0/12     *     *
320       Outbound   Deny    192.168.0.0/16    *     *
```

### Localhost Binding

Configure OpenClaw to bind its management interface strictly to `127.0.0.1`:

```json
{
  "gateway": {
    "mode": "local",
    "port": 18789,
    "host": "127.0.0.1"
  }
}
```

Never bind to `0.0.0.0`, which would expose the control dashboard to the network.

---

