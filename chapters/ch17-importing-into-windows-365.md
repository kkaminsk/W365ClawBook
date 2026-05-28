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

