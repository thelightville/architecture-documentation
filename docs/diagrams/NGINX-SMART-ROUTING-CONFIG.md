# Nginx Reverse Proxy - Final Configuration
## Completed: November 16, 2025

## Architecture Overview

**Smart Default Routing Strategy:**
- Only **explicitly map** the specific container domains (CT103, CT105, CT107, CT109)
- **All other domains** automatically route to CT101 (cPanel)
- **No manual maintenance** required when adding new domains to cPanel

## Configuration

### HTTP (Port 80) - Layer 7 Proxy

**Explicit Routes** (`/etc/nginx/sites-available/`):
- `onlineradio-ct103.conf`: app, admin, deepsound.onlineradio.com.ng → 172.16.16.103
- `onlineradio-ct107.conf`: api, auth, billing, music, podcasts, stations.onlineradio.com.ng → 172.16.16.107
- `onlineradio-ct109.conf`: channel.onlineradio.com.ng → 172.16.16.109
- `claapp-ct105.conf`: claapp.myapps.com.ng + 13 subdomains → 172.16.16.105

**Default Route** (`/etc/nginx/sites-available/default-ct101.conf`):
- Catches ALL other domains → 172.16.16.101 (cPanel)
- Includes all 30+ cPanel domains automatically
- Any new domain added to cPanel works immediately

### HTTPS (Port 443) - Layer 4 SSL Passthrough

**Configuration**: `/etc/nginx/stream.conf`
- Uses nginx `stream` module for raw TCP proxying
- `ssl_preread on` reads SNI (Server Name Indication) from TLS ClientHello
- Routes based on domain name WITHOUT decrypting SSL
- Backend containers handle SSL termination with their own certificates

**Explicit Routes** (map in stream.conf):
```
app.onlineradio.com.ng              → 172.16.16.103:443
admin.onlineradio.com.ng            → 172.16.16.103:443
deepsound.onlineradio.com.ng        → 172.16.16.103:443
api.onlineradio.com.ng              → 172.16.16.107:443
auth.onlineradio.com.ng             → 172.16.16.107:443
billing.onlineradio.com.ng          → 172.16.16.107:443
music.onlineradio.com.ng            → 172.16.16.107:443
podcasts.onlineradio.com.ng         → 172.16.16.107:443
stations.onlineradio.com.ng         → 172.16.16.107:443
channel.onlineradio.com.ng          → 172.16.16.109:443
claapp.myapps.com.ng                → 172.16.16.105:443
chatbot.claapp.myapps.com.ng        → 172.16.16.105:443
admin.claapp.myapps.com.ng          → 172.16.16.105:443
... (14 claapp subdomains total)
```

**Default Route**:
```
default → 172.16.16.101:443
```

## Container Breakdown

### CT101 - cPanel Hosting (172.16.16.101)
**Role**: Default backend for all unmatched domains
**Domains** (30+):
- thelightville.xyz, thelightville.com, thelightville.net
- myapps.com.ng
- onlineradio.com.ng (main domain, different from subdomains on CT103/107/109)
- openpay.com.ng, openpaygateway.com, opensms.com.ng, openshop.com.ng
- bajantropicaltreats.com, electricmall.com.ng, emergeglobalservices.com
- fastrackenergy.com, mother2mother.com.ng, isaudit.com.ng
- 123accountingsolutions.com.ng, marketinginsights.com.ng
- bill.i.ng, chatbot.com.ng, t-hub.com.ng, tnfc.com.ng
- And any new domains added via cPanel

**SSL**: Let's Encrypt certificates managed by cPanel
**Ports**: 2082, 2083, 2086, 2087, 2095, 2096 (cPanel/WHM) still forwarded directly via iptables

### CT103 - OnlineRadio Web Apps (172.16.16.103)
- app.onlineradio.com.ng
- admin.onlineradio.com.ng
- deepsound.onlineradio.com.ng

