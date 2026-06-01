## Chapter 41: Disaster Recovery and Regional Failover

### The Gap Between "Cloud" and "Resilient"

Windows 365 is not inherently geo-redundant. The service is highly available within an Azure region — storage is separated from compute, the OS disk is replicated with 11 nines of durability, and in-zone compute failures recover automatically. But if the Azure region where your Cloud PCs are deployed goes dark, your users lose access until that region is restored, unless you have explicitly configured cross-region disaster recovery.

For a deployment running AI coding agents, this matters more than it would for a standard Cloud PC rollout. An OpenClaw or Claude Code session that is interrupted mid-task leaves partial work, open tool calls, and potentially uncommitted code. The stateless design of Windows 365 (OneDrive for user data, Git for code) limits the blast radius, but it does not eliminate the need for a recovery plan.

This chapter covers all four layers of Windows 365 resilience — what is built-in, what is configurable, and what requires an additional license — then gives you the runbook for activating and testing regional failover.

---

### The Four Resilience Layers

Windows 365 provides resilience in layers, each with different scope, cost, and operator involvement.

```
┌────────────────────────────────────────────────────────────────────┐
│  Layer 4: Disaster Recovery Plus (add-on license)                  │
│  Cross-region, pre-reserved capacity, RPO <61 min, RTO <31 min     │
├────────────────────────────────────────────────────────────────────┤
│  Layer 3: Cross-Region Disaster Recovery (add-on license)          │
│  Cross-region, best-effort capacity, RPO <4h, RTO <4h             │
├────────────────────────────────────────────────────────────────────┤
│  Layer 2: Point-in-Time Restore (configurable, included)           │
│  In-region, manual restore, RPO = snapshot interval, RTO up to 4h  │
├────────────────────────────────────────────────────────────────────┤
│  Layer 1: In-Zone Compute DR (automatic, included)                 │
│  In-zone, automatic, RPO ~0, RTO <10 min                           │
└────────────────────────────────────────────────────────────────────┘
```

**Layer 1 — In-Zone Compute DR** is automatic and requires no configuration. When a host-level failure (vNic, power, storage plane) occurs, Azure moves the workload to another host in the same zone. The user experiences a brief interruption and must reconnect. Because storage is separate from compute and uses an up-to-date OS disk copy, the RPO is effectively zero. This layer protects against the most common class of failure and is always active.

**Layer 2 — Point-in-Time Restore** protects against user-level failures: a bad OS update, a corrupted driver, a misconfigured policy, or a runaway agent that trashed the system. An admin (or, if permitted, the user) restores the Cloud PC to a known-good snapshot. This is manual, in-region, and restores to a different zone within the region if the original zone is unavailable. If the entire region is down, this layer provides no relief.

**Layer 3 — Cross-Region Disaster Recovery** is an optional add-on that creates geographically distant snapshots of each licensed Cloud PC in a backup region of your choice. An admin manually activates failover when a regional outage occurs. Recovery targets are RTO and RPO of less than four hours.

**Layer 4 — Disaster Recovery Plus** extends Layer 3 by pre-allocating compute capacity in the backup region. This eliminates the capacity-at-time-of-outage risk inherent in Layer 3, achieving RPO under 61 minutes and RTO under 31 minutes.

The two add-on layers are separate licenses. You can assign them to a subset of users — you do not need to license the entire tenant.

---

### Configuring Point-in-Time Restore

Point-in-time restore is the foundation that all higher layers build on. Configure it before you begin assigning cross-region DR settings.

#### Restore Point Schedules

| Type | Frequency | Count Kept | Configurable |
|------|-----------|------------|--------------|
| Short-term | 4, 6, 12, 16, or 24 hours | 10 | Yes |
| Long-term | Every 7 days | 4 | No |
| Manual | On-demand | 1 (expires ~28 days) | N/A |

A 4-hour short-term interval gives you 10 restore points spanning the last 40 hours. A 24-hour interval gives you 10 restore points spanning the last 10 days. Choose based on your RPO tolerance and the rate of change on your Cloud PCs. For developer Cloud PCs running AI agents, a 4- or 6-hour interval is appropriate — the content of a developer session (installed extensions, cloned repos, configuration changes) changes frequently enough that losing 24 hours of state is disruptive even if code is in Git.

The four weekly long-term restore points are always created regardless of your short-term setting. They provide a safety net for problems that aren't discovered until several days after they occur.

#### Configuration Steps

