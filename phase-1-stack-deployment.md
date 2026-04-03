# Phase 1: Stack deployment

> Objective: Establish a secure, high-performance local deployment of Wazuh SIEM using Docker containers via WSL2, with infrastructure migrated to a secondary SSD where possible.

---

## 1. Lab environment

| Component | Details |
|---|---|
| Host OS | Windows |
| Hardware | AMD Ryzen 7 5700X · Aorus B450-M · NVIDIA RTX 3050 |
| Hypervisor | Windows Subsystem for Linux (WSL2) |
| Linux distro | Ubuntu — successfully migrated to D: drive |
| Container engine | Docker Desktop v29.3.1 (WSL2 backend — remains linked to C:) |
| SIEM version | Wazuh v4.14.4 — single-node architecture |

---

## 2. Deployment runbook

### Phase 1.1 — WSL2 & Ubuntu relocation

By default, Windows installs WSL distributions on the C: drive. The following steps migrate the Ubuntu environment to the D: drive to preserve space on the primary OS drive.

**Execute in PowerShell (Administrator):**

```powershell
# 1. Install WSL and default Ubuntu distribution
wsl --install -d Ubuntu

# 2. Halt all WSL services
wsl --shutdown

# 3. Create destination directory
mkdir D:\WSL_Wazuh

# 4. Export the base image to a tarball
wsl --export Ubuntu D:\WSL_Wazuh\ubuntu_backup.tar

# 5. Unregister the C: drive installation
wsl --unregister Ubuntu

# 6. Import the distribution to the new D: drive location
wsl --import Ubuntu D:\WSL_Wazuh\Ubuntu D:\WSL_Wazuh\ubuntu_backup.tar
```

**Execute inside Ubuntu terminal (`wsl -d Ubuntu`) — restore default user if necessary:**

```bash
# Replace <your_username> with the UNIX user created during installation
echo -e "[user]\ndefault=<your_username>" > /etc/wsl.conf
# Exit and restart WSL to apply: wsl --shutdown
```

---

### Phase 1.2 — Docker engine migration attempt & known limitations

> **Community note / open challenge**

The unified `docker-desktop` WSL distribution was migrated to the D: drive using the standard export/import method. While the WSL commands executed successfully after forcefully killing Docker's background processes, observation confirmed that Docker Desktop still heavily relies on the C: drive for actual container volume data and image caching — likely hardcoded into `AppData\Local\Docker`.

**Attempted migration sequence:**

```powershell
taskkill /IM "Docker Desktop.exe" /F
taskkill /IM "com.docker.backend.exe" /F
wsl --shutdown
wsl --export docker-desktop D:\WSL_Wazuh\docker-desktop.tar
wsl --unregister docker-desktop
wsl --import docker-desktop D:\WSL_Wazuh\DockerDesktop D:\WSL_Wazuh\docker-desktop.tar
```

**Status:** Partially successful at the WSL layer. Container payload remains on C:. Deployment proceeded as-is.

---

### Phase 1.3 — Wazuh infrastructure deployment

All commands executed from the isolated Ubuntu environment (`wsl -d Ubuntu`).

```bash
# 1. Increase virtual memory map limit — prevents Wazuh Indexer kernel crashes
sudo sysctl -w vm.max_map_count=262144

# 2. Clone the official repository
cd ~
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker

# 3. Checkout the latest stable v4.x release
# Strict regex to avoid alphas, betas, and release candidates
git checkout $(git tag | grep -E "^v4\.[0-9]+\.[0-9]+$" | sort -V | tail -n 1)

# 4. Navigate to single-node architecture
cd single-node

# 5. Generate internal TLS certificates using the ephemeral generator container
docker compose -f generate-indexer-certs.yml run --rm generator

# 6. Deploy the SIEM stack in detached mode
docker compose up -d
```

At this point the Wazuh Dashboard is accessible at `https://localhost`, but uses default credentials. Phase 1.4 replaces them with a high-entropy password before the lab proceeds.

---

### Phase 1.4 — Security hardening & credential management

#### Objective

Eradicate the default administrative credentials (`admin` / `SecretPassword`) across the Wazuh v4.14.4 Docker single-node deployment. The goal is to enforce a high-entropy password policy while maintaining the integrity of the OpenSearch internal database and inter-container TLS communication.

---

#### Troubleshooting & iterative diagnostics

