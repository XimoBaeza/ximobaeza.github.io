---
layout: single
title: Attacktive Directory
excerpt: "En este artículo veremos como resolver el reto de Try Hack Me *Attacktive Directory*, donde veremos como aprovecharnos de varias vulnerabilidades para explotar un controlador de dominio Windows Server. Nos aprovecharemos del script *kerbrute* para everiguar los usuarios del dominio. Después usaremos el script de impacket *GetNPUsers.py* para hacer un ataque de tipo *ASREPRoasting* y hacernos con un ticket de kerberos que posteriormente crackearemos para sacar la contraseña del usuario. Con esas credenciales descubriremos por SMB un fichero que contiene otra contraseña. Y descubriremos que ese usuario puede hacer un DCSync y hacer un volcado de los hashes del dominio. Ya con el hash del administrador usaremos *psexec.py* para conectarnos."
date: 2021-03-19
classes: wide
header:
  teaser: /assets/images/AttacktiveDirectory/active-directory.png
  teaser_home_page: true
categories:
  - Active Directory
tags:
  - Windows
  - Pentesting
  - Privilege Escalation
---

![](/assets/images/AttacktiveDirectory/active-directory.png)

En este artículo veremos como resolver el reto de Try Hack Me *Attacktive Directory*, donde veremos como aprovecharnos de varias vulnerabilidades para explotar un controlador de dominio Windows Server. Nos aprovecharemos del script *kerbrute* para everiguar los usuarios del dominio. Después usaremos el script de impacket *GetNPUsers.py* para hacer un ataque de tipo *ASREPRoasting* y hacernos con un ticket de kerberos que posteriormente crackearemos para sacar la contraseña del usuario. Con esas credenciales descubriremos por SMB un fichero que contiene otra contraseña. Y descubriremos que ese usuario puede hacer un DCSync y hacer un volcado de los hashes del dominio. Ya con el hash del administrador usaremos *psexec.py* para conectarnos.

## Escaneo de puertos con nmap

Empezamos escaneando todo el rango de puertos de la máquina y exportando el resultado al fichero allPorts

```
nmap -p- --open -T5 -v -n 10.10.225.26 -oG allPorts
```

![](/assets/images/AttacktiveDirectory/nmap.png)

Vemos puertos como kerberos, smb, winrm y rdp entre otros.

Escaneo con scripts de enumeración y detección de versiones a los puertos abiertos encontrados.

```
nmap -sC -sV -p53,80,88,135,139,389,445,464,593,636,3268,3269,3389,5985,9389,47001,49664,49666,49669,49672,49675,49676,49679,49685,49696,49816 10.10.225.26 -oN targeted
```

![](/assets/images/AttacktiveDirectory/targeted.png)


Lo primero que nos pregunta la plataforma es:

1. What tool will allow us to enumerate port 139/445?

	enum4linux

Utilizamos enum4linux para enumerar la máquina y el resultado es el siguiente:

