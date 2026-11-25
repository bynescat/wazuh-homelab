# Log analysis: authentication events

This chapter follows authentication events from the monitored VM (`lab`) to the Wazuh manager. Failed SSH logins and a `sudo` session were generated on the VM. Each event was first read as a raw log line in the VM and then located as an alert in the Wazuh dashboard, so that the raw data and the detection result can be compared directly.

The VM, the agent and the network are described in [03 Monitored VM](03-monitored-vm.md) and [04 Wazuh agent](04-wazuh-agent.md).


The following topics are covered:

- generating failed SSH logins (for an existing and for a non-existent user) and a privileged `sudo` session
- reading the corresponding raw lines in the authentication log of the VM
- locating the same events as alerts in the Wazuh dashboard, with rule, severity, extracted fields and MITRE ATT&CK mapping
- counting the failed logins per source address from the raw log with `grep`, `cut`, `sort` and `uniq`

Other log sources of the VM, such as system messages, kernel messages and service errors, are not covered here.

a
## 1. Generating authentication events

Failed SSH logins were generated from a terminal on the host against the host-only address of the VM. Two cases were tested, because they leave different traces in the log: a wrong password for an existing account (`nescat`) and a login attempt with a user name that does not exist on the VM (`wronguser`). In both cases, three wrong passwords were entered:

```bash
ssh nescat@192.168.56.107
ssh wronguser@192.168.56.107
```

## 2. Raw log lines in the VM

The events were read in the VM from the authentication log. The option `-a` makes `grep` treat the file as text, because `grep` otherwise stops with `Binary file /var/log/auth.log matches` as soon as the file contains a non-text byte:

```bash
sudo grep -aE "Invalid user|Failed password" /var/log/auth.log | tail -n 10
```

**Output:**

```text
2026-11-17T16:29:07.068176+00:00 lab sshd[1490]: Invalid user wazuh-agent-vm from 10.0.2.2 port 53274
2026-11-18T11:03:29.663357+00:00 lab sshd[6136]: Failed password for nescat from 192.168.56.1 port 34642 ssh2
2026-11-18T11:03:40.052316+00:00 lab sshd[6136]: message repeated 2 times: [ Failed password for nescat from 192.168.56.1 port 34642 ssh2]
2026-11-18T11:04:10.127069+00:00 lab sshd[6143]: Invalid user wronguser from 192.168.56.1 port 37932
2026-11-18T11:04:13.498951+00:00 lab sshd[6143]: Failed password for invalid user wronguser from 192.168.56.1 port 37932 ssh2
2026-11-18T11:04:16.789089+00:00 lab sshd[6143]: Failed password for invalid user wronguser from 192.168.56.1 port 37932 ssh2
2026-11-18T11:04:21.593709+00:00 lab sshd[6143]: Failed password for invalid user wronguser from 192.168.56.1 port 37932 ssh2
2026-11-18T11:07:18.621315+00:00 lab sudo:    nescat : TTY=pts/0 ; PWD=/home/nescat ; USER=root ; COMMAND=/usr/bin/grep -aE 'Invalid user|Failed password' /var/log/auth.log
```

The log shows the following:

- **Wrong password for an existing user:** the three failed attempts for `nescat` belong to one connection (`sshd[6136]`). The second line is a summary of the logging service: `message repeated 2 times` stands for two further failures. The log therefore holds three failures although only two lines were written, and counting lines alone undercounts the attempts.
- **Non-existent user:** the attempts for `wronguser` belong to another connection (`sshd[6143]`). The service first logs `Invalid user wronguser` and then one line `Failed password for invalid user` per attempt. The wording `for invalid user` distinguishes this case from a failure for an existing account.
- **Source address:** both cases come from `192.168.56.1`, the address of the host on the host-only network. The first line, from 17 November, is an earlier attempt through the NAT port forward, where the VM sees the host as `10.0.2.2` (see [03 Monitored VM](03-monitored-vm.md), step 8).
- **The `sudo` line:** the command that read the log was logged as well, together with the user, the working directory, the target user and the complete command. The log therefore also serves as an audit trail of privileged commands.
- **Time zone:** the time stamps are in UTC (`+00:00`). The dashboard shows the local time of the browser, which is one hour later on the host.


