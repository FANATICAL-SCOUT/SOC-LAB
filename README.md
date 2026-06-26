# 🛡️ Home SOC Lab

A personal Security Operations Center (SOC) home lab built on VMware Workstation Pro, designed to simulate real-world threat detection, log analysis, and incident response workflows. This project demonstrates hands-on experience with industry-standard open-source security tools across a self-configured multi-VM environment.

---

## 🎯 What This Project Is About

The goal of this lab is to replicate — at a small scale — how a real SOC operates. In a production environment, a SOC team monitors an organisation's infrastructure for signs of attack, investigates alerts, and responds to incidents. This lab reproduces that workflow from scratch using open-source tools on a home machine.

The core idea is simple: **generate realistic attack traffic, detect it, and alert on it.** To do that, the lab is split into two logical sides — an attacker side and a defender side — running simultaneously on the same host.

### The Attacker Side

A Kali Linux VM acts as the adversary. It sits on the same virtual network as the other machines and is used to simulate real attack techniques: port scanning, brute-force login attempts, malware staging, and suspicious command execution. This gives the defender side something real to detect — not synthetic test data, but actual attack patterns hitting actual services.

### The Defender Side

The defender side is built around **Wazuh**, an open-source SIEM that acts as the brain of the operation. Two endpoint VMs — an Ubuntu Linux machine and a Windows 10 machine — run Wazuh agents that continuously ship logs, system events, and file changes to a central Wazuh manager running on a dedicated Ubuntu Server VM. The manager ingests all of this, runs it through detection rules, and raises alerts when something looks malicious.

On top of host-level telemetry, **Suricata** — a network intrusion detection system — runs on the Ubuntu Linux VM and passively monitors all traffic flowing across the virtual network. Because this VM sits on the same virtual switch as every other machine in the lab, Suricata sees traffic between all of them. Its alerts are forwarded into Wazuh, giving the SIEM both host-level and network-level visibility at the same time.

**Sysmon** runs on the Windows 10 VM to fill a gap that standard Windows event logs leave open. Where Windows logs tell you *that* something happened, Sysmon tells you *how* — capturing process trees, parent-child relationships, network connections spawned by specific processes, and hashed executables. This is the kind of telemetry that makes Windows incidents actually investigable.

**VirusTotal** is integrated with Wazuh's File Integrity Monitoring module. When a monitored file changes, Wazuh computes its hash and queries the VirusTotal API. If the file is known-malicious, a high-confidence alert fires immediately — no manual lookup required.

### The End-to-End Flow

An attack originates on the Kali VM → network traffic hits Suricata, which raises a network-level alert → the targeted endpoint's Wazuh agent picks up the host-level activity (failed logins, suspicious commands, file changes) → both streams flow into the Wazuh manager → correlated alerts appear in the dashboard → Active Response can automatically block the attacker's IP.

Everything is self-hosted, self-configured, and tested against real simulated attacks — not a pre-built lab environment.

---

## 🔧 Tools & Technologies

| Tool | Role |
|---|---|
| **Wazuh** | SIEM — centralized log collection, correlation, and alerting |
| **Suricata** | IDS — network traffic inspection and signature-based detection |
| **Sysmon** | Windows host telemetry — process, network, and file event logging |
| **VirusTotal API** | Threat intelligence — hash lookups on flagged files |
| **VMware Workstation Pro** | Hypervisor — multi-VM lab environment |

---

## ✅ Implemented Features

### 1. Wazuh SIEM
- Wazuh manager deployed on a dedicated Ubuntu Server VM
- Agents enrolled from both Ubuntu Linux and Windows 10 VMs
- Real-time log ingestion and alert generation visible in the Wazuh dashboard

### 2. File Integrity Monitoring (FIM)
- Configured on the Ubuntu Linux VM monitoring `/home/labuser321`
- Detects and alerts on file **additions**, **modifications**, and **deletions**
- Rules applied at both the manager and agent level for correct event propagation

### 3. Suricata IDS + Wazuh Integration
- Suricata 7.0.3 installed on the Ubuntu Linux VM
- `eve.json` log output forwarded to Wazuh via `<localfile>` block in `ossec.conf`
- `community-id` enabled in `suricata.yaml` for cross-tool flow correlation
- Suricata alerts surface in the Wazuh dashboard under network events

### 4. Sysmon (Windows 10)
- Sysmon deployed on the Windows 10 VM with a community-recommended config
- Enriched telemetry covering process creation, network connections, and file events
- Logs forwarded to Wazuh for correlation with network-level detections

### 5. Vulnerability Detection
- Wazuh vulnerability detection module enabled
- Periodic scans against the agent OS and installed packages
- CVEs surfaced in the Wazuh dashboard with severity ratings

### 6. Malicious Command Detection
- Custom Wazuh rules to detect suspicious command patterns (e.g. reverse shells, recon commands)
- Tested via simulated attack commands from the Kali VM

### 7. SSH Brute Force Detection & Active Response
- Wazuh detects repeated failed SSH login attempts using built-in decoder rules
- Active Response configured to automatically block offending IPs via `firewall-drop`
- Tested end-to-end using `hydra` from the Kali VM

### 8. VirusTotal Integration
- Wazuh FIM events trigger VirusTotal hash lookups via the free public API
- High-fidelity alerts generated when a monitored file matches a known malicious hash
- Rate-limiting handled to stay within the free tier

### 9. Custom Detection Rules
- Custom Wazuh rules written in XML targeting lab-specific attack patterns
- Rules cover privilege escalation attempts, suspicious cron modifications, and recon activity

---

## 🗂️ Repository Structure

```
SOC-LAB/
├── README.md
├── docs/
│   ├── lab-setup.md                      # VM specs, network topology, data flow
│   ├── components.md                     # Deep-dive on each component and config
│   ├── wazuh-setup.md                    # Wazuh server install and agent enrollment
│   ├── vm-setup-and-ram-planning.md      # Host RAM planning and VM sizing
│   └── detections/
│       ├── FILE_INTEGRITY_MONITORING.md
│       ├── suricata-ids and working.md
│       └── ssh-bruteforce.md
└── assets/
    ├── SOC_ARCHITECTURE_DIAGRAM_.png
    └── wazuh-agents-active.png
```

---

## 🏗️ Architecture

![SOC Lab Architecture](./assets/SOC_ARCHITECTURE_DIAGRAM_.png)

The architecture diagram above shows the full VM layout, network topology, and data flow across the lab. Alert screenshots from the Wazuh dashboard are documented inline in each detection write-up under [`docs/detections/`](./docs/detections/).

---

## 📄 License

This project is for educational and portfolio purposes. No warranty is provided.
