\# 🛡️ Defensive Hardening \& Mitigation Strategies



\## 1. Core Defensive Philosophy: Defense-in-Depth



In professional cybersecurity, security should never rely on a single lock. If an adversary bypasses the perimeter or first line of defense, subsequent layers must detect, throttle, or neutralize the intrusion. 



Authentication mechanisms represent the primary gatekeeper for servers and applications. When services such as FTP, SMB, or Web portals allow unrestricted login attempts, attackers leverage high-speed automation tools like \*\*Medusa\*\* and \*\*THC-Hydra\*\* to exhaust dictionary spaces. 



This document details the defensive controls, architectural hardening, and modern Blue Team methodologies designed to systematically defeat the attack vectors demonstrated in our laboratory.



\---



\## 2. FTP Hardening (Port 21) — Neutralizing High-Speed Brute Force



\### The Vulnerability Mechanic

In Scenario 1, Medusa executed hundreds of login requests in seconds against the vsftpd service. The service responded directly to raw TCP requests without artificial latency, rate constraints, or connection limits.



\### Defensive Countermeasures



\#### A. Protocol Modernization (SFTP / FTPS)

\* \*\*The Vulnerability:\*\* Legacy FTP transmits authentication credentials, commands, and file contents in cleartext across the wire, susceptible to network sniffing.

\* \*\*The Solution:\*\* Deprecate unencrypted FTP in favor of \*\*SFTP\*\* (SSH File Transfer Protocol, running over port 22) or \*\*FTPS\*\* (FTP over TLS). SFTP leverages robust public-key cryptography and eliminates plaintext transmission.



\#### B. Automated Intrusion Prevention with Fail2ban

Fail2ban functions as an automated digital bouncer by actively inspecting system authentication logs (e.g., `/var/log/auth.log` or `/var/log/vsftpd.log`):

\* Every time Medusa triggers a failed authentication attempt, the daemon writes an audit failure entry.

\* If Fail2ban identifies \*\*3 failed attempts within a 10-minute window\*\* (`maxretry = 3`, `findtime = 600`), it automatically injects a dynamic firewall rule into Linux `iptables` or `nftables` to drop all network packets from the offending source IP for \*\*1 hour\*\* (`bantime = 3600`).

\* \*\*Operational Impact:\*\* Medusa is completely severed from the network on its 3rd attempt, making dictionary attacks mathematically impossible.



```ini

\[vsftpd]

enabled  = true

port     = ftp,ftp-data,ftps,ftps-data

filter   = vsftpd

logpath  = /var/log/vsftpd.log

maxretry = 3

findtime = 600

bantime  = 3600

```



\#### C. Account Sanitization \& Root Prohibition

\* \*\*Default Account Removal:\*\* Eliminate factory default accounts (`msfadmin`, `test`, `guest`, `service`).

\* \*\*Root Login Restriction:\*\* Explicitly prohibit the `root` superuser from authenticating directly via file transfer protocols (`/etc/ftpusers`).



\---



\## 3. SMB Hardening (Port 445) — Breaking the Attack Chain



\### The Vulnerability Mechanic

Scenario 2 demonstrated a three-stage kill chain:

1\. \*\*Reconnaissance:\*\* `enum4linux` extracted every valid system username via unauthenticated Null Sessions.

2\. \*\*Exploitation:\*\* `medusa` executed targeted Password Spraying using the discovered username corpus.

3\. \*\*Post-Exploitation:\*\* `smbclient` established an interactive session inside administrative shares (`ADMIN$`, `/tmp`).



Severing the initial reconnaissance stage breaks the entire attack chain downstream.



\### Defensive Countermeasures



\#### A. Disabling Null Sessions (Anonymous IPC$)

The enumeration succeeded because the Samba service allowed unauthenticated connections to poll user metadata via MSRPC:

\* In `/etc/samba/smb.conf`, configure:

&#x20; ```ini

&#x20; \[global]

&#x20; restrict anonymous = 2

&#x20; null passwords = no

&#x20; ```

