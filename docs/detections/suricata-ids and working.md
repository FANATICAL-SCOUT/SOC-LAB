# Suricata IDS — Setup, Integration & Attack Detection

## Overview

Suricata is a **Network Intrusion Detection System (NIDS)**. Its sole job is to passively monitor network traffic, match it against thousands of threat signatures, and log suspicious activity to a file called `eve.json`.

In this lab, Suricata runs on the **Ubuntu Linux VM** and forwards alerts to the **Wazuh SIEM** via the Wazuh agent already present on the same machine.

---

## Why Suricata Lives on the Ubuntu Linux VM

A common question is: why not install Suricata on every VM?

The answer is that Suricata only needs to be installed **once**, on a VM that sits on the shared virtual network. In this lab, all VMs (Ubuntu, Windows 10, Kali) are connected to the same VMware virtual switch. When Kali launches an attack against any other VM, that traffic passes through the virtual network — and Suricata on Ubuntu, listening on its network interface, can see all of it.

Installing Suricata on every VM would be:
- Redundant — one instance already covers the full virtual network
- Wasteful — Suricata consumes ~540MB RAM per instance
- Harder to manage — rules and configs would need to be maintained in multiple places

The Ubuntu VM was chosen specifically because it already runs a Wazuh agent, meaning Suricata alerts can be forwarded to the Wazuh server without any additional infrastructure.

**Detection coverage breakdown:**

| Tool | VM | What it detects |
|---|---|---|
| Suricata | Ubuntu Linux | Network-level attacks across all VMs |
| Wazuh agent | Ubuntu Linux | Host-level events on Ubuntu |
| Wazuh agent | Windows 10 | Host-level events on Windows |
| Sysmon (planned) | Windows 10 | Enriched process/command telemetry |

---

## Lab Environment

| Component | Details |
|---|---|
| Host | Windows 11 |
| Hypervisor | VMware |
| Suricata VM | Ubuntu Linux (`labuser321`, interface: `ens33`, IP: `192.168.241.134`) |
| Wazuh Server | Ubuntu Server VM |
| Attacker | Kali Linux VM |
| Suricata Version | 7.0.3 |

---

## Architecture

```
Kali Linux (attacker)
       │
       │  network traffic (nmap, ping flood, etc.)
       ▼
VMware Virtual Switch
       │
       ▼
Ubuntu Linux VM (ens33 — 192.168.241.134)
  ├── Suricata 7.0.3
  │     └── sniffs ens33 → writes alerts to /var/log/suricata/eve.json
  └── Wazuh Agent
        └── reads eve.json → forwards to Wazuh Server → Dashboard
```

---

## Step 1 — Verify Suricata Installation

**On: Ubuntu Linux VM**

```bash
suricata --version
```

Expected output:
```
Suricata version 7.0.3 RELEASE
```

Confirm the network interface Suricata should listen on:

```bash
ip a
```

Look for the interface with your VM's IP (e.g., `192.168.241.134`). In this lab it is `ens33`.

---

## Step 2 — Configure suricata.yaml

**On: Ubuntu Linux VM**

Open the Suricata config file:

```bash
sudo nano /etc/suricata/suricata.yaml
```

### 2a — Set the network interface

Search for the `af-packet` section (`Ctrl+W` → `af-packet`). Confirm the interface is set to your VM's interface:

```yaml
af-packet:
  - interface: ens33
```

`af-packet` is a high-performance Linux method for capturing raw network packets. This is where Suricata is told which interface to sniff.

### 2b — Enable Community ID

Search for `community-id` (`Ctrl+W` → `community-id`) and set it to `true`:

```yaml
community-id: true
```

**Why community-id matters:** Community ID generates a unique fingerprint for each network flow based on source/destination IP, ports, and protocol. When the same attack generates both a Suricata network alert and a Wazuh host alert, both logs share the same community ID — making cross-tool correlation possible in the Wazuh dashboard.

Save and exit: `Ctrl+X` → `Y` → `Enter`

### 2c — Validate the config

