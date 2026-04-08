# Runbook: Rule 100005 — Potential Reverse Shell triage

> **Trigger:** Wazuh Alert `100005` — Level 12  
> **Description:** `CRITICAL: Suspicious network command executed. Potential Reverse Shell.`  
> **ATT&CK mapping:** T1095 — Non-Application Layer Protocol (Command and Control)

---

## Context

Rule 100005 uses a broad behavioral detection net rather than highly specific signatures. It fires on any terminal command containing network-socket primitives (`dev/tcp`, `import socket`, `nc -`, `netcat`, `socat`, `fsockopen`), which means it will catch a wide array of potentially malicious activity — including some that may be benign in certain contexts.

When this Level 12 alert triggers, the following triage procedure must be executed to determine the exact nature and severity of the threat.

---

## Triage steps

### Step 1 — Payload extraction

Review the `full_log` field within the Wazuh Security Events dashboard to extract the exact script or syntax executed by the threat actor.

Determine the variant: was it a Python `socket` import, a Bash `/dev/tcp` redirection, a `netcat` listener, or another method? The specific syntax informs subsequent steps.

### Step 2 — Connection verification

Access the compromised endpoint and verify whether the reverse shell successfully established an external connection:

```bash
ss -antp | grep <suspicious_port>
```

If a connection is active, identify the remote IP address — this is the attacker's C2 server.

### Step 3 — Process tree analysis

Trace the origin of the shell by examining the process hierarchy on the endpoint:

```bash
ps -ef --forest
```

Identify the parent process of the suspicious shell. Key questions to answer: was the shell spawned by an exploited web server (Nginx, Apache), an unauthorized SSH session, a cron job, or a user-space process? The parent process reveals the initial access vector.

### Step 4 — Containment & eradication

Once the malicious process and attacker IP are confirmed:

**Terminate the process:**

```bash
kill -9 <PID>
```

**Block the attacker's IP** with an immediate firewall rule to sever the Command and Control channel:

```bash
sudo iptables -A INPUT -s <attacker_ip> -j DROP
sudo iptables -A OUTPUT -d <attacker_ip> -j DROP
```

### Step 5 — Post-incident documentation

Record the following in the incident log: timestamp of initial alert, full payload from `full_log`, attacker IP, parent process chain, and containment actions taken. This information feeds back into the detection engineering cycle for potential rule refinement.

---

## False positive guidance

The following scenarios may trigger Rule 100005 without malicious intent:

- System administrators using `netcat` for legitimate network diagnostics.
- Automated scripts that use Python `socket` for health checks or monitoring.
- Development environments where `socat` is used for port forwarding during testing.

If a pattern is identified as a recurring false positive, consider adding a `<if_sid>` exception rule with a `<match>` filter on the specific user or process, rather than broadening the detection net — maintaining coverage while reducing noise.

---

*Part of the [Wazuh SIEM Home Lab](../README.md) project · Rule definition in [Phase 5 § 6](../phase-5-custom-rules.md#6-advanced-detection-engineering--the-pyramid-of-pain) · ATT&CK mapping in [detections/](../detections/mitre-attack-map.md)*
