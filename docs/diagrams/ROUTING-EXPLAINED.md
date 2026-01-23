# Traffic Routing Explanation

## Your Concern

> "Cloudflare routes to CT101, but domains for CT103/105/107/109 won't get to their containers"

## Answer: They WILL Get There! ✅

The **nginx reverse proxy on CT101** routes traffic to ALL containers based on hostname!

---

## How It Works (Step by Step)

### Example 1: Request to app.onlineradio.com.ng (CT103)

```
1. User types: https://app.onlineradio.com.ng
   ↓
2. DNS lookup returns: Cloudflare IPs (104.x.x.x)
   ↓
3. Browser connects to: Cloudflare Edge
   ↓
4. Cloudflare Edge applies: DDoS protection, WAF filtering
   ↓
5. Traffic enters: Encrypted Tunnel (4 QUIC connections)
   ↓
6. Tunnel delivers to: CT101 port 8888 (nginx)
   ↓
7. nginx reads HTTP header: Host: app.onlineradio.com.ng
   ↓
8. nginx matches server block:
   server {
       server_name app.onlineradio.com.ng;
       location / {
           proxy_pass https://172.16.16.103:443;  ← Routes to CT103!
       }
   }
   ↓
9. nginx forwards request to: CT103:443
   ↓
10. CT103 (Laravel nginx) processes request
    ↓
11. CT103 sends response back to: CT101 nginx
    ↓
12. CT101 nginx forwards response to: Cloudflare
    ↓
13. Cloudflare delivers to: User browser
    ✅ User sees website from CT103!
```

### Example 2: Request to claapp.myapps.com.ng (CT105)

```
1. User types: https://claapp.myapps.com.ng
   ↓
2-6. [Same as above - DNS → Cloudflare → Tunnel → CT101:8888]
   ↓
7. nginx reads HTTP header: Host: claapp.myapps.com.ng
   ↓
8. nginx matches server block:
   server {
       server_name claapp.myapps.com.ng *.claapp.myapps.com.ng;
       location / {
           proxy_pass https://172.16.16.105:443;  ← Routes to CT105!
       }
   }
   ↓
9. nginx forwards request to: CT105:443
   ↓
10. CT105 (insurance microservices) processes request
    ✅ User sees website from CT105!
```

### Example 3: Request to api.onlineradio.com.ng (CT107)

```
1. User types: https://api.onlineradio.com.ng
   ↓
2-6. [Same - DNS → Cloudflare → Tunnel → CT101:8888]
   ↓
7. nginx reads HTTP header: Host: api.onlineradio.com.ng
   ↓
8. nginx matches server block:
   server {
       server_name api.onlineradio.com.ng auth.onlineradio.com.ng ...;
       location / {
           proxy_pass https://172.16.16.107:443;  ← Routes to CT107!
       }
   }
   ↓
9. nginx forwards request to: CT107:443
    ✅ API endpoints on CT107 respond correctly!
```

### Example 4: Request to thelightville.com (CT101 cPanel)

```
1. User types: https://thelightville.com
   ↓
2-6. [Same - DNS → Cloudflare → Tunnel → CT101:8888]
   ↓
7. nginx reads HTTP header: Host: thelightville.com
   ↓
8. nginx matches: default_server (no specific match)
   server {
       listen 8888 default_server;
       location / {
           proxy_pass http://127.0.0.1:80;  ← Routes to Apache on CT101!
       }
   }
   ↓
9. nginx forwards request to: Apache on CT101:80
    ✅ cPanel website served from Apache!
```

---

## Visual Network Diagram

```
                         INTERNET
                            |
                            |
                    [User's Browser]
                            |
                            | DNS lookup
                            | (returns Cloudflare IPs)
                            ↓
                    ╔═══════════════╗
                    ║   CLOUDFLARE  ║
                    ║  Edge Network ║
                    ║               ║
                    ║ • DDoS Block  ║
                    ║ • WAF Rules   ║
                    ║ • Bot Filter  ║
                    ║ • CDN Cache   ║
                    ╚═══════════════╝
                            |
                            | Encrypted Tunnel
                            | (4 QUIC connections)
                            | Bypasses public internet
                            |
                            ↓
           ╔════════════════════════════════╗
           ║     CT101 (172.16.16.101)     ║
           ║                                ║
           ║  nginx :8888 (REVERSE PROXY)  ║
           ║  /etc/nginx/conf.d/...conf    ║
           ║                                ║
           ║  Reads: HTTP Host header      ║
           ║  Routes: Based on hostname    ║
           ╚════════════════════════════════╝
                            |
        ┌───────────────────┼───────────────────┐
        |                   |                   |
        ↓                   ↓                   ↓
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │ Apache  │        │  CT103  │        │  CT105  │
   │ :80     │        │ :443    │        │ :443    │
   │         │        │         │        │         │
   │ 29 cPanel│       │ Laravel │        │Insurance│
   │ domains  │       │  apps   │        │  apps   │
   └─────────┘        └─────────┘        └─────────┘
  thelightville.com   app.onlineradio   claapp.myapps
                      admin.onlineradio  *.claapp...
                      deepsound...
        
        ↓                   ↓
   ┌─────────┐        ┌─────────┐
   │  CT107  │        │  CT109  │
   │ :443    │        │ :443    │
   │         │        │         │
   │  API    │        │AzuraCast│
   │Gateway  │        │Streaming│
   └─────────┘        └─────────┘
   api.onlineradio    channel...
   auth.onlineradio
   music...
   stations...
   billing...
   podcasts...
```