## 3. The same events in Wazuh

The alerts were searched in the Wazuh dashboard under **Discover** with the index pattern `wazuh-alerts-*`, the time range **Last 1 hour** and this query:

```text
agent.name: lab and rule.groups: sshd
```

### Wrong password for an existing user

The first failed password for `nescat` (log line of 11:03:29) produced the following alert:

```json
{
  "_index": "wazuh-alerts-4.x-2026.11.18",
  "_id": "tTtX7qEBsBlJ772X0kIc",
  "_source": {
    "predecoder": {
      "hostname": "lab",
      "program_name": "sshd",
      "timestamp": "Nov 18 11:03:29"
    },
    "agent": {
      "ip": "192.168.56.107",
      "name": "lab",
      "id": "001"
    },
    "manager": {
      "name": "wazuh.manager"
    },
    "data": {
      "srcip": "192.168.56.1",
      "dstuser": "nescat",
      "srcport": "34642"
    },
    "rule": {
      "level": 5,
      "description": "sshd: authentication failed.",
      "groups": ["syslog", "sshd", "authentication_failed"],
      "firedtimes": 1,
      "mitre": {
        "technique": ["Password Guessing", "SSH"],
        "id": ["T1110.001", "T1021.004"],
        "tactic": ["Credential Access", "Lateral Movement"]
      },
      "id": "5760"
    },
    "location": "journald",
    "decoder": {
      "parent": "sshd",
      "name": "sshd"
    },
    "full_log": "Nov 18 11:03:29 lab sshd[6136]: Failed password for nescat from 192.168.56.1 port 34642 ssh2",
    "timestamp": "2026-11-18T11:08:13.338+0000"
  }
}
```

*Excerpt: the compliance mappings (PCI DSS, HIPAA, GDPR, NIST 800-53, TSC, GPG 13) and the display fields of the dashboard were omitted.*

The alert shows the following:

- **Rule:** rule 5760 with the level 5 (medium severity) detects a failed authentication for an existing account. The group `authentication_failed` is shared with the alert for non-existent users, but the group `invalid_login` is missing, because the account exists.
- **Extracted fields:** the decoder split the raw line into the source address (`data.srcip`), the source port (`data.srcport`) and the targeted account (`data.dstuser`). The port `34642` is the same as in the `auth.log` line of this connection, so the alert can be traced back to a single SSH connection.
- **MITRE ATT&CK:** the rule is mapped to the same techniques as the next alert, T1110.001 (Password Guessing) and T1021.004 (SSH).
- **Counter:** `firedtimes` is 1, so this was the first time the rule fired.

### Login attempt with a non-existent user

The log line `Failed password for invalid user wronguser` of 11:04:16 produced the following alert:

```json
{
  "_index": "wazuh-alerts-4.x-2026.11.18",
  "_id": "vDtY7qEBsBlJ772XkUKO",
  "_source": {
    "predecoder": {
      "hostname": "lab",
      "program_name": "sshd",
      "timestamp": "Nov 18 11:04:16"
    },
    "agent": {
      "ip": "192.168.56.107",
      "name": "lab",
      "id": "001"
    },
    "manager": {
      "name": "wazuh.manager"
    },
    "data": {
      "srcuser": "wronguser",
      "srcip": "192.168.56.1"
    },
    "rule": {
      "level": 5,
      "description": "sshd: Attempt to login using a non-existent user",
      "groups": ["syslog", "sshd", "authentication_failed", "invalid_login"],
      "firedtimes": 3,
      "mitre": {
        "technique": ["Password Guessing", "SSH"],
        "id": ["T1110.001", "T1021.004"],
        "tactic": ["Credential Access", "Lateral Movement"]
      },
      "id": "5710"
    },
    "location": "journald",
    "decoder": {
      "parent": "sshd",
      "name": "sshd"
    },
    "full_log": "Nov 18 11:04:16 lab sshd[6143]: Failed password for invalid user wronguser from 192.168.56.1 port 37932 ssh2",
    "timestamp": "2026-11-18T11:09:01.476+0000"
  }
}
```

