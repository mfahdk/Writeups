# HackTheBox: Outbound – Writeup
Outbound is an easy Linux machine where RCE in Roundcube (CVE-2025-49113) provides initial access, a DES key is used to decrypt credentials from the database, and sudo access to a vulnerable below binary allows privilege escalation to root.

The nmap scan shows port 22(ssh) and port 80(nginx web server) as up.

## Enumeration

### Nmap scan

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-07-15 22:44 +04
Nmap scan report for 10.10.11.77
Host is up (0.30s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://mail.outbound.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.58 seconds
`

### Web server

Enumerating the web server, we see it is actually a mail server, specifically roundcube webmail v1.6.10. 

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/1.png)

## Initial Foothold

Searching for any vulnerabilities in this version of round cube, we come across CVE-2025-49113.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/2.png)

This vulnerability allows the attacker to conduct a RCE on the webserver.

### Exploitation

[The exploit is found here](https://github.com/hakaioffsec/CVE-2025-49113-exploit/blob/main/README.md)

On the attacker machine, the exploit is ran and the netcat listener is opened.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/3.png)

### Linpeas enumeration

Upon running linpeas, some sensitive information regarding a SQL DB and a DES encryption key is revealed. 

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/4.png)

Opening the DB usernames, and session hashes are found.

### Database enumeration

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/5.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/6.png)

The session data was encoded at first, but can be decoded using the command; `SELECT FROM_BASE64(vars) AS decoded_vars FROM session;`

Which gives us the password for the hashed password of the 'Jacob' user.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/7.png)

After being stuck on this part for quite some time, trying to crack the hash or look for any other hashes or keys, I remembered coming across a DES key, which likely is how the hash is being encrypted. After looking into deeper i came across [this](https://github.com/roundcube/roundcubemail/blob/master/program/lib/Roundcube/rcube.php##L938).

### DES Key decryption

The decryption function used in the roundcube first decodes the base64 then extracts the first 8 bytes, to be the IV. Rest of it being the ciphertext, which can be decrypted using the DES Key (which we already have) and the IV. This can be done through cyberchef.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/8.png)

Using the decrypted password, we are able to log into Jacob and obtain the user flag.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/9.png)

## Privilege Escalation

Checking for any sudo perms Jacob has, we see that he is able to run `/usr/bin/below`, which open researching is [vulnerable](https://github.com/BridgerAlderson/CVE-2025-27591-PoC)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/10.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Outbound/Screenshots/11.png)


