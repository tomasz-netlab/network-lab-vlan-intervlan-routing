# implementation.md — Lab 01: IOS Basic Configuration & Security Hardening

## Overview

This document describes the full configuration process for Lab 01. Each section explains not only what was configured, but why — following the principle that understanding the reasoning behind a configuration is more valuable than memorizing the commands themselves.

The lab was implemented in Cisco Packet Tracer using IOS 15.4 (R1) and IOS 15.0 (SW1). Known platform limitations are documented separately in `docs/lessons-learned.md`.

---

## Part 1 — Router R1 Base Configuration

### 1.1 Hostname and Domain Name

```
hostname R1
ip domain-name lab.local.ccna
```

Setting a hostname is a fundamental operational practice — it identifies the device in logs, SSH banners, and CLI prompts. The domain name is required for RSA key generation and SSH operation. Without it, `crypto key generate rsa` will fail.

`no ip domain-lookup` was also applied to prevent the router from attempting DNS resolution on mistyped commands, which would cause unnecessary delays in the CLI.

### 1.2 Security Banner

```
banner motd # WARNING: Authorized access only. Violators will be prosecuted. #
```

A legal warning banner is configured before any login prompt. This is both a security requirement in many organizations and a legal prerequisite in some jurisdictions — without a warning banner, unauthorized access may be harder to prosecute. The banner appears before authentication, ensuring visibility to any connecting party.

### 1.3 Privileged EXEC Password

```
enable secret PassWD789101112
```

`enable secret` uses MD5 hashing (Type 5) by default in IOS 15.4. This is the strongest algorithm available in Packet Tracer. On a real IOS image, the preferred approach is:

```
enable algorithm-type scrypt secret <password>
```

This would produce a Type 9 hash (scrypt), significantly more resistant to brute-force attacks. This limitation is documented in `lessons-learned.md`.

### 1.4 Local User Account

```
username admin secret Password9101112
```

A local user account `admin` is created for SSH and console authentication. On a real IOS image, the account would be configured with privilege level 15 and scrypt hashing:

```
username admin privilege 15 algorithm-type scrypt secret <password>
```

Privilege level 15 grants direct access to privileged EXEC mode upon login, which is appropriate for an emergency administrative account when AAA (RADIUS/TACACS+) is not in scope for this lab.

### 1.5 Password Policies

```
security passwords min-length 12
service password-encryption
login block-for 300 attempts 3 within 120
login on-failure log
```

- `security passwords min-length 12` enforces a minimum password length at the IOS level.
- `service password-encryption` applies Type 7 (Vigenère cipher) encryption to all plaintext passwords in the configuration. This is not cryptographically strong, but prevents casual shoulder-surfing of running configs.
- `login block-for` implements a basic brute-force lockout: after 3 failed login attempts within 120 seconds, all logins are blocked for 300 seconds (5 minutes).
- `login on-failure log` sends a syslog message on every failed login attempt, enabling detection of brute-force activity.

### 1.6 Console Line Configuration

```
line console 0
 login local
 exec-timeout 300 0
 logging synchronous
```

- `login local` requires authentication using the local user database.
- `exec-timeout 300 0` terminates an idle console session after 5 minutes (300 seconds, 0 milliseconds).
- `logging synchronous` prevents syslog messages from interrupting command input, keeping the CLI usable during active logging.

### 1.7 VTY Lines — SSH Only

```
line vty 0 15
 login local
 exec-timeout 300 0
 transport input ssh
```

All 16 VTY lines (0–15) are configured identically to ensure consistent behavior regardless of which line is allocated for an incoming SSH session. `transport input ssh` explicitly blocks Telnet — any Telnet connection attempt is rejected. This is a critical hardening step: Telnet transmits credentials in plaintext and must not be permitted on any managed device.

### 1.8 SSH Configuration

```
crypto key generate rsa
ip ssh version 2
ip ssh time-out 120
ip ssh authentication-retries 3
```

RSA keys were generated at 4096 bits (maximum supported in this PT build). SSH version 2 is enforced — SSHv1 has known vulnerabilities and is not acceptable in any production or lab hardening scenario.

`ip ssh time-out 120` sets the authentication window to 120 seconds. The lab task specified 60 seconds, but 120 seconds was chosen to allow more comfortable access during lab work. This is a deliberate deviation from the task specification, made with full awareness of the security trade-off.

`ip ssh authentication-retries 3` limits the number of SSH authentication attempts before the connection is dropped.

---

## Part 2 — Router R1 Interface Configuration

### 2.1 Subinterface Gi0/0/0.10 — VLAN 10 (Users)

