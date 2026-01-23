# Architecture Documentation

Comprehensive infrastructure architecture documentation, routing diagrams, email architecture, storage design, and migration strategies.

## 🏗️ Overview

Complete architectural documentation covering:
- **Network Routing**: Domain routing, NGINX configurations, port mapping
- **Email Architecture**: Mail routing, SMTP flow, DNS configurations
- **Storage Design**: Backup/DR architecture, storage strategies
- **Migration Architecture**: Safe migration patterns and procedures

## 📋 Contents

### Routing Diagrams
- **COMPLETE-DOMAIN-ROUTING-MAP.md**: Complete domain routing infrastructure map
- **EMAIL-ROUTING-FIXED.md**: Email routing configuration fixes
- **NGINX-SMART-ROUTING-CONFIG.md**: NGINX reverse proxy routing
- **PORT-2087-ROUTING-DIAGNOSIS.md**: Port routing diagnostics
- **ROUTING-ARCHITECTURE-DIAGRAM.md**: Network routing architecture
- **ROUTING-EXPLAINED.md**: Routing architecture explanation

### Architecture Guides
- **BACKUP-DR-ARCHITECTURE.md**: Backup and disaster recovery architecture
- **EMAIL-ARCHITECTURE-CLARIFICATION.md**: Email infrastructure architecture
- **HOLISTIC-ARCHITECTURE-OVERVIEW.md**: Complete infrastructure overview
- **PROPOSED-STORAGE-ARCHITECTURE-2026-01-08.md**: Storage architecture proposal
- **SAFE-MIGRATION-ARCHITECTURE-V2.md**: Safe migration architecture patterns

## 🔗 Infrastructure Components

- **Proxmox Cluster**: 3-node HA cluster (pve, pve2, pve3)
- **Networking**: Sophos XG firewall, NGINX reverse proxy, Cloudflare tunnels
- **Storage**: NFS, PBS (Proxmox Backup Server), Backblaze B2, Cloudflare R2
- **Email**: SMTP2GO, cPanel mail, Cloudflare email routing

## 📚 Related Repositories

- [networking-firewall](https://github.com/thelightville/networking-firewall): Firewall and routing configs
- [cluster-management](https://github.com/thelightville/cluster-management): Cluster architecture
- [backup-disaster-recovery](https://github.com/thelightville/backup-disaster-recovery): Backup infrastructure

## 📝 Notes

Architecture evolved from single-server to distributed HA cluster with multi-region backup strategy.
