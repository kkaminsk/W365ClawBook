## Chapter 30: Network Segmentation

> **Note:** Outbound network filtering is **optional** and should be weighed against operational impact. AI coding agents routinely fetch documentation, download packages from diverse registries (PyPI, crates.io, Maven Central, NuGet), pull container images, query web APIs, and retrieve context from URLs provided in prompts. A strict allowlist will block these activities until each new endpoint is manually approved, creating ongoing maintenance overhead and slowing agent productivity. If your threat model does not require egress filtering, you can omit the NSG rules below and rely on the lateral-movement blocks and Defender for Endpoint protections described in Chapter 31 instead. The rules below are provided for organisations that need defence-in-depth at the network layer.

### Layered Azure Network Architecture

Azure networking provides two complementary controls for Windows 365 ANC deployments. They operate at different layers and should both be in place for defence-in-depth:

| Layer | Control | Scope | What it does well | What it cannot do |
|---|---|---|---|---|
| **Subnet** | Network Security Group (NSG) | IP/port, service tags | Fast, stateful L4 allow/deny; lateral movement blocking between subnets | Cannot filter by FQDN; cannot inspect payload |
| **Tenant egress** | Azure Firewall | FQDN, threat intelligence, TLS inspection | Blocks by domain name; threat-intel feed blocks known C2; blocks ports NSGs cannot express cleanly | Adds latency; requires UDR to steer traffic; requires Firewall Premium for TLS inspection |

The recommended architecture for ANC environments is to use **NSGs for subnet isolation and lateral-movement blocking** and **Azure Firewall for FQDN-based egress filtering and threat intelligence**. Microsoft's guidance for ANC deployments is to route Windows 365 traffic through Azure Firewall using UDRs and to use FQDN tags and service tags rather than hand-maintained IP lists.

> **Microsoft-hosted networking:** If you use Microsoft-hosted networking instead of ANC, Microsoft manages the network boundary. NSG and Azure Firewall do not apply at the Cloud PC layer in that model. The Windows 365 connectivity is handled by Microsoft's infrastructure, and you inherit its no-inbound, no-cross-Cloud-PC-traffic design without configuration. The host-level controls in Chapter 31 still apply.

### Default Deny Outbound (NSG)

Configure the NSG associated with the Cloud PC subnet to enforce a **default deny outbound** policy, then allowlist only necessary endpoints. NSG rules use service tags and IP/port expressions; they cannot express FQDNs. For FQDN-based filtering, see the Azure Firewall section below.

```text
Priority  Direction  Action  Destination                          Port      Protocol  Purpose
---------------------------------------------------------------------------------------------
# AI Agent API Endpoints
100       Outbound   Allow   api.anthropic.com                    443       TCP       Claude Code / OpenClaw API
110       Outbound   Allow   api.openai.com                       443       TCP       Codex CLI / OpenAI API

# Developer Tool Endpoints
120       Outbound   Allow   github.com                           443       TCP       Git operations
121       Outbound   Allow   *.githubusercontent.com              443       TCP       GitHub raw content / releases
130       Outbound   Allow   registry.npmjs.org                   443       TCP       npm package registry

# Azure Services
140       Outbound   Allow   AzureCloud (service tag)             443       TCP       Azure management plane

# Windows 365 Required Endpoints (see note below)
200       Outbound   Allow   WindowsVirtualDesktop (service tag)  443       TCP       W365 RDP gateway
201       Outbound   Allow   WindowsVirtualDesktop (service tag)  3478      UDP       W365 relayed RDP (STUN/TURN)
210       Outbound   Allow   AzureFrontDoor.Frontend (svc tag)    443       TCP       W365 web client
220       Outbound   Allow   *.manage.microsoft.com               443       TCP       Intune management
230       Outbound   Allow   *.dm.microsoft.com                   443       TCP       Delivery Optimization
240       Outbound   Allow   login.microsoftonline.com            443       TCP       Entra ID authentication
241       Outbound   Allow   login.microsoftonline.com            5671      TCP       Device provisioning / IoT registration
250       Outbound   Allow   *.windowsupdate.com                  443,80    TCP       Windows Update

# Default Deny
4096      Outbound   Deny    *                                    *         *         Block all other traffic
```

