# Proxmox Cluster Storage Architecture & Migration Strategy
**Date:** January 8, 2026  
**Status:** Proposed  
**Cluster:** lightville (PVE1, PVE2, PVE3)

---

## Executive Summary

This document defines the storage architecture and migration strategy for a 3-node Proxmox VE cluster, emphasizing:
- **Performance isolation** between runtime and backup workloads
- **Predictable recovery** without complex HA dependencies
- **Data integrity** through validated migrations
- **Disaster recovery** capability from cold storage

**Recovery Model:** Option A — Very Fast Restore (No automatic HA, emphasis on reliable backups)

---

## Cluster Overview

### Current Infrastructure

| Node | OS Disk | Additional Storage | Current Workloads |
|------|---------|-------------------|-------------------|
| **PVE1** | 500GB Samsung SSD | 2TB NVMe (installed) | 7 containers + 1 VM |
| **PVE2** | 500GB Samsung SSD | 500GB HDD RAID (backup) | CT111, PBS (planned) |
| **PVE3** | 500GB Samsung SSD | 500GB HDD (single disk) | Migration target |

### Planned Infrastructure

| Node | OS Disk | Production Storage | Backup Storage | Notes |
|------|---------|-------------------|----------------|-------|
| **PVE1** | 500GB Samsung SSD | **2TB NVMe** (installed) | 1TB SSD RAID + 1.1TB SAS RAID | Production ready |
| **PVE2** | 500GB Samsung SSD | **2TB NVMe** (pending) | 500GB HDD RAID + PBS | Awaiting NVMe |
| **PVE3** | 500GB Samsung SSD | **2TB NVMe** (pending) | 500GB HDD RAID (pending 2nd disk) | Awaiting hardware |

---

## Storage Architecture

### 1. Primary Workload Storage (Runtime Layer)

**Purpose:** Host all production containers and VMs with optimal performance

**Implementation:**
- **PVE1:** 2TB NVMe (currently in production)
- **PVE2:** 2TB NVMe (pending acquisition)
- **PVE3:** 2TB NVMe (pending acquisition)

**Temporary Placements** (until NVMe arrives):
- **PVE2:** CT111 and PBS on 500GB OS SSD (logically isolated)
- **PVE3:** CT101 on 500GB OS SSD (logically isolated)

**Design Principles:**
- Each node's workloads run on local NVMe for maximum IOPS
- No shared storage dependency for runtime operations
- Temporary workloads on OS disk must be migration-ready

---

### 2. Shared Storage on PVE1 (Non-Authoritative)

**Purpose:** Staging, hot-restore, and local backup distribution

**PVE1 RAID Configuration:**

#### 2.1 SanDisk SSD RAID (NFS Export)
- **Disks:** 2x 1TB SanDisk SSDs
- **RAID Level:** To be confirmed (RAID1 recommended)
- **Mount:** NFS share to cluster
- **Use Cases:**
  - Hot-restore staging area
  - High-priority standby workloads (limited, selective)
  - **NOT** the primary recovery source

#### 2.2 SAS HDD RAID (NFS Export)
- **Disks:** 2x 1.1TB SAS HDDs
- **RAID Level:** To be confirmed (RAID1 recommended)
- **Mount:** NFS share to cluster
- **Use Cases:**
  - Local backups of PVE1 workloads
  - Secondary, non-authoritative backup location
  - Restore staging area

**Critical Constraint:**
> These NFS storages are **NOT** treated as the primary cluster-wide backup source.

---

### 3. Primary Backup System (Authoritative)

**Proxmox Backup Server (PBS) Deployment:**

**Location:** Container on PVE2  
**Storage Backend:** 500GB HDD RAID on PVE2  
**RAID Level:** To be confirmed (RAID1 recommended)

**Responsibilities:**
1. ✅ Single authoritative backup system for all cluster nodes
2. ✅ All VMs and containers (PVE1, PVE2, PVE3)
3. ✅ Cluster-wide recovery operations
4. ✅ Disaster recovery scenarios

**Benefits:**
- Isolates backup I/O from production workloads (different node)
- Centralized backup management
- Deduplication and compression
- Incremental backups
- Verification and integrity checks

---

### 4. Local Backup Storage (Per-Node)

| Node | Configuration | Purpose | Status |
|------|--------------|---------|--------|
| **PVE2** | 500GB HDD RAID | PBS datastore + local backups | Active |
| **PVE3** | 500GB HDD (single) → RAID (planned) | Local backups | Pending 2nd HDD |

**PVE3 Upgrade Path:**
- Currently: Single 500GB HDD at `/mnt/hdd-backup`
- Planned: Add 2nd 500GB HDD, configure RAID1
- Benefit: Improved resilience for local backups

---

### 5. Offline / Disaster Recovery Storage

**Device:** External USB LaCie Drive  
**Purpose:** Cold backup archive (offsite/offline storage)

**Disaster Recovery Capabilities:**
1. ✅ Restore a single VM or container
2. ✅ Rebuild an entire node from scratch
3. ✅ Recover the entire cluster in worst-case scenario