---

## Comparison: OLD vs NEW Architecture

### OLD Architecture (Direct to Proxmox)

```
Internet → 142.180.236.143 (Public IP - EXPOSED)
    ↓
Proxmox Host (nginx stream)
Reads: SNI hostname (Layer 4 - TLS)
Routes based on: Domain name in TLS handshake
    ↓
    ├─→ thelightville.com:443 → CT101:80
    ├─→ app.onlineradio.com.ng:443 → CT103:443
    ├─→ claapp.myapps.com.ng:443 → CT105:443
    └─→ api.onlineradio.com.ng:443 → CT107:443

Problems:
❌ No DDoS protection
❌ No WAF/bot filtering
❌ Origin IP exposed
❌ No CDN caching
❌ Vulnerable to attacks
```

### NEW Architecture (Cloudflare Tunnel)

```
Internet → Cloudflare Edge (DDoS/WAF/CDN)
    ↓
Encrypted Tunnel (HIDDEN - No public IP exposed)
    ↓
CT101:8888 (nginx reverse proxy)
Reads: HTTP Host header (Layer 7)
Routes based on: Domain name in HTTP request
    ↓
    ├─→ thelightville.com → Apache:80 (CT101)
    ├─→ app.onlineradio.com.ng → CT103:443
    ├─→ claapp.myapps.com.ng → CT105:443
    └─→ api.onlineradio.com.ng → CT107:443

Benefits:
✅ DDoS protection (172 Tbps)
✅ WAF blocking attacks
✅ Origin IP HIDDEN
✅ CDN caching (faster)
✅ Bot management
✅ SSL/TLS encryption
✅ Rate limiting
```

---

## The Key Concept

### It's the SAME routing concept!

**Proxmox nginx stream** (old):
- Layer 4 routing
- Reads hostname from SNI (TLS handshake)
- Routes to container based on hostname

**CT101 nginx** (new):
- Layer 7 routing
- Reads hostname from HTTP Host header
- Routes to container based on hostname

**SAME RESULT:** Traffic reaches the correct container! ✅

The only difference: Layer 4 vs Layer 7 routing
Both do the same job: **Route traffic based on domain name**

---

## nginx Reverse Proxy Configuration

The actual config on CT101 (`/etc/nginx/conf.d/cloudflare-tunnel-proxy.conf`):

```nginx
# Define backend containers
upstream ct103_webapps {
    server 172.16.16.103:443;
}

upstream ct105_insurance {
    server 172.16.16.105:443;
}

upstream ct107_apis {
    server 172.16.16.107:443;
}

upstream ct109_streaming {
    server 172.16.16.109:443;
}

# Route CT103 domains (OnlineRadio Laravel apps)
server {
    listen 8888;
    server_name app.onlineradio.com.ng 
                admin.onlineradio.com.ng 
                deepsound.onlineradio.com.ng;
    
    location / {
        proxy_pass https://ct103_webapps;  # ← Routes to CT103!
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# Route CT105 domains (CLA Insurance)
server {
    listen 8888;
    server_name claapp.myapps.com.ng 
                *.claapp.myapps.com.ng;
    
    location / {
        proxy_pass https://ct105_insurance;  # ← Routes to CT105!
        proxy_ssl_verify off;
        proxy_set_header Host $host;
    }
}

# Route CT107 domains (OnlineRadio APIs)
server {
    listen 8888;
    server_name api.onlineradio.com.ng 
                auth.onlineradio.com.ng 
                music.onlineradio.com.ng 
                stations.onlineradio.com.ng 
                billing.onlineradio.com.ng 
                podcasts.onlineradio.com.ng;
    
    location / {
        proxy_pass https://ct107_apis;  # ← Routes to CT107!
        proxy_ssl_verify off;
        proxy_set_header Host $host;
    }
}

# Route CT109 domains (AzuraCast)
server {
    listen 8888;
    server_name channel.onlineradio.com.ng;
    
    location / {
        proxy_pass https://ct109_streaming;  # ← Routes to CT109!
        proxy_ssl_verify off;
        proxy_set_header Host $host;
    }
}

# Default route (cPanel domains on CT101)
server {
    listen 8888 default_server;
    server_name _;
    
    location / {
        proxy_pass http://127.0.0.1:80;  # ← Routes to Apache on CT101!
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## Why This Works

### 1. Hostname-Based Routing
nginx reads the `Host:` header in every HTTP request and matches it to the correct `server_name`.

### 2. Proxy to Backend
Once matched, nginx forwards (proxies) the request to the correct backend container.

### 3. Response Flow
Backend sends response to nginx → nginx sends to Cloudflare → Cloudflare sends to user.

### 4. Transparent to Containers
Each container doesn't know about Cloudflare or the tunnel. They just receive requests and respond normally.

---

## Testing: Verify Routing Works

### Test 1: Check nginx is routing correctly

```bash
# Test cPanel domain (should stay on CT101)
pct exec 101 -- curl -H "Host: thelightville.com" http://127.0.0.1:8888 -I

