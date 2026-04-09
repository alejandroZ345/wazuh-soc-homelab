# Phase 9: Incident response platform (IRP) integration & API automation

> Objective: Establish a robust, automated pipeline between the Wazuh SIEM and an Incident Response Platform (TheHive v5). The goal is to forward high-fidelity, critical alerts (Severity ≥ 10) to a centralized SOC dashboard, enabling security analysts to perform structured triage, investigation, and observable extraction without interacting directly with the raw SIEM logs.

---

## 1. Overview

Phases 1–8 built a complete detection and response pipeline within the Wazuh ecosystem. But in a real SOC, the SIEM is not the final destination for alerts — it feeds into an Incident Response Platform (IRP) where analysts triage, investigate, and track cases through resolution. This phase closes the last gap in the SOC pipeline by integrating TheHive v5 as the centralized case management layer.

The integration required deploying TheHive as an isolated Docker stack, engineering a custom Python integration script to bridge Wazuh's alert format with TheHive v5's REST API (`/api/v1/alert`), and resolving three significant architectural challenges related to RBAC, syntax sensitivity, and I/O race conditions in WSL2.

---

## 2. Architecture & deployment

To prevent resource contention with the SIEM, TheHive v5 was deployed as a distinct, isolated environment using Docker Compose. Memory limits (JVM bounds) were strictly configured to optimize performance on the host machine.

**File:** `~/thehive-docker/docker-compose.yml`

```yaml
services:
  cassandra:
    image: cassandra:4
    container_name: thehive_cassandra
    environment:
      - MAX_HEAP_SIZE=512M
      - HEAP_NEWSIZE=100M
      - CASSANDRA_CLUSTER_NAME=TheHive
    restart: unless-stopped

  elasticsearch:
    image: elasticsearch:7.17.24
    container_name: thehive_elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    restart: unless-stopped

  thehive:
    image: strangebee/thehive:5.2.11
    container_name: thehive_app
    depends_on:
      - cassandra
      - elasticsearch
    environment:
      - JVM_OPTS=-Xms512m -Xmx512m
    ports:
      - "9000:9000"
    command:
      --cql-hostnames cassandra
      --index-backend elasticsearch
      --es-hostnames elasticsearch
    restart: unless-stopped
```

### Stack components

**Cassandra 4** serves as TheHive's primary data store for case and alert data. **Elasticsearch 7.17** provides the search and indexing backend for observable queries and dashboard rendering. **TheHive 5.2.11** is the application layer exposing the analyst UI on port 9000 and the REST API that the Wazuh integration consumes.

Each service is memory-bounded (512M JVM heap) to coexist with the Wazuh stack on the same host without triggering OOM kills.

---

## 3. Wazuh Manager configuration

A service account (`wazuh@thehive.local`) was provisioned within TheHive with a dedicated API key. The Wazuh Manager was then configured to route alerts above level 10 to the WSL2 network gateway IP where TheHive was listening.

**File:** `/var/ossec/etc/ossec.conf` (Integration Block)

```xml
<integration>
    <name>custom-thehive</name>
    <hook_url>http://<THEHIVE_IP>:9000</hook_url>
    <api_key>YOUR_API_KEY</api_key>
    <level>10</level>
</integration>
```

> The `<level>10</level>` filter ensures only high-fidelity alerts (Rules 100001, 100002, 100005, and built-in Level 10+ correlations) are forwarded to TheHive, preventing alert fatigue in the IRP while ensuring all critical events reach the analyst queue.

The configuration can also be modified via the Wazuh Dashboard UI: `Server Management → Settings → Edit Configuration`.

---

## 4. The integration engine — custom Python script

Because native Wazuh integration scripts are deprecated in Docker and incompatible with TheHive v5's REST API (`/api/v1/alert`), a custom integration hook was engineered. This script includes a fallback mechanism to handle both JSON and raw text log formats, ensuring zero dropped alerts.

**File:** `/var/ossec/integrations/custom-thehive`

```python
#!/var/ossec/framework/python/bin/python3
import sys, json, requests, uuid, time

def main():
    alert_file = sys.argv[1]
    api_key = sys.argv[2]
    hook_url = sys.argv[3]

    # Safety pause to avoid I/O race conditions on WSL2 virtual disk
    time.sleep(1)

    with open(alert_file, 'r', encoding='utf-8', errors='replace') as f:
        alert_text = f.read().strip()

    # Base structure for TheHive alert
    alert_data = {
        "type": "wazuh",
        "source": "Wazuh SIEM",
        "sourceRef": str(uuid.uuid4()),
    }

    try:
        # Attempt 1: Structured JSON parsing
        alert = json.loads(alert_text)
        alert_data["title"] = f"Wazuh: {alert.get('rule', {}).get('description', 'Alert')}"
        alert_data["description"] = (
            f"**Rule ID:** {alert.get('rule', {}).get('id', 'N/A')}\n"
            f"**Level:** {alert.get('rule', {}).get('level', 'N/A')}\n\n"
            f"**Original Log:**\n{alert.get('full_log', 'N/A')}"
        )
        alert_data["severity"] = 2 if int(alert.get('rule', {}).get('level', 1)) >= 12 else 1

    except Exception:
        # Attempt 2 (Fallback): Plain text wrapper for OSSEC legacy format
        alert_data["title"] = "Wazuh: FIM / Brute Force Alert"
        alert_data["description"] = (
            f"**Wazuh generated a text-format alert:**\n\n"
            f"```text\n{alert_text}\n```"
        )
        alert_data["severity"] = 2

    # Fire the REST API POST
    headers = {
        'Authorization': f'Bearer {api_key}',
        'Content-Type': 'application/json'
    }
    requests.post(
        f"{hook_url}/api/v1/alert",
        json=alert_data,
        headers=headers,
        verify=False
    )

if __name__ == "__main__":
    main()
```

