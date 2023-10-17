# HTB - Visual
### TL;DR



### PHASE 1: SERVICE ENUMERATION
Launch a port scan with `nmap`
```markdown
nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-02 06:42 EDT
Stats: 0:04:21 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Nmap scan report for 10.129.131.71
Host is up, received user-set (0.20s latency).
Not shown: 65534 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 495.11 seconds
```
Scan the open services found
```markdown
nmap $target -p80 -sV -sC -Pn -v      
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-02 06:53 EDT
Nmap scan report for 10.129.131.71
Host is up (0.15s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.1.17)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.1.17
|_http-favicon: Unknown favicon MD5: 556F31ACD686989B1AFCF382C05846AA
|_http-title: Visual - Revolutionizing Visual Studio Builds

Nmap done: 1 IP address (1 host up) scanned in 13.43 seconds
```
So there is an Apache 2.4.56 web server running on port 80. We can inspect the site in firefox entering the url http://visual.htb









