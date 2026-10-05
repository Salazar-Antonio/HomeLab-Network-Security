# Network Architecture

## The problem this solves

My room has WiFi but no Ethernet drop. The whole build starts from that constraint: a WiFi-to-Ethernet bridge feeds a real firewall, which feeds a managed switch, which distributes to four segmented VLANs. Every single packet passes through the firewall — that's the entire point of the design: one central point of control and visibility.

## Packet path

```
Internet → ISP router → WiFi → Bridge (client/repeater mode) → OPNsense WAN
→ OPNsense LAN → Managed switch (trunk port) → VLAN access ports → devices
```

Every device sends anything leaving its own subnet through OPNsense, which checks it against the firewall rules — allowed traffic goes out to WAN through the bridge; blocked traffic is silently dropped.

## VLAN layout

| VLAN | Name | Purpose |
|---|---|---|
| 1 | Management | Admin — firewall, switch, hypervisor host |
| 10 | Servers | Internal VMs — DNS, monitoring, jump box |
| 50 | DMZ | Internet-facing services — screened subnet |
| 99 | Lab | Isolated pentest range — Kali + Metasploitable |

A fifth logical range is reserved for WireGuard VPN peers (see [02-core-services.md](./02-core-services.md)).

## Firewall rule matrix (default-deny, top to bottom, first match wins)

| From | To | Result |
|---|---|---|
| Management | Servers | Allow — manage the VMs |
| Management | WAN | Allow — management needs internet |
| Management | Lab | Block — admin stays out of the sandbox |
| Servers | WAN | Allow — VMs need updates/internet |
| Servers | Management | Block — a compromised VM can't reach the firewall or hypervisor |
| Servers | Lab | Block — servers isolated from the pentest range |
| DMZ | Internal VLANs | Block — a compromised public service can't pivot inward |
| Lab | Anywhere | Block — fully isolated sandbox |
| Jump box | Internal VLANs | Block — the bastion is a dead end |
| Jump box | WAN | Block — the bastion can't reach the internet either |

## Hardware

- **WiFi-to-Ethernet bridge** — a small travel router running OpenWrt in client/repeater mode. Only serves the lab; other devices in the house stay on regular home WiFi.
- **Firewall appliance** — a small enterprise-grade box, wiped and running OPNsense. WAN port connects to the bridge; LAN port feeds the switch.
- **Managed switch** — 8 ports, supports 802.1Q VLAN tagging, which is what makes segmentation physically possible on shared cabling. A dumb switch floods every port; this one only sends traffic where it belongs.
- **Hypervisor host** — runs every VM in the lab, bridged to their respective VLANs. **The host itself lives on the Management VLAN only** — never on Lab. If a lab VM is compromised, it must not be able to reach Management or control the hypervisor running everything else. This single placement decision is the difference between a contained incident and total compromise.
- **UPS** — battery backup on USB to the hypervisor host, monitored for graceful shutdown on power loss (see [02-core-services.md](./02-core-services.md)).

## Switch configuration — 802.1Q tagging

When traffic leaves the firewall for the switch, it's tagged with the VLAN it belongs to (802.1Q, the industry-standard trunking method). The switch reads those tags and delivers accordingly. The port to the firewall is a **trunk** — it carries all VLANs simultaneously, tagged. Every other port is an **access** port belonging to one VLAN, passing untagged traffic.

**PVID (Port VLAN ID)** is the default VLAN for untagged traffic on a port. All ports default to PVID 1, so an untagged device lands on Management unless explicitly configured otherwise.

### The night this broke everything

After the DNS VM was installed, it had no internet. Root cause: the switch ports the firewall and hypervisor were actually plugged into weren't tagged for the Servers VLAN — because patch-panel port numbers had been confused with actual switch-port numbers. Once the real physical ports were identified and correctly tagged, internet came up immediately.

**The lesson:** physical labeling matters as much as logical config. A perfectly correct VLAN setup does nothing if it's applied to the wrong physical port.
