# Publisher: A HackMyVM Machine

Machine IP: 192.168.29.102
HOST NAME: publisher.hmv

Open ports: port 22, port 80

Potential usernames: admin

Interesting Link: http://192.168.29.101/spip/spip.php?article1
Reflected values found, might be a sign of XSS.

There is stored HTML injection in the webpage. If the title of the comment is enclosed in the h1 tags, it will be stored and shown in the comments section.

Interesting Links:
        - http://192.168.29.101/spip/spip.php?page=backend
        - http://192.168.29.101/spip/spip.php?page=login&url=spip.php%3Fpage%3Drecherche%26amp%3Brecherche%3Dtitie&lang=en

Pages found by fuzzing page=FUZZ:
plan
sommaire
recherche
ical
login


Looking up CVE-2023-27372

we find an at https://github.com/0SPwn/CVE-2023-27372-PoC/blob/main/exploit.py

Now we get a shell, which doesnt allow network based commands. So we encode the shell in base64 and then run it as follows
echo "base64 encoded reverse shell" | base64 -d | bash

Which will allow us to move the reverse shell out of the exploit shell into an actual shell.

Now we can find an SSH private key for the user think. Create a file publisher_rsa on the attacker machine and use the command 
chmod 600 [private key file]

Now connect to the machine using the following command:
ssh -i [private key file] think@publisher.hmv


And we connect


On running linpeas.sh we find a file /opt/run_container.sh  

This file is not modifiable, so we harlink this file to a file in /tmp as follows
ln /opt/run_container.sh /tmp/run_container.sh

This file runs as root on startup, so we can add some reverse shell code to the file.

We start a netcat listener on our machine and restart the target machine. This gives us the root shell.
