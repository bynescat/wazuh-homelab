# Wazuh agent installation and enrollment

The Wazuh agent was installed on the monitored VM (`lab`) and enrolled with the Wazuh manager on the host. The manager runs as a Docker stack on the host and is reached over the host-only network at `192.168.56.1`. The agent uses port 1515 for the one-time enrollment and port 1514 for the event stream. Both ports were verified as reachable in [03 Monitored VM](03-monitored-vm.md), step 10.

All commands in steps 2 to 5 were run inside the VM over SSH, unless stated otherwise.


## 1. Determining the manager version

The Wazuh documentation guarantees compatibility between agent and manager only if the version of the manager is equal to or newer than the version of the agent. The agent must therefore not be newer than the manager. The version of the manager was read from the running containers on the host:

```bash
sudo docker ps --format '{{.Names}}\t{{.Image}}'
```

**Output:**

```text
single-node-wazuh.dashboard-1	wazuh/wazuh-dashboard:4.14.8
single-node-wazuh.manager-1	wazuh/wazuh-manager:4.14.8
single-node-wazuh.indexer-1	wazuh/wazuh-indexer:4.14.8
```

All three components of the stack run version 4.14.8, so the agent was installed in exactly this version.


## 2. Adding the Wazuh repository

The official Wazuh package repository was added to the VM. The commands install the required tools, import the signing key of Wazuh into a dedicated keyring, register the repository so that only packages signed with this key are accepted, and refresh the package list:

```bash
sudo apt-get install -y gnupg apt-transport-https curl
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt-get update
```

**Output (excerpt):**

```text
gpg: keyring '/usr/share/keyrings/wazuh.gpg' created
gpg: key 96B3EE5F29111145: public key "Wazuh.com (Wazuh Signing Key) <support@wazuh.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1
deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main
Get:1 https://packages.wazuh.com/4.x/apt stable InRelease [17.3 kB]
Get:5 https://packages.wazuh.com/4.x/apt stable/main amd64 Packages [56.3 kB]
```

The key was imported, and `apt` fetched the package list of the Wazuh repository.


## 3. Checking the available agent versions

The repository provides several versions of the agent. The available versions were listed to confirm that the version of the manager is among them:

```bash
apt-cache madison wazuh-agent | head -n 5
```

**Output:**

```text
wazuh-agent |  4.14.10-1 | https://packages.wazuh.com/4.x/apt stable/main amd64 Packages
wazuh-agent |   4.14.9-1 | https://packages.wazuh.com/4.x/apt stable/main amd64 Packages
wazuh-agent |   4.14.8-1 | https://packages.wazuh.com/4.x/apt stable/main amd64 Packages
wazuh-agent |   4.14.7-1 | https://packages.wazuh.com/4.x/apt stable/main amd64 Packages
wazuh-agent |   4.14.6-1 | https://packages.wazuh.com/4.x/apt stable/main amd64 Packages
```

The newest packages, 4.14.9 and 4.14.10, are newer than the manager (4.14.8). A plain `apt-get install wazuh-agent` would have installed 4.14.10 and thereby violated the compatibility rule. The version `4.14.8-1` was therefore installed explicitly.


## 4. Installing, pinning and starting the agent

