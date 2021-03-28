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

Resolución de la máquina *Luanne* de Hack The Box, una máquina NetBSD de dificultad fácil según la plataforma, en la cual accedemos con credenciales por defecto al servicio *medusa supervisor process manager*, encontramos un script en lua vulneable a inyección de código a través del cual accederemos a la máquina. Posteriormente encontraremos la clave privada de ssh del usuario r.michaels y accederemos por ssh, para encontrar un fichero de backup encriptado que conseguiremos desencriptar y nos revelará la contraseña del usuario r.michaels, y por último encontraremos que el usuario r.michaels puede ejecutar cualquier comando con doas (equivalente a sudo en NetBSD) para convertirnos en el usuario root.

## Escaneo de puertos con nmap

Empezamos escaneando todo el rango de puertos y exportando el resultado en el fichero allPorts.

```bash
nmap -p- -sS --min-rate 5000 --open -vvv -n 10.10.10.218 -Pn -oG allPorts
```
![](/assets/images/Luanne-Hackthebox/allPorts.png)

Luego hacemos un escaneo con scripts por defecto y detección de versiones a los puertos abiertos encontrados.

```bash
nmap -sC -sV -p22,80,9001 10.10.10.218 -oN targeted
```
![](/assets/images/Luanne-Hackthebox/targeted.png)

Vemos que en puerto 80 hay un servidor web nginx y en el puerto 9001 un Medusa httpd 1.12 (Supervisor process manager).

## Acceso inicial

Empezamos con el puerto 80. Accedemos a la web y vemos que nos pide autenticación. Probamos con credenciales típicas pero no hay suerte, y al cancelar vemos que aparece una url que apunta al puerto 3000 127.0.0.1:3000

![](/assets/images/Luanne-Hackthebox/puerto80.png)


