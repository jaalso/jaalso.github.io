---
layout: post
title: "Windows Forensics: KAPE Triage of a Compromised Host"
date: 2026-02-10
tags: [dfir, kape, windows, incident-response, ez-tools, psexec, forensics]
category: incident-response-labs
description: "Full KAPE triage walkthrough on a Windows 10 VM after a simulated PsExec compromise — 3,303 artifacts collected in 13 minutes, PsExec service installs recovered from parsed Event Logs, MFT and SRUM analysis demonstrated."
---

After completing the Wayne Corp IR simulation — where CrackMapExec brute-forced a Windows 10 host and impacket-psexec established lateral movement — I ran a full KAPE triage to understand what a defender collects and how fast they can do it.

The question: can a single command collect everything needed to reconstruct the attack, and how long does it actually take?

**Answer: 3,303 artifacts, 1.6 GB, 13 minutes.**

## What is KAPE

KAPE (Kroll Artifact Parser and Extractor) is a portable forensic triage tool that does two things in sequence:

1. **Targets phase** — copies forensic artifacts from a live or offline disk (Prefetch, Amcache, Shimcache, Event Logs, Registry hives, MFT, SRUM, browser history, jump lists, LNK files)
2. **Modules phase** — runs EZ Tools parsers against everything it collected, producing analyst-ready CSVs

The key value: a single command replaces hours of manual artifact hunting and parsing.

## Setup

KAPE requires a one-time email registration at kroll.com. The download is a portable ZIP (~70 MB) — no installer needed.

```powershell
# Extract to C:\KAPE\
# Sync latest Targets and Modules from GitHub
cd C:\KAPE
Set-ExecutionPolicy Bypass -Scope Process -Force
.\kape.exe --sync
```

After sync: 267 Targets · 288 Modules available.

> **Note:** If `Get-KAPEUpdate.ps1` can't reach s3.amazonaws.com (common in VirtualBox host-only environments), use `--sync` instead — it pulls directly from GitHub and works reliably.

## The collection command

```powershell
.\kape.exe `
  --tsource C: `
  --target KapeTriage `
  --tdest C:\KAPEoutput\Triage `
  --module !EZParser `
  --mdest C:\KAPEoutput\Parsed `
  --tflush --mflush
```

