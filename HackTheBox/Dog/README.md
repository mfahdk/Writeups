#  HackTheBox: Dog – Writeup
Dog is an easy-level Linux machine where attackers exploit an exposed Git repository to access sensitive data and obtain admin credentials for BackdropCMS. With admin access, they upload a malicious PHP archive to achieve remote code execution. The same credentials are reused by the johncusack user, whose account can run the bee tool with sudo, ultimately allowing privilege escalation to root.

##  Enumeration
### Nmap Scan

```bash
nmap -sCV -T4 10.10.11.58
```
The nmap scan provides us some info about  the machine, it is running the SSH (Port 22) and Apache (Port 80) services, whereby the apache web server contains a ‘.git’ repository, likely hosting some sensitive information.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/nmap.png)

## Initial foothold

Tried using Git-dumper but it would crash, so had to look for other similar tools, found the ‘GitHack.py’ tool, and got the contents in the .git repo.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/git.png)

Looking through the extracted git, we find an email ‘tiffany’ and her password, looking deeper into it, it is the login for the web server.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/user.png)

### RCE on CMS

After logging in, we see that the web server is running on Backdrop CMS v1.27.1, which on further enumeration, is vulnerable to RCE.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/version2.png)
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/exploit.png)

Also, we are able to see all the users that have an account on the web server.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/users.png)

## Exploitation

Running the exploit, the RCE module is installed on the attacker machine, which is to be uploaded on the CMS modules.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/exploit1.png)
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/exploit2.png)

After obtaining the module, in is installed in ‘/installer/manual’.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/shellInstall.png)

After which we are able to execute commands on the shell module.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/workingExploit.png)

We then execute a reverse shell command, obtaining the ‘www-data’ user on the web server.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/shell.png)

## Privelege Escalation

Trying the privilege escalate to ‘johncusack’ we see that the user has the same password as the ‘tiffany’ user. After which, we obtain the user flag.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/priv.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/privs.png)

In order, to obtain the root flag, I see if the john user has any sudo permissions, which shows the user has perms to execute ‘bee’ via sudo. Using that, we spawn a bash shell, obtaining the root flag.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Dog/Screenshots/root.png)

