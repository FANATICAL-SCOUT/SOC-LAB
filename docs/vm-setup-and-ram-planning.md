# VM Setup & RAM Planning for a Home SOC Lab

Before diving into tool configuration, it's worth spending a few minutes on resource planning. One of the most common reasons home SOC labs stall is running out of RAM mid-setup. This page covers how to calculate a sensible VM allocation based on your hardware, along with a quick overview of each VM you'll need and how to get them running.

---

## Why RAM Planning Matters

A functional home SOC lab needs at least 4 VMs running together:

| VM | Role |
|----|------|
| Wazuh Server (Ubuntu Server) | SIEM manager — must always be on |
| Agent VM (Ubuntu Linux) | Suricata IDS + Wazuh agent |
| Agent VM (Windows 10) | Sysmon + Wazuh agent |
| Kali Linux | Attacker / attack simulation |

The thing to keep in mind: your host OS (Windows 11) also needs RAM to function smoothly. Allocating everything to VMs without leaving headroom for the host tends to result in sluggish performance across the board — so a bit of upfront planning goes a long way.

---

## The RAM Formula

```
Usable RAM for VMs = Total Host RAM - Host OS Overhead

Host OS Overhead ≈ 3–4 GB (Windows 11 baseline)
```

So for common host RAM sizes:

| Host RAM | Usable for VMs | Notes |
|----------|---------------|-------|
| 8 GB | ~4–5 GB | Very tight. Run Wazuh Server + one agent only. Kali separately. |
| 16 GB | ~12 GB | Comfortable. Run Server + Kali + one agent at a time. |
| 32 GB | ~28 GB | Can run all 4 simultaneously with headroom. |

> **Rule of thumb:** Never allocate more than 75% of your total RAM across all running VMs combined.

---

## Recommended Allocation by Host RAM

### 16 GB Host (this lab's setup)

| VM | Allocated RAM | Run When? |
|----|-------------|-----------|
| Ubuntu Server (Wazuh) | 3 GB | Always on |
| Kali Linux | 4 GB | Always on during attack simulations |
| Ubuntu Linux (Suricata) | 3 GB | When testing Linux-side detections |
| Windows 10 | 3 GB | When testing Windows-side detections |
| **Host OS headroom** | **~3 GB** | Always reserved |

**Total in active use:** ~10 GB (Server + Kali + one agent at a time)

The key decision: **only one agent VM runs at a time**. Ubuntu Linux and Windows 10 are never on simultaneously. Swap them depending on what you're testing.

---

### 8 GB Host (minimal viable setup)

| VM | Allocated RAM | Run When? |
|----|-------------|-----------|
| Ubuntu Server (Wazuh) | 2 GB | Always on |
| Kali Linux | 2 GB | Only during attack simulations |
| Ubuntu Linux or Windows 10 | 2 GB | One at a time |
| **Host OS headroom** | **~2 GB** | Always reserved |

At 8 GB things are tight but workable. It helps to avoid running heavy applications on the host (like a browser with many tabs) while VMs are active. Be intentional about what's running at any given time and you'll be fine.

---

### 32 GB Host (comfortable)

| VM | Allocated RAM |
|----|-------------|
| Ubuntu Server (Wazuh) | 4 GB |
| Kali Linux | 4–6 GB |
| Ubuntu Linux | 4 GB |
| Windows 10 | 4 GB |
| **Host headroom** | **~14+ GB** |

At 32 GB you can run all four simultaneously and keep your host browser open without issue.

---

## Hypervisor Choice: VMware vs VirtualBox

Both work fine for this lab. I used **VMware Workstation Pro** — it has slightly better performance and networking flexibility for multi-VM setups. VirtualBox is free and fully capable if that's what you have.

The VM configs, network setup, and everything else in this repo applies equally to both. Just adapt the UI steps.

---

## VM Installation — Quick Overview

Rather than a full step-by-step walkthrough (each hypervisor has great official docs and there are plenty of video guides for each OS), here's a quick summary of what each VM is, where to get it, and what to keep in mind during setup:

---

### Ubuntu Server (Wazuh SIEM)

- Download: [ubuntu.com/download/server](https://ubuntu.com/download/server) — LTS version
- New VM → Linux → Ubuntu 64-bit
- Minimal install, no desktop environment needed
- Set a static IP after install (or reserve one in your hypervisor's NAT/network settings)
- This becomes your Wazuh manager

---

### Ubuntu Linux (Suricata + Wazuh Agent)

- Download: [ubuntu.com/download/desktop](https://ubuntu.com/download/desktop) — LTS version
- Standard desktop install is fine
- This VM needs to sit on the same virtual network as your other VMs so Suricata can sniff their traffic
- In VMware: use **NAT** or **Host-Only** adapter — same for all VMs

---

### Windows 10 (Wazuh Agent + Sysmon)

- Download: [microsoft.com/software-download/windows10](https://www.microsoft.com/software-download/windows10) — Media Creation Tool → ISO
- Standard install, no activation required for a lab
- Disable Windows Update after setup (it'll eat your RAM and disk in the background)
- This is where Sysmon will be installed later for enriched Windows telemetry

---

### Kali Linux (Attacker Machine)

- Download: [kali.org/get-kali](https://www.kali.org/get-kali/) — VMware pre-built image is the easiest option
- Kali ships a ready-made `.vmx` file for VMware — just import and run
- No manual install needed if you use the pre-built image
- All the attack tools you'll need (nmap, hydra, metasploit) come pre-installed

---

## Networking — One Setting That Matters

All VMs should be on the **same virtual network** so they can talk to each other (and Suricata can see their traffic). In VMware, the simplest approach is to set all VMs to **NAT** — they share the host's internet connection and can reach each other.

Avoid mixing NAT and Bridged adapters across VMs — it can introduce routing issues that are tricky to debug later.

---

## What's Next

Once your VMs are up and connected to the same network, the actual SOC setup begins:
- Install and configure Wazuh on the server
- Enroll agent VMs
- Deploy Suricata on the Ubuntu Linux VM
- Install Sysmon on Windows 10

Each of these is documented separately in this repo.
