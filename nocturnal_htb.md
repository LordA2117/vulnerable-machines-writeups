Portscan results: port 22 and port 80 open

Port 80 has hostname nocturnal.htb, so add it to /etc/hosts.

Now in the view.php file, we can find that usernames are unauthenticated. So we can view and download other people's files, let's now bruteforce this.

We get a user **amanda** and a **privacy.odt** file, download it. 

.odt files are just zip files so we'll unzip them by changing extension to .zip and doing `unzip zipname.zip -d dest_folder` and find creds by grepping for password, using `grep -r password`, obtained this:

`amanda:arHkG7HAI68X8s1J`

login to the website using this and go to /admin.php

So we'll create a backup, using the inbuilt functionality in /admin.php and download it, it gives us a db file as well, let's use it to get the users table
```
1|admin|d725aeba143f575736b07e045d8ceebb
2|amanda|df8b20aa0c935023f99ea58358fb63c4
4|tobias|55c82b1ccd55ab219b3b109b07d5061d
6|asd|7815696ecbf1c96e6894b779456d330e
```

So the user tobias yields a password, so let's crack it to get:

`tobias:slowmotionapocalypse`

Now, doing some enum with **linpeas.sh**, we see a service running on port 8080, let's access it. This service is ISPConfig.

We can login using admin:slowmotionapocalypse. Now the obvious step is to hunt for some exploits for this and I found this [exploit](https://github.com/bipbopbup/CVE-2023-46818-python-exploit/blob/main/exploit.py).

Using this I got root.

So what did we learn? Test all parameters of a website, and to properly enumerate services for privesc.

Enjoy :)
