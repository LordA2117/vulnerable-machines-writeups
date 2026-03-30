# Kobold: A HackTheBox Machine
Difficulty: Easy


## Recon
- Ports 22,80, 443 and 3552 are open
- All traffic to port 80 is redirected to https (port 443)
- Subdomains: bin.kobold.htb, mcp.kobold.htb


## Initial Foothold

- mcp.kobold.htb is vulnerable to RCE via [this PoC](https://github.com/advisories/GHSA-232v-j27c-5pp6). Try and gain the shell on your own or by seeing my previous walkthroughs.
- This gives us a shell as `ben` where we can get the user flag.

## Privilege Escalation

- Through linpeas.sh we can see that our user can join any group. In our current state we're unable to execute any docker commands.
- So we'll create the docker group and run a classic docker container exploit to gain root.
- Add **ben** to the docker group using the command:

```bash
newgrp docker
```

- First, check what containers are installed:

```bash
docker ps -a
```

- We can see that the `privatebin/nginx-fpm-alpine:2.0.2`. So let's run this in privileged mode.

```bash
docker run --rm -it -u 0 --entrypoint sh -v /:/mnt privatebin/nginx-fpm-alpine:2.0.2
```

- Once we're in the docker container we'll go from there to a root shell.

```bash
chroot /mnt sh
```

That's it! We've pwned kobold.

Happy Hacking :)
