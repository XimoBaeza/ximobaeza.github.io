---
layout: single
title: Attacktive Directory
excerpt: "En este artículo veremos como resolver el reto de Try Hack Me *Attacktive Directory*, donde veremos como aprovecharnos de varias vulnerabilidades para explotar un controlador de dominio Windows Server. Nos aprovecharemos del script *kerbrute* para everiguar los usuarios del dominio. Después usaremos el script de impacket *GetNPUsers.py* para hacer un ataque de tipo *ASREPRoasting* y hacernos con un ticket de kerberos que posteriormente crackearemos para sacar la contraseña del usuario. Con esas credenciales descubriremos por SMB un fichero que contiene otra contraseña. Y descubriremos que ese usuario puede hacer un DCSync y hacer un volcado de los hashes del dominio. Ya con el hash del administrador usaremos *psexec.py* para conectarnos."
date: 2021-03-19
classes: wide
header:
  teaser: /assets/images/AttacktiveDirectory/AD.png
  teaser_home_page: true
categories:
  - Active Directory
tags:
  - Windows
  - Pentesting
  - Privilege Escalation
---

![](/assets/images/AttacktiveDirectory/AD.png)

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

