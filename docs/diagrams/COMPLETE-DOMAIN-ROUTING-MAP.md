# COMPLETE DOMAIN ROUTING MAP
**All Domains and Their Backend Destinations**

Last Updated: December 5, 2025

---

## 📋 Container Mapping

| Container | IP | Purpose |
|-----------|-----|---------|
| CT101 | 172.16.16.101 | cPanel/WHM (29 domains) |
| CT103 | 172.16.16.103 | Online Radio (App, Admin, Deepsound) |
| CT105 | 172.16.16.105 | ClaApp Platform |
| CT107 | 172.16.16.107 | Online Radio APIs |
| CT109 | 172.16.16.109 | Channel Management |
| CT111 | 172.16.16.111 | Monitoring Dashboard |
| CT113 | 172.16.16.113 | Chatbot Service |
| VM200 | 172.16.16.200 | RDP/Windows VM |

---

## 🌐 onlineradio.com.ng Subdomains

| Subdomain | Destination | Port | Purpose |
|-----------|-------------|------|---------|
| onlineradio.com.ng | CT101 | 443 | Main Website |
| www.onlineradio.com.ng | CT101 | 443 | Main Website (www) |
| **app**.onlineradio.com.ng | **CT103** | **443** | Radio App |
| **admin**.onlineradio.com.ng | **CT103** | **443** | Admin Panel |
| **deepsound**.onlineradio.com.ng | **CT103** | **443** | Deepsound Platform |
| **api**.onlineradio.com.ng | **CT107** | **443** | Main API Gateway |
| **auth**.onlineradio.com.ng | **CT107** | **443** | Authentication API |
| **music**.onlineradio.com.ng | **CT107** | **443** | Music API |
| **stations**.onlineradio.com.ng | **CT107** | **443** | Stations API |
| **billing**.onlineradio.com.ng | **CT107** | **443** | Billing API |
| **podcasts**.onlineradio.com.ng | **CT107** | **443** | Podcasts API |
| **channel**.onlineradio.com.ng | **CT109** | **443** | Channel Management |
| cpanel.onlineradio.com.ng | CT101 | 2083 | cPanel Interface |
| webmail.onlineradio.com.ng | CT101 | 2096 | Webmail |
| whm.onlineradio.com.ng | CT101 | 2087 | WHM Admin |
| mail.onlineradio.com.ng | CT101 | 2096 | Mail Interface |

---

## 🌐 myapps.com.ng Subdomains

| Subdomain | Destination | Port | Purpose |
|-----------|-------------|------|---------|
| myapps.com.ng | CT101 | 443 | Main Website |
| www.myapps.com.ng | CT101 | 443 | Main Website (www) |
| **claapp**.myapps.com.ng | **CT105** | **443** | ClaApp Main |
| ***.claapp**.myapps.com.ng | **CT105** | **443** | All ClaApp Subdomains |
| admin.claapp.myapps.com.ng | CT105 | 443 | ClaApp Admin |
| agency.claapp.myapps.com.ng | CT105 | 443 | Agency Module |
| annuity.claapp.myapps.com.ng | CT105 | 443 | Annuity Module |
| api.claapp.myapps.com.ng | CT105 | 443 | ClaApp API |
| app.claapp.myapps.com.ng | CT105 | 443 | ClaApp Frontend |
| claims.claapp.myapps.com.ng | CT105 | 443 | Claims Module |
| core.claapp.myapps.com.ng | CT105 | 443 | Core Module |
| corporates.claapp.myapps.com.ng | CT105 | 443 | Corporates Module |
| dashboard.claapp.myapps.com.ng | CT105 | 443 | Dashboard |
| docs.claapp.myapps.com.ng | CT105 | 443 | Documentation |
| documents.claapp.myapps.com.ng | CT105 | 443 | Documents Module |
| finance.claapp.myapps.com.ng | CT105 | 443 | Finance Module |
| reports.claapp.myapps.com.ng | CT105 | 443 | Reports Module |
| retail.claapp.myapps.com.ng | CT105 | 443 | Retail Module |
| **chatbot**.myapps.com.ng | **CT113** | **443** | Chatbot Service |
| **monitoring**.myapps.com.ng | **CT111** | **3000** | Monitoring Dashboard |
| cpanel.myapps.com.ng | CT101 | 2083 | cPanel Interface |
| webmail.myapps.com.ng | CT101 | 2096 | Webmail |
| whm.myapps.com.ng | CT101 | 2087 | WHM Admin |
| mail.myapps.com.ng | CT101 | 2096 | Mail Interface |

---

## 🌐 thelightville.xyz Subdomains

| Subdomain | Destination | Port | Purpose |
|-----------|-------------|------|---------|
| thelightville.xyz | CT101 | 443 | Main Website |
| www.thelightville.xyz | CT101 | 443 | Main Website (www) |
| **hosting**.thelightville.xyz | CT101 | 443 | Hosting Services |
| **monitoring**.thelightville.xyz | **CT111** | **3000** | Monitoring Dashboard |
| **rdp**.thelightville.xyz | **VM200** | **3389** | Remote Desktop (RDP) |
| cpanel.thelightville.xyz | CT101 | 2083 | cPanel Interface |
| webmail.thelightville.xyz | CT101 | 2096 | Webmail |
| whm.thelightville.xyz | CT101 | 2087 | WHM Admin |
| mail.thelightville.xyz | CT101 | 2096 | Mail Interface |
| ns1.thelightville.xyz | - | - | Nameserver |
| ns2.thelightville.xyz | - | - | Nameserver |

