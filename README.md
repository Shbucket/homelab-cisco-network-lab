# homelab-cisco-network-lab
# Cisco Enterprise Network Lab

## Overview
Building a segmented enterprise network using physical Cisco hardware as part of CCNA study and homelab projects.

## Hardware
- Cisco Catalyst 3560G (Layer 3 core switch)
- 2x Cisco Catalyst 2960-24PC-L (access switches)
- 2x Cisco 1941 ISR routers
- Dell OptiPlex 7060 running Proxmox VE

## Phase 1: Basic Device Configuration (Complete)

Configured all five devices with:
- Hostnames, enable secrets, console/VTY passwords
- Password encryption
- Login banners
- Local user accounts (`username admin`) with `login local` for SSH
- SSH version 2 with generated RSA keys
- Management IP addresses on VLAN 1 (switches) / GigabitEthernet0/0 (routers)
- Telnet fallback where SSH legacy cipher support was limited

### Management IP Plan

| Device | Role | Management IP | IOS Version |
|---|---|---|---|
| Core-Switch | 3560G - Layer 3 core | 10.0.0.2 | 12.2(25)SED1 |
| Access-SW1 | 2960 - Access switch | 10.0.0.13 | 15.0(2)SE11 |
| Access-SW2 | 2960 - Access switch | 10.0.0.14 | 15.0(2)SE11 |
| Router1 | 1941 - ISR router | 10.0.0.15 | 15.0(1)M6 |
| Router2 | 1941 - ISR router | 10.0.0.16 | 15.7(3)M6 |

## Lessons Learned / Troubleshooting — Phase 1: Basic Configuration

### Legacy SSH Cipher Negotiation
Modern OpenSSH clients have deprecated older key exchange, host key, cipher,
and MAC algorithms by default for security reasons. Older Cisco IOS images
(2005-2017 era) only support these legacy algorithms. Connecting required
explicitly re-enabling them client-side:

\`\`\`
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oCiphers=+aes128-cbc -oMACs=+hmac-sha1 admin@<ip>
\`\`\`

(Core-Switch required `diffie-hellman-group1-sha1` instead of group14 due to
its older 2005 IOS image.)

### SSH Requires `login local`, Telnet Does Not
Configuring only `password <pw>` + `login` on VTY lines works for Telnet,
but SSH requires a local username database. Without `username <user> secret
<pw>` and `login local` on the VTY lines, SSH authentication fails with
"Permission denied" even with the correct password.

### Configuration Register Bug (Password Recovery Mode)
One router (Router2) was left at configuration register `0x2142` — the
setting used during password recovery to skip loading the saved
configuration on boot. This caused the router to silently revert to a
blank configuration on every reboot, despite `write memory` completing
successfully. Fixed by setting the register back to the default `0x2102`:

\`\`\`
configure terminal
config-register 0x2102
end
write memory
\`\`\`

Confirmed via:
\`\`\`
show version | include register
\`\`\`

### Missing `transport input ssh` on VTY Lines
After resolving the above issues, Router2 still rejected all SSH
connections immediately during the identification exchange
(`kex_exchange_identification: Connection closed by remote host`) —
before any algorithm negotiation occurred. `debug ip ssh` revealed:

\`\`\`
SSH: Could not get a vty line for incoming session
\`\`\`

The VTY lines did not have `ssh` included in their `transport input`
setting (platform-default behavior differed from the switches and other
router). Fixed with:

\`\`\`
configure terminal
line vty 0 4
transport input ssh
end
write memory
\`\`\`

### Persistence Verification
After all fixes, performed a full power cycle of the rack and confirmed
all five devices retained their configuration and remained accessible via
SSH (with Telnet fallback for Core-Switch due to unresolved cipher-only
limitations).

## SSH Config (Client-Side)

To simplify daily access, created an SSH config profile
(`~/.ssh/config`) with per-device legacy algorithm overrides:

\`\`\`
Host core-switch
    HostName 10.0.0.2
    User admin
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc
    MACs +hmac-sha1

Host access-sw1
    HostName 10.0.0.13
    User admin
    KexAlgorithms +diffie-hellman-group14-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc
    MACs +hmac-sha1

Host access-sw2
    HostName 10.0.0.14
    User admin
    KexAlgorithms +diffie-hellman-group14-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc
    MACs +hmac-sha1

Host router1
    HostName 10.0.0.15
    User admin
    KexAlgorithms +diffie-hellman-group14-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc
    MACs +hmac-sha1
Host router2
    HostName 10.0.0.16
    User admin
    KexAlgorithms +diffie-hellman-group14-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc
    MACs +hmac-sha1
\`\`\`


## VLANs — Database Creation Across All Three Switches

As part of the CCNA hardware capstone build, created the four VLANs that will structure the rest of the network segmentation work (trunking, inter-VLAN routing, ACLs) coming next.

**VLANs created:**

| VLAN | Name | Purpose |
|------|------|---------|
| 10 | WORKSTATIONS | End-user devices |
| 20 | SERVERS | Server infrastructure |
| 30 | MGMT | Network device management traffic |
| 99 | NATIVE | Dedicated native VLAN for trunk hardening (see note below) |

**Configuration applied identically across Core-Switch, Access-SW1, and Access-SW2:**

```
configure terminal
vlan 10
 name WORKSTATIONS
