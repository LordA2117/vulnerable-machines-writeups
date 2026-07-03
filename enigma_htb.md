# Enigma - HackTheBox

Difficulty: Easy

## Enumeration

### Nmap Scan

| Port | Service |
|------|---------|
| 22   | SSH |
| 80   | HTTP |
| 110  | Dovecot pop3 |
| 111  | rpcbind |
| 143  | Dovecot imap |
| 993  | Dovecot imap |
| 995  | Dovecot pop3 |
| 2049 | nfs |


Other ports were irrelevant to this box


### Port 80

- Port 80 has hostname enigma.htb, root subdomain contains nothing of value.
- Enumerating subdomains using the seclists `subdomains-top1million-110000.txt` wordlist gives the subdomain `mail001`.
- This contains roundcube webmail, protected by a login.

### Port 2049

Port 2049 contains an nfs share which we can enumerate as follows

```bash
showmount -e enigma.htb

Export list for enigma.htb:
/srv/nfs/onboarding *
```

- We can mount this share using the following command to extract its contents:

```bash
sudo mount -t nfs enigma.htb:/srv/nfs/onboarding nfs_mount
```

- This gives us a pdf document which contains the credentials for the roundcube login.


### Roundcube webmail

- On logging into roundcube as the `kevin` user, we can see a mail sent from `sarah@enigma.htb`.
- We will try stuffing the password for kevin to the sarah user, which surprisingly works, to give us access to sarah's webmail.
- Sarah's mail gives us a set of credentials and a new subdomain `support_001.enigma.htb`.


### OpenSTAManager

- We will use the credentials we obtained from `sarah` to login to openstamanager.
- This is of version 2.9.8 which is vulnerable to RCE as described [here](https://github.com/lukasz-rybak/CVE-2025-69212).
- This gives us a shell as `www-data`.


## Pivoting

- On entering the instance, we can find a `config.inc.php` file which contains credentials to the database. We'll use this to extract the users from the `zz_users` table in the openstamanager db.
- Cracking these passwords gives us the credentials to the `haris` user, giving us the **user flag**.


## Privilege Escalation

- On running `ss -nltp` we can see a service running on **port 1337**.
- This is OliveTin 3000.10.0, which is vulnerable to [CVE-2026-27626](https://github.com/advisories/GHSA-49gm-hh7w-wfvf). But this exact PoC is not usable on our instance.
- Reading the OliveTin config file located at `/etc/OliveTin`, we see that the database backup action has shell permissions. This allows us to inject commands into the server.
- So we'll enter the Database backup action and in the `db_pass` field we'll enter this payload: `x';cat /root/root.txt;#` and run the command.
- This provides RCE and the **root flag**.


Happy Hacking :)