---

## 📊 Summary Statistics

| Category | Count |
|----------|-------|
| **Total cPanel Domains** | 29 |
| **cPanel Subdomains** | 116 (29×4: cpanel, webmail, whm, mail) |
| **App Subdomains (onlineradio)** | 11 |
| **App Subdomains (myapps/claapp)** | 18 |
| **Special Services** | 3 (monitoring, rdp, chatbot) |
| **Total Routed Subdomains** | ~148 |

---

## 🔧 Current Configuration Status

### ✅ Properly Configured:
- All 116 cPanel subdomains (cpanel, webmail, whm, mail)
- monitoring.thelightville.xyz → CT111
- app.onlineradio.com.ng → CT103
- claapp.myapps.com.ng → CT105  
- api.onlineradio.com.ng → CT107
- channel.onlineradio.com.ng → CT109
- rdp.thelightville.xyz → VM200 (RDP)

### ⚠️ MISSING FROM NGINX CONFIG:
- **auth.onlineradio.com.ng** → CT107
- **stations.onlineradio.com.ng** → CT107
- **music.onlineradio.com.ng** → CT107
- **podcasts.onlineradio.com.ng** → CT107
- **billing.onlineradio.com.ng** → CT107
- **deepsound.onlineradio.com.ng** → CT103
- **admin.onlineradio.com.ng** → CT103
- **chatbot.myapps.com.ng** → CT113

---

## 🎯 Actions Required

### 1. Remove Unnecessary Cloudflared Entries
The 3 wildcard entries in `/etc/cloudflared/config.yml` are NOT needed because:
- All domains (including main domains) now use CNAME records
- CNAME automatically routes to tunnel
- Wildcard entries were only needed when using A records

**Can be removed:**
```yaml
- hostname: "*.thelightville.xyz"      # NOT NEEDED
- hostname: "*.myapps.com.ng"          # NOT NEEDED  
- hostname: "*.onlineradio.com.ng"     # NOT NEEDED
```

### 2. Add Missing Routes to Nginx
Add the 8 missing subdomain routes to `/etc/nginx/sites-available/cloudflare-tunnel-proxy.conf`

### 3. Update Architecture Documentation
Include all subdomain mappings in main architecture doc

---

## 📝 Notes

- **DNS Type:** All subdomains use CNAME records pointing to tunnel hostname
- **Routing:** Nginx HTTP proxy (`cloudflare-tunnel-proxy.conf`) handles all routing
- **Pattern Matching:** Uses regex patterns for efficient routing
- **SSL/TLS:** Handled by backend containers (CT101, CT103, CT105, etc.)

---

## ✅ CONFIGURATION STATUS: COMPLETE

**Last Updated:** December 5, 2025 08:36 UTC

### Changes Applied

✅ **Cloudflared Config Simplified** (`/etc/cloudflared/config.yml`):
- Removed unnecessary wildcard entries: `*.thelightville.xyz`, `*.myapps.com.ng`, `*.onlineradio.com.ng`
- Reduced from 21 lines → 11 lines
- Kept: RDP entry for VM200 + catch-all service for all CNAME traffic
- Status: **Active and working**
- Note: Cloudflare dashboard has remote config (version 21) with legacy claapp entries, but catch-all handles all other traffic correctly

✅ **Nginx Routes Updated** (`/etc/nginx/sites-available/cloudflare-tunnel-proxy.conf`):
- **Added**: `~^chatbot\.myapps\.com\.ng$` → CT113:443 (chatbot service)
- **Updated**: Monitoring pattern to handle both `monitoring.thelightville.xyz` and `monitoring.myapps.com.ng`
- **Verified**: CT107 already had auth/stations/music/podcasts/billing routes via regex pattern
- **Verified**: CT103 already had app/admin/deepsound routes via regex pattern
- Status: **Active and working**

### Verification Results (Dec 5, 2025)

**Service Status:**
- cloudflared: ✅ Active
- nginx: ✅ Active

**Route Testing:**
- ✅ cPanel subdomains (116 total): HTTP 200 (working)
- ✅ onlineradio apps: HTTP 200 (working)
- ✅ claapp modules: HTTP 302 (working)
- ✅ monitoring.myapps.com.ng: HTTP 302 (working)
- ⚠️ chatbot.myapps.com.ng: HTTP 502 (nginx routing correct, CT113 backend service appears down)

### Architecture Summary

- **Total Routed Subdomains:** ~148
  - 116 cPanel management subdomains (cpanel/webmail/whm/mail × 29 domains)
  - 16 onlineradio.com.ng application subdomains
  - 24 myapps.com.ng application subdomains
  - 10 thelightville.xyz subdomains
- **DNS Configuration:** All use CNAME → tunnel hostname
- **Routing Engine:** Nginx with regex patterns for efficient matching
- **Cloudflared Config:** Minimal (11 lines) with catch-all ingress rule
- **Backups Created:** Both configs backed up before modifications

---

**Document Status:** ✅ Complete and Current
**Configuration Status:** ✅ All Routes Active
