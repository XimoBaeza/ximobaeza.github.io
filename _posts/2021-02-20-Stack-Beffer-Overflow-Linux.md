---
layout: single
title: Stack Buffer Overflow en Linux
excerpt: "En este artículo vemos cómo explotar un stack buffer overflow en un sistema Linux. Nos aprovecharemos de un desbordamiento de memoria en un binario en el que a nivel de código no se ha tenido en cuenta el controlar la cantidad de bytes que se le pueden enviar. Además, al tener el binario permisos SUID nos permitirá realizar una escalada de privilegios y obtener una shell de root."
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

En este artículo vemos cómo explotar un stack buffer overflow en un sistema Linux. Nos aprovecharemos de un desbordamiento de memoria en un binario en el que a nivel de código no se ha tenido en cuenta el controlar la cantidad de bytes que se le pueden enviar. Además, al tener el binario permisos SUID nos permitirá realizar una escalada de privilegios y obtener una shell de root.

De nuevo agradecimientos a s4vitar por sus tutoriales.

Utilizaremos una máquina virtual con Ubuntu Server 14.04 x86, gdb como debuguer y peda para añadir funcionalidades extra a gdb.

Nos conectamos por ssh a la máquina ubuntu y empezamos.

![](/assets/images/Stack-Buffer-Overflow-Linux/ssh.png)

Primero instalaremos gdb y git desde la línea de comandos `apt install gdb git`

Después necesitaremos instalar peda como complemento de gdb, para ello vamos a la página del proyecto en github [https://github.com/longld/peda](https://github.com/longld/peda) y ejecutamos estos dos comandos:

`git clone https://github.com/longld/peda.git ~/peda`<br>
`echo "source ~/peda/peda.py" >> ~/.gdbinit`

A continuación crearemos un sencillo código en lenguaje C para después compilarlo y obtener el binario sobre el que trabajaremos.

```
#include <stdio.h>

void vulnerable(char *buff){
        char buffer[64];
        strcpy(buffer, buff);
}

void main(int argc, char **argv){
        vulnerable(argv[1]);
}

```

