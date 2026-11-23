# Wazuh 4.14.8 Single-Node Docker Homelab

A centralized Security Information and Event Management (SIEM) and Extended Detection and Response (XDR) deployment running **Wazuh v4.14.8** in Docker containers. The environment aggregates endpoint telemetry, monitors file integrity, and maps real-time threat detections to the **MITRE ATT&CK** matrix.

## Architecture & Lab Environment

- **Host OS:** Ubuntu 24.04.6 LTS
- **Hardware:** 4 CPU cores, 16 GB RAM, 512 GB SSD
- **Stack:** Wazuh v4.14.8 (Single-Node Docker Compose v2)

| Service | Image | Host Binding | Purpose |
| :--- | :--- | :--- | :--- |
| `wazuh.dashboard` | `wazuh/wazuh-dashboard:4.14.8` | `127.0.0.1:443` → `5601`/TCP | Web UI (`https://localhost`) |
| `wazuh.indexer` | `wazuh/wazuh-indexer:4.14.8` | `127.0.0.1:9200` → `9200`/TCP | OpenSearch-based storage & REST API |
| `wazuh.manager` | `wazuh/wazuh-manager:4.14.8` | `1514`/TCP, `1515`/TCP, `514`/UDP, `127.0.0.1:55000` | Agent events, enrollment, syslog intake & API |

## Key Security & Hardening Highlights

- **Credential Rotation:** Replaced the default OpenSearch indexer `admin` password and regenerated hashes via `securityadmin.sh` (with `kibanaserver` and `wazuh-wui` documented as accepted lab risks behind loopback bindings).
- **Loopback Isolation:** Bound management and UI ports (`9200`, `55000`, `443`) to `127.0.0.1` in `docker-compose.yml` so administrative interfaces are unreachable from the local network.
- **Docker-Aware Firewalling:** Configured `ufw` with a default-deny inbound policy and enforced subnet-level filtering (`192.168.1.0/24`) on agent/syslog ports (`1514`, `1515`, `514/udp`) via persistent `DOCKER-USER` iptables rules in `/etc/ufw/after.rules` (bypassing Docker's `FORWARD` chain NAT behavior).

## Documentation

- **[Deployment & Installation Guide](docs/setup.md):** Host kernel tuning (`vm.max_map_count`), Docker Compose v2 plugin setup, TLS certificate generation, and container stack deployment.
- **[Hardening & Network Restriction Guide](docs/hardening.md):** Credential rotation, loopback port bindings, `ufw` + `DOCKER-USER` firewall rules, and network verification tests.

## Quick Overview

1. Configured `vm.max_map_count=262144` on the Ubuntu host and installed the Docker Compose v2 CLI plugin.
2. Cloned `wazuh/wazuh-docker` (`v4.14.8`), generated internal TLS certificates, and launched the single-node stack.
3. Rotated the indexer `admin` credentials and updated the OpenSearch security index.
4. Restricted management APIs to `127.0.0.1` and filtered agent/syslog traffic to the trusted subnet via `DOCKER-USER` rules.
