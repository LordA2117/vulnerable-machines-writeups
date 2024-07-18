# liceo.hvm
Hackmyvm liceo reference

MACHINE IP: [Machine IP]

Interesting stuff:
port 22: ssh (standard shit)
port 80: http (standard shit)
port 21: ftp (vsftpd 3.0.5, allows anonymous login)

Nothing interesting on the website so far. Let's try the anonymous login with ftp on port 21

Found a note on the anonymous ftp login, which might be useful later.

In the note, a name is mentioned called Matias. Could this be a username of some sort?
It also mentions that it left some incomplete work on the web. What does this mean? Could this be the attack surface?

It turns out there is an upload.php file which for some unknown reason isn't there in any wordlist that I possess 💀.

Anyways, it has some kind of a blacklist which doesnt allow php file upload for .php.

So, I changed the file extension to .phar, and uploaded it. On visiting the /uploads page, (found by fuzzing) and clicking on the .phar file, I got a reverse shell.

Now, I uploaded linpeas.sh to the /tmp/privesc directory and running it shows me that I can run bash as sudo. However, I don't know the password. So let's try and find it.

That didnt work 😭.

 So we try running /usr/bin/bash in privileged mode:
command: /usr/bin/bash -p

This makes us root.
