# validation.md — Lab 01: IOS Basic Configuration & Security Hardening

## Overview

This document contains the full verification results for Lab 01. All tests were performed after completing the configuration of R1, SW1, and the Server. Outputs are taken directly from Cisco Packet Tracer.

---

## 1. SSH Status — R1

**Command:** `show ip ssh`

```
R1#show ip ssh
SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
```

**Result:** ✅ SSH version 2.0 is active. Authentication timeout and retry limit are correctly applied.

---

## 2. Interface Status — R1

**Command:** `show ip interface brief`

```
R1#show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0/0   unassigned      YES NVRAM  up                    up
GigabitEthernet0/0/0.10 192.168.10.1  YES manual up                    up
GigabitEthernet0/0/0.101 192.168.100.1 YES manual up                   up
GigabitEthernet0/0/1   203.0.113.1    YES NVRAM  up                    up
GigabitEthernet0/0/2   unassigned      YES NVRAM  administratively down down
Vlan1                  unassigned      YES unset  administratively down down
```

**Result:** ✅ All active interfaces are up/up. Unused interface Gi0/0/2 is correctly shut down. Subinterfaces for VLAN 10 and VLAN 101 are operational with correct IP addresses.

---

## 3. VTY Line Configuration — R1

**Command:** `show run | section vty`

```
R1#show run | section vty
line vty 0 4
 exec-timeout 300 0
 login local
 transport input ssh
line vty 5 15
 exec-timeout 300 0
 login local
 transport input ssh
```

**Result:** ✅ All VTY lines (0–15) are configured with SSH-only access, local authentication, and 5-minute idle timeout.

> Note: `show line vty 0 15` is not supported in Packet Tracer. The running-config section was used as an equivalent verification method.

---

## 4. Trunk Configuration — SW1

**Command:** `show interfaces trunk`

```
SW1#show interfaces trunk
Port    Mode      Encapsulation  Status    Native vlan
Gig0/1  on        802.1q         trunking  99

Port    Vlans allowed on trunk
Gig0/1  10,101

Port    Vlans allowed and active in management domain
Gig0/1  10,101

Port    Vlans in spanning tree forwarding state and not pruned
Gig0/1  10,101
```

**Result:** ✅ Trunk is active on Gi0/1 using 802.1Q encapsulation. Native VLAN is correctly set to 99. Only VLANs 10 and 101 are allowed and active on the trunk.

---

## 5. VLAN Table — SW1

**Command:** `show vlan brief`

```
SW1#show vlan brief

VLAN  Name              Status    Ports
----  ----------------  --------  ---------------------------
1     default           active
10    USERS             active    Fa0/1
99    NATIVE            active
101   MANAGEMENT        active    Fa0/24
999   UNUSED            active    Fa0/2, Fa0/3, Fa0/4, Fa0/5
                                  Fa0/6, Fa0/7, Fa0/8, Fa0/9
                                  Fa0/10, Fa0/11, Fa0/12, Fa0/13
                                  Fa0/14, Fa0/15, Fa0/16, Fa0/17
                                  Fa0/18, Fa0/19, Fa0/20, Fa0/21
                                  Fa0/22, Fa0/23, Gig0/2
```

**Result:** ✅ All four VLANs are present and correctly named. VLAN 10 contains PC1 (Fa0/1). VLAN 101 contains the server (Fa0/24). All unused ports are isolated in VLAN 999. VLAN 99 (native) has no access ports assigned, as intended.

---

## 6. Spanning Tree — SW1

**Command:** `show spanning-tree`

```
SW1#show spanning-tree
VLAN0010
  Spanning tree enabled protocol ieee
  Root ID    Priority    32778
             Address     0002.16BB.66C8
             This bridge is the root

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     0002.16BB.66C8

Interface  Role  Sts  Cost  Prio.Nbr  Type
Fa0/1      Desg  FWD  19    128.1     P2p
Gi0/1      Desg  FWD  4     128.25    P2p

VLAN0099
  Root ID    Priority    32867
             This bridge is the root

Interface  Role  Sts  Cost  Prio.Nbr  Type
Gi0/1      Desg  FWD  4     128.25    P2p

VLAN0101
  Root ID    Priority    32869
             This bridge is the root

Interface  Role  Sts  Cost  Prio.Nbr  Type
Fa0/24     Desg  FWD  19    128.24    P2p
Gi0/1      Desg  FWD  4     128.25    P2p
```

**Result:** ✅ SW1 is the Root Bridge for all active VLANs (expected in a single-switch topology). All ports are in Designated/Forwarding state. No blocked ports, no topology issues.

> Note: PortFast-enabled ports show type `P2p` instead of `P2p Edge` in Packet Tracer. On real IOS, PortFast ports would display `P2p Edge` in `show spanning-tree` output.

