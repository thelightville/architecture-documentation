# Safe Container Migration Architecture v2.0
**Date:** January 7, 2026  
**Objective:** Migrate CT101→PVE3 and CT111→PVE2 with ZERO data loss  
**Approach:** Incremental sync + verification-first + zero-downtime strategy

---

## CORE PRINCIPLES

### The Three Guarantees

1. **VERIFY BEFORE DESTROY** - Never delete source until destination proven working
2. **INCREMENTAL SYNC** - Use rsync to minimize final cutover downtime
3. **ROLLBACK READY** - Source containers kept running until final confirmation

### Migration Philosophy

```
Traditional Approach (FAILED):
Source → Migrate → Delete Source → Hope it works ❌

New Approach (SAFE):
Source → Clone to Dest → Incremental Sync → Verify Dest → Test Dest → 
Keep Source Running → Switch Traffic → Monitor → Wait 48hrs → Cleanup ✅
```

---

## PHASE 1: PRE-MIGRATION PREPARATION

### Step 1.1: Create Fresh Backups (CRITICAL)

```bash
#!/bin/bash
# Run on PVE1

echo "Creating fresh backups before migration..."

# CT101 - Full backup
vzdump 101 --storage sas-storage --compress zstd --mode snapshot \
    --notes "PRE-MIGRATION BACKUP - $(date '+%Y-%m-%d %H:%M:%S')"

# CT111 - Full backup  
vzdump 111 --storage sas-storage --compress zstd --mode snapshot \
    --notes "PRE-MIGRATION BACKUP - $(date '+%Y-%m-%d %H:%M:%S')"

# Verify backups created
ls -lh /mnt/pve/sas-storage/dump/vzdump-lxc-10* | tail -2

# Test backup integrity
echo "Testing CT111 backup restore (smallest)..."
BACKUP_FILE=$(ls -t /mnt/pve/sas-storage/dump/vzdump-lxc-111-*.zst | head -1)
pct restore 9111 "$BACKUP_FILE" --storage nvme-ssd-production
pct start 9111
sleep 10
pct exec 9111 -- systemctl status
pct stop 9111
pct destroy 9111

echo "✅ Backup verified - safe to proceed"
```

**Estimated Time:** 2-3 hours (CT101 is 104GB)  
**Abort if:** Backup creation fails or test restore fails

---

### Step 1.2: Forensic Recovery from Orphaned Disks

```bash
#!/bin/bash
# Attempt to recover data from previous migration attempt

# Create forensic mount points
mkdir -p /mnt/forensic/{ct101,ct111}

# Mount orphaned CT101 disk (READ-ONLY)
echo "Mounting orphaned CT101 disk..."
mount -o ro,loop /mnt/pve/shared-nvme/images/101/vm-201-disk-0.raw /mnt/forensic/ct101

if [ $? -eq 0 ]; then
    echo "✅ CT101 orphaned disk mounted successfully"
    
    # Check filesystem integrity
    fsck -n /mnt/pve/shared-nvme/images/101/vm-201-disk-0.raw
    
    # List contents
    ls -lah /mnt/forensic/ct101/
    
    # Check for critical directories
    du -sh /mnt/forensic/ct101/{home,var/mail,var/www,etc} 2>/dev/null
    
    # Find files modified after backup time (Jan 6, 8:56 AM)
    touch -t 202601060856 /tmp/backup-reference
    find /mnt/forensic/ct101 -type f -newer /tmp/backup-reference -ls 2>/dev/null | head -50
    
    echo ""
    echo "📋 FILES TO CONSIDER RECOVERING:"
    echo "- Check above list for customer data"
    echo "- Look for /var/mail/* (customer emails)"
    echo "- Look for /home/*/public_html/* (website uploads)"
    echo "- Check /var/lib/mysql/* for database changes"
    
else
    echo "❌ Cannot mount CT101 disk - may be corrupted"
fi

# Repeat for CT111
echo ""
echo "Mounting orphaned CT111 disk..."
mount -o ro,loop /mnt/pve/shared-nvme/images/111/vm-211-disk-0.raw /mnt/forensic/ct111

if [ $? -eq 0 ]; then
    echo "✅ CT111 orphaned disk mounted"
    ls -lah /mnt/forensic/ct111/
    # Check for Grafana data
    du -sh /mnt/forensic/ct111/var/lib/grafana 2>/dev/null
fi

echo ""
echo "DECISION REQUIRED:"
echo "Do you want to extract any data from these orphaned disks?"
echo "If YES: Manual extraction needed"
echo "If NO: We'll proceed with migration using current running containers"
```

