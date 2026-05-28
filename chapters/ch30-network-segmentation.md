## Chapter 30: Network Segmentation

> **Note:** Outbound network filtering is **optional** and should be weighed against operational impact. AI coding agents routinely fetch documentation, download packages from diverse registries (PyPI, crates.io, Maven Central, NuGet), pull container images, query web APIs, and retrieve context from URLs provided in prompts. A strict allowlist will block these activities until each new endpoint is manually approved, creating ongoing maintenance overhead and slowing agent productivity. If your threat model does not require egress filtering, you can omit the NSG rules below and rely on the lateral-movement blocks and Defender for Endpoint protections described in Chapter 31 instead. The rules below are provided for organisations that need defence-in-depth at the network layer.

> **⚠️ Azure Egress Change — March 31, 2026:** Azure no longer provides default outbound internet access for newly created Virtual Networks. Windows 365 customers using Azure Network Connection (ANC) must now configure explicit outbound connectivity — a NAT Gateway, Azure Firewall, or load balancer with outbound rules — before Cloud PCs can reach any internet endpoint. If your ANC VNet was created after March 31, 2026, apply this chapter's egress configuration before provisioning Cloud PCs. VNets created before this date retain their default outbound access until Microsoft completes the migration.

### Layered Azure Network Architecture

Azure networking provides two complementary controls for Windows 365 Azure Network Connection (ANC) deployments. They operate at different layers and should both be in place for defence-in-depth:

| Layer | Control | Scope | What it does well | What it cannot do |
|---|---|---|---|---|
| **Subnet** | Network Security Group (NSG) | IP/port, service tags | Fast, stateful L4 allow/deny; lateral movement blocking between subnets | Cannot filter by FQDN; cannot inspect payload |
| **Tenant egress** | Azure Firewall | FQDN, threat intelligence, TLS inspection | Blocks by domain name; threat-intel feed blocks known C2; blocks ports NSGs cannot express cleanly | Adds latency; requires UDR to steer traffic; requires Firewall Premium for TLS inspection |
| **Host** | Windows Firewall (Intune) | Process, port, direction | Blocks intra-host port exposure (e.g., OpenClaw port 18789); complements NSG for loopback traffic | Cannot filter by destination FQDN; cannot inspect TLS |
| **DNS** | Azure Firewall DNS Proxy | DNS resolution | Forces all DNS through the firewall; enables FQDN filtering in network rules; detects DNS tunneling | Cannot decrypt DoH (port 443); requires client DNS config pointing to firewall |

The recommended architecture for ANC environments is to use **NSGs for subnet isolation and lateral-movement blocking**, **Azure Firewall for FQDN-based egress filtering and threat intelligence**, **Azure Firewall DNS Proxy for DNS security**, and **Intune-deployed Windows Firewall rules for host-level controls**. Microsoft's Zero Trust for AI guidance (March 2026) requires that traffic filtering and segmentation be applied consistently before access is granted to any network resource, including communications between AI services.

> **Microsoft-hosted networking:** If you use Microsoft-hosted networking instead of ANC, Microsoft manages the network boundary. NSG and Azure Firewall do not apply at the Cloud PC layer in that model. The Windows 365 connectivity is handled by Microsoft's infrastructure, and you inherit its no-inbound, no-cross-Cloud-PC-traffic design without configuration. The host-level Windows Firewall controls in this chapter still apply.

### Default Deny Outbound (NSG)

Configure the NSG associated with the Cloud PC subnet to enforce a **default deny outbound** policy, then allowlist only necessary endpoints. NSG rules use service tags and IP/port expressions; they cannot express FQDNs. For FQDN-based filtering, see the Azure Firewall section below.

> **Note:** Azure NSG rules cannot match on FQDNs — they accept IP addresses and service tags only. Use Azure Firewall FQDN tags, Application Gateway, or a Network Virtual Appliance (NVA) for FQDN-based egress filtering. The FQDN values in this table are provided as reference for configuring those solutions alongside NSGs.

