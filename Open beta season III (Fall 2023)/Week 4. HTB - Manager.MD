# HTB - Manager

### TL;DR


__Keywords:__<br>
<br>

__References:__<br>

<br>
### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-21 16:51 EDT
Nmap scan report for manager.htb (10.129.142.151)
Host is up, received user-set (0.32s latency).
Not shown: 65515 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE        REASON
53/tcp    open  domain         syn-ack
80/tcp    open  http           syn-ack
88/tcp    open  kerberos-sec   syn-ack
135/tcp   open  msrpc          syn-ack
139/tcp   open  netbios-ssn    syn-ack
389/tcp   open  ldap           syn-ack
445/tcp   open  microsoft-ds   syn-ack
464/tcp   open  kpasswd5       syn-ack
593/tcp   open  http-rpc-epmap syn-ack
636/tcp   open  ldapssl        syn-ack
1433/tcp  open  ms-sql-s       syn-ack
3268/tcp  open  ldap           syn-ack
5985/tcp  open  wsman          syn-ack
9389/tcp  open  adws           syn-ack
49667/tcp open  unknown        syn-ack
49677/tcp open  unknown        syn-ack
49678/tcp open  unknown        syn-ack
49679/tcp open  unknown        syn-ack
50489/tcp open  unknown        syn-ack
50568/tcp open  unknown        syn-ack
65495/tcp open  unknown        syn-ack

Nmap done: 1 IP address (1 host up) scanned in 1379.15 seconds
```
