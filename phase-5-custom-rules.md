# Phase 5: Custom detection engineering & rule tuning (XML rules)

> Objective: Enhance the SIEM's default detection capabilities by authoring custom XML rules tailored to specific threat vectors. This phase documents the complete detection engineering lifecycle: initial deployment, attack simulation, raw log analysis, and rule tuning to adapt to the specific OS environment.

---

## 1. Overview

Out-of-the-box SIEM rules cover broad patterns, but they are rarely optimized for a specific operating system, authentication stack, or privilege model. This phase moves beyond passive monitoring into active detection engineering — writing rules that target exact behaviors observed in this lab environment, then iterating based on what the telemetry actually reveals.

The three threat vectors targeted are: unauthorized modification of `/etc/passwd`, high-frequency SSH brute-force from a single source, and repeated failed `sudo` execution attempts indicating privilege escalation probing.

---

## 2. Initial implementation & first test

Custom rules were drafted and deployed via the Wazuh Dashboard:

**Path:** `Server Management → Rules → Manage Rules → local_rules.xml`

### Initial rule targets

| Rule ID | Threat vector | Initial parent SID |
|---|---|---|
| 100001 | Critical modification of `/etc/passwd` (FIM) | 550 |
| 100002 | SSH brute-force | 5716 |
| 100003 | Unauthorized `sudo` abuse | 2502 |

### Active testing result

Attack simulations were executed against the `Ubuntu_Admin` agent:

- Rule 100001 triggered successfully on `/etc/passwd` modification.
- Rule 100002 failed to generate alerts despite the Hydra attack running.
- Rule 100003 failed to generate alerts despite failed `sudo` attempts.

---

## 3. Telemetry analysis & rule tuning (troubleshooting)

To determine why rules 100002 and 100003 failed, raw system logs and security events from the `Ubuntu_Admin` agent were analyzed in real time during the attack simulations.

### SSH brute-force finding — Hydra from Kali

Ubuntu handles SSH authentication via PAM (Pluggable Authentication Modules), which generates a different log footprint than expected.

| Expected SID | Actual SID triggered | Description |
|---|---|---|
| 5716 | **5760** | `sshd: authentication failed` |

The rule was targeting a parent SID that Ubuntu's PAM stack never fires. The correlation engine never had a matching base event to count against.

### Sudo abuse finding — `sudo -k`

A deeper dive into the authentication logs showed that a single failed password attempt did not match the generic syslog rule (`2502`) cleanly. Instead, Ubuntu explicitly logs "three failed attempts to run sudo", which Wazuh's decoder natively translates into SID `5404`.

| Expected SID | Actual SID triggered | Description |
|---|---|---|
| 2502 | **5404** | Multiple failed `sudo` execution attempts |

### Action taken

The XML logic for both rules was updated to inherit from the environment-accurate parent SIDs discovered through log analysis, ensuring reliable triggering on this specific OS configuration.

---

## 4. Final XML configuration

The optimized payload was deployed to `/var/ossec/etc/rules/local_rules.xml`. The Manager was restarted via Docker to ensure a clean rule compilation:

```bash
docker compose restart
```

```xml
<group name="custom_rules,">

  <!-- Rule 100001: Critical /etc/passwd modification -->
  <rule id="100001" level="12">
    <if_sid>550</if_sid>
    <match>/etc/passwd</match>
    <description>CRITICAL: /etc/passwd has been modified. Potential persistence or user creation detected.</description>
    <mitre>
      <id>T1078</id>
    </mitre>
  </rule>

  <!-- Rule 100002: High-frequency SSH brute-force from same source IP -->
  <rule id="100002" level="10" frequency="5" timeframe="60">
    <if_matched_sid>5760</if_matched_sid>
    <same_source_ip />
    <description>ALERT: High frequency SSH brute force attack detected from the same Source IP.</description>
    <mitre>
      <id>T1110.001</id>
    </mitre>
  </rule>

  <!-- Rule 100003: Multiple failed sudo execution attempts -->
  <rule id="100003" level="8">
    <if_sid>5404</if_sid>
    <description>WARNING: Multiple failed sudo execution attempts detected. Potential privilege escalation probing.</description>
    <mitre>
      <id>T1548.003</id>
    </mitre>
  </rule>

</group>
```

### Rule logic breakdown

**Rule 100001** inherits from SID `550` (FIM: integrity checksum changed) and adds a `<match>` filter on the `/etc/passwd` path to narrow the scope from any FIM event to specifically this critical file. Level 12 marks it as a critical alert.

**Rule 100002** uses `<if_matched_sid>` with frequency and timeframe to implement correlation: it only fires when SID `5760` is seen 5 or more times within 60 seconds from the same source IP. The `<same_source_ip />` tag is what makes this a coordinated brute-force detection rather than a single-failure alert.

**Rule 100003** inherits from SID `5404`, which Wazuh fires natively when Ubuntu logs three consecutive failed `sudo` attempts. No additional correlation logic is needed — the OS itself aggregates the failures.

---

## 5. Validation confirmation

A final round of active threat simulations was conducted after deploying the tuned rules. All three vectors were successfully captured and escalated in the Security Events dashboard.

| Rule | Trigger condition | Alert level | Validated |
|---|---|---|---|
| 100001 | `echo "# SOC Rule Test Validation" \| sudo tee -a /etc/passwd` executed | 12 — Critical | Yes |
| 100002 | Hydra SSH dictionary attack launched from Kali | 10 — Alert | Yes |
| 100003 | Multiple consecutive failed `sudo` login attempts | 8 — Warning | Yes |

### Alert evidence

**Rule 100001** fired at `Apr 3, 2026 @ 15:46:10` — the instant the `tee -a /etc/passwd` command touched the file.

**Rule 100002** fired at `Apr 3, 2026 @ 16:10:42` alongside the base SID `5760` events from Hydra, confirming the correlation engine counted the frequency threshold correctly.

**Rule 100003** fired at `Apr 3, 2026 @ 17:11:15`, sandwiched between individual `PAM: User login failed` (SID `5503`) events — exactly the pattern expected when a user fails `sudo` authentication three times in succession.

---

## 6. Key technical decisions

**Why tune rules against real telemetry instead of assuming SIDs?** Documentation and community guides often reference generic SID numbers that vary across OS distributions. Ubuntu's PAM authentication stack produces different Wazuh event IDs than RedHat/CentOS-based systems. Discovering SID `5760` through actual log analysis rather than copying documentation is the correct approach — and the documented troubleshooting process demonstrates exactly the kind of analytical mindset expected in a SOC analyst role.

**Why use `<if_matched_sid>` with frequency/timeframe for the brute-force rule instead of `<if_sid>`?** `<if_sid>` fires on every single matching event. `<if_matched_sid>` with a frequency/timeframe window creates a stateful correlation rule — it only alerts after the threshold is crossed, which eliminates false positives from legitimate failed login attempts while reliably catching automated dictionary attacks.

**Why level 12 for the `/etc/passwd` rule?** Wazuh levels above 10 are considered high severity and will appear prominently in the Security Events dashboard and trigger email/webhook notifications if configured. Modification of `/etc/passwd` is one of the clearest indicators of persistence or privilege escalation activity on a Linux system — assigning it the highest custom level reflects that severity accurately.

---

*Previous: [Phase 4 — File Integrity Monitoring](./phase-4-fim.md/) · Next: [Phase 6 — MITRE ATT&CK complete mapping](./phase-6-mitre-mapping.md/)*
