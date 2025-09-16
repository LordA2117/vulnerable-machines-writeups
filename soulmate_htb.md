# Soulmate: HackTheBox

Difficulty: Easy (very shit machine ngl)

## Enumeration
Port 80 -> 2 subdomains soulmate.htb and ftp.soulmate.htb 

ftp subdomain is protected by CrushFTP which is vulnerable to the login bypass, mainly [this exploit](https://github.com/Immersive-Labs-Sec/CVE-2025-31161/blob/main/cve-2025-31161.py).

So using this exploit create a new user, and then using this new admin user, change the password of the ben account, and login using this.

Using the new login as ben, modify the config of the `webprod/` folder and allow file uploads. This is where we'll upload our reverse shell. I uploaded my own custom webshell in php to gain a basic webshell but you can also use the [p0wny shell](https://github.com/flozz/p0wny-shell/blob/master/shell.php).

Use this to gain a standard reverse shell.

After the getting the shell, in `/var/www/html/soulmate.htb/data` I found this in the db entry

```
1|admin|$2y$12$u0AC6fpQu0MJt7uJ80tM.Oh4lEmCMgvBs3PwNNZIR7lor05ING3v2
```

So let's crack the password

This didnt yield much, but on checking the stuff in linpeas.sh this stood out

```bash
root        1143  0.0  1.6 2251672 67380 ?       Ssl  05:03   0:06 /usr/local/lib/erlang_login/start.escript -B -- -root /usr/local/lib/erlang -bindir /usr/local/lib/erlang/erts-15.2.5/bin -progname erl -- -home /root -- -noshell -boot no_dot_erlang -sname ssh_runner -run escript start -- -- -kernel inet_dist_use_interface {127,0,0,1} -- -extra /usr/local/lib/erlang_login/start.escript
```

So I checked this script, located in `/usr/local/lib/erlang_login/start.escript` got the credentials to ben user. This yields user flag.

Credentials:

```
{"ben", "HouseH0ldings998"}
```

## Privilege Escalation

To gain root, all you really need to do is to check the running services. Here you can see a service running in port 2222. This is the Erlang SSH service. Connect to it using

```bash
ssh 127.0.0.1 -p 2222
```

Then use the password of `ben` to gain access.

Use the command `os:cmd("cat /root/root.txt").` to get the flag.
