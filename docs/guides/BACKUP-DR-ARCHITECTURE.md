# 🏗️ Backup & Disaster Recovery Architecture

**Last Updated:** January 13, 2026  
**Infrastructure:** 3-Node Proxmox Cluster + Cloud DR

---

## 📊 Current Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    PROXMOX CLUSTER                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│  │   pve    │    │   pve2   │    │   pve3   │            │
│  │ (Primary)│    │(Services)│    │ (Hosting)│            │
│  │ 32c/125GB│    │ 20c/31GB │    │ 4c/62GB  │            │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘            │
│       │               │               │                    │
│  5 Containers    CT115 (PBS)     CT101 (cPanel)           │
│  CT103,105,      CT111 (Monitor)                          │
│  107,109,113                                              │
│       │               │               │                    │
└───────┼───────────────┼───────────────┼────────────────────┘
        │               │               │
        │               │               │
        ▼               ▼               ▼
┌──────────────────────────────────────────────────────────┐
│             LOCAL BACKUP LAYER (Tier 1)                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────┐         │
│  │ PBS (CT115 on pve2) - Orchestrator         │         │
│  │ IP: 172.16.16.115                          │         │
│  │ WebUI: https://pbs.thelightville.xyz:8007  │         │
│  ├────────────────────────────────────────────┤         │
│  │ • Schedules daily backups (2:00 AM)        │         │
│  │ • Coordinates backups across all nodes     │         │
│  │ • Deduplication & compression              │         │
│  │ • Retention: 7 daily, 4 weekly, 3 monthly  │         │
│  └────────────────┬───────────────────────────┘         │
│                   │                                      │
│                   ▼                                      │
│  ┌────────────────────────────────────────────┐         │
│  │ 5.5TB LaCie Drive (on pve - Primary Node)  │         │
│  │ Mount: /mnt/backup-primary (/dev/sde2)     │         │
│  │ Used: 3.2TB / 5.5TB (62%)                  │         │
│  ├────────────────────────────────────────────┤         │
│  │ Storage:                                   │         │
│  │ • /dump/ - All container backups (2.5TB)   │         │
│  │ • VM200 backups (45GB each)                │         │
│  │ • Configuration snapshots                  │         │
│  │ • SSL certificates                         │         │
│  └────────────────────────────────────────────┘         │
│                                                          │
└──────────────────────────────────────────────────────────┘
                       │
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│          CRITICAL CONFIG BACKUP (Tier 2)                 │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────┐         │
│  │ AWS S3 (new-aws profile)                   │         │
│  │ Bucket: thelightville-critical-backups     │         │
│  │ Cost: $0/month (Free Tier - 5GB)           │         │
│  ├────────────────────────────────────────────┤         │
│  │ Daily: 2:00 AM (backup-to-s3.sh)           │         │
│  │ Source: pve (Primary Node)                 │         │
│  │ Content:                                   │         │
│  │ • SSH keys (all 3 nodes)                   │         │
│  │ • Infrastructure scripts                   │         │
│  │ • AWS/GCP/Cloudflare credentials           │         │
│  │ • Documentation                            │         │
│  │ • Cluster configs                          │         │
│  │ Retention: 10 days rotation                │         │
│  │ Size: ~500MB                               │         │
│  └────────────────────────────────────────────┘         │
│                                                          │
└──────────────────────────────────────────────────────────┘
                       │
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│      OFF-SITE DISASTER RECOVERY (Tier 3)                │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────┐         │
│  │ Backblaze B2 Cloud Storage                 │         │
│  │ Bucket: proxmox-dr-backups                 │         │
│  │ Cost: $1.30/month (10GB free + 216GB paid) │         │
│  ├────────────────────────────────────────────┤         │
│  │ Daily: 3:00 AM (b2-dr-backup.sh)           │         │
│  │ Source: pve /mnt/backup-primary/dump/      │         │
│  │ Method: rclone sync (incremental)          │         │
│  │ Content:                                   │         │
│  │ ✅ CT101 (cPanel) - 102GB                  │         │
│  │ ✅ CT103 (OnlineRadio app) - 6GB           │         │
│  │ ✅ CT105 (ClaApp) - 1.6GB                  │         │
│  │ ✅ CT107 (OnlineRadio API) - 1.3GB         │         │
│  │ ✅ CT109 (Streaming) - 115GB               │         │
│  │ ✅ CT113 (Chatbot) - 0.6GB                 │         │
│  │ ❌ VM200 (EXCLUDED - local only)           │         │
│  │ Total: ~226GB                              │         │
│  │ Retention: 90 days                         │         │
│  │ Recovery: Instant (no delays)              │         │
│  └────────────────────────────────────────────┘         │
│                                                          │
└──────────────────────────────────────────────────────────┘