Before starting the service, test the config for errors:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```

Expected output confirms:
- `eve-log output device initialized: eve.json`
- `50771 rules successfully loaded, 0 rules failed`
- `Configuration provided was successfully loaded`

---

## Step 3 — Start and Enable Suricata

**On: Ubuntu Linux VM**

```bash
sudo systemctl enable --now suricata
sudo systemctl status suricata
```

Expected: `Active: active (running)`

> **RAM note:** Suricata loads ~50,000 rules into memory at startup, consuming around 540MB RAM. If RAM is a constraint, disable auto-start with `sudo systemctl disable suricata` and start it manually when needed with `sudo systemctl start suricata`.

---

## Step 4 — Integrate Suricata with Wazuh Agent

**On: Ubuntu Linux VM**

The Wazuh agent needs to be told to read Suricata's `eve.json` file. Open the agent config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the following block just before the closing `</ossec_config>` tag:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

**What this does:** The `<localfile>` block tells the Wazuh agent to monitor this file and forward its contents to the Wazuh server. `log_format: json` tells Wazuh to parse each line as structured JSON rather than plain text, so all Suricata fields (src_ip, dest_ip, signature, community_id, etc.) become searchable fields in the dashboard.

Save and restart the Wazuh agent:

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

> **Note:** No changes are required on the Wazuh server's `ossec.conf` for Suricata integration. The server automatically processes JSON logs forwarded by the agent and maps them to Suricata's built-in decoder rules.

---

## Step 5 — Verify end-to-end Alert Flow

**On: Ubuntu Linux VM**

Trigger a test alert using the NIDS test URL:

```bash
curl http://testmynids.org/uid/index.html
```

This URL returns a response resembling a root user check (`uid=0(root) gid=0(root)`). Suricata has a built-in rule (`GPL ATTACK_RESPONSE id check returned root`) that flags this pattern.

Confirm Suricata logged the alert:

```bash
sudo tail /var/log/suricata/eve.json | grep -i "GPL ATTACK"
```

Then check the **Wazuh dashboard** → Security Events for a Suricata alert with rule.id `86601`.

---

## Step 6 — Attack Simulation from Kali Linux

All commands below are run on the **Kali Linux VM** targeting Ubuntu (`192.168.241.134`).

### 6a — Ping Flood

```bash
sudo ping -f 192.168.241.134
```

Sends a flood of ICMP packets to Ubuntu. Suricata detects the volume and unusual ICMP codes.

**Screenshot — Kali command:**

![Kali ping flood command](../../assets/kali-ping-flood-command.png)

**Screenshot — Wazuh alert:**

![Wazuh ICMP alert](../../assets/wazuh-suricata-icmp-alert.png)

---

### 6b — Nmap SYN Scan

```bash
nmap -sS 192.168.241.134
```

A SYN scan sends half-open TCP connections to probe which ports are open without completing the handshake. A classic reconnaissance technique.

**Screenshot — Kali command:**

![Kali nmap SYN scan](../../assets/kali-nmap-syn-scan.png)

**Screenshot — Wazuh alert:**

![Wazuh nmap SYN alert](../../assets/wazuh-suricata-nmap-syn.png)

---

### 6c — Nmap Aggressive Scan

```bash
nmap -A 192.168.241.134
```

`-A` enables OS detection, service version detection, and default script scanning. Generates richer traffic patterns for Suricata to inspect.

**Screenshot — Kali command:**

![Kali nmap aggressive scan](../../assets/kali-nmap-aggressive.png)

**Screenshot — Wazuh alert:**

![Wazuh nmap aggressive alert](../../assets/wazuh-suricata-nmap-aggressive.png)

---

### 6d — Nmap Vulnerability Script Scan

```bash
nmap --script vuln 192.168.241.134
```

Runs Nmap's built-in vulnerability detection scripts, probing for known CVEs and misconfigurations. Generates the most diverse set of Suricata alerts across multiple rule categories.

**Screenshot — Kali command:**

![Kali nmap vuln scan](../../assets/kali-nmap-vuln-scan.png)

**Screenshot — Wazuh alert:**

![Wazuh vuln scan alerts](../../assets/wazuh-suricata-vuln-alerts.png)

---

### 6e — Passive Detection: Kali Hostname Fingerprinting

This alert fires automatically when Kali joins the network — no manual attack needed. Suricata detects the Kali Linux hostname in a DHCP request packet and flags it using a threat intelligence rule.

**Screenshot — Wazuh alert:**

![Wazuh Kali DHCP fingerprint alert](../../assets/wazuh-suricata-kali-dhcp-fingerprint.png)

This demonstrates Suricata doing **OS and tool fingerprinting** at the network level, not just signature matching on attack payloads.

---

## What a Suricata Alert Looks Like in Wazuh

When a Suricata alert arrives in the Wazuh dashboard, it exposes the following key fields:

| Field | Description |
|---|---|
| `data.alert.signature` | The rule that fired (e.g., `GPL ATTACK_RESPONSE id check returned root`) |
| `data.alert.category` | Alert category (e.g., `Potentially Bad Traffic`) |
| `data.alert.severity` | Severity level (1=high, 3=low) |
| `data.src_ip` | Source IP address |
| `data.dest_ip` | Destination IP address |
| `data.proto` | Protocol (TCP, UDP, ICMP) |
| `data.flow.community_id` | Unique flow fingerprint for cross-tool correlation |
| `rule.id` | Wazuh rule ID (86601 = Suricata alert) |

**Screenshot — Full alert expanded in Wazuh dashboard:**

![Full Suricata alert expanded in Wazuh dashboard](../../assets/wazuh-suricata-alert-expanded.png)

---

## Config Files Changed

### 1. `/etc/suricata/suricata.yaml` — Ubuntu Linux VM

This is Suricata's main configuration file. Two changes were made here.

**How to open it:**

```bash
sudo nano /etc/suricata/suricata.yaml
```

---

**Change 1 — Set the network interface (`af-packet` section)**

The `af-packet` section tells Suricata which network interface to sniff on. To find it quickly in nano:

```
Ctrl+W  →  type: af-packet  →  Enter
```

This jumps you to the section. It should already look like this — confirm `ens33` is set:

```yaml
# Linux high speed capture support
af-packet:
  - interface: ens33
