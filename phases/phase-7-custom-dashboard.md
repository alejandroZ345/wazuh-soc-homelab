# Phase 7: Custom SOC dashboard engineering & telemetry visualization

> Objective: Design and deploy a centralized "Single Pane of Glass" custom dashboard to visualize Key Performance Indicators (KPIs) and critical threat telemetry. The goal is to provide SOC analysts with immediate situational awareness regarding active agents, alert severities, file integrity, and specific MITRE ATT&CK triggers in real-time.

---

## 1. Overview

Previous phases focused on generating and validating detections — rules fire, alerts appear in the Security Events log, and analysts query them individually. This phase shifts the focus from detection engineering to operational visibility by building a unified dashboard that surfaces the most critical telemetry at a glance.

The dashboard aggregates data from the `wazuh-alerts-*` index pattern across five purpose-built visualizations, laid out in an asymmetric grid optimized for SOC workflow: rapid triage on the right, deep-dive analysis on the left.

---

## 2. Dashboard Deployment & UI workaround

To correctly implement the dashboard, you must configure the proper visualization tools (explained in **3. KPI visualization development**) by going through Menu → Explore → Visualize → Create Visualization. 

Once you have finished configuring your visualization tools, you can use them to create your own custom dashboard. To do this, follow the path: Menu → Explore → Dashboards → Create Dashboard.

### URL bypass

If you are unable to follow the standard navigation path, you can use a direct routing method as follows:

```
https://<Manager_IP>/app/dashboards
```

This forces an exit from the Wazuh plugin and grants full administrative access (after authentication process) to the native OpenSearch visualization engine, where custom index patterns, visualizations, and dashboards can be created without restriction.

---

## 3. KPI visualization development

Five critical metrics were engineered by aggregating data from the `wazuh-alerts-*` index pattern. Each visualization targets a specific operational question a SOC analyst needs answered immediately.

### 3.1 — Timeline (Line Chart)

**Question answered:** *Is there an active attack happening right now?*

A time-series graph mapping event volume (`Count`) against time (`@timestamp`). Sudden vertical spikes are the primary visual indicator of automated attacks — brute-force campaigns, scanning sweeps, or mass file modifications produce event volumes that are visually unmistakable against the environment's normal baseline.

### 3.2 — Rules Level (Pie Chart)

**Question answered:** *What is the overall severity distribution of the environment?*

Groups all alerts by their `rule.level` field, providing an instant health snapshot. A healthy environment shows the majority of the pie in low-severity segments (levels 3–5). If high-severity segments (levels 10–12) dominate, the environment is under active attack or experiencing persistent misconfigurations that need attention.

### 3.3 — Syscheck Events (Pie Chart)

**Question answered:** *Are there unauthorized filesystem changes happening?*

A highly filtered visualization using `rule.groups: "syscheck"` to isolate File Integrity Monitoring telemetry exclusively. The pie segments map to `syscheck.event` values — `added`, `modified`, and `deleted` — providing immediate visibility into the type of filesystem activity occurring across monitored directories (`/etc`, `/bin`, `/usr/bin`).

### 3.4 — Events (Data Table)

**Question answered:** *What are the most frequent threats right now?*

Aggregates alert descriptions (`rule.description`) sorted by `Count` in descending order. This surfaces recurring threat vectors immediately — if a single rule description dominates the table (e.g., "sshd: authentication failed" appearing thousands of times), it confirms an ongoing brute-force attack without requiring any manual query.

### 3.5 — Top agents (Data Table)

**Question answered:** *Which endpoints are generating the most alerts?*

Correlates event volume to specific endpoints (`agent.name`) to pinpoint heavily targeted or potentially compromised hosts. An agent generating disproportionate alert volume compared to others warrants immediate investigation — it may be under active attack or already compromised.

---

## 4. Dashboard assembly & UX optimization

The five visualizations were compiled into a unified master dashboard named **Endpoint Telemetry Dashboard** using an asymmetric, SOC-optimized layout designed around two complementary reading patterns:

**Horizontal focus (left, ~70% width):** The `Timeline` line chart and `Events` data table occupy the majority of the horizontal screen space. These are the deep-dive panels — the timeline reveals *when* something happened, and the events table reveals *what* is happening most frequently. Together they support granular trend analysis and full-text reading of complex alert descriptions.

**Vertical quick-reference (right, ~30% width):** The `Rules Level` pie chart, `Syscheck Events` pie chart, and `Top agents` data table are stacked vertically as a rapid telemetry sidebar. These panels answer the "how bad is it?" and "where is it?" questions at a glance without requiring any scrolling or interaction.

This layout ensures that the most time-sensitive information (active attack indicators) is always visible without scrolling, while detailed investigation data is immediately adjacent for drill-down.

---

## 5. Key technical decisions

**Why bypass the Wazuh UI rather than use its built-in dashboards?** The Wazuh 4.14.4 unified application provides pre-built dashboards (Security Events, Threat Hunting, FIM) that are useful for individual module analysis. However, they cannot be customized or combined into a single view. The OpenSearch Dashboards canvas provides full control over visualization types, index patterns, field aggregations, and layout — essential for building an operationally useful SOC dashboard rather than a module-specific view.

**Why these five specific KPIs?** They map directly to the four fundamental SOC triage questions: *Is there an attack happening?* (Timeline), *How severe is it?* (Rules Level), *What is the attack?* (Events), and *Where is it hitting?* (Top agents). The Syscheck Events panel was added as a fifth metric because FIM violations are a critical indicator in this environment — Phases 4 and 5 demonstrated that filesystem tampering is the primary persistence mechanism being detected.

**Why an asymmetric layout instead of a uniform grid?** SOC analysts scan dashboards in an F-pattern — left-to-right across the top, then down the left side. Placing the timeline (the most time-sensitive indicator) at the top-left and the events table directly below it aligns with this natural reading pattern. The right sidebar serves as a peripheral awareness column that can be checked with a glance without redirecting focus from the primary analysis panels.

---

*Previous: [Phase 6 — MITRE ATT&CK mapping](./phase-6-mitre-mapping.md) · Next: [Phase 8 — Active Response scripts](./phase-8-active-response.md)*