```

---

## 🎯 Which Node Runs What?

### **pve (Primary Node - 32 cores/125GB RAM)**
**Role:** Primary coordinator and storage host

**Runs:**
- ✅ 5 production containers (CT103, 105, 107, 109, 113)
- ✅ **Backup storage:** 5.5TB LaCie (/mnt/backup-primary)
- ✅ **Cloud backup scripts:**
  - backup-to-s3.sh (2:00 AM)
  - b2-dr-backup.sh (3:00 AM)
  - backup-monitoring-stack.sh (4:00 AM)

**Why pve?**
- ✅ Has the 5.5TB LaCie drive physically attached
- ✅ Most powerful node (32 cores)
- ✅ Houses PBS dump directory at /mnt/backup-primary/dump/
- ✅ Direct access to backup files (no network transfer)
- ✅ Primary cluster node (Node ID: 1)

---

### **pve2 (Services Node - 20 cores/31GB RAM)**
**Role:** Infrastructure services

**Runs:**
- ✅ **CT115:** PBS (Proxmox Backup Server) at 172.16.16.115
- ✅ **CT111:** Monitoring (Prometheus/Grafana)

**Why pve2?**
- ✅ Dedicated to infrastructure services
- ✅ Isolates backup orchestration from production workloads
- ✅ CT115 PBS manages backup schedule and retention
- ✅ Can coordinate backups across all cluster nodes

---

### **pve3 (Hosting Node - 4 cores/62GB RAM)**
**Role:** cPanel hosting platform

**Runs:**
- ✅ CT101 (cPanel hosting)

**Why pve3?**
- ✅ Isolated hosting environment
- ✅ High RAM for client workloads
- ✅ Separate from main production apps

---

## 🔄 How PBS (CT115) Fits Into the Architecture

### **PBS Role: Backup Orchestrator**

**What PBS Does:**
1. **Schedules automated backups** at 2:00 AM daily
2. **Coordinates backups** across all 3 Proxmox nodes
3. **Stores backups** to pve's 5.5TB LaCie (/mnt/backup-primary)
4. **Manages retention:**
   - Keep-daily: 7 (last 7 days)
   - Keep-weekly: 4 (last 4 weeks)
   - Keep-monthly: 3 (last 3 months)
5. **Deduplicates** backup data to save space
6. **Compresses** backups (zstd/lzo)
7. **Verifies** backup integrity

**PBS Configuration:**
- **Container:** CT115 on pve2
- **IP:** 172.16.16.115 (internal)
- **WebUI:** https://pbs.thelightville.xyz:8007
- **Datastore:** backup-store → /mnt/backup-primary on pve
- **Storage Path:** pve:/mnt/backup-primary/dump/

**How PBS Talks to pve's Storage:**
```
CT115 (pve2) ──[PBS Protocol]──> pve:/mnt/backup-primary
     │                                    │
     │                                    ▼
     └─────> Schedules & orchestrates    5.5TB LaCie
             backups across cluster       /dump/ directory
```

---

## 📋 Backup Flow: From VM to Cloud

### **Step-by-Step Backup Process:**

```
1. 2:00 AM - PBS (CT115) Wakes Up
   ├─> Initiates backup jobs for all containers/VMs
   ├─> Connects to each node (pve, pve2, pve3)
   └─> Snapshots containers: CT101, 103, 105, 107, 109, 111, 113, 115, VM200

2. PBS Backup Process (2:00-2:30 AM)
   ├─> Compresses snapshots (zstd/lzo)
   ├─> Transfers to pve:/mnt/backup-primary/dump/
   ├─> Creates: vzdump-lxc-XXX-YYYY_MM_DD-HH_MM_SS.tar.zst
   ├─> Deduplicates blocks
   └─> Verifies backup integrity

3. 2:00 AM - AWS S3 Critical Config Backup (pve)
   ├─> Script: /root/scripts/backup-to-s3.sh
   ├─> Source: /root (SSH keys, scripts, docs)
   ├─> Destination: s3://thelightville-critical-backups
   ├─> Email: Start + Completion to admin@thelightville.com
   └─> Size: ~50MB

