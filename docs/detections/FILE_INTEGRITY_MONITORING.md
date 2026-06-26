# File Integrity Monitoring (FIM)

This document covers how File Integrity Monitoring is configured in this lab, what it detects, and how alerts are verified in the Wazuh dashboard.

---

## What is FIM?

File Integrity Monitoring tracks changes to files and directories on a monitored system. Wazuh's FIM engine — called `syscheck` — computes a cryptographic checksum of every file in a monitored directory and compares it against a stored baseline. When a file is created, modified, or deleted, the checksum changes, and Wazuh raises an alert.

In a real SOC environment, FIM is used to detect:

- Attackers dropping malicious files onto a compromised host
- Tampering with sensitive system files like `/etc/passwd` or `/etc/sudoers`
- Unauthorized changes to configuration files

In this lab, FIM is configured to monitor the home directory of the Ubuntu Linux agent VM, as well as key system paths that are monitored by default.

---

## How It Works

Wazuh's `syscheck` module runs on the agent and periodically scans monitored directories. When `realtime` monitoring is enabled, inotify kernel events are used instead of scheduled scans — meaning alerts fire the moment a change occurs rather than waiting for the next scan cycle.

The agent detects the change, computes a new checksum, compares it to the stored baseline, and forwards an alert to the Wazuh manager. The manager matches the alert against its ruleset and surfaces it on the dashboard.

```
File change on Ubuntu Linux VM
        ↓
syscheck detects via inotify (realtime)
        ↓
Wazuh agent computes new checksum
        ↓
Alert forwarded to Wazuh manager
        ↓
Rule matched → Alert on dashboard
```

---

## Configuration

FIM is configured in `ossec.conf` on **both** the Wazuh manager and the Wazuh agent. This is a critical detail — configuring it only on the manager side silently fails with no error. The agent needs its own `ossec.conf` updated to know which directories to watch.

---

### Wazuh Manager — `/var/ossec/etc/ossec.conf`

The `syscheck` block on the manager side was already enabled by default (`<disabled>no</disabled>`). The only change made was adding the custom directory entry below the existing default directories:

```xml
<!-- File integrity monitoring -->
<syscheck>
    <disabled>no</disabled>

    <!-- Frequency that syscheck is executed default every 12 hours -->
    <frequency>43200</frequency>

    <scan_on_start>yes</scan_on_start>

    <!-- Generate alert when new file detected -->
    <alert_new_files>yes</alert_new_files>

    <!-- Don't ignore files that change more than 'frequency' times -->
    <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>

    <!-- Directories to check (perform all possible verifications) -->
    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>
    <directories realtime="yes">/home/labuser321</directories>   <!-- Added -->

    ...
</syscheck>
```

The `realtime="yes"` attribute enables inotify-based monitoring, so file events trigger alerts immediately instead of waiting for the 12-hour scheduled scan.

---

### Wazuh Agent — `/var/ossec/etc/ossec.conf` on Ubuntu Linux VM

The same directory entry must also be added to the agent's local `ossec.conf`. Without this, the agent does not watch the directory regardless of what the manager config says — this is the key gotcha with Wazuh FIM.

```xml
<!-- File integrity monitoring -->
<syscheck>
    <disabled>no</disabled>

    <frequency>43200</frequency>

    <scan_on_start>yes</scan_on_start>

    <alert_new_files>yes</alert_new_files>

    <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>

    <!-- Directories to check -->
    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>
    <directories realtime="yes">/home/labuser321</directories>   <!-- Added -->

    ...
</syscheck>
```

After editing both config files, the Wazuh manager and agent were restarted to apply the changes:

```bash
# On Wazuh Server VM
sudo systemctl restart wazuh-manager

# On Ubuntu Linux VM
sudo systemctl restart wazuh-agent
```

---

## Rules and Decoders

No custom rules or decoders were written for FIM — Wazuh ships with built-in syscheck rules out of the box. These rules live at `/var/ossec/ruleset/rules/0016-wazuh_syscheck.xml` on the manager and fire automatically whenever the syscheck module reports a file event.

