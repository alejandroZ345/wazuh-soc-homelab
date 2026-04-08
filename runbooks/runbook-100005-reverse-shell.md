# Runbook: Rule 100005 — Potential Reverse Shell

> **Trigger:** Wazuh Alert `100005` — Level 12  
> **Description:** `CRITICAL: Suspicious network command executed. Potential Reverse Shell.`  
> **ATT&CK mapping:** T1095 — Non-Application Layer Protocol (Command and Control)  
> **Detection method:** `bash_audit` syslog pipeline — matches `dev/tcp`, `import socket`, `nc -`, `netcat`, `socat`, `fsockopen`

---

## Context

Rule `100005` uses a broad behavioral detection net targeting network-socket primitives that appear across virtually every reverse shell variant regardless of scripting language. Because it catches a wide array of activity, each trigger requires triage to determine the exact nature and severity of the threat.

This is a Level 12 critical alert because establishing an outbound shell is a clear post-compromise action with no legitimate use case in this environment.

---

## Triage steps

### Step 1 — Payload extraction

Review the `full_log` field within the Wazuh Security Events dashboard to extract the exact command executed.

Determine the variant: was it a Python `socket` import, a Bash `/dev/tcp` redirection, a `netcat` listener, or another method? The specific syntax informs subsequent steps.

### Step 2 — Connection verification

Access the endpoint and verify whether the reverse shell successfully established an external connection:

```bash
ss -antp | grep <suspicious_port>
```

If a connection is active, identify the remote IP address — this is the attacker's C2 server.

### Step 3 — Process tree analysis

Trace the origin of the shell:

```bash
ps -ef --forest
```

Identify the parent process. Key questions: was the shell spawned by an exploited web server (Nginx, Apache), an unauthorized SSH session, a cron job, or a user-space process? The parent process reveals the initial access vector.

### Step 4 — Containment & eradication

Once the malicious process and attacker IP are confirmed:

**Terminate the process:**

```bash
kill -9 <PID>
```

**Block the attacker's IP** to sever the C2 channel:

```bash
sudo iptables -A INPUT -s <attacker_ip> -j DROP
sudo iptables -A OUTPUT -d <attacker_ip> -j DROP
```

### Step 5 — Post-incident documentation

Record: timestamp of initial alert, full payload from `full_log`, attacker IP, parent process chain, and containment actions taken. This information feeds back into the detection engineering cycle for potential rule refinement.

---

## False positive guidance

The following scenarios may trigger Rule 100005 without malicious intent:

- System administrators using `netcat` for legitimate network diagnostics.
- Automated scripts that use Python `socket` for health checks or monitoring.
- Development environments where `socat` is used for port forwarding during testing.

If a pattern is identified as a recurring false positive, consider adding a `<if_sid>` exception rule with a `<match>` filter on the specific user or process, rather than broadening the detection net.

---

*Part of the [Wazuh SIEM Home Lab](../README.md) project · Rule definition in [Phase 5 § 6](../phase-5-custom-rules.md#6-advanced-detection-engineering--the-pyramid-of-pain) · ATT&CK mapping in [detections/](../detections/mitre-attack-map.md)*
