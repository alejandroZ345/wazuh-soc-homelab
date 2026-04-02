# Wazuh SIEM Home Lab — Architecture

```mermaid
graph TB
    subgraph HOST["🖥️ Host Machine — Windows (AMD Ryzen 7 5700X)"]
        subgraph WSL2["WSL2 Hypervisor Layer"]
            subgraph UBUNTU["Ubuntu (D: drive)"]
                subgraph DOCKER["Docker Desktop v29.3.1"]
                    IDX["🔍 Wazuh Indexer\nOpenSearch · TLS · Bcrypt auth"]
                    MGR["⚙️ Wazuh Manager\nRule engine · Alert correlation"]
                    DASH["📊 Wazuh Dashboard\nhttps://localhost"]
                end
            end
            subgraph KALI["Kali Linux (D: drive) — Attacker"]
                HYDRA["Hydra\nSSH brute-force\nrockyou.txt"]
                NMAP["Nmap\nPort scanning"]
            end
        end
        WIN_AGENT["🪟 Windows Agent\nWazuhSvc · Telemetry"]
    end

    subgraph UBUNTU_AGENT["Ubuntu Agent — Target endpoint"]
        SSH["OpenSSH Server\nDeliberate attack surface"]
        LIN_AGENT["🐧 Wazuh Agent\nauth.log monitoring"]
    end

    IDX <-->|"Internal TLS"| MGR
    MGR <-->|"Internal TLS"| DASH

    WIN_AGENT -->|"Encrypted telemetry\n1514/tcp"| MGR
    LIN_AGENT -->|"Encrypted telemetry\n1514/tcp"| MGR

    HYDRA -->|"SSH brute-force\n1815 auth failures"| SSH
    NMAP  -->|"Port scan"| SSH
    SSH   --- LIN_AGENT

    MGR -->|"Level 5 alerts: auth failure\nLevel 10 alerts: brute-force"| IDX
    IDX -->|"Indexed events"| DASH
```

## Detection coverage

| Attack simulated       | Tool   | Wazuh rule level | ATT&CK technique             |
|------------------------|--------|------------------|------------------------------|
| SSH brute-force        | Hydra  | Level 10         | T1110.001 — Password Guessing |
| Auth failure (single)  | Hydra  | Level 5          | T1110 — Brute Force           |
| Network reconnaissance | Nmap   | Level 5          | T1046 — Network Service Scan  |
| File integrity change  | Manual | Level 7          | T1565 — Data Manipulation     |

## Stack versions

| Component        | Version      | Notes                        |
|------------------|--------------|------------------------------|
| Wazuh Manager    | 4.14.4       | Single-node Docker           |
| Wazuh Indexer    | 4.14.4       | OpenSearch + Bcrypt auth     |
| Wazuh Dashboard  | 4.14.4       | Self-signed TLS              |
| Docker Desktop   | 29.3.1       | WSL2 backend                 |
| Ubuntu (WSL2)    | Latest LTS   | Migrated to D: drive         |
| Kali Linux (WSL2)| Rolling      | Migrated to D: drive         |
```
