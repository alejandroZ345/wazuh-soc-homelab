# Phase 6: MITRE ATT&CK telemetry mapping & detection standardization

> Objective: Elevate the SIEM's analytical value by transitioning from isolated alert generation to standardized threat intelligence. This phase maps all validated detection rules from previous offensive simulations to the MITRE ATT&CK® Enterprise Framework, establishing a universal cybersecurity taxonomy and introducing "Detection as Code" principles through a dedicated version-controlled repository structure.

---

## 1. Overview

Individual alerts tell you *what happened*. The MITRE ATT&CK framework tells you *why it matters* — where in the adversary kill chain the activity sits, and what the attacker was trying to accomplish. Mapping detections to ATT&CK transforms a collection of rule IDs into a structured picture of coverage gaps and capabilities.

This phase also introduces a `detections/` directory in the repository as a dedicated home for all mapping and rule logic artifacts, separating operational detection intelligence from phase-by-phase deployment documentation.

---

## 2. Implementation strategy

Three actions were executed to complete this phase:

**Log auditing** — All active triggers generated during the Phase 3, 4, and 5 attack simulations (SSH brute-force, FIM violations, and sudo abuse) were reviewed and correlated against their source events.

**Framework mapping** — Each triggered alert was categorized by its corresponding MITRE ATT&CK Tactic (the adversary's technical goal) and Technique (the specific method used). Where the initial mapping hypothesis was inaccurate, telemetry was re-analyzed and the mapping corrected.

**Repository architecture** — A dedicated `detections/` directory was created within the project repository to host the `mitre-attack-map.md` file. This modular structure ensures detection logic and intelligence mappings remain scalable, maintainable, and independently versioned from deployment documentation.

---

## 3. Consolidated telemetry & tactic matrix

The complete mapping is maintained as a standalone artifact in [`detections/mitre-attack-map.md`](./detections/mitre-attack-map.md).

Summary of coverage by phase:

| Phase | Threat vectors covered | Rules active |
|---|---|---|
| Phase 3 | SSH dictionary attack | 5760 / 5712 (built-in) |
| Phase 4 | Backdoor file creation, FIM integrity violation | 554, 550 (built-in) |
| Phase 5 | `/etc/passwd` modification, SSH brute-force, sudo probing | 100001, 100002, 100003 (custom) |
| Phase 5+ | Terminal-based Discovery, Reverse Shell establishment | 100004, 100005 (custom — behavioral) |

> Custom rules (IDs `10000X`) were specifically engineered during Phase 5 to reduce alert fatigue by establishing strict behavioral thresholds tailored to the environment's baseline. They complement the built-in Wazuh rules rather than replacing them. Rules 100004 and 100005 represent an evolution from signature-based to TTP-based detection.

---

## 4. Expanded MITRE mapping — Discovery & Command and Control

With the integration of the user-space auditing rules (bash_audit pipeline), the SIEM's threat hunting coverage has been expanded to address previously identified visibility gaps:

| Phase | Simulated threat vector | Wazuh Rule ID | Alert Level | ATT&CK Tactic | ATT&CK Technique | Technique ID |
|---|---|---|---|---|---|---|
| Phase 5+ | Terminal-based Discovery (`whoami`, `id`, `uname`) | 100004 | 7 | Discovery | Account Discovery / System Information Discovery | T1087 / T1082 |
| Phase 5+ | Reverse Shell establishment (Bash/Netcat) | 100005 | 12 | Command and Control | Non-Application Layer Protocol | T1095 |

These two rules close the Discovery and Command & Control gaps identified in the initial coverage analysis, leaving Lateral Movement and Exfiltration as the remaining blind spots for future phases.

---

## 5. ATT&CK mapping notes

### On the FIM detections (Phase 4)

The backdoor file creation (`/etc/soc_backdoor.conf`) and subsequent modification events were initially considered for mapping to T1105 (Ingress Tool Transfer) and T1070 (Indicator Removal on Host) respectively. After reviewing the ATT&CK technique definitions:

- **T1105** describes adversaries transferring tools or files *from an external system over a network channel* — not applicable to a file created locally via `touch` and `tee`.
- **T1070** describes adversaries *deleting or altering logs and artifacts to hide their tracks* — the opposite of what occurred here, where the *SIEM detected* a change.

The correct mapping for both FIM events is **T1565.001 (Data Manipulation: Stored Data Manipulation)**, which describes adversaries inserting, deleting, or manipulating data at rest to influence system behavior or create persistence — directly matching the backdoor configuration file scenario.

This distinction matters: accurate ATT&CK mapping ensures that coverage analysis (what tactics are we detecting vs. missing?) reflects reality.

### On the behavioral rules (Phase 5+)

Rule 100005 uses a broad behavioral net by design (see [Phase 5 § 6](./phase-5-custom-rules.md#6-advanced-detection-engineering--the-pyramid-of-pain)). Because it catches a wide array of network-socket activity, it requires a triage runbook to determine the exact nature of each alert. The operational procedure is documented in [`runbooks/runbook-100005-reverse-shell.md`](./runbooks/runbook-100005-reverse-shell.md).

---

## 6. Repository structure after Phase 6

```
wazuh-siem-homelab/
│
├── README.md
├── lab_architecture.md
│
├── phase-1-stack-deployment.md
├── phase-2-agent-deployment.md
├── phase-3-threat-simulation.md
├── phase-4-fim.md
├── phase-5-custom-rules.md
├── phase-6-mitre-mapping.md
│
├── detections/
│   └── mitre-attack-map.md        ← complete ATT&CK mapping table
│
└── runbooks/
    └── runbook-100005-reverse-shell.md  ← Rule 100005 triage SOP
```

---

*Previous: [Phase 5 — Custom detection engineering](./phase-5-custom-rules.md/) · Next: [Phase 7 — Custom Wazuh dashboard](./phase-7-dashboard.md/)*