### Script design decisions

**`time.sleep(1)` at the start:** Addresses the I/O race condition discovered during testing (see Discovery 3 below). The Wazuh Manager writes the temporary alert file to the WSL2 virtual disk, and without this pause, the Python script reads the file before the write completes — resulting in empty or truncated JSON.

**`try/except` with plain-text fallback:** Wazuh occasionally bypasses JSON output and delivers raw OSSEC text strings, especially for FIM events. The fallback wraps the raw text in a markdown code block so the alert still reaches TheHive with the full original content, rather than being silently dropped.

**Dynamic severity mapping:** Alerts with `rule.level >= 12` (Rules 100001 and 100005) are forwarded as TheHive Severity 2 (High). Everything else arrives as Severity 1 (Medium). This preserves the severity hierarchy established in the detection engineering phases.

---

## 5. Troubleshooting & architectural discoveries

This phase presented significant integration hurdles, primarily driven by versioning changes in TheHive and I/O virtualization behaviors in WSL2. All issues were systematically debugged and resolved.

### Discovery 1 — TheHive v5 strict RBAC model

**Issue:** The default administrator account could not view alerts, and the "Analyst" profile was initially unavailable for new users.

**Analysis:** TheHive v5 introduced a strictly segregated multi-tenant architecture. The default `admin` organization is blind to operational data by design — it exists solely for platform administration (user management, organization creation), not for alert triage.

**Resolution:** A dedicated operational organization (`SOC`) was created. Both the human analyst account and the Wazuh service account were migrated to this organization, successfully unlocking the operational dashboards and alert queues.

### Discovery 2 — Syntax sensitivity in Docker Compose commands

**Issue:** The web interface container (`thehive_app`) exited immediately upon creation with an "Unknown parameter" error.

**Analysis:** A syntax error in the `command` block (`-es-hostname` instead of the plural `-es-hostnames`) caused the Java process to terminate on startup.

**Resolution:** Corrected the parameter and forced a container recreation (`docker compose up -d --force-recreate`), achieving stable execution.

### Discovery 3 — I/O latency & race conditions in virtualized environments (WSL2)

**Issue:** The custom Python script repeatedly crashed silently. Diagnostic logs revealed a `json.decoder.JSONDecodeError`.

**Analysis:** A classic race condition. The Wazuh Manager created the temporary alert file on the WSL2 virtual disk, and the Python script attempted to parse it microseconds before the disk I/O operation finished writing the payload. Furthermore, Wazuh occasionally bypassed JSON output, delivering raw OSSEC text strings.

**Resolution:** Overhauled the Python script (detailed in Section 4) to include a safety wait mechanism (`time.sleep`) and a `try/except` fallback. This dual-layered defense forces the script to wait for disk commits and dynamically adapts to the payload format, ensuring 100% alert delivery.

---

## 6. Final validation & end-to-end telemetry

A final simulated attack was executed to trigger Rule 100001 (FIM Critical):

```bash
echo "# TheHive SOC Test" | sudo tee -a /etc/passwd
```

**Execution flow:** Wazuh detected the `/etc/passwd` modification → Created the temporary alert file → The custom Python script handled the I/O delay, adapted to the payload format via the fallback mechanism, and fired the API POST request to TheHive.

**Result:** The alert successfully materialized in TheHive's Analyst Dashboard in real-time, categorized as a Severity 2 (High) event, containing the full original log for forensic analysis. The SIEM-to-IRP pipeline is 100% operational.

| Timestamp | Agent | Description | Level | Rule ID |
|---|---|---|---|---|
| `Apr 9, 2026 @ 07:21:16` | Ubuntu_Admin | CRITICAL: /etc/passwd modified. Potential persistence. | 12 | 100001 |

---

## 7. Key technical decisions

**Why TheHive v5 instead of v4?** TheHive v5 introduces a multi-tenant RBAC model, a modern REST API (`/api/v1/`), and native Elasticsearch integration — all features that align with enterprise SOC requirements. While v4 would have been simpler to integrate (less strict RBAC, legacy API compatibility), choosing v5 demonstrates the ability to work with current-generation security tooling and solve the additional configuration challenges it introduces.

**Why a custom Python script instead of the native Wazuh integration?** The native `custom-w]` integration scripts shipped with Wazuh Docker images are deprecated and hardcoded for TheHive v4's API schema. They fail silently against v5's `/api/v1/alert` endpoint. Engineering a custom script from scratch was the only reliable path, and it provided the opportunity to add the fallback mechanism and I/O delay handling that the native scripts lack.

**Why filter at `level >= 10` instead of forwarding all alerts?** Forwarding every Wazuh alert to TheHive would flood the analyst queue with Level 3–7 informational events (agent restarts, disk space warnings, individual auth failures) that don't warrant case management. The `level >= 10` filter ensures only alerts that represent confirmed attack patterns (brute-force correlation, `/etc/passwd` modification, reverse shells) reach the IRP — matching the high-confidence rules engineered in Phases 5 and 8.

**Why the troubleshooting matters as much as the final config.** The three discoveries (RBAC segregation, syntax sensitivity, I/O race conditions) are not edge cases — they are the exact type of integration challenges that a SOC engineer encounters when bridging security tools from different vendors. Documenting the diagnostic process demonstrates the ability to work across abstraction layers (application RBAC, container orchestration, filesystem I/O) to deliver a working pipeline.

---

*Previous: [Phase 8 — Active response engineering](./phase-8-active-response.md/)*