### CT105 - CLA Insurance Platform (172.16.16.105)
All claapp.myapps.com.ng subdomains (14 total):
- claapp, chatbot, admin, agency, annuity, api
- claims, core, corporates, docs, documents
- finance, reports, retail

### CT107 - OnlineRadio API Services (172.16.16.107)
- api.onlineradio.com.ng
- auth.onlineradio.com.ng
- billing.onlineradio.com.ng
- music.onlineradio.com.ng
- podcasts.onlineradio.com.ng
- stations.onlineradio.com.ng

### CT109 - OnlineRadio Channel (172.16.16.109)
- channel.onlineradio.com.ng

## Network Flow

### Incoming HTTPS Request:
```
Internet → 142.180.236.143:443
  ↓
Proxmox vmbr1 (192.168.2.50:443)
  ↓
Nginx stream module (reads SNI)
  ↓
If domain in map → route to specific container:443
If not in map → route to 172.16.16.101:443 (CT101)
  ↓
Backend container handles SSL with its own certificate
```

### Incoming HTTP Request:
```
Internet → 142.180.236.143:80
  ↓
Proxmox vmbr1 (192.168.2.50:80)
  ↓
Nginx http module (reads Host header)
  ↓
If server_name matches → route to specific container:80
If no match → default_server routes to 172.16.16.101:80 (CT101)
  ↓
Backend container responds (usually redirects to HTTPS)
```

## Benefits of This Approach

1. **Zero Maintenance for CT101**:
   - Add domain in cPanel → It works immediately
   - No nginx config changes needed
   - No list to maintain

2. **SSL Passthrough**:
   - Each container manages its own SSL certificates
   - No certificate management on Proxmox
   - No SSL termination overhead
   - No SNI conflicts

3. **Explicit Container Routes**:
   - Only map what needs special handling (CT103, 105, 107, 109)
   - Clear separation of services
   - Easy to add new containers

4. **Performance**:
   - Stream module is extremely fast (Layer 4)
   - No SSL overhead on Proxmox
   - Direct TCP passthrough for HTTPS

5. **Flexibility**:
   - Easy to move domains between containers
   - Just update one map entry
   - No iptables changes needed

## Testing Verification

**HTTP Tests**:
```bash
curl -I http://thelightville.xyz         # → CT101 (default)
curl -I http://app.onlineradio.com.ng    # → CT103 (explicit)
curl -I http://claapp.myapps.com.ng      # → CT105 (explicit)
```

**HTTPS Tests**:
```bash
curl -kI https://thelightville.xyz       # → CT101 (default)
curl -kI https://app.onlineradio.com.ng  # → CT103 (explicit)
curl -kI https://claapp.myapps.com.ng    # → CT105 (explicit)
```

**SSL Certificate Verification**:
```bash
openssl s_client -connect 142.180.236.143:443 -servername thelightville.xyz
# Shows: Let's Encrypt certificate from CT101
```

## Maintenance

### Add New Container:
1. Create server block in `/etc/nginx/sites-available/`
2. Enable: `ln -s ../sites-available/newcontainer.conf /etc/nginx/sites-enabled/`
3. Add HTTPS mapping to `/etc/nginx/stream.conf`
4. Test: `nginx -t`
5. Reload: `systemctl reload nginx`

### Move Domain from CT101 to Another Container:
1. Add domain to target container's config
2. Add HTTPS entry in stream.conf map
3. Reload nginx
4. Remove domain from cPanel if desired

### View Active Configuration:
```bash
ls /etc/nginx/sites-enabled/
cat /etc/nginx/stream.conf
nginx -T  # Show full parsed config
```

### Monitor Traffic:
```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

## Status: ✅ PRODUCTION READY

- HTTP routing: Working ✓
- HTTPS passthrough: Working ✓
- SSL certificates: Managed by backends ✓
- Default routing to CT101: Working ✓
- All 40+ domains configured: ✓
- Zero manual maintenance for new cPanel domains: ✓