\* \*\*Why it works:\*\* Setting `restrict anonymous = 2` forces the server to reject any connection that has not completed prior authentication with a verified account. As a result, tools like `enum4linux` return zero users, leaving the adversary blind.



\#### B. Network Segmentation \& Isolated VLANs

Internal file-sharing services (SMB) should never reside on the same network segment as general workstation clients or guest Wi-Fi. Placing storage servers within an isolated \*\*VLAN\*\* governed by stateful firewall rules ensures that only authorized management subnets can initiate port 445 handshakes.



\---



\## 4. Modern Enterprise Defense Against Password Spraying



\### Why Traditional Lockout Fails

Password spraying is specifically designed to circumvent conventional \*\*Account Lockout Policies\*\*. If an enterprise policy locks an account after 5 consecutive failed attempts, an attacker tests only \*\*1 password per account every hour\*\*. The account threshold is never reached, allowing the attack to persist undetected for days or weeks.



\### Modern Remediation Blueprint



\#### A. Multi-Factor Authentication (MFA / 2FA)

MFA serves as the definitive mitigation against credential discovery:

\* Even when an adversary uncovers valid credentials (such as `msfadmin:msfadmin` or corporate patterns like `Summer2026!`), access cannot be established without the secondary, time-sensitive hardware or cryptographic token (e.g., FIDO2 hardware keys, Microsoft Authenticator, TOTP).



\#### B. Behavioral Smart Lockout (Azure AD / Modern IAM)

Rather than locking the victim's account and causing a widespread Denial-of-Service (DoS) condition across the enterprise, modern identity providers employ \*\*Smart Lockout\*\*:

\* The authentication platform evaluates client telemetry: source IP reputation, geolocation, device fingerprint, and velocity.

\* When anomalous failures occur across multiple accounts from a single unknown IP, the system \*\*blocks the offending IP address\*\* while keeping the actual user accounts unlocked for legitimate corporate employees.



\#### C. Banned Password Lists \& Breached Credential Screening

Preventing users from adopting predictable patterns is critical:

\* Integrate Active Directory Password Protection or HaveIBeenPwned API screening.

\* Block terms containing the company name, current seasons, incremental digits (`123456`), or known leaked corpus entries at the time of password creation.



\#### D. Centralized SIEM Detection Rules

Configure Security Information and Event Management (SIEM) correlation alerts:

\* \*\*Detection Rule:\*\* Trigger a High-Severity alert when a single internal or external IP address generates single authentication failure events across 5 or more distinct usernames within a 15-minute sliding window.



\---



\## 5. Web Application Form Defense (Port 80 - DVWA)

# Defensive Hardening & Mitigation Strategies

## 1. Core Defensive Philosophy: Defense-in-Depth

In professional cybersecurity, security should never rely on a single lock. If an adversary bypasses the perimeter or first line of defense, subsequent layers must detect, throttle, or neutralize the intrusion.

Authentication mechanisms represent the primary gatekeeper for servers and applications. When services such as FTP, SMB, or Web portals allow unrestricted login attempts, attackers leverage high-speed automation tools like Medusa and THC-Hydra to exhaust dictionary spaces.

This document details the defensive controls, architectural hardening, and modern Blue Team methodologies designed to systematically defeat the attack vectors demonstrated in our laboratory.

---

## 2. Threat Mitigation Matrix

| Target Vector | Exploit Mechanism | Impact Severity | Primary Remediation Control |
| :--- | :--- | :---: | :--- |
| **FTP (Port 21)** | Dictionary Brute Force | **HIGH** | Fail2ban IP jail + Migration to SFTP |
| **SMB (Port 445)** | Null Session Enumeration | **HIGH** | Set `restrict anonymous = 2` |
| **SMB (Port 445)** | Low-Noise Password Spraying | **CRITICAL** | Enterprise MFA + Smart Behavioral Lockout |
| **Web Form (Port 80)** | Automated HTTP Session Post | **HIGH** | Rate Limiting + CAPTCHA + Anti-CSRF |

