# homelab-cisco-network-lab
# Cisco Enterprise Network Lab

## Overview
Building a segmented enterprise network using physical Cisco hardware as part of CCNA study and homelab projects.

## Hardware
- Cisco Catalyst 3560G (Layer 3 core switch)
- 2x Cisco Catalyst 2960-24PC-L (access switches)
- 2x Cisco 1941 ISR routers
- Dell OptiPlex 7060 running Proxmox VE

## Basic Device Configuration 

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

## Spanning Tree Protocol — Closing the Loop and a Live Failover Test

With VLANs and trunking already in place across all three switches, ran a new physical cable directly between Access-SW1 and Access-SW2, deliberately closing a loop in the topology (Core-Switch ↔ SW2 ↔ SW1, plus a direct SW1↔SW2 link) so Spanning Tree Protocol would have an actual redundant path to manage.

**Configuration applied to the new link (both ends):**

```
configure terminal
interface fa0/X
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 1,10,20,30,99
 switchport nonegotiate
end
write memory
```

The new port on Access-SW2 (Fa0/8) required a `no shutdown` first — it had been administratively disabled during an earlier hardening pass on unused ports.

**Predicting before verifying:**

Before checking anything, predicted the root bridge (Core-Switch, based on its known MAC address from earlier CDP output) and expected the newest, most redundant link — the new direct SW1↔SW2 connection — to end up as the blocked path, since the original Core→SW2→SW1 chain was the pre-existing, shorter route to root.

**Verification — matched the prediction exactly:**

```
show spanning-tree vlan 1
```

Confirmed Core-Switch as root bridge (lowest bridge ID). On Access-SW1: the original link (Fa0/1, toward SW2) showed role **Root / FWD**, while the new direct link (Fa0/3, toward SW2) showed role **Altn / BLK** — correctly identified and blocked as the redundant path.

**Live failover test:**