*Excerpt: the compliance mappings and the display fields of the dashboard were omitted.*

The alert shows the following:

- **Rule:** rule 5710 with the level 5 detects a login attempt with a non-existent user. The groups `sshd`, `authentication_failed` and `invalid_login` can be used as search filters.
- **Extracted fields:** the attempted user name is stored in `data.srcuser` here, whereas the alert for the existing account stores it in `data.dstuser`. The field names therefore depend on the log line and the decoder, and a search for a user name has to take both into account.
- **Raw event:** the field `full_log` contains the unchanged log line, which is identical to the entry in `auth.log`.
- **Counter:** `firedtimes` is 3, which means that this was the third time rule 5710 fired. The counter rises with every matching event.
- **Log source:** the field `location` is `journald` in all alerts, so the agent collected the events from the systemd journal and not from the file `/var/log/auth.log`. The content is the same.

### Privileged command with `sudo`

The `sudo` call at 11:07:18, the command that read the log in section 2, produced a third alert:

```json
{
  "_index": "wazuh-alerts-4.x-2026.11.18",
  "_id": "xTtb7qEBsBlJ772XaEIl",
  "_source": {
    "predecoder": {
      "hostname": "lab",
      "program_name": "sudo",
      "timestamp": "Nov 18 11:07:18"
    },
    "agent": {
      "ip": "192.168.56.107",
      "name": "lab",
      "id": "001"
    },
    "manager": {
      "name": "wazuh.manager"
    },
    "data": {
      "srcuser": "nescat",
      "uid": "1000",
      "dstuser": "root"
    },
    "rule": {
      "level": 3,
      "description": "PAM: Login session opened.",
      "groups": ["pam", "syslog", "authentication_success"],
      "firedtimes": 3,
      "mitre": {
        "technique": ["Valid Accounts"],
        "id": ["T1078"],
        "tactic": ["Defense Evasion", "Persistence", "Privilege Escalation", "Initial Access"]
      },
      "id": "5501"
    },
    "location": "journald",
    "decoder": {
      "parent": "pam",
      "name": "pam"
    },
    "full_log": "Nov 18 11:07:18 lab sudo[6247]: pam_unix(sudo:session): session opened for user root(uid=0) by nescat(uid=1000)",
    "timestamp": "2026-11-18T11:12:03.652+0000"
  }
}
```

*Excerpt: the compliance mappings and the display fields of the dashboard were omitted.*

The alert shows the following:

- **Rule:** rule 5501 with the level 3 (low severity) reports a session opened through the PAM authentication framework. For `sudo` this is the start of a privileged session. The group `authentication_success` marks it as a successful event.
- **Source and target user:** `data.srcuser` is `nescat` and `data.dstuser` is `root`, so the alert records who became which user.
- **Same second as the log line:** the event time of 11:07:18 matches the `sudo` line in `auth.log`. The analysis in section 2 therefore generated an alert itself.


## 4. Raw event and alert side by side

| Raw log line (VM) | Rule | Level | Meaning |
| :--- | :--- | :--- | :--- |
| `Failed password for nescat from 192.168.56.1` | 5760 | 5 | Authentication failure for an existing account (T1110.001) |
| `Failed password for invalid user wronguser from 192.168.56.1` | 5710 | 5 | Login attempt with a non-existent user (T1110.001) |
| `pam_unix(sudo:session): session opened for user root(uid=0) by nescat(uid=1000)` | 5501 | 3 | Privileged session opened with `sudo` (T1078) |

The comparison shows that Wazuh does more than store log lines. The decoder extracts structured fields, the rule assigns a severity and a description, and the MITRE ATT&CK mapping classifies the behavior. The raw line stays available in `full_log` for verification.

## 5. Counting failed logins per source address

A count of the failed password attempts per source address is the basis for recognizing a brute-force attack. The reference value is known, because six wrong passwords were entered in section 1 (three for `nescat`, three for `wronguser`).

### Finding the matching lines

The lines with failed password attempts were listed with `grep`, which prints every line of a file that contains a given text:

