# 🔍 Port 2087 (WHM) Routing Issue - Diagnosis & Fix

**Date**: October 13, 2025  
**Issue**: Reverse proxy forwarding traffic to itself instead of cPanel server  
**Discovered by**: tcpdump analysis  
**Priority**: 🔴 HIGH - Affecting WHM access

---

## 🎯 PROBLEM IDENTIFIED

### tcpdump Evidence

#### On Proxmox Host (enp0s25):
```
05:26:01.861194 enp0s25 P ifindex 2 7c:5a:1c:7c:ef:e1 ethertype IPv4 (0x0800), length 72: 
182.77.75.12.29747 > 172.16.16.100.2087: Flags [S], seq 1852113410, win 65535, 
options [mss 1250,nop,wscale 8,nop,nop,sackOK], length 0
```

**Analysis**: External IP **182.77.75.12** connects to reverse proxy **172.16.16.100:2087** ✅

#### On Reverse Proxy (eth0):
```
09:26:01.861212 eth0 In ifindex 2 7c:5a:1c:7c:ef:e1 ethertype IPv4 (0x0800), length 72: 
172.16.16.20.29747 > 172.16.16.100.2087: Flags [S], seq 1852113410, win 65535, 
options [mss 1250,nop,wscale 8,nop,nop,sackOK], length 0
```

**Analysis**: Source IP changed from **182.77.75.12** to **172.16.16.20** (Proxmox host) ❌  
**Problem**: Traffic destination shows **172.16.16.100:2087** (itself) instead of **172.16.16.101:2087** ❌

---

## 🔍 ROOT CAUSE ANALYSIS

### Issue 1: Source IP NAT
The source IP is being NAT'd from the external IP (**182.77.75.12**) to the Proxmox host IP (**172.16.16.20**).

**Why this happens**:
- iptables MASQUERADE rule on Proxmox host
- This is actually CORRECT behavior for internal routing
- The reverse proxy should still forward to the correct backend

### Issue 2: Destination Not Changing (The Real Problem)
The destination IP remains **172.16.16.100** instead of changing to **172.16.16.101**.

**This suggests**:
1. Nginx is listening on 2087 but NOT proxy_passing correctly, OR
2. There's a routing loop, OR
3. The proxy_pass directive has the wrong IP

---

## ✅ CURRENT CONFIGURATION

### Nginx Config (Confirmed)
```nginx
server {
    listen 2087;
    proxy_pass 172.16.16.101:2087;
    proxy_timeout 300s;
    proxy_connect_timeout 10s;
}
```

**Status**: Configuration looks CORRECT ✅

### Backend Status
```bash
# CT 101 (cPanel) listening on 2087:
LISTEN 0  45  0.0.0.0:2087  0.0.0.0:*  users:(("cpsrvd (SSL)",pid=244,fd=6))
```

**Status**: cPanel WHM service is running ✅

### Connectivity Test
```bash
# Direct connection test:
echo > /dev/tcp/172.16.16.101/2087
Result: ✅ Connection successful
```

**Status**: Network routing works ✅

---

## 🔧 THE ISSUE: TCPDUMP OUTPUT INTERPRETATION

### What tcpdump Actually Shows

When you see this on the reverse proxy:
```
172.16.16.20.29747 > 172.16.16.100.2087
```

This is the **INCOMING** packet (before nginx processes it). The key word in your tcpdump output is:
```
eth0  In  ifindex 2
```

The **"In"** means this is an incoming packet that nginx is about to receive and process.

### What You Should See Next

To confirm nginx is forwarding correctly, you need to capture **OUTGOING** traffic:

```bash
# On reverse proxy, capture outgoing traffic to CT 101:
pct exec 100 -- tcpdump -i eth0 -n 'dst host 172.16.16.101 and port 2087' -c 5
```

This should show packets going FROM 172.16.16.100 TO 172.16.16.101.

---

## 🧪 PROPER DIAGNOSTIC PROCEDURE

### Step 1: Capture on Reverse Proxy (Both Directions)
```bash
# Run this on Proxmox host
pct exec 100 -- timeout 10 tcpdump -i eth0 -n 'port 2087' -v &

# Then make a test connection:
curl -k https://142.180.236.143:2087 --connect-timeout 5
```

