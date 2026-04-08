# Wazuh SIEM Home Lab

> End-to-end SOC pipeline: deployment · hardening · agent management · active threat simulation · detection validation.

![Wazuh](https://img.shields.io/badge/Wazuh-4.14.4-blue?style=flat-square)
![Docker](https://img.shields.io/badge/Docker-29.3.1-2496ED?style=flat-square&logo=docker&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-WSL2%20%2F%20Windows-0078D4?style=flat-square&logo=windows&logoColor=white)
![Status](https://img.shields.io/badge/Status-Active%20development-green?style=flat-square)
![ISC2](https://img.shields.io/badge/Cert-ISC2%20CC-purple?style=flat-square)

---

## Overview

This project documents the deployment and operation of an enterprise-grade SIEM/XDR platform using **Wazuh v4.14.4** orchestrated via Docker Compose on a WSL2 environment. The goal is to simulate a real-world Security Operations Center (SOC) pipeline — from infrastructure provisioning to active threat detection — covering skills directly applicable to Junior SOC Analyst and Network Security Engineer roles.

The lab is structured as a series of documented phases, each building on the previous one, so that every configuration decision and troubleshooting step is reproducible and auditable.

---

## Lab Environment

| Component | Details |
|---|---|
| Host OS | Windows (AMD Ryzen 7 5700X, Aorus B450-M, RTX 3050) |
| Hypervisor | WSL2 |
| Linux distro | Ubuntu (WSL2) — migrated to D: drive |
| Container engine | Docker Desktop v29.3.1 (WSL2 backend) |
| SIEM version | Wazuh v4.14.4 — single-node architecture |
| Attacker OS | Kali Linux (WSL2) — migrated to D: drive |

---

## Phases

| Phase | Title | Status |
|---|---|---|
| [Lab Architecture](./architecture/lab_architecture.md/) | Design lab architecture | ✅ Complete |
| [Phase 1](./phases/phase-1-stack-deployment.md/) | Stack deployment & security hardening | ✅ Complete |
| [Phase 2](./phases/phase-2-agent-deployment.md/) | Agent deployment & lifecycle management | ✅ Complete |
| [Phase 3](./phases/phase-3-threat-simulation.md/) | Linux agent & active threat simulation | ✅ Complete |
| [Phase 4](./phases/phase-4-fim.md/) | File Integrity Monitoring (FIM) | ✅ Complete |
| [Phase 5](./phases/phase-5-custom-rules.md/) | Custom detection rules | ✅ Complete |
| [Phase 6](./phases/phase-6-mitre-mapping.md/) | MITRE ATT&CK mapping & detection standardization | ✅ Complete |
| [Phase 7](./phases/phase-7-custom-dashboard.md/)| Custom Wazuh Dashboard | ✅ Complete |
| Phase 8 | Active Response Scripts | 🔄 In progress |
| Phase 9 | TheHive integration — incident case management | 🔜 Planned |

---

## Key Objectives

**Infrastructure as Code (IaC)** — The entire stack is deployed via `docker-compose.yml`, with credentials pre-hashed and injected at initialization time. No manual post-boot modifications to running containers.

**Credential hardening** — Default `admin / SecretPassword` credentials were replaced with a high-entropy password. The Wazuh Indexer requires a Bcrypt hash in `internal_users.yml` prior to initialization; the process to derive and synchronize that hash across the Manager and Dashboard is fully documented.

**Multi-agent telemetry** — Agents deployed on both a Windows host (`WazuhSvc`) and a Linux endpoint (`Ubuntu_Admin`), both reporting to the Manager over encrypted channels.

**Active threat simulation** — A Hydra dictionary attack from a dedicated Kali Linux instance against an exposed OpenSSH server generated **1,815 authentication failure events** in seconds, triggering Level 5 and Level 10 alerts in the correlation engine.

**Custom detection engineering** — Five custom XML rules (100001–100005) were authored, tested against real telemetry, and iteratively tuned to the environment's specific OS and authentication stack. The detection strategy evolved from rigid signature matching to behavioral TTP detection aligned with the Pyramid of Pain methodology.

**MITRE ATT&CK standardization** — All validated detections are mapped to the ATT&CK® Enterprise Framework with a dedicated `detections/` directory for the coverage matrix and a `runbooks/` directory providing standardized triage procedures for each custom rule.

**Customized Dashboard Creation** — Improve operational visibility by building a unified dashboard that features the most critical KPIs built in.

---

## Detection Coverage

### Built-in Wazuh rules (validated through simulation)

| Phase | Attack simulated | Tool | Wazuh Rule ID | Alert level | ATT&CK technique |
|---|---|---|---|---|---|
| Phase 3 | SSH brute-force (bulk) | Hydra + rockyou.txt | 5712 | Level 10 | T1110.001 — Password Guessing |
| Phase 3 | SSH auth failure (single) | Hydra | 5760 | Level 5 | T1110.001 — Brute Force |
| Phase 4 | File added to monitored path | Manual | 554 | Level 5 | T1565.001 — Stored Data Manipulation |
| Phase 4 | File integrity violation | Manual | 550 | Level 7 | T1565.001 — Stored Data Manipulation |

### Custom rules (engineered in Phase 5)

| Phase | Attack simulated | Wazuh Rule ID | Alert level | ATT&CK technique |
|---|---|---|---|---|
| Phase 5 | Critical `/etc/passwd` modification | 100001 | Level 12 | T1078.003 — Valid Accounts: Local |
| Phase 5 | High-frequency SSH brute-force (≥5 in 60s, same IP) | 100002 | Level 10 | T1110.001 — Password Guessing |
| Phase 5 | Multiple failed `sudo` execution attempts | 100003 | Level 8 | T1548.003 — Sudo Abuse |
| Phase 5+ | Terminal-based Discovery (`whoami`, `id`, `uname`) | 100004 | Level 7 | T1087 / T1082 — Discovery |
| Phase 5+ | Reverse Shell establishment (Bash/Netcat/Socat) | 100005 | Level 12 | T1095 — Non-Application Layer Protocol |

> The complete ATT&CK mapping matrix is maintained in [`detections/mitre-attack-map.md`](./detections/mitre-attack-map.md). Triage procedures for each custom rule are documented in the [`runbooks/`](./runbooks/) directory.

---

## Notable Technical Challenges Solved

**Docker engine migration** — Ubuntu, Kali and Docker (virtual disk) were successfully migrated to D: drive. The Docker Desktop UI feature for relocating the virtual disk (`Settings → Resources → Disk image location`) was discovered after an initial failed attempt using WSL export/import, permanently resolving C: drive storage saturation.

**Bcrypt credential synchronization** — Injecting a custom password into a Wazuh Docker deployment requires three synchronized changes: hashing the password with the OpenSearch hash tool in an ephemeral container, writing the hash to `config/wazuh_indexer/internal_users.yml`, and updating plaintext credentials in `docker-compose.yml` with strict double-quote wrapping for special characters. Two failed attempts (in-container script execution and pure IaC injection) are documented before arriving at the validated procedure.

**WSL2 shared network for attack simulation** — The Kali attacker and Ubuntu target both share the WSL2 `eth0` interface, enabling realistic network-layer attack simulation without external hardware. The Manager IP was resolved to `127.0.0.1` by leveraging Docker's port exposure on localhost.

**SID discovery through telemetry analysis** — Custom rules 100002 and 100003 initially failed because community documentation referenced generic SIDs (5716, 2502) that Ubuntu's PAM authentication stack does not produce. Live log analysis during attack simulation revealed the environment-accurate SIDs (5760, 5404), and the rules were tuned accordingly.

**Wazuh Error 1113 & the Pyramid of Pain** — An attempt to write per-language reverse shell signatures using `<pcre2>` failed due to the Wazuh Web UI's strict validator rejecting nested alternation groups. Rather than finding workarounds for fragile signatures, the detection strategy was elevated to target behavioral TTPs using the OSSEC `<regex>` engine and `<program_name>` tag, producing rules 100004/100005 that are resilient to adversary evasion.

**User-space command auditing without kernel recompilation** — Full terminal command visibility was achieved on WSL2 (where auditd is unavailable) by injecting a `PROMPT_COMMAND` hook into `/etc/bash.bashrc` that pipes every executed command through `logger` to syslog, tagged as `bash_audit`. This created the telemetry pipeline that rules 100004 and 100005 consume.

---

## Repository Structure

```
wazuh-siem-homelab/
│
├── README.md
├── LICENSE
│
├── architecture/ 
│    └── lab_architecture.md
│
├── phases/ 
│    ├── phase-1-stack-deployment.md      
│    ├── phase-2-agent-deployment.md         
│    ├── phase-3-threat-simulation.md           
│    ├── phase-4-fim.md    
│    ├── phase-5-custom-rules.md
│    ├── phase-6-mitre-mapping.md
│    └── phase-7-custom-dashboard.md
│
├── detections/
│   └── mitre-attack-map.md                   
│
├── runbooks/
│    ├── runbook-100001-passwd-modification.md  
│    ├── runbook-100002-ssh-bruteforce.md       
│    ├── runbook-100003-sudo-abuse.md           
│    ├── runbook-100004-discovery.md            
│    └── runbook-100005-reverse-shell.md     
│ 
├── .github/
│   ├── SECURITY.md                            
│   ├── CODE_OF_CONDUCT.md                     
│   ├── CONTRIBUTING.md                        
│   └── ISSUE_TEMPLATE/
│       ├── config.yml                         
│       ├── bug-report.yml                    
│       └── phase-suggestion.yml               
```

---

## Roadmap

- [x] Phase 4: Configure FIM on `/etc`, `/bin`, `/usr/bin` and document alert output
- [x] Phase 5: Write custom Wazuh XML detection rules (brute-force threshold, sudo abuse, `/etc/passwd` modification)
- [x] Phase 6: MITRE ATT&CK mapping table (complete coverage across phases 1 to 5)
- [x] Phase 7: Implement a Custom Wazuh Dashboard starring at critical KPIs in lab
- [ ] Phase 8: Implement Active Response scripts for automated threat containment
- [ ] Phase 9: Integrate TheHive for incident case management

---

## About

**Alejandro Zavala** — Systems Engineer & Cybersecurity Professional  
ISC2 Certified in Cybersecurity (CC) · Specialization: SOC Operations, Incident Detection, Technical Documentation  
[linkedin.com/in/alejandro-zavala-zenteno](https://www.linkedin.com/in/alejandro-zavala-zenteno) · [ISC2 Badge](https://www.credly.com/badges/fe6eeefe-d684-45fa-8f57-9f7d025db2d6/public_url)
