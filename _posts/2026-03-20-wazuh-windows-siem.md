---
layout: post
title: "Building Detections in Wazuh: Windows Lab SIEM"
date: 2026-03-20
tags: [wazuh, siem, blue-team, detection, windows, fim, cis-benchmark, mitre-attack]
category: blue-team-labs
description: "Wazuh v4.14.3 deployed on VirtualBox with Kali Linux as monitored endpoint — CIS Benchmark automated assessment (190 controls, 45% score), File Integrity Monitoring, MITRE ATT&CK-tagged threat detection (258 events), and CVE vulnerability scanning."
---

After several weeks of running offensive tools on WIN10TEST — nmap scans, brute force, PsExec lateral movement — I wanted to see what those activities look like from the defender's side in real time. This lab deploys Wazuh as an open-source SIEM/XDR and enrolls Kali Linux as the monitored endpoint to answer the question: what does normal Kali tooling look like to a detection platform?

## What is Wazuh

Wazuh is a free, open-source SIEM/XDR/EDR platform used in production environments worldwide. It collects security events from endpoints, correlates them against MITRE ATT&CK, runs automated compliance assessments, and monitors file system changes in real time.

Commercial equivalents: Microsoft Sentinel, Splunk, Elastic Security.

## Environment

| Component | Role | IP |
|---|---|---|
| Wazuh OVA (Ubuntu) | SIEM server + dashboard | 192.168.56.105 |
| Kali Linux (bornia01) | Monitored endpoint (agent) | 192.168.56.101 |

Both VMs on VirtualBox Host-Only network — no internet required after initial setup.

## Deployment

Wazuh is distributed as a pre-configured OVA — download, import into VirtualBox, and boot. The dashboard is accessible immediately at `https://192.168.56.105`.

**Service recovery — startup timeout fix:**

On first boot, `wazuh-manager` shows as Offline. The cause: `wazuh-indexer` (Java, ~4.6 GB RAM) saturates memory before `wazuh-manager` can complete startup — a known timing issue with the OVA on low-RAM hosts.

```bash
# SSH into Wazuh server
ssh wazuh-user@192.168.56.105
# Password: wazuh (default OVA credential)

# Start manager manually after indexer fully initialises
sudo systemctl start wazuh-manager
sudo systemctl restart wazuh-dashboard

# Verify all three services running
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard
```

## Agent enrollment

```bash
# Download and install Wazuh agent with pre-configured variables
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.3-1_amd64.deb \
  && sudo WAZUH_MANAGER='192.168.56.105' \
         WAZUH_AGENT_GROUP='default' \
         WAZUH_AGENT_NAME='Kali-attacker' \
         dpkg -i ./wazuh-agent_4.14.3-1_amd64.deb

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent

# Verify connection
sudo tail -f /var/ossec/logs/ossec.log
# INFO: Connected to the server (192.168.56.105:1514)
```

Agent active within 30 seconds of starting — Kali visible in the Wazuh dashboard immediately.

## Results

| Metric | Result |
|---|---|
| Total events captured | 258 |
| CIS Benchmark score | 45% (83/190 controls passed) |
| Files monitored (FIM) | 7,958 |
| High CVEs identified | 1 |
| MITRE ATT&CK techniques tagged | T1046 · T1548 · T1078 · T1074 |

## CIS Benchmark assessment

Wazuh's Security Configuration Assessment (SCA) module runs an automated CIS benchmark against the enrolled endpoint — 190 controls, mapped simultaneously to ISO 27001, NIST SP 800-53, and PCI-DSS.

**45% on a default Kali installation is expected and informative.** Kali is a pentesting distro, not a hardened workstation. The failing 55% of controls are exactly the ones you'd expect: password complexity not enforced, unnecessary services running, audit logging not fully configured by default. Each failing control has a specific remediation command.

## File Integrity Monitoring

FIM monitors every file under `/etc/` in real time. Creating a test file triggers an alert within seconds:

```bash
# Create test file in monitored directory
sudo touch /etc/test-confidential.txt
echo 'RESTRICTED' | sudo tee /etc/test-confidential.txt

# Restart agent to force immediate FIM scan
sudo systemctl restart wazuh-agent
```

Dashboard alert: `Rule fired: "File added to the system"` — FIM automatically monitors `/etc/passwd`, `/etc/shadow`, `/etc/group`, and 7,958 other files.

## MITRE ATT&CK correlation

The most revealing finding: running standard Kali tools while the agent was active triggered automatic MITRE ATT&CK tagging.

| Rule | Event ID | MITRE Technique | What triggered it |
|---|---|---|---|
| 5402 | Sudo to ROOT | T1548 Privilege Escalation | Running `sudo` commands |
| 5501/5502 | PAM login sessions | T1078 Valid Accounts | Normal terminal logins |
| Custom | nmap -sS 127.0.0.1 | T1046 Network Service Discovery | Port scanning own host |

**The key lesson:** Wazuh fires on Kali's own normal tooling. In a real SOC, a Kali machine appearing as an enrolled endpoint would immediately look suspicious — every tool run generates alerts tagged with offensive MITRE techniques. This is how blue teams detect red team activity even before a payload is dropped.

## Recon detection test

To confirm detection, ran nmap against the agent's own localhost interface (scanning outbound doesn't trigger inbound alerts):

```bash
# Scan Kali's own IP — this appears as inbound recon to the agent
nmap -sS 127.0.0.1
```

Result: **T1046 Network Service Discovery** alert in Threat Hunting within seconds.

## Class topic mapping

| Class topic | Wazuh capability | What it demonstrates |
|---|---|---|
| Know Your Environment | Asset inventory + hardware fingerprint | Automatic endpoint baseline on connect |
| Security Architecture | EDR agent + MITRE ATT&CK correlation | Every event tagged with attack technique IDs |
| Data Governance | File Integrity Monitoring | Real-time alert on sensitive file changes |
| CIS Benchmarks | SCA — 190-control automated assessment | Same CIS checks as class, automated |
| VDP / Attack Simulation | Threat hunting + recon detection | Own nmap scans appear as T1046 alerts |

## Next steps (planned)

| Session | Exercise | Goal |
|---|---|---|
| A | Hydra brute-force against Wazuh SSH → Rule 5712 spike | Brute force pattern recognition |
| B | Deploy Windows 10 VM agent → compare CIS score vs Kali 45% | Multi-platform SIEM |
| C | Apply CIS remediation → verify score improvement | Full hardening loop |

## Takeaway

The 45% CIS score and 258 events are less important than what they show: every offensive action on a Kali host — even `sudo`, even a login, even `nmap localhost` — generates MITRE-tagged telemetry in a properly deployed SIEM. The same tools that generate those alerts are the ones I've been using for weeks in red team labs. Understanding both sides of that data flow is the point.

---

*Lab conducted on isolated VirtualBox Host-Only network. Wazuh OVA and Kali Linux VM — no external connectivity. All scanning activity against personally-owned VMs.*
