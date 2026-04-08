# Runbook: Rule 100003 — Unauthorized `sudo` abuse

> **Trigger:** Wazuh Alert `100003` — Level 8  
> **Description:** `WARNING: Multiple failed sudo execution attempts detected. Potential privilege escalation probing.`  
> **ATT&CK mapping:** T1548.003 — Abuse Elevation Control Mechanism: Sudo and Sudo Caching (Privilege Escalation)  
> **Parent SID:** 5404 (three consecutive failed `sudo` authentication attempts — aggregated by OS)

---

## Context

Rule `100003` inherits from Wazuh SID `5404`, which fires when Ubuntu logs three consecutive failed `sudo` authentication attempts by a single user. This pattern indicates either a legitimate user who forgot their password or an adversary who has gained low-privilege access and is attempting to escalate to root.

The alert fires at Level 8 (warning) because isolated `sudo` failures have a higher legitimate false positive rate than network-layer attacks. However, when combined with other indicators (Discovery commands triggering Rule 100004, or reverse shell activity triggering Rule 100005), failed `sudo` attempts become a strong signal of active post-compromise privilege escalation.

---

## Triage steps

### Step 1 — Identify the user and context

Review the `data.srcuser` or `full_log` field in the Wazuh alert to extract the username:

```bash
# Review specific sudo failures around the alert timestamp
grep "sudo" /var/log/auth.log | grep "authentication failure" | tail -20
```

### Step 2 — Assess the user's legitimacy

Determine whether the user account is expected to use `sudo`:

```bash
# Check if the user is in the sudo group
groups <username>

# Check sudoers configuration
sudo grep -r "<username>" /etc/sudoers /etc/sudoers.d/ 2>/dev/null
```

If the user is **not** in the sudo group and should not have elevated privileges, this is a strong indicator of an adversary testing credentials — escalate immediately.

### Step 3 — Correlate with session origin

Determine how the user's session was established:

```bash
# Check where the user logged in from
last <username> | head -10

# Check current active sessions
w
```

Red flags: login from an unexpected IP, login at unusual hours, session established via SSH from an unknown source, multiple users failing `sudo` in quick succession.

### Step 4 — Cross-reference with other alerts

Check the Wazuh dashboard for related alerts within the same time window from the same agent:

| Correlated alert | Implication | Escalation |
|---|---|---|
| Rule 100002 (SSH brute-force) | Adversary brute-forced entry, now attempting escalation | High — active compromise |
| Rule 100004 (Discovery commands) | Classic post-compromise enumeration → escalation pattern | High — kill chain progression |
| Rule 100005 (Reverse shell) | Active C2 channel; sudo probing is part of broader compromise | Critical — full compromise |

The combination of Discovery + sudo failure is a strong indicator of an active adversary in the privilege escalation phase of the kill chain.

### Step 5 — Check for successful escalation

```bash
# Check for successful sudo executions by the same user
grep "sudo.*<username>" /var/log/auth.log | grep -v "authentication failure"
```

If a successful `sudo` execution follows the failures, the adversary may have guessed the password — treat as a full compromise.

### Step 6 — Containment

**If the user account is confirmed compromised or unauthorized:**

```bash
# Lock the account
sudo usermod -L <username>

# Kill all active sessions for the user
sudo pkill -u <username>

# Force password expiry
sudo passwd -e <username>
```

If the `sudo` failures are part of a broader attack chain, execute containment in parallel with the corresponding runbooks for other triggered alerts.

### Step 7 — Post-incident documentation

Record: username, number of failed attempts, time window, session origin (IP and method), whether escalation eventually succeeded, correlated alerts, and containment actions.

---

## False positive guidance

Common benign triggers include legitimate administrators mistyping their password (especially after a password change), users unfamiliar with `sudo` entering incorrect credentials, and automated scripts running `sudo` commands with stale passwords.

If a specific user consistently triggers false positives, consider adding a monitored exception rather than lowering the rule's sensitivity.

---

*Part of the [Wazuh SIEM Home Lab](../README.md) project · Rule definition in [Phase 5 § 4](../phase-5-custom-rules.md#4-final-xml-configuration) · ATT&CK mapping in [detections/](../detections/mitre-attack-map.md)*
