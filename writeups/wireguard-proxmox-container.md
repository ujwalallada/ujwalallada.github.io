# 🔐 Homelab Remote Access using Proxmox + WireGuard (LXC Attempt)

> Portfolio documentation: objective, architecture, configs, troubleshooting, root-cause, and final decision to move WG server from LXC → Debian VM.

## Objective
- Secure remote access to homelab; Keep hypervisor clean; Segment access; Document it.

## Environment
- Proxmox Host (static): **192.168.1.10**
- Debian VPN server (LXC, attempt 1): **192.168.1.20**
- Router/Gateway: **192.168.1.1**
- LAN: **192.168.1.0/24**
- WireGuard subnet: **10.8.0.0/24**
- Port forward: **UDP 51820 → 192.168.1.20**
- Public IPv4 present; PVE firewall disabled for tests.

## Architecture

```mermaid
flowchart LR
  internet((Internet))
  phone[Phone (au-phone)]
  laptop[Work Laptop (au-worklaptop)]
  wan[Home Router (WAN Public IP)]

  phone -->|UDP 51820| internet
  laptop -->|UDP 51820| internet
  internet -->|UDP 51820| wan

  subgraph LAN [Home LAN 192.168.1.0/24]
    direction LR
    lanrouter[Home Router 192.168.1.1] --- proxmox[Proxmox Host 192.168.1.10]
    subgraph PVE [Proxmox]
      direction TB
      lxc[Debian LXC (Attempt 1)\nWireGuard Server\n192.168.1.20]:::warn
      vm[Debian VM (Planned)\nWireGuard Server]:::future
    end
  end

  wan -->|Port forward UDP 51820| lxc
  wan -.->|Planned: forward UDP 51820| vm

  classDef warn fill:#fff3cd,stroke:#d39e00,color:#000;
  classDef future fill:#e7f1ff,stroke:#2f6fdd,color:#000;


**Key Configs (LXC Attempt)**
Server /etc/wireguard/wg0.conf (minimal, IPv4):

[Interface]
Address    = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <server_private_key>
PostUp   = iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

[Peer]  # au-phone
PublicKey = <au_phone_public>
AllowedIPs = 10.8.0.2/32
PersistentKeepalive = 25

[Peer]  # au-worklaptop
PublicKey = <au_worklaptop_public>
AllowedIPs = 10.8.0.3/32
PersistentKeepalive = 25

**Client template (Android/Laptop):**
[Interface]
PrivateKey = <client_private>
Address = 10.8.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <server_public>
Endpoint = <PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25

**Troubleshooting Highlights**

Verified: ss -lunp | grep 51820 (WG listening), sysctl net.ipv4.ip_forward=1, MASQUERADE present, router port forward correct, public IPv4 present, PVE firewall disabled.

tcpdump path checks:

Host vmbr0: check UDP/51820 arrives from WAN.

Host veth104i0: check forwarding into the CT.

Inside LXC eth0: check packets inside CT.

Outcome: no reliable handshake at the CT boundary → containerization/kernel/capability constraints, not WG config.

**Root Cause**

Running a WG server inside LXC adds kernel/capability hurdles (module loading, /dev/net/tun, CAP_NET_ADMIN, AppArmor/cgroup constraints, bridge/veth path). Result: handshakes fail despite correct configs.

**Decision**

Migrate WG server to a Debian VM on Proxmox (full kernel control, simpler routing/NAT, production-like).

**Future (Design Only)**

Build Debian VM for WG; move port forward to VM.

After NIC (Intel i350-T2/T4) install: introduce pfSense as edge firewall/router; add VLANs, DNS control, reverse proxy.

