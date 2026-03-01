# WingData: A HackTheBox Machine

## Enumeration

- Nmap Scan: Port 22,80 open
- Port 80: 2 vhosts
    - wingdata.htb
    - ftp.wingdata.htb (found by clicking the log in to client portal button).

## Initial Foothold

So ftp.wingdata.htb runs Wing FTP Server 7.4.3 which is vulnerable to [this exploit](https://www.exploit-db.com/exploits/52347). So use this to gain an initial shell (see my other walkthroughs for reverse shell payloads).

## Pivoting

We need to pivot to the user `wacky` to get the user flag. So we'll try to find any data on him in our current user.

In `/opt/wingftpserver/Data/1/settings.xml` we can see that a salt *WingFTP* is being used to salt the passwords. Then we can also see that the hash for wacky's password is in `/opt/wingftpserver/Data/1/users/wacky.xml`. So we'll use this plus hashcat to crack the password.

So make a hashes.txt file like this:

```
<password_hash>:WingFTP
```

Now use hashcat for the password cracking:

```bash
hashcat -m 1410 hashes.txt /usr/share/wordlists/rockyou.txt --show
```

This will crack the password. SSH into the box using these credentials to gain the user flag.

## Privilege Escalation

So when we do `sudo -l` we get a python backup script that's running. The `tar` module in python has a vulnerability described in CVE-2025-4517. 

When I was hunting for PoCs I found [this exploit](https://github.com/AzureADTrent/CVE-2025-4517-POC-HTB-WingData/blob/main/CVE-2025-4517-POC.py). This ironically is the privilege escalation for this box 😭. So use this to gain root :) and spawn a shell.

However, since we are tryna learn here, I will tell you how this PoC exploits the CVE. For those too lazy to read, what this is essentially doing is modifying the sudoers file to allow our user to run all commands as root. So on simply doing `sudo su` we spawn a root shell.

1. The script creates VERY LONG directory names (247 chars long) and nests them 16 layers deep.
2. It creates a symlink pointing to its parent (../) repeated 16 times.
3. Now it creates an *escape hatch* pointing to /etc, `escape -> (symlink chain)/../../../../../../../etc`. This points to etc when resolved.
4. It creates a hardlink to sudoers `sudoers_link (hardlink) → escape/sudoers`. Since we already have escape linking to /etc `escape → /etc` we now get the entire thing resolving to `/etc/sudoers`.
5. Now we write the actual content into sudoers, giving us the ability to simply gain a root shell.

Yeah, this is a little mental sounding, so try and exploit this manually once, you might have better luck understanding this.

Happy Hacking :)
