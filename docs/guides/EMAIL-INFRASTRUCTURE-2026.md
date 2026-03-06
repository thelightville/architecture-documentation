# Email Infrastructure — Definitive Guide
**Last Updated:** March 6, 2026  
**Supersedes:** `EMAIL-ARCHITECTURE-CLARIFICATION.md` (Nov 2025), `EMAIL-ROUTING-FIXED.md` (Nov 2025)

---

## Architecture Overview

All email for Thelightville domains is handled by **Mailcow Dockerized** on **CT201** (172.16.16.201). Outbound email relays through **SMTP2GO** to avoid IP reputation issues.

```
              INBOUND                                    OUTBOUND
              ───────                                    ────────

  Sender MTA                                         Recipient MTA
      │                                                    ▲
      ▼                                                    │
  mail.thelightville.net                              mail.smtp2go.com
  (A: 142.180.236.143)                                (port 2525)
      │                                                    ▲
      ▼                                                    │
  PVE Node (VIP holder)                              CT201 Postfix
  nginx stream :25                                   sender_dependent_relayhost
      │                                                    ▲
      ▼                                                    │
  localhost:2525 ──(SSH tunnel)──▶ CT201:25           Mailcow sends
      │                     (or Postfix relay)       via SMTP2GO per domain
      ▼
  CT201 Mailcow Postfix
      │
      ▼
  Dovecot (LMTP :24)
      │
      ▼
  User Mailbox
```

---

## Domain Categories

### Category A: Mailcow-Primary Domains (MX → mail.thelightville.net)
These domains receive mail directly to CT201 Mailcow.

| Domain | MX Record | Notes |
|--------|-----------|-------|
| cordeiservices.com | 10 mail.thelightville.net | Primary Mailcow domain |
| 123accountingsolutions.com.ng | 10 mail.thelightville.net | |
| bajantropicaltreats.com | 10 mail.thelightville.net | |
| dujourevents.com | 10 mail.thelightville.net | |
| electricmall.com.ng | 10 mail.thelightville.net | |
| emergeglobalservices.com | 10 mail.thelightville.net | |
| fastrackenergy.com | 10 mail.thelightville.net | |
| lacasaristorante.com | 10 mail.thelightville.net | |
| mother2mother.com.ng | 10 mail.thelightville.net | |
| opentechnologies.com.ng | 10 mail.thelightville.net | |
| sethlabs.com | 10 mail.thelightville.net | |
| tnfc.com.ng | 10 mail.thelightville.net | |
| awomanswants.com | 10 mail.thelightville.net | |
| aiinbits.com | 10 mail.thelightville.net | |
| thelightville.com.ng | 10 mail.thelightville.net | |
| thelightville.ng | 10 mail.thelightville.net | |

### Category B: Microsoft 365 Domains (MX → *.mail.protection.outlook.com)
These domains receive mail through Microsoft 365. Mailcow has mailboxes as **backup** for future migration. **Do NOT change their MX records.**

| Domain | MX Record | Notes |
|--------|-----------|-------|
| thelightville.com | thelightville-com.mail.protection.outlook.com | Primary org domain |
| thelightville.net | thelightville-net.mail.protection.outlook.com | |
| thelightville.org | thelightville-org.mail.protection.outlook.com | |
| thelightville.xyz | thelightville-xyz.mail.protection.outlook.com | |
| haodheights.com | haodheights-com.mail.protection.outlook.com | |
| iandaconsulting.com.ng | M365 | |
| isaudit.com.ng | M365 | |
| marketinginsights.com.ng | M365 | |
| missienterprises.com | M365 | |
| myapps.com.ng | M365 | |
| onlineradio.com.ng | M365 | |
| openpay.com.ng | M365 | |
| openpaygateway.com | M365 | |
| openshop.com.ng | M365 | |
| opensms.com.ng | M365 | |
| sethhse.com | M365 | |
| t-hub.com.ng | M365 | |
| bill.i.ng | M365 | |

### Category C: External Domains (not in Cloudflare)
| Domain | Notes |
|--------|-------|
| ice-consulting.com | Zone not in this Cloudflare account |
| ice-financials.com | Zone not in this Cloudflare account |

---

## Inbound Email Path (Port 25)

### DNS
- **`mail.thelightville.net`** → A record → `142.180.236.143` (**DNS-only, NOT proxied**)
- All Category A domains have MX pointing to `mail.thelightville.net`
- Cloudflare CANNOT proxy SMTP traffic — mail subdomains must be DNS-only (grey cloud)

### Network Path
```
Internet → 142.180.236.143:25
         → Firewall DNAT → VIP 172.16.16.10:25
         → nginx stream.conf (listen 25) → proxy_pass 127.0.0.1:2525
         → CT201 Postfix (172.16.16.201:25)
         → Dovecot LMTP (172.22.1.250:24)
         → Mailbox
```

