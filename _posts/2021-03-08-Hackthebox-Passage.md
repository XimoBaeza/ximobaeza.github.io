---
layout: single
title: Passage - Hack The Box
excerpt: "Resolución de la máquina *Passage* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de un exploit para el gestor de noticias *CuteNews* para obtener la intrusión inicial. Después encontraremos el hash del usuario paul y lo crackearemos para obtener su contraseña y migrar a su centa de usuario. Veremos como éste usuario tiene la clave privada ssh del usuario navdav en su directorio home, la utilizaremos y migraremos a su cuenta de usuario, y finalmente nos aprovecharemos de una vulnerabilidad que nos permite crear ficheros con permisos de root para poner nuestra clave pública de ssh en las claves autorizadas del usuario root para conectarnos por ssh sin contraseña con la cuenta de root."
date: 2021-03-08
classes: wide
header:
  teaser: /assets/images/Passage-Hackthebox/passage.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Web Exploiting
  - Privilege Escalation
  - Pentesting
  - Linux
---

![](/assets/images/Passage-Hackthebox/passage.png)

Resolución de la máquina *Passage* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de un exploit para el gestor de noticias *CuteNews* para obtener la intrusión inicial. Después encontraremos el hash del usuario paul y lo crackearemos para obtener su contraseña y migrar a su centa de usuario. Veremos como éste usuario tiene la clave privada ssh del usuario navdav en su directorio home, la utilizaremos y migraremos a su cuenta de usuario, y finalmente nos aprovecharemos de una vulnerabilidad que nos permite crear ficheros con permisos de root para poner nuestra clave pública de ssh en las claves autorizadas del usuario root para conectarnos por ssh sin contraseña con la cuenta de root.

## Escaneo de puertos

![](/assets/images/Passage-Hackthebox/nmap.png)

Puertos 22 y 80 abiertos.

Empiezo mirando la web en el puerto 80.

![](/assets/images/Passage-Hackthebox/web.png)

No hay robots.txt y en el código fuente tampoco hay nada interesante.

Decido fuzzear con ffuf pero me doy cuenta de que a los pocos segundos de empezar a buscar directorios se corta el acceso a la web, como si me hubieran baneado la ip.

## Intrusión inicial

En el pie de página veo “Powered by CuteNews” así que pruebo con searchsploit a ver si encuentra algo relacionado.

Y bingo! Encuentro ésto CuteNews 2.1.2 - Remote Code Execution

Lo descargo con searchsploit -m php/webapps/48800.py, lo ejecuto y me da diréctamente ejecución de comandos.

![](/assets/images/Passage-Hackthebox/cutenews.png)