**Sync Strategy:**
- Periodic exports from PBS
- Scheduled rsync from PBS datastore
- Manual verification after each sync

**Storage Location:** Physically separate from cluster (fire/theft protection)

---

## Recovery Model: Option A — Very Fast Restore

### Philosophy
- **NO** automatic HA failover (complexity reduction)
- **YES** reliable, tested backups
- **YES** predictable restore procedures
- **YES** minimal dependencies

### Recovery Scenarios

#### Scenario 1: Single Container/VM Failure
1. Identify failure
2. Restore from PBS (latest backup)
3. Validate service health
4. Return to production

**RTO:** Minutes to hours (depending on size)

#### Scenario 2: Node Failure
1. Identify failed node
2. Restore all workloads to surviving nodes from PBS
3. Rebalance if needed
4. Repair/replace failed node when available

**RTO:** Hours to rebuild workloads

#### Scenario 3: Total Cluster Loss
1. Rebuild cluster from scratch (new or repaired hardware)
2. Restore PBS from USB LaCie archive
3. Restore all workloads from PBS
4. Validate and return to production

**RTO:** Days (includes hardware acquisition/repair)

---

## Migration Strategy & Implementation

### Core Principles

1. **Safety First:** No data loss, no data corruption
2. **Validation Required:** Every migration must be verified before cutover
3. **Incremental Approach:** One workload at a time
4. **Capacity Awareness:** Confirm free space before migration
5. **I/O Stability:** Avoid operations causing read/write errors or contention

### Migration Methodology

#### Two-Phase Rsync Approach

**Phase 1: Pre-Copy (Hot)**
```bash
rsync -avz --progress /source/ /destination/
```
- Workload remains running
- Bulk data transfer while live
- Minimal service disruption

**Phase 2: Delta Sync & Cutover**
```bash
pct stop <VMID>  # Stop workload
rsync -avz --delete /source/ /destination/  # Final sync
# Validate destination
pct start <VMID>  # Start on new location
# Validate service health
# Retire original after confirmation
```

### Pre-Migration Checklist

- [ ] Identify source and destination storage
- [ ] Verify destination has sufficient free space (minimum 120% of source size)
- [ ] Review disk content for orphaned data
- [ ] Document current workload configuration
- [ ] Ensure PBS backup exists and is recent (<24 hours)
- [ ] Schedule migration during low-usage window
- [ ] Notify stakeholders of planned downtime

### Post-Migration Validation

- [ ] Data integrity check (file count, sizes)
- [ ] Service health verification (all services running)
- [ ] Application functionality test
- [ ] Backup inclusion in PBS confirmed
- [ ] Performance baseline comparison
- [ ] Documentation updated with new storage location

### Disk Content Review Protocol

**Before ANY destructive action:**
1. List all volumes, containers, VMs on target storage
2. Identify active workloads
3. Identify orphaned/obsolete data
4. Classify data as:
   - ✅ Active (keep)
   - ⚠️ Unknown (investigate)
   - ❌ Obsolete (candidate for deletion)
5. **Require explicit confirmation** before deletion

**No destructive action without approval.**

---

## Implementation Phases

### Phase 1: Immediate (PVE1 - Already Complete)
- ✅ PVE1 NVMe in production
- ✅ 7 containers + 1 VM running on ZFS pool
- ✅ 1.30TB free capacity available

### Phase 2: PBS Deployment (PVE2)
1. Confirm 500GB HDD RAID configuration
2. Deploy PBS container on PVE2
3. Configure PBS datastore on HDD RAID
4. Configure backup jobs for all cluster workloads
5. Validate backups from all nodes
6. Test restore procedure

**Timeline:** 1-2 days

### Phase 3: PVE2 NVMe Integration (When Available)
1. Install 2TB NVMe in PVE2
2. Configure ZFS/LVM on NVMe
3. Migrate CT111 from OS SSD to NVMe
4. Validate and retire original
5. Keep PBS on HDD RAID (backup I/O isolation)

**Timeline:** 1 day (after hardware arrival)

### Phase 4: PVE3 NVMe Integration (When Available)
1. Install 2TB NVMe in PVE3
2. Configure ZFS/LVM on NVMe
3. Migrate CT101 from OS SSD to NVMe
4. Validate and retire original
5. Add 2nd 500GB HDD for RAID1 local backup

**Timeline:** 1 day (after hardware arrival)

### Phase 5: USB Archive Setup
1. Configure LaCie USB drive
2. Create PBS export/sync script
3. Schedule periodic sync (weekly recommended)
4. Test restore from USB archive
5. Document recovery procedure

**Timeline:** 2-3 hours

### Phase 6: NFS Share Configuration (PVE1)
1. Confirm RAID configuration on both pairs
2. Create NFS exports:
   - SSD RAID: `/export/ssd-staging`
   - SAS RAID: `/export/sas-backup`
3. Mount on PVE2 and PVE3
4. Configure as Proxmox storage (not primary)
5. Test hot-restore workflow

