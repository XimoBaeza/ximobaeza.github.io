---
layout: single
title: Smb Relay - Parte 3
excerpt: "En esta tercera parte veremos como configurar una gpo en el controlador de dominio para conseguir que las peticiones SMB vayan firmadas, deshabilitar los protocolos LLMNR y NBT-NS, y añadir una clave de registro para deshabilitar también mDNS y conseguir que el responder no pueda capturar tráfico de red en todo el dominio."
date: 2022-10-16
classes: wide
header:
  teaser: /assets/images/Smb-Relay/responder2.png
  teaser_home_page: true
categories:
  - Ciberseguridad
tags:
  - Windows
  - Privilege Escalation
  - Pentesting
---

En esta tercera parte veremos como configurar una gpo en el controlador de dominio para conseguir que las peticiones SMB vayan firmadas, deshabilitar los protocolos LLMNR y NBT-NS, y añadir una clave de registro para deshabilitar también mDNS y conseguir que el responder no pueda capturar tráfico de red en todo el dominio.


