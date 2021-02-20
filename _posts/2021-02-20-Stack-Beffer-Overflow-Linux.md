---
layout: single
title: Stack Buffer Overflow en Linux
excerpt: "En este artículo vemos cómo explotar un stack buffer overflow en un sistema Linux. Nos aprovecharemos de un desbordamiento de memoria en un binario compilado con unos parámetros que lo hacen vulnerable. Además, al tener el binario permisos SUID nos permitirá realizar una escalada de privilegios y obtener una shell de root."
date: 2021-02-20
classes: wide
header:
  teaser: /assets/images/Stack-Buffer-Overflow-Linux/buffer-overflow.jpg
  teaser_home_page: true
categories:
  - Buffer-Overflow
tags:
  - Buffer Overflow
  - Linux
  - Pentesting
---

![](/assets/images/Stack-Buffer-Overflow-Linux/buffer-overflow.jpg)

En este artículo vemos cómo explotar un stack buffer overflow en un sistema Linux. Nos aprovecharemos de un desbordamiento de memoria en un binario compilado con unos parámetros que lo hacen vulnerable. Además, al tener el binario permisos SUID nos permitirá realizar una escalada de privilegios y obtener una shell de root.

De nuevo agradecimientos a s4vitar por sus tutoriales.

Utilizaremos una máquina virtual con Ubuntu Server 14.04 x86, gdb como debuguer y peda para añadir funcionalidades extra a gdb.

Nos conectamos por ssh a la máquina ubuntu y empezamos.

![](/assets/images/Stack-Buffer-Overflow-Linux/ssh.jpg)