**Expected Output**:
```
# Incoming (what you saw):
172.16.16.20.X > 172.16.16.100.2087 [SYN]

# Outgoing (what should also appear):
172.16.16.100.Y > 172.16.16.101.2087 [SYN]
```

### Step 2: Capture on cPanel Backend
```bash
# Run simultaneously on CT 101
pct exec 101 -- timeout 10 tcpdump -i eth0 -n 'port 2087' -v
```

**Expected**: You should see packets arriving FROM 172.16.16.100

### Step 3: Three-Point Capture (Complete Picture)
```bash
# Terminal 1: Proxmox host
tcpdump -i vmbr0 -n 'host 172.16.16.100 and port 2087' -c 20 &

# Terminal 2: Reverse proxy
pct exec 100 -- tcpdump -i eth0 -n 'port 2087' -c 20 &

# Terminal 3: cPanel backend
pct exec 101 -- tcpdump -i eth0 -n 'port 2087' -c 20 &

# Terminal 4: Test connection
sleep 2 && curl -k https://142.180.236.143:2087 --connect-timeout 5
```

---

## 🛠️ TROUBLESHOOTING STEPS

### Test 1: Verify Nginx is Actually Proxying
```bash
# Check nginx stream access logs
pct exec 100 -- tail -20 /var/log/nginx/stream-access.log | grep 2087
```

**Expected**: No entries (2087 is in stream block, not HTTP block)

Wait, this might be the issue! Let me check...

### Test 2: Check if 2087 is in HTTP or Stream Block
```bash
pct exec 100 -- nginx -T 2>/dev/null | grep -B 10 "listen 2087" | grep -E "stream|http"
```

### Test 3: Verify No Conflicting Iptables Rules
```bash
# Check if there's a DNAT loop
iptables -t nat -L OUTPUT -n -v | grep 2087
```

---

## 🔧 POTENTIAL FIXES

### Fix 1: Ensure proxy_bind is Set (If Needed)
If nginx can't determine the correct source IP for outgoing connections:

```nginx
stream {
    server {
        listen 2087;
        proxy_pass 172.16.16.101:2087;
        proxy_bind 172.16.16.100;  # <-- Add this
        proxy_timeout 300s;
        proxy_connect_timeout 10s;
    }
}
```

### Fix 2: Check if Port 2087 Config is in Stream Block
The configuration should be in the `stream {}` block, not `http {}` block:

```nginx
stream {
    server {
        listen 2087;
        proxy_pass 172.16.16.101:2087;
    }
}
```

NOT in:
```nginx
http {
    server {
        listen 2087;  # <-- WRONG BLOCK!
    }
}
```

### Fix 3: Add Explicit Routing (If Needed)
```bash
# Add static route if nginx can't reach 172.16.16.101
pct exec 100 -- ip route add 172.16.16.101/32 via 172.16.16.1 dev eth0
```

---

## ✅ VERIFICATION COMMANDS

### After Any Fix, Run These Tests:

#### Test 1: End-to-End Connection
```bash
timeout 5 curl -k -I https://142.180.236.143:2087 2>&1 | head -5
```

**Expected**: HTTP response from cPanel WHM

#### Test 2: Three-Point tcpdump
```bash
# All in parallel:
tcpdump -i vmbr0 -n 'port 2087' -c 5 2>&1 | tee /tmp/host.log &
pct exec 100 -- tcpdump -i eth0 -n 'port 2087' -c 5 2>&1 | tee /tmp/proxy.log &
pct exec 101 -- tcpdump -i eth0 -n 'port 2087' -c 5 2>&1 | tee /tmp/backend.log &

sleep 2
curl -k https://142.180.236.143:2087 --connect-timeout 5

# Check logs
echo "=== HOST ===" && cat /tmp/host.log
echo "=== PROXY ===" && cat /tmp/proxy.log  
echo "=== BACKEND ===" && cat /tmp/backend.log
```

#### Test 3: Nginx Logs
```bash
# Check for connection attempts
pct exec 100 -- tail -f /var/log/nginx/stream-access.log | grep 2087
```

---

## 📊 EXPECTED vs ACTUAL TRAFFIC FLOW

### Expected Flow
```
External Client (182.77.75.12:X)
    ↓
Sophos Firewall (172.16.16.16)
    ↓ DNAT
Proxmox Host (172.16.16.20) - iptables DNAT
    ↓ Forward to 172.16.16.100:2087
Reverse Proxy (172.16.16.100)
    ↓ Nginx proxy_pass to 172.16.16.101:2087
cPanel Server (172.16.16.101:2087)
    ↓ Response
[Return path reverses]
```

