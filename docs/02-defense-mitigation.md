# 🛡️Defensive Hardening & Mitigation Strategies

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
