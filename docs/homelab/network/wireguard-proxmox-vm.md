# 🔐 WireGuard VPN Setup on Debian VM

A documentation of my attempt to set up WireGuard VPN on a Debian VM, the challenges I encountered, and why I'm moving to an OPNsense-based solution.

---

## Overview

**Goal:** Set up a WireGuard VPN to securely access my home network from anywhere.

**Environment:**
- Proxmox Host (static): **192.168.1.10**
- Debian VPN server (LXC, attempt 1): **192.168.1.171**
- Home network with dynamic IPv6
- Mobile and laptop clients

**Outcome:** Approach failed due to ISP router limitations. Switching to OPNsense firewall solution.

---

## System Architecture (Attempted)

```
Internet
   │
   └─ ISP Router (192.168.1.1)
        ├─ Proxmox Host
        │    └─ Debian VM (192.168.1.171)
        │         └─ WireGuard Server (port 51820)
        │
        └─ Clients (mobile, laptop)
```

---

## Setup Process

### 1. Install WireGuard

On the Debian VM:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wireguard wireguard-tools curl -y
```

### 2. Enable IP Forwarding

Create `/etc/sysctl.d/99-sysctl.conf` and add:

```ini
net.ipv4.ip_forward=1
```

Apply with:
```bash
sudo sysctl --system
```

### 3. Generate Keys

For the server and each client:

```bash
cd /etc/wireguard
sudo umask 077

# Server keys
sudo wg genkey | sudo tee server_private.key | wg pubkey | sudo tee server_public.key

# Mobile client keys
sudo wg genkey | sudo tee mobile_private.key | wg pubkey | sudo tee mobile_public.key
```

### 4. Configure the Server

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = <SERVER_PRIVATE_KEY>
Address = 10.8.0.1/24
ListenPort = 51820

# NAT rules - replace ens18 with your interface name
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE

[Peer]    # Mobile Client
PublicKey = <MOBILE_PUBLIC_KEY>
AllowedIPs = 10.8.0.2/32
PersistentKeepalive = 25
```

Start WireGuard:
```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

### 5. Configure Firewall

Set up UFW to allow WireGuard:

```bash
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 51820/udp   # WireGuard
sudo ufw enable
```

### 6. Configure Clients

**Mobile (Android):**

Create a config file:

```ini
[Interface]
PrivateKey = <MOBILE_PRIVATE_KEY>
Address = 10.8.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <YOUR_IPV6_OR_DOMAIN>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Import into the WireGuard app (via QR code or manual entry).

---

## The Challenges I Faced

### Challenge 1: Router Firewall Blocking VPN

**The Problem:**
My router's firewall is set to "High" security mode, blocking all incoming VPN connections. While I *can* disable the firewall, **I choose not to** due to serious security implications (explained below).

**What I Tried:**

1. **Port Forwarding Configuration**
   - Correctly set up UDP 51820 → 192.168.1.171
   - Verified multiple times, rebooted router

2. **Firewall Security Levels**
   - Tried Medium and Low security settings
   - Still blocked incoming VPN traffic

3. **Tested with Firewall Disabled**
   - Temporarily disabled firewall to test
   - VPN worked perfectly during this time

### Challenge 2: IPv6 + DuckDNS Approach

**The Attempt:**
Since IPv6 doesn't require port forwarding (your device gets a real public address), I tried using IPv6 with DuckDNS for dynamic DNS.

**DuckDNS Setup:**

DuckDNS is a free service that gives you a permanent domain name that automatically updates to point to your current IP.

Setup on server:

```bash
mkdir ~/duckdns
cd ~/duckdns
nano duck.sh
```

Script content:
```bash
#!/bin/bash
IPV6=$(curl -6 -s ifconfig.me)
echo url="https://www.duckdns.org/update?domains=YOUR-SUBDOMAIN&token=YOUR-TOKEN&ipv6=$IPV6&clear=true" | curl -k -o ~/duckdns/duck.log -K -
```

**Important:** The `&clear=true` parameter removes the IPv4 address from DNS, forcing clients to use IPv6 only.

Set to auto-update every 5 minutes:
```bash
chmod +x duck.sh
crontab -e
```

