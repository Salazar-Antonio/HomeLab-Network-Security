# Key Lessons Learned

**Config drift is real and silent.**
A setting configured correctly once can be quietly overridden later by a drop-in config file, a package update, or provisioning tooling reasserting its defaults. Verify with a tool's own merged-output command rather than trusting a single static file — "I set this already" and "this is actually still true" are different claims.

**Network firewalls and host firewalls see different things.**
A router-level firewall only governs traffic that actually crosses the router — different VLANs, or out to the internet. Two devices on the same subnet talk directly at Layer 2, completely invisible to it. A host-based firewall is the only thing that can close that gap. Real defense-in-depth means both layers, not either one alone.

**Placement is a security decision, not a convenience one.**
"Where should this live" should be answered by blast radius — what's exposed if this specific thing is compromised — not by which box is easiest to reach. The most-hardened box isn't automatically the safest place to add something new; it's often the box deliberately built to absorb a hit, which is a reason to add less to it, not more.

**Per-user accounts matter for accountability.**
Sharing one login between multiple people destroys the ability to tell who did what from the logs. Separate accounts, locked (not deleted) when no longer needed, keep that trail intact.

**Verify, don't assume — especially your own past work.**
Nearly every finding documented across this build came from actually checking a setting or reading a log, rather than trusting it was still true because it was configured once. That habit is worth more than any single fix.

**The troubleshooting methodology isn't something to memorize — it gets lived.**
Identify the problem → form a theory → test it → verify the fix → document it. Every "war story" in this build followed that loop, on real owned hardware, repeatedly, until it stopped being a checklist and became how I actually think through a problem.

## What's next

- A direct Ethernet run to eliminate the double-NAT entirely
- Extending intrusion detection visibility to the DMZ segment (currently WAN-only)
- Periodic re-verification of hardening settings using merged-config output rather than a single file
- Continued periodic re-audits — several of the findings above were only caught because an old assumption got checked instead of trusted
