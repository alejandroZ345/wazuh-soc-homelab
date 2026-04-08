# Phase 3: Linux agent deployment & active threat simulation

> Objective: Validate the complete SOC pipeline (Attack → Telemetry → Alert → Analysis) by deploying a Linux Wazuh Agent, establishing a deliberately vulnerable service, and executing an active brute-force simulation using a dedicated offensive environment within the WSL2 shared network architecture.

---

## 1. Overview

With the Windows agent reporting telemetry from Phase 2, this phase extends the lab to a second endpoint — a Linux target — and introduces a dedicated attacker machine. The result is a two-node monitored environment where real offensive tooling (Hydra, Nmap) is used against a live SSH service, and every event flows through the SIEM pipeline for validation.

The network layer is provided by WSL2's shared `eth0` interface, which allows all three distributions (Ubuntu host, Kali attacker, Wazuh stack) to communicate over the same virtual network without additional configuration.

---

## 2. Attack infrastructure provisioning — Kali Linux via WSL2

To keep the offensive tooling isolated from the primary Ubuntu environment and avoid consuming C: drive space, a dedicated Kali Linux distribution was deployed and immediately migrated to the secondary SSD.

### Deployment and migration to D: drive

Execute in **PowerShell (Administrator)**:

```powershell
# 1. Install Kali Linux from the WSL store
wsl --install -d kali-linux

# 2. Halt all WSL services before export
wsl --shutdown

# 3. Export the image to a tarball on the secondary drive
wsl --export kali-linux D:\WSL_Wazuh\kali_backup.tar

# 4. Unregister the C: drive installation
wsl --unregister kali-linux

# 5. Create the destination directory (name and path are user preference)
mkdir D:\WSL_Wazuh\Kali

# 6. Import the distribution to D: drive
wsl --import kali-linux D:\WSL_Wazuh\Kali D:\WSL_Wazuh\kali_backup.tar

# 7. Cleanup — remove the backup tarball (optional)
Remove-Item -Path "D:\WSL_Wazuh\kali_backup.tar"
```

### Offensive toolset installation

Access the new Kali environment and install the required attack tools and wordlists:

```bash
sudo apt update && sudo apt install nmap hydra wordlists -y
```

---

## 3. Target environment & agent deployment — `Ubuntu_Admin`

The primary Ubuntu distribution hosting the Docker engine was designated as the target endpoint. This deliberate choice reflects a realistic scenario where an attacker and a monitored endpoint share the same network segment.

### Intentional vulnerability setup — OpenSSH

An SSH daemon was installed and started to expose a legitimate, monitored attack surface:

```bash
sudo apt update && sudo apt install openssh-server -y
sudo service ssh start
```

### Wazuh agent installation

The agent was deployed directly on the Ubuntu host OS. Because the Wazuh Manager runs inside Docker with its ports exposed on `localhost`, the manager IP is set to `127.0.0.1`:

```bash
# Download the agent package
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.4-1_amd64.deb

# Install with manager registration parameters
sudo WAZUH_MANAGER='127.0.0.1' WAZUH_AGENT_NAME='Ubuntu_Admin' \
  dpkg -i wazuh-agent_4.14.4-1_amd64.deb

# Reload systemd to register the new unit file
sudo systemctl daemon-reload  # Good practice after dpkg installs

# Enable and start the agent
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

## 4. Threat simulation execution — credential access

A high-frequency dictionary attack was launched from the Kali instance against the Ubuntu SSH port over the shared WSL2 `eth0` interface.

### Payload preparation

```bash
# Decompress the rockyou wordlist (Kali ships it compressed)
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

### Attack execution — Hydra

```bash
# Launch SSH brute-force against the target
# Note: the victim IP will vary depending on your WSL2 network assignment
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://127.0.0.1
```

> To find the correct target IP in your own environment, run `hostname -I | awk '{print $1}'` inside the Ubuntu terminal before launching the attack.

---

## 5. Validation & telemetry analysis

### Event ingestion

The Wazuh Manager successfully ingested and parsed the authentication logs from `/var/log/auth.log` on the `Ubuntu_Admin` agent in real time.

### Telemetry observed

The Threat Hunting dashboard registered a sharp volume spike immediately after the Hydra attack began.

| Metric | Value |
|---|---|
| Total events captured | 1,815 |
| Event type | Authentication failure |
| Authentication successes | 0 |
| Time window | Seconds |

### Alert levels triggered

| Level | Description | Trigger |
|---|---|---|
| Level 5 | Individual SSH authentication failure | Each failed attempt by Hydra |
| Level 10 | Multiple authentication failures — brute-force pattern | Correlation engine identifying high-frequency failures from the same source |

### Conclusion

The threat detection pipeline is fully operational end-to-end. The SIEM correctly ingested raw auth logs, parsed them, and escalated individual failures to a coordinated brute-force alert — matching exactly the behavior expected in a real SOC environment when an attacker runs a dictionary attack.

---

## 6. Key technical decisions

**Why use the Ubuntu host as the attack target instead of a separate VM?** WSL2's shared `eth0` interface allows all distributions to communicate over the same virtual subnet without any network configuration. This eliminates the need for VirtualBox or Hyper-V while still producing authentic network-layer telemetry — the SIEM sees real source/destination IPs, not loopback traffic.

**Why target `root` with Hydra?** Targeting a privileged account produces the clearest, most unambiguous alert signal in Wazuh. Authentication failures against `root` generate consistent log entries that the correlation engine can pattern-match without requiring additional rule tuning at this stage.

**Why install OpenSSH deliberately?** The goal of this phase is to validate that the SIEM detects attacks, not to harden the endpoint. Exposing a real service with a real protocol produces authentic telemetry. A simulated or mocked attack would not exercise the full log ingestion and parsing pipeline in the same way.

---

*Previous: [Phase 2 — Agent deployment & lifecycle management](./phase-2-agent-deployment.md) · Next: [Phase 4 — File Integrity Monitoring](./phase-4-fim.md)*