1. Sign in to the [Microsoft Intune admin center](https://intune.microsoft.com) > **Devices** > **Cloud PC Settings** > **Add**.
2. Type a **Name** for the user setting.
3. Under **Point-in-time restore service**, set **Frequency of restore-point service** to your chosen interval.
4. If you want users to be able to initiate their own restores, enable **Allow user to initiate restore service**.
5. On the **Assignments** page, add the Entra ID groups containing your Cloud PC users.
6. **Review + Create**.

#### What Is (and Is Not) in a Restore Point

A restore point captures the full OS disk: installed applications, user profile data stored locally, registry, and configuration. It does not capture:

- Files stored in OneDrive (these are already resilient at the OneDrive layer)
- Agent state stored outside the local disk (Key Vault references, Git repositories)
- MCP server configuration if it is stored in cloud-backed paths

For OpenClaw and Claude Code deployments, any agent state that lives in `C:\Users\<agent-account>\AppData\Local` and is not synced to OneDrive will be rolled back by a restore. The managed-settings policy and skill directory in `C:\ProgramData\OpenClaw\` will also revert to the restore point's state — which is the intended behavior for OS-level configuration, but worth noting if you push policy updates between snapshot intervals.

#### Initiating a Restore

To restore a single Cloud PC: Intune admin center > **Devices** > **All devices** > select the Cloud PC > **Restore** > choose a restore point > confirm.

To restore multiple Cloud PCs in bulk: **Devices** > **All devices** > **Bulk device actions** > OS: Windows, Device type: Cloud PCs, Device action: Restore.

The restore process automatically places the Cloud PC on available infrastructure in the same region. If the original availability zone is unavailable but the region itself is healthy, the Cloud PC lands in a different zone within the region — no administrator action required for zone selection.

---

### Cross-Region Disaster Recovery

Cross-region DR protects against full regional outages. When activated, it creates a temporary Cloud PC in your chosen backup region using the most recent restore point for each affected user.

#### What Gets Replicated — and What Doesn't

A common misconception: cross-region DR replicates the user's **running Cloud PC OS disk**, not the provisioning image from Azure Compute Gallery. Your ACG image only needs to be in one region because it is used only at provisioning time. Once a Cloud PC is provisioned and running, the OS disk drifts from the original image as users install software, apply configuration, and accumulate state. Cross-region DR snapshots that evolved disk — not the base image — into the backup region.

This means:
- ACG image replication to the backup region is **not required** for cross-region DR.
- The backup contains the user's full Cloud PC state, including post-provisioning changes.
- There is no "image version mismatch" problem between primary and backup.

#### Licensing

Each user whose Cloud PC should participate in failover requires a **Windows 365 Cross Region Disaster Recovery add-on license**. Cloud PCs without this license do not participate in failover — they remain accessible only when the primary region is restored.

RTO and RPO targets apply to the licensed Cloud PCs, not to the total tenant Cloud PC count.

#### Setup

Cross-region DR is configured in the same user setting as point-in-time restore.

1. Intune admin center > **Devices** > **Cloud PC Settings** > **Add** (or edit an existing setting).
2. Configure your restore point frequency as described above.
3. Expand **Cross region disaster recovery configuration (Optional)**.
4. Set **Enable cross region disaster recovery** to **Yes**.
5. Choose **Network type**:
   - **Microsoft-hosted network**: Select the **Geography** and **Region** for your backup location. Microsoft manages the network in the backup region.
   - **Azure network connection (ANC)**: The ANC you select determines the backup region. You must have a separate ANC already configured for the backup region. The backup Cloud PCs use that ANC's vNet for connectivity.
6. Assign to groups > **Create**.

After configuration, the first backup of each Cloud PC may take several days. Use the **Cloud PCs cross region disaster recovery status** report (Intune admin center > **Devices** > **Windows 365** > **Cross region disaster recovery status**) to confirm backups are healthy before you consider the environment DR-ready.

#### ANC Considerations

If your Cloud PCs use a custom ANC — which is the likely choice for deployments that require corporate network access for MCP servers, Key Vault, or internal repositories — you need a second ANC configured for your backup region before enabling cross-region DR with the ANC network type.

The backup ANC must:
- Connect to an Azure vNet in the backup region
- Have the same firewall rules, DNS forwarders, and routing that your primary ANC has
- Be able to reach the same Key Vault endpoints (Key Vault is a global service, so this is usually straightforward)
- Be tested for connectivity before the DR policy is assigned

Microsoft recommends creating multiple ANC connections and prioritizing them. The first ANC in priority order is used for normal operations; subsequent ANCs serve as network-level failover even apart from the disaster recovery feature.

#### Region Selection

Any region where Windows 365 is available can serve as a backup region. There is no fixed list of required region pairs. Select a backup region based on:

- **Data sovereignty**: Full copies of user OS disks are held in the backup region. If your users' data must remain within a jurisdiction, select a backup region in the same jurisdiction.
- **Geographic distance**: The backup region should be physically distant from the primary to protect against events that affect multiple data centres in close proximity.
- **Network latency**: Users connect to the backup Cloud PC during an outage. If the backup region is geographically distant from your users, performance will be degraded. This is acceptable for a temporary outage but worth modelling for your user population.

---

### Failover Runbook

**Pre-condition**: Cross-region DR is configured, licensed, and the DR status report shows all Cloud PCs as healthy.

#### Step 1 — Detect and Declare

Confirm that the outage is regional and not a local network or client issue. Check the [Azure Service Health dashboard](https://status.azure.com) and your tenant's Service Health blade in the Azure portal for a declared regional incident. Cross-region DR is designed for regional outages; activating it for smaller-scope incidents wastes the temporary Cloud PC capacity and starts the 7-day automatic failback clock.

#### Step 2 — Check the DR Status Report

Before activating, open the **Cloud PCs cross region disaster recovery status** report and confirm there are no errors. Errors at this point (unhealthy backups, missing ANC connectivity) indicate Cloud PCs that may not successfully restore in the backup region. Address them if time permits; proceed if the outage is active and most devices are healthy.

#### Step 3 — Activate

1. Intune admin center > **Devices** > **All devices** > **Bulk device actions**.
2. **OS**: Windows | **Device type**: Cloud PCs | **Device action**: Optional disaster recovery.
3. **Action type**: **Activate cross region disaster recovery**.
4. Select the devices or groups to activate.
5. **Create**.

Users attempting to sign in during the activation see a warning message. Once the temporary Cloud PC is ready (within the RTO window), they sign in normally and find their Cloud PC in the state it was at the last restore point.

#### Step 4 — Communicate to Users

Notify users that:
- Their Cloud PC is running in a temporary backup location.
- The temporary Cloud PC reflects their state as of the last restore point (up to 4 hours prior).
- Any changes they make to the local C: drive will not be preserved when the outage ends and they return to their primary Cloud PC.
- Files saved to OneDrive sync normally and will be available on their primary device.
- Agent sessions should treat the temporary Cloud PC as read-only except for OneDrive-backed paths.

#### Step 5 — Monitor Recovery Target

The 4-hour RTO applies to tenants with fewer than 50,000 cross-region DR-licensed Cloud PCs. The service does not pre-reserve capacity, so recovery happens using available resources in the backup region at the time of the outage. If the backup region is also under stress, some Cloud PCs may recover in waves as capacity becomes available. Monitor the DR status report for per-device status.

#### Step 6 — Deactivate (Failback)

When Microsoft declares the primary region healthy, or when you determine the outage is resolved:

1. Intune admin center > **Devices** > **All devices** > **Bulk device actions**.
2. **Device action**: Optional disaster recovery | **Action type**: **Deactivate cross region disaster recovery**.
3. Select devices > **Create**.

Deactivation takes up to one hour. During this window, users may continue working on their temporary Cloud PC. When the transfer completes, they sign back into their primary device. The temporary Cloud PC is then deleted — any files written to the local C: drive during the outage are gone. OneDrive files sync automatically to the primary device.

**7-day automatic failback**: If you do not manually deactivate, the platform automatically fails back after 7 days — measured from when the outage was declared over, or from when you activated DR (if there was no Microsoft-declared outage). This timer cannot be extended.

---

### When to Use Disaster Recovery Plus

Standard cross-region DR is sufficient for most deployments. Consider Disaster Recovery Plus when:

- You need guaranteed capacity — not best-effort — in the backup region.
- Your SLA requires RTO under 1 hour (DR Plus targets RTO <31 min, RPO <61 min).
- You have already seen capacity constraints during a cross-region DR test.
- Your Cloud PCs support revenue-generating or compliance-critical workflows where extended outages have direct business or regulatory consequences.

DR Plus creates three copies of the OS disk to the backup region and pre-allocates compute. The initial full disk copy takes up to three days after configuration; subsequent incremental copies take only minutes. Activation and deactivation follow the same runbook steps as standard cross-region DR.

---

### Agent Workload Considerations

OpenClaw and Claude Code sessions have specific behaviours during regional outages and DR failover.

**Active agent sessions are interrupted.** There is no session migration. Any in-flight tool call, file write, or external API request that has not completed at the point of interruption is lost. Design agent workflows to use atomic commits to Git and to checkpoint progress in OneDrive-backed paths rather than in-memory state or local temporary files.

**Key Vault access** is a global service and should be accessible from the backup region without reconfiguration, assuming the ANC in the backup region has the correct network routes. Verify this during your DR test (see below).

**MCP server state.** MCP servers running as local processes on the Cloud PC are restarted from the restore point on the temporary Cloud PC. Ephemeral state (model context, tool results held in memory) is lost. The managed-settings policy and MCP server configuration in `C:\ProgramData\OpenClaw\` is restored to the state at the last restore point.

**Git repositories** cloned to the local disk are present at the restore point state. Any commits made between the last restore point and the outage that were not pushed to the remote are lost. Enforce frequent push discipline in agent workflow design. Use a Git pre-push hook to checkpoint agent context to an OneDrive-backed location before each push:

**Example `.git/hooks/pre-push`** (create in each repository; mark executable):

```bash
#!/bin/sh
# Checkpoint OpenClaw session state to OneDrive before every push.
# Runs in the Git repository's context — adjust ONEDRIVE_PATH as needed.

ONEDRIVE_PATH="$USERPROFILE/OneDrive - $(hostname)/AgentBackups"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_DIR="$ONEDRIVE_PATH/$(basename $(pwd))/$TIMESTAMP"

mkdir -p "$BACKUP_DIR"

# Checkpoint OpenClaw session state if gateway is running
if command -v openclaw >/dev/null 2>&1; then
    openclaw session export "$BACKUP_DIR/session.json" 2>/dev/null || true
fi

# Copy agent workspace files to the backup location
WORKSPACE="$USERPROFILE/Documents/OpenClawWorkspace"
if [ -d "$WORKSPACE" ]; then
    cp -r "$WORKSPACE" "$BACKUP_DIR/workspace" 2>/dev/null || true
fi

# Always allow the push to proceed
exit 0
```

Install this hook for all new repositories by adding it to your Git template directory (`git config --global init.templateDir`), or deploy it via an Intune script that runs `git config --global core.hooksPath` pointing to a shared hooks directory in a OneDrive-backed location.

---

### DR Testing

Microsoft explicitly recommends testing the full cross-region flow before relying on it in production. Do not discover setup problems during an actual outage.

#### Quarterly DR Drill

1. Pick a non-production subset of Cloud PCs (a group of test devices works well).
2. Assign the cross-region DR license and user setting to the test group.
3. Wait for the DR status report to show all test devices as healthy (up to several days after first configuration).
4. Activate cross-region DR on the test group using the activation runbook steps above.
5. Verify:
   - Test Cloud PCs provision in the backup region within the expected RTO.
   - Users can sign in and see their last restore point state.
   - OneDrive sync is functional.
   - Key Vault is reachable (test with `az keyvault secret show`).
   - MCP servers start and the OpenClaw config is correct.
   - Agent API keys resolve correctly.
6. Deactivate and confirm failback completes within 1 hour.
7. Document the observed RTO and RPO for each test device.

#### DR Readiness Checklist (see also Chapter 36)

- [ ] Cross-region DR add-on license assigned to all users in scope
- [ ] User setting configured with cross-region DR enabled and backup region selected
- [ ] ANC for backup region configured and tested (if not using Microsoft-hosted network)
- [ ] DR status report shows all in-scope Cloud PCs as **Healthy**
- [ ] Restore point frequency set to 4 or 6 hours (developer workloads)
- [ ] Quarterly DR drill executed and results documented
- [ ] User communication template prepared and approved
- [ ] OneDrive Known Folder Move enabled for all users (preserves Desktop, Documents, Pictures across failover)
- [ ] Agent Git repositories configured with remote push on every commit
- [ ] Key Vault reachability confirmed from backup region ANC

---

### Summary

Windows 365 provides four layers of resilience. The bottom two (in-zone compute DR and point-in-time restore) are available to all tenants with no additional licensing. The top two (cross-region DR and DR Plus) require add-on licenses but provide protection against full regional outages.

For ClawBook deployments, configure point-in-time restore at a 4- or 6-hour interval as a baseline. Assign cross-region DR licenses to users whose Cloud PCs host production agent workloads, and select a backup region that satisfies your data sovereignty requirements. Run a quarterly drill to confirm that the backup region ANC, Key Vault access, and MCP server configuration are functional before you need them.

The failover process itself takes only a few minutes of administrator action — the preparation is what takes time.

---

*See also: [Cross region disaster recovery in Windows 365](https://learn.microsoft.com/en-us/windows-365/enterprise/cross-region-disaster-recovery) · [Business continuity and disaster recovery with Windows 365](https://learn.microsoft.com/en-us/windows-365/business-continuity-disaster-recovery) · [Point-in-time restore overview](https://learn.microsoft.com/en-us/windows-365/enterprise/restore-overview) · [Windows 365 disaster recovery plus](https://learn.microsoft.com/en-us/windows-365/enterprise/disaster-recovery-plus)*
