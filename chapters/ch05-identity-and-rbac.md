## Chapter 5: Identity and RBAC

### The Managed Identity for AIB

Azure VM Image Builder requires a managed identity to perform its work. The solution uses a **user-assigned managed identity** rather than a system-assigned identity because:

1. **Lifecycle independence** — the identity persists if the AIB template resource is destroyed or re-created, maintaining consistent permissions across rebuilds.
2. It can be reused across multiple image template versions
3. RBAC assignments persist independently of the AIB template lifecycle

```hcl
resource "azurerm_user_assigned_identity" "aib" {
  name                = "id-aib-w365-dev"
  resource_group_name = var.resource_group_name
  location            = var.location
  tags                = var.tags
}
```

### Least-Privilege Role Assignments

Instead of granting the broad `Contributor` role, the solution assigns exactly four roles, the minimum required for AIB to function:

```hcl
# 1. Virtual Machine Contributor -- create/manage the build VM
resource "azurerm_role_assignment" "aib_vm_contributor" {
  scope                            = var.resource_group_id
  role_definition_name             = "Virtual Machine Contributor"
  principal_id                     = azurerm_user_assigned_identity.aib.principal_id
  skip_service_principal_aad_check = true

  lifecycle {
    ignore_changes = [skip_service_principal_aad_check]
  }
}

# 2. Network Contributor -- create transient networking for the build VM
resource "azurerm_role_assignment" "aib_network_contributor" {
  scope                            = var.resource_group_id
  role_definition_name             = "Network Contributor"
  principal_id                     = azurerm_user_assigned_identity.aib.principal_id
  skip_service_principal_aad_check = true

  lifecycle {
    ignore_changes = [skip_service_principal_aad_check]
  }
}

# 3. Managed Identity Operator -- assign the identity to the build VM
resource "azurerm_role_assignment" "aib_identity_operator" {
  scope                            = var.resource_group_id
  role_definition_name             = "Managed Identity Operator"
  principal_id                     = azurerm_user_assigned_identity.aib.principal_id
  skip_service_principal_aad_check = true

  lifecycle {
    ignore_changes = [skip_service_principal_aad_check]
  }
}

# 4. Compute Gallery Artifacts Publisher -- write image versions to the gallery
resource "azurerm_role_assignment" "aib_gallery_contributor" {
  scope                            = var.gallery_id
  role_definition_name             = "Compute Gallery Artifacts Publisher"
  principal_id                     = azurerm_user_assigned_identity.aib.principal_id
  skip_service_principal_aad_check = true

  lifecycle {
    ignore_changes = [skip_service_principal_aad_check]
  }
}
```

| Role | Scope | Purpose |
|------|-------|---------|
| Virtual Machine Contributor | Resource Group | Create/manage the ephemeral build VM |
| Network Contributor | Resource Group | Create transient vNIC/NSG for the build VM |
| Managed Identity Operator | Resource Group | Assign the identity to the build VM |
| Compute Gallery Artifacts Publisher | Gallery | Write image versions to ACG |

> **💡 Tip:** When creating a managed identity and assigning roles in the same `terraform apply`, the Entra ID principal may not have propagated yet, causing intermittent failures. The solution includes `skip_service_principal_aad_check = true` on each `azurerm_role_assignment` to handle this, with `lifecycle { ignore_changes }` to prevent perpetual diffs. Without `ignore_changes`, Terraform detects drift on this attribute after Azure silently updates it, causing a perpetual planned change on every subsequent `terraform plan`.

### Why Not Contributor?

The `Contributor` role grants write access to every resource type in the scope. For an AIB identity that only needs to create VMs, networking, and write gallery images, `Contributor` is massively over-privileged. If the identity were compromised, `Contributor` would allow an attacker to deploy any resource type, modify existing resources, or exfiltrate data. The four-role approach limits the blast radius to exactly the operations AIB performs.

Chapter 6 builds the full Terraform module structure that consumes this identity configuration.

---

