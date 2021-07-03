---
layout: single
title: Spectra - Hack The Box
excerpt: "Resolución de la máquina *Spectra* de Hack The Box, una máquina ChromeOS de dificultad fácil según la plataforma en la cual nos aprovechamos de un error en el entorno de pruebas para descubrir que hay un *Directory Listing* en el cual se expone un fichero de backup del wp-config.php de Wordpress para obtener el acceso inicial, para posteriormente encontrar la contraseña del usuario *katie* y acceder por ssh a la máquina, y finalmente aprovecharnos de poder ejecutar con privilegios de sudo sin proporcionar contraseña el comando initctl para modificar un servicio y obtener el acceso de root."
date: 2021-07-03
classes: wide
header:
  teaser: /assets/images/Spectra-Hackthebox/spectra.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Web Exploiting
  - Privilege Escalation
  - Pentesting
  - Linux
---

![](/assets/images/Spectra-Hackthebox/spectra.png)

Resolución de la máquina *Spectra* de Hack The Box, una máquina ChromeOS de dificultad fácil según la plataforma en la cual nos aprovechamos de un error en el entorno de pruebas para descubrir que hay un *Directory Listing* en el cual se expone un fichero de backup del wp-config.php de Wordpress para obtener el acceso inicial, para posteriormente encontrar la contraseña del usuario *katie* y acceder por ssh a la máquina, y finalmente aprovecharnos de poder ejecutar con privilegios de sudo sin proporcionar contraseña el comando initctl para modificar un servicio y obtener el acceso de root.

## Escaneo con nmap

Empezamos escaneando todos los puertos

![](/assets/images/Spectra-Hackthebox/nmap-full.png)

Vemos los puertos 22 (SSH), 80 (HTTP) y 3306 (MYSQL) abiertos.

Escaneo específico sobre los puertos encontrados usando scripts básicos de enumeración y detección de versiones.

![](/assets/images/Spectra-Hackthebox/targeted.png)

## Intrusión inicial

Accedo a la web y veo que los links te redirigen a spectra.htb.

![](/assets/images/Spectra-Hackthebox/web.png)

Añado ```10.10.10.229 spectra.htb``` al fichero hosts para que me resuelva el nombre y accedo.

En el primer link hay un wordpress

![](/assets/images/Spectra-Hackthebox/wordpress.png)

En el segundo hay una página en la que aparece un error

![](/assets/images/Spectra-Hackthebox/testing.png)

Viendo que en la URL pone la palabra testing, se intuye que puede ser un entorno de pruebas. Le quito el index.php para acceder al directorio testing y veo que tiene capacidad de Directory Listing, y se ven diréctamente los archivos.

![](/assets/images/Spectra-Hackthebox/testing2.png)

Me llama la atención el fichero wp-config.save. Hago click en el fichero y nose ve nada, pero al acceder al código fuente de la página se puede ver el contenido del fichero, y unas credenciales.

![](/assets/images/Spectra-Hackthebox/creds.png)

Se ve claramente que la página wordpress tiene en la URL spectra.htb/main y la otra spectra.htb/testing, con lo cual entiendo que el segundo es un entorno de pruebas del primero. Así que intento acceder con el usuario admin y la contraseña devteam01 a la página principal pero me dice usuario erróneo.

Viendo que el creador del único post que hay en la página es el usuario Administrator pruebo con ese usuario y la misma contraseña y accedo.

Pruebo modificando la plantilla 404.php añadiendo ```<?php system($_REQUEST['cmd']); ?>``` para ejecutar comandos a partir de la variable cmd pero veo que no me deja guardar los cambios.

Llegados a este punto lo intento con msfconsole, y obtengo una sesión de meterpreter.

![](/assets/images/Spectra-Hackthebox/shell.png)

## Escalada al usuario katie

Empiezo a enumerar el sistema y veo en /opt un archivo llamado autologin.conf.orig, en el que pone que lee una contraseña de /etc/autologin. Dentro del directorio hay un fichero passwd que contiene una contraseña.

![](/assets/images/Spectra-Hackthebox/passwd.png)

