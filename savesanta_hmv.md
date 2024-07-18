# SaveSanta - A hackmyvm machine

## Enumeration
- ***MACHINE_IP***: `[TARGET IP]`

- The machine contains a /santa page which allows for a login (found in /robots.txt on a nikto and nmap default scripts (-sC) scan).

- The request made by the page for logging in looks like this *http://[TARGET IP]/santa/?username=admin&password=admin*

- However, trying to open this url a second time doesn't appear to work. It instead returns an *Error Code 404*.

- On trying to navigate to other pages, we found that it still returned an error code 404, so we resorted to another nmap scan.

## Initial Access

- On running an nmap scan for the second time, after trying to login to the page, I found a port running an unknown service.

- Using the command
```bash
nc [TARGET IP] [TARGET PORT]
```

- I login to the machine and gain the user flag.

- I upgraded the shell and moved on to privesc.

## Privilege Escalation

***NOTE: THIS IS LIKELY NOT THE INTENDED METHOD OF PRIVILEGE ESCALATION***

- So, all I did was download this exploit, which is the latest *[linux nftables privesc exploit](https://github.com/Notselwyn/CVE-2024-1086/releases/download/v1.0.0/exploit)* and ran it on the target machine.

***NOTE: ON FURTHER TESTING OF THIS METHOD, WITHOUT GIVING THE VM MORE RAM, IT REGULARLY CRASHED THE MACHINE, IT ALSO CRASHED AFTER GIVING IT MORE RAM SO I CAN SAFELY SAY THAT THIS IS NOT THE INTENDED METHOD OF PRIVESC AS IT IS A ONE OFF FOR ME***

Voila! We got root.

