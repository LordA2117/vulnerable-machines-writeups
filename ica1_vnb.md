# ica1 - A VulnHub Machine

MACHINE IP: 192.168.1.17
HOST NAME: ica1.vnb

in the /conre/config/databases.yml file, passwords are exposed.
Login Credentials:
```
qdpmadmin:<?php echo urlencode('UcVQCMQk2STVeS6J') ; ?>
```

Instead, using the credentials to log in to mysql gives us these tables:
```
+------+---------+--------------------------+
| id   | user_id | password                 |
+------+---------+--------------------------+
|    1 |       2 | c3VSSkFkR3dMcDhkeTNyRg== |
|    2 |       4 | N1p3VjRxdGc0MmNtVVhHWA== |
|    3 |       1 | WDdNUWtQM1cyOWZld0hkQw== |
|    4 |       3 | REpjZVZ5OThXMjhZN3dMZw== |
|    5 |       5 | Y3FObkJXQ0J5UzJEdUpTeQ== |
+------+---------+--------------------------+
```
```
+------+---------------+--------+---------------------------+
| id   | department_id | name   | role                      |
+------+---------------+--------+---------------------------+
|    1 |             1 | Smith  | Cyber Security Specialist |
|    2 |             2 | Lucas  | Computer Engineer         |
|    3 |             1 | Travis | Intelligence Specialist   |
|    4 |             1 | Dexter | Cyber Security Analyst    |
|    5 |             2 | Meyer  | Genetic Engineer          |
+------+---------------+--------+---------------------------+
```

Let's use the credentials (which look like base64) and brute force our ssh Login
```
user_id:pass
2:suRJAdGwLp8dy3rF
4:7ZwV4qtg42cmUXGX
1:X7MQkP3W29fewHdC
3:DJceVy98W28Y7wLg
5:cqNnBWCByS2DuJSy
```


So in the end, the username Travis with the password for his id gives us ssh login
**User flag: ICA{Secret_Project}**

The system is vulnerable to the dirtypipe exploit found in [Dirtypipe Exploit](https://github.com/Arinerron/CVE-2022-0847-DirtyPipe-Exploit)

Running this will give an error but if you login to root with password aaron, after running the exploit, it works.

This is not the intended method of privesc however, so some other time try the intended method of privesc
