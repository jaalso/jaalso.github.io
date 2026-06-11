---
title: "Windows Forensics: KAPE Triage of a Compromised Host"
date: 2026-02-10
category: incident-response-labs
tags: [dfir, kape, windows, incident-response, ez-tools]
---

> **Template post** — replace with your real KAPE write-up.

Artifact collection and timeline reconstruction with KAPE and EZ Tools —
registry, prefetch, and event logs to answer who, what, and when.

## Collection

```
kape.exe --tsource C: --target !SANS_Triage --tdest E:\triage
```

## Parsing with EZ Tools

What you ran (`PECmd`, `MFTECmd`, `EvtxECmd`, ...) and what each surfaced.

## Timeline

Reconstructed sequence of events.

## Findings

| Time (UTC) | Artifact | Observation |
|-----------|----------|-------------|
| ... | Prefetch | ... |