**Timeline:** 2-3 hours

---

## Capacity Planning

### Current Allocation (PVE1)

| Dataset | Size | Used | Available | Workload |
|---------|------|------|-----------|----------|
| subvol-100 | 300GB | 127GB | 174GB | CT101 cPanel data |
| subvol-101 | 300GB | 5GB | 295GB | CT101 current (temporary) |
| subvol-103 | 25GB | 7.8GB | 17.4GB | CT103 app.onlineradio |
| subvol-105 | 8GB | 3.1GB | 5.5GB | CT105 claapp |
| subvol-107 | 40GB | 2.2GB | 38.1GB | CT107 api.onlineradio |
| subvol-109 | 320GB | 215GB | 105GB | CT109 channel (largest) |
| subvol-113 | 20GB | 1GB | 19.1GB | CT113 chatbot |
| vm-200 | 150GB | 152GB | - | VM200 Windows dev |

**Total Used:** ~568GB / 1.86TB (29%)  
**Total Free:** ~1.30TB (71%)

### Projected Growth

- **Conservative:** 10% annual growth = ~57GB/year
- **Moderate:** 20% annual growth = ~114GB/year
- **Aggressive:** 30% annual growth = ~170GB/year

**Runway at current growth:**
- Conservative: 22+ years
- Moderate: 11+ years
- Aggressive: 7+ years

**Conclusion:** Excellent headroom for expansion

---

## Risk Analysis & Mitigation

### Risk 1: Single Point of Failure (PBS on PVE2)
**Impact:** High  
**Likelihood:** Low  
**Mitigation:**
- USB LaCie cold backup archive
- Regular export validation
- Consider PBS replication in future

### Risk 2: NVMe Failure on Production Node
**Impact:** High  
**Likelihood:** Low (SMART monitoring)  
**Mitigation:**
- Daily PBS backups
- SMART monitoring and alerts
- Hot-restore capability from NFS staging

### Risk 3: Temporary Storage on OS SSD (PVE2, PVE3)
**Impact:** Medium  
**Likelihood:** Low  
**Mitigation:**
- Logical isolation (LVM thin provisioning)
- Migration-ready architecture
- PBS backups in place

### Risk 4: Migration Data Loss/Corruption
**Impact:** Critical  
**Likelihood:** Very Low (with rsync validation)  
**Mitigation:**
- Two-phase rsync methodology
- Pre-migration PBS backup
- Post-migration validation checklist
- No original deletion until confirmed

### Risk 5: Insufficient Backup Storage
**Impact:** Medium  
**Likelihood:** Low  
**Mitigation:**
- PBS deduplication and compression
- Retention policy enforcement
- Monitor datastore usage (alert at 80%)
- USB archive offload

---

## Success Criteria

### Technical Metrics
- ✅ All workloads running on designated storage tier
- ✅ PBS successfully backing up all cluster nodes
- ✅ Zero data loss during migrations
- ✅ Storage utilization <70% on all tiers
- ✅ I/O performance within baseline (no degradation)

### Operational Metrics
- ✅ Recovery time <4 hours for single workload
- ✅ Recovery time <24 hours for full node
- ✅ USB archive sync completed weekly
- ✅ All migrations validated before cutover
- ✅ Documentation complete and current

### Business Metrics
- ✅ Zero unplanned downtime during implementation
- ✅ Stakeholder confidence in disaster recovery
- ✅ Predictable restore procedures
- ✅ Minimal operational complexity

---

## Outstanding Questions & Confirmations Required

1. **RAID Configuration:** Confirm RAID level for:
   - [ ] PVE1 SanDisk SSD pair
   - [ ] PVE1 SAS HDD pair
   - [ ] PVE2 500GB HDD pair

2. **Hardware Timeline:**
   - [ ] ETA for PVE2 2TB NVMe
   - [ ] ETA for PVE3 2TB NVMe
   - [ ] ETA for PVE3 2nd 500GB HDD

3. **Backup Retention:**
   - [ ] Define PBS retention policy (days/weeks/months)
   - [ ] Define USB archive sync frequency

4. **Migration Schedule:**
   - [ ] Preferred maintenance windows
   - [ ] Stakeholder notification requirements

---

## Conclusion

This storage architecture provides:

1. **Performance Isolation:** Production workloads on NVMe, backups on HDD RAID
2. **Predictable Recovery:** PBS as single authoritative backup source
3. **Disaster Resilience:** USB cold archive for total cluster recovery
4. **Safe Migration:** Two-phase rsync with validation gates
5. **Operational Simplicity:** No complex HA dependencies

**Recommendation:** Proceed with phased implementation, starting with PBS deployment on PVE2.

---

## Document History

| Date | Version | Author | Changes |
|------|---------|--------|---------|
| 2026-01-08 | 1.0 | System | Initial architecture proposal |

---

**Next Steps:**
1. Review and approve architecture
2. Confirm RAID configurations
3. Deploy PBS on PVE2
4. Begin validation testing
5. Schedule NVMe installations