```
 ========================== 
|    Target Information    |
 ========================== 
Target ........... 10.10.225.26
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ==================================================== 
|    Enumerating Workgroup/Domain on 10.10.225.26    |
 ==================================================== 
[E] Can't find workgroup/domain


 ============================================ 
|    Nbtstat Information for 10.10.225.26    |
 ============================================ 
Looking up status of 10.10.225.26
No reply from 10.10.225.26

 ===================================== 
|    Session Check on 10.10.225.26    |
 ===================================== 
[+] Server 10.10.225.26 allows sessions using username '', password ''
[+] Got domain/workgroup name: 

 =========================================== 
|    Getting domain SID for 10.10.225.26    |
 =========================================== 
Domain Name: THM-AD
Domain Sid: S-1-5-21-3591857110-2884097990-301047963
[+] Host is part of a domain (not a workgroup)

 ====================================== 
|    OS information on 10.10.225.26    |
 ====================================== 
[+] Got OS info for 10.10.225.26 from smbclient: 
[+] Got OS info for 10.10.225.26 from srvinfo:
Could not initialise srvsvc. Error was NT_STATUS_ACCESS_DENIED

 ============================= 
|    Users on 10.10.225.26    |
 ============================= 
[E] Couldn't find users using querydispinfo: NT_STATUS_ACCESS_DENIED

[E] Couldn't find users using enumdomusers: NT_STATUS_ACCESS_DENIED

 ========================================= 
|    Share Enumeration on 10.10.225.26    |
 ========================================= 

	Sharename       Type      Comment
	---------       ----      -------
SMB1 disabled -- no workgroup available

[+] Attempting to map shares on 10.10.225.26

 ==================================================== 
|    Password Policy Information for 10.10.225.26    |
 ==================================================== 
[E] Dependent program "polenum.py" not present.  Skipping this check.  Download polenum from http://labs.portcullis.co.uk/application/polenum/


 ============================== 
|    Groups on 10.10.225.26    |
 ============================== 

[+] Getting builtin groups:

[+] Getting builtin group memberships:

[+] Getting local groups:

[+] Getting local group memberships:

[+] Getting domain groups:

[+] Getting domain group memberships:

 ======================================================================= 
|    Users on 10.10.225.26 via RID cycling (RIDS: 500-550,1000-1050)    |
 ======================================================================= 
[I] Found new SID: S-1-5-21-3591857110-2884097990-301047963
[I] Found new SID: S-1-5-21-3532885019-1334016158-1514108833
[+] Enumerating users using SID S-1-5-21-3591857110-2884097990-301047963 and logon username '', password ''
S-1-5-21-3591857110-2884097990-301047963-500 THM-AD\Administrator (Local User)
S-1-5-21-3591857110-2884097990-301047963-501 THM-AD\Guest (Local User)
S-1-5-21-3591857110-2884097990-301047963-502 THM-AD\krbtgt (Local User)
S-1-5-21-3591857110-2884097990-301047963-512 THM-AD\Domain Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-513 THM-AD\Domain Users (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-514 THM-AD\Domain Guests (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-515 THM-AD\Domain Computers (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-516 THM-AD\Domain Controllers (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-517 THM-AD\Cert Publishers (Local Group)
S-1-5-21-3591857110-2884097990-301047963-518 THM-AD\Schema Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-519 THM-AD\Enterprise Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-520 THM-AD\Group Policy Creator Owners (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-521 THM-AD\Read-only Domain Controllers (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-522 THM-AD\Cloneable Domain Controllers (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-525 THM-AD\Protected Users (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-526 THM-AD\Key Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-527 THM-AD\Enterprise Key Admins (Domain Group)
S-1-5-21-3591857110-2884097990-301047963-1000 THM-AD\ATTACKTIVEDIREC$ (Local User)
[+] Enumerating users using SID S-1-5-21-3532885019-1334016158-1514108833 and logon username '', password ''
S-1-5-21-3532885019-1334016158-1514108833-500 ATTACKTIVEDIREC\Administrator (Local User)
S-1-5-21-3532885019-1334016158-1514108833-501 ATTACKTIVEDIREC\Guest (Local User)
S-1-5-21-3532885019-1334016158-1514108833-503 ATTACKTIVEDIREC\DefaultAccount (Local User)
S-1-5-21-3532885019-1334016158-1514108833-504 ATTACKTIVEDIREC\WDAGUtilityAccount (Local User)
S-1-5-21-3532885019-1334016158-1514108833-513 ATTACKTIVEDIREC\None (Domain Group)

 ============================================= 
|    Getting printer info for 10.10.225.26    |
 ============================================= 
Could not initialise spoolss. Error was NT_STATUS_ACCESS_DENIED


enum4linux complete on Fri Mar 19 20:36:50 2021

```

2. What is the NetBIOS-Domain Name of the machine?

	THM-AD

3. What invalid TLD do people commonly use for their Active Directory Domain?

	.local


