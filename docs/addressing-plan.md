# Addressing Plan — Lab 01: VLAN Segmentation, Inter-VLAN Routing and IOS Hardening

## Overview

This document provides the complete addressing plan for Lab 01, covering IP addressing, VLAN assignments, subnet design rationale, and port mapping. It serves as a single reference for all Layer 2 and Layer 3 addressing decisions made in this lab.

---

## IP Addressing

| Device | Interface | IP Address | Subnet Mask | CIDR | Network | Role |
|---|---|---|---|---|---|---|
| R1 | Gi0/0/0.10 | 192.168.10.1 | 255.255.255.0 | /24 | 192.168.10.0/24 | VLAN 10 default gateway |
| R1 | Gi0/0/0.101 | 192.168.100.1 | 255.255.255.248 | /29 | 192.168.100.0/29 | Management segment gateway |
| R1 | Gi0/0/1 | 203.0.113.1 | 255.255.255.252 | /30 | 203.0.113.0/30 | WAN uplink (point-to-point) |
| SW1 | VLAN 101 SVI | 192.168.100.2 | 255.255.255.248 | /29 | 192.168.100.0/29 | Switch management access (SSH) |
| Server | NIC | 192.168.100.5 | 255.255.255.248 | /29 | 192.168.100.0/29 | NTP + Syslog server |
| PC1 | NIC | 192.168.10.10 | 255.255.255.0 | /24 | 192.168.10.0/24 | User endpoint |
| PC2 | NIC | 203.0.113.2 | 255.255.255.252 | /30 | 203.0.113.0/30 | WAN endpoint (Internet simulation) |

---

## Subnet Summary

| Network | CIDR | Mask | Usable Hosts | Purpose |
|---|---|---|---|---|
| 192.168.10.0 | /24 | 255.255.255.0 | 254 | VLAN 10 — User segment |
| 192.168.100.0 | /29 | 255.255.255.248 | 6 | VLAN 101 — Management segment |
| 203.0.113.0 | /30 | 255.255.255.252 | 2 | WAN point-to-point link |

### Subnet sizing rationale

**192.168.10.0/24** — User VLAN 10 uses a /24 to accommodate future endpoint growth without renumbering.

**192.168.100.0/29** — Management segment sized deliberately to 6 usable hosts. Current occupants: R1 (.1), SW1 (.2), Server (.5). A /29 provides exactly the headroom needed without unnecessarily expanding the management attack surface. Larger subnets (e.g. /24) would provide no operational benefit and would widen the broadcast domain for a segment that should remain as restricted as possible.

**203.0.113.0/30** — WAN point-to-point link uses a /30 (2 usable hosts), which is standard practice for serial or routed point-to-point connections. Prefix 203.0.113.0/24 belongs to IANA TEST-NET-3 (RFC 5737) — a documentation range that will never appear as a real routable prefix on the internet, making the lab topology safe to publish publicly.

---

## VLAN Assignment

| VLAN ID | Name | Purpose | IP Segment | Notes |
|---|---|---|---|---|
| 10 | USERS | End-user traffic | 192.168.10.0/24 | PC1 on Fa0/1 |
| 99 | NATIVE | Dedicated native VLAN on trunk | none | No IP, no access ports — VLAN hopping mitigation |
| 101 | MANAGEMENT | Administrative access to R1 and SW1 | 192.168.100.0/29 | Server on Fa0/24, SW1 SVI |
| 999 | UNUSED | Isolation VLAN for all inactive ports | none | All unused ports assigned here and shut down |

---

## Switch Port Mapping — SW1

| Port | Mode | VLAN | Connected Device | Notes |
|---|---|---|---|---|
| Fa0/1 | Access | VLAN 10 | PC1 | User endpoint |
| Fa0/24 | Access | VLAN 101 | Server | NTP + Syslog |
| Gi0/1 | Trunk | Native: 99, Allowed: 10, 101 | R1 Gi0/0/0 | 802.1Q, DTP disabled |
| Fa0/2–23, Gi0/2 | Access | VLAN 999 | — | Unused, administratively shut down |

---

## Default Gateway Summary

| Device | Default Gateway | Notes |
|---|---|---|
| PC1 | 192.168.10.1 | R1 subinterface Gi0/0/0.10 |
| SW1 | 192.168.100.1 | R1 subinterface Gi0/0/0.101 (ip default-gateway) |
| Server | 192.168.100.1 | R1 subinterface Gi0/0/0.101 |
| PC2 | — | Directly connected to R1 WAN interface |

---

## Management Services

| Service | Server Address | Protocol / Port | Clients |
|---|---|---|---|
| NTP | 192.168.100.5 | UDP 123 | R1, SW1 |
| Syslog | 192.168.100.5 | UDP 514 | R1, SW1 |
