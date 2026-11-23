
### Security Considerations

The default single-node deployment publishes all service ports on every network interface (`0.0.0.0`) and ships with default credentials. This is acceptable for an isolated lab, but would require hardening in any shared or production environment.

- **Exposed management interfaces:** The indexer API (9200/TCP) and the Wazuh manager API (55000/TCP) are reachable from the local network and are protected only by default credentials. Possible mitigations include changing all default passwords, restricting access with a host firewall (e.g. `ufw`), or binding these ports to the loopback interface (`127.0.0.1`) in `docker-compose.yml`.

- **Least-privilege network access:** Wazuh agents only need to reach the manager on ports 1514/TCP (event forwarding) and 1515/TCP (enrollment), and syslog-sending devices only need 514/UDP. Access to these ports can therefore be limited to the relevant source addresses.

- **Host firewall limitation:** Docker publishes container ports through its own iptables rules, which bypass the host firewall ufw. A firewall rule alone therefore does not secure these ports.

- **Dashboard access:** The web interface (443/TCP) uses a self-signed certificate generated during initial setup. For long-term use, a certificate issued by a trusted CA would be preferable, and access would ideally be limited to administrative hosts.


## Hardening Plan

The hardening proceeds in the following order: credentials first, then reduction of network exposure, followed by verification of the result and the addition of an agent to confirm that detection works.


### 1. Change of default credentials

Replacing the default credentials is the most important measure. The procedure below was carried out for the `admin` user only. The `kibanaserver` user is changed in the same way (new hash in `internal_users.yml`, `DASHBOARD_PASSWORD` in `docker-compose.yml`, then `securityadmin.sh`). The `wazuh-wui` API user is a Wazuh API user and is changed through a different procedure (Wazuh API or dashboard, then `API_PASSWORD` in `docker-compose.yml` and `config/wazuh_dashboard/wazuh.yml`). Both users still use their default credentials in this lab.

Before modifying configuration files, backups were created to serve as a safety net in case of syntax or configuration errors:

```bash
cp docker-compose.yml docker-compose.yml.bak
cp config/wazuh_indexer/internal_users.yml config/wazuh_indexer/internal_users.yml.bak
```
After logging out of the Wazuh dashboard (persistent session cookies otherwise cause errors after the change), the INDEXER_PASSWORD variable was updated under both the wazuh.manager and wazuh.dashboard services in docker-compose.yml. 

The updated password entry in the compose file was validated:

```bash
grep -n "INDEXER_PASSWORD" docker-compose.yml
```

> **Note on Password Constraints:** 
> Passwords containing special characters like `$` were escaped as `$$` in `docker-compose.yml`. For the API user, the password was required to be between 8 and 64 characters long and include an uppercase letter, a lowercase letter, a number, and a symbol.

#### Applying the new password in the indexer backend

To complete the password update inside the OpenSearch backend, the following procedure was executed:

1. **Stack shutdown and hash generation**

The stack was stopped, and the hash of the new password was generated with the tool included in the indexer image:

 ```bash
sudo docker compose down
sudo docker run --rm -ti wazuh/wazuh-indexer:4.14.8 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
 ```

2. **Hash insertion** 

The existing hash of the admin user in config/wazuh_indexer/internal_users.yml was replaced with the newly generated hash:

```bash
nano config/wazuh_indexer/internal_users.yml
```
   *The existing hash for the admin user was replaced with the newly generated hash:*
   ```yaml
   admin:
     hash: "$2y$12$...new hash..."
     reserved: true
   ```

3. **Stack restart** 

The container stack was restarted to load the changed configuration:

```bash
sudo docker compose up -d
```

4. **Application of the change inside the indexer container** 

The running indexer container was accessed with:

```bash
sudo docker exec -it single-node-wazuh.indexer-1 bash
```

Inside the container shell, the following variables were defined for the security script:

```bash
export INSTALLATION_DIR=/usr/share/wazuh-indexer
export CONFIG_DIR=$INSTALLATION_DIR/config
CACERT=$CONFIG_DIR/certs/root-ca.pem
KEY=$CONFIG_DIR/certs/admin-key.pem
CERT=$CONFIG_DIR/certs/admin.pem
export JAVA_HOME=/usr/share/wazuh-indexer/jdk
```

**INSTALLATION_DIR:** the installation directory of the Wazuh indexer inside the container.
**CONFIG_DIR:** the configuration directory of the indexer, derived from the installation directory.
**CACERT:** the path of the root CA certificate used to verify the indexer's TLS certificate.
**KEY:** the path of the private key belonging to the administrator client certificate.
**CERT:** the path of the administrator client certificate used to authenticate against the indexer.
**JAVA_HOME:** the bundled Java runtime required by the security tool.

