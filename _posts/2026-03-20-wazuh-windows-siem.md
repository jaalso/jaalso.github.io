---
title: "Building Detections in Wazuh: Windows Lab SIEM"
date: 2026-03-20
category: blue-team-labs
tags: [wazuh, siem, blue-team, detection, windows]
---

> **Template post** — replace with your real Wazuh write-up.

Setting up Wazuh against a `WIN10TEST` VM, generating attack telemetry, and
writing rules that actually fire — including the noisy ones that didn't.

## Lab setup

- Wazuh manager: `<VM>`
- Endpoint: `WIN10TEST` with agent installed
- Attack source: Kali

## Generating telemetry

What you ran to produce events worth detecting.

## Writing the rule

```xml
<rule id="100001" level="10">
  <if_sid>...</if_sid>
  <description>Suspicious ... detected</description>
</rule>
```

## What fired, what didn't

Be honest about false positives and tuning.

## ATT&CK mapping

| Technique | ID | Detected by |
|-----------|-----|------------|
| ... | T1059 | rule 100001 |
