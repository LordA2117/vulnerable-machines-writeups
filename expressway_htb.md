# Expressway: HackTheBox

Difficulty: Easy (very non-standard box)

## Enumeration

On doing masscan we see only port 22 open. But I tried other ways to check the open ports (mainly nmap -sU -F) and it returned port 500. Idk why masscan didnt work because I use the "U" flag there also. I figured this out only due to a shit ton of hints from htb discord.

## Initial Foothold

So to pentest isakmp I used [this article](https://blog.silentsignal.eu/2014/04/17/isakmp-aggressive-psk-tools/)

So I used these 2 snippets:

```bash
sudo ike-scan -A --trans=5,2,1,2 --id=vpnclient -Ppsk.txt 10.129.174.173
```

This gives a username `ike@expressway.htb`

```bash
psk-crack -d /usr/share/wordlists/rockyou.txt psk   
```

This will give the password

From this you can see this set of credentials:

```
ike@expressway.htb:freakingrockstarontheroad
```

## Privilege Escalation

Just use [this script](https://github.com/pr0v3rbs/CVE-2025-32463_chwoot/blob/main/sudo-chwoot.sh) after initial access to run as root.

Overall pretty mid box (just user access, but its my skill issue I guesss). Aside from that pretty nice, HIHGLY beginner oriented lol.

Happy Hacking :)
