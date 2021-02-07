---
layout: single
title: Doctor - Hack The Box
excerpt: "Una máquina muy interesante, en la cual nos aprovechamos de una vulnerabilidad llamada *Server Side Template Injection* para la intrusión inicial, y de una mala configuración en splunkd para la escalada de privilegios."
date: 2021-02-07
classes: wide
header:
  teaser: /assets/images/Doctor-Hackthebox/doctor-hackthebox.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Sqli
  - Web Exploiting
  - Privilege Escalation
  - Python
  - Pentesting
---


![](/assets/images/Doctor-Hackthebox/doctor-hackthebox.png)

Una máquina muy interesante, en la cual nos aprovechamos de una vulnerabilidad llamada Server Side Template Injection para la intrusión inicial, y de una mala configuración en splunkd para la escalada de privilegios.

## Escaneo de puertos

Empezamos haciendo un escaneo de puertos con nmap. Vemos que están abiertos los puertos 22, 80 y 8089 que corresponden a los servicios SSH, Apache y Splunkd respectívamente.

![](/assets/images/Doctor-Hackthebox/nmap-doctor.png)

Accedo a la página web a través de la ip.

![](/assets/images/Doctor-Hackthebox/doctor-home.png)

VIendo la página principal veo que hay una dirección de email info@doctors.htb, así que añado doctors.htb a mi fichero hosts para que resuelva el dominio y obtengo un panel login.

![](/assets/images/Doctor-Hackthebox/doctor-login.png)

Me registro en la página y veo que se pueden postear mensajes, y además en el código fuente de la página el desarrollador ha dejado un comentario que dice que la página archive está en fase beta.

![](/assets/images/Doctor-Hackthebox/doctor-beta.png)

Viendo la página http://doctors.htb/archive aparece en blanco, pero en el código fuente se ve que recoge como parámetro lo que se ponga en el título del mensaje que se postea.
Con wappalizer veo que la web utiliza el lenguaje python y la tecnología flask.
Busco información acerca de como explotar flask y encuentro lo siguiente:

[https://www.exploit-db.com/exploits/46386](https://www.exploit-db.com/exploits/46386)<br>
[https://pequalsnp-team.github.io/cheatsheet/flask-jinja2-ssti](https://pequalsnp-team.github.io/cheatsheet/flask-jinja2-ssti)<br>
[https://www.onsecurity.co.uk/blog/server-side-template-injection-with-jinja2/](https://www.onsecurity.co.uk/blog/server-side-template-injection-with-jinja2/)<br>

Parece ser que se puede explotar una vulnerabilidad llamada `Server Side Template Injection`.
Éstos links ayudan a entender la vulnerabilidad, y en el último link veo que se pueden ejecutar comandos con ésta inyección:
```html
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```
Creo un mensaje que contenga la inyección en el título
