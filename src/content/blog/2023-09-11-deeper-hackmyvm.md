---
title: "Deeper Write-Up - HackMyVM"
description: "Deeper Write-Up - HackMyVM"
pubDatetime: 2023-09-11
tags: ["fácil", "hackmyvm", "cracking"]
---

# Reconocimiento

Empezamos con un escaneo de puertos con `nmap`.

```bash
❯ sudo nmap -sSCV -p- --open -Pn -n 192.168.18.152
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-12 00:35 CEST
Nmap scan report for 192.168.18.152
Host is up (0.00013s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2 (protocol 2.0)
| ssh-hostkey:
|   256 37:d1:6f:b5:a4:96:e8:78:18:c7:77:d0:3e:20:4e:55 (ECDSA)
|_  256 cf:5d:90:f3:37:3f:a4:e2:ba:d5:d7:25:c6:4a:a0:61 (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Deeper
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 08:00:27:F9:93:35 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.90 seconds
```

Tenemos SSH (22) y HTTP (80). Empezemos con el reconocimieto web.

## Reconocimiento web

Nos encontramos con la siguiente página.

![](@assets/HackMyVM/deeper/index.png)

Viendo el código fuente nos chivan una pista.

![](@assets/HackMyVM/deeper/deeper.png)

Vamos a `http://192.168.18.152/deeper/`, donde de nuevo nos chivan otra pista.

![](@assets/HackMyVM/deeper/evendeeper.png)

Vamos a `http://192.168.18.152/deeper/evendeeper` y en el código fuente de esta página encontramos dos cadenas de texto.

![](@assets/HackMyVM/deeper/string1.png)

![](@assets/HackMyVM/deeper/string2.png)

Introduciéndolas en [cyberchef](https://cyberchef.org/) con el modo **Magic** decodeamos ambas cadenas.

![](@assets/HackMyVM/deeper/cyberchef1.png)

![](@assets/HackMyVM/deeper/cyberchef2.png)

Parece que tenemos un usuario y su contraseña así que intento acceder por SSH (alice:IwillGoDeeper).

![](@assets/HackMyVM/deeper/user.png)

# Movimiento lateral

Primeramente estamos dentro como `alice`. En su directorio personal hay un archivo oculto `.bob.txt`.

![](@assets/HackMyVM/deeper/bob.png)

De nuevo lo introducimos en cyberchef.

![](@assets/HackMyVM/deeper/cyberchef3.png)

Existe un usuario bob por lo que me logueo con la contraseña obtenida.

# Escalada de privilegios

Una vez como bob tiene un archivo comprimido **root.zip** en su directorio personal.

![](@assets/HackMyVM/deeper/user2.png)

## Crackeando el zip

Me traigo el archivo a mi máquina local para descomprimirlo.

![](@assets/HackMyVM/deeper/transfer.png)

Pero tiene contraseña así que con `zip2john` lo convierto a un formato crackeable con la herramienta `john`.

```bash
❯ unzip root.zip
Archive:  root.zip
[root.zip] root.txt password: %
❯ zip2john root.zip > hash
ver 1.0 efh 5455 efh 7875 root.zip/root.txt PKZIP Encr: 2b chk, TS_chk, cmplen=33, decmplen=21, crc=2D649941

❯ john -w:/usr/share/seclists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
No password hashes left to crack (see FAQ)

❯ john --show hash
root.zip/root.txt:bob:root.txt:root.zip::root.zip

1 password hash cracked, 0 left
```

La contraseña es **bob** así que descomprimimos el archivo.

```bash
❯ unzip root.zip
Archive:  root.zip
[root.zip] root.txt password:
 extracting: root.txt

❯ cat root.txt
root:IhateMyPassword
```

Parecen credenciales así que pruebo por la sesión de teníamos de antes.

![](@assets/HackMyVM/deeper/root.png)

Tenemos shell como root y la última flag.