```bash
sudo env WAZUH_MANAGER="192.168.56.1" apt-get install -y wazuh-agent=4.14.8-1
sudo apt-mark hold wazuh-agent
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

- **`WAZUH_MANAGER`:** the installation script writes this address into the agent configuration as the manager address. It is the address of the host on the host-only network.
- **`wazuh-agent=4.14.8-1`:** pins the installed version to the version of the manager.
- **`apt-mark hold`:** prevents a later `apt upgrade` from updating the agent beyond the version of the manager. The agent is only updated deliberately, together with the manager.
- **`systemctl enable --now`:** starts the agent immediately and at every boot of the VM.


## 5. Verifying enrollment and connection

The log of the agent was checked for the messages of its connection component, `wazuh-agentd`:

```bash
sudo grep wazuh-agentd /var/ossec/logs/ossec.log | tail -n 15
```

**Output:**

```text
2026/11/18 10:46:28 wazuh-agentd: INFO: Version detected -> Linux |lab |6.8.0-149-generic |#149-Ubuntu SMP PREEMPT_DYNAMIC Fri Oct 23 13:21:10 UTC 2026 |x86_64 [Ubuntu|ubuntu: 24.04.5 LTS (Noble Numbat)] - Wazuh v4.14.8
2026/11/18 10:46:28 wazuh-agentd: INFO: Started (pid: 3697).
2026/11/18 10:46:28 wazuh-agentd: INFO: Requesting a key from server: 192.168.56.1
2026/11/18 10:46:28 wazuh-agentd: INFO: No authentication password provided
2026/11/18 10:46:28 wazuh-agentd: INFO: Using agent name as: lab
2026/11/18 10:46:28 wazuh-agentd: INFO: Waiting for server reply
2026/11/18 10:46:28 wazuh-agentd: INFO: Valid key received
2026/11/18 10:46:28 wazuh-agentd: INFO: Waiting 20 seconds before server connection
2026/11/18 10:46:48 wazuh-agentd: INFO: (1410): Reading authentication keys file.
2026/11/18 10:46:48 wazuh-agentd: INFO: Using AES as encryption method.
2026/11/18 10:46:48 wazuh-agentd: INFO: Trying to connect to server ([192.168.56.1]:1514/tcp).
2026/11/18 10:46:48 wazuh-agentd: INFO: (4102): Connected to the server ([192.168.56.1]:1514/tcp).
2026/11/18 10:47:08 wazuh-agentd: INFO: Agent is reloading due to shared configuration changes.
2026/11/18 10:47:16 wazuh-agentd: INFO: SIGNAL [(10)-(User defined signal 1)] Received. Reload agentd.
2026/11/18 10:47:16 wazuh-agentd: INFO: Buffer agent.conf updated, enable: 1 size: 5000
```

The log shows the complete sequence:

- **Detection:** the agent identified the system (Ubuntu 24.04.5 LTS, kernel 6.8.0-149-generic) and its own version, Wazuh v4.14.8.
- **Enrollment:** the agent requested a key from the manager at `192.168.56.1` (port 1515). The manager answered with a valid key. The agent name was taken from the hostname of the VM, `lab`.
- **No enrollment password:** the manager accepted the enrollment without an authentication password. In a production deployment the enrollment should be protected with a password. In this lab, the exposure is limited by the firewall rules of the host (see [03 Monitored VM](03-monitored-vm.md), step 10).
- **Connection:** after a waiting time of 20 seconds, the agent connected to the manager at `192.168.56.1:1514` over TCP, using AES encryption.
- **Shared configuration:** shortly after the connection, the agent reloaded because it had received a centrally managed configuration from the manager.

After the reload, the other modules of the agent started, as the end of the same log file shows:

```bash
sudo tail -n 20 /var/ossec/logs/ossec.log
```

**Output (excerpt):**

```text
2026/11/18 10:47:11 rootcheck: INFO: Starting rootcheck scan.
2026/11/18 10:47:12 wazuh-logcollector: INFO: Started (pid: 4830).
2026/11/18 10:47:13 sca: INFO: Loaded policy '/var/ossec/ruleset/sca/cis_ubuntu24-04.yml'
2026/11/18 10:47:13 wazuh-modulesd:syscollector: INFO: Module started.
2026/11/18 10:47:14 wazuh-syscheckd: INFO: (6009): File integrity monitoring scan ended.
```

The excerpt shows the rootcheck scan, the log collector, the security configuration assessment (SCA) with the CIS benchmark for Ubuntu 24.04, the system inventory (syscollector) and the first scan of the file integrity monitoring (FIM).

## Sources

- [Deploying Wazuh agents on Linux endpoints](https://documentation.wazuh.com/4.14/installation-guide/wazuh-agent/wazuh-agent-package-linux.html) (Wazuh documentation)