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

## Lessons Learned / Troubleshooting

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

## Next Steps
- VLAN configuration and 802.1Q trunking between core and access switches
- Inter-VLAN routing via SVIs on Core-Switch
- OSPF between routers and Core-Switch
- ACLs for traffic segmentation
- Connect VMs (Windows Server / Windows 10) to physical VLANs
