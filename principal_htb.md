# Principal: A HackTheBox Machine


## Enumeration

- Port 22,8080 are open
- Port 8080 is running 

- Port 8080 has an admin dashboard which cannot be logged into (for now).
- Looking at the page source in the browser inspector, we can find a `/static/js/app.js`.
- Here we can see that it is creating a JWT and then using a JWE to encrypt the token, which is then used as an authorization header.

## Initial Foothold

- In the main login page, we can see that the site uses pac4j, which is vulnerable to [this exploit](https://www.cve.news/cve-2026-29000/).
- I used this [public PoC](https://github.com/c0gnit00/CVE-2026-29000/blob/main/exploit.py) to forge the JWE.
- Just set the username to admin and role to be **ROLE_ADMIN**, which allows you to get a valid token.
- Modify the browser session storage, by setting **auth_value** to your **JWE**.
- This will allow you to login to the dashboard found in `/dashboard`.
- In the settings page there is an exposed password. This allows you to login to the box. 
- To get the username we can simply bruteforce the SSH login using the usernames provided in the *users* page.

```bash
hydra ssh <ip> -L users.txt -p <obtained_password>
```

- This command gives us the `svc-deploy` user, which then can be used to gain SSH login and get the user flag.


## Privilege Escalation

- By running the `id` command we see that we also belong to the **deployers** group.
- The deployers group has access to read `/opt/principal/ssh`. Here we can see a keypair with a private key in file `ca` and public key in `ca.pub`

- By seeing the config here `/etc/ssh/sshd_config.d/60-principal.conf` we get this:

```
# Principal machine SSH configuration
PubkeyAuthentication yes
PasswordAuthentication yes
PermitRootLogin prohibit-password
TrustedUserCAKeys /opt/principal/ssh/ca.pub
```

- This means that root login is allowed in SSH but is blocked by a password. But, we can see a trusted CA key set to be our `ca` keypair.
- This means that we can use these keys to make our own SSH keys, sign them with the principal as root and get root access.
- First create the ssh keypair as follows:

```bash
ssh-keygen -t rsa-sha2-256 -b 2048
```

- Then sign the keypair using the keypair in `/opt/principal/ssh`:

```bash
ssh-keygen -s /opt/principal/ssh/ca -I hacker -n root -V +5h .ssh/id_rsa.pub
```

- SSH as root:

```bash
ssh -i id_rsa root@127.0.0.1
```

- This gives root.
- If you want a shortcut, yes, copy-fail exploit also works here.


Happy Hacking :)
