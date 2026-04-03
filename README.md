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

[Architecture](./wazuh_lab_architecture.md/)

---

## Lab Environment

| Component | Details |
|---|---|
| Host OS | Windows (AMD Ryzen 7 5700X, Aorus B450-M, RTX 3050) |
| Hypervisor | WSL2 |
| Linux distro | Ubuntu — migrated to D: drive |
| Container engine | Docker Desktop v29.3.1 (WSL2 backend) |
| SIEM version | Wazuh v4.14.4 — single-node architecture |
| Attacker OS | Kali Linux (WSL2) — migrated to D: drive |

---

## Phases

| Phase | Title | Status |
|---|---|---|
| [Phase 1](./phase-1-stack-deployment.md/) | Stack deployment & security hardening | ✅ Complete |
| [Phase 2](./phase-2-agent-deployment/) | Agent deployment & lifecycle management | ✅ Complete |
| [Phase 3](./phase-3-threat-simulation/) | Linux agent & active threat simulation | ✅ Complete |
| Phase 4 | File Integrity Monitoring (FIM) | 🔄 In progress |
| Phase 5 | Custom detection rules | 🔜 Planned |

---

## Key Objectives

**Infrastructure as Code (IaC)** — The entire stack is deployed via `docker-compose.yml`, with credentials pre-hashed and injected at initialization time. No manual post-boot modifications to running containers.

**Credential hardening** — Default `admin / SecretPassword` credentials were replaced with a high-entropy password. The Wazuh Indexer requires a Bcrypt hash in `internal_users.yml` prior to initialization; the process to derive and synchronize that hash across the Manager and Dashboard is fully documented.

**Multi-agent telemetry** — Agents deployed on both a Windows host (`WazuhSvc`) and a Linux endpoint (`Ubuntu_Admin`), both reporting to the Manager over encrypted channels.

**Active threat simulation** — A Hydra dictionary attack from a dedicated Kali Linux instance against an exposed OpenSSH server generated **1,815 authentication failure events** in seconds, triggering Level 5 and Level 10 alerts in the correlation engine.

**Detection validation** — All alert levels, rule IDs, and dashboard views are documented with screenshots and mapped to MITRE ATT&CK techniques.

---

## Detection Coverage

| Attack simulated | Tool | Wazuh alert level | ATT&CK technique |
|---|---|---|---|
| SSH brute-force (bulk) | Hydra + rockyou.txt | Level 10 | T1110.001 — Password guessing |
| SSH auth failure (single) | Hydra | Level 5 | T1110 — Brute force |
| Network reconnaissance | Nmap | Level 5 | T1046 — Network service scan |
| File integrity violation | Manual | Level 7 | T1565 — Data manipulation *(Phase 4)* |

---

## Notable Technical Challenges Solved

**Docker engine migration** — Docker Desktop v29.3.1 hardcodes container volume data to `AppData\Local\Docker` regardless of WSL distribution location. Ubuntu and Kali were successfully migrated to D: drive; Docker's payload remains on C:. This is a known architectural constraint documented as a community note.

**Bcrypt credential synchronization** — Injecting a custom password into a Wazuh Docker deployment requires three synchronized changes: hashing the password with the OpenSearch hash tool in an ephemeral container, writing the hash to `config/wazuh_indexer/internal_users.yml`, and updating plaintext credentials in `docker-compose.yml` with strict double-quote wrapping for special characters. Two failed attempts (in-container script execution and pure IaC injection) are documented before arriving at the validated procedure.

**WSL2 shared network for attack simulation** — The Kali attacker and Ubuntu target both share the WSL2 `eth0` interface, enabling realistic network-layer attack simulation without external hardware. The Manager IP was resolved to `127.0.0.1` by leveraging Docker's port exposure on localhost.

---

## Roadmap

- [ ] Phase 4: Configure FIM on `/etc`, `/bin`, `/usr/bin` and document alert output
- [ ] Phase 5: Write custom Wazuh XML detection rules (brute-force threshold, sudo abuse, `/etc/passwd` modification)
- [ ] MITRE ATT&CK mapping table (complete coverage across all phases)
- [ ] Integrate TheHive for incident case management
- [ ] Implement Active Response scripts for automated threat containment

---

## About

**Alejandro Zavala** — Systems Engineer & Cybersecurity Professional  
ISC2 Certified in Cybersecurity (CC) · Specialization: SOC Operations, Incident Detection, Technical Documentation  
[linkedin.com/in/alejandro-zavala-zenteno](https://www.linkedin.com/in/alejandro-zavala-zenteno) · [ISC2 Badge](https://www.credly.com/badges/fe6eeefe-d684-45fa-8f57-9f7d025db2d6/public_url)
