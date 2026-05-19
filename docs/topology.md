# topology.md — Lab 01: IOS Basic Configuration & Security Hardening

## Project Overview

This lab covers the design and implementation of a small enterprise-like network with a focus on IOS device hardening, secure administrative access, Layer 2 segmentation, inter-VLAN routing, and centralized management services. The topology reflects deliberate architectural decisions made at the design stage to reflect real-world security and operational best practices.

---

## Topology Diagram

```
[PC1]───[SW1 (Cisco 2960)]───[R1 (Cisco ISR 4331)]───[PC2 - WAN Host]
                    │
              [Server: NTP + Syslog]
```

Physical connections:

- PC1 → SW1 Fa0/1 (Access, VLAN 10)
- Server → SW1 Fa0/24 (Access, VLAN 101)
- SW1 Gi0/1 → R1 Gi0/0/0 (802.1Q Trunk)
- R1 Gi0/0/1 → PC2 (WAN simulation link, /30)

---

## Devices

| Device | Model | Role |
|---|---|---|
| R1 | Cisco ISR 4331 | Core router, inter-VLAN routing, WAN uplink, SSH gateway |
| SW1 | Cisco Catalyst 2960-24TT | Access and distribution switch |
| PC1 | PC-PT | End-user host, VLAN 10 |
| PC2 | PC-PT | WAN-side host simulating Internet endpoint |
| Server | Server-PT | NTP server, Syslog server |

---

## IP Addressing Plan

| Device | Interface | IP Address | Mask | Network | Role |
|---|---|---|---|---|---|
| R1 | Gi0/0/0.10 | 192.168.10.1 | /24 | 192.168.10.0/24 | VLAN 10 gateway |
| R1 | Gi0/0/0.101 | 192.168.100.1 | /29 | 192.168.100.0/29 | Management gateway |
| R1 | Gi0/0/1 | 203.0.113.1 | /30 | 203.0.113.0/30 | WAN uplink |
| SW1 | VLAN 101 SVI | 192.168.100.2 | /29 | 192.168.100.0/29 | Management access (SSH) |
| Server | NIC | 192.168.100.5 | /29 | 192.168.100.0/29 | NTP + Syslog |
| PC1 | NIC | 192.168.10.10 | /24 | 192.168.10.0/24 | User endpoint |
| PC2 | NIC | 203.0.113.2 | /30 | 203.0.113.0/30 | WAN endpoint (Internet simulation) |

---

## VLAN Design

| VLAN ID | Name | Purpose |
|---|---|---|
| 10 | USERS | User traffic — PC1 and future endpoint devices |
| 99 | NATIVE | Dedicated native VLAN on trunk links — carries no user or management traffic |
| 101 | MANAGEMENT | Out-of-band management segment for SSH access to R1 and SW1 |
| 999 | UNUSED | All inactive switch ports are isolated here and administratively shut down |

---

## Architectural Decisions

### Management VLAN 101 on a dedicated /29 segment

The management plane is separated from the user plane at both Layer 2 (dedicated VLAN 101) and Layer 3 (192.168.100.0/29 vs. 192.168.10.0/24). This reflects the principle of management plane isolation: administrative access to network devices should never share the same broadcast domain or IP segment as end-user traffic.

A /29 mask (6 usable hosts) was chosen deliberately. The management segment contains only R1, SW1, and the NTP/Syslog server. A larger subnet would provide no operational benefit and would unnecessarily expand the attack surface of the management plane.

VLAN 101 was assigned instead of the commonly used VLAN 100 as a minor security-through-obscurity measure and to avoid predictable defaults that automated scanners or attackers might target first.

### Native VLAN 99 — dedicated and unused

IEEE 802.1Q trunk ports require a native VLAN for untagged frames. Using the default native VLAN 1 is a known security risk: VLAN hopping attacks can exploit the default native VLAN to gain unauthorized access to the management or user plane.

VLAN 99 (NATIVE) was created explicitly for the native VLAN role. It carries no IP addressing and no ports are assigned to it in access mode. Its sole purpose is to receive any untagged frames that may appear on the trunk link, effectively neutralizing VLAN hopping attempts.

### VLAN 999 — isolation of unused ports

All inactive switch ports are assigned to VLAN 999 (UNUSED) and administratively shut down. This follows the principle of least privilege at Layer 2: an unused port should have no path to any active VLAN or segment. Even in shutdown state, the explicit VLAN assignment ensures that if a port is accidentally re-enabled, it does not inherit access to VLAN 1 or any operational VLAN.

### PC2 as WAN endpoint

The second host in the topology (PC2) is placed on the WAN-side subnet (203.0.113.0/30), connected directly to R1's Gi0/0/1 interface. This allows simulation of WAN-side reachability testing without requiring an actual ISP connection or additional routing infrastructure. The 203.0.113.0/24 prefix is from the IANA-reserved TEST-NET-3 documentation range, making the lab topology self-contained and safe to document publicly.

---

## Security Zones

```
[ VLAN 10 — User Zone ]        192.168.10.0/24
       ↕ inter-VLAN routing via R1
[ VLAN 101 — Management Zone ] 192.168.100.0/29
       ↕ WAN interface
[ WAN — Internet Simulation ]  203.0.113.0/30
```

Administrative access (SSH) is only possible from within the management zone. CDP is disabled on the WAN-facing interface. Proxy ARP is disabled on all interfaces.

---

## Lab Environment

| Property | Value |
|---|---|
| Simulation platform | Cisco Packet Tracer |
| IOS version | 15.4 (R1), 15.0 (SW1) |
| Known platform limitations | See docs/lessons-learned.md |