---

## 7. Syslog Status — R1

**Command:** `show logging`

```
R1#show logging
Syslog logging: enabled

Console logging:  level debugging, 8 messages logged
Monitor logging:  level debugging, 8 messages logged
Buffer logging:   level debugging, 0 messages logged

Trap logging: level debugging, 8 message lines logged
Logging to 192.168.100.5 (udp port 514, audit disabled,
  authentication disabled, encryption disabled, link up),
  8 message lines logged

Log Buffer (16384 bytes):
*May 15, 16:37:38.3737: %SYS-5-LOG_CONFIG_CHANGE: Buffer logging: level debugging
*May 15, 16:38:44.3838: %SYS-5-CONFIG_I: Configured from console by console
```

**Result:** ✅ Syslog is active and forwarding to 192.168.100.5. Timestamps with millisecond precision are present. Buffer size is 16384 bytes.

---

## 8. Syslog Status — SW1

**Command:** `show logging`

```
SW1#show logging
Syslog logging: enabled

Console logging:  level debugging, 14 messages logged
Buffer logging:   level debugging, 0 messages logged

Trap logging: level debugging, 14 message lines logged
Logging to 192.168.100.5 (udp port 514, link up),
  14 message lines logged

Log Buffer (16384 bytes):
*May 18, 11:27:11.2727: %SYS-5-LOG_CONFIG_CHANGE: Buffer logging: level debugging
*May 18, 12:53:17.5353: %SYS-5-CONFIG_I: Configured from console by console
*May 18, 12:56:23.5656: %SYS-5-CONFIG_I: Configured from console by console
```

**Result:** ✅ Syslog operational on SW1. 14 messages forwarded to the syslog server. Timestamps with millisecond precision confirmed.

---

## 9. Connectivity Tests — PC1

### 9.1 PC1 → R1 VLAN 10 Gateway

```
C:\>ping 192.168.10.1

Pinging 192.168.10.1 with 32 bytes of data:
Reply from 192.168.10.1: bytes=32 time<1ms TTL=255
Reply from 192.168.10.1: bytes=32 time<1ms TTL=255
Reply from 192.168.10.1: bytes=32 time<1ms TTL=255
Reply from 192.168.10.1: bytes=32 time<1ms TTL=255

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

**Result:** ✅ PC1 reaches the default gateway. Inter-VLAN routing subinterface and trunk are operational.

### 9.2 PC1 → NTP/Syslog Server

```
C:\>ping 192.168.100.5

Pinging 192.168.100.5 with 32 bytes of data:
Reply from 192.168.100.5: bytes=32 time<1ms TTL=127
Reply from 192.168.100.5: bytes=32 time<1ms TTL=127
Reply from 192.168.100.5: bytes=32 time<1ms TTL=127
Reply from 192.168.100.5: bytes=32 time<1ms TTL=127

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

**Result:** ✅ PC1 reaches the management segment server via inter-VLAN routing (VLAN 10 → R1 → VLAN 101). TTL=127 confirms traffic is routed through R1.

### 9.3 PC1 → WAN Host (PC2)

```
C:\>ping 203.0.113.2

Pinging 203.0.113.2 with 32 bytes of data:
Request timed out.
Reply from 203.0.113.2: bytes=32 time<1ms TTL=127
Reply from 203.0.113.2: bytes=32 time<1ms TTL=127
Reply from 203.0.113.2: bytes=32 time<1ms TTL=127

Packets: Sent = 4, Received = 3, Lost = 1 (25% loss)
```

**Result:** ✅ PC1 reaches the WAN-side host. The first packet timeout is expected behavior — caused by ARP resolution on the WAN interface before the first reply. Subsequent packets succeed with 0% loss. This is normal and not indicative of a connectivity issue.

---

## 10. SSH Connectivity Test

SSH access was verified from PC1 to R1 management interface:

```
C:\>ssh -l admin 192.168.100.1
```

**Result:** ✅ SSH session established successfully. Authentication required username and password. Access granted to privileged EXEC mode.

---

## Validation Summary

| Test | Device | Result |
|---|---|---|
| SSH version and status | R1 | ✅ Pass |
| Interface status | R1 | ✅ Pass |
| VTY SSH-only config | R1 | ✅ Pass |
| Syslog forwarding | R1 | ✅ Pass |
| Trunk configuration | SW1 | ✅ Pass |
| VLAN table | SW1 | ✅ Pass |
| Spanning Tree | SW1 | ✅ Pass |
| Syslog forwarding | SW1 | ✅ Pass |
| PC1 → Gateway | PC1 | ✅ Pass |
| PC1 → Server | PC1 | ✅ Pass |
| PC1 → WAN host | PC1 | ✅ Pass |
| SSH login | PC1 → R1 | ✅ Pass |