```

If it says `eth0` or something else, change it to match your interface name from `ip a`.

**Why:** Without this, Suricata doesn't know which network interface to listen on and captures nothing.

---

**Change 2 — Enable Community ID**

To find the community-id setting in nano:

```
Ctrl+W  →  type: community-id  →  Enter
```

Change this line:

```yaml
community-id: false
```

To:

```yaml
community-id: true
```

**Why:** Enables a unique fingerprint on every network flow so alerts from Suricata and Wazuh can be correlated by the same flow ID in the dashboard.

---

**Save and exit nano:**

```
Ctrl+X  →  Y  →  Enter
```

---

### 2. `/var/ossec/etc/ossec.conf` — Ubuntu Linux VM

This is the Wazuh agent's configuration file. One block was added here to tell the agent to read Suricata's output file.

**How to open it:**

```bash
sudo nano /var/ossec/etc/ossec.conf
```

**Finding the right place to add the block:**

The new block must go just before the very last line of the file (`</ossec_config>`). To jump there in nano:

```
Ctrl+W  →  type: </ossec_config>  →  Enter
```

Place your cursor on that closing tag line and add the following block **above** it:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>

</ossec_config>
```

The final lines of the file should look exactly like this when done:

```xml
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/kern.log</location>
  </localfile>

  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>

</ossec_config>
```

**Why:** The `<localfile>` block registers `eve.json` as a monitored log source. `log_format: json` tells the agent to parse each line as structured JSON, preserving all Suricata fields (src_ip, dest_ip, signature, community_id, etc.) as searchable fields in the Wazuh dashboard. Without this block, the Wazuh agent ignores `eve.json` entirely.

**Save and exit nano:**

```
Ctrl+X  →  Y  →  Enter
```

**Restart the Wazuh agent to apply the change:**

```bash
sudo systemctl restart wazuh-agent
```

---

### Summary Table

| File | VM | What changed | Why |
|---|---|---|---|
| `/etc/suricata/suricata.yaml` | Ubuntu Linux | `interface: ens33` confirmed in `af-packet` section | Tells Suricata which interface to sniff |
| `/etc/suricata/suricata.yaml` | Ubuntu Linux | `community-id: false` → `true` | Enables flow fingerprinting for cross-tool correlation |
| `/var/ossec/etc/ossec.conf` | Ubuntu Linux | Added `<localfile>` block for `eve.json` | Tells Wazuh agent to read and forward Suricata alerts |

> **No changes needed on the Wazuh server.** The server handles Suricata's JSON format natively via built-in decoder rules. All config changes are agent-side only.

---

## Key Takeaways

- Suricata operates at the **network layer** — it sees traffic before it even reaches the host OS
- One Suricata instance on the Ubuntu VM provides **full coverage** of the virtual network
- The `community-id` field enables **cross-tool correlation** between Suricata network alerts and Wazuh host alerts from other agents
- Suricata's alert flow: `eve.json` → Wazuh agent → Wazuh server → dashboard
- No Wazuh server config changes are needed — the server handles Suricata's JSON format natively via built-in decoders
