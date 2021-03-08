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

![](/assets/images/Academy-Hackthebox/passage.png)


