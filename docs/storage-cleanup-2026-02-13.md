# Storage Cleanup Report - February 13, 2026

## Executive Summary

Successfully executed high-priority storage cleanup on PVE1 SAS RAID storage, recovering **354GB** of usable space and reducing utilization from 71% to 34%.

## Storage Status

### Before Cleanup
- **Capacity**: 1007GB
- **Used**: 675GB (71%)
- **Available**: 282GB
- **Status**: ⚠️ Approaching capacity limits

### After Cleanup
- **Capacity**: 1007GB  
- **Used**: 321GB (34%)
- **Available**: 635GB
- **Status**: ✅ Healthy - 125% increase in free space

## Cleanup Operations

### 1. S3 Music Backup Archive (253GB recovered)
**Source**: `/mnt/sas-storage/s3-music-backup`  
**Destination**: `/mnt/backup-primary/archived-s3-backup`

- **Files archived**: 159,736 files
- **Directories**: 
  - `audio/` - 765MB
  - `custom-upload/` - 119GB
  - `genius-audio/` - 119GB
  - `upload/` - 9.1GB
  - `music/` - 5.4GB
  - `.covers/` - 360KB
- **Date range**: June 2023 S3 bucket backup
- **Transfer speed**: 148-158 MB/s (USB 3.0)
- **Integrity verified**: ✅ Size match, file count match
- **Action taken**: Source deleted after verification

### 2. cPanel Migration Backups Archive (70GB recovered)
**Source**: `/mnt/sas-storage/cpanel-backups`  
**Destination**: `/mnt/backup-primary/cpanel-migration-backups`

- **Files archived**: 22 website backup archives
- **Largest backups**:
  - `cpmove-onlineradio.tar.gz` - 27GB
  - `cpmove-isaudit.tar.gz` - 17GB  
  - `cpmove-sethhsec.tar.gz` - 12GB
- **Purpose**: One-time migration backups from October 2024
- **Transfer speed**: 198 MB/s (USB 3.0)
- **Integrity verified**: ✅ Size match (70GB source = 70GB archive)
- **Action taken**: Source deleted after verification

### 3. Dump Directory Cleanup (31GB recovered)
**Location**: `/mnt/sas-storage/dump`

**Actions taken**:
- ✅ Deleted old VM100 backup from Feb 7 (22GB)
- ✅ Deleted container backups older than 7 days (9GB)
- ✅ Kept recent backups for disaster recovery

**Remaining backups** (32GB total):
- `vzdump-qemu-100-2026_02_10-03_30_03.vma.zst` - 23GB (Feb 10)
- `vzdump-lxc-103-2026_02_07-03_35_35.tar.zst` - 5.8GB (Feb 7)
- `vzdump-lxc-119-2026_02_06-05_20_18.tar.zst` - 2.0GB (Feb 6)
- `vzdump-lxc-105-2026_02_06-03_40_22.tar.zst` - 2.3GB (Feb 6)

## Archive Storage (LaCie USB)

### Storage Pool: `/mnt/backup-primary`
- **Device**: LaCie d2 Quadra v3C 5.5TB
- **Connection**: USB 3.0, Bus 004, 5000Mbps
- **Current usage**: 1.5TB / 5.5TB (29%)
- **Available**: 3.7TB
- **Performance**: Optimized from USB 2.0 (480Mbps) on Feb 12 - **10.4x faster**

### Archives Created
```
253GB  /mnt/backup-primary/archived-s3-backup
70GB   /mnt/backup-primary/cpanel-migration-backups
```

### NFS Exports (Multi-node access)
- **PVE1**: Local mount at `/mnt/backup-primary`
- **PVE2**: NFS mount at `/mnt/nfs/nfs-lacie-usb`
- **PVE3**: NFS mount at `/mnt/nfs/nfs-lacie-usb`
- **Protocol**: NFS v3 and v4.2

## Current SAS RAID Contents (321GB)

### Active Storage
1. **PBS Datastore**: 274GB
   - Location: `/mnt/sas-storage/pbs-datastore`
   - Purpose: Proxmox Backup Server automated backups
   - Retention: Per PBS configuration

2. **Dump Directory**: 32GB
   - Recent manual VM/CT backups
   - Retention: Keep latest + recent snapshots

