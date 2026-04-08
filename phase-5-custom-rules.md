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

## 6. Advanced detection engineering — the "Pyramid of Pain"

### Evolution of detection strategy: from fragile signatures to behavioral TTPs

Following the successful deployment of basic XML rules, the objective shifted to detecting post-compromise reconnaissance (Discovery) and the establishment of Command & Control (Reverse Shells) via the Linux terminal.

### Initial approach — signature-based

The first iteration attempted to create highly specific rules for every known reverse shell syntax (Bash, Python, Perl, Socat, etc.) using complex `<pcre2>` expressions and the `<match>` tag.

### The problem — API Error 1113

This approach led to continuous `1113 - XML syntax error` failures in the Wazuh Web UI. Intensive troubleshooting revealed two core limitations:

**Validator strictness:** The Wazuh Node.js Web UI schema strictly rejects nested alternations `(A|B)` within `<pcre2>` groups, causing compilation panics despite the XML being logically sound.

**Decoder misalignment:** The `<match>` tag strictly evaluates the *body* of a log. Because the syslog workaround injected the `bash_audit` identifier into the *header* (program name) rather than the payload, the rules failed to trigger during live testing.

### Architectural shift — the Pyramid of Pain

Instead of playing "whack-a-mole" by writing fragile signatures for specific scripting languages (which adversaries can easily obfuscate), the detection strategy was elevated to target **TTPs (Tactics, Techniques, and Procedures)**. The logic was consolidated into two broad, behavioral rules: one for Discovery commands and one for generic network-socket syntax within the terminal.

### Final broad-spectrum XML configuration

To bypass the UI validator bugs and align with the syslog decoder, the `<pcre2>` tags were replaced with the universally supported OSSEC `<regex>` engine (without nested groups), and the `<match>` tag was replaced with `<program_name>`.

```xml
<!-- Rule 100004: Terminal-based system discovery commands -->
<rule id="100004" level="7">
  <if_group>syslog</if_group>
  <program_name>bash_audit</program_name>
  <regex>ran: whoami|ran: id|ran: uname</regex>
  <description>WARNING: System discovery command executed.</description>
  <mitre><id>T1087</id></mitre>
</rule>

<!-- Rule 100005: Suspicious network commands — potential reverse shell -->
<rule id="100005" level="12">
  <if_group>syslog</if_group>
  <program_name>bash_audit</program_name>
  <regex>dev/tcp|import socket|nc -|netcat|socat|fsockopen</regex>
  <description>CRITICAL: Suspicious network command executed. Potential Reverse Shell.</description>
  <mitre><id>T1095</id></mitre>
</rule>
```

### Script Used for Monitoring

To achieve full command-line visibility without the overhead of recompiling the WSL2 kernel, a user-space auditing workaround was designed and implemented. The strategy leverages the native syslog daemon to capture terminal activity.

The following script was injected into the global Bash configuration file (/etc/bash.bashrc) on the monitored endpoints:

```bash
# Injecting the auditing command into the global bashrc
echo 'export PROMPT_COMMAND='\''logger -p user.info -t bash_audit "$USER ran: $(history 1 | sed "s/^[ ]*[0-9]\+[ ]*//")"'\''' | sudo tee -a /etc/bash.bashrc

# Reloading the environment for the current session
source /etc/bash.bashrc
```

**Script Breakdown**

`PROMPT_COMMAND`: A native Bash environment variable that executes a specified command immediately before printing the primary prompt ($PS1). This ensures the logging occurs exactly after a user executes a command.

`history 1`: Retrieves the most recently executed command from the session's history buffer.

`sed "s/^[ ]*[0-9]\+[ ]*//"`: A regular expression pipeline that strips the leading spaces and history index numbers, isolating the raw command payload.

`logger -p user.info -t bash_audit`: Forwards the sanitized command string to `/var/log/syslog` under the `user.info` facility, explicitly tagging the event with the identifier `bash_audit`.

### Rule logic breakdown

