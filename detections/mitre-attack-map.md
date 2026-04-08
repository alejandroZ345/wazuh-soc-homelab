# MITRE ATT&CK detection coverage map

> Complete mapping of all Wazuh detections validated through live attack simulations across the lab phases.  
> Framework: MITRE ATT&CK® Enterprise — [attack.mitre.org](https://attack.mitre.org)  
> Last updated: Phase 6+

---

## Coverage summary

| Tactic | Techniques covered |
|---|---|
| Credential Access | T1110.001 |
| Persistence | T1078.003, T1565.001 |
| Privilege Escalation | T1548.003, T1078.003 |
| Defense Evasion | T1565.001 |
| Discovery | T1087, T1082 |
| Command and Control | T1095 |

---

## Full detection matrix

| Phase | Simulated threat vector | Wazuh rule ID | Type | Alert level | ATT&CK tactic | ATT&CK technique | Technique ID | Validated |
|---|---|---|---|---|---|---|---|---|
| Phase 3 | SSH dictionary attack (Hydra) | 5760 | Built-in | 5 | Credential Access | Brute Force: Password Guessing — individual failure | T1110.001 | Yes |
| Phase 3 | SSH dictionary attack correlation | 5712 | Built-in | 10 | Credential Access | Brute Force: Password Guessing — frequency threshold | T1110.001 | Yes |
| Phase 4 | Backdoor config file creation (`/etc/soc_backdoor.conf`) | 554 | Built-in | 5 | Persistence | Stored Data Manipulation — file drop | T1565.001 | Yes |
| Phase 4 | File integrity violation — checksum change (`/etc/`) | 550 | Built-in | 7 | Persistence / Defense Evasion | Stored Data Manipulation — content modification | T1565.001 | Yes |
| Phase 5 | Critical modification of `/etc/passwd` | 100001 | Custom | 12 | Persistence / Privilege Escalation | Valid Accounts: Local Accounts | T1078.003 | Yes |
| Phase 5 | High-frequency SSH brute-force (same source IP, ≥5 in 60s) | 100002 | Custom | 10 | Credential Access | Brute Force: Password Guessing | T1110.001 | Yes |
| Phase 5 | Multiple failed `sudo` execution attempts | 100003 | Custom | 8 | Privilege Escalation | Abuse Elevation Control: Sudo and Sudo Caching | T1548.003 | Yes |
| Phase 5+ | Terminal-based Discovery (`whoami`, `id`, `uname`) | 100004 | Custom | 7 | Discovery | Account Discovery / System Information Discovery | T1087 / T1082 | Yes |
| Phase 5+ | Reverse Shell establishment (Bash/Netcat/Socat) | 100005 | Custom | 12 | Command and Control | Non-Application Layer Protocol | T1095 | Yes |

---

## ATT&CK technique reference

| Technique ID | Full name | Tactic | MITRE link |
|---|---|---|---|
| T1110.001 | Brute Force: Password Guessing | Credential Access | [link](https://attack.mitre.org/techniques/T1110/001/) |
| T1565.001 | Data Manipulation: Stored Data Manipulation | Impact / Persistence | [link](https://attack.mitre.org/techniques/T1565/001/) |
| T1078.003 | Valid Accounts: Local Accounts | Persistence, Privilege Escalation, Defense Evasion | [link](https://attack.mitre.org/techniques/T1078/003/) |
| T1548.003 | Abuse Elevation Control Mechanism: Sudo and Sudo Caching | Privilege Escalation, Defense Evasion | [link](https://attack.mitre.org/techniques/T1548/003/) |
| T1087 | Account Discovery | Discovery | [link](https://attack.mitre.org/techniques/T1087/) |
| T1082 | System Information Discovery | Discovery | [link](https://attack.mitre.org/techniques/T1082/) |
| T1095 | Non-Application Layer Protocol | Command and Control | [link](https://attack.mitre.org/techniques/T1095/) |

---

## Mapping notes

**T1565.001 for FIM events (Phase 4):** The backdoor file creation and modification events were mapped to Stored Data Manipulation rather than T1105 (Ingress Tool Transfer) or T1070 (Indicator Removal on Host). T1105 applies to files transferred from external systems over a network channel. T1070 describes an adversary deleting evidence — the opposite of what occurred, where the SIEM detected the change. T1565.001 accurately describes an adversary inserting or altering data at rest to influence system behavior or establish persistence.

**Custom rules reduce alert fatigue:** Rules 100001–100003 were engineered with behavioral thresholds (frequency, timeframe, path matching) tuned to the environment's baseline. This means they fire on meaningful patterns rather than individual events, producing high-confidence alerts rather than noise.

**Dual tactic assignment:** Rule 100001 (`/etc/passwd` modification) is listed under both Persistence and Privilege Escalation because adding a new user to `/etc/passwd` can serve either purpose depending on the attacker's intent — both are valid contextual interpretations of the same indicator.

**Behavioral detection for Discovery and C2 (Phase 5+):** Rules 100004 and 100005 were designed using the Pyramid of Pain methodology — targeting TTPs rather than fragile per-language signatures. Rule 100005 in particular uses a broad behavioral net, which makes it resilient to evasion but also means it requires a triage runbook for each triggered alert. The operational procedure is documented in [`runbooks/runbook-100005-reverse-shell.md`](../runbooks/runbook-100005-reverse-shell.md).

---

## Coverage gaps (to be addressed in future phases)

| ATT&CK tactic | Gap | Planned phase |
|---|---|---|
| Lateral Movement | No detection for internal network reconnaissance post-compromise | TBD |
| Exfiltration | No detection for data staging or transfer | TBD |

---

*Part of the [Wazuh SIEM Home Lab](../README.md) project · Detection logic in [Phase 5](../phase-5-custom-rules.md/) · Triage procedures in [runbooks/](../runbooks/)*
