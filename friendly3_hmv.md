# friendly3- A HackMyVM Machine 

MACHINE IP: [MACHINE IP]

NMAP SCAN:
        - Port 22
        - Port 80
        - Port 21

The website contains this message:
--- BEGIN ---
```
Hi, sysadmin
I want you to know that I've just uploaded the new files into the FTP Server.
See you,
juan.
```
--- END ---

So we already have 2 usernames here, one is **sysadmin** and one is **juan**.

Buy doing some good old fashioned brute force on the ftp server using this command
```bash
hydra 192.168.0.107 ftp -L usernames.txt -P /usr/share/wordlists/SecLists/Passwords/xato-net-10-million-passwords-10000.txt -V -F
```
We get this valid combination 
`juan:alexis`

Just out of a hunch I tried this pair in ssh, and it worked.
So we have gained initial access.

User flag:cb40b159c8086733d57280de3f97de30

Root flag: eb9748b67f25e6bd202e5fa25f534d51

So by using a tool called pspy64, which allows us to monitor any processes running on a machine without root access, we find that a program called check_for_install.sh keeps running periodically. Reading this file, it turns out that it executes a bash file **/tmp/a.bash**

So we create a bash file **/tmp/a.bash** containing our reverse shell payload.

Spawning a nc listener on our machine, we can gain a root shell.

Run pspy again and when the process runs we gain a reverse shell.