```
interface GigabitEthernet0/0/0.10
 description VLAN10-USER-SEGMENT
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 no ip proxy-arp
 no shutdown
```

The subinterface implements Router-on-a-Stick inter-VLAN routing. `encapsulation dot1Q 10` tags outgoing frames with VLAN ID 10 and accepts only frames tagged with VLAN 10 from the trunk. Without this command, the subinterface will not pass any traffic.

`no ip proxy-arp` is applied on all subinterfaces. Proxy ARP causes the router to respond to ARP requests on behalf of hosts in other subnets, potentially masking misconfigured hosts and generating unnecessary load. Disabling it enforces correct host configuration and reduces exposure to ARP-based attacks.

### 2.2 Subinterface Gi0/0/0.101 — VLAN 101 (Management)

```
interface GigabitEthernet0/0/0.101
 description MANAGEMENT
 encapsulation dot1Q 101
 ip address 192.168.100.1 255.255.255.248
 no ip proxy-arp
 no shutdown
```

The management subinterface provides Layer 3 connectivity for the management segment (192.168.100.0/29). It serves as the default gateway for SW1 SVI and the NTP/Syslog server.

### 2.3 WAN Interface Gi0/0/1

```
interface GigabitEthernet0/0/1
 description WAN-UPLINK
 ip address 203.0.113.1 255.255.255.252
 no ip proxy-arp
 no cdp enable
 no shutdown
```

The WAN interface uses a /30 subnet, which is standard practice for point-to-point links — it minimizes wasted address space.

`no cdp enable` disables CDP on this interface. CDP is a Layer 2 discovery protocol that advertises device model, IOS version, IP addresses, and capabilities in plaintext. On an external-facing interface, this information would be visible to anyone on the WAN segment, providing free reconnaissance data to a potential attacker.

### 2.4 Unused Interface

```
interface GigabitEthernet0/0/2
 no ip proxy-arp
 shutdown
```

Unused interfaces are administratively shut down to prevent accidental activation.

---

## Part 3 — Router R1 Security Hardening

### 3.1 IOS Hardening Commands

```
no ip source-route
```

IP Source Routing is an IP header option that allows the sender to specify the path a packet must take through the network. This can be exploited to bypass ACLs and firewalls or to conduct man-in-the-middle attacks. There is no legitimate use case for IP source routing in modern networks. **Note: this command was not accepted by Packet Tracer — it is documented here as a required hardening step for real IOS deployments.**

The following hardening commands are also required in production but are not supported in Packet Tracer:

```
no service tcp-small-servers
no service udp-small-servers
no service finger
no ip http server
no ip https server
no service bootp
```

Each of these disables a legacy or unnecessary service that increases the attack surface of the device. They are included here as documentation of intended hardening posture, even though they could not be applied in this simulation environment.

### 3.2 NTP Configuration

```
ntp server 192.168.100.5
```

All devices synchronize time from the internal NTP server. Accurate time is essential for log correlation, certificate validity, and forensic analysis. Without synchronized time, it is impossible to reliably correlate events across multiple devices.

The following commands were intended but are not supported in Packet Tracer:

```
clock timezone CET 1
clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00
```

On real IOS, these commands set the local timezone (CET, UTC+1) and configure automatic daylight saving time transition (CEST, UTC+2) for the Polish timezone.

### 3.3 Syslog Configuration

```
service timestamps log datetime msec
service timestamps debug datetime msec
logging buffered 16384
logging trap debugging
logging host 192.168.100.5
```

- `service timestamps log datetime msec` adds a precise timestamp (date, time, milliseconds) to every log entry. This is critical for forensic analysis and incident response.
- `logging buffered 16384` stores log messages in a 16 KB RAM buffer for local review via `show logging`.
- `logging trap debugging` sets the severity level for messages sent to the syslog server to the maximum (level 7 — debugging). This ensures all events, including informational and diagnostic messages, are forwarded.
- `logging host 192.168.100.5` forwards all log messages to the centralized syslog server.

`logging source-interface` and `service sequence-numbers` were intended but are not supported in Packet Tracer.

---

## Part 4 — Switch SW1 Base Configuration

### 4.1 Hostname, Domain, SSH, Users, Passwords

SW1 was configured with the same base hardening posture as R1:

```
hostname SW1
ip domain-name lab.local.ccna
username admin secret Password9101112
enable secret PassWD789101112
service password-encryption
banner motd # WARNING: Authorized access only. Violators will be prosecuted. #
ip ssh version 2
ip ssh time-out 120
ip ssh authentication-retries 3
line console 0
 login local
 exec-timeout 300 0
 logging synchronous
line vty 0 15
 login local
 transport input ssh
 exec-timeout 300 0
```

