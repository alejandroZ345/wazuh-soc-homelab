# Phase 2: Agent deployment & lifecycle management

> Objective: Deploy the Wazuh Agent on the primary Windows host to initiate telemetry collection, while implementing a lifecycle management strategy to prevent resource contention during non-lab hours.

---

## 1. Overview

With the Wazuh stack fully operational and hardened (Phase 1), this phase extends monitoring coverage to the host machine itself by enrolling the Windows endpoint as a managed agent. A lightweight lifecycle control procedure is also established to allow the entire lab to be spun up and down on demand without impacting daily host performance.

---

## 2. Deployment execution

### Step 1 — Download the official installer

Execute in **PowerShell** on the Windows host:

```powershell
Invoke-WebRequest `
  -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.4-1.msi `
  -OutFile ${env:tmp}\wazuh-agent.msi
```

### Step 2 — Silent installation

```powershell
msiexec.exe /i ${env:tmp}\wazuh-agent.msi /q `
  WAZUH_MANAGER='<MANAGER_IP>' `
  WAZUH_REGISTRATION_SERVER='<MANAGER_IP>' `
  WAZUH_AGENT_NAME='<AGENT_NAME>'
```

> **Tip:** To resolve the Manager IP from within the Ubuntu WSL2 environment, run:
> ```bash
> hostname -I | awk '{print $1}'
> ```

### Step 3 — Start the service

```powershell
Start-Service -Name WazuhSvc

# Verify the service reached a running state
Get-Service -Name "WazuhSvc"
```

Expected output:

```
Status   Name               DisplayName
------   ----               -----------
Running  WazuhSvc           Wazuh
```

---

## 3. Service lifecycle control

To preserve host performance, the full SIEM stack (Manager + Indexer + Dashboard + Windows agent) is operated on an on-demand basis rather than starting automatically at boot.

### Spin-down (end of lab session)

```powershell
# 1. Stop the Windows agent
Stop-Service -Name WazuhSvc
```

```bash
# 2. Stop the Docker stack (run inside Ubuntu terminal)
docker compose stop
```

### Spin-up (start of lab session)

```bash
# 1. Start the Docker stack (run inside Ubuntu terminal)
docker compose start
```

```powershell
# 2. Start the Windows agent
Start-Service -Name WazuhSvc
```

> **Order matters on spin-up:** always start the Docker stack first and wait ~30 seconds for the Manager to reach a healthy state before starting the Windows agent. Starting the agent against an unavailable Manager will trigger reconnection loops.

---

## 4. Verification

After spin-up, confirm the agent appears as **Active** in the Wazuh Dashboard:

**Wazuh Dashboard → Agents → (agent name) → Status: Active**

Alternatively, query the Manager API from within the Ubuntu terminal:

```bash
curl -k -u admin:<password> \
  https://localhost:55000/agents?pretty=true | grep -E '"status"|"name"'
```

---

## 5. Key technical decisions

**Why a silent MSI installation?** The `/q` flag suppresses the GUI installer entirely, which is consistent with treating infrastructure as reproducible and scriptable. Any future re-enrollment of the agent can be performed from a single PowerShell one-liner without UI interaction.

**Why on-demand lifecycle instead of autostart?** The Wazuh stack (Indexer + Manager + Dashboard) consumes significant RAM. Running it persistently on a development machine would degrade performance during normal use. The on-demand model keeps the lab active only when intentionally working on it, which also reflects a more realistic SOC resource management mindset.

**Why start the Docker stack before the Windows agent?** The agent registration protocol requires the Manager to be reachable at startup. If the Manager container is still initializing when the agent service starts, the agent enters a reconnection backoff loop that can delay telemetry by several minutes.

---

*Previous: [Phase 1 — Stack deployment](./phase-1-stack-deployment.md/) · Next: [Phase 3 — Linux agent & active threat simulation](./phase-3-threat-simulation.md/)*