```text
Priority  Direction  Action  Destination                          Port      Protocol  Purpose
---------------------------------------------------------------------------------------------
# AI Agent API Endpoints
100       Outbound   Allow   api.anthropic.com                    443       TCP       Claude Code / OpenClaw API
110       Outbound   Allow   api.openai.com                       443       TCP       Codex CLI / OpenAI API

# Developer Tool Endpoints
120       Outbound   Allow   github.com                           443       TCP       Git operations
121       Outbound   Allow   *.githubusercontent.com              443       TCP       GitHub raw content / releases

# Package Registries (see Azure Artifacts proxy section for alternative)
130       Outbound   Allow   registry.npmjs.org                   443       TCP       npm package registry
131       Outbound   Allow   pypi.org                             443       TCP       Python package index
132       Outbound   Allow   files.pythonhosted.org               443       TCP       PyPI file downloads
133       Outbound   Allow   crates.io                            443       TCP       Rust package registry
134       Outbound   Allow   static.crates.io                     443       TCP       Rust crate downloads
135       Outbound   Allow   api.nuget.org                        443       TCP       .NET package registry
136       Outbound   Allow   registry-1.docker.io                 443       TCP       Docker Hub images
137       Outbound   Allow   auth.docker.io                       443       TCP       Docker Hub authentication
138       Outbound   Allow   repo1.maven.org                      443       TCP       Maven Central (Java)

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

# Block Protocol Relay Paths (see SMB/NTLM section below)
300       Outbound   Deny    *                                    445       TCP       Block SMB — NTLM relay prevention
301       Outbound   Deny    *                                    139       TCP       Block NetBIOS session
302       Outbound   Deny    *                                    137-138   UDP       Block NetBIOS name/datagram
303       Outbound   Deny    *                                    853       TCP       Block DNS over TLS (DoT)
304       Outbound   Deny    *                                    5985      TCP       Block WinRM HTTP
305       Outbound   Deny    *                                    5986      TCP       Block WinRM HTTPS

# Lateral Movement Blocks (RFC 1918)
310       Outbound   Deny    10.0.0.0/8                           *         *         Block corporate LAN access
311       Outbound   Deny    172.16.0.0/12                        *         *         Block corporate LAN access
312       Outbound   Deny    192.168.0.0/16                       *         *         Block corporate LAN access

# Default Deny
4096      Outbound   Deny    *                                    *         *         Block all other traffic
```