---

## 3. Host Hardening: FTP Service (Port 21)

Unencrypted legacy FTP transmits authentication tokens in cleartext over TCP, rendering internal traffic vulnerable to sniffing and credential theft.

### A. Protocol Modernization (SFTP / FTPS)
* **The Vulnerability:** Legacy FTP transmits credentials, commands, and file contents in cleartext.
* **The Solution:** Deprecate unencrypted FTP in favor of SFTP (SSH File Transfer Protocol, running over port 22) or FTPS (FTP over TLS). SFTP leverages public-key cryptography and eliminates plaintext transmission.

### B. Dynamic Jailing with Fail2ban
Deploy Fail2ban to actively inspect service logs and dynamically block attacker IP addresses:

```ini
[vsftpd]
enabled  = true
port     = ftp,ftp-data,ftps,ftps-data
filter   = vsftpd
logpath  = /var/log/vsftpd.log
maxretry = 3
findtime = 600
bantime  = 3600
```

### C. Account Isolation
* Restrict root login explicitly within `/etc/ftpusers`.
* Disable legacy and factory default user accounts (`msfadmin`, `test`, `service`).

---

## 4. Host Hardening: SMB Service (Port 445)

The success of Password Spraying relies entirely on unauthenticated reconnaissance. Severing directory enumeration at the boundary renders the subsequent spraying attack blind.

### A. Null Session Disablement (Anonymous IPC$)
Reconfigure the Samba configuration file (`/etc/samba/smb.conf`) to enforce mandatory pre-authentication before exposing metadata:

```ini
[global]
   restrict anonymous = 2
   null passwords = no
   server min protocol = SMB2_10
   client min protocol = SMB2_10
```

### B. Network Segmentation
Isolate internal network shares and enterprise storage pools behind dedicated management VLANs. Access control lists (ACLs) must restrict SMB traffic (TCP 445) exclusively to trusted workstation subnets.

---

## 5. Enterprise Identity Protection: Defeating Password Spraying

Traditional account lockout policies lock an account after multiple failed attempts on that specific account. In Password Spraying, an adversary circumvents this threshold by attempting only **one password per user per cycle**, remaining below the threshold while testing thousands of corporate identities.

### Strategic Countermeasures for Financial Institutions

1. **Mandatory Multi-Factor Authentication (MFA / 2FA):**
   Even when an adversary correctly guesses a valid corporate credential, authentication is blocked without the physical, cryptographic possession factor (e.g., FIDO2 hardware keys, Microsoft Authenticator, TOTP).
2. **Behavioral Smart Lockout:**
   Implement AI-driven risk scoring (e.g., Microsoft Entra ID Smart Lockout). The engine correlates geographic anomaly, IP reputation, and velocity to ban the offending IP while keeping corporate accounts unlocked.
3. **Breached Credential Defense:**
   Incorporate dynamic password filters that block common corporate templates (`Company@2026`, `Winter@123`) and passwords present in known leaked databases.
4. **SIEM Correlation Rules:**
   Configure Security Information and Event Management (SIEM) alerts to trigger a high-severity alert when a single internal or external IP address generates failed authentication events across 5 or more distinct usernames within a 10-minute window.

---

## 6. Web Application Hardening (Port 80 - DVWA)

### A. Layer 7 Rate Limiting & CAPTCHA
* Deploy Cloudflare Turnstile or Google reCAPTCHA dynamically when failed login attempts exceed 3.
* Automated CLI engines like THC-Hydra cannot execute JavaScript challenges or visual proofs.

### B. Anti-Automation Tokens (CSRF Mitigation)
* Generate non-predictable, cryptographically signed, single-use anti-CSRF tokens for every rendered login form.
* Requests submitted without a valid token pair matching the active server session are rejected with HTTP 403 Forbidden.

### C. Web Application Firewall (WAF) Rate Limiting
* Configure WAF rate-limiting rules targeting `/login` endpoints: throttle to maximum 5 requests/minute per client fingerprint, returning HTTP 429 Too Many Requests.