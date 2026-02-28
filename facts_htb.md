# Facts: HackTheBox

## Enumeration

Port 22,80,54321 open.

### Web Enumeration

Host name is `facts.htb`

We can't see anything of value in the normal pages so we can fuzz around, where we find an admin panel.

We can create an account and log in to the admin, which gives us a low privilege account.

Use [this exploit](https://github.com/whiteov3rflow/CVE-2025-2304-POC) to gain an admin account.

Once we gain the admin panel, we can see the site settings in `http://facts.htb/admin/settings/site`

Here, we can see an aws secret key and aws secret access key. Configure this in aws cli along with the other data on there. Here there's a bucket called internal which contains an ssh key. Download it.

Now we have to crack this key, so use the `ssh2john` tool.

```bash
ssh2john <private_key_file>
```

Copy this into a hash.txt file and crack the password using `JohnTheRipper` and `rockyou.txt` as the wordlist.

Once all this is done is where the ass-pulls start. So apparently this thing is vulnerable to CVE-2024-46987, which is a CVE for camaleon cms 2.8.2, but we're out here on 2.9.0, so I am not sure how this works, but it does I guess (lately a lot of htb machines have these ass-pulls on there for whatever reason, but no one says it, because it's like the souls games, just skill issue even if this shit is just genuinely ass. Maybe it is a skill issue on my part, but I feel like this is an ass-pull).

<!-- I spammed a writeup for this but I can't seem to find it anywhere -->

## Initial Foothold

So use the CVE to exploit the LFI and read /etc/passwd, which will tell us the user for which this private key works, which is `trivia`.

Use this and the private key to login.

Flag is at `/home/william`.

## Privilege Escalation

On running `sudo -l` we can see that `/usr/bin/facter` can run as root. So use [this exploit](https://gtfobins.org/gtfobins/facter/) mentioned here to get a root shell.

Basically I created an exploit.rb file in /tmp/privesc looking like this

```ruby
system("/bin/bash")
```

Then run facter using

```bash
sudo /usr/bin/facter --custom-dir=/tmp/privesc 
```

This grants a root shell.

Happy Hacking :)

<!-- This is a shit box my gng idk how it's sitting at a 4.5 rating. -->