# Test CT103 domain (should proxy to CT103)
pct exec 101 -- curl -k -H "Host: app.onlineradio.com.ng" https://127.0.0.1:8888 -I

# Test CT105 domain (should proxy to CT105)
pct exec 101 -- curl -k -H "Host: claapp.myapps.com.ng" https://127.0.0.1:8888 -I

# Test CT107 domain (should proxy to CT107)
pct exec 101 -- curl -k -H "Host: api.onlineradio.com.ng" https://127.0.0.1:8888 -I
```

### Test 2: Check tunnel is active

```bash
pct exec 101 -- systemctl status cloudflared
# Should show: 4 registered QUIC connections
```

### Test 3: Check from public internet (after DNS updates)

```bash
# Should show Cloudflare headers
curl -I https://app.onlineradio.com.ng | grep -i cf-
curl -I https://claapp.myapps.com.ng | grep -i cf-
curl -I https://api.onlineradio.com.ng | grep -i cf-
```

---

## Why NOT Route Tunnel to Proxmox?

You might wonder: "Why not terminate tunnel on Proxmox and use existing nginx stream routing?"

### Problems with that approach:

1. **Can't run cloudflared on Proxmox easily**
   - Would need to install in host OS (risky)
   - Containers are isolated (better security)

2. **Protocol mismatch**
   - Tunnel delivers HTTP traffic
   - Proxmox nginx stream reads SNI from TLS
   - Would need to re-encrypt just for Proxmox to read, then decrypt again

3. **Less flexible**
   - Can't modify HTTP headers
   - Can't do intelligent caching decisions
   - Can't add authentication layers

4. **Single point of failure**
   - If Proxmox routing breaks, everything breaks
   - Current setup: Containers still accessible via fallback

5. **Origin IP still exposed**
   - Proxmox would be the entry point
   - Attackers could find it via other services

---

## Why CT101 nginx is the RIGHT Approach

### Industry Standard
This is how major websites handle traffic:
- **Netflix**: Uses nginx reverse proxies
- **Airbnb**: Uses nginx reverse proxies
- **GitHub**: Uses nginx reverse proxies
- **CloudFlare itself**: Uses nginx internally!

### Benefits:

1. ✅ **Layer 7 routing** - Can route by URL path, headers, cookies
2. ✅ **Centralized management** - One config file for all routing
3. ✅ **Easy to debug** - Can log all requests in one place
4. ✅ **Flexible** - Can add authentication, rate limiting per domain
5. ✅ **Scalable** - Easy to add new containers/domains
6. ✅ **Proven technology** - Millions of sites use this pattern

---

## Summary: Your Domains WILL Reach Correct Containers! ✅

### How routing works:

1. **All traffic enters** through Cloudflare Tunnel → CT101:8888
2. **nginx reads** the hostname (domain name)
3. **nginx routes** to the correct container based on hostname
4. **Container processes** request and responds
5. **Response flows back** through nginx → Cloudflare → user

### This is EXACTLY like Proxmox routing:
- Proxmox: Read SNI hostname → Route to container
- CT101 nginx: Read HTTP hostname → Route to container

### Same concept, same result! ✅

---

## What About thelightville.com DNS?

**If thelightville.com is in Cloudflare:** Already updated to tunnel CNAME ✅

**If NOT in Cloudflare:** You should add CNAME record:
```
thelightville.com    CNAME    a0bd27a8-26c6-498a-bb7f-a88284a532ff.cfargotunnel.com
www.thelightville.com CNAME   a0bd27a8-26c6-498a-bb7f-a88284a532ff.cfargotunnel.com
```

**All 29 cPanel domains should have tunnel CNAME!**

---

## Trust the Architecture! ✅

This setup is:
- ✅ **Proven** - Industry standard pattern
- ✅ **Secure** - Origin IP hidden, full protection
- ✅ **Fast** - nginx is extremely performant
- ✅ **Flexible** - Easy to extend and modify
- ✅ **Reliable** - Used by millions of websites

**Your domains WILL reach their correct containers!** 🎉
