# wifinetictwo-reference

1. After nmap, found OpenPLC running on port 8080.
2. Login page has no real vulnerabilities, but can be logged into using default credentials openplc:openplc

3. From here, we can run an authenticaated RCE attack, which this version of OpenPLC is vulnerable to.

After running this attack, you log in as root which reveals the user flag. (💀)

Now, to get the root flag, I don't even know wtf was going on so I just used this walkthrough 
https://mrbandwidth.medium.com/wifinetictwo-writeup-walkthrough-htb-hackthebox-remote-code-execution-33b501b69579

Thank him for his genius
