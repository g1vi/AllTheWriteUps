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
**80/tcp open  http    syn-ack**

Nmap done: 1 IP address (1 host up) scanned in 495.11 seconds
```
