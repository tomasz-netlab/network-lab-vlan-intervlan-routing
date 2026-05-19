# Lessons Learned — Lab 01: IOS Basic Configuration & Security Hardening

## Overview

This document reflects on what was built, what required additional understanding during the process, and what would be approached differently in a future iteration. It is based on the actual implementation documented in `docs/implementation.md`, the verification results in `docs/validation.md`, and the architectural decisions recorded in `docs/topology.md`.

---

## What this lab actually covered

The lab combined five areas implemented together as a single coherent hardening scenario:

- VLAN segmentation and inter-VLAN routing via Router-on-a-Stick
- SSH-only remote access with explicit Telnet blocking on all VTY lines
- IOS device hardening — passwords, banners, timeouts, brute-force protection, unused service disabling
- STP configuration with PortFast and BPDUGuard on access ports
- Centralized Syslog and NTP synchronization across all devices

The value of this lab was not in any single command, but in understanding how these controls work together as a layered security posture on a managed network device.

---

## Architectural decisions that required deliberate reasoning

### Management segment on /29 — not a larger subnet

The management VLAN 101 uses a 192.168.100.0/29 subnet (6 usable hosts). The segment contains exactly three devices: R1, SW1, and the NTP/Syslog server. A larger subnet was considered and rejected — it would provide no operational benefit and would unnecessarily expand the attack surface of the management plane. Sizing a subnet to its actual purpose is a discipline worth applying consistently.

### VLAN 101 instead of VLAN 100

VLAN 100 is a commonly used and predictable default in many network designs. VLAN 101 was chosen as a minor measure against automated scanners and attackers that target predictable VLAN numbers. This is a small decision, but it reflects the principle that avoiding defaults has value even when the security gain is modest.

### Native VLAN 99 — dedicated and carrying no traffic

IEEE 802.1Q trunks require a native VLAN for untagged frames. Using VLAN 1 (the default) is a known risk: VLAN hopping attacks can exploit the native VLAN to send untagged frames that land on VLAN 1 and from there potentially reach the management or user plane. VLAN 99 (NATIVE) was created solely for this role — no IP addressing, no access ports, no user or management traffic. Its only function is to absorb any untagged frames that appear on the trunk, neutralizing the attack vector.

### VLAN 999 for unused ports — least privilege at Layer 2

All inactive switch ports are assigned to VLAN 999 and shut down. The shutdown alone is not sufficient: if a port is accidentally re-enabled without the VLAN assignment, it would inherit VLAN 1 and gain access to the default broadcast domain. The explicit assignment to VLAN 999 ensures that even an accidentally re-enabled port has no path to any operational VLAN.

### 203.0.113.0/30 for the WAN simulation link

The WAN-side subnet uses addresses from the 203.0.113.0/24 range, which is IANA-reserved TEST-NET-3 — a documentation prefix that will never appear as a real routable prefix on the internet. Using it makes the lab topology safe to document and publish publicly without risk of accidentally referencing real infrastructure.

---

## Things that required deeper understanding during this lab

### Why SSH requires both hostname and domain name

It was not initially obvious why `crypto key generate rsa` depends on a domain name being set. The reason: the RSA key identifier in IOS is constructed from `hostname.domain-name`. Without both, the key cannot be named and the generation fails. This is a hard dependency, not a best practice — the command simply will not execute without it.

### `enable secret` vs `enable password` — and what Type 7 actually means

`enable secret` stores the password as an MD5 hash (Type 5) and always takes precedence over `enable password` if both are configured. `enable password` stores in plaintext unless `service password-encryption` is applied, which only adds Type 7 (Vigenère cipher) — a weak, reversible encoding, not real encryption. The practical rule: always use `enable secret`, never `enable password`, and understand that `service password-encryption` is a basic protection against casual config exposure, not a cryptographic control.

On real IOS (not Packet Tracer), the stronger option is:

```
enable algorithm-type scrypt secret <password>
```

This produces a Type 9 hash (scrypt), significantly more resistant to brute-force attacks. Packet Tracer does not support this.

### What `transport input ssh` actually blocks

The command `transport input ssh` on VTY lines does not just "enable SSH" — it explicitly restricts the line to SSH only and rejects any Telnet connection attempt. The security reasoning: Telnet transmits everything in plaintext, including credentials. A single packet capture on the path between the management workstation and the router exposes the full session. This is why Telnet is unacceptable on any managed device regardless of whether the network is considered "internal."

### The `no ip domain-lookup` decision

This command prevents IOS from interpreting mistyped commands as hostnames and attempting DNS resolution. Without it, a typo at the CLI causes a multi-second hang while the router waits for a DNS timeout. It has no direct security function, but it is included in every hardening baseline because the delay it causes can interfere with operational work and obscure real errors.

### Banner types and the legal distinction

Three banner types exist in IOS: `banner motd`, `banner login`, and `banner exec`. They appear at different points in the connection sequence. The important distinction for hardening: `banner motd` is displayed before authentication, making it the legally relevant one. Its purpose is to warn that unauthorized access is prohibited — not to welcome users. Banners that say "Welcome" or "Authorized users only" can create legal ambiguity. The correct approach is an explicit warning, which was applied in this lab.

