## Chapter 17: Importing into Windows 365

### Intune Portal Walkthrough

1. Sign in to the **Microsoft Intune admin center** (intune.microsoft.com)
2. Navigate to **Devices** -> **Windows 365** -> **Custom images**
3. Click **Add** -> **Azure Compute Gallery**
4. Select:
   - **Subscription:** Your subscription containing the gallery
   - **Gallery:** `acgW365Dev`
   - **Image definition:** `W365-W11-25H2-ENU`
   - **Image version:** `1.0.0` (or your target version)
5. Complete the verification checklist in Chapter 15 before proceeding with the import.
6. Click **Add**; the import takes a few minutes

### Provisioning Policy Setup

1. Navigate to **Devices** -> **Windows 365** -> **Provisioning policies**
2. Create or edit a provisioning policy:
   - **Image:** Select the imported custom image
   - **Network:** Azure Network Connection (or Microsoft-hosted network)
   - **Join type:** Entra join
   - **Assignment:** Target your developer security group

> **Network selection:** Choose an Azure Network Connection (ANC) if your Cloud PCs must reach on-premises resources or internal Azure services. Choose the Microsoft-hosted network option for a zero-infrastructure, fully cloud-managed deployment with no VNet configuration required.

> **⚠️ Warning:** The Windows 365 custom image import step in Intune remains a **manual portal operation**. There is no public Graph API or PowerShell cmdlet to automate the "Add custom image from ACG (Azure Compute Gallery)" action. Your automation pipeline ends at "image version published to ACG," and an admin picks it up from there.

> **Common import failures:** image was not generalized with Sysprep, was previously Entra-joined or Intune-enrolled, does not meet the Windows 365 OS version requirements, or the image definition is missing required attributes. See Chapter 36 for the complete requirements checklist.

---

### Pre-Import Checklist

The import action in Intune is irreversible in the sense that triggering it against an invalid image wastes time and forces a rebuild. Confirm every condition below before clicking **Add**.

> **Windows 365 for Agents (Preview):** Microsoft announced a consumption-based "Windows 365 for Agents" SKU in public preview on January 22, 2026, purpose-built for agentic AI workloads. When evaluating provisioning policy options, note that this SKU may appear in the Intune provisioning policy interface before GA. Until Windows 365 for Agents reaches GA (projected Q4 2026), use standard Windows 365 Enterprise for production deployments. The import workflow described in this chapter applies equally to both SKUs.

1. The image has passed the full verification checklist in Chapter 15.
2. Sysprep completed successfully — the image log shows a clean generalize pass with no errors.
3. The image is **generalized**, not specialized. A specialized image retains user accounts and machine identity and is rejected by Windows 365.
4. The image has **never been Entra-joined or Intune-enrolled** in its current generalized state. If a test VM was enrolled before Sysprep ran, the image carries enrollment artifacts that will cause the import to fail or produce an unhealthy Cloud PC.
5. The image version is published in Azure Compute Gallery with a replication status of **Succeeded** for the target region. A version still replicating appears as Pending to the Intune import pipeline.
6. The ACG is connected to the Intune tenant. Verify under **Tenant administration** -> **Connectors and tokens** -> **Azure Compute Gallery** that the gallery subscription is listed and status is **Active**.
7. The account performing the import holds the **Intune Administrator** role or a custom role that includes the `deviceManagement/windowsImages/create` permission. A Global Administrator who has never been assigned the Intune Administrator role does not automatically have this permission.

---

### Import Failure Modes

| Failure symptom | Root cause | Resolution |
|---|---|---|
| "Image is not generalized" | Sysprep did not run, ran with the wrong switches, or failed silently. The image was captured from a running or specialized state. | Rebuild the image. Sysprep cannot be re-run on an already-captured generalization failure. |
| "Image rejected: previously joined" | The source VM was Entra-joined or Intune-enrolled before Sysprep was executed. Enrollment artifacts survive Sysprep in some configurations. | Rebuild from a clean base image that was never joined or enrolled. Do not attempt to unjoin and re-Sysprep the same VM. |
| Import stuck in **Pending** | ACG replication to the tenant's assigned region has not completed. The Intune pipeline cannot access the image version until all target replicas report **Succeeded**. | Wait for replication to complete. Check ACG replication status in the Azure portal under the image version's **Replication** tab. Do not trigger a second import attempt on the same version while the first is pending. |
| "Validation failed: requirements not met" | The image is missing a required Windows feature, app, or OS configuration. Common triggers: unsupported Windows edition, missing Winlogon registry keys, or a blocked or absent provisioning package. | Consult the Chapter 36 requirements checklist. Identify the failing requirement, fix it in the image pipeline, publish a new image version, and retry. |
| "Permission error during import" | The account performing the import lacks the required Intune RBAC permissions. | Verify that the account holds the **Intune Administrator** role or a custom role that includes image import rights. Assigning the role may take several minutes to propagate before the portal reflects the updated permissions. |

---

### Post-Import Verification

A completed import is not the same as a successful import. After Intune reports the import as finished, confirm the following before updating any provisioning policy to reference the new image:

1. Navigate to **Devices** -> **Windows 365** -> **Custom images**. Locate the image and confirm its status is **Ready**. An **Error** status here means the import pipeline accepted the submission but validation failed after the fact — treat it as a failed import and do not proceed.
2. Confirm the image version displayed in the **Custom images** list matches the ACG version you intended to import (e.g., `1.0.0`). If multiple versions exist in the gallery, a mis-click during import is easy and the version number is the only field that distinguishes them at a glance.
3. Open the provisioning policy you intend to update and verify that selecting the imported image does not produce a validation warning. Some requirement failures surface at policy assignment time rather than at import time.
4. Do not assign the provisioning policy to a broad group until you have provisioned at least one test Cloud PC against the new image and confirmed that it reaches a **Provisioned** state. A single failed test provision is faster to diagnose than a batch failure across a developer group.

---