4. 3:00 AM - Backblaze B2 DR Backup (pve)
   ├─> Script: /root/scripts/b2-dr-backup.sh
   ├─> Source: /mnt/backup-primary/dump/ (PBS output)
   ├─> Destination: backblaze-b2:proxmox-dr-backups
   ├─> Excludes: VM200 (qemu-200*)
   ├─> Method: rclone sync (incremental)
   ├─> Transfers: 4 parallel uploads
   ├─> Email: Start + Completion with stats
   └─> Size: ~226GB (first run), incremental after

5. 4:00 AM - Monitoring Stack Backup (pve)
   ├─> Script: /root/scripts/backup-monitoring-stack.sh
   ├─> Source: CT111 (Prometheus/Grafana configs)
   ├─> Destination: /root/backups/monitoring
   ├─> Email: Start + Completion
   └─> Size: ~100MB
```

---

## 💾 Storage Allocation

### **Local Storage (pve - 5.5TB LaCie)**
```
/mnt/backup-primary/dump/
├── Container Backups: ~2.5TB
│   ├── CT101 (cPanel): 102GB per snapshot
│   ├── CT103 (OnlineRadio): 6GB
│   ├── CT105 (ClaApp): 1.6GB
│   ├── CT107 (API): 1.3GB
│   ├── CT109 (Streaming): 115GB
│   ├── CT111 (Monitoring): 2GB
│   ├── CT113 (Chatbot): 0.6GB
│   └── CT115 (PBS): 1GB
│
├── VM200 (Windows Dev): ~45GB per snapshot
│
└── Multiple retention cycles:
    ├── Daily: 7 snapshots
    ├── Weekly: 4 snapshots
    └── Monthly: 3 snapshots

Total Used: 3.2TB / 5.5TB (62%)
Remaining: 2.3TB (38% free)
```

### **Cloud Storage**

**AWS S3 (Critical Configs):**
- Size: 500MB (10-day rotation)
- Cost: $0/month (Free Tier)
- Purpose: Disaster recovery access credentials

**Backblaze B2 (DR Backups):**
- Size: 226GB (containers only, no VM200)
- Cost: $1.30/month (10GB free + 216GB @ $0.006/GB)
- Purpose: Off-site disaster recovery
- Retention: 90 days

---

## 🔐 Why This Architecture?

### **Multi-Tier Defense:**

**Tier 1 - Local PBS (Fast Recovery):**
- ✅ Instant restore (<1 hour for any container)
- ✅ Network-speed recovery (10Gbps local)
- ✅ 7-day daily snapshots (accidental deletion protection)
- ✅ Multiple retention cycles (daily/weekly/monthly)
- ⚠️ Vulnerable to: Node failure, fire, theft

**Tier 2 - AWS S3 Critical Configs (Emergency Access):**
- ✅ If cluster totally lost, still have:
  - SSH keys to access Oracle VPN
  - AWS/GCP credentials to restore from Backblaze
  - Infrastructure scripts to rebuild
- ✅ Fast download (small size)
- ⚠️ Only configs, not full backups

**Tier 3 - Backblaze B2 (True Disaster Recovery):**
- ✅ Geographic separation (off-site)
- ✅ Survives total datacenter loss
- ✅ 90-day retention (ransomware protection)
- ✅ Instant recovery (no glacier delays)
- ⚠️ Large download time (hours for 226GB)

---

## ⚡ Recovery Scenarios

### **Scenario 1: Single Container Corruption**
**Recovery Time: 5-15 minutes**
```
1. Go to PBS WebUI (pbs.thelightville.xyz:8007)
2. Select container backup
3. Restore to original or new CT ID
4. Start container
✅ Data loss: 0-24 hours (since last backup)
```

### **Scenario 2: Node Failure (pve goes down)**
**Recovery Time: 1-2 hours**
```
1. Migrate containers from pve to pve2/pve3
2. If pve unrecoverable:
   - Restore from PBS backups to other nodes
   - Reattach 5.5TB LaCie to pve2
   - Update PBS storage path
✅ All backups intact on LaCie
✅ Cloud backups still available
```

### **Scenario 3: Complete Cluster Loss (Fire/Theft)**
**Recovery Time: 4-8 hours**
```
1. Get AWS S3 credentials from another location
2. Download critical configs from S3
3. Setup new Proxmox node
4. Configure rclone with Backblaze credentials (from S3)
5. Download containers from Backblaze B2:
   - rclone copy backblaze-b2:proxmox-dr-backups /restore/
