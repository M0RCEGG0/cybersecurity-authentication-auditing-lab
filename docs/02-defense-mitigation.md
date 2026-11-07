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



\### The Vulnerability Mechanic

In Scenario 3, THC-Hydra automated HTTP GET/POST requests against the DVWA authentication endpoint, extracting valid credentials by evaluating differences in the HTML response stream.



\### Defensive Countermeasures



\#### A. CAPTCHA Integration (Human Verification)

\* After 2 or 3 failed login attempts, enforce challenges such as Cloudflare Turnstile or Google reCAPTCHA.

\* Automated command-line tools like Hydra lack the headless browser and computer vision capabilities required to solve interactive challenges, halting execution immediately.



\#### B. Anti-Automation Tokens (CSRF)

\* Generate unique, cryptographically strong, single-use anti-CSRF tokens tied to the server-side user session on every form render.

\* Because the token changes with every page request, automated multi-threaded attacks that submit static request strings fail validation.



\#### C. Web Application Firewall (WAF) Rate Limiting

\* Deploy a reverse-proxy WAF (e.g., Cloudflare, AWS WAF, ModSecurity) configured with aggressive rate limiting on authentication routes (e.g., `/login.php`).

\* Restrict requests to a maximum of 5 attempts per minute per source IP, returning an HTTP `429 Too Many Requests` status code upon violation.

