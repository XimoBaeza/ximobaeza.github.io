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