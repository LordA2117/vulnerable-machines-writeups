# RoosterRun - A HackMyVM Machine
## Enumeration
MACHINE IP: [MACHINE IP]
<br/>
Port 80:<br/>
	- Uses CMS Made Simple version 2.2.9.1<br/>
	- It's also running apache httpd.<br/>
	- msfconsole has exploits for this version of CMSMS but none of them work.
<br/>

ON RUNNING THE EXPLOITDB tool `https://www.exploit-db.com/exploits/46635` after debugging:<br/>
---BEGIN OUTPUT---
```
[+] Salt for password found: 1a0112229fbd699d
[+] Username found: admin
[+] Email found: admin@localhost.com
[+] Password found: 4f943036486b9ad48890b2efbf7735a8
```
---END OUTPUT---

But, directly inputting this password doesn't appear to work, so I am assuming that the password is encrypted (it is actually encrypted with a salt and md5). 

But when using this exploit `https://srcincite.io/pocs/cve-2021-26120.py.txt`, we get this output
Command: 
```bash
python ssti_exploit.py [MACHINE IP] / "whoami"
```
---BEGIN OUTPUT---
```
(+) targeting http://192.168.1.102/
(+) sql injection working!
(+) leaking the username...
(+) username: admin
(+) resetting the admin's password stage 1
(+) leaking the pwreset token...
(+) pwreset: c1e98093f7bc11fa29534207132e97e3
(+) done, resetting the admin's password stage 2
(+) logging in...
(+) leaking simplex template...
(+) injecting payload and executing cmd...

www-data
```
---END OUTPUT---

You can see that it has output ***www-data*** as the current user. However, this command execution is only arbitrary.

This cannot achieve a reverse shell however. So instead, I went through the code and found out, that this exploit *reset the admin username and password* to ***admin:admin***.

## Initial Access
This way, we can login to the admin page, and try to get RCE from there.

Now, in the admin page, there is a file manager, which allows for file uploads. However, it blocks all php file uploads, so instead I changed my .php file to a .phar file. This allows the upload, and enables rce as www-data.

On looking at config.php in the var/www/html folder, we see this snippet:<br/>
---BEGIN OUTPUT---
```php
$config['dbms'] = 'mysqli';
$config['db_hostname'] = 'localhost';
$config['db_username'] = 'admin';
$config['db_password'] = 'j42W9kDq9dN9hK';
$config['db_name'] = 'cmsms_db';
$config['db_prefix'] = 'cms_';
$config['timezone'] = 'Europe/Berlin'
```
---END OUTPUT---

We'll use this to connect to the mysql database with the command:
```bash
mysql -u admin -h localhost -p
```
And then enter the password.

Here, we can use standard mysql commands in a MariaDB server to sift through the database, but this doesn't yield any results.

On enumrating further, we find that a user ***matthieu*** exists, which contains the user flag.

However, I have not managed to gain access as matthieu yet.
