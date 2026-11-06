# Attack Scenarios and Technical Execution

## Target Architecture & Laboratory Setup
* **Attacker Machine:** Kali Linux (Medusa v2.2+)
* **Target Machine:** Metasploitable 2 - IP `192.168.56.101`
* **Network Topology:** VirtualBox Host-Only Adapter (Isolated Testing Environment)

---

## Scenario 1: Online Brute Force against FTP (Port 21)

### 1. Objective & Threat Vector
Audit authentication resilience on the vsftpd service (port 21/TCP). The attack simulates an external or internal adversary leveraging parallel dictionary attacks to identify default and weak credentials across multiple potential accounts.

### 2. Execution Command
```bash
medusa -h 192.168.56.101 -U users.txt -P passwords.txt -M ftp -f -v 4
```

### 3. Parameter Breakdown
* `-h 192.168.56.101`: Specifies the target host IP.
* `-U users.txt`: Wordlist containing target system and service usernames.
* `-P passwords.txt`: Wordlist containing common and weak password candidates.
* `-M ftp`: Directs Medusa to load the dynamic `ftp.mod` authentication module.
* `-f`: **Stop on Success** — aborts attack threads once valid credentials are confirmed for the target host.
* `-v 4`: High verbosity output displaying real-time authentication checks and thread states.

### 4. Findings & Evidence
During execution, the attack successfully cracked the credentials for the administrative user account:

* **Discovered Credentials:** `msfadmin:msfadmin`
* **Status:** `[SUCCESS]` confirmed by the FTP service response.

![Medusa FTP Attack Evidence](../images/02-medusa-ftp-attack.png)

### 5. Technical Risk Analysis
* **Root Cause:** Use of default administrative credentials and complete absence of authentication rate limiting.
* **Immediate Risk:** An attacker obtaining FTP access can upload malicious payloads, exfiltrate sensitive files, or inspect system directories depending on privilege configurations.