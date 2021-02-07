---
layout: single
title: Doctor - Hack The Box
excerpt: "Una máquina muy interesante, en la cual nos aprovechamos de una vulnerabilidad llamada Server Side Template Injection para la intrusión inicial, y de una mala configuración en splunk para la escalada de privilegios."
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

<p align="center">
<img src="/assets/images/Doctor-Hackthebox/doctor-hackthebox.png">
</p>

Una máquina muy interesante, en la cual nos aprovechamos de una vulnerabilidad llamada Server Side Template Injection para la intrusión inicial, y de una mala configuración en splunk para la escalada de privilegios.

## Escaneo de puertos

Empezamos haciendo un escaneo de puertos con nmap. Vemos que están abiertos los puertos 22, 80 y 8089 que corresponden a los servicios SSH, Apache y Splunkd respectívamente.

![](/assets/images/Doctor-Hackthebox/nmap-doctor.png)