> **💡 Important:** Windows 365 Cloud PCs require connectivity to specific Microsoft endpoints for RDP gateway, Intune management, Windows Update, Defender, and Entra ID authentication. The full list of required endpoints is published at [learn.microsoft.com/windows-365/enterprise/requirements-network](https://learn.microsoft.com/windows-365/enterprise/requirements-network). Review this list before deploying your NSG — missing a required endpoint will cause provisioning failures or management gaps.

> **⚠️ No blanket internet rule:** A previous version of this configuration included a priority-900 rule allowing all outbound HTTPS to `Internet`. That rule has been removed. It defeated the purpose of the NSG by allowing any agent process to reach any HTTPS destination. If your agents require endpoints not listed above, add explicit rules at priorities 139–199 for each one. The opening note of this chapter describes how to evaluate whether egress filtering is right for your threat model.

> **⚠️ TLS inspection caveat:** Microsoft explicitly says that Windows 365 required endpoints must **not** be TLS-inspected. If you deploy Azure Firewall Premium with TLS inspection enabled, exclude the Windows 365 service FQDNs from inspection. Inspecting these connections will break provisioning, Intune management, and RDP connectivity.

---

### SMB, NTLM Relay, and Protocol Blocking

NTLM relay remains one of the most consistently exploited lateral movement paths in enterprise environments. CISA added CVE-2025-24054 (Windows NTLM hash disclosure) to its Known Exploited Vulnerabilities catalog in March 2025 following confirmed active exploitation. The threat is directly relevant to this deployment: Chapter 28 described how a prompt injection that causes Claude Code or OpenClaw to access a UNC path (e.g., `\\attacker.example.com\share`) triggers automatic NTLM hash transmission from the Windows authentication subsystem — a kernel-level operation that bypasses all application-level controls.

Network-layer blocking is therefore a required defense-in-depth layer, not optional. The NSG rules at priorities 300–305 above implement this blocking. This section explains the rationale for each rule.

#### Block SMB Egress (TCP 445)

SMB (Server Message Block) on TCP 445 is the primary NTLM relay coercion channel. When a Windows process accesses a UNC path on an external IP, Windows sends an NTLM authentication attempt to that IP on port 445. Blocking outbound TCP 445 from the Cloud PC subnet prevents the hash from reaching the attacker's server:

```text
300  Outbound  Deny  *  445  TCP  Block SMB egress — NTLM relay prevention
```

**Impact:** Developers on these Cloud PCs cannot access SMB file shares over the internet. Internal SMB shares on the corporate network are already blocked by the RFC 1918 deny rules.

#### Block NetBIOS (TCP 139, UDP 137–138)

NetBIOS provides a legacy authentication coercion path on TCP 139 and UDP 137–138. Block all three:

```text
301  Outbound  Deny  *  139     TCP  Block NetBIOS session
302  Outbound  Deny  *  137-138 UDP  Block NetBIOS name/datagram
```

#### Block WinRM (TCP 5985/5986)

Windows Remote Management (WinRM) on ports 5985 (HTTP) and 5986 (HTTPS) enables lateral movement via PowerShell remoting. An agent process that establishes a WinRM session to a remote host can execute arbitrary PowerShell on that host:

```text
304  Outbound  Deny  *  5985  TCP  Block WinRM HTTP
305  Outbound  Deny  *  5986  TCP  Block WinRM HTTPS
```

#### WebDAV Over HTTPS: FQDN-Layer Mitigation

WebDAV over HTTPS (port 443) cannot be blocked at the NSG layer without breaking all HTTPS traffic. The WebClient service uses HTTP/HTTPS for WebDAV, so a WebDAV-based NTLM coercion attack arrives on the same port as legitimate web traffic. Mitigation at this layer is via two complementary controls:

1. **Disable the WebClient service** on the Cloud PC via Intune (Chapter 32):
   ```powershell
   Set-Service -Name WebClient -StartupType Disabled -Status Stopped
   ```
2. **Azure Firewall FQDN deny rules** for known WebDAV tunneling patterns — non-allowlisted HTTPS destinations are blocked by the default deny rule at the firewall, which catches WebDAV coercion to external attacker servers.

---

### DNS Security Layer

DNS is the most commonly overlooked exfiltration and C2 channel in agent deployments. A compromised skill or prompt-injected command can use DNS tunneling to exfiltrate data — encoding information in DNS query hostnames and receiving commands in DNS responses — even when all outbound HTTPS is blocked. DNS over HTTPS (DoH) and DNS over TLS (DoT) further obscure this channel by encrypting DNS traffic.

#### Azure Firewall DNS Proxy

Enabling Azure Firewall as a DNS proxy is a **prerequisite for FQDN filtering in network rules** — without it, Azure Firewall cannot resolve FQDNs in network rules and falls back to IP-only matching. This is a frequently missed configuration step.

```hcl
resource "azurerm_firewall_policy" "hub" {
  name                = "fw-policy-hub"
  resource_group_name = var.resource_group_name
  location            = var.location
  sku                 = "Premium"

  dns {
    proxy_enabled = true
    servers       = ["168.63.129.16"]  # Azure platform DNS resolver
  }

  threat_intelligence_mode = "Deny"
}
```

With DNS proxy enabled, configure the Cloud PC subnet's DNS server setting to point to the Azure Firewall's private IP address. All DNS queries from Cloud PCs flow through the firewall, where they are resolved using the configured upstream servers and logged.

#### Force DNS Through the Firewall

Update the VNet DNS server setting to send all DNS traffic to the Azure Firewall:

```hcl
resource "azurerm_virtual_network" "cloudpc" {
  # ... other settings
  dns_servers = [azurerm_firewall.hub.ip_configuration[0].private_ip_address]
}
```

This forces all DNS queries from Cloud PCs through the Azure Firewall DNS proxy. Direct DNS to any other server (Google 8.8.8.8, Cloudflare 1.1.1.1) is blocked by the default deny NSG rule, since port 53 from external DNS servers will not be in the allow list.

#### Block DNS over TLS (DoT) — Port 853

DNS over TLS on port 853 is the simplest encrypted DNS bypass to block — the port is exclusively used for DoT and has no other legitimate use in this environment:

```text
303  Outbound  Deny  *  853  TCP  Block DNS over TLS
```

This rule is already included in the NSG table above. It has zero false-positive risk.

#### Detect DNS over HTTPS (DoH)

DNS over HTTPS uses port 443 and is therefore indistinguishable from normal HTTPS traffic at the NSG/port layer. Mitigation requires blocking known DoH resolver FQDNs at the Azure Firewall application rule layer:

```hcl
application_rule_collection {
  name     = "block-doh-resolvers"
  priority = 50   # High priority — evaluated before allow rules
  action   = "Deny"

  rule {
    name = "block-known-doh"
    protocols { type = "Https"; port = 443 }
    source_addresses = [azurerm_subnet.cloudpc.address_prefixes[0]]
    target_fqdns = [
      "dns.google",
      "dns.cloudflare.com",
      "mozilla.cloudflare-dns.com",
      "doh.opendns.com",
      "doh.cleanbrowsing.org",
      "dns10.quad9.net",
      "doh.dns.sb",
    ]
  }
}
```

The default-deny posture at the Azure Firewall catches DoH to unknown resolvers — only allow-listed FQDNs pass the firewall, so an agent attempting to reach an unlisted DoH endpoint is blocked by the catch-all deny.

#### DNS Tunneling Detection

Enable Microsoft Defender for DNS on the Cloud PC subscription to detect DNS tunneling patterns:

```hcl
resource "azurerm_security_center_subscription_pricing" "dns" {
  tier          = "Standard"
  resource_type = "Dns"
}
```

Defender for DNS raises alerts when it detects:
- High-volume DNS queries to a single domain (tunneling indicator)
- Unusually long DNS query names (data encoded in subdomains)
- DNS queries to domains with high entropy names (DGA — domain generation algorithm)
- DNS queries to known C2 infrastructure

Route these alerts to Microsoft Sentinel (Chapter 33) for correlation with other agent behavioral signals.

---

### Azure Firewall Egress Control (ANC Deployments)

For Azure Network Connection deployments, Azure Firewall provides FQDN-based egress filtering that NSGs cannot. NSGs allow any destination that matches a service tag; Azure Firewall can enforce allow lists at the domain level and block traffic to rare or unknown destinations — a critical control against agent processes exfiltrating data over HTTPS to attacker-controlled domains.

#### Route Cloud PC Traffic Through Azure Firewall

Use a User-Defined Route (UDR) on the Cloud PC subnet to force all non-local traffic through the Azure Firewall:

> **Note:** These examples assume a hub-and-spoke network topology where hub resources (VNet, subnets, route tables) already exist. See the companion W365Claw repository for the complete hub deployment module.

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
    name     = "block-doh-resolvers"
    priority = 50
    action   = "Deny"

    rule {
      name = "block-known-doh"
      protocols { type = "Https"; port = 443 }
      source_addresses = [azurerm_subnet.cloudpc.address_prefixes[0]]
      target_fqdns = [
        "dns.google", "dns.cloudflare.com", "mozilla.cloudflare-dns.com",
        "doh.opendns.com", "doh.cleanbrowsing.org", "dns10.quad9.net",
      ]
    }
  }

  application_rule_collection {
    name     = "allow-windows365-required"
    priority = 100
    action   = "Allow"

    rule {
      name = "windows365-fqdn-tag"
      protocols { type = "Https"; port = 443 }
      source_addresses  = [azurerm_subnet.cloudpc.address_prefixes[0]]
      fqdn_tags         = ["WindowsVirtualDesktop", "MicrosoftIntune", "WindowsUpdate"]
    }

    rule {
      name = "ai-agent-apis"
      protocols { type = "Https"; port = 443 }
      source_addresses  = [azurerm_subnet.cloudpc.address_prefixes[0]]
      target_fqdns      = [
        "api.anthropic.com",
        "api.openai.com",
        "github.com",
        "*.githubusercontent.com",
        "registry.npmjs.org",
        "pypi.org",
        "files.pythonhosted.org",
        "crates.io",
        "static.crates.io",
        "api.nuget.org",
        "registry-1.docker.io",
        "auth.docker.io",
        "repo1.maven.org",
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

#### Azure Firewall Premium: Threat Intelligence and IDPS

Enable the Azure Firewall threat intelligence feed to automatically block traffic to known malicious IP addresses and domains. This catches C2 infrastructure that would otherwise blend into normal HTTPS traffic:

```hcl
resource "azurerm_firewall_policy" "hub" {
  name                = "fw-policy-hub"
  resource_group_name = var.resource_group_name
  location            = var.location
  sku                 = "Premium"

  dns {
    proxy_enabled = true
    servers       = ["168.63.129.16"]
  }

  threat_intelligence_mode = "Deny"

  intrusion_detection {
    mode = "Alert"  # Start in Alert; move to Deny after validating no false positives
  }
}
```

Setting `threat_intelligence_mode = "Deny"` blocks outbound connections to Microsoft-classified malicious destinations without any manual rule maintenance. This is particularly relevant for agent runtimes because a compromised skill or prompt-injected command may attempt to reach an attacker domain — threat intelligence blocks that even if the FQDN is not yet on your explicit deny list.

The Intrusion Detection and Prevention System (IDPS) in `Alert` mode logs signature matches for known attack patterns without blocking traffic during the baseline period. After 30 days of baselining, promote to `Deny` mode. IDPS in Deny mode blocks prompt injection delivery payloads observed at the network layer and known malware communication patterns.

---

### Package Registry Egress: Azure Artifacts as a Private Proxy

The opening note acknowledges that AI coding agents need access to multiple package registries (PyPI, crates.io, NuGet, Docker Hub, Maven Central). Adding all of these registries to the NSG allowlist widens the egress surface and does nothing to protect against compromised packages.

**Azure Artifacts provides a better architecture:** configure Azure Artifacts upstream sources to proxy each public registry. Agents request packages from the Azure Artifacts feed; the feed fetches from the public registry on first request and caches the result. Subsequent requests are served from the Azure Artifacts cache without hitting the public registry.

This provides two security benefits:
1. **Reduced egress surface:** the NSG allowlist contains only the Azure Artifacts endpoint (`*.pkgs.visualstudio.com`, `*.feeds.visualstudio.com`) instead of eight separate public registry domains
2. **Supply chain integrity:** packages are cached in your Azure subscription; an outage or compromise of the public registry does not immediately affect builds

#### Azure Artifacts Feed Configuration (Terraform)

```hcl
resource "azuredevops_feed" "agent_packages" {
  name    = "agent-packages"
  project = azuredevops_project.main.id
}

# Configure upstream sources for each ecosystem
resource "azuredevops_feed_upstream" "pypi" {
  feed_id            = azuredevops_feed.agent_packages.id
  name               = "PyPI"
  protocol           = "pypi"
  upstream_source_id = "https://pypi.org/simple/"
}

resource "azuredevops_feed_upstream" "npm" {
  feed_id            = azuredevops_feed.agent_packages.id
  name               = "npmjs"
  protocol           = "npm"
  upstream_source_id = "https://registry.npmjs.org"
}

resource "azuredevops_feed_upstream" "nuget" {
  feed_id            = azuredevops_feed.agent_packages.id
  name               = "NuGet.org"
  protocol           = "nuget"
  upstream_source_id = "https://api.nuget.org/v3/index.json"
}
```

The LiteLLM PyPI compromise (February 2026 — malicious code deploying credential harvesting and Kubernetes lateral movement across downstream builds) demonstrated that direct registry access means a compromised upstream package reaches developer machines immediately. Azure Artifacts' caching introduces a delay that allows time for compromise detection before cached packages proliferate.

> After configuring Azure Artifacts, update the NSG allowlist: replace the individual registry entries (priorities 131–138) with a single rule for your Azure DevOps organization's domain. Also update the `pip`, `npm`, and `cargo` configuration files in the Cloud PC image to point to the Azure Artifacts feed URL by default.

---

### Windows Firewall: Host-Level Rules

NSG rules block traffic at the Azure network layer. Windows Firewall rules on each Cloud PC block traffic at the host level — including intra-host traffic that NSGs cannot control, and providing defense-in-depth for traffic that reaches the VM before the NSG rule fires.

Deploy these rules via Intune PowerShell remediation (Chapter 32) targeting the Cloud PC device group:

```powershell
# Block inbound on OpenClaw gateway port from non-loopback sources
New-NetFirewallRule `
  -DisplayName "OpenClaw Gateway - Block Non-Loopback Inbound" `
  -Direction Inbound -Protocol TCP -LocalPort 18789 `
  -RemoteAddress Any -Action Block -Profile Any

New-NetFirewallRule `
  -DisplayName "OpenClaw Gateway - Allow Loopback" `
  -Direction Inbound -Protocol TCP -LocalPort 18789 `
  -RemoteAddress 127.0.0.1 -Action Allow -Profile Any

# Block SMB inbound (defense-in-depth — NSG also blocks this)
New-NetFirewallRule `
  -DisplayName "Block SMB Inbound" `
  -Direction Inbound -Protocol TCP -LocalPort 445 `
  -Action Block -Profile Any

# Block WinRM inbound
New-NetFirewallRule `
  -DisplayName "Block WinRM Inbound" `
  -Direction Inbound -Protocol TCP -LocalPort 5985,5986 `
  -Action Block -Profile Any

# Block NetBIOS inbound
New-NetFirewallRule `
  -DisplayName "Block NetBIOS Inbound" `
  -Direction Inbound -Protocol TCP -LocalPort 139 `
  -Action Block -Profile Any

New-NetFirewallRule `
  -DisplayName "Block NetBIOS UDP Inbound" `
  -Direction Inbound -Protocol UDP -LocalPort 137,138 `
  -Action Block -Profile Any
```

These rules complement the NSG-level blocks. NSGs operate at the Azure SDN layer and provide subnet-level enforcement; Windows Firewall rules operate at the OS network stack and protect against intra-host process-to-process communication that never leaves the VM.

---

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

With Traffic Analytics enabled, flow data lands in Log Analytics and becomes queryable alongside Defender for Endpoint telemetry and the agent OTel streams from Chapters 28 and 29.

#### Agent-Specific Network Monitoring Queries

Configure Microsoft Sentinel (Chapter 33) analytics rules based on these KQL patterns targeting agent workload network behavior:

```kql
// Large outbound data transfer from Cloud PC subnet — exfiltration indicator
AzureNetworkAnalytics_CL
| where SubType_s == "FlowLog"
| where SrcIP_s startswith "10.x.x."  // Cloud PC subnet
| where FlowDirection_s == "O"         // Outbound
| where BytesSentD_d > 10000000        // >10 MB in a single flow
| where DestPort_d == 443
| project TimeGenerated, SrcIP_s, DestIP_s, DestPort_d, BytesSentD_d, FlowStatus_s

// Denied flow to RFC 1918 — lateral movement attempt
AzureNetworkAnalytics_CL
| where SubType_s == "FlowLog"
| where SrcIP_s startswith "10.x.x."  // Cloud PC subnet
| where FlowStatus_s == "D"           // Denied
| where DestIP_s matches regex @"^(10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\.)"
| project TimeGenerated, SrcIP_s, DestIP_s, DestPort_d, NSGList_s

// Denied flow to port 445 — NTLM relay attempt caught by NSG
AzureNetworkAnalytics_CL
| where SubType_s == "FlowLog"
| where SrcIP_s startswith "10.x.x."
| where FlowStatus_s == "D"
| where DestPort_d == 445
| project TimeGenerated, SrcIP_s, DestIP_s

// Unusual destination diversity — port-scanning or C2 beacon pattern
AzureNetworkAnalytics_CL
| where SubType_s == "FlowLog"
| where SrcIP_s startswith "10.x.x."
| where FlowDirection_s == "O"
| summarize DistinctDestinations = dcount(DestIP_s) by SrcIP_s, bin(TimeGenerated, 1h)
| where DistinctDestinations > 50
```

Azure Firewall also writes diagnostic logs to Log Analytics. Use the `AzureDiagnostics` table with `Category == "AzureFirewallApplicationRule"` or `"AzureFirewallNetworkRule"` to audit FQDN-level allow/deny decisions across the fleet. Azure Firewall deny events for agent Cloud PC source IPs are the highest-priority alert category: they indicate an agent process is attempting to reach a destination outside the allowlist, which is either a legitimate new endpoint (add to the allowlist) or an exfiltration attempt (investigate immediately).

### Localhost Binding

The OpenClaw Gateway's localhost binding behavior is covered in Chapter 29. The Windows Firewall rules above (port 18789 block from non-loopback sources) provide the host-level enforcement. NSG rules at the Azure network layer cannot restrict intra-host traffic — the Windows Firewall rules are the correct control for that path.

---
