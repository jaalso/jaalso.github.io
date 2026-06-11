---
title: "Booking.com Storm-1865 Phishing Triage"
date: 2026-05-13
category: real-world
featured: true
tags: [phishing, cti, threat-intel, ncsc, urlscan, mitre-attack, storm-1865]
---

A relative in Switzerland received three WhatsApp messages — French, German, and
English — impersonating their hotel's reservation team and demanding they "verify"
a booking through a link. Using a structured CTI reading workflow and a triage chain
(Inoreader feed → open-source research → URLscan.io → Have I Been Pwned → NCSC.ch
advisories), I identified the campaign as Storm-1865's "I Paid Twice" variant — a
Russian-origin operation abusing Booking.com's April 2026 partner-portal breach. I
reported it to the Swiss NCSC, which confirmed the attribution and acknowledged the
URLscan IOC for downstream blocklist action.

## The lure

The victim received three messages in quick succession on WhatsApp from an unknown
number, claiming to be from "Diana, your check-in manager" at the booked hotel. The
messages stated that the hotel was ending its Booking.com partnership, that the
reservation had to move to the hotel's "direct booking system", that a 50% discount
was available for rebooking through a personal link, and that a full refund of the
original payment would follow. To "verify", the victim was told to approve two push
notifications or SMS codes from their bank.

The link was hosted at `booking.roomstation.help/reservation/[redacted]`.

## Initial red flags

| Indicator | Why it's suspicious |
|-----------|---------------------|
| WhatsApp contact | Real Booking.com communication happens in-app, never via WhatsApp |
| 50% discount lure | Classic financial-incentive social engineering |
| Domain `roomstation.help` | Legitimate Booking.com domains are always `booking.com` |
| `.help` TLD | Uncommon, cheap, favoured by phishers |
| Multilingual flood | Profiling trick — attacker doesn't know the victim's language |
| "Approve two bank requests" | The actual attack vector — both approvals debit the victim |
| Exact booking details quoted | Confirms breach data is in use |
| Urgent, time-limited framing | Forced-decision pressure |

## Triage chain

**Phase 1 — CTI feed search.** Searched my personal Inoreader CTI dashboard for
Booking.com. Three relevant items surfaced from the prior weeks: BleepingComputer on
the breach forcing reservation PIN resets (April 2026), SecurityWeek confirming
attacker access to user information (April 2026), and an earlier BleepingComputer piece
on a Booking.com phishing campaign using a lookalike `ん` character (August 2025).

