---
layout: writeup
title: Extortion
child: true
description: Cyber Apocalypse 2021 CTF Web challenge
logo: /assets/img/writeups/extortion_info.png
show-avatar: false
permalink: /writeups/extortion.html
difficulty: 1/4?
creator: <a href="https://www.hackthebox.eu/home/users/profile/">????</a>
published: false
redirect_from: /walkthroughs/ctf-extortion.html
---

If we check out the main page we see  a few links.

![mainpage](/assets/img/writeups/extortion_index.png)

Checking out *airship* or *flaying saucer* we see a parameter is being used

![parameter](/assets/img/writeups/extortion_param.png)

This indicates a php include statement is likely being used to load content into the page. Let's give it a file we know likely doesn't exist and see if we get any error messages that might give us some information.

With the param *?f=hilbert.php* we don't notice anything on the page, but if we look at the page source we see

![failed to load](/assets/img/writeups/extortion_failed_to_open.png)

the value of the parameter is indeed being used in *include()*. Let's check if we can load arbitrary files by the standard test of trying to load the */etc/passwd* file. It looks like we are starting out in */var/www/html/files/* so we issue the following request

![/etc/passwd](/assets/img/writeups/extortion_passwd.png)

and viola. So now if we can get some malicious PHP code into a file on the system we'll have a pathway to the flag. A common technique here would be *log poisoning* however after a bit of fuzzing we don't turn up any log files we can access. If we check out the rest of the functionality of the web page we'll see that the *send* link takes us to a page where we can issue a POST request and are also given a session cookie. In PHP the default is to save session data in a file in the */tmp* directory with the filename of "sess_" and then whatever the PHPSESSID is.  Let's see if what we are entering is making it's way to the session file.

After issuing a POST request if we then try and load the session file

![session check](/assets/img/writeups/extortion_session_check.png)

we see that we have a successful inclusion of the data. So now let's see if we can issue some system commands

![system id](/assets/img/writeups/extortion_system.png)

jackpot, we have RCE. Now we simply need to find the location and name of the flag

![flag name](/assets/img/writeups/extortion_flag_name.png)

and then read it

![flag contents](/assets/img/writeups/extortion_flag_contents.png)

