# DMZ & Production Application

## Why a DMZ

A fourth VLAN was added specifically to host anything that must be reachable from the public internet, without exposing anything else. This is the textbook screened-subnet pattern: a compromise here cannot reach Management, Servers, or the Lab VLAN — the firewall rules confirm the DMZ can reach the internet but nothing internal.

## The application

A real production website, built end-to-end (not via a website builder) for a friend's small business, hosted inside the DMZ.

**Stack, top to bottom:**

```
Internet → CDN/DNS → Reverse proxy (Nginx) → App server (Gunicorn) → Flask app → Linux
```

A tunneling service hides the home network's public IP entirely — the internet only ever sees the CDN, never the real server. Security controls built into the app itself: input validation, rate limiting, secrets kept out of source code, and sanitization against cross-site scripting on all user-submitted content.

### Problems hit

- **Emails silently stopped working.** The app's logs showed an authentication error against the mail provider. Root cause: a credential had changed and the app was still using the old one. A couple of real submissions were lost before it was caught — the fix was generating a fresh app-specific password.
- **Mobile display bug.** A hero image displayed incorrectly (scaling/cropping) on smaller screens. Fixed with a dedicated responsive image for mobile.

**Recurring lesson from this project:** when changes don't seem to show up, verify file locations, restart the service, and check logs *before* touching code again — most "it's not working" moments were actually stale processes or caches, not real bugs.

## A full four-layer security audit

Worked layer by layer from the outside in, like a home inspection — find what's solid, what needs work, what could bite later.

### Layer 0 — the CDN (first wall)

Already correctly hiding the origin server and had basic bot-integrity checks on. Turned on additional protections that were off by default: bot-challenge mode for suspected automated traffic, blocking of AI scraping bots, and a decoy-maze response for bots that ignore the block entirely.

### Layer 1 — network firewall / intrusion prevention

Already in great shape — the intrusion prevention system had been actively blocking dozens of inbound brute-force attempts at the WAN before they touched anything inside. One gap found: the IPS only watched the WAN interface, not the DMZ, so malware "calling home" from a compromised DMZ box would go unseen. **Fix:** routed the DMZ's DNS through the internal DNS server instead of a public resolver, so every domain the server looks up gets logged and checked against a known-bad-domain blocklist automatically.

### Layer 2 — the VM itself (OS & SSH)

Login history was clean — but a port scan found an unrelated test tool exposed publicly by accident, and SSH was still using password auth with no brute-force protection installed. **Fix:** removed the exposed tool entirely, installed and tuned brute-force protection, generated a real key pair, and disabled password auth — SSH became key-only.

### Layer 3 — the web app itself

Logs showed only normal internet background noise — no real attacks. But automated bots were filling out the contact form to spam it. **Fix:** added a honeypot — a hidden form field real users never see; if the backend finds it filled in, it fakes a success response and silently discards the submission, so the bot thinks it worked and doesn't adapt or retry with a different technique. Also added a crawler-exclusion rule to cut down on scanner noise in the logs.

### The stack, after the audit

CDN (hides origin, blocks bots) → intrusion prevention (blocks known-bad IPs before they reach anything) → DMZ isolation (contains any compromise) → DNS logging (visibility into outbound calls) → brute-force protection → key-only SSH.
