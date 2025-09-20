# HackTheBox: Fluffy – Writeup
Fluffy is an easy Windows machine, vulnerable to CVE-2025-24071 allowing us to obtain the NTLM hash of a user listening on the system, after which a vulnerable ESC16 certificate is found, enabling the attacker to gain the administrators hash.

**NOTE:** The `KRB_AP_ERR_SKEW` (Clock Skew Too Great) error in Kerberos occurs when your system time differs significantly from the target machine. To fix this, you can synchronize your clock with the target using tools like `rdate` or adjust your system time manually. This ensures Kerberos tickets are accepted, preventing authentication errors during operations like Kerberoasting. For a detailed guide, see [this article](https://medium.com/@danieldantebarnes/fixing-the-kerberos-sessionerror-krb-ap-err-skew-clock-skew-too-great-issue-while-kerberoasting-b60b0fe20069).


## Enumeration

### Nmap scan

The nmap scan shows that the machine hosts services including; Kerberos, ldap, winrm, and smb

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-07-10 23:33 +04
Nmap scan report for 10.10.11.69
Host is up (0.22s latency).
Not shown: 989 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-07-11 02:34:07Z)
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-07-11T02:35:31+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
|_ssl-date: 2025-07-11T02:35:32+00:00; +7h00m00s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
|_ssl-date: 2025-07-11T02:35:31+00:00; +7h00m00s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-07-11T02:35:32+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s
| smb2-time: 
|   date: 2025-07-11T02:34:55
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 105.39 seconds
```

### File shared enumeration

Using the provided confidential provided, I am able look into the files shared via SMB.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/1.png)

Which includes a pdf file, containing the number of vulnerabilities in the machine.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/2.png)

## Initial foothold

Amongst which upon researching, CVE-2025-24071 has an exploit, that enables us to obtain the NTLM hash of the current user on the machine. The exploit can be found [here](https://github.com/DeshanFer94/CVE-2025-24071-POC-NTLMHashDisclosure-/tree/main/POC).

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/3.png)

The NTLM responder is opened on the machine using the command `sudo responder -i tun0`.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/5.png)

And then the hash is cracked using john.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/6.png)

The bloodhound data is obtained using bloodhound-python, using the user 'p.agila' using the command `bloodhound-python -u p.agila -p 'prometheusx-303' -ns 10.10.11.69 -d fluffy.htb -c all
`.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/7.png)

### Shadow credential attack

Upon adding the obtained AD data to bloodhound, we see that agila is a member of service account managers, and has DACL GenericAll rights over service accounts, which is responsible for WinRM and CA. Our aim is to enable us to WinRM using agila's account, which is currently disabled. In order to do so, follow the commands instructed by bloodhound (on the right pane). 

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/8.png)

`Note: Adding the user to service account won't be permanent due to GPO' we can verify if agila is now part of service accounts.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/9.png)

After Agila is added to Service Accounts, it gains GenericWrite over the WINRM_SVC account. This allows us to use Certipy to perform a Shadow Credentials attack, appending our own public key to the msDS-KeyCredentialLink attribute. As a result, we can authenticate as the WINRM_SVC user without knowing its password.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/10.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/11.png)

Allowing us to winrm into the user, using the hash and obtain the user flag.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/12.png)

## Privilege escalation

We then perform the same shadow credentials attack on the CA service account, to look for any misconfigured AD CS.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/13.png)

After which we use certipy-ad to look for any vulnerable AD certificates, and come across ESC16.

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/14.png)

The exploit for ESC16 can be found [here](https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation#esc16-security-extension-disabled-on-ca-globally)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/15.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/16.png)

![Image Alt](https://github.com/mfahdk/Writeups/blob/main/HackTheBox/Fluffy/Screenshots/17.png)

After following the steps, we're able to gain access to the administrator and obtain the root flag.