Note: `security passwords min-length`, `login block-for`, and `login on-failure log` are not supported on the Cisco 2960 in Packet Tracer. These are documented in `lessons-learned.md`.

### 4.2 VLAN Configuration

```
vlan 10
 name USERS
vlan 99
 name NATIVE
vlan 101
 name MANAGEMENT
vlan 999
 name UNUSED
```

Four VLANs were created with explicit names. Naming VLANs is an operational best practice — it makes the purpose of each VLAN immediately clear in `show vlan brief` output and reduces the risk of misconfiguration.

### 4.3 Management SVI

```
interface Vlan101
 ip address 192.168.100.2 255.255.255.248
 no shutdown

ip default-gateway 192.168.100.1
```

The SVI for VLAN 101 provides Layer 3 reachability for SSH management access to SW1. The default gateway points to R1’s management subinterface. Since SW1 operates at Layer 2, it requires a default gateway for any traffic that needs to leave its local segment.

### 4.4 Trunk Port to R1

```
interface GigabitEthernet0/1
 description TRUNK-TO-R1
 switchport mode trunk
 switchport trunk allowed vlan 10,101
 switchport trunk native vlan 99
 switchport nonegotiate
 no shutdown
```

The trunk port is configured with explicit allowed VLANs (10 and 101 only). VLANs not explicitly listed are blocked on the trunk, limiting the blast radius of any misconfiguration or VLAN hopping attempt.

`switchport nonegotiate` disables DTP (Dynamic Trunking Protocol) negotiation. DTP is a Cisco proprietary protocol that automatically negotiates trunk formation between switches. Leaving DTP active on a port exposes it to VLAN hopping attacks using DTP exploitation. Disabling it enforces static configuration.

Native VLAN is set to VLAN 99 — a dedicated VLAN that carries no user or management traffic, mitigating VLAN hopping via native VLAN manipulation.

### 4.5 Access Port — PC1

```
interface FastEthernet0/1
 description PC1-ACCESS
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 no shutdown
```

### 4.6 Server Port

```
interface FastEthernet0/24
 description LINK-TO-SERVER
 switchport mode access
 switchport access vlan 101
 switchport nonegotiate
 no shutdown
```

The server is placed in VLAN 101 (management segment), giving it direct Layer 2 adjacency with SW1’s management SVI and Layer 3 reachability to R1’s management subinterface.

### 4.7 Unused Ports

```
interface range FastEthernet0/2-23, GigabitEthernet0/2
 description NOT IN USE
 switchport mode access
 switchport access vlan 999
 switchport nonegotiate
 shutdown
```

All unused ports are assigned to VLAN 999 and shut down. This eliminates any possibility of an unauthorized device gaining access to an operational VLAN by plugging into an unused port.

### 4.8 Spanning Tree Configuration

```
spanning-tree mode pvst
spanning-tree portfast default
spanning-tree portfast bpduguard default
```

PVST+ (Per-VLAN Spanning Tree Plus) is used, providing a separate STP instance per VLAN.

`spanning-tree portfast default` enables PortFast globally on all non-trunk ports. PortFast skips the STP listening and learning states (normally 30 seconds total) and immediately transitions the port to forwarding state. This is appropriate for access ports connected to end devices that do not participate in STP.

`spanning-tree portfast bpduguard default` enables BPDUGuard on all PortFast-enabled ports. If a BPDU is received on a PortFast port (indicating a switch has been connected), the port is immediately placed into `err-disabled` state. This protects against both accidental switch connections and deliberate STP topology manipulation attacks (e.g., BPDU spoofing to become the root bridge).

Since `portfast default` only applies to non-trunk ports, the trunk to R1 (Gi0/1) is automatically excluded from both PortFast and BPDUGuard — no per-interface override is required.

### 4.9 SW1 Syslog and NTP

```
service timestamps log datetime msec
service timestamps debug datetime msec
logging buffered 16384
logging trap debugging
logging host 192.168.100.5
ntp server 192.168.100.5
```

Identical syslog and NTP configuration as R1, ensuring consistent log format and time synchronization across all devices.

---

## Part 5 — Server Configuration

The Server-PT was configured with:

- IP address: 192.168.100.5 /29
- Default gateway: 192.168.100.1
- NTP service: enabled
- Syslog service: enabled

The server acts as the single source of truth for time synchronization and centralized log collection across the entire lab topology.
