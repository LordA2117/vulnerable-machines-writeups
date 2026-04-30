# Pterodactyl: HackTheBox

Difficulty: Medium (though imo it's pretty easy)

## Enumeration

- Port 22,80 open
- Port 80 has subdomains panel.pterodactyl.htb

- It runs a version of Pterodactyl panel vulnerable to [this exploit](https://github.com/dollarboysushil/CVE-2025-49132-Pterodactyl-Panel-Unauthenticated-Remote-Code-Execution-RCE-).
- The phpinfo asked for in this can be found at http://pterodactyl.htb/phpinfo.php


## Pivoting

- So on gaining access we go to /var/www/pterodactyl and read the .env containing db creds.

```bash
mysql -u pterodactyl -h 127.0.0.1 -pPteraPanel
```

- We find 2 password hashes in the users table, cracking one of them gives the password for `phileasfogg3` being `!QAZ2wsx`
- Use this to login via ssh and gain user.

## Privilege Escalation

- Again we can do the brand-new [copy fail exploit](https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py).
- Since this runs an older version of python, we'll make a full executable and run that as so:

```bash
pyinstaller --onefile exploit.py
```

- This creates the exploit, download this into the victim and run it, you get root.

Happy Hacking :)
