---
layout: single
title: Zerologon en Windows Server 2019
excerpt: "En éste artículo atacaremos un Windows Server 2019 con el rol de Active Directory utilizando Zerologon para obtener los hashes de las contraseñas de todos los usuarios y máquinas del
dominio. Quiero recalcar que para realizar éste ataque no necesitamos
credenciales de ningún tipo ni estar unidos al dominio, lo que hace de ésta
vulnerabilidad una de las más críticas encontradas hasta la fecha. Teniendo el
hash de la contraseña del administrador del dominio utilizaremos la conocida
técnica de pass the hash para obtener acceso al sistema con la cuenta del
administrador."
date: 2021-03-13
classes: wide
header:
  teaser: /assets/images/Zerologon/zerologon.png
  teaser_home_page: true
categories:
  - Windows
tags:
  - Windows
  - Privilege Escalation
  - Pentesting
---