### `no cdp enable` on the WAN interface — and why not globally

CDP advertises device model, IOS version, IP addresses, and interface capabilities in plaintext to directly connected neighbors. On an external-facing interface, this is free reconnaissance data for anyone on the WAN segment. Disabling CDP globally (`no cdp run`) would also disable it on internal interfaces where it may be useful for management. The targeted approach — disabling only on the WAN interface — is the correct balance.

### `no ip proxy-arp` — what it masks and why it matters

Proxy ARP causes the router to respond to ARP requests on behalf of hosts in other subnets. This can mask misconfigured hosts by making them appear to communicate normally when they should not. Disabling it enforces correct host configuration and reduces exposure to ARP-based attacks. Applied on all subinterfaces in this lab.

### NTP and why accurate time is not optional

`service timestamps log datetime msec` adds a timestamp with millisecond precision to every log entry. Without NTP synchronization, those timestamps are meaningless for cross-device correlation — if R1 and SW1 have different clocks, a sequence of events spanning both devices cannot be reconstructed reliably. Time synchronization is a prerequisite for any meaningful log analysis or incident response.

### `clock summer-time` — a Packet Tracer limitation with real-world implications

The command `clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00` is required in Poland to handle the transition between CET (UTC+1) and CEST (UTC+2). Packet Tracer does not support this command. In a real deployment, omitting it means log timestamps will be offset by one hour during summer months — a subtle but operationally significant issue.

### STP — PortFast applies only to non-trunk ports

`spanning-tree portfast default` enables PortFast globally, but IOS automatically excludes trunk ports from this setting. The trunk to R1 (Gi0/1) is correctly excluded from both PortFast and BPDUGuard without requiring manual per-interface configuration.

### BPDUGuard — what it actually protects against

BPDUGuard shuts down a PortFast-enabled port the moment a BPDU is received on it. BPDUs indicate a switch is connected — either accidentally or deliberately. An attacker sending crafted BPDUs can manipulate STP topology to become the root bridge and intercept traffic. The `err-disabled` state that BPDUGuard triggers requires manual intervention to recover, which is intentional — it forces awareness of the event.

---

## Packet Tracer limitations discovered during implementation

Several hardening commands were intended but rejected by Packet Tracer:

- `no ip source-route` — not accepted by the IOS version in PT
- `no service tcp-small-servers` / `no service udp-small-servers` — not present in PT
- `no service finger`, `no ip http server`, `no service bootp` — not supported in PT
- `security passwords min-length`, `login block-for`, `login on-failure log` — not supported on Cisco 2960 in PT
- `clock summer-time CEST recurring` — not supported in PT
- `logging source-interface`, `service sequence-numbers` — not supported in PT
- `username admin privilege 15 algorithm-type scrypt secret` — Type 9 hashing not available in PT

Each of these is documented in `implementation.md` as intended hardening posture for real IOS deployments.

---

## What worked without issues

- SSH connectivity from PC1 to R1 via the management subinterface (VLAN 101)
- All VTY lines correctly restricted to SSH; Telnet rejected
- Inter-VLAN routing operational across VLAN 10 and VLAN 101 via Router-on-a-Stick
- Trunk configuration correct — only VLANs 10 and 101 allowed, native VLAN 99 with no user traffic
- STP stable — SW1 is root bridge for all VLANs, all ports in Forwarding state
- Syslog forwarding confirmed on both R1 and SW1, timestamps with millisecond precision present
- NTP server operational; time synchronized across devices
- First-packet timeout on WAN ping identified correctly as ARP resolution delay, not a fault

---

## What would be done differently

- Add `ip ssh source-interface` to restrict SSH to originate only from the management interface — not implemented but standard in hardening guides
- Apply `exec-timeout` to a production-realistic value (5–10 minutes) from the start rather than treating it as a lab convenience setting
- Test syslog reception explicitly on the server side, not just confirm forwarding from the device side
- Use a named ACL if access-class filtering on VTY lines were added in a future iteration — named ACLs are easier to read and modify than numbered ones

---

## Broader observations

This lab demonstrates that basic IOS hardening is not a checklist of commands — it is a set of decisions, each with a specific threat model behind it. The difference between a secure and an insecure configuration of the same device is often a handful of lines, but understanding why those lines matter is what allows the same reasoning to be applied in an audit, a review, or a real deployment.

The gap between what Packet Tracer supports and what real IOS requires was a consistent theme. Documenting unsupported commands alongside their security rationale ensures that the lab has real-world value beyond the simulation environment.

---

## Possible extensions

- Add an ACL restricting SSH access by source IP, applied with `access-class` on VTY lines
- Configure AAA with a local user database using `aaa new-model`, replacing line-level `login local`
- Add `ip ssh source-interface` to restrict management plane traffic to the management segment
- Extend the topology with a second switch to make STP port roles more meaningful (root port, designated port, blocked port)
- Test syslog parsing on the server side and verify that events from both devices are correlated correctly by timestamp