The decoder side is similarly built-in. The `wazuh_syscheck` decoder parses the structured FIM event sent by the agent — extracting fields like `syscheck.path`, `syscheck.event`, `syscheck.md5_before`, `syscheck.md5_after` — and makes them available for rule matching and dashboard display.

The relevant built-in rules that fired during this lab:

| Rule ID | Event Type | Description | Severity | Rule File |
|---|---|---|---|---|
| 554 | `added` | File added to the system | 5 | `0016-wazuh_syscheck.xml` |
| 550 | `modified` | Integrity checksum changed | 7 | `0016-wazuh_syscheck.xml` |
| 553 | `deleted` | File deleted | 7 | `0016-wazuh_syscheck.xml` |

Rule severity 7 for modified and deleted is intentionally higher than 5 for added — modifications and deletions on monitored paths are considered more suspicious than new file creation.

---

## Detection Verification

Four scenarios were tested to verify FIM was working end to end — three event types on a test file in the home directory, and one sensitive system file tamper simulation.

---

### 1. File Created — `testfile1.txt`

A new file was created inside the monitored home directory:

```bash
# On Ubuntu Linux VM
touch /home/labuser321/testfile1.txt
```

Wazuh fired rule **554** — "File added to the system" — with event type `added`, severity level 5.

<!-- SCREENSHOT: Dashboard showing /home/labuser321/testfile1.txt added event, rule 554 -->
![FIM file added alert](../../assets/fim-file-added.png)

---

### 2. File Modified — `testfile1.txt`

The same file was then modified by writing content into it:

```bash
# On Ubuntu Linux VM
echo "test content" >> /home/labuser321/testfile1.txt
```

Wazuh fired rule **550** — "Integrity checksum changed" — with event type `modified`, severity level 7. The checksum of the file changed because its contents changed.

<!-- SCREENSHOT: Dashboard showing /home/labuser321/testfile1.txt modified event, rule 550 -->
![FIM file modified alert](../../assets/fim-file-modified.png)

---

### 3. File Deleted — `testfile1.txt`

The file was then deleted:

```bash
# On Ubuntu Linux VM
rm /home/labuser321/testfile1.txt
```

Wazuh fired rule **553** — "File deleted" — with event type `deleted`, severity level 7.

<!-- SCREENSHOT: Dashboard showing /home/labuser321/testfile1.txt deleted event, rule 553 -->
![FIM file deleted alert](../../assets/fim-file-deleted.png)

---

### 4. Sensitive File Tamper — `/etc/passwd`

To simulate a realistic attacker scenario, `/etc/passwd` was opened in nano and saved. In a real attack, an adversary who has gained a foothold would edit this file to add a backdoor user or modify an existing account for privilege escalation. Even opening and saving the file without any content change is enough to alter its metadata and trigger a checksum mismatch.

```bash
# On Ubuntu Linux VM
sudo nano /etc/passwd
# File opened and saved with Ctrl+O → Enter → Ctrl+X
```

Wazuh fired rule **550** — "Integrity checksum changed" — on `/etc/passwd`, severity level 7.

<!-- SCREENSHOT: Dashboard showing /etc/passwd modified event, rule 550 -->
![FIM passwd alert](../../assets/fim-passwd-modified.png)

This is a meaningful detection from a SOC perspective — `/etc/passwd` modifications are a well-known indicator of privilege escalation and persistence attempts (MITRE ATT&CK T1098, T1136).

---

## Key Observations

- `realtime="yes"` is essential — without it FIM only runs on the 12-hour scheduled scan, making it completely useless for live detection
- The directory rule must be present in **both** the manager and agent `ossec.conf` — manager-only config silently fails with no error, which is easy to miss
- No custom rules or decoders were needed — Wazuh's built-in syscheck ruleset handles all FIM event types out of the box
- Wazuh also catches background OS activity within monitored paths — snap package updates, CUPS config changes, GNOME tracker cache writes — showing how noisy realtime FIM can be in practice; scoping monitored directories carefully matters in production
- All three event types (added, modified, deleted) confirmed working as expected on `/home/labuser321`
- Sensitive file monitoring on `/etc/passwd` confirmed working, mapping directly to real-world attacker TTPs