After allowing one to five minutes for the indexer to fully initialize, the security script was executed:

```bash
bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -cd $CONFIG_DIR/opensearch-security/ -nhnv -cacert $CACERT -cert $CERT -key $KEY -p 9200 -icl
   ```
* `securityadmin.sh ...`: The script connects to the running indexer on port 9200 using the administrator certificate and loads the updated security configuration files, including the new password hash, into the security index. The option -nhnv skips host name verification, and -icl ignores the cluster name check.


5. **Access test:** 

   The web dashboard (https://localhost) was accessed, and authentication with the new administrator credentials succeeded.



### 2. Restriction of Network Exposure

The network exposure was reduced in three layers, each covering a different part of the attack surface:

- **Loopback binding:** ports 9200 and 55000 are bound to 127.0.0.1, so the indexer API and the manager API are no longer reachable from other hosts on the network. They remain accessible from the laptop itself and from other containers on the compose network.
- **DOCKER-USER rules:** the ports that must stay reachable (1514/TCP, 1515/TCP and 514/UDP) are limited to the trusted subnet. Docker-published ports bypass ufw, so these must be filtered in the DOCKER-USER chain. Only published ports that must stay reachable but should not be open to everyone need a DOCKER-USER rule.
- **ufw:** covers services running directly on the host (such as SSH) and sets a default-deny inbound policy for them. It does not filter Docker-published ports.

#### Port Exposure Strategy

| Port | Measure | Reason |
| :--- | :--- | :--- |
| `9200` (Indexer API) | Bound to `127.0.0.1` | Required locally only; accessed internally by the dashboard and manager |
| `55000` (Manager API) | Bound to `127.0.0.1` | Required locally only; accessed internally by the dashboard |
| `1514`, `1515` | Kept open, limited to the trusted subnet (DOCKER-USER)| Required for remote agents to forward events and enroll |
| `514/udp` | Kept open, limited to the trusted subnet (DOCKER-USER), or removed if unused | Required exclusively for external syslog ingestion |
| `443` (Dashboard) | Optional loopback binding | Depends on whether the dashboard is accessed remotely |

#### Loopback binding of the Management Ports

Container-to-container communication continues uninterrupted over Docker's internal network, so the dashboard keeps working when the management ports are no longer reachable from outside.

1. **Restore point and baseline** 

Before any change, a restore point of the compose configuration was created, and the listening ports of the unmodified deployment were recorded as the "before" state:

```bash
cd ~/wazuh-docker/single-node
cp docker-compose.yml docker-compose.yml.pre-ports.bak
sudo ss -tulnp | grep -E ':(9200|55000|443|1514|1515|514)\b' | tee ~/ports-before.txt
```

2. **Editing the port mappings**

To edit the port mappings, the configuration file was opened in the text editor:

```bash
nano docker-compose.yml
```

The port mappings of the indexer API and the manager API were changed so that they bind strictly to the loopback interface:

- "9200:9200" to - "127.0.0.1:9200:9200"
- "55000:55000" to - "127.0.0.1:55000:55000"

3. **Validation and restart**

```bash
sudo docker compose config --quiet && echo OK
sudo docker compose up -d
```
>**Note on binding the dashboard:**
>(443:5601) to 127.0.0.1:443:5601 is an optional further measure. It is suitable only if the dashboard is opened exclusively from the host itself; where remote administrators need access, the dashboard remains published, and access is limited by other means.

The expected operational status and listening ports were verified and recorded:

```bash
sudo docker compose ps
sudo ss -tulnp | grep -E ':(9200|55000|443|1514|1515|514)\b' | tee ~/ports-after.txt
```

Local functionality and API responsiveness from the host were confirmed:

```bash
  curl -k -u admin https://localhost:9200 
```
>**Note on expected result:**
> After the loopback binding, ports 9200 and 55000 are listed with `127.0.0.1` instead of `0.0.0.0` and `[::]` in the `ss` output, while 443, 1514, 1515 and 514 remain unchanged. A connection attempt to the indexer API from another machine on the network, such as `curl -k https://<host>:9200`, is expected to fail with a refused or timed-out connection.



### 3. Restricting Access with a Host Firewall (ufw)

A host firewall was configured to limit which networks can reach the Wazuh ports. With Docker, this requires a distinction between two kinds of traffic:

- **Services running directly on the host** (such as SSH) are filtered by `ufw` as usual.
- **Docker-published container ports** (such as 1514/TCP) are not filtered by `ufw`. Docker publishes these ports through its own iptables rules (NAT and the `FORWARD` chain), so the traffic never passes through the `INPUT` chain, where `ufw` evaluates its rules. A rule such as `ufw deny 9200` may therefore have no effect on a published container port, and the default-deny policy of `ufw` does not apply to these ports either.

For this reason, loopback binding is the more reliable method for the management APIs, and the ports that must stay reachable for agents and syslog sources (1514/TCP, 1515/TCP and 514/UDP) are restricted in Docker's `DOCKER-USER` chain, which Docker evaluates before its own rules.

`ufw` is nevertheless used in two roles: it protects the services that run directly on the host, and it provides a persistent place for the `DOCKER-USER` rules. These rules are appended to `/etc/ufw/after.rules` (step 5 below), so that they are loaded together with `ufw` whenever it starts, including after a reboot. The configuration was carried out in the following steps:


1. **Gathering network information**

The trusted subnet and the LAN interface were taken from the routing table, and it was checked whether an SSH server is running:

```bash 
ip -4 route
sudo ss -tlnp | grep ':22 '
sudo ufw status verbose
```

2. **SSH access**

No SSH server was listening on the host (the check in step 1 returned no output), so no SSH rule was required and none was added.

Where an SSH server is running, access should be allowed before the firewall is enabled to prevent a lockout, and limited to the trusted subnet instead of the whole network:


3. **Default policies**

All incoming traffic was blocked by default, and all outgoing traffic was allowed:

```bash 
sudo ufw default deny incoming
sudo ufw default allow outgoing
grep -E '^DEFAULT_(INPUT|OUTPUT)_POLICY' /etc/default/ufw
```

4. **Host-level allow rules for Wazuh traffic**

Port 1514/TCP (agent connections), 1515/TCP (agent registration) and 514/UDP (syslog) were opened for the trusted source network only (the example subnet <YOUR_LAN_SUBNET> stands for the specific local network). These rules govern services that listen directly on the host; for the Docker-published ports, the equivalent restriction is enforced in the next step:

```bash 
sudo ufw allow from <YOUR_LAN_SUBNET> to any port 1514 proto tcp
sudo ufw allow from <YOUR_LAN_SUBNET> to any port 1515 proto tcp
sudo ufw allow from <YOUR_LAN_SUBNET> to any port 514 proto udp
sudo ufw show added
```

5. **Restriction of the Docker-published ports**

After a backup of `/etc/ufw/after.rules`, a block was appended that drops traffic to the agent and syslog ports when it arrives on the LAN interface from outside the trusted subnet. All other traffic is returned to Docker.

```bash
IFACE=<YOUR_LAN_IFACE>
SUBNET=<YOUR_LAN_SUBNET>
sudo cp /etc/ufw/after.rules /etc/ufw/after.rules.bak
sudo tee -a /etc/ufw/after.rules > /dev/null <<EOF

# BEGIN WAZUH DOCKER RESTRICTIONS
*filter
:DOCKER-USER - [0:0]
-A DOCKER-USER -i $IFACE -p tcp -m multiport --dports 1514,1515 ! -s $SUBNET -j DROP
-A DOCKER-USER -i $IFACE -p udp --dport 514 ! -s $SUBNET -j DROP
-A DOCKER-USER -j RETURN
COMMIT
# END WAZUH DOCKER RESTRICTIONS
EOF
```

The result was checked before the firewall was enabled. The first command compares the current file with the backup and shows only the differing lines. The second shows the end of the file:

```bash
sudo diff /etc/ufw/after.rules /etc/ufw/after.rules.bak
sudo tail -n 15 /etc/ufw/after.rules
```

**Output of `diff`:**

```text
31,38d30
< # BEGIN WAZUH DOCKER RESTRICTIONS
< *filter
< :DOCKER-USER - [0:0]
< -A DOCKER-USER -i <YOUR_LAN_IFACE> -p tcp -m multiport --dports 1514,1515 ! -s <YOUR_LAN_SUBNET> -j DROP
< -A DOCKER-USER -i <YOUR_LAN_IFACE> -p udp --dport 514 ! -s <YOUR_LAN_SUBNET> -j DROP
< -A DOCKER-USER -j RETURN
< COMMIT
< # END WAZUH DOCKER RESTRICTIONS
```

**Output of `tail`:**

```text
-A ufw-after-input -p udp --dport 68 -j ufw-skip-to-policy-input

# don't log noisy broadcast
-A ufw-after-input -m addrtype --dst-type BROADCAST -j ufw-skip-to-policy-input

# don't delete the 'COMMIT' line or these rules won't be processed
COMMIT
# BEGIN WAZUH DOCKER RESTRICTIONS
*filter
:DOCKER-USER - [0:0]
-A DOCKER-USER -i <YOUR_LAN_IFACE> -p tcp -m multiport --dports 1514,1515 ! -s <YOUR_LAN_SUBNET> -j DROP
-A DOCKER-USER -i <YOUR_LAN_IFACE> -p udp --dport 514 ! -s <YOUR_LAN_SUBNET> -j DROP
-A DOCKER-USER -j RETURN
COMMIT
# END WAZUH DOCKER RESTRICTIONS
EOF
```

The `diff` output contains only the eight lines of the `WAZUH DOCKER RESTRICTIONS` block (lines 31 to 38 of the current file, marked with `<`). No other line of the original ufw rules was changed or removed. The `tail` output shows the block once at the end of the file, after the `COMMIT` line of the ufw rules.

>**Note on the Scope of the `DOCKER-USER` rules:**
> - `DOCKER-USER` is the chain that Docker evaluates first for traffic to containers, before its own rules.
> - The `DOCKER-USER` rules apply to IPv4 only. The agent and syslog ports are also published on IPv6 (`[::]`). IPv6 connections to these ports reach the `docker-proxy` process on the host and are therefore subject to the default-deny inbound policy of `ufw`. 


6. **Enabling the firewall and verifying its status**

The firewall was enabled, and the active rules were checked:

```bash 
sudo ufw enable
sudo ufw status verbose
```
**Output `ufw status verbose`:**

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
1514/tcp                   ALLOW IN    <YOUR_LAN_SUBNET>          
1515/tcp                   ALLOW IN    <YOUR_LAN_SUBNET>          
514/udp                    ALLOW IN    <YOUR_LAN_SUBNET>  
```

The output confirms the following:
- **Status:** `ufw` is active and is started automatically at boot.
- **Defaults:** incoming traffic is denied and outgoing traffic is allowed, as set in step 3.
- **Allow rules:** only 1514/tcp, 1515/tcp and 514/udp are allowed, and only from the trusted subnet.
- **Routed traffic:** the default `deny (routed)` is a `ufw` default for forwarded traffic and was not set manually. It does not restrict the Docker-published ports: Docker handles them through its own NAT and `FORWARD` rules before `ufw` evaluates anything, which is why the `DOCKER-USER` rules from step 5 are required.

```bash
sudo iptables -S DOCKER-USER
```
**Output of `iptables -S DOCKER-USER`:**

```text
-N DOCKER-USER
-A DOCKER-USER ! -s <YOUR_LAN_SUBNET> -i <YOUR_LAN_IFACE> -p tcp -m multiport --dports 1514,1515 -j DROP
-A DOCKER-USER ! -s <YOUR_LAN_SUBNET> -i <YOUR_LAN_IFACE> -p udp -m udp --dport 514 -j DROP
-A DOCKER-USER -j RETURN
```

The output confirms the following:
- **Chain:** the `DOCKER-USER` chain exists and contains exactly the rules from step 5.
- **Drop rules:** TCP 1514 and 1515 and UDP 514 are dropped when they arrive on the LAN interface from a source outside the trusted subnet. `iptables` prints the negated source (`! -s`) first and adds `-m udp` itself, so the rules match what was written to `after.rules`.
- **Return rule:** all other traffic is handed back to Docker's own rules, so local access and the dashboard are not affected.


7. **Verification of the exposure restrictions**

The effect of the restrictions was verified in two parts: the listening ports on the host, and connection tests from a second device on the local network.

**Listening ports on the host**

The listening ports were recorded after the changes:

```bash
sudo ss -tulnp | grep -E ':(9200|55000|443|1514|1515|514)\b' | tee ~/ports-after.txt
```

Ports 9200 and 55000 are bound to `127.0.0.1`, while 443, 1514, 1515 and 514 remain published on all interfaces. The latter are restricted by the `DOCKER-USER` rules from step 5.

**Connection tests from a second device**

The tests were run from a second device (a Mac) connected to the trusted subnet. `<HOST_LAN_IP>` is used as the example address of the Wazuh host.

The management APIs must not be reachable over the network:

```bash
curl -k --connect-timeout 5 https://<HOST_LAN_IP>:9200
curl -k --connect-timeout 5 https://<HOST_LAN_IP>:55000
```
**Output:**

```text
curl: (28) Failed to connect to <HOST_LAN_IP> port 9200 after 5006 ms: Timeout was reached
curl: (28) Failed to connect to <HOST_LAN_IP> port 55000 after 5002 ms: Timeout was reached
```
Both connections timed out, so the ports are not reachable from the network.


The agent port must be reachable from the trusted subnet:

```bash
nc -zv -G 5 <HOST_LAN_IP> 1514
```
**Output:**

```text
Connection to <HOST_LAN_IP> port 1514 [tcp/fujitsu-dtcns] succeeded!
```

The connection succeeded. (`fujitsu-dtcns` is only the service name that `nc` looks up for port 1514 and has no meaning here.)


As proof that the `DOCKER-USER` chain takes effect on published container ports, a temporary rule blocking the test device (in this case, a Mac) was inserted on the Wazuh host:

```bash
sudo iptables -I DOCKER-USER 1 -i <YOUR_LAN_IFACE> -p tcp --dport 1514 -s <<MAC_IP>> -j DROP
```
Check on Mac, if the command will be blocked as expected:

```bash
nc -zvw3 <HOST_LAN_IP> 1514
```

**Output Mac:**

```text
nc: connectx to <HOST_LAN_IP> port 1514 (tcp) failed: Operation timed out
```

As a control test, the dashboard port was checked from the same device, to show that the Wazuh host itself is reachable:

```bash
curl -k -s -o /dev/null -w "%{http_code}\n" --connect-timeout 5 https://<HOST_LAN_IP>
```

**Output:**

```text
302
```

The dashboard responded with a redirect (to its login page), so the network path between the two devices works. The timeouts on ports 9200 and 55000 therefore result from the restrictions and not from a connectivity problem.


The packet counter of the temporary rule was checked, and the rule was removed again. The rule must be deleted with exactly the same parameters it was inserted with:

```bash
sudo iptables -L DOCKER-USER -n -v
sudo iptables -D DOCKER-USER -i <YOUR_LAN_IFACE> -p tcp --dport 1514 -s <<MAC_IP>> -j DROP
sudo iptables -S DOCKER-USER
```

The `nc` test was repeated a final time. Expected: the connection succeeds again.

**Output of nc -zvw3 <HOST_LAN_IP> 1514**

```text
Connection to <HOST_LAN_IP> port 1514 [tcp/fujitsu-dtcns] succeeded!
```


## Summary of Hardening Measures

- **Credentials:** The default password and hash of the indexer `admin` user were replaced. The default credentials of the `kibanaserver` and `wazuh-wui` users were not changed in this lab. This is documented as an accepted risk: the management APIs are bound to the loopback interface, and the dashboard is reachable only from a trusted network. For any shared or production deployment, both credentials should be replaced with the same procedure as for `admin`.
- **Management ports and dashboard (9200, 55000, 443):** These ports are bound to the loopback interface in docker-compose.yml ("127.0.0.1:9200:9200", "127.0.0.1:55000:55000" and "127.0.0.1:443:5601") and are therefore no longer reachable from other hosts on the network. They remain accessible from the host itself, and internal communication between the containers is unaffected.
- **Agent and syslog ports (1514/TCP, 1515/TCP, 514/UDP):** These ports remain reachable for agents and syslog sources, but only from the trusted subnet. The restriction is enforced through `DOCKER-USER` rules in `/etc/ufw/after.rules` and applies to IPv4 traffic arriving on the configured LAN interface.
- **Host firewall:** `ufw` blocks unsolicited incoming connections to services running directly on the host (default-deny inbound policy, with SSH allowed only from the trusted subnet) and keeps the `DOCKER-USER` rules together with the other firewall rules in `/etc/ufw/after.rules`. It does not filter Docker-published ports.
- **Dashboard certificate:** The dashboard keeps the self-signed certificate generated during setup. This is acceptable for the lab, because the dashboard is bound to the loopback interface and is therefore opened only from the host itself. For longer-lived or shared deployments, a certificate issued by a trusted CA, or a reverse proxy in front of the dashboard, is the more suitable option.
- **Key finding:** Docker publishes ports through its own iptables rules, which bypass `ufw`. A rule such as `ufw deny 9200` may therefore have no effect on published container ports. Loopback binding is the most reliable method for ports that need no external access, and `DOCKER-USER` rules are the appropriate method for ports that must stay reachable.

