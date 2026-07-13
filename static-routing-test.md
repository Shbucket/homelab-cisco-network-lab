## Static Routing — Isolated Test Topology

As part of learning static routing (CCNA), built a standalone, isolated test topology on real hardware to validate the concept end-to-end without touching the production 10.0.0.0/24 network.

**Topology:**

PC1 (172.16.1.10/24) → Router1 (g0/0: 172.16.1.1) → transit link (172.16.100.0/30) → Router2 (g0/0: 172.16.2.1) → PC2 — a disposable Alpine Linux VM (172.16.2.10/24) on Proxmox

**Configuration:**

- Router1 and Router2's existing `g0/0` interfaces (already on the production network) each received a **secondary IP address** to host the new isolated test subnet, avoiding any new cabling on that side.
- A new physical link was added between Router1 `g0/1` and Router2 `g0/1` for the transit network.
- Static routes configured on each router pointing at the other side's LAN via the transit link:
  - Router1: `ip route 172.16.2.0 255.255.255.0 172.16.100.2`
  - Router2: `ip route 172.16.1.0 255.255.255.0 172.16.100.1`

**Result:** Full end-to-end connectivity confirmed — PC1 successfully pings PC2 across both routers. Validated by removing the static route on Router1, confirming the ping failed, then re-adding the route and confirming connectivity was restored.

**Real troubleshooting encountered:**

Router1 required explicit legacy SSH negotiation flags to connect from a modern OpenSSH client — legacy KEX algorithm, legacy host key type, legacy cipher, and legacy MAC, all needing to be explicitly permitted:
