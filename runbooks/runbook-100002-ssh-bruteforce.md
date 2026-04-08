# Runbook: Rule 100002 — High-frequency SSH brute force

> **Trigger:** Wazuh Alert `100002` — Level 10  
> **Description:** `ALERT: High frequency SSH brute force attack detected from the same Source IP.`  
> **ATT&CK mapping:** T1110.001 — Brute Force: Password Guessing (Credential Access)  
> **Parent SID:** 5760 (sshd: authentication failed) — correlation: ≥5 events in 60s from same source IP

---

## Context

Rule `100002` is a stateful correlation rule that fires when SID `5760` (PAM-based SSH authentication failure) is observed 5 or more times within a 60-second window from the same source IP. The `<same_source_ip />` constraint ensures this alert exclusively represents coordinated, single-origin attacks — not unrelated individual login failures.

By the time this rule fires, the threshold has already confirmed automated attack behavior (Hydra, Medusa, etc.). The triage focus is on determining whether any attempt succeeded and identifying the attacker's IP for containment.

---

## Triage steps

### Step 1 — Source IP extraction

The `<same_source_ip />` constraint means the alert itself confirms a single attacker. Extract the IP from the `srcip` field in the Wazuh Security Events dashboard.

Classify the source: is it internal (WSL2 shared network, `127.0.0.1`, known lab IPs) or external/unexpected?

### Step 2 — Volume assessment

Query the total event count for the identified source IP to gauge attack intensity:

**Wazuh Dashboard:** Filter `rule.id: 5760 AND data.srcip: <attacker_ip>` and review the event count and time distribution.

Hundreds or thousands of events in minutes with no successful authentications confirms an automated dictionary attack. A small cluster (5–10 events) may indicate a less sophisticated or manually-driven attempt.

### Step 3 — Success verification (critical)

Determine whether any authentication attempt from the attacker IP actually succeeded. This is the highest-priority step — a successful brute-force means the system is compromised.

**On the endpoint (Ubuntu_Admin):**

```bash
# Check for successful logins from the attacker IP
grep "Accepted" /var/log/auth.log | grep "<attacker_ip>"
```

**On the Wazuh Dashboard:** Search for `rule.id: 5715 AND data.srcip: <attacker_ip>` (SID 5715 = successful SSH login).

If a successful login is found, **escalate immediately** — the attacker has valid credentials. Proceed directly to Step 5 (compromised containment).

### Step 4 — Active session check

Verify whether the attacker currently has an active session:

```bash
# List all active SSH sessions
who
w

# Check for connections from the attacker IP
ss -antp | grep "<attacker_ip>"
```

### Step 5 — Containment

**If no successful login occurred (attack in progress or failed):**

```bash
# Block the attacker IP at the host firewall
sudo iptables -A INPUT -s <attacker_ip> -j DROP
```

For persistent blocking:

```bash
sudo ufw deny from <attacker_ip>
```

**If a successful login occurred (system compromised):**

1. Terminate the attacker's session:

```bash
ps -ef | grep "sshd:.*<attacker_ip>"
kill -9 <PID>
```

2. Block the IP (as above).
3. Force a password reset for the compromised account.
4. Audit `/etc/passwd` and `/etc/shadow` for new accounts (cross-reference with Rule 100001).
5. Check for SSH key persistence:

```bash
find /home -name "authorized_keys" -exec echo "--- {} ---" \; -exec cat {} \;
```

### Step 6 — Post-incident documentation

Record: attacker source IP, timestamp window of the attack, total event count (SID 5760), whether any login succeeded, targeted username(s), and containment actions taken.

---

## False positive guidance

This rule has a low false positive rate by design — the combination of frequency threshold (≥5), timeframe (60s), and same-source-IP constraint filters out most benign failures. However, the following scenarios may trigger it:

- Automated monitoring tools or CI/CD pipelines configured with stale SSH credentials that retry rapidly.
- Legitimate users with SSH key passphrase issues causing rapid re-authentication attempts.

For recurring false positives from a known internal IP, consider adding a targeted `<if_sid>` exception rule scoped to that specific source rather than adjusting the detection threshold.

---

*Part of the [Wazuh SIEM Home Lab](../README.md) project · Rule definition in [Phase 5 § 4](../phase-5-custom-rules.md#4-final-xml-configuration) · Brute-force simulation in [Phase 3](../phase-3-threat-simulation.md/) · ATT&CK mapping in [detections/](../detections/mitre-attack-map.md)*
