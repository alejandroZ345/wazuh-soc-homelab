# Runbook: Rule 100004 — System Discovery commands

> **Trigger:** Wazuh Alert `100004` — Level 7  
> **Description:** `WARNING: System discovery command executed.`  
> **ATT&CK mapping:** T1087 — Account Discovery / T1082 — System Information Discovery (Discovery)  
> **Detection method:** `bash_audit` syslog pipeline — matches `whoami`, `id`, `uname`

---

## Context

Rule `100004` fires when the `bash_audit` user-space auditing pipeline logs the execution of common reconnaissance commands. These are the first commands an adversary runs after gaining a foothold — answering "who am I?", "what privileges do I have?", and "what system is this?".

This rule is set at Level 7 (warning) because `whoami`, `id`, and `uname` have entirely legitimate uses. The alert's value comes from correlation — a Discovery command followed by `sudo` probing (Rule 100003) or a reverse shell attempt (Rule 100005) is a clear indicator of an active adversary progressing through the kill chain.

---

## Triage steps

### Step 1 — Extract the executed command

Review the `full_log` field in the Wazuh alert. The `bash_audit` pipeline logs the format `<user> ran: <command>`. Identify the username and the specific Discovery command.

### Step 2 — Assess context and timing

A single `whoami` from a known administrator during business hours is almost certainly benign. Look for adversary patterns:

**Red flags:** multiple Discovery commands in rapid succession (`whoami` → `id` → `uname -a` within seconds), commands executed by a user who does not normally use the terminal, commands from a session established via unusual method, timing outside normal administrative hours.

### Step 3 — Correlate with other alerts (most important step)

Check the Wazuh dashboard for related alerts from the same agent within a ±5 minute window:

| Correlated alert | Implication | Escalation |
|---|---|---|
| Rule 100002 (SSH brute-force) | Adversary brute-forced entry, now enumerating | High — active compromise |
| Rule 100003 (sudo failures) | Discovery → privilege escalation attempt | High — kill chain progression |
| Rule 100005 (reverse shell) | Discovery → C2 establishment | Critical — full compromise |
| Rule 100001 (/etc/passwd mod) | Discovery → persistence | Critical — full compromise |
| No correlated alerts | Likely benign administrative use | Low — document and close |

If correlated with any attack-chain alert, treat the Discovery activity as confirmation of an active adversary and follow the corresponding runbook for the higher-severity alert.

### Step 4 — Session origin investigation

If the context warrants further investigation:

```bash
# Who is currently logged in
w

# Check the process tree for the command's origin
ps -ef --forest | grep <username>

# Review recent SSH logins
last <username> | head -5
```

### Step 5 — Containment

Discovery commands alone rarely require containment — they are indicators, not damage. Containment should be driven by the correlated higher-severity alerts.

If Discovery is the only alert and the session origin is suspicious, monitor the user's activity in real time rather than immediately terminating:

```bash
sudo tail -f /var/log/auth.log | grep <username>
```

If subsequent escalation attempts are observed, proceed to containment as defined in the relevant runbook.

### Step 6 — Post-incident documentation

Record: command executed, username, session origin, timestamp, correlated alerts (if any), and disposition (benign vs. part of attack chain).

---

## False positive guidance

This rule has the highest false positive rate of all custom rules. Common benign triggers include administrators verifying their context before privileged commands, shell initialization scripts (`.bashrc`, `.profile`) calling `uname` or `id` on login, and monitoring tools executing system information commands periodically.

If a specific user or automated process generates persistent noise, consider creating a targeted suppression rule using `<if_sid>100004</if_sid>` with a `<user>` filter. Ensure the suppression is documented and reviewed periodically.

---

*Part of the [Wazuh SIEM Home Lab](../README.md) project · Rule definition in [Phase 5 § 6](../phase-5-custom-rules.md#6-advanced-detection-engineering--the-pyramid-of-pain) · ATT&CK mapping in [detections/](../detections/mitre-attack-map.md)*
