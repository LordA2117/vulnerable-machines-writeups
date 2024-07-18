# chromatica- A HackMyVM Machine

MACHINE IP: 192.168.1.11

Nmap scan:
        - Port 22
        - Port 80
        - Port 5353

On visiting /robots.txt in the website, we find a hidden page, /dev-portal/

We find that directly visiting this page will return forbidden, but on a re analysis of /robots.txt we can see that the allowed user agent is dev.

So let's try accessing this page using curl and setting the user agent to dev
```bash
curl http://192.168.1.11/dev-portal/ -A dev
```

This returns some html and a status code of 200. Therefore, we will edit our user agent in firefox to access the webpage.

Now this page contains a search query and the search url is vulnerable to sql injection (obtained via testing on sqlmap).
```bash
sqlmap -u http://192.168.1.11/dev-portal/search.php?city=random --user-agent=dev
```
This reveals that it is vulnerable to sql injection.

Now to get the database tables and passwords use this [link](https://book.hacktricks.xyz/pentesting-web/sql-injection/sqlmap)

We get the usernames and passwords as follows:
```
Database: Chromatica                                                                                                                                        
Table: users
[5 entries]
+----+-----------------------------------------------+-----------+-----------------------------+
| id | password                                      | username  | description                 |
+----+-----------------------------------------------+-----------+-----------------------------+
| 1  | 8d06f5ae0a469178b28bbd34d1da6ef3              | admin     | admin                       |
| 2  | 1ea6762d9b86b5676052d1ebd5f649d7              | dev       | developer account for taz   |
| 3  | 3dd0f70a06e2900693fc4b684484ac85 (keeptrying) | user      | user account for testing    |
| 4  | f220c85e3ff19d043def2578888fb4e5              | dev-selim | developer account for selim |
| 5  | aaf7fb4d4bffb8c8002978a9c9c6ddc9              | intern    | intern                      |
+----+-----------------------------------------------+-----------+-----------------------------+
```

Now I set the options to crack the hashes as well which is why the password to the `user` username is shown.

So we have gained a pair of credentials as `user:keeptrying`.
The other accounts may come in handy so we will keep an eye on them.

So I was unable to gain access using the obtained credentials. So let's try using [crackstation](https://crackstation.net/) to crack the passwords.

The credentials we have are as follows:
```
user:keeptrying
intern:inttern00
dev:flaghere
admin:adm!n
```

Now it's time to see which one works.

Trying *dev:flaghere*
```bash
dev@192.168.1.11's password: 
GREETINGS,
THIS ACCOUNT IS NOT A LOGIN ACCOUNT
IF YOU WANNA DO SOME MAINTENANCE ON THIS ACCOUNT YOU HAVE TO
EITHER CONTACT YOUR ADMIN
OR THINK OUTSIDE THE BOX
BE LAZY AND CONTACT YOUR ADMIN
OR MAYBE YOU SHOULD USE YOUR HEAD MORE heh,,
REGARDS

brightctf{ALM0ST_TH3R3_34897ffdf69}
Connection to 192.168.1.11 closed.
```

So there probably is a way to get in using these credentials.

We also dont have a password on **dev-selim** so I am under the assumption that might be the login. But the question arises, for exactly what are these usernames intended? Maybe its on the webpage??

Ok so with a little bit of fuzzing I found that there is a `dev-portal/login.php`.
Let's test these credentials on them.

I assumed that this login page leads nowhere since any and every combination of data gave an Error Code 500, but maybe some sort of code injection might be possible.

I was right in the hunch that the login page leads nowhere.

Actually when you ssh using ***dev:flaghere*** credentials, and keep the window size of your terminal to a minimum, you will find a **more** option.

Just type the command
!/bin/bash

And you now have access to the user flag.

Now by running linpeas.sh we find a cron job called end_of_day.sh. This file is modifiable so I modify this file in order to give me access to the device as *analyst*. This is done by simply adding a *reverse shell* script to the file.

This gave us access to analyst.

Running linpeas.sh on analyst, I found that it can run **/usr/bin/nmap** as **sudo**. 

Referring to **[gtfobins](https://gtfobins.github.io/gtfobins/nmap/)** I found the exploit that led to privilege escalation and got the root flag.
                                                                                                     
