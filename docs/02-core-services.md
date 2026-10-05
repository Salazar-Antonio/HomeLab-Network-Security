# Core Services — DNS, VPN, Monitoring, Power

## DNS — Pi-hole, DNSSEC & DNS over TLS

A centralized DNS server answers queries for every VLAN, giving full visibility into what every device is trying to resolve — it came up with over 80,000 domains on its blocklist and started logging immediately.

**DNSSEC** validates that DNS responses are digitally signed and untampered. **DNS over TLS** wraps queries in TLS, so the ISP can no longer see or spoof lookups — queries are now both signed and encrypted, configured against two public DoT resolvers.

### Problems hit

- **Install-time chicken-and-egg** — the DNS VM's install failed to auto-configure networking because its VLAN had no DHCP yet (nothing was on it to serve a lease). Fixed by manually setting a static IP during install, then resolved for good once the switch-port tagging issue (see architecture doc) was fixed.
- **Resolver pointed wrong** — pointing the firewall's resolver directly at the DNS server broke resolution entirely. Fixed by enabling query forwarding with system nameservers instead.
- **Stale DHCP lease** — after a bridge device reset, the PC could ping IPs but not browse. Fixed with a DHCP release/renew — a classic stale-lease symptom.
- **A single device not resolving a local hostname** — fixed by disabling IPv6 on that device's adapter and adding a host override.

## Remote access — WireGuard VPN

Runs directly on the firewall (not a separate VM) with several peers, each in their own dedicated VPN subnet, providing full access to every homelab service from anywhere.

**Tunnel mode is a real tradeoff, not a fixed setting.** Split tunnel routes only homelab traffic through the VPN, keeping regular browsing fast. Full tunnel routes everything through home, so every device also benefits from intrusion detection and encrypted DNS everywhere. Both are legitimate — the setting shifts based on whether speed or blanket protection matters more at the time. Switching modes is client-side only; no server-side NAT changes required.

### Problems hit

- **Handshake up, traffic dead.** The tunnel showed an active handshake, but every ping timed out. Two firewall rules were both required: one allowing the VPN port inbound at the WAN, and a *separate* rule allowing the VPN subnet outbound to anywhere. The firewall blocks by default — even an authenticated tunnel needs an explicit pass rule. This was the single best VPN-troubleshooting lesson of the whole build.
- **Double NAT.** The WiFi bridge sits between the ISP router and the firewall, so the VPN port had to be forwarded twice — once at the ISP router, once at the bridge. A direct Ethernet run between the two would eliminate this entirely (see "what's next" in the main README).
- **Dynamic DNS detour.** The original plan (a cron job on the firewall) failed because a required command-line tool wasn't available in that firmware version. Fell back to the bridge device's own built-in dynamic DNS feature, which auto-updates on its own.

## Monitoring & alerting

A dedicated monitoring VM runs a metrics collector that scrapes agents on the DNS server, the jump box, and the hypervisor itself. A dashboard tool visualizes it all — CPU, RAM, disk, and network per VM.

One VM (the Kali box) couldn't run the monitoring agent since it wasn't available in that distro's package repos, so monitoring was skipped for that one on purpose.

### Two separate alerting bugs, back to back

Email alerting on a "service down" condition didn't work at first because the SMTP config lines were commented out by default — a classic config-file trap. After fixing that, the alert still never fired: the query condition was set to trigger "above zero," which a value of exactly zero never satisfies. Fixed the condition logic, and alerts immediately went from pending to firing correctly — for two VMs that actually were down at the time. A separate step (setting a default notification policy) was also required, since an alert can fire internally without ever actually sending an email until that's configured.

**Lesson:** an alert "firing" internally and an alert actually notifying you are two different things — verify both independently.

## Power protection — UPS + NUT

A UPS on USB to the hypervisor host, monitored so the host shuts down gracefully before the battery dies, rather than crashing hard mid-write.

### The recurring problem

After a real power flicker, the Linux kernel's generic USB driver kept claiming the UPS's USB device before the monitoring service could get to it — causing "driver not connected" and permission errors on every reboot following a power event.

**Fix:** a `udev` rule granting the specific device the correct group permissions, plus a custom `systemd` service that unbinds the device from the kernel, waits, rebinds it, then fixes socket permissions — all automatically, before the monitoring service starts. This turned a recurring manual fix into something that survives reboots and power events on its own.

Verified via the monitoring tool's own status command: battery charge, load percentage, and "online, charging" status all reporting correctly.