```bash
sudo grep -a "Failed password" /var/log/auth.log
```

**Output:**

```text
2026-11-18T11:03:29.663357+00:00 lab sshd[6136]: Failed password for nescat from 192.168.56.1 port 34642 ssh2
2026-11-18T11:03:40.052316+00:00 lab sshd[6136]: message repeated 2 times: [ Failed password for nescat from 192.168.56.1 port 34642 ssh2]
2026-11-18T11:04:13.498951+00:00 lab sshd[6143]: Failed password for invalid user wronguser from 192.168.56.1 port 37932 ssh2
2026-11-18T11:04:16.789089+00:00 lab sshd[6143]: Failed password for invalid user wronguser from 192.168.56.1 port 37932 ssh2
2026-11-18T11:04:21.593709+00:00 lab sshd[6143]: Failed password for invalid user wronguser from 192.168.56.1 port 37932 ssh2
2026-11-18T11:36:59.100932+00:00 lab sudo:    nescat : TTY=pts/0 ; PWD=/home/nescat ; USER=root ; COMMAND=/usr/bin/grep -a 'Failed password' /var/log/auth.log
```

The output contains three kinds of lines:

| Lines | Number | Meaning |
| :--- | :--- | :--- |
| `Failed password for nescat ...` | 1 | One failed login attempt |
| `message repeated 2 times: [ Failed password ... ]` | 1 | A summary of the logging service that stands for **two** further failed attempts |
| `Failed password for invalid user wronguser ...` | 3 | Three failed login attempts for a non-existent user |
| `sudo: ... COMMAND=... 'Failed password' ...` | 5 | Records of earlier searches. They contain the search text, but they are not login attempts. |

### Extracting the source address

Only the source address is needed for the count. The option `-o` makes `grep` print only the part of the line that matches, and `-E` allows a pattern instead of a fixed text. The pattern `from [0-9.]+` stands for the word `from`, a space, and a run of digits and dots:

```bash
sudo grep -a "Failed password" /var/log/auth.log | grep -oE "from [0-9.]+"
```

**Output:**

```text
from 192.168.56.1
from 192.168.56.1
from 192.168.56.1
from 192.168.56.1
from 192.168.56.1
```

The five lines with a login attempt remain. The five `sudo` lines no longer appear, because they contain no `from` followed by an address.

### Counting per address

The addresses were counted with three more commands in the pipeline:

```bash
sudo grep -a "Failed password" /var/log/auth.log | grep -oE "from [0-9.]+" | cut -d" " -f2 | sort | uniq -c
```

- **`cut -d" " -f2`:** splits every line at the spaces and keeps the second part, which is the address.
- **`sort`:** puts identical addresses next to each other, which `uniq` needs.
- **`uniq -c`:** collapses identical neighbors into one line and puts the number of occurrences in front.

**Output:**

```text
      5 192.168.56.1
```

### Lines versus attempts

The count is 5, but six wrong passwords were entered. The difference comes from the summary line:

| Log lines | Lines | Attempts |
| :--- | :--- | :--- |
| `Failed password for nescat` | 1 | 1 |
| `message repeated 2 times` | 1 | 2 |
| `Failed password for invalid user wronguser` | 3 | 3 |
| **Total** | **5** | **6** |

For `nescat`, the logging service wrote the first failure as a normal line. When the same message repeated twice more within a short time, it wrote one summary line instead of two further lines. The three attempts for `wronguser` were several seconds apart and were all written out. The command therefore counts lines, and the exact number of attempts is the number of lines plus the extra attempts hidden in the summary line: 5 + (2 − 1) = 6.

### Findings

- **Lines are not events:** the logging service collapses identical messages into a summary line, so counting lines undercounts the attempts. The summary line has to be taken into account.
- **Self-reference:** the log contains the commands of the analyst, so a search for a phrase also finds the log entries of the search itself. Extracting a pattern that only appears in real login lines, here `from` followed by an address, leaves these entries out.
- **Plausibility check:** a count should be compared with a known value whenever one exists. Here, the known six attempts showed that the line count was one too low.

