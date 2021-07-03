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