### What tcpdump Should Show

**On Proxmox host (vmbr0)**:
```
182.77.75.12.X > 172.16.16.100.2087  [SYN]     # Incoming from external
172.16.16.100.Y > 172.16.16.101.2087 [SYN]     # Forwarded to backend
172.16.16.101.2087 > 172.16.16.100.Y [SYN-ACK] # Backend response
172.16.16.100.2087 > 182.77.75.12.X  [SYN-ACK] # Response to external
```

**On Reverse Proxy (172.16.16.100 eth0)**:
```
172.16.16.20.X > 172.16.16.100.2087  [SYN]     # Incoming (NAT'd source)
172.16.16.100.Y > 172.16.16.101.2087 [SYN]     # Outgoing to backend
172.16.16.101.2087 > 172.16.16.100.Y [SYN-ACK] # Backend response
172.16.16.100.2087 > 172.16.16.20.X  [SYN-ACK] # Response back
```

**On cPanel Backend (172.16.16.101 eth0)**:
```
172.16.16.100.Y > 172.16.16.101.2087 [SYN]     # From reverse proxy
172.16.16.101.2087 > 172.16.16.100.Y [SYN-ACK] # Response
```

---

## 🎯 IMMEDIATE ACTION ITEMS

1. **Run three-point tcpdump** to see complete traffic flow
2. **Check if packets reach CT 101** 
3. **Verify nginx is in stream block** (not http block)
4. **Check nginx error logs** for proxy failures

---

## 📝 DIAGNOSTIC SCRIPT

Save this as `/root/diagnose-port-2087.sh`:

```bash
#!/bin/bash
echo "=== Port 2087 Diagnostic Script ==="
echo ""

echo "1. Checking Nginx Configuration..."
pct exec 100 -- nginx -T 2>/dev/null | grep -A 5 "listen 2087"
echo ""

echo "2. Checking if cPanel is listening..."
pct exec 101 -- ss -tlnp | grep 2087
echo ""

echo "3. Starting tcpdump on all three points..."
echo "   Starting backend capture..."
pct exec 101 -- timeout 10 tcpdump -i eth0 -n 'port 2087' 2>&1 | tee /tmp/backend-2087.log &
sleep 1

echo "   Starting proxy capture..."
pct exec 100 -- timeout 10 tcpdump -i eth0 -n 'port 2087' 2>&1 | tee /tmp/proxy-2087.log &
sleep 1

echo "   Starting host capture..."
timeout 10 tcpdump -i vmbr0 -n 'port 2087' 2>&1 | tee /tmp/host-2087.log &
sleep 2

echo "4. Making test connection..."
curl -k -I --connect-timeout 5 https://142.180.236.143:2087 > /tmp/curl-result.txt 2>&1

sleep 3

echo ""
echo "5. Analysis:"
echo ""
echo "=== CURL Result ==="
cat /tmp/curl-result.txt
echo ""
echo "=== Host tcpdump (Proxmox) ==="
cat /tmp/host-2087.log 2>/dev/null | head -20
echo ""
echo "=== Proxy tcpdump (CT 100) ==="
cat /tmp/proxy-2087.log 2>/dev/null | head -20
echo ""
echo "=== Backend tcpdump (CT 101) ==="
cat /tmp/backend-2087.log 2>/dev/null | head -20

echo ""
echo "=== Summary ==="
if grep -q "172.16.16.101" /tmp/proxy-2087.log 2>/dev/null; then
    echo "✅ Proxy IS forwarding to backend (172.16.16.101)"
else
    echo "❌ Proxy NOT forwarding to backend - CHECK NGINX CONFIG!"
fi

if grep -q "172.16.16.100" /tmp/backend-2087.log 2>/dev/null; then
    echo "✅ Backend IS receiving traffic from proxy"
else
    echo "❌ Backend NOT receiving traffic - ROUTING ISSUE!"
fi
```

**Usage**:
```bash
chmod +x /root/diagnose-port-2087.sh
/root/diagnose-port-2087.sh
```

---

**Status**: 🔍 Diagnosis in progress  
**Next Step**: Run diagnostic script to confirm if traffic reaches CT 101  
**Priority**: Fix if WHM access is required externally