3. **Other**: ~15GB
   - ISO images
   - Container templates
   - Miscellaneous files

## Performance Metrics

### Transfer Speeds (USB 3.0)
- S3 backup rsync: 148-158 MB/s sustained
- cPanel backup rsync: 198 MB/s sustained
- **Estimated time saved vs USB 2.0**: ~85 minutes (10.4x faster)

### Total Operation Time
- S3 backup: ~18 minutes (253GB)
- cPanel backup: ~6 minutes (70GB)
- Dump cleanup: <1 minute
- **Total**: ~25 minutes for 354GB cleanup

## Recommendations

### Immediate
- ✅ Monitor SAS RAID storage monthly
- ✅ Implement automated cleanup for dump directory (>30 days)
- ✅ Keep dump retention policy at 7 days maximum

### Short-term
- 📋 Document backup retention policies in PBS
- 📋 Create automated archival script for old dump backups
- 📋 Set up monitoring alerts at 50% SAS storage usage

### Long-term
- 📋 Consider additional archival storage for older PBS snapshots
- 📋 Evaluate dump directory necessity (PBS provides comprehensive backups)
- 📋 Implement lifecycle policies for automatic archival

## Backup Redundancy Status

### Current Protection Levels
1. **PBS Automated Backups** (274GB on SAS RAID)
   - Daily automated snapshots
   - Datastore: `sas-datastore`
   - Container: CT115 on dedicated 128GB SSD

2. **Manual Dump Backups** (32GB on SAS RAID)
   - Recent VM/CT snapshots
   - 7-day retention
   - Quick restore capability

3. **LaCie USB Archives** (323GB)
   - Long-term archival
   - Historical data preservation
   - NFS accessible across cluster

## Related Infrastructure Changes

### Recent Optimizations (Feb 12-13, 2026)
- ✅ CT115 (PBS) migrated to dedicated 128GB SSD for NVMe independence
- ✅ LaCie USB upgraded to USB 3.0 port (10.4x performance improvement)
- ✅ NFS exports re-established on all cluster nodes
- ✅ PCI slot reorganization (NVMe, GPU, 10GbE network card)
- ✅ EFI boot UUID conflict resolved (128GB SSD)

### Hardware Configuration
- **Node**: PVE1 (Dell Precision Tower 5810)
- **CPU**: Xeon E5-2698 v3
- **Storage Pools**:
  - NVMe: 952GB ZFS (13% used) - Production VMs/CTs
  - SATA Mirror: 952GB ZFS (0.5% used) - CT117 docker-host
  - 128GB SSD: ext4 (2.5% used) - CT115 PBS container
  - SAS RAID: 1TB LVM (34% used) - PBS datastore + dump
  - LaCie USB: 5.5TB ext4 (29% used) - Archives

## Verification Commands

### Check SAS Storage Status
```bash
df -h /mnt/sas-storage
du -sh /mnt/sas-storage/{dump,pbs-datastore}
```

### Verify Archives
```bash
du -sh /mnt/backup-primary/archived-s3-backup
du -sh /mnt/backup-primary/cpanel-migration-backups
find /mnt/backup-primary/archived-s3-backup -type f | wc -l  # Should be 159736
```

### Monitor Storage Usage
```bash
# SAS RAID utilization
df -h | grep sas-storage

# LaCie USB available space
df -h | grep backup-primary

# Recent backups in dump
ls -lht /mnt/sas-storage/dump/*.zst | head -10
```

## Conclusion

Storage cleanup operation successfully completed with:
- ✅ **354GB** reclaimed from SAS RAID (71% → 34% utilization)
- ✅ **323GB** safely archived to LaCie USB with verified integrity
- ✅ **635GB** free space available on SAS RAID
- ✅ All critical backups preserved in PBS and dump directories
- ✅ USB 3.0 optimization enabled fast, efficient archival operations

The SAS RAID storage is now in healthy condition with ample headroom for growth. Regular monitoring and automated cleanup policies will maintain optimal storage utilization.

---

**Date**: February 13, 2026  
**Operator**: Infrastructure Team  
**Status**: ✅ Complete  
**Next Review**: March 2026
