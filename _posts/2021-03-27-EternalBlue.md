---
layout: single
title: Explotando EternalBlue sin Metasploit
excerpt: "En este artículo veremos como atacar un Windows 7 Profesional de 64 bits y conseguir acceso al sistema con los máximos privilegios utilizando la vulnerabilidad eternalblue sin usar metasploit, de forma manual."
date: 2021-03-27
classes: wide
header:
  teaser: /assets/images/EternalBlue/EternalRocks2.png
  teaser_home_page: true
categories:
  - Windows
tags:
  - Windows
  - Privilege Escalation
  - Pentesting
---

![](/assets/images/EternalBlue/EternalRocks.png)

En este artículo veremos como atacar un Windows 7 Profesional de 64 bits y conseguir acceso al sistema con los máximos privilegios utilizando la vulnerabilidad eternalblue sin usar metasploit, de forma manual.

EternalBlue aprovecha una vulnerabilidad en la implementación del protocolo Server Message Block (SMB) de Microsoft. Esta vulnerabilidad, denotada como CVE-2017-0144 en el catálogo Common Vulnerabilities and Exposures (CVE), se debe a que la versión 1 del servidor SMB (SMBv1) acepta en varias versiones de Microsoft Windows paquetes específicos de atacantes remotos, permitiéndoles ejecutar código en el ordenador en cuestión.

Usaré una máquina virtual con Windows 7 x64 que será la víctima y otra con
Linux para la parte del atacante.

![](/assets/images/EternalBlue/windows7.png)

Vemos que la ip de nuestro Windows 7 es 192.168.160.132

Lo primero será descargarnos un repositorio de github que contiene los exploits
que necesitaremos.

```bash
git clone https://github.com/3ndG4me/AutoBlue-MS17-010.git
```

Nos metemos dentro del directorio, y ejecutamos el script que comprueba si la
máquina es vulnerable.

![](/assets/images/EternalBlue/checker.png)

Y nos dice que no está parcheado, con lo cual es vulnerable al eternalblue.

Según el creador de los exploits los siguientes sistemas son vulnerables:

Tested on:
 - Windows 2016 x64
 - Windows 10 Pro Build 10240 x64
 - Windows 2012 R2 x64
 - Windows 8.1 x64
 - Windows 2008 R2 SP1 x64
 - Windows 7 SP1 x64
 - Windows 2008 SP1 x64
 - Windows 2003 R2 SP2 x64
 - Windows XP SP2 x64
 - Windows 8.1 x86
 - Windows 7 SP1 x86
 - Windows 2008 SP1 x86
 - Windows 2003 SP2 x86
 - Windows XP SP3 x86
 - Windows 2000 SP4 x86

A continuación nos metemos en el directorio shellcode y ejecutamos
./shell_prep.sh

![](/assets/images/EternalBlue/shell_prep.png)

En LHOST escribiremos nuestra ip de atacante, en mi caso 192.168.160.129, en
LPORT x64 el puerto para recibir la conexión usando el archivo x64, en LPORT
x86 el puerto para recibir la conecxión en x86 que tiene que ser distinto del
que pongamos en x64, en tipo de shell elegimos la 1 que no utiliza metasploit
y en la última opción stageless, opción 1.

Esto nos generará un archivo para x86 (sc_x86.bin), otro para x64 (sc_x64.bin)
y otro que probará el exploit con las dos arquitecturas (sc_all.bin).

Solo nos queda ponernos a la escucha en el puerto 9001 que es el que hemos
elegido para x64, y al ejecutar el exploit recibiremos una conexión de cmd de
la máquina víctima con los máximos privilegios (system).

Ejecutamos el exploit de la siguiente forma: python eternalblue_exploit7.py IP
Archivo.bin

![](/assets/images/EternalBlue/exploit.png)

Si ejecutamos el comando whoami vemos que efectivamente tenemos privilegios de
system diréctamente.

![](/assets/images/EternalBlue/system.png

)
