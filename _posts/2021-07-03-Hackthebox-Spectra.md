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