Add:
```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

**Cron Syntax Explained:**
- `*/5` = Every 5 minutes
- `* * * *` = Every hour, day, month, and day of week
- `>/dev/null 2>&1` = Discard output to avoid email spam

**Why This Failed:**

Despite having IPv6 and DuckDNS configured:
- VPN worked when firewall was disabled (testing only)
- With firewall re-enabled at any security level, all connections blocked
- **The router's firewall blocks incoming traffic regardless of IPv4 or IPv6**
- DuckDNS correctly updated addresses, but traffic never reached my server

**Lesson Learned:** IPv6 bypasses NAT/port forwarding complexity, but it doesn't bypass firewall rules. Whether using IPv4 or IPv6, if the firewall blocks incoming connections on port 51820, the VPN won't work.

---

## Why This Approach Failed

### Summary of Blockers

| Issue | Impact | Possible Solution |
|-------|--------|-------------------|
| Router firewall blocks VPN | Cannot allow incoming connections | ❌ Could disable, but security risk |
| CGNAT | IPv4 port forwarding difficult | ✅ Use IPv6 (but still blocked by firewall) |
| Firewall blocks all incoming | Both IPv4 and IPv6 blocked | ❌ Need firewall control |
| Dynamic IPs | Config breaks frequently | ✅ DuckDNS (works but doesn't solve firewall) |

**The Core Problem:** While I *could* disable the router's firewall to make VPN work, I'm **unwilling to compromise security**. I need a solution that provides both VPN access and proper network protection.

---

## The Solution: Moving to OPNsense

### Why OPNsense?

Since I choose not to disable the router's firewall for security reasons, I need to place my own firewall/router **in front** of my network that I have full control over - one that can properly allow VPN traffic while maintaining security.

### Technical Lessons

✅ **WireGuard setup itself works** - My configuration was correct  
✅ **DuckDNS handles dynamic IPs** - Useful for any future setup  
✅ **IPv6 bypasses CGNAT** - But not firewall restrictions  
❌ **Cannot fix ISP router limitations** - Need different approach  
❌ **IPv4 port forwarding fails with CGNAT** - Common ISP limitation  



### Understanding CGNAT

**What is CGNAT?**
Carrier-Grade NAT is a technique ISPs use when they run out of public IPv4 addresses. Instead of giving each customer a unique public IP, they share one public IP among many customers.

**Analogy:**
Think of it like an apartment building:
- **Normal setup:** Your house has its own street address (public IP)
- **CGNAT:** You live in apartment #5 in a building. The building has one street address (ISP's public IP), and mail is sorted internally.

**Why ISPs Use It:**
- IPv4 address exhaustion (not enough IPs for everyone)
- Cost savings (public IPs are expensive)
- Network management and control

**How to Detect:**
```bash
# Your router's WAN IP (check in router web interface)
WAN IP: 100.64.15.42  # Private range = definitely CGNAT

# vs. what internet sees
curl ifconfig.me
72.136.107.72  # Different = CGNAT
```

**Impact on VPN:**
- ✅ Outbound connections work fine (you can connect to other VPNs)
- ❌ Inbound connections impossible (can't host your own VPN)
- ❌ Port forwarding doesn't work
- ❌ Running any server from home blocked

**Solutions:**
1. Ask ISP for a "public IP" (often costs extra)
2. Switch ISPs (not always possible)
3. Use IPv6 (no CGNAT, but may still have firewall issues)
4. Use VPS relay (tunnel through cloud server)
5. Use proper edge firewall like OPNsense in bridge mode

---

## Useful Commands for Future Reference

**Check if VPN is working:**
```bash
# On server
sudo wg show
# Should show connected peers with recent handshakes
```

**Monitor connections:**
```bash
# Watch for incoming VPN packets
sudo tcpdump -i ens18 udp port 51820 -n

# Watch decrypted VPN traffic
sudo tcpdump -i wg0 -n
```

**Verify your current IPs:**
```bash
curl -6 ifconfig.me  # IPv6
curl -4 ifconfig.me  # IPv4
```

**Check for CGNAT:**
```bash
curl ifconfig.me
# Compare with router WAN IP in web interface
# Different = CGNAT
```

**Test port forwarding:**
```bash
# From external network
nc -zvu YOUR_PUBLIC_IP 51820
# Should see "succeeded" if port is open
```


---

## Conclusion

While the WireGuard setup on Debian VM was technically sound, **the router's firewall blocking incoming connections made it unviable** without disabling security features. Since I'm unwilling to compromise network security by disabling the firewall, I need a better architectural solution.

**Moving to OPNsense** provides:
- ✅ Full control over network edge
- ✅ Proper firewall capabilities
- ✅ Professional-grade features
- ✅ Sustainable long-term solution
- ✅ No security compromises

This experience taught me valuable lessons about network architecture, ISP limitations, and the importance of proper infrastructure. The time spent troubleshooting wasn't wasted - understanding *why* this approach failed will inform better decisions in the OPNsense implementation.

---

**Status:** Failed Approach - Moving to OPNsense  