Two approaches were attempted and documented before arriving at the validated procedure. This section is preserved deliberately — understanding *why* approaches fail is as important as knowing what works.

**Attempt 1 — In-container script execution**

- **Approach:** Used the native `wazuh-passwords-tool.sh` utility inside the running Indexer container.
- **Challenges encountered:**
  1. **Filesystem refactor:** Tool path changed in v4.14.4 from `/bin/` to the application root.
  2. **Privilege escalation:** Docker defaults to the non-privileged `wazuh` user. Overridden via `docker exec -u root`.
  3. **Missing dependencies:** The minimal Docker image lacked `sudo`. Dynamically provisioned via `yum install sudo`.
  4. **Architecture mismatch:** The script failed with `"User does not exist"` because it expected a native Linux filesystem structure and could not locate the Docker-mapped `internal_users.yml` configuration file.
- **Conclusion:** Modifying running containers manually violates immutable infrastructure principles. Pivoted to an Infrastructure as Code (IaC) approach.

**Attempt 2 — Pure IaC via `docker-compose.yml`**

- **Approach:** Injected the new high-entropy password directly into the environment variables of `docker-compose.yml`.
- **Challenges encountered:**
  1. **YAML parsing errors:** Unquoted special characters (e.g., `[`, `^`) in the complex password caused parsing failures. Mitigated by wrapping the string in double quotes (`"..."`).
  2. **Client-server authentication desync:** While the Compose file updated the client-side credentials (Dashboard and Manager), the Indexer (Server) image ships with the legacy `SecretPassword` pre-hashed in its internal database. The Dashboard was consequently locked out.
- **Conclusion:** The Indexer requires a pre-computed Bcrypt hash in its core configuration file **prior to initialization**.

---

#### Final validated procedure

To deploy custom credentials on a fresh Docker installation, the password must be manually hashed and synchronized across both the server configuration and the client deployment manifest.

**Step 0 — Destroy the previous stack** *(optional but recommended)*

```bash
docker compose down -v
```

**Step 1 — Generate the Bcrypt hash**

Spawn an ephemeral container to use the OpenSearch hash tool. Input the password carefully to avoid blind-typing errors.

```bash
docker run --rm -ti wazuh/wazuh-indexer:4.14.4 \
  bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
# Output: copy the resulting Bcrypt hash to clipboard
```

**Step 2 — Update the Indexer's internal database (server-side)**

```bash
nano config/wazuh_indexer/internal_users.yml
# Replace the default hash for the 'admin' user with the Bcrypt hash from Step 1
```

**Step 3 — Update the deployment manifest (client-side)**

```bash
nano docker-compose.yml
# Update INDEXER_PASSWORD, DASHBOARD_PASSWORD, and wazuh.manager API passwords
# Wrap the value in double quotes: "Your_Custom_Password"
```

**Step 4 — Clean initialization**

Force the Indexer to read the new `internal_users.yml` on boot, regenerate certificates, and launch the stack.

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
docker compose up -d
```

---

## 3. Outcome & verification

| Check | Result |
|---|---|
| `wazuh.indexer` container | Started |
| `wazuh.manager` container | Started |
| `wazuh.dashboard` container | Started |
| Dashboard accessible at `https://localhost` | Confirmed (bypass self-signed certificate warning) |
| Indexer initialized with custom Bcrypt credentials | Confirmed |
| Manager and Dashboard API connection | Established without authentication loops |
| Admin login with high-entropy password | Verified |

---

## 4. Key technical decisions

**Why single-node architecture?** Sufficient for a home lab with one engineer. Multi-node adds operational complexity without benefit at this scale.

**Why migrate WSL distributions to D: drive?** To preserve C: drive space over the course of long lab sessions. Docker Desktop's container payload cannot be fully migrated due to hardcoded paths in `AppData\Local\Docker` — a known architectural constraint documented here as a community note for future reference.

**Why pre-hash the Bcrypt credential?** The Wazuh Indexer (OpenSearch) reads `internal_users.yml` only on first initialization. Injecting a plaintext password into `docker-compose.yml` only updates the client-side services (Manager, Dashboard) — the server-side Indexer still holds the legacy hash. Both sides must be synchronized before `docker compose up`.

---

*Next: [Phase 2 — Agent deployment & lifecycle management](../phase-2-agent-deployment/)*
