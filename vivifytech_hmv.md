# vivifytech.hvm

Machine IP: found by netdiscover

/wordpress directory in port 80

Used writeup for this bit:
Reading the story of the company, we can find a bunch of users mentioned. Make that into a wordlist.

Another one us in /wordpress/wp-includes/secrets.txt

This has the passwords.

Use this to bruteforce ssh to get

sarah:bohicon

-- My own from now --
On using linpeas,
we find that db name is wordpress and password is password. Use this to log in to mysql

After that, go to the wp_users table to obtain this bit:
username:password = sancelisso:$P$BPhGmUp9fmz6VHYL1FOPr33qtX.yyf1

This can be cracked to login to the admin page

Another way to do this is to cd to the .private folder
we find this
gbodja:4Tch055ouy370N

Let's log in using ssh

Here you can run git as sudo, found by using sudo -l
So we use GTFObins to bypass and get privesc.