> **💡 Important:** Windows 365 Cloud PCs require connectivity to specific Microsoft endpoints for RDP gateway, Intune management, Windows Update, Defender, and Entra ID authentication. The full list of required endpoints is published at [learn.microsoft.com/windows-365/enterprise/requirements-network](https://learn.microsoft.com/windows-365/enterprise/requirements-network). Review this list before deploying your NSG — missing a required endpoint will cause provisioning failures or management gaps. The rules above cover the most critical service tags; consult the published list for the complete set.

> **⚠️ No blanket internet rule:** A previous version of this configuration included a priority-900 rule allowing all outbound HTTPS to `Internet`. That rule has been removed. It defeated the purpose of the NSG by allowing any agent process to reach any HTTPS destination, which eliminates the egress control layer entirely. If your agents require endpoints not listed above (for example, PyPI, crates.io, additional documentation sites, or third-party APIs), add explicit rules at priorities 131–199 for each one. Yes, this creates maintenance overhead as agents discover new endpoints — that overhead is the cost of defence-in-depth. The opening note of this chapter describes how to evaluate whether egress filtering is right for your threat model.

> **⚠️ TLS inspection caveat:** Microsoft explicitly says that Windows 365 required endpoints must **not** be TLS-inspected. If you deploy Azure Firewall Premium with TLS inspection enabled, exclude the Windows 365 service FQDNs from inspection. Inspecting these connections will break provisioning, Intune management, and RDP connectivity.

### Block Lateral Movement

Explicitly deny traffic to private IP ranges (RFC 1918) to prevent a compromised agent from scanning or attacking the internal corporate network:

```text
Priority  Direction  Action  Destination       Port  Protocol
300       Outbound   Deny    10.0.0.0/8        *     *
310       Outbound   Deny    172.16.0.0/12     *     *
320       Outbound   Deny    192.168.0.0/16    *     *
```

### Azure Firewall Egress Control (ANC Deployments)

For Azure Network Connection deployments, Azure Firewall provides FQDN-based egress filtering that NSGs cannot. NSGs allow any destination that matches a service tag; Azure Firewall can enforce allow lists at the domain level and block traffic to rare or unknown destinations — a critical control against agent processes exfiltrating data over HTTPS to attacker-controlled domains.

#### Route Cloud PC Traffic Through Azure Firewall

Use a User-Defined Route (UDR) on the Cloud PC subnet to force all non-local traffic through the Azure Firewall:

```hcl
resource "azurerm_route_table" "cloudpc_egress" {
  name                = "rt-cloudpc-egress"
  location            = var.location
  resource_group_name = var.resource_group_name

  route {
    name                   = "default-to-firewall"
    address_prefix         = "0.0.0.0/0"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = azurerm_firewall.hub.ip_configuration[0].private_ip_address
  }
}

resource "azurerm_subnet_route_table_association" "cloudpc" {
  subnet_id      = azurerm_subnet.cloudpc.id
  route_table_id = azurerm_route_table.cloudpc_egress.id
}
```

> **Important:** When routing through Azure Firewall, add an explicit route to the `WindowsVirtualDesktop` service tag with `next_hop_type = "Internet"` to avoid routing W365 RDP gateway traffic through the firewall hairpin, which adds latency to the user's remote desktop session. Direct-route the RDP session traffic; inspect everything else.

#### Azure Firewall Policy: FQDN Application Rules

Configure Azure Firewall application rules to allow only documented destinations. Microsoft publishes FQDN tags specifically for Windows 365 and related services:

```hcl
resource "azurerm_firewall_policy_rule_collection_group" "cloudpc" {
  name               = "rcg-cloudpc"
  firewall_policy_id = azurerm_firewall_policy.hub.id
  priority           = 300

  application_rule_collection {
    name     = "allow-windows365-required"
    priority = 100
    action   = "Allow"

    rule {
      name = "windows365-fqdn-tag"
      protocols {
        type = "Https"
        port = 443
      }
      source_addresses  = [azurerm_subnet.cloudpc.address_prefixes[0]]
      fqdn_tags         = ["WindowsVirtualDesktop", "MicrosoftIntune", "WindowsUpdate"]
    }

    rule {
      name = "ai-agent-apis"
      protocols {
        type = "Https"
        port = 443
      }
      source_addresses  = [azurerm_subnet.cloudpc.address_prefixes[0]]
      target_fqdns      = [
        "api.anthropic.com",
        "api.openai.com",
        "github.com",
        "*.githubusercontent.com",
        "registry.npmjs.org",
      ]
    }
  }

  network_rule_collection {
    name     = "allow-windows365-network"
    priority = 200
    action   = "Allow"

    rule {
      name                  = "w365-rdp-relay"
      protocols             = ["UDP"]
      source_addresses      = [azurerm_subnet.cloudpc.address_prefixes[0]]
      destination_fqdn_tags = ["WindowsVirtualDesktop"]
      destination_ports     = ["3478"]
    }
  }
}
```

#### Azure Firewall Premium: Threat Intelligence

Enable the Azure Firewall threat intelligence feed to automatically block traffic to known malicious IP addresses and domains. This catches C2 infrastructure that would otherwise blend into normal HTTPS traffic:

```hcl
resource "azurerm_firewall_policy" "hub" {
  name                = "fw-policy-hub"
  resource_group_name = var.resource_group_name
  location            = var.location
  sku                 = "Premium"

  threat_intelligence_mode = "Deny"

  intrusion_detection {
    mode = "Alert"  # Start in Alert; move to Deny after validating no false positives
  }
}
```

Setting `threat_intelligence_mode = "Deny"` blocks outbound connections to Microsoft-classified malicious destinations without any manual rule maintenance. This is particularly relevant for agent runtimes because a compromised skill or prompt-injected command may attempt to reach an attacker domain — threat intelligence blocks that even if the FQDN is not yet on your explicit deny list.

### NSG Flow Logs and Network Monitoring

NSG rules block traffic silently unless you enable flow logging. Enable NSG flow logs on the Cloud PC subnet so you have a record of allowed and denied flows for incident response and baseline analysis:

```hcl
resource "azurerm_network_watcher_flow_log" "cloudpc" {
  network_watcher_name = azurerm_network_watcher.main.name
  resource_group_name  = var.resource_group_name
  name                 = "flowlog-cloudpc-nsg"

  network_security_group_id = azurerm_network_security_group.cloudpc.id
  storage_account_id        = azurerm_storage_account.flowlogs.id
  enabled                   = true
  version                   = 2  # Version 2 includes byte/packet counts

  retention_policy {
    enabled = true
    days    = 30
  }

  traffic_analytics {
    enabled               = true
    workspace_id          = azurerm_log_analytics_workspace.security.workspace_id
    workspace_region      = var.location
    workspace_resource_id = azurerm_log_analytics_workspace.security.id
    interval_in_minutes   = 10
  }
}
```

With Traffic Analytics enabled, flow data lands in Log Analytics and becomes queryable alongside Defender for Endpoint telemetry. This matters for agent workloads: if a Cloud PC is exfiltrating data through an allowed HTTPS destination, flow logs capture the destination IP and byte count even if the application layer is encrypted.

Azure Firewall also writes diagnostic logs to Log Analytics. Use the `AzureDiagnostics` table with `Category == "AzureFirewallApplicationRule"` or `"AzureFirewallNetworkRule"` to audit FQDN-level allow/deny decisions across the fleet.

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