**Phase 2 — open-source research.** Searching for the hotel-partner phishing scam
tied the activity to Storm-1865 (Microsoft's attribution), using the ClickFix technique
against hotel employees to deploy XWorm and VenomRAT. Bridewell tracks it as intrusion
set BR-UNC-030 with Russian-origin code comments in the customer phishing kit; Krebs
documented the phishing-as-a-service infrastructure behind the 50%-discount fraud; and
the earlier name "I Paid Twice" came from Sekoia (November 2025).

**Phase 3 — Have I Been Pwned.** Checking the victim's exposure showed three breaches.
The most operationally relevant was Luxottica (2021) — name, DOB, phone, address — the
likely source of the phone number used to reach them on WhatsApp. Booking.com's April
2026 breach supplied the booking-specific data (hotel name, dates).

| Breach | Date | Relevance |
|--------|------|-----------|
| Synthient Credential Stuffing | 2025 | Email + password in active credential-stuffing lists |
| Luxottica | 2021 | Name, DOB, phone, address — likely source of the WhatsApp number |
| Dropbox | 2012 | Salted hashes — low current relevance |

**Phase 4 — URLscan.io infrastructure analysis.** Submitting the domain returned the
detail that made the whole thing click:

| Property | Value | Interpretation |
|----------|-------|----------------|
| Domain age at scan | 1 minute | Active campaign — freshly rotated infrastructure |
| Main IP | 188.114.97.3 | Cloudflare (AS13335) — hides the real backend |
| TLS issuer | Let's Encrypt E8 | Free, throwaway cert |
| TLS issued | 11 May 2026 | Two days before the victim was messaged |
| Page title | "Nur einen Moment..." | Loading-page lure — classic ClickFix pattern |
| Page banner | "Sicherheitsüberprüfung wird durchgeführt" | Fake security check |
| Redirects | 2 | Typical of phishing kits |
| URLscan / Safe Browsing | No classification | Too new to be on blocklists yet |

A domain one minute old, on a cert issued two days before contact, not yet on any
blocklist — this was live, freshly rotated infrastructure, not a stale link.

**Phase 5 — NCSC.ch corroboration.** The Swiss NCSC archive showed the campaign in
Wochenrückblick 47/2023 and 10/2024, confirming continuous activity against Swiss
residents for over two years. I reported the case on 13 May 2026; an NCSC analyst
confirmed the report, validated the attribution to the Booking.com breach data, and
acknowledged the URLscan IOC for downstream blocklist action.

## Indicators of compromise

```
DOMAIN:   booking.roomstation.help
IP:       188.114.97.3 (Cloudflare front)
IP:       104.18.94.41 (Cloudflare front)
ASN:      AS13335 (CLOUDFLARENET)
TLS CN:   Let's Encrypt E8 intermediate — issued 11 May 2026
TTP:      ClickFix — "Sicherheitsüberprüfung wird durchgeführt"
THEME:    Hotel partnership-termination + 50% discount + dual-bank approval
GROUP:    Storm-1865 (Microsoft) / BR-UNC-030 (Bridewell)
CAMPAIGN: "I Paid Twice" / Booking.com partner phishing
NCSC REF: RNR-277766 (13 May 2026)
```

## MITRE ATT&CK mapping

| Tactic | Technique | ID |
|--------|-----------|-----|
| Initial Access | Phishing: Spearphishing via Service | T1566.003 |
| Initial Access | Phishing: Spearphishing Link | T1566.002 |
| Resource Development | Acquire Infrastructure: Domains | T1583.001 |
| Resource Development | Acquire Infrastructure: Web Services | T1583.006 |
| Resource Development | Obtain Capabilities: Code Signing Certificates | T1588.003 |
| Credential Access | Steal Web Session Cookie | T1539 |
| Defense Evasion | Hide Infrastructure | T1665 |

## Takeaways

The decisive signal wasn't any single red flag — it was the URLscan domain age. A
one-minute-old domain on a two-day-old cert reframes the whole case from "probably
phishing" to "live, actively rotated campaign infrastructure". The HIBP pivot was the
other lesson: the attacker's contact channel (the WhatsApp number) traced back to an
unrelated 2021 breach, while the booking specifics came from the 2026 one — two
separate leaks combined into one convincing lure.

## References

- [Malwarebytes — Booking.com breach gives scammers what they need](https://www.malwarebytes.com/blog/data-breaches/2026/04/booking-com-breach-gives-scammers-what-they-need-to-target-guests)
- [Krebs on Security — Booking.com Phishers May Leave You With Reservations](https://krebsonsecurity.com/2024/11/booking-com-phishers-may-leave-you-with-reservations/)
- [Bridewell — The Booking.com Phishing Campaign](https://www.bridewell.com/insights/blogs/detail/the-booking.com-phishing-campaign-targeting-hotels-and-customers)
- [State of Surveillance — Booking.com Breach Timeline](https://stateofsurveillance.org/news/booking-com-data-breach-reservation-data-supply-chain-phishing-2026/)
- [NCSC.ch — Wochenrückblick 47/2023](https://www.ncsc.admin.ch/ncsc/de/home/aktuell/im-fokus/2023/wochenrueckblick_47.html)
- [NCSC.ch — Wochenrückblick 10/2024](https://www.ncsc.admin.ch/ncsc/de/home/aktuell/im-fokus/2024/wochenrueckblick_10.html)

---

*The targeted family member's name and identifying details have been omitted. The case
was triaged with their consent and no personal data is reproduced.*
