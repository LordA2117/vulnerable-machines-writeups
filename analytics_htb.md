# analytics-reference
Analytics.htb writeup

## Enumeration
Using nmap we get port 22 and port 80 open

## Exploitation
Examining port 80, we can navigate to the login page, which will take us to data.analytical.htb (must be added in hosts).

This is powered by metabase, which can be exploited here [Metabase Exploit](https://github.com/m3m0o/metabase-pre-auth-rce-poc). This exploit is the best one I could find as all the others require BurpSuite Collaborator, which is premium.

After gaining reverse shell access, start a python http server on your machine, which will contain the **linpeas.sh** Privelege Escalation module. Use **wget *link to http server*** to download it. Run this on your target machine.

We'll see 2 environment variables, which provide the username as *metalytics* and it's password. Use this to log in to SSH.

## Privilege Escalation
In the same linpeas.sh PrivEsc scan, we can get the OS version. This reveals that the OS is vulnerable to a PE attack whose PoC is registered as [CVE-2023-2640](https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629/tree/main).
Of course other exploits are available so feel free to use whatever you want