vlan 20
 name SERVERS
vlan 30
 name MGMT
vlan 99
 name NATIVE
end
write memory
```
## Lessons Learned / Troubleshooting — VLANs:
**Why all three switches, not just one:** VLANs are a shared, logical construct — every switch that might carry or receive traffic for a given VLAN needs that VLAN in its own local database, or it has no way to correctly handle traffic tagged for it. Verified via `show vlan brief` that all four VLANs are present identically on all three switches before moving forward.

**Why a dedicated native VLAN (99) instead of leaving it as the default (VLAN 1):** VLAN 1 is the default VLAN every port starts in, the default management VLAN, and the default native VLAN — all at once. Leaving the native VLAN as VLAN 1 means untagged trunk traffic shares a VLAN with a lot of default device traffic, which is a known, avoidable soft spot. Using a dedicated, otherwise-unused VLAN as the native VLAN isolates untagged trunk traffic from default management traffic — a hardening step that will be applied once trunking is configured in the next phase of this build.

**Current state:** all four VLANs exist identically on all three switches. No ports have been reassigned yet — every port remains on VLAN 1, unchanged. Trunking, native VLAN assignment, and port reassignment come next.

## Trunking — Core-Switch ↔ SW2 ↔ SW1, with a Real Mid-Config Outage

With VLANs already created identically across all three switches, converted both inter-switch uplinks to 802.1Q trunks with a hardened native VLAN, carrying all four VLANs across the existing physical topology (Core-Switch ↔ SW2 ↔ SW1).

**Configuration applied to both trunk links:**

```
configure terminal
interface <uplink port>
 switchport trunk encapsulation dot1q   ! 3560G only — 2960s are dot1q-only
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 1,10,20,30,99
 switchport nonegotiate
end
write memory
```

Applied to:
- Core-Switch Gi0/1 ↔ Access-SW2 Fa0/4
- Access-SW2 Fa0/5 ↔ Access-SW1 Fa0/1

VLAN 1 was deliberately kept in the allowed list on both trunks so management/SSH traffic continues to flow across the newly-trunked links rather than being cut off.

## Lessons Learned / Troubleshooting — Trunking :

Configured SW2's two uplink ports (both sides of the Core-Switch and SW1 links) first, in a single session. Immediately after, lost SSH reachability to Core-Switch entirely — `ping 10.0.0.2` returned `Destination host unreachable` from the local gateway rather than a timeout, indicating no route existed to reach it at all.

**Root cause:** SW2's side of the Core-Switch link was already converted to trunk mode, but Core-Switch's matching port (Gi0/1) was still in its old access-mode configuration. A trunk port negotiating against a non-trunk port on the same link is a real mode mismatch, and it broke the path for management traffic (including VLAN 1) across that link entirely — a live demonstration of why both ends of a trunk link need to be reconfigured together, not one side at a time with a gap in between.

**Resolution:** Since the network path itself was down, SSH was not an option to fix the far end. Connected to Core-Switch directly via console cable (diagnosing a separate PuTTY serial connection issue along the way — confirming the correct COM port in Device Manager first), and converted Gi0/1 to match SW2's trunk configuration. Once both ends agreed on trunk mode, SSH reachability to Core-Switch was immediately restored.

**Lesson documented for future reference:** when converting a live uplink to a trunk, either do both ends within the same short window, or have console access ready before starting — a temporary mismatch between two ends of a trunk link can cut off the very management path being used to fix it.

**Verification — confirmed on all three switches:**
```
show interfaces trunk
```
All three show mode `on`, encapsulation `802.1q`, native VLAN `99`, and all four VLANs (1, 10, 20, 30, 99) allowed and active.

**Current state:** Core-Switch, Access-SW2, and Access-SW1 are fully trunked end to end across the existing topology, native VLAN hardened, management traffic confirmed flowing. No physical loop exists yet (topology is a straight line, not a triangle) — Spanning Tree Protocol and a new SW1↔SW2 direct link come next.

## Next Steps
- Close the physical loop (new SW1↔SW2 link) and configure Spanning Tree Protocol — predict port roles, verify root election, test failover
- EtherChannel — bundle a second link between two switches into one logical trunk
- Inter-VLAN routing via SVIs on Core-Switch
- OSPF between routers and Core-Switch
- ACLs for traffic segmentation
- Connect VMs (Windows Server / Windows 10) to physical VLANs