### Per-Node Configuration

| Node | Port 25 → localhost:2525 via | Config |
|------|------|--------|
| **PVE1** (172.16.16.20) | SSH tunnel (`ssh -L 2525:127.0.0.1:25 root@172.16.16.201`) | Active VIP holder |
| **PVE2** (172.16.16.22) | Local Postfix relay (`relayhost = [172.16.16.201]:25`) | HA backup |
| **PVE3** (172.16.16.23) | Local Postfix relay (`relayhost = [172.16.16.201]:25`) | HA backup |

All three nodes have identical `stream.conf` with `listen 25; proxy_pass 127.0.0.1:2525;`

### Additional Email Ports (all in stream.conf, all SSH-tunneled from PVE1)
| Port | Service | PVE1 Tunnel |
|------|---------|-------------|
| 25 | SMTP | 2525 → CT201:25 |
| 587 | Submission (STARTTLS) | 2587 → CT201:587 |
| 465 | SMTPS | 2465 → CT201:465 |
| 143 | IMAP | Via stream.conf (check PVE1) |
| 993 | IMAPS | Via stream.conf |
| 110 | POP3 | Via stream.conf |
| 995 | POP3S | Via stream.conf |

---

## Outbound Email Path (SMTP2GO Relay)

### Configuration
- All outbound mail routes through **SMTP2GO** (`[mail.smtp2go.com]:2525`)
- Each domain has a dedicated SMTP2GO sender user (e.g., `smtp-cordeiservices`, `smtp-thelightville-com`)
- Configured via Mailcow's `sender_dependent_relayhost_maps` in the database
- SMTP authentication password for all: `Thelightville123.`

### Verification
- SMTP2GO API Key: stored in SMTP2GO dashboard
- All 34 domains have SPF, DKIM, and DMARC records configured in Cloudflare

---

## CT201 Mailcow Details

### Server
- **IP:** 172.16.16.201
- **Hostname:** webmail.thelightville.net
- **Installation:** `/opt/mailcow-dockerized/`
- **Docker network:** `br-mailcow` (172.22.1.0/24, gateway 172.22.1.1)
- **Portainer:** Not managed via Portainer — uses native docker compose

### Key Containers
| Container | IP | Role |
|-----------|-----|------|
| postfix-mailcow | 172.22.1.253 | SMTP (Postfix) |
| dovecot-mailcow | 172.22.1.250 | IMAP/POP3/LMTP (Dovecot) |
| rspamd-mailcow | 172.22.1.4 | Spam filtering |
| nginx-mailcow | 172.22.1.3 | Webmail proxy |
| mysql-mailcow | 172.22.1.12 | Database |
| redis-mailcow | 172.22.1.249 | Cache |
| unbound-mailcow | 172.22.1.254 | DNS resolver |

### Key Files on CT201
| File | Path | Purpose |
|------|------|---------|
| Postfix overrides | `/opt/mailcow-dockerized/data/conf/postfix/extra.cf` | Custom Postfix settings |
| Custom transport | `/opt/mailcow-dockerized/data/conf/postfix/custom_transport` | Transport overrides (**MUST be empty** unless explicitly needed) |
| Rspamd actions | `/opt/mailcow-dockerized/data/conf/rspamd/local.d/actions.conf` | Spam thresholds |

### Database Access
```bash
ssh root@172.16.16.201
docker exec -it $(docker ps -qf name=mysql-mailcow) mysql -umailcow -p'dwgQ8yCOx3v34rJIiOQ1q9OhN2PB' mailcow
```

---

## Postfix Hardening (Applied March 2026)

The following overrides are in `extra.cf`:
```
myhostname = webmail.thelightville.net
mynetworks = 127.0.0.0/8 172.22.1.0/24 [::1]/128
smtpd_sender_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unknown_sender_domain
smtpd_helo_required = yes
disable_vrfy_command = yes
```

**Critical:** `172.16.16.0/24` was **removed from mynetworks** to prevent unauthenticated relay from the LAN.

### Rspamd Thresholds
```
reject = 12;
add_header = 6;
greylist = 4;
```

---

## DNS Records Checklist

For every Mailcow-primary domain, verify these records exist in Cloudflare:

| Record | Type | Value | Purpose |
|--------|------|-------|---------|
| `@` | MX | `10 mail.thelightville.net` | Inbound mail routing |
| `@` | TXT (SPF) | `v=spf1 include:spf.smtp2go.com ~all` | Authorize SMTP2GO |
| `s2go._domainkey` | CNAME | `*.dkim.smtp2go.net` | DKIM signing |
| `_dmarc` | TXT | `v=DMARC1; p=quarantine; ...` | DMARC policy |
| `mail` (on thelightville.net only) | A | `142.180.236.143` | Mail server hostname (DNS-only!) |

