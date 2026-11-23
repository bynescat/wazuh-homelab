
This project demonstrates the deployment of a centralized Security Information and Event Management (SIEM) and Extended Detection and Response (XDR) platform using **Wazuh 4.14.8** running entirely in Docker containers. The environment is configured to aggregate endpoint telemetry, monitor system integrity, and enable real-time threat detection mapped to the **MITRE ATT&CK** matrix.

## Deployment & Installation Overview

### 1. System Requirements & Lab Environment

#### Official Minimum Requirements (Wazuh Docker Single-Node)

To run the containerized Wazuh stack reliably, the official baseline requirements consist of:

* **CPU:** 4 cores (64-bit x86_64 or ARM64)
* **RAM:** 8 GB allocated to the Docker host
* **Storage:** 50 GB free disk space (SSD recommended for Indexer I/O)
* **Software:** Linux host (e.g., Ubuntu), Docker Engine, and Docker Compose

#### Lab Environment

* **OS:** Ubuntu 24.04.6 LTS
* **Hardware:** 4 CPU cores, 16 GB RAM, 512 GB SSD
* **Stack:** Wazuh v4.14.8 (Single-Node Docker Compose)


### 2. Host System Preparation and Kernel Tuning

The Wazuh indexer (OpenSearch) requires a higher virtual memory mapping limit to function properly. Before bringing up the container stack, the host kernel parameters were adjusted to ensure indexer storage stability.

The setting was applied immediately to the active session:

```bash
sudo sysctl -w vm.max_map_count=262144
```

The configuration was then written to `/etc/sysctl.d/` and reloaded so the change persists across system reboots:

```bash
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-wazuh-max-map-count.conf
sudo sysctl --system
```

### 3. Cloning the Official Wazuh Docker Repository

The official Wazuh Docker repository was cloned using the `v4.14.8` branch, and the working directory was switched to the single-node deployment folder:

```bash
git clone [https://github.com/wazuh/wazuh-docker.git](https://github.com/wazuh/wazuh-docker.git) -b v4.14.8
cd wazuh-docker/single-node/
```

> **Note on Docker Compose Requirements:** 
> The official Wazuh deployment files rely on **Docker Compose v2** (the modern `docker compose` plugin). Because the host environment initially only included legacy `docker-compose` (v1), the Docker Compose v2 CLI plugin had to be installed before proceeding.

To enable Docker Compose v2 on the Linux host, the plugin binary was downloaded to the Docker CLI plugins directory and made executable:

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo wget [https://github.com/docker/compose/releases/download/v2.29.7/docker-compose-linux-x86_64](https://github.com/docker/compose/releases/download/v2.29.7/docker-compose-linux-x86_64) -O /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

### 4. Generating SSL Certificates

Wazuh requires encrypted internal TLS communication across its core components (Manager, Indexer, and Dashboard). The official certificate generator helper was executed to automatically create the required SSL/TLS keys and configuration files within Docker volumes:

```bash
sudo docker compose -f generate-indexer-certs.yml run --rm generator
```

### 5. Starting the Container Stack

Once the certificates were generated, the multi-service Wazuh stack was started in detached mode:

```bash
sudo docker compose up -d
```

The operational status of all deployed containers was then verified:

```bash
sudo docker compose ps
```


## Deployment 

The deployment utilizes the single-node Docker Compose stack, isolating core SIEM components into dedicated container services:

| Service | Image | Published ports (host → container) | Purpose |
| :--- | :--- | :--- | :--- |
| `wazuh.dashboard` | `wazuh/wazuh-dashboard:4.14.8` | `443` → `5601`/TCP | Web UI (`https://localhost`) |
| `wazuh.indexer` | `wazuh/wazuh-indexer:4.14.8` | `9200`/TCP | Indexer REST API (OpenSearch-based storage and search) |
| `wazuh.manager` | `wazuh/wazuh-manager:4.14.8` | `1514`/TCP, `1515`/TCP, `514`/UDP, `55000`/TCP | Agent events (1514), agent enrollment (1515), syslog intake (514/UDP), Wazuh API (55000) |