6. Import containers to new Proxmox
7. Restore Cloudflare DNS (configs from S3)
✅ Full recovery possible
✅ Data loss: Max 24 hours
⚠️ Requires: Internet + new hardware
```

### **Scenario 4: Ransomware Attack**
**Recovery Time: 2-4 hours**
```
1. Identify when infection occurred
2. Restore from Backblaze B2 backup BEFORE infection
3. 90-day retention = can go back 3 months
4. Restore clean containers
5. Audit and patch vulnerability
✅ Can recover from old, clean backups
✅ 90-day history protects against delayed ransomware
```

---

## 📧 Monitoring & Notifications

**All backup jobs send email to:** admin@thelightville.com

**Daily Email Schedule:**
- 2:00 AM - AWS S3 backup start
- 2:05 AM - AWS S3 backup complete
- 3:00 AM - Backblaze B2 backup start
- 3:30 AM - Backblaze B2 backup complete (with stats)
- 4:00 AM - Monitoring backup start
- 4:02 AM - Monitoring backup complete

**What Emails Include:**
- ✅ Start/completion timestamps
- ✅ Backup sizes
- ✅ Storage usage stats
- ✅ Error notifications (if any)
- ✅ Log file locations

---

## 💰 Cost Breakdown

### **Monthly Costs:**
- **Local PBS:** $0 (5.5TB LaCie already owned)
- **AWS S3:** $0 (Free Tier)
- **Backblaze B2:** $1.30 (DR backups)
- **Cloudflare R2:** $2.73 (OnlineRadio media - separate)
- **Total Backup Costs:** $1.30/month

### **Annual Backup Costs:**
- $15.60/year protecting $10,000+ infrastructure
- **ROI:** 0.16% of infrastructure value

---

## 🎯 Best Practices Being Followed

✅ **3-2-1 Backup Rule:**
- 3 copies of data (Local PBS + AWS S3 + Backblaze B2)
- 2 different storage types (Local disk + Cloud)
- 1 off-site copy (Backblaze B2)

✅ **Automated & Monitored:**
- Daily automated backups
- Email notifications on every job
- No manual intervention required

✅ **Multiple Retention Periods:**
- Local: 7 daily, 4 weekly, 3 monthly
- Cloud: 90 days
- Critical configs: 10 days

✅ **Cost Optimized:**
- VM200 excluded from cloud (dev environment)
- Free tiers maximized (AWS S3, Backblaze 10GB)
- Incremental cloud backups (not full)

✅ **Geographically Distributed:**
- Local: On-premise
- AWS: Multi-region
- Backblaze: Off-site cloud

---

## 📊 Key Metrics

**Backup Coverage:**
- Containers: 9/9 (100%)
- VMs: 1/1 local only (VM200)
- Cloud-backed: 6 containers (226GB)

**Recovery Objectives:**
- **RTO** (Recovery Time Objective):
  - Local: 15 minutes
  - Cloud: 4-8 hours
- **RPO** (Recovery Point Objective):
  - Daily backups = Max 24 hours data loss

**Storage Efficiency:**
- PBS deduplication saves ~30-40%
- Cloud storage: 226GB (vs 271GB without exclusions)
- Cost: $1.30/month (vs $3.00 GCP previously)

---

## ✅ Summary

**Why This Works:**

1. **PBS orchestrates** backups from CT115 on pve2
2. **Storage lives on pve** (5.5TB LaCie with backups)
3. **Cloud sync runs on pve** (direct access to /mnt/backup-primary)
4. **Geographic separation** via Backblaze B2
5. **Critical configs** backed up separately to AWS S3
6. **Email monitoring** for all backup jobs
7. **Cost optimized** by excluding VM200 from cloud

**This architecture provides:**
- ✅ Fast local recovery (PBS)
- ✅ Off-site disaster recovery (Backblaze)
- ✅ Emergency access credentials (AWS S3)
- ✅ Multi-tier protection (3-2-1 rule)
- ✅ Automated and monitored
- ✅ Cost effective ($1.30/month)

---

**Current Status:** All tiers operational ✅  
**Next Backup:** Tomorrow 2:00 AM  
**Monitoring:** Email notifications active
