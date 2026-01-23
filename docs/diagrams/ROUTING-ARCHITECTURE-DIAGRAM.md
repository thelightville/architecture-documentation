# COMPLETE ROUTING ARCHITECTURE
**Thelightville Infrastructure - December 2025**

---

## 📊 Visual Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT BROWSER                               │
│                    (https://cpanel.bill.i.ng)                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│                       CLOUDFLARE CDN                                 │
│  • DNS Resolution: CNAME → tunnel hostname                          │
│  • DDoS Protection, WAF, Bot Management                             │
│  • SSL/TLS Termination                                              │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS (encrypted tunnel)
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    CLOUDFLARE TUNNEL (cloudflared)                   │
│  Location: Proxmox Host (PID 341164)                                │
│  Config: /etc/cloudflared/config.yml (21 lines)                     │
│  • *.thelightville.xyz → localhost:8080                             │
│  • *.myapps.com.ng → localhost:8080                                 │
│  • *.onlineradio.com.ng → localhost:8080                            │
│  • ALL other domains route via CNAME (automatic)                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTP to localhost:8080
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    NGINX HTTP PROXY (Port 8080)                      │
│  Config: /etc/nginx/sites-available/cloudflare-tunnel-proxy.conf    │
│                                                                       │
│  Routing Logic (Host header inspection):                             │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ ~^cpanel\..*$     → 172.16.16.101:2083 (CT101 cPanel)        │  │
│  │ ~^webmail\..*$    → 172.16.16.101:2096 (CT101 Webmail)       │  │
│  │ ~^whm\..*$        → 172.16.16.101:2087 (CT101 WHM)           │  │
│  │ ~^mail\..*$       → 172.16.16.101:2096 (CT101 Mail)          │  │
│  │                                                                │  │
│  │ monitoring.*      → 172.16.16.111:3000 (CT111 Monitoring)     │  │
│  │ app.onlineradio.* → 172.16.16.103:443  (CT103 Radio App)     │  │
│  │ claapp.*          → 172.16.16.105:443  (CT105 ClaApp)        │  │
│  │ api.onlineradio.* → 172.16.16.107:443  (CT107 APIs)          │  │
│  │ channel.*         → 172.16.16.109:443  (CT109 Channel)       │  │
│  │                                                                │  │
│  │ default           → 172.16.16.101:443  (CT101 cPanel/Sites)  │  │
│  └───────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS to backend
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      BACKEND CONTAINERS (LXC)                        │
│                                                                       │
│  CT101 (172.16.16.101) - cPanel/WHM                                 │
│    • Port 443:  Apache (main sites + cPanel redirects)              │
│    • Port 2083: cPanel User Interface                               │
│    • Port 2087: WHM Admin Interface                                 │
│    • Port 2096: Webmail Interface                                   │
│    • 29 domains hosted                                              │
│                                                                       │
│  CT103 (172.16.16.103) - Online Radio Apps                          │
│  CT105 (172.16.16.105) - ClaApp                                     │
│  CT107 (172.16.16.107) - Online Radio APIs                          │
│  CT109 (172.16.16.109) - Channel Management                         │
│  CT111 (172.16.16.111) - Monitoring Dashboard                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Configuration Files

### 1. DNS Records (Cloudflare Dashboard)
**Purpose:** Route all domains to Cloudflare Tunnel
**Type:** CNAME records
**Example:**
```
cpanel.bill.i.ng        → a0bd27a8-26c6-498a-bb7f-a88284a532ff.cfargotunnel.com
webmail.myapps.com.ng   → a0bd27a8-26c6-498a-bb7f-a88284a532ff.cfargotunnel.com
whm.thelightville.xyz   → a0bd27a8-26c6-498a-bb7f-a88284a532ff.cfargotunnel.com
```
**Total:** 116 CNAME records (29 domains × 4 subdomains: cpanel, webmail, whm, mail)

---

### 2. Cloudflare Tunnel Config
**File:** `/etc/cloudflared/config.yml`
**Purpose:** Accept incoming tunnel connections and forward to nginx
**Lines:** 21 (cleaned up from 70)

```yaml
tunnel: a0bd27a8-26c6-498a-bb7f-a88284a532ff
credentials-file: /etc/cloudflared/a0bd27a8-26c6-498a-bb7f-a88284a532ff.json

ingress:
  - hostname: rdp.thelightville.xyz
    service: rdp://172.16.16.200:3389
  
  - hostname: "*.thelightville.xyz"
    service: http://localhost:8080
  
  - hostname: "*.myapps.com.ng"
    service: http://localhost:8080
  
  - hostname: "*.onlineradio.com.ng"
    service: http://localhost:8080
  
  - service: http_status:404
```

**Why only 3 domains?** 
- Other 26 domains route automatically via CNAME records
- No need to list them explicitly
- Cleaner, more maintainable config

---

### 3. Nginx HTTP Proxy
**File:** `/etc/nginx/sites-available/cloudflare-tunnel-proxy.conf`
**Purpose:** Route HTTP requests to correct backend container/port
**Status:** ✅ ESSENTIAL - DO NOT REMOVE

This is the **main routing brain** that directs traffic based on hostname patterns.

---

### 4. Nginx Stream Module (Optional)
**File:** `/etc/nginx/stream.conf`
**Purpose:** Direct HTTPS routing (TLS passthrough) without tunnel
**Status:** ⚠️ NOT CURRENTLY USED (but harmless)

This provides SNI-based routing for direct port 443 access, bypassing the tunnel.
Currently inactive because all traffic goes through Cloudflare Tunnel on port 8080.

**When would this be used?**
- If tunnel goes down and you need direct access
- If you expose port 443 directly without Cloudflare
- Legacy fallback mechanism

**Recommendation:** Keep it, doesn't hurt and provides redundancy option.

---

## 🔄 Traffic Flow Examples

### Example 1: Client accesses cpanel.bill.i.ng

```
1. Browser → https://cpanel.bill.i.ng
2. DNS lookup → CNAME: a0bd27a8-26c6-498a-bb7f-a88284a532ff.cfargotunnel.com
3. Cloudflare CDN → Tunnel connection
4. cloudflared → localhost:8080
5. Nginx reads Host: cpanel.bill.i.ng
6. Pattern match: ~^cpanel\..*$ 
7. Proxy to: https://172.16.16.101:2083
8. CT101 cPanel responds with login page
9. Response travels back same path
10. Browser displays cPanel login
```

### Example 2: Client accesses main website bill.i.ng

```
1. Browser → https://bill.i.ng
2. DNS lookup → CNAME: tunnel hostname (if exists) OR A record
3. Cloudflare CDN → Tunnel
4. cloudflared → localhost:8080
5. Nginx default route → https://172.16.16.101:443
6. CT101 Apache serves website
7. Response back to browser
```

---

## 📝 Configuration Summary

| Component | File | Lines | Purpose | Status |
|-----------|------|-------|---------|--------|
| DNS | Cloudflare Dashboard | 116 records | CNAME to tunnel | ✅ Required |
| Tunnel | /etc/cloudflared/config.yml | 21 | Tunnel ingress | ✅ Required |
| HTTP Proxy | /etc/nginx/.../cloudflare-tunnel-proxy.conf | ~60 | Route to backends | ✅ Required |
| Stream | /etc/nginx/stream.conf | ~200 | Direct HTTPS routing | ⚠️ Optional |
| Disabled | /etc/nginx/.../whm-ssl.conf.disabled | N/A | Old config | ❌ Keep disabled |

---

## 🧹 Cleanup Completed

### Before:
- cloudflared config: 70 lines with 30 explicit hostname entries
- Bloated, difficult to maintain
- Redundant with CNAME DNS routing

### After:
- cloudflared config: 21 lines with 3 wildcard entries
- Clean, maintainable
- Relies on CNAME records for automatic routing

### Backups:
- `/etc/cloudflared/config.yml.bloated-backup` - Previous bloated version
- `/etc/cloudflared/config.yml.backup-*` - Earlier backups
- `/root/nginx-backup-20251205/` - All nginx configs

---

## 🎯 Key Principles

1. **DNS CNAME records do the routing** - No need to list every domain in tunnel config
2. **Nginx HTTP proxy is the routing brain** - Inspects Host header and routes accordingly
3. **Stream module is backup/optional** - Not used for tunnel traffic but provides redundancy
4. **Simplicity wins** - Fewer configs = easier maintenance

---

## 🔍 Troubleshooting

**If subdomain doesn't work:**
1. Check DNS: Should be CNAME to tunnel hostname
2. Check tunnel is running: `systemctl status cloudflared`
3. Check nginx is running: `systemctl status nginx`
4. Check nginx routing: `curl -H "Host: subdomain.domain.com" http://localhost:8080`

**If website loads instead of cPanel:**
- This is an Apache vhost issue on CT101, not routing issue
- Use subdomain access (cpanel.domain.com) instead of path-based (/cpanel)

---

**Last Updated:** December 5, 2025
**Configuration Status:** ✅ Cleaned and Optimized
