# Lab 01 — VLAN Segmentation, Inter-VLAN Routing and IOS Hardening

Small enterprise-like network lab built in Cisco Packet Tracer. The focus is on
Layer 2 segmentation, inter-VLAN routing, SSH-only management access, device
hardening, STP protection, and centralized logging with NTP.

**Status:** Published

---

## Objective

Design and implement a segmented network with a hardened management plane,
secure remote access, and centralized event logging — treating the lab as a
coherent security scenario rather than a collection of isolated commands.

---

## Topology

```
[PC1]───[SW1 (Cisco 2960)]───[R1 (Cisco ISR 4331)]───[PC2 - WAN Host]
                    │
              [Server: NTP + Syslog]
```

| Device | Model | Role |
|---|---|---|
| R1 | Cisco ISR 4331 | Core router, inter-VLAN routing, WAN uplink, SSH gateway |
| SW1 | Cisco Catalyst 2960-24TT | Access switch |
| PC1 | PC-PT | User endpoint (VLAN 10) |
| PC2 | PC-PT | WAN host (Internet simulation) |
| Server | Server-PT | NTP + Syslog |

---

## IP Addressing

| Device | Interface | IP Address | Mask | Role |
|---|---|---|---|---|
| R1 | Gi0/0/0.10 | 192.168.10.1 | /24 | VLAN 10 gateway |
| R1 | Gi0/0/0.101 | 192.168.100.1 | /29 | Management gateway |
| R1 | Gi0/0/1 | 203.0.113.1 | /30 | WAN uplink |
| SW1 | VLAN 101 SVI | 192.168.100.2 | /29 | SSH management access |
| Server | NIC | 192.168.100.5 | /29 | NTP + Syslog |
| PC1 | NIC | 192.168.10.10 | /24 | User endpoint |
| PC2 | NIC | 203.0.113.2 | /30 | WAN endpoint |

---

## VLAN Design

| VLAN | Name | Purpose |
|---|---|---|
| 10 | USERS | User traffic |
| 99 | NATIVE | Dedicated native VLAN on trunk — carries no traffic |
| 101 | MANAGEMENT | Out-of-band management for SSH access to R1 and SW1 |
| 999 | UNUSED | All inactive switch ports isolated here and shut down |

---

## What Was Implemented

**Layer 2 / Switching**
- VLAN segmentation (VLANs 10, 99, 101, 999)
- 802.1Q trunk SW1 → R1, allowed VLANs explicitly restricted to 10 and 101
- Native VLAN 99 — dedicated, no user or management traffic
- DTP disabled on all ports (`switchport nonegotiate`)
- All unused ports assigned to VLAN 999 and shut down

**Layer 3 / Routing**
- Inter-VLAN routing via Router-on-a-Stick (subinterfaces Gi0/0/0.10, Gi0/0/0.101)
- WAN interface /30 with `no cdp enable`
- `no ip proxy-arp` on all subinterfaces

**SSH and Management Access**
- SSH v2, RSA 4096 bit, Telnet explicitly blocked on all 16 VTY lines
- Local user account, `enable secret`, `security passwords min-length 12`
- `login block-for 300 attempts 3 within 120` + `login on-failure log`
- `exec-timeout 300 0` on console and all VTY lines
- Banner MOTD before authentication

**STP**
- PVST+ mode
- `spanning-tree portfast default` + `spanning-tree portfast bpduguard default`

**Syslog and NTP**
- Centralized syslog to Server 192.168.100.5, `logging buffered 16384`
- `service timestamps log datetime msec` on R1 and SW1
- NTP synchronized from Server, `clock timezone CET 1`

**Hardening documented but not supported in Packet Tracer**
- `no ip source-route`, TCP/UDP small servers, finger, HTTP server, bootp
- Type 9 scrypt password hashing
- `clock summer-time CEST recurring`
- `logging source-interface`, `service sequence-numbers`

---

## Validation

All tests passed after full configuration of R1, SW1, and Server.

| Test | Result |
|---|---|
| SSH v2 active on R1 | ✅ Pass |
| All VTY lines SSH-only, Telnet blocked | ✅ Pass |
| Interface status R1 — all active interfaces up/up | ✅ Pass |
| Trunk 802.1Q, native VLAN 99, allowed VLANs 10 and 101 | ✅ Pass |
| VLAN table correct, unused ports in VLAN 999 | ✅ Pass |
| STP — SW1 root bridge, all ports Forwarding | ✅ Pass |
| Syslog forwarding with msec timestamps — R1 and SW1 | ✅ Pass |
| PC1 → VLAN 10 gateway | ✅ Pass |
| PC1 → Management server (inter-VLAN routing) | ✅ Pass |
| PC1 → WAN host | ✅ Pass |
| SSH login PC1 → R1 | ✅ Pass |

Full verification output: [`docs/validation.md`](docs/validation.md)

---

## Documentation

- [`docs/topology.md`](docs/topology.md) — topology, addressing, VLAN design, architectural decisions
- [`docs/implementation.md`](docs/implementation.md) — full configuration with reasoning behind each command
- [`docs/validation.md`](docs/validation.md) — verification commands and results
- [`docs/lessons-learned.md`](docs/lessons-learned.md) — deeper explanations, PT limitations, and possible extensions

---

## Technologies

Cisco Packet Tracer · Cisco IOS 15.4 (R1) / 15.0 (SW1) · VLANs · 802.1Q trunking ·
Router-on-a-Stick · SSH v2 · PVST+ · BPDUGuard · PortFast · Syslog · NTP

---

## Possible Extensions

- ACL restricting SSH access by source IP with `access-class` on VTY lines
- AAA with `aaa new-model` replacing line-level `login local`
- `ip ssh source-interface` to bind management traffic to the management segment
- Second switch to create meaningful STP port roles (root port, designated, blocked)
- Verify syslog reception on the server side and correlate timestamps across devices
