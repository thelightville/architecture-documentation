# HANDOVER — architecture-documentation

**Target Host**: pve — `172.16.16.20` (this host, no session change needed)  
**Repo**: `git@github.com:thelightville/architecture-documentation.git`  
**Active Branch**: `feature/storage-cleanup-feb2026`  
**Base Branch**: `main`  
**Last Commit**: `e26d0e6` — docs: add SAS RAID storage cleanup report (354GB recovered)  
**Status**: ✅ Clean and in sync with origin

---

## What This Repo Does

Architecture diagrams, routing documentation, and decision records for the Thelightville infrastructure. It is a **docs-only repo** — no code deployed, no services depending on it.

---

## Quick Reference

```bash
# Repo location on pve
cd /root/github-repos/architecture-documentation

# Current branch
git branch
# → feature/storage-cleanup-feb2026

# Switch to main to merge or start new work
git checkout main

# Merge the storage cleanup feature branch
git checkout main
git merge feature/storage-cleanup-feb2026
git push origin main
```

---

## Branch Status

| Branch | Status |
|--------|--------|
| `main` | Base — comprehensive infrastructure architecture (commit `ebdddee`) |
| `feature/storage-cleanup-feb2026` | Active — adds SAS RAID cleanup report (354GB recovered) — **not yet merged to main** |

> **Action needed**: The `feature/storage-cleanup-feb2026` branch should be merged to `main` when the storage cleanup work is considered complete.

---

## Directory Structure

```
architecture-documentation/
├── README.md
└── docs/
    ├── diagrams/
    │   ├── COMPLETE-DOMAIN-ROUTING-MAP.md      ← All 46+ domains → backend mapping
    │   ├── EMAIL-ROUTING-FIXED.md              ← Email routing architecture
    │   ├── NGINX-SMART-ROUTING-CONFIG.md       ← Nginx smart routing config explanation
    │   ├── PORT-2087-ROUTING-DIAGNOSIS.md      ← WHM/cPanel port 2087 routing diagnosis
    │   ├── ROUTING-ARCHITECTURE-DIAGRAM.md     ← High-level traffic flow diagram
    │   └── ROUTING-EXPLAINED.md                ← Plain-English routing explanation
    ├── guides/
    │   ├── HOLISTIC-ARCHITECTURE-OVERVIEW.md   ← RECOMMENDED STARTING POINT
    │   ├── BACKUP-DR-ARCHITECTURE.md           ← Backup/DR strategy and architecture
    │   ├── EMAIL-ARCHITECTURE-CLARIFICATION.md ← Email flow (Mailcow, Sophos, Cloudflare)
    │   ├── PROPOSED-STORAGE-ARCHITECTURE-2026-01-08.md ← Storage migration plan
    │   └── SAFE-MIGRATION-ARCHITECTURE-V2.md   ← Storage migration safety guide
    └── storage-cleanup-2026-02-13.md           ← SAS RAID cleanup report (354GB recovered)
```

---

## Key Documents

| Document | When to Use |
|----------|-------------|
| `docs/guides/HOLISTIC-ARCHITECTURE-OVERVIEW.md` | Start here — full cluster overview |
| `docs/diagrams/COMPLETE-DOMAIN-ROUTING-MAP.md` | Understand how a domain routes to its backend |
| `docs/diagrams/ROUTING-ARCHITECTURE-DIAGRAM.md` | Visual traffic flow (Cloudflare → pve2 → CT) |
| `docs/guides/BACKUP-DR-ARCHITECTURE.md` | Backup and DR strategy reference |
| `docs/storage-cleanup-2026-02-13.md` | SAS RAID cleanup (Feb 2026) — 354GB recovered |

---

## Development Notes

- This repo **stays on pve** — open Remote SSH to `172.16.16.20`, work in `/root/github-repos/architecture-documentation/`
- Docs-only repo: no deployments, no CI/CD, no services depending on it
- Add new architecture decision records (ADRs) or diagrams as separate files in `docs/guides/` or `docs/diagrams/`
- The `feature/storage-cleanup-feb2026` branch was created to track the Feb 2026 SAS RAID cleanup work — merge to `main` when complete