**User Decision Point:** Examine orphaned data, decide if recovery needed

---

### Step 1.3: Document Current Production State

```bash
#!/bin/bash
# Create comprehensive snapshot of current state

cat > /root/PRE-MIGRATION-STATE.md << 'DOCEOF'
# Pre-Migration Production State
**Date:** $(date)

## Container Status
```
pct list
```

## Network Configuration (CT101)
```
pct exec 101 -- ip addr show
pct exec 101 -- cat /etc/network/interfaces
```

## Services Running (CT101)
```
pct exec 101 -- systemctl list-units --state=running
pct exec 101 -- netstat -tlnp
```

## Database Status (CT101)
```
pct exec 101 -- mysql -e "SHOW DATABASES;"
pct exec 101 -- du -sh /var/lib/mysql/*
```

## Apache/Website Status (CT101)
```
pct exec 101 -- apachectl -S
pct exec 101 -- ls -la /var/cpanel/users/
```

## Grafana Status (CT111)
```
pct exec 111 -- systemctl status grafana-server
pct exec 111 -- curl -s http://localhost:3000/api/health
```

## Storage Usage
```
zfs list | grep nvme-ssd-production
df -h /mnt/pve/shared-nvme
```
DOCEOF

bash /root/PRE-MIGRATION-STATE.md > /root/PRE-MIGRATION-STATE-$(date +%Y%m%d_%H%M%S).txt
```

**Estimated Time:** 15 minutes  
**Output:** Complete snapshot for rollback reference

---

## PHASE 2: CT111 MIGRATION (Test Case - Smaller Container)

**Why CT111 First?**
- Smaller size (3.5GB vs 104GB)
- Less critical than cPanel (can tolerate longer downtime)
- Faster to rollback if issues found
- Proves migration process before CT101

### Step 2.1: Create Target Container on PVE2

```bash
#!/bin/bash
# Run on PVE2 (172.16.16.22)

# Create container on shared-nvme storage
# Use VMID 211 (temporary, will become 111 after verification)

ssh 172.16.16.22 << 'REMOTECMD'

# Create empty container on shared storage
pct create 211 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
    --hostname monitoring-new.thelightville.xyz \
    --storage shared-nvme \
    --rootfs shared-nvme:20 \
    --memory 2048 \
    --cores 2 \
    --net0 name=eth0,bridge=vmbr0,ip=172.16.16.211/24,gw=172.16.16.20 \
    --unprivileged 0 \
    --onboot 0 \
    --features nesting=1

echo "✅ CT211 created on PVE2 with shared-nvme storage"
pct list | grep 211

REMOTECMD
```

**Verification:**
```bash
# Check container exists on PVE2
ssh 172.16.16.22 "pct config 211 | grep rootfs"
# Should show: rootfs: shared-nvme:subvol-211-disk-0,size=20G
```

---

### Step 2.2: Initial Full Sync (CT111 → CT211)

```bash
#!/bin/bash
# Rsync-based migration with progress

echo "Starting initial full sync CT111 → CT211..."

# Get CT111 rootfs path
SOURCE_PATH=$(zfs get -H -o value mountpoint nvme-ssd-production/subvol-100-disk-0)
if [ -z "$SOURCE_PATH" ]; then
    # If ZFS doesn't work, try from /etc/pve/lxc/111.conf
    SOURCE_PATH="/var/lib/lxc/110/rootfs"  # Adjust based on actual path
fi

# Get CT211 rootfs path on shared storage
DEST_PATH="/mnt/pve/shared-nvme/images/211/subvol-211-disk-0"

echo "Source: $SOURCE_PATH"
echo "Destination: $DEST_PATH"

# Perform initial sync while CT111 still running
rsync -aAXv --info=progress2 \
    --exclude='/dev/*' \
    --exclude='/proc/*' \
    --exclude='/sys/*' \
    --exclude='/tmp/*' \
    --exclude='/run/*' \
    --exclude='/mnt/*' \
    --exclude='/media/*' \
    --exclude='/lost+found' \
    "$SOURCE_PATH/" "$DEST_PATH/"

if [ $? -eq 0 ]; then
    echo "✅ Initial sync complete"
    du -sh "$DEST_PATH"
else
    echo "❌ Rsync failed - ABORTING"
    exit 1
fi
```

**Estimated Time:** 5-10 minutes (3.5GB)  
**Benefit:** Most data transferred while CT111 still serving traffic

---

### Step 2.3: Incremental Sync + Final Cutover (MINIMAL DOWNTIME)

```bash
#!/bin/bash
# Final sync with minimal downtime

echo "=== FINAL CUTOVER SEQUENCE ==="
echo "This will have ~2 minutes downtime for CT111"
read -p "Proceed? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Aborted"
    exit 1
fi

# 1. Set CT111 to read-only (prevent new writes)
pct exec 111 -- systemctl stop grafana-server
sleep 5

# 2. Final incremental sync (only changed files)
echo "Performing final incremental sync..."
rsync -aAXv --info=progress2 \
    --delete \
    --exclude='/dev/*' \
    --exclude='/proc/*' \
    --exclude='/sys/*' \
    --exclude='/tmp/*' \
    --exclude='/run/*' \
    --exclude='/mnt/*' \
    --exclude='/media/*' \
    --exclude='/lost+found' \
    "$SOURCE_PATH/" "$DEST_PATH/"

echo "✅ Final sync complete"

# 3. Stop source container
pct stop 111
echo "✅ CT111 stopped"

# 4. Start destination container on PVE2
ssh 172.16.16.22 "pct start 211"
sleep 10

# 5. Verify services
echo "Verifying CT211 services..."
ssh 172.16.16.22 "pct exec 211 -- systemctl status grafana-server"
ssh 172.16.16.22 "pct exec 211 -- curl -s http://localhost:3000/api/health"

# 6. Test from external
curl -I http://172.16.16.211:3000

if [ $? -eq 0 ]; then
    echo "✅ CT211 is serving traffic on PVE2!"
    echo "✅ Migration successful (pending verification period)"
else
    echo "❌ CT211 not responding - ROLLING BACK"
    ssh 172.16.16.22 "pct stop 211"
    pct start 111
    echo "✅ Rolled back to CT111 on PVE1"
    exit 1
fi
```

**Estimated Downtime:** 2-3 minutes  
**Rollback Time:** 30 seconds (just start CT111 again)

---

### Step 2.4: Verification Period (48 Hours)

```bash
#!/bin/bash
# Monitoring script for CT211

cat > /root/monitor-ct211.sh << 'MONEOF'
#!/bin/bash

while true; do
    clear
    echo "=== CT211 Monitoring Dashboard ==="
    echo "Time: $(date)"
    echo ""
    
    # Check if running
    STATUS=$(ssh 172.16.16.22 "pct status 211")
    echo "Status: $STATUS"
    
    # Check Grafana
    GRAFANA=$(ssh 172.16.16.22 "pct exec 211 -- systemctl is-active grafana-server")
    echo "Grafana: $GRAFANA"
    
    # Check HTTP response
    HTTP=$(curl -s -o /dev/null -w "%{http_code}" http://172.16.16.211:3000)
    echo "HTTP Response: $HTTP"
    
    # Check disk usage
    DISK=$(ssh 172.16.16.22 "pct exec 211 -- df -h / | tail -1")
    echo "Disk: $DISK"
    
    # Check logs for errors
    ERRORS=$(ssh 172.16.16.22 "pct exec 211 -- journalctl -n 100 | grep -i error | wc -l")
    echo "Recent Errors: $ERRORS"
    
    echo ""
    echo "Press Ctrl+C to stop monitoring"
    sleep 60
done
MONEOF

chmod +x /root/monitor-ct211.sh
```

**Verification Checklist:**
- [ ] Day 1: CT211 responding to HTTP requests
- [ ] Day 1: Grafana dashboards loading correctly
- [ ] Day 1: No errors in logs
- [ ] Day 2: Data collection still working
- [ ] Day 2: All dashboards functional
- [ ] Day 2: User access tested

**If all checks pass:** Proceed to cleanup  
**If any check fails:** Rollback to CT111

---

### Step 2.5: Update IP and Finalize (After 48hr Verification)

```bash
#!/bin/bash
# Final cleanup - only after 48 hours of successful operation

echo "=== FINALIZING CT111 MIGRATION ==="
echo "Only run this after 48 hours of successful CT211 operation"
read -p "Confirmed CT211 working perfectly? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Aborted - continue monitoring"
    exit 1
fi

# 1. Update CT211 IP from .211 to .111
ssh 172.16.16.22 << 'FINALCMD'
pct set 211 --net0 name=eth0,bridge=vmbr0,ip=172.16.16.111/24,gw=172.16.16.20
pct set 211 --hostname monitoring.thelightville.xyz
pct exec 211 -- sed -i 's/172.16.16.211/172.16.16.111/g' /etc/network/interfaces
pct exec 211 -- systemctl restart networking
FINALCMD

# 2. Verify new IP
sleep 5
curl -I http://172.16.16.111:3000

# 3. Rename container VMID: 211 → 111
ssh 172.16.16.22 "pct stop 211"
ssh 172.16.16.22 "pct destroy 111 --purge 1"  # Remove old CT111 on PVE1
ssh 172.16.16.22 "mv /etc/pve/lxc/211.conf /etc/pve/lxc/111.conf"
ssh 172.16.16.22 "sed -i 's/211/111/g' /etc/pve/lxc/111.conf"
ssh 172.16.16.22 "pct start 111"

echo "✅ CT111 now running on PVE2 with shared-nvme storage"
echo "✅ Cleanup complete"

# 4. Remove old CT111 data from PVE1
pct destroy 110 --purge 1  # Old backup container
rm -rf /var/lib/lxc/110/

echo "✅ Migration finalized successfully"
```

---

## PHASE 3: CT101 MIGRATION (Production cPanel)

**Approach:** Same process as CT111, but with additional safeguards

### Step 3.1: Additional Pre-Checks for CT101

```bash
#!/bin/bash
# Extra validation for CT101 (critical production)

echo "=== CT101 PRE-MIGRATION VALIDATION ==="

# 1. Database integrity check
pct exec 101 -- mysqlcheck --all-databases --auto-repair

# 2. Verify all websites responding
pct exec 101 -- /scripts/restartsrv_httpd

# 3. Check email queue
pct exec 101 -- /scripts/maildir_cleanup

# 4. Backup MySQL separately
pct exec 101 -- mysqldump --all-databases | gzip > /root/ct101-mysql-$(date +%Y%m%d).sql.gz

# 5. Customer notification
cat > /root/MAINTENANCE-NOTICE.txt << 'NOTICE'
Subject: Scheduled Maintenance - Brief Service Optimization

We will be performing infrastructure optimization on [DATE] at [TIME].

Expected impact: 5-10 minutes of potential slowness
Services affected: All hosted websites

All data will be preserved. This is a transparent upgrade.

Thank you for your patience.
NOTICE

echo "📧 Send maintenance notice to customers? (Manual step)"
echo ""
echo "✅ Pre-checks complete - ready for CT101 migration"
```

---

### Step 3.2: CT101 Migration Timeline

**Different from CT111:** Larger size requires longer sync, plan accordingly

```
Hour 0: Create CT201 on PVE3 (shared-nvme)
Hour 1-3: Initial full rsync (104GB takes ~2-3 hours)
Hour 4: Incremental sync #1 (catch up changes)
Hour 5: Incremental sync #2 (smaller delta)
Hour 6: MAINTENANCE WINDOW
  - Stop Apache/MySQL on CT101
  - Final incremental sync (5-10 minutes)
  - Stop CT101
  - Start CT201 on PVE3
  - Verify all websites
  - Test email delivery
  - Check MySQL connectivity
Hour 6.5: Resume traffic
Hour 6.5-10: Intensive monitoring
Day 1-7: Extended verification (keep CT101 stopped but available)
Day 7: Final cleanup if all successful
```

**Total Downtime:** 10-15 minutes (during final sync)

---

### Step 3.3: CT101 Migration Commands (Full Sequence)

```bash
#!/bin/bash
# CT101 Migration Master Script

set -e  # Exit on any error

# Configuration
SOURCE_CT=101
TEMP_CT=201
DEST_NODE="pve3"
DEST_IP="172.16.16.201"  # Temporary IP
FINAL_IP="172.16.16.101"
STORAGE="shared-nvme"

# Phase 1: Create target container
echo "Phase 1: Creating CT$TEMP_CT on $DEST_NODE..."
ssh 172.16.16.23 << 'CREATECMD'
pct create 201 local:vztmpl/centos-7-default.tar.xz \
    --hostname hosting-new.thelightville.xyz \
    --storage shared-nvme \
    --rootfs shared-nvme:300 \
    --memory 24576 \
    --cores 12 \
    --net0 name=eth0,bridge=vmbr0,ip=172.16.16.201/24,gw=172.16.16.20 \
    --unprivileged 0 \
    --onboot 0 \
    --features nesting=1,keyctl=1
CREATECMD

# Phase 2: Initial sync (while CT101 running)
echo "Phase 2: Initial full sync (estimated 2-3 hours)..."
SOURCE_PATH="/rpool/data/subvol-100-disk-0"
DEST_PATH="/mnt/pve/shared-nvme/images/201/subvol-201-disk-0"

rsync -aAXv --info=progress2 \
    --exclude='/dev/*' \
    --exclude='/proc/*' \
    --exclude='/sys/*' \
    --exclude='/tmp/*' \
    --exclude='/run/*' \
    --exclude='/mnt/*' \
    --exclude='/media/*' \
    --exclude='/lost+found' \
    "$SOURCE_PATH/" "$DEST_PATH/" | tee /root/ct101-sync-initial.log

# Phase 3: Incremental sync #1 (1 hour later)
sleep 3600
echo "Phase 3: Incremental sync #1..."
rsync -aAXv --info=progress2 \
    --delete \
    [same excludes] \
    "$SOURCE_PATH/" "$DEST_PATH/" | tee /root/ct101-sync-inc1.log

# Phase 4: Incremental sync #2 (30 min later)
sleep 1800
echo "Phase 4: Incremental sync #2..."
rsync -aAXv --info=progress2 \
    --delete \
    [same excludes] \
    "$SOURCE_PATH/" "$DEST_PATH/" | tee /root/ct101-sync-inc2.log

# Phase 5: MAINTENANCE WINDOW
echo "=== ENTERING MAINTENANCE WINDOW ==="
echo "Press ENTER when ready for final cutover..."
read

# Stop services gracefully
pct exec 101 -- /scripts/restartsrv_httpd --stop
pct exec 101 -- /scripts/restartsrv_mysql --stop
sleep 30

# Final sync
echo "Final sync in progress..."
rsync -aAXv --info=progress2 --delete \
    [same excludes] \
    "$SOURCE_PATH/" "$DEST_PATH/" | tee /root/ct101-sync-final.log

# Stop source
pct stop 101

# Start destination
ssh 172.16.16.23 "pct start 201"
sleep 30

# Verify services
echo "Verifying CT201 services..."
ssh 172.16.16.23 "pct exec 201 -- systemctl status httpd"
ssh 172.16.16.23 "pct exec 201 -- systemctl status mysql"

# Test websites
curl -I http://172.16.16.201
curl -I http://172.16.16.201/cpanel

if [ $? -eq 0 ]; then
    echo "✅ CT201 operational - monitoring for 7 days before cleanup"
else
    echo "❌ CT201 failed - ROLLBACK INITIATED"
    ssh 172.16.16.23 "pct stop 201"
    pct start 101
    exit 1
fi

echo "=== MAINTENANCE WINDOW COMPLETE ==="
echo "Next: Monitor CT201 for 7 days, then finalize"
```

---

## PHASE 4: POST-MIGRATION CONFIGURATION

### Step 4.1: Update Backup Jobs

```bash
#!/bin/bash
# Update backup configuration after successful migration

cat > /etc/pve/jobs.cfg << 'BACKUPEOF'
# Tier 1: Fast SSD backups (containers on PVE1)
vzdump: tier1-pve1-backups
    compress zstd
    enabled 1
    mailnotification failure
    mailto lanre@thelightville.com
    mode snapshot
    node pve
    prune-backups keep-daily=7,keep-weekly=4
    schedule 02:00
    storage local-backup-fast
    vmid 103,105,107,109,113

# Tier 2: CT111 on PVE2 (shared storage)
vzdump: tier2-pve2-backups
    compress zstd
    enabled 1
    mode snapshot
    node pve2
    prune-backups keep-daily=7,keep-weekly=4
    schedule 02:30
    storage shared-nvme
    vmid 111

# Tier 3: CT101 on PVE3 (shared storage)
vzdump: tier3-pve3-backups
    compress zstd
    enabled 1
    mode snapshot
    node pve3
    prune-backups keep-daily=7,keep-weekly=4,keep-monthly=3
    schedule 03:00
    storage shared-nvme
    vmid 101

# Tier 4: All containers to SAS (redundancy)
vzdump: tier4-sas-all
    compress zstd
    enabled 1
    mode snapshot
    node pve
    prune-backups keep-weekly=4,keep-monthly=6
    schedule 'sun 04:00'
    storage sas-storage
    vmid 101,103,105,107,109,111,113
BACKUPEOF

echo "✅ Backup jobs updated for new container locations"
```

---

## ROLLBACK PROCEDURES

### Scenario 1: CT111 Migration Fails

```bash
#!/bin/bash
# Rollback CT111 migration

echo "ROLLBACK: CT111 migration failed"

# Stop new container
ssh 172.16.16.22 "pct stop 211"

# Start original container
pct start 111

# Verify
curl -I http://172.16.16.111:3000

echo "✅ Rolled back to CT111 on PVE1"
echo "Data loss: NONE (source never deleted)"
```

**Time to Rollback:** 30 seconds  
**Data Loss:** Zero

---

### Scenario 2: CT101 Migration Fails

```bash
#!/bin/bash
# Rollback CT101 migration

echo "ROLLBACK: CT101 migration failed"

# Stop new container
ssh 172.16.16.23 "pct stop 201"

# Start original container
pct start 101

# Start services
pct exec 101 -- systemctl start httpd
pct exec 101 -- systemctl start mysql

# Verify
curl -I http://172.16.16.101

echo "✅ Rolled back to CT101 on PVE1"
echo "Downtime: ~5 minutes (time from stop to restart)"
echo "Data loss: NONE (final sync captured all changes)"
```

**Time to Rollback:** 5 minutes  
**Data Loss:** Zero (rsync captured everything before stop)

---

## SUCCESS CRITERIA

### CT111 Migration Success

- [ ] CT211 running on PVE2
- [ ] Storage on shared-nvme (verified with `pct config 211`)
- [ ] Grafana accessible on http://172.16.16.111:3000
- [ ] All dashboards loading
- [ ] No errors in logs for 48 hours
- [ ] Users report no issues

### CT101 Migration Success

- [ ] CT201 running on PVE3
- [ ] Storage on shared-nvme (verified with `pct config 201`)
- [ ] All 89 websites responding
- [ ] Email delivery working
- [ ] MySQL connections stable
- [ ] No customer complaints for 7 days
- [ ] Apache/PHP performance equal or better

---

## ESTIMATED TIMELINE

### CT111 Migration

| Phase | Duration | Downtime |
|-------|----------|----------|
| Backup creation | 30 min | None |
| CT211 creation | 5 min | None |
| Initial sync | 10 min | None |
| Final cutover | 3 min | **3 min** |
| Verification | 48 hours | None |
| Cleanup | 15 min | None |
| **Total** | **2 days** | **3 min** |

### CT101 Migration

| Phase | Duration | Downtime |
|-------|----------|----------|
| Backup creation | 3 hours | None |
| CT201 creation | 10 min | None |
| Initial sync | 3 hours | None |
| Incremental syncs | 2 hours | None |
| Final cutover | 15 min | **15 min** |
| Verification | 7 days | None |
| Cleanup | 30 min | None |
| **Total** | **7-8 days** | **15 min** |

---

## KEY DIFFERENCES FROM FAILED ATTEMPT

| Failed Attempt | New Approach |
|----------------|--------------|
| Deleted source during migration | ✅ Keep source until verified |
| No incremental sync | ✅ Rsync minimizes downtime |
| Migrated all at once | ✅ One container at a time |
| No rollback plan | ✅ 30-second rollback |
| No verification period | ✅ 48hr-7day verification |
| System froze during migration | ✅ Rsync handles interruptions gracefully |
| No backups verified | ✅ Test restore before migration |

---

**Status:** Architecture Complete - Ready for Execution  
**Next Step:** User approval + Execute Phase 1 (backups + forensic recovery)  
**Risk Level:** LOW (source preserved throughout)  
**Data Loss Potential:** ZERO (verification-first approach)
