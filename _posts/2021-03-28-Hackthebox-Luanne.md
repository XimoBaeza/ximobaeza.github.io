---
layout: single
title: Luanne - Hack The Box
excerpt: "Resolución de la máquina *Luanne* de Hack The Box, una máquina NetBSD de dificultad fácil según la plataforma, en la cual accedemos con credenciales por defecto al servicio *medusa supervisor process manager*, encontramos un script en lua vulneable a inyección de código a través del cual accederemos a la máquina. Posteriormente encontraremos la clave privada de ssh del usuario r.michaels y accederemos por ssh, para encontrar un fichero de backup encriptado que conseguiremos desencriptar y nos revelará la contraseña del usuario r.michaels, y por último encontraremos que el usuario r.michaels puede ejecutar cualquier comando con doas (equivalente a sudo en NetBSD) para convertirnos en el usuario root."
date: 2021-03-28
classes: wide
header:
  teaser: /assets/images/Luanne-Hackthebox/luanne.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Web Exploiting
  - Privilege Escalation
  - Pentesting
  - BSD
---

![](/assets/images/Luanne-Hackthebox/luanne.png)
![](/assets/images/Luanne-Hackthebox/dificultad.png)

Resolución de la máquina *Luanne* de Hack The Box, una máquina NetBSD de dificultad fácil según la plataforma, en la cual accedemos con credenciales por defecto al servicio *medusa supervisor process manager*, encontramos un script en lua vulneable a inyección de código a través del cual accederemos a la máquina. Posteriormente encontraremos la clave privada de ssh del usuario r.michaels y accederemos por ssh, para encontrar un fichero de backup encriptado que conseguiremos desencriptar y nos revelará la contraseña del usuario r.michaels, y por último encontraremos que el usuario r.michaels puede ejecutar cualquier comando con doas (equivalente a sudo en NetBSD) para convertirnos en el usuario root.

