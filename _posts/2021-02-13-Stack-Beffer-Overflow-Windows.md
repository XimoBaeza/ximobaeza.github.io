---
layout: single
title: Stack Buffer Overflow en Windows
excerpt: "En este artículo vemos cómo explotar un stack buffer overflow en una aplicación vulnerable llamada SLMail corriendo en una máquina virtual con windows 7 x86."
date: 2021-02-13
classes: wide
header:
  teaser: /assets/images/Stack-Buffer-Overflow-Windows/buffer-overflow-windows.jpg
  teaser_home_page: true
categories:
  - Buffer-Overflow
tags:
  - Buffer Overflow
  - Python
  - Pentesting
---

![](/assets/images/Stack-Buffer-Overflow-Windows/buffer-overflow-windows.jpg)

En este artículo veremos cómo explotar un buffer overflow en una aplicación vulnerable llamada SLMail corriendo en una máquina virtual Windows 7 x86.

Primero quiero agradecer a s4vitar por su tutorial y por servirme de inspiración.

Necesitaremos una máquina virtual con windows 7 x86, el software SLMail v5.5, inmunity debuguer y mona.py para añadir funcionalidades al debuguer.

Links de descarga:

[SLMail 5.5](https://slmail.software.informer.com/5.5/)<br>
[Inmunity debuguer](https://www.immunityinc.com/products/debugger/)<br>
[Mona.py](https://github.com/corelan/mona/blob/master/mona.py)<br>

Instalaremos SLMail y inmunity debuguer en la máquina windows, y luego añadiremos mona al inmunity debuguer. Para ello descargaremos el script y lo colocaremos en la ruta C:\Program Files\Inmunity Inc\Inmunity Debuguer\PyCommands\.

Además tendremos que crear reglas en el firewall de windows para permitan el tráfico en los puertos 25 y 110. Y por último deshabilitar el DEP con el siguiente comando ejecutado como administrador:

`bcdedit.exe /set {current} nx AllwaysOff`

Cuando tengamos esto ya podemos empezar a practicar.

Si lanzamos un nmap a la ip de la máquina tiene que responder con los puertos 25 y 110 abiertos, entre otros.

![](/assets/images/Stack-Buffer-Overflow-Windows/nmap-slmail.png)


