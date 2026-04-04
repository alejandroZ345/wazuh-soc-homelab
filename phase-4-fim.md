# Phase 4: File Integrity Monitoring (FIM)

> Objective: Configure the Wazuh `syscheck` module on the `Ubuntu_Admin` agent to establish real-time cryptographic monitoring over critical Linux system directories (`/etc`, `/bin`, `/usr/bin`). Validate the telemetry pipeline by simulating a persistent threat actor creating and modifying a backdoor configuration file.

---

## 1. Overview

File Integrity Monitoring is one of the most direct detections available against post-exploitation activity. After an attacker gains initial access, their next steps — dropping persistence scripts, modifying system binaries, or altering user databases — all involve touching the filesystem. FIM detects these actions by computing cryptographic hashes of files at baseline and alerting whenever those hashes change.

This phase configures real-time FIM on the `Ubuntu_Admin` agent, then validates it by simulating exactly the kind of filesystem manipulation a threat actor would perform to establish persistence.

---

## 2. Agent FIM configuration — `Ubuntu_Admin`

The Wazuh `syscheck` module is configured per-agent inside `ossec.conf`. The following directive was injected to enable real-time monitoring with full attribute checking and diff reporting on three critical directories.

### File edited

```
/var/ossec/etc/ossec.conf
```

### Directive added

Inject the following XML string inside the existing `<syscheck>` block:

```xml
<directories check_all="yes" realtime="yes" report_changes="yes">/etc,/bin,/usr/bin</directories>
```

**Attribute breakdown:**

| Attribute | Value | Effect |
|---|---|---|
| `check_all` | `yes` | Monitors permissions, ownership, size, inode, MD5, SHA1, and SHA256 |
| `realtime` | `yes` | Uses inotify for immediate event reporting instead of scheduled scans |
| `report_changes` | `yes` | Captures and reports the exact content diff of modified text files |

### Service reload

Restart the agent to apply the new syscheck baseline and begin real-time monitoring:

```bash
sudo systemctl restart wazuh-agent
```

> After restart, Wazuh performs an initial baseline scan of all monitored directories. Allow 1–2 minutes for this to complete before running threat simulations — alerts triggered during the baseline scan may generate noise.

---

## 3. Threat simulation — configuration tampering

A persistence scenario was simulated where a threat actor drops a malicious configuration file into `/etc` and subsequently modifies it to alter its cryptographic fingerprint.

### Execution — on the `Ubuntu_Admin` endpoint

```bash
# 1. Create the initial backdoor file in the monitored /etc directory
sudo touch /etc/soc_backdoor.conf

# 2. Inject the initial payload
echo "allow_root_access=true" | sudo tee /etc/soc_backdoor.conf

# 3. Modify the file to alter its cryptographic hash (simulates payload update)
echo "allow_root_access=true; reverse_shell=enabled" | sudo tee /etc/soc_backdoor.conf
```

**Why three steps?** This sequence exercises three distinct FIM event types in one simulation: file creation, initial write, and subsequent modification — each producing a different alert that would appear in a real incident timeline.

---

## 4. Telemetry validation & forensic analysis

### Dashboard path

`Wazuh Dashboard → File Integrity Monitoring → Events`

### Alerts observed

The SIEM successfully captured the full lifecycle of the tampered file on the `Ubuntu_Admin` agent:

| Timestamp | Agent | Path | Event | Rule ID | Level |
|---|---|---|---|---|---|
| Apr 2, 2026 @ 16:11:32 | Ubuntu_Admin | `/etc/soc_backdoor.conf` | added | 554 | 5 |
| Apr 2, 2026 @ 16:11:35 | Ubuntu_Admin | `/etc/soc_backdoor.conf` | modified | 550 | 7 |
| Apr 2, 2026 @ 16:11:43 | Ubuntu_Admin | `/etc/soc_backdoor.conf` | modified | 550 | 7 |
| Apr 2, 2026 @ 16:11:46 | Ubuntu_Admin | `/etc/soc_backdoor.conf` | modified | 550 | 7 |

### Rule reference

| Rule ID | Description | Level | ATT&CK technique |
|---|---|---|---|
| 554 | File added to the system | 5 | T1565 — Data manipulation |
| 550 | Integrity checksum changed | 7 | T1565 — Data manipulation |

### Conclusion

The FIM module successfully detected unauthorized file creation and subsequent integrity violations in real time. The dashboard provided forensic visibility into the exact timestamp and nature of each filesystem change — which in a real incident would allow an analyst to reconstruct the attacker's precise sequence of actions on the endpoint.

---

## 5. Key technical decisions

**Why monitor `/etc`, `/bin`, and `/usr/bin`?** These are the three most critical directories for detecting post-exploitation activity on Linux. `/etc` contains system and service configuration files — a common target for persistence (cron jobs, SSH authorized keys, passwd modification). `/bin` and `/usr/bin` contain system binaries — replacing or injecting code here is a classic technique for maintaining persistent access or escalating privileges.

**Why `realtime="yes"` instead of scheduled scans?** Wazuh's default syscheck behavior is to run periodic scans (every 12 hours by default). For a SOC lab validating detection capabilities, real-time monitoring via inotify is essential — it demonstrates that the SIEM can detect an attacker's filesystem activity within seconds of it occurring, not hours later.

**Why `report_changes="yes"`?** This attribute enables Wazuh to capture the actual content diff of modified files (stored in `/var/ossec/queue/diff/`). In a real incident, knowing *what changed* inside a file — not just *that it changed* — is the difference between a low-confidence alert and an actionable forensic finding.

---

*Previous: [Phase 3 — Linux agent & active threat simulation](../phase-3-threat-simulation/) · Next: [Phase 5 — Custom detection engineering & rule tuning](../phase-5-custom-rules/)*
