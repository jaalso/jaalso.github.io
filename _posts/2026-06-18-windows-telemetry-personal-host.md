---
title: "Windows Telemetry — What Microsoft Sees from a Personal Host"
date: 2026-06-18
category: real-world
featured: false
tags: [privacy, telemetry, dns, windows, pi-hole]
---

I wanted a concrete answer to a question most people wave away: what does my own
Windows machine actually send to Microsoft, without me asking it to? Not the marketing
answer — the DNS answer. So I put a DNS sinkhole between my laptop and the internet and
read the query log. The telemetry surface was immediately visible, and blocking it was
one blocklist away. This investigation is what led me to build the [hardened Pi-hole
home lab](/labs/).

## The method — and its honest limits

A DNS sinkhole (Pi-hole) sees every domain a host tries to resolve before any
connection is made. That's the right layer for this question: I'm not decrypting
payloads or claiming to read what Microsoft received — I'm documenting *where my host
reaches out*, and then stopping the connection at the door. DNS reveals the telemetry
surface; blocking at DNS means the request never leaves the network.

- Observation: Pi-hole query log on a Raspberry Pi 5, laptop DNS pointed at it
- Attribution: `pihole -q <domain>` to prove *which* blocklist caught each endpoint
- Scope: my own EliteBook only, on my own network

## What showed up

Filtering the query log for Microsoft-owned destinations, these telemetry endpoints
were being requested by the host — and blocked:

| Endpoint | Associated with | Caught by |
|----------|-----------------|-----------|
| `vortex.data.microsoft.com` | Windows diagnostic telemetry (DiagTrack) | WindowsSpyBlocker |
| `telemetry.microsoft.com` | Windows telemetry collection | WindowsSpyBlocker |
| `browser.events.data.msn.com` | Edge / MSN browser event telemetry | HaGeZi multi |
| `g.live.com` | Microsoft account / Live service beacon | HaGeZi (as `g.msn.com`) |

Each was blocked on both **A and AAAA** lookups — IPv6 telemetry doesn't slip past a
sinkhole that filters AAAA too, which is a common gap when people only set IPv4 DNS.

## The scale

Across normal use, the dashboard settled at roughly **24% of all DNS queries blocked**.
That number isn't all telemetry — it includes ads and trackers from a 2.3-million-domain
blocklist — but the Microsoft/Edge diagnostic endpoints are a consistent, recurring
slice of it, requested in the background without user action.

## Why it matters

At a personal level, this is the difference between *assuming* a machine phones home and
*proving* which endpoints it contacts, with per-domain attribution. At an organisational
level, the same DNS-layer visibility is how you'd inventory and control outbound
telemetry across an estate — and under GDPR / the Swiss nFADP, "what leaves the endpoint,
to whom" is a question a business is expected to be able to answer.

The honest caveat stands: DNS tells you the destination, not the contents. But for
telemetry, the destination *is* the finding — and it's also the control point.

## From research to mitigation

Seeing the telemetry surface concretely is what justified building the infrastructure to
block it network-wide: a hardened Pi-hole sinkhole, verified end to end. That build is
written up separately as a [lab](/labs/) — this piece is the reason it exists.

## Takeaways

- **DNS is the cheapest visibility you'll ever get.** No agent, no packet capture — a
  sinkhole shows the outbound telemetry surface of every device behind it.
- **Filter AAAA, not just A.** IPv6 telemetry is a silent bypass otherwise.
- **Attribution beats a blocklist count.** `pihole -q` turns "something was blocked" into
  "this specific endpoint, caught by this list" — which is what makes it evidence.

---

*All observation was performed on my own host and network. No third-party systems were
accessed; this documents outbound DNS requests from a personal machine and their
mitigation.*