| Flag | Meaning |
|---|---|
| `--tsource C:` | Source drive to collect from |
| `--target KapeTriage` | Built-in triage target — covers all key forensic artifacts |
| `--tdest` | Destination for raw collected files (preservation copy) |
| `--module !EZParser` | Run all EZ Tools parsers on the collected artifacts |
| `--mdest` | Destination for parsed CSVs (analyst's working set) |
| `--tflush --mflush` | Clear destinations first if they exist |

> **Defender note:** Windows Defender may flag KAPE during collection — dismiss any popups. Do not quarantine. KAPE is a legitimate forensic tool flagged by heuristics.

## Results

| Metric | Result |
|---|---|
| Files collected (Triage) | 3,303 files · 1.6 GB |
| Files deduplicated | 362 |
| Parsing processors run | 18 |
| Output CSVs produced | 30 (305 MB) |
| **Total runtime** | **13 minutes** |

## Inspecting the output

```powershell
# Triage folder structure (mirrors original disk layout)
Get-ChildItem C:\KAPEoutput\Triage -Directory | Select-Object Name

# Parsed output categories
Get-ChildItem C:\KAPEoutput\Parsed -Directory | Select-Object Name

# Count of CSVs produced
Get-ChildItem C:\KAPEoutput\Parsed -Recurse -Filter "*.csv" |
  Measure-Object | Select-Object Count

# Disk usage
$triage = (Get-ChildItem C:\KAPEoutput\Triage -Recurse |
  Measure-Object -Property Length -Sum).Sum / 1MB
$parsed = (Get-ChildItem C:\KAPEoutput\Parsed -Recurse |
  Measure-Object -Property Length -Sum).Sum / 1MB
"Triage: $([math]::Round($triage, 0)) MB"
"Parsed: $([math]::Round($parsed, 0)) MB"
```

Parsed output folders produced:

```
EventLogs\
ProgramExecution\
FileSystem\
FileFolderAccess\
FileDeletion\
Registry\
SQLDatabases\
SRUMDatabase\
SUMDatabase\
```

## Proving the workflow — finding the PsExec attack

This is the key validation: without targeting PsExec specifically, KAPE collected and parsed every artifact that recorded it. The impacket-psexec attack from the Wayne Corp simulation produced two service installs — `ysZf` and `VSGy` — logged as Event 7045 in System.evtx.

```powershell
# Find KAPE's parsed Event Log CSV
Get-ChildItem C:\KAPEoutput\Parsed\EventLogs -Filter "*.csv"

# Hunt for PsExec service installs (Event 7045)
$csv = Get-ChildItem C:\KAPEoutput\Parsed\EventLogs `
  -Filter "*EvtxECmd*.csv" | Select-Object -First 1
Import-Csv $csv.FullName |
  Where-Object {$_.EventId -eq "7045"} |
  Select-Object TimeCreated, Channel, PayloadData1, PayloadData2 |
  Format-List
```

Result: `ysZf` and `VSGy` service installations confirmed in the parsed output —
the same entries extracted manually during the Wayne Corp engagement. KAPE found
them automatically as part of its standard collection.

**Bonus finding during analysis — `KslD`:**

A 4-character service name appeared twice in the parsed data (April 12 and April 15),
matching the naming pattern of PsExec-generated services. Investigation:

```powershell
Get-Service -Name KslD -ErrorAction SilentlyContinue
Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Services\KslD `
  -ErrorAction SilentlyContinue |
  Select-Object DisplayName, ImagePath, Description, ObjectName
```

Returned details → legitimate service. Demonstrating exactly how KAPE surfaces
artifacts that need analyst judgement — not every 4-character service is malicious,
but every one deserves a check.

## Bonus artifacts: MFT and SRUM

### MFT — Master File Table

```powershell
Get-ChildItem C:\KAPEoutput\Parsed\FileSystem -Filter "*.csv"

# Files created during the PsExec attack window
$mft = Get-ChildItem C:\KAPEoutput\Parsed\FileSystem -Filter "*MFT_Output*"
Import-Csv $mft.FullName |
  Where-Object {$_.Created0x10 -like "2026-04-24 22:1*"} |
  Select-Object Created0x10, FileName, ParentPath -First 10 |
  Format-Table -AutoSize
```

Output: 207 MB MFT CSV (~200,000+ file records). Files created in the PsExec
attack window visible — including traces of `qSqxGVfD.exe` even after the binary
was deleted. MFT records persist after deletion — this is why attackers can't
fully sanitize a live system.

### SRUM — System Resource Usage Monitor

```powershell
$srum = Get-ChildItem C:\KAPEoutput\Parsed\SRUMDatabase `
  -Filter "*NetworkUsages*"
Import-Csv $srum.FullName | Select-Object -First 5 | Format-List
```

SRUM records every program's CPU, memory, and network usage by user with hourly
granularity — retained for 30+ days. If an attacker's binary made any network
connections, SRUM has bytes-sent and bytes-received logged even if the binary
is now deleted.

## What KAPE gives you that manual collection doesn't

| Capability | Manual collection | KAPE |
|---|---|---|
| Artifact collection | Hours (tool-by-tool) | ~5 minutes |
| Parsing | Hours (tool-by-tool) | ~8 minutes |
| MFT extraction | Complex (requires specialist tool) | Automatic |
| SRUM parsing | Often missed entirely | Automatic |
| Chain-of-custody log | Manual documentation | Auto-generated |
| Reproducibility | Varies by analyst | Identical every run |

## Key takeaway

The 13-minute collection produced the same forensic evidence that took hours to
extract manually during the Wayne Corp investigation — plus artifacts (MFT, SRUM,
SRUM NetworkUsages) that weren't covered in the manual workflow at all.

In a real IR engagement, this is the first command you run on a live host before
anything changes. Everything else — Timeline Explorer, EvtxECmd queries, MFT
analysis — is analysis of what KAPE already collected.

---

*Lab conducted on Windows 10 Enterprise (WIN10TEST) — isolated VirtualBox environment.
All attack activity referenced was part of the authorised Wayne Corp IR simulation.*