**Rule 100004** hooks into the `syslog` group and filters by the `bash_audit` program name — the custom user-space auditing pipeline that logs every terminal command. The `<regex>` pattern matches the most common Discovery commands (`whoami`, `id`, `uname`) that an attacker runs immediately after gaining a foothold. Level 7 keeps it as an informational warning since these commands also have legitimate uses.

**Rule 100005** uses the same `syslog` / `bash_audit` hook but targets network-socket primitives (`dev/tcp`, `import socket`, `nc -`, `netcat`, `socat`, `fsockopen`) that appear across virtually every reverse shell variant regardless of scripting language. Level 12 marks it critical because establishing an outbound shell is a clear post-compromise action with no legitimate use case in this environment.

### Validation confirmation

Live threat simulations triggered both alerts instantly:

| Rule | Trigger condition | Alert level | Validated |
|---|---|---|---|
| 100004 | `whoami` and `id` executed on `Ubuntu_Admin` | 7 — Warning | Yes |
| 100005 | Bash reverse shell (`/dev/tcp`) executed on `Ubuntu_Admin` | 12 — Critical | Yes |

**Rule 100004** fired at `Apr 7, 2026 @ 15:54:34` and `15:54:36` for consecutive Discovery commands.

**Rule 100005** fired at `Apr 7, 2026 @ 15:54:58` when the reverse shell syntax was executed, confirming the behavioral net successfully caught the C2 establishment attempt.

---

## 7. Key technical decisions

**Why tune rules against real telemetry instead of assuming SIDs?** Documentation and community guides often reference generic SID numbers that vary across OS distributions. Ubuntu's PAM authentication stack produces different Wazuh event IDs than RedHat/CentOS-based systems. Discovering SID `5760` through actual log analysis rather than copying documentation is the correct approach — and the documented troubleshooting process demonstrates exactly the kind of analytical mindset expected in a SOC analyst role.

**Why use `<if_matched_sid>` with frequency/timeframe for the brute-force rule instead of `<if_sid>`?** `<if_sid>` fires on every single matching event. `<if_matched_sid>` with a frequency/timeframe window creates a stateful correlation rule — it only alerts after the threshold is crossed, which eliminates false positives from legitimate failed login attempts while reliably catching automated dictionary attacks.

**Why level 12 for the `/etc/passwd` rule?** Wazuh levels above 10 are considered high severity and will appear prominently in the Security Events dashboard and trigger email/webhook notifications if configured. Modification of `/etc/passwd` is one of the clearest indicators of persistence or privilege escalation activity on a Linux system — assigning it the highest custom level reflects that severity accurately.

**Why `<regex>` over `<pcre2>` for rules 100004/100005?** The Wazuh Web UI's XML validator rejects nested alternation groups in `<pcre2>`, causing Error 1113 even when the regex is logically valid. The OSSEC `<regex>` engine supports pipe-delimited alternation natively and compiles without issues, making it the pragmatic choice for rules that need to match multiple patterns in a single expression.

**Why `<program_name>` instead of `<match>` for the bash_audit rules?** The `<match>` tag evaluates only the *body* of the syslog message. Because the custom auditing pipeline injects `bash_audit` as the program name in the syslog header, `<program_name>` correctly targets the identifier while `<regex>` handles the payload matching. This two-field approach ensures both the source and the content are validated before the rule fires.

**Why behavioral TTP rules instead of per-language signatures?** An adversary can trivially obfuscate a Python reverse shell or switch to Perl — but they cannot avoid using socket primitives or network file descriptors. By targeting the underlying behavior rather than specific syntax, rules 100004 and 100005 are resilient to evasion techniques that would defeat signature-based detection. This aligns with the Pyramid of Pain concept: the higher up the pyramid (TTPs), the more costly it is for an adversary to evade detection.

---

*Previous: [Phase 4 — File Integrity Monitoring](./phase-4-fim.md/) · Next: [Phase 6 — MITRE ATT&CK complete mapping](./phase-6-mitre-mapping.md/)*