Unplugged the active link (SW1's Fa0/1). SSH session briefly dropped (`Connection reset`) as the management path was riding on that link. Immediately re-ran `show spanning-tree vlan 1` on reconnect — Fa0/3 had transitioned from **Altn/BLK to Root/FWD**, taking over as the active path automatically. Reconnected the original cable afterward and confirmed the topology reconverged back to its original state.

**What this demonstrated concretely:** a real, physical link failure, and the network continued operating without manual intervention — the previously-blocked redundant port activated automatically, and even the SSH session used to observe it recovered through the new active path in real time.

**Current state:** physical loop safely managed by STP across all three switches, root bridge and port roles confirmed and tested against a real failure, not just configured. Native VLAN and trunk hardening from the previous phase remain intact across the new link. EtherChannel comes next, bundling the busiest link in this topology into one logical connection.

## EtherChannel — Bundling Core-Switch ↔ SW2, and a Config-vs-Operational-State Mismatch

With STP verified and the physical loop confirmed safe, bundled a second physical link between Core-Switch and Access-SW2 into a single logical EtherChannel using LACP, completing the Layer 2 build.

**Configuration applied to both member ports (Core-Switch Gi0/1 + Gi0/3, matched on Access-SW2 Fa0/4 + Fa0/9):**

```
configure terminal
interface g0/1
 channel-group 1 mode active
interface g0/3
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 99
 switchport trunk allowed vlan 1,10,20,30,99
 switchport mode trunk
 switchport nonegotiate
 channel-group 1 mode active
interface port-channel 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 1,10,20,30,99
 switchport nonegotiate
end
write memory
```

**Real troubleshooting encountered:**

After configuring both member ports identically on both switches, `show etherchannel summary` showed one port bundled (`P`) and the second suspended (`s`) — despite `show running-config` on the suspended port displaying the exact same trunk configuration as its working counterpart.

**Root cause:** `show interfaces <port> etherchannel` returned the actual reason directly: *"trunk mode of Gi0/3 is dynamic, Po1 is trunk"* — meaning the port's configured state (`switchport mode trunk`, sitting correctly in the running-config) had not fully propagated to the port's live operational state. IOS was still treating the interface as running dynamic trunk negotiation at the hardware level, a mismatch between what the config said and what the interface was actually doing.

**Resolution:** bounced the interface (`shutdown` / `no shutdown`) to force it to fully re-apply its configuration and re-negotiate from a clean state. Confirmed via `show etherchannel summary` immediately after — both ports showed `(P)`, fully bundled.

**Lesson documented for future reference:** a port's running-config and its actual operational state can disagree, particularly around trunk mode negotiation. `show run` alone isn't sufficient to diagnose an EtherChannel bundling failure — `show interfaces <port> etherchannel` gives the actual, specific suspension reason, and a targeted interface bounce is often the fix when configuration appears correct but isn't reflected in real-time behavior.

**Verification:**
```
show etherchannel summary
```
Both Core-Switch (Gi0/1, Gi0/3) and Access-SW2 (Fa0/4, Fa0/9) show `Po1(SU)` with both member ports flagged `(P)`.

**Confirmed STP now treats the bundle as a single logical link:**
```
show spanning-tree vlan 1
```
Port-channel1 appears as one interface (not two), with a lower path cost (12, versus 19 for a single physical link) — a direct, measurable demonstration of why link aggregation improves a switched topology, not just for redundancy but for STP's own path selection.

## Inter-VLAN Routing — SVIs on Core-Switch

With the Layer 2 build complete (VLANs, trunking, STP, EtherChannel), enabled Layer 3 routing on Core-Switch and created SVIs to allow devices in different VLANs to communicate with each other, moving from a purely switched topology into the start of the Layer 3 build.

**Configuration applied on Core-Switch:**

```
configure terminal
ip routing
interface vlan 10
 ip address 10.10.10.1 255.255.255.0
 no shutdown
interface vlan 20
 ip address 10.10.20.1 255.255.255.0
 no shutdown
interface vlan 30
 ip address 10.10.30.1 255.255.255.0
 no shutdown
end
write memory
```

`ip routing` turns the 3560G into a Layer 3-capable device in addition to its existing Layer 2 switching role — the same physical switch now makes routing decisions between VLANs while continuing to switch traffic within them.

**Verification:**

```
show ip interface brief
show ip route
```

All three SVIs came up `up/up`, and each VLAN's network appeared as a new directly-connected (`C`) route in Core-Switch's routing table — no static routes needed for this step, since each VLAN's network is now directly attached to the switch itself via its SVI.

**Real-world attempt at end-to-end validation, and a genuine lesson (see full write-up [here](https://github.com/Shbucket/homelab-cisco-network-lab/blob/main/Inter-VLAN%20Routing%20Test%20%E2%80%94%20a%20Single-NIC%20Proxmox%20Lesson) ): a single-NIC Proxmox host cannot have one VM's VLAN membership changed independently of the host's own management network via a simple access-port move — doing so takes down the entire host, not just the target VM.** Given that risk, full end-to-end inter-VLAN connectivity (a real device in one VLAN successfully routing to a real device in another) was validated in Packet Tracer instead, confirming the routing concept without risking the live homelab's stability.

## OSPF — Core-Switch, Router1, Router2, and a Three-Layer Troubleshooting Saga

Moved both routers off Access-SW2 and cabled them directly into Core-Switch, built a dedicated transit VLAN, and configured single-area OSPF (Area 0) across all three Layer 3 devices — completing the start of the Layer 3 build alongside the SVIs from the previous milestone.

**Configuration:**

```
! Core-Switch
configure terminal
vlan 100
 name TRANSIT
exit
interface g0/2
 switchport mode access
 switchport access vlan 100
 description CONNECTION-TO-ROUTER1
interface g0/4
 switchport mode access
 switchport access vlan 100
 description CONNECTION-TO-ROUTER2
interface vlan 100
 ip address 10.0.100.1 255.255.255.0
 no shutdown
exit
router ospf 1
 router-id 1.1.1.1
 network 10.0.100.0 0.0.0.255 area 0
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.30.0 0.0.0.255 area 0
```

```
! Router1 / Router2 (mirrored addressing)
interface loopback 0
 ip address 192.168.101.1 255.255.255.255   ! .102.1 on Router2
interface g0/1
 ip address 10.0.100.2 255.255.255.0        ! .100.3 on Router2
 no shutdown
router ospf 1
 router-id 2.2.2.2                          ! 3.3.3.3 on Router2
 network 10.0.100.0 0.0.0.255 area 0
 network 192.168.101.1 0.0.0.0 area 0       ! .102.1 on Router2
```

**Real troubleshooting — three layers deep, worth documenting in full since each layer masked the next:**

**Layer 1 — configuration applied to the wrong physical ports.** Initially configured VLAN 100 and interface descriptions on Gi0/5 and Gi0/6, but the routers were physically cabled into Gi0/2 and Gi0/4. `show ip interface brief` showed the configured ports as down (nothing plugged in) and the actual cabled ports still on their old VLAN — config and physical reality simply didn't match. Fixed by moving the VLAN 100 configuration onto the correct physical ports.

**Layer 2 — a leftover test link created a false positive.** After fixing the port assignment, pings from Core-Switch to both routers still failed. Router-to-router pings, however, succeeded — which turned out to be misleading. A direct physical link between Router1 and Router2, left over from an earlier standalone static routing exercise, was still connected and carrying that traffic, masking the fact that the actual Core-Switch transit path had never been tested at all. Removing that leftover link (as it should have been removed once that earlier exercise was done) caused router-to-router connectivity to fail too, confirming the transit-via-Core-Switch path was the real, still-unverified path.

**Layer 3 — the actual root cause: an unplugged cable.** With the misleading link removed, `show ip interface brief` on Router1 revealed `GigabitEthernet0/1` sitting at `down/down` — the transit cable to Core-Switch was never actually seated. All configuration on both ends had been correct the entire time. Reseating the physical cable resolved it immediately.

**Verification, once the physical link was actually in place:**

```
show ip ospf neighbor
```
Both routers show `FULL` adjacency (Router1 as BDR, Router2 as DR).

```
show ip route ospf
```
Both loopback addresses appear as OSPF-learned (`O`) routes via the transit network.

**A DR/BDR election detail worth noting:** despite being the central device in the topology, Core-Switch lost the DR/BDR election on this segment (`DROTHER`) — all three devices shared the default OSPF priority of 1, so the election came down purely to highest Router ID as the tiebreaker (Router2's `3.3.3.3` won DR, Router1's `2.2.2.2` took BDR, Core-Switch's `1.1.1.1` lost on both counts). A good, concrete reminder that OSPF's DR/BDR election is decided entirely by priority and Router ID — never by a device's actual role or importance in the topology.

**Current state:** single-area OSPF running cleanly across Core-Switch and both routers, full adjacencies confirmed, loopbacks learned dynamically rather than requiring static routes. Next: ACLs for inter-VLAN traffic segmentation.

## ACLs — Inter-VLAN Traffic Segmentation

With OSPF and inter-VLAN routing complete, applied an extended ACL on Core-Switch to deliberately restrict traffic between VLAN 10 (WORKSTATIONS) and VLAN 20 (SERVERS), while leaving VLAN 30 (MGMT) fully reachable — real, targeted segmentation rather than an all-or-nothing block.

**Configuration:**

```
ip access-list extended BLOCK-V10-TO-V20
 deny ip 10.10.10.0 0.0.0.255 10.10.20.0 0.0.0.255
 permit ip any any
exit
interface vlan 10
 ip access-group BLOCK-V10-TO-V20 in
```

**Test setup:** temporarily brought up an unused access port on Access-SW2 (Fa0/10), assigned it to VLAN 10, and connected a real test PC (`10.10.10.50/24`, gateway `10.10.10.1`) to validate the ACL against actual traffic rather than a simulated topology.

**Verification — three-part test, all results matched expectations exactly:**

- `ping 10.10.20.1` (VLAN 20 gateway) — **failed**, with `Reply from 10.10.10.1: Destination net unreachable` — the precise signature of a local ACL rejection at the ingress interface, not a routing or reachability failure elsewhere in the network
- `ping 10.10.30.1` (VLAN 30 gateway) — **succeeded**, confirming the ACL is targeted specifically at the VLAN 10 → VLAN 20 path, not blocking VLAN 10 broadly
- `ping 10.10.10.1` (own gateway) — **succeeded** cleanly, confirming same-VLAN traffic is entirely unaffected by an inter-VLAN ACL, as expected

**Hit counter confirmation:**

```
show access-lists
```

```
Extended IP access list BLOCK-V10-TO-V20
    10 deny ip 10.10.10.0 0.0.0.255 10.10.20.0 0.0.0.255 (8 matches)
    20 permit ip any any (570 matches)
```

Deny and permit counters both incremented exactly as expected — the deny count reflecting the blocked test traffic, the permit count reflecting all other traffic crossing the VLAN 10 SVI.

**Current state:** targeted inter-VLAN segmentation confirmed working end to end, on real hardware, with a real test device — not just configured, but validated against actual traffic and hit counters. Next: port security, DHCP snooping, and Dynamic ARP Inspection, followed by NTP/SNMP/Syslog across all five devices.

## Next Steps
- Port security, DHCP Snooping, and Dynamic ARP Inspection — access-edge hardening, labbed together since DAI depends on the DHCP snooping binding table
- NTP, SNMP, and Syslog configuration across all five devices — centralized time sync and monitoring, doubling as the data-source groundwork for a future network operations dashboard
- Netmiko automation script — remote config backup across all five devices, run against the completed real hardware build
- Full power-cycle sign-off — power down and restore the entire rack, verify every feature (VLANs, trunking, STP, EtherChannel, SVIs, OSPF, ACLs) survives intact
- Connect VMs (Windows Server / Windows 10) to physical VLANs