### Critical DNS Rules
1. **`mail.thelightville.net` MUST be DNS-only** (grey cloud) — Cloudflare cannot proxy SMTP
2. **Do NOT add `_dc-mx` MX records** — these are Cloudflare Email Routing and conflict with Mailcow
3. **Do NOT change M365 domain MX records** — they intentionally use Microsoft 365

---

## iptables Persistence

### PVE1 (`/etc/iptables/rules.v4`)
```
-A PREROUTING -i vmbr1 -p tcp -m tcp --dport 25 -j DNAT --to-destination 172.16.16.201:25
```
This rule ensures external SMTP goes to CT201 even if it bypasses nginx stream.

### PVE2 and PVE3
No iptables DNAT for port 25 — they rely on `stream.conf` → local Postfix relay → CT201.

---

## Troubleshooting

### Email not being received
1. **Check MX:** `dig +short MX <domain> @1.1.1.1` — must show `mail.thelightville.net`
2. **Check DNS:** `dig +short A mail.thelightville.net` — must show `142.180.236.143` (NOT a Cloudflare IP)
3. **Check SMTP:** `echo "EHLO test" | nc -w5 172.16.16.201 25` — must show `220 SMTP ESMTP ready`
4. **Check postfix logs:** `ssh root@172.16.16.201 "docker logs --tail 50 $(docker ps -qf name=postfix-mailcow) 2>&1 | grep status="`
5. **Check queue:** `ssh root@172.16.16.201 "docker exec $(docker ps -qf name=postfix-mailcow) mailq"`

### Email not being sent (Spamhaus/blacklist)
1. Outbound MUST go through SMTP2GO — check `sender_dependent_relayhost_maps` in Mailcow DB
2. `custom_transport` file MUST be empty — if it has entries, mail bypasses SMTP2GO
3. Rebuild postmap after clearing: `docker exec <postfix> sh -c "postmap /opt/postfix/conf/custom_transport"`
4. Check IP reputation: https://mxtoolbox.com/blacklists.aspx

### VIP failover — email stops working
1. Verify VIP holder: `ip addr show | grep 172.16.16.10`
2. On new VIP holder, check port 2525: `ss -tlnp | grep 2525`
3. PVE1: SSH tunnel must be running (auto-reconnect via systemd or supervisord)
4. PVE2/PVE3: Local Postfix with `relayhost = [172.16.16.201]:25`

### Adding a new Mailcow domain
1. Add domain in Mailcow UI: `https://webmail.thelightville.net`
2. Create mailboxes
3. Add SMTP2GO sender user for the domain
4. In Cloudflare: create MX, SPF, DKIM, DMARC records (see DNS checklist above)
5. Configure sender-dependent relayhost in Mailcow DB

---

## Security Incident — March 2026

### What happened
- **110,000+ spam messages** found in the deferred queue
- Server IP (142.180.236.143) blacklisted on Spamhaus
- Root cause: `172.16.16.0/24` was in Postfix `mynetworks`, allowing unauthenticated relay from the LAN

### What was fixed
1. Removed LAN from `mynetworks` (only Docker bridge + localhost now)
2. Added sender restrictions and HELO requirements
3. Purged 107,904 deferred messages
4. Lowered Rspamd spam thresholds
5. Disabled catch-all relay on `thelightville.ng`
6. Fixed SPF records across all 44 Cloudflare zones
7. Verified DKIM for 33/34 SMTP2GO domains
8. Created missing DMARC records for 10 domains
9. Cleared stale `custom_transport` entry that was bypassing SMTP2GO for thelightville.com
10. Fixed `mail.thelightville.net` DNS (was proxied Cloudflare Tunnel CNAME, now DNS-only A record)
11. Changed port 25 DNAT from CT101 (cPanel) to CT201 (Mailcow)
12. Replaced Cloudflare Email Routing MX records (`_dc-mx`) with `mail.thelightville.net` for 16 domains
13. Configured PVE2/PVE3 Postfix to relay to CT201 for HA failover

### Prevention
- NEVER add LAN subnets to Postfix `mynetworks`
- NEVER add entries to `custom_transport` unless specifically needed (and clear the `.db` file too)
- NEVER proxy `mail.thelightville.net` through Cloudflare (must be DNS-only / grey cloud)
- Monitor queue regularly: `docker exec <postfix> mailq | tail -1`
- Review Rspamd stats in Mailcow UI

---

## Webmail Access
- **URL:** https://webmail.thelightville.net (proxied through Cloudflare Tunnel)
- **Admin:** Mailcow admin panel at same URL, `/admin` path
- **SOGo:** Built-in webmail client

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-03-06 | Complete email infrastructure rebuild — fixed inbound/outbound, DNS, iptables, HA | AI Agent |
| 2025-11 | Original cPanel/Exim architecture (now replaced by Mailcow) | — |
