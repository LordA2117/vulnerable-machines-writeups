# Enumeration
Open ports: 22,80,3000 -> found by masscan

The hostname found is greenhorn, which I did by decoding the base64 cookie found during the nmap -sC -sV enumeration

Port 80 runs pluck 4.7.18 which is vulnerable to authenticated RCE via unauthenticated file upload
Port 3000 contains the repo for greenhorn where the password is found at data/settings/pass.php

# Exploitation
Use this password in this exploit https://github.com/Rai2en/CVE-2023-50564_Pluck-v4.7.18_PoC
Once you are in, you can pivot to the junior user using the password iloveyou1 (which is the password we cracked from the hash in pass.php)

# Privilege Escalation
Now there is a pdf file called 'Using OpenVas.pdf'
Download this file and use this command
```bash
pdfimages -png Using\ OpenVAS.pdf out.png
```
to isolate the pixelated part of the image.

Use this tool https://github.com/spipm/Depix to depixelate this.

Now the password will be something like
sidefromsidetheothersidesidefromsidetheotherside

Use this to login as root via ssh, or use the su command, your wish.
