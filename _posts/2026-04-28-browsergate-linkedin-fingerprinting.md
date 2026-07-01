---
title: "BrowserGate — Independent Verification of LinkedIn Browser Fingerprinting"
date: 2026-04-28
category: real-world
featured: false
tags: [privacy, browser-fingerprinting, devtools, gdpr, threat-verification, osint]
---

When the "BrowserGate" disclosure broke in April 2026 — claiming LinkedIn's site
silently fingerprints visitors' browsers, down to their installed extensions — I didn't
want to take the headline on faith. So I reproduced it myself in Chrome DevTools,
confirmed the data was leaving my browser, then proved a one-step mitigation actually
stopped it. Verify, or don't believe it.

## The claim

BrowserGate, disclosed by Fairlinked e.V. and covered by BleepingComputer and The Next
Web, alleged that LinkedIn's production JavaScript transmits an encrypted browser
fingerprint — including the list of installed browser extensions — to its servers on
every page load, without the user's knowledge or consent. I treated it as a
threat-verification exercise rather than news to repeat: capture the evidence, or
discard the claim.

## Verifying it in DevTools

With Chrome's DevTools Network tab open on LinkedIn, I filtered the traffic and found
the endpoint the disclosure named:

- **3 × HTTP 200 POST** requests to the `sensorCollect` endpoint per session
- **~0.4 kB per call, ~1.2 kB per session** — small encrypted payloads consistent with
  a fingerprint, not page content
- Source traced to an **obfuscated Webpack bundle** (`chunk.905`, module 75023) —
  deliberately hard to read
- A **third-party tracker (Human Security / PerimeterX)** injected via a hidden iframe

The claim held up: the data was real, it was leaving the browser, and the delivering
script was obfuscated.

## Mitigation — and proving it worked

Verification isn't just confirming the problem; it's confirming a fix. I switched to
Brave (which blocks this class of tracking by default) and re-ran the identical capture:

| Metric | Chrome (before) | Brave (after) |
|--------|-----------------|---------------|
| sensorCollect calls | 3 × HTTP 200 | 3 × blocked:other |
| Data transferred | 1.2 kB | 0.0 kB |
| LinkedIn received data | yes | no |
| Extension list exposed | full list | randomized |
| Time to block | — | 14 ms |

Then I corroborated with an independent method — EFF's Cover Your Tracks, which tests a
browser against 311,004+ real fingerprints:

| Browser | Protection | Fingerprint |
|---------|-----------|-------------|
| Chrome | Weak | unique — 18.25 bits of identifying info |
| Brave | Strong | randomized |

Two independent methods agreed: the data left Chrome, and Brave stopped it.

## Why it matters — the regulatory angle

An installed-extension list can reveal sensitive traits — health conditions, politics,
or religion inferred from what someone installs. Under GDPR Article 9, processing
special-category data requires explicit consent, which silent fingerprinting doesn't
obtain. The context isn't hypothetical: LinkedIn's parent, Microsoft, was fined
EUR 310M by the Irish DPC in October 2024 for prior data-protection violations, and the
ceiling for a breach at that scale is roughly 4% of global turnover.

## Takeaways

- **Verify, don't repeat.** A disclosure is a hypothesis until reproduced. DevTools
  turned a headline into confirmed evidence.
- **A fix isn't done until it's measured.** "Switch to Brave" only counts because I
  re-captured and saw `blocked:other` and 0.0 kB.
- **Privacy is a security problem.** Fingerprinting is a data-protection and regulatory
  exposure, not just an annoyance — the lens a Swiss or EU business actually needs.

## Source &amp; references

Full capture notes and screenshots: [jaalso/security-research](https://github.com/jaalso/security-research).

- Fairlinked e.V. — original BrowserGate disclosure
- [The Next Web — LinkedIn BrowserGate extension scanning](https://thenextweb.com/news/linkedin-browsergate-extension-scanning-privacy-fingerprint)
- EFF Cover Your Tracks — browser fingerprinting test

---

*Testing was performed on my own browser and account; no LinkedIn systems were attacked
or accessed beyond normal browsing. This is independent verification of a publicly
disclosed issue.*
