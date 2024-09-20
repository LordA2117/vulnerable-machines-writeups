# HackingToys - A HackMyVM machine

1. Enumeration: Standard 22 and 80 ports open
2. Initial Access: The message parameter in the search page is vulnerable to SSTI. Just use SSTImap to gain a reverse shell. Setup persistence using ssh
3. Pivot: The FastCGI service is running on port 9000. Use it to gain a shell to the dodi user.
4. Privilege enum: Running `sudo -l` gives us /usr/local/bin/rvm_rails.sh. This allows us to run this script. On reading this script we can see that it runs the rails binary, which is owned by the lidia user, aka rvm.
5. So we will replace this binary with a ruby revshell script. mv [our revshell] [target] && chmod 777 target. (Run this on the lidia user).
6. Run this as sudo from the dodi user and boom we got a shell.


Thats it. Fuck this im out
