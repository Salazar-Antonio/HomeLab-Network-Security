# Jump Box (Bastion Host) & Security Hardening

## What it is

The only server directly reachable from the public internet. A minimal VM on the Servers VLAN — anyone accessing the lab remotely connects here first. Kept deliberately small in purpose: accept inbound SSH, serve one local app, and nothing else.

## Original hardening

- Root login disabled in SSH config
- Brute-force auto-ban tool installed and active
- Separate per-user accounts for each external person given access — never a shared login, so logs can show who actually connected
- Public reachability via a firewall rule allowing the access port inbound, plus a NAT rule forwarding it to the jump box specifically — nothing else on the network is directly reachable

## The hardening pass — checking if security had actually held

Prompted by a simple question: was security still in place, or had it drifted? The answer, on two boxes checked this way, was: partially drifted.

### Finding 1 — a security setting had silently reverted

A box that had been locked to key-only SSH weeks earlier was found, during an unrelated log check, to be accepting password logins again. Root cause: a config file that loads automatically on VM provisioning re-enabled password authentication, and files like this load *after* — and override — the main SSH config.

> **The lesson:** a security setting isn't done just because it was set once. Drop-in config files load after the main file and can silently override it. Always check both the main config *and* its drop-in directory, and verify with the tool's own merged-config output rather than trusting a single file that was edited once and never rechecked.

The jump box was checked the same way: its main config had never explicitly disabled password auth, and the same drop-in behavior applied there. It was brought in line with the key-only standard used everywhere else, verified with the merged-config output.

### Finding 2 — brute-force protection was active, but on soft defaults

Confirmed genuinely working (hundreds of failed attempts logged, dozens of IPs banned historically) — but the default settings (5 attempts, short ban window) are lenient for a box now carrying multiple external accounts. Tightened via a proper override file (never edit the tool's main config directly — package updates overwrite it): fewer attempts allowed, longer initial ban, and ban duration multiplying on repeat offenses up to a hard cap.

### Finding 3 — the jump box could reach far more than its job required

Its only real function is to accept inbound SSH and serve one static app locally. It never needed to reach any other internal server or the open internet — any legitimate need to reach internal servers already goes through the VPN directly. Decision: make it a true dead end.

**The firewall-level fix didn't fully work.** Two rules were added blocking the jump box from reaching internal networks and from reaching the internet. The internet-block worked immediately. The internal block did **not** — pings to another server on the same VLAN kept succeeding.

**Root cause, found by watching the firewall's live traffic log:** the jump box and the other server sit on the *same* VLAN. Traffic between two hosts on the same subnet is switched at Layer 2 and never routes through the firewall — so a router-level rule never gets the chance to evaluate it. A ping to an outside address (crossing the router) appeared in the log; the same-VLAN ping never did.

**The actual fix had to be host-based**, since the router structurally cannot see same-subnet traffic. Installed a host firewall directly on the jump box with explicit deny rules for every internal subnet, allow rules only for the two ports it actually needs to serve.

**Verified result:**

- Ping to the same-VLAN server: 100% loss — blocked by the host firewall
- Ping to the internet: 100% loss — blocked by the network firewall
- The jump box's own local app: still responding fine — loopback traffic is unaffected by outbound rules

> **Why both layers were needed:** the network-level firewall governs anything crossing a router boundary — different VLANs, or out to the internet. The host-level firewall governs traffic the router never sees — two devices on the same subnet talking directly at Layer 2. Real defense-in-depth needs both, because each one is blind to exactly what the other covers.

## External access accounts

Each external person given access has their own separate account rather than a shared login — locked (not deleted) when access is no longer needed, so accountability in the logs is never lost. Access is always via a local SSH tunnel to the one app the jump box serves; no direct exposure of anything else.

## OPNsense firewall hardening checklist

| Item | What it does |
|---|---|
| Firmware kept current | Patches known vulnerabilities (auth bypass, DoS vectors, RCE) before they can be exploited |
| HTTPS enforced on web UI | Prevents credential interception on the admin interface |
| Session auto-timeout | An unattended admin session can't be hijacked |
| Access logging | Every web UI login attempt is logged |
| SYN flood protection (adaptive) | Defends against connection-table exhaustion attacks, only activating when actually needed |
| Config version control | Every firewall config change is tracked and reversible |
| CPU microcode updates | Patches CPU-level vulnerabilities (Spectre/Meltdown class) |
| Intrusion Prevention (inline) | Inspects packet contents against known attack signatures, not just port/IP — actively blocks in real time |
| DNSSEC + DNS over TLS | Signed and encrypted DNS resolution |
| GeoIP blocking | Documented and ready to enable, pending a free account with the IP-geolocation data provider |
