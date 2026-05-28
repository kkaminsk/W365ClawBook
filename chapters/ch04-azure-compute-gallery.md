# Part II: Infrastructure

---

## Chapter 4: Azure Compute Gallery for Windows 365

### Why Azure Compute Gallery

> **⚠️ Preview Integration:** Azure Compute Gallery itself is a **generally available** Azure service with full SLA coverage. However, the **integration between ACG and Windows 365** (importing ACG image versions into Intune as custom images for Cloud PC provisioning) was in **public preview** at the time of writing. This means the ACG-to-W365 import workflow may change in behaviour, require different permissions, or introduce breaking changes before reaching GA. The preview status carries no SLA for the integration path specifically. Organizations with strict change-management policies should evaluate whether this preview dependency is acceptable for production image pipelines, or whether to continue using managed images until the ACG integration reaches GA. Monitor [learn.microsoft.com/windows-365](https://learn.microsoft.com/windows-365/) for status updates.

If you've been managing Windows 365 custom images using Azure managed images, you've felt the limitations: no versioning, no Trusted Launch support, no replication, and no staged rollout capability. Azure Compute Gallery changes the equation fundamentally.

For this developer image scenario, ACG provides three capabilities that matter:

1. **Trusted Launch compatibility.** Windows 365 now requires ACG image definitions to declare Trusted Launch support. Managed images cannot participate in this security model, and Microsoft is actively moving the ecosystem toward Trusted Launch as the default.

2. **Semantic versioning and staged rollout.** When your image includes AI agents with rapidly evolving dependencies, you need the ability to publish a new version, test it with a pilot group, and promote it to production, without maintaining a naming convention spreadsheet. ACG's `Major.Minor.Patch` versioning and `excludeFromLatest` flag give you this workflow natively.

3. **Pipeline integration.** Azure VM Image Builder and HashiCorp Packer both have first-class support for publishing to ACG. Your developer image becomes an "image-as-code" artifact: source-controlled templates, automated builds, and full audit trails.

### Why AIB Over Packer

Both Azure VM Image Builder (AIB) and HashiCorp Packer can produce ACG image versions. This book uses AIB for three reasons:

1. **No build infrastructure to manage.** AIB is a fully managed Azure service; it provisions the build VM, runs your customizers, captures the image, and tears down the VM automatically. With Packer, you manage the build VM lifecycle, networking, and credentials yourself.
2. **Native Terraform integration via azapi.** The AIB template is defined as a Terraform resource, so the entire pipeline (from gallery creation through build trigger) is a single `terraform apply`. Packer requires a separate build step outside Terraform.
3. **Cost.** AIB charges only for the compute time of the ephemeral build VM. There is no AIB service fee. Packer itself is free, but you pay for the VM infrastructure and must manage its lifecycle.

Packer remains a strong choice if you need cross-cloud image builds (AWS AMIs + Azure VHDs from the same template) or if your team already has Packer expertise. For a Windows 365-only pipeline, AIB is simpler and more cost-effective.

### Image Definition Requirements for Windows 365

Before you build anything, the ACG image definition must satisfy Windows 365's compatibility contract. The image definition **must** include all five of the following features:

| Terraform Attribute | Purpose |
|---------|---------|
| `trusted_launch_enabled = true` | Enables Trusted Launch (Secure Boot + vTPM) |
| `hibernation_enabled = true` | Required for Cloud PC hibernation |
| `disk_controller_type_nvme_enabled = true` | Supports NVMe disk controller |
| `accelerated_network_support_enabled = true` | Required for accelerated networking |

> **⚠️ Warning:** Missing any of these feature attributes will cause the import into Windows 365 to fail. This is non-negotiable. The error message from Intune is often unhelpful; if your import fails, check these attributes first. Note that AzureRM provider 4.x uses dedicated boolean attributes instead of the legacy `features {}` blocks.

Additionally, the image definition must declare:

- **Architecture:** x64
- **OS Type:** Windows
- **Hyper-V Generation:** V2
- **OS State:** Generalized

### Why Terraform Only

You could accomplish the gallery setup with Bicep, ARM templates, or the Azure CLI. This book uses Terraform exclusively because the entire image build pipeline (gallery, identity, RBAC, AIB template, and build trigger) is managed as a single Terraform state. Mixing IaC tools would split the lifecycle management across two toolchains, complicating both the initial deployment and ongoing version bumps. The Azure CLI example below is provided for verification and one-off operations, not as an alternative deployment path.

### Gallery Setup with Terraform

```hcl
resource "azurerm_shared_image_gallery" "this" {
  name                = var.gallery_name
  resource_group_name = var.resource_group_name
  location            = var.location
  tags                = var.tags

  lifecycle {
    prevent_destroy = true
  }
}

resource "azurerm_shared_image" "this" {
  name                = var.image_definition_name
  gallery_name        = azurerm_shared_image_gallery.this.name
  resource_group_name = var.resource_group_name
  location            = var.location
  os_type             = "Windows"
  hyper_v_generation  = "V2"
  architecture        = "x64"

  identifier {
    publisher = var.image_publisher
    offer     = var.image_offer
    sku       = var.image_sku
  }

  # -- Windows 365 ACG Import Requirements --
  # These features are mandatory for Windows 365 ingestion.
  trusted_launch_enabled              = true
  hibernation_enabled                 = true
  disk_controller_type_nvme_enabled   = true
  accelerated_network_support_enabled = true

  tags = var.tags

  lifecycle {
    prevent_destroy = true
  }
}
```

### Gallery Setup with Azure CLI

```bash
az sig create \
  --resource-group rg-w365-images \
  --gallery-name acgW365Dev

az sig image-definition create \
  --resource-group rg-w365-images \
  --gallery-name acgW365Dev \
  --gallery-image-definition W365-W11-25H2-ENU \
  --publisher BigHatGroupInc \
  --offer W365-W11-25H2-ENU \
  --sku W11-25H2-ENT-Dev \
  --os-type Windows \
  --os-state Generalized \
  --hyper-v-generation V2 \
  --architecture x64 \
  --features "SecurityType=TrustedLaunchSupported" \
             "IsHibernateSupported=True" \
             "DiskControllerTypes=SCSI,NVMe" \
             "IsAcceleratedNetworkSupported=True" \
             "IsSecureBootSupported=True"
```

### Multi-Image Definitions for Different Team Profiles

This book assumes a single image definition (`W365-W11-25H2-ENU`) with one toolchain. In practice, organizations with diverse development teams (frontend, backend, data science, infrastructure) may benefit from **multiple image definitions**, each tailored to a team's specific requirements.

Consider mapping image definitions to AI agent personas. OpenClaw's persona configuration (covered in Chapter 29) is defined through SOUL.md; persona differences influence which runtimes and tools belong in each image variant. OpenClaw supports persona configuration through its `SOUL.md` and agent profiles. A frontend team's agent might specialize in React and TypeScript, while a data science team's agent focuses on Python, Jupyter, and pandas. These persona differences often align with different runtime and tooling requirements in the base image:

| Image Definition | Target Team | Additional Runtimes | Agent Persona |
|---|---|---|---|
| `W365-W11-25H2-W365` | Productivity Worker | Office, Pandoc | Information Worker |
| `W365-W11-25H2-Frontend` | Frontend developers | Node.js, Bun | React/TypeScript specialist |
| `W365-W11-25H2-Backend` | Backend developers | Node.js, Python, Docker | API and microservices focus |
| `W365-W11-25H2-DataSci` | Data science | Python, Conda, CUDA drivers | ML/analytics focus |
| `W365-W11-25H2-Platform` | Platform engineering | Terraform, kubectl, Helm | Infrastructure-as-code focus |

Each image definition lives in the same Azure Compute Gallery and follows the same build pipeline pattern; only the Phase 1--3 customizers differ. The Terraform module can be parameterized with a `team_profile` variable that selects the appropriate toolchain. This approach scales the "image-as-code" pattern to serve the full breadth of an engineering organization while maintaining a single, consistent build and governance process.

### RBAC for Windows 365 Consumption

To import an ACG image into Windows 365 through Intune, the admin account needs the **Compute Gallery Image Reader** role on the gallery. This is intentionally narrow: your image engineering team manages the gallery, and your Cloud PC admins consume from it without the ability to modify or delete gallery resources.

> **💡 Tip:** Separate duties between the image engineering team (who build and publish) and the Cloud PC operations team (who consume and assign). The RBAC model in ACG supports this cleanly.

---

