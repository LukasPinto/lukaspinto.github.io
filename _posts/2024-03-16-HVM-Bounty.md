---
title: HackMyVM - Bounty - Writeup
author: lukas
date: 2024-03-16 12:00:00 +0800
categories: [HackMyVM]
tags: [Gitea,MSF,Cron Task]
pin: true
image:
    path: /images/2024-03-16-HVM-Bounty/bounty.png
---


## Renocimiento

### Escaneo de puertos

Comenzamos haciendo un escaneo de todos los puertos de la máquina victima para lograr encontrar cuales están abiertos y cuales no.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled.png)

### Escaneo servicios

Ahora una vez obtenidos los puertos podemos examinar más a fondo cada uno para encontrar el servicio y versión asociados.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%201.png)

El ssh es una versión “más o menos” reciente y el otro corresponde a un servicio web.

---

## Ganando acceso

### Fuzzing

Para esta parte primero reviso la web, la examino un poco.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%202.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%203.png)

Lo primero que me llama la atención es que en la raiz de la web hay un output que parece una tarea cron y en la página que se nos indica no aparece nada. Ya que no hay más información que esto hago `Fuzzing`.

 `wfuzz -c --hc 404 -u [http://192.168.1.106/FUZZ.FUZ2Z](http://192.168.1.106/FUZZ.FUZ2Z) -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -z list,html-php`

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%204.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%205.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%206.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%207.png)

### CuteEditor.php

Ya que encontramos que se está utilizando una app llamada CuteEditor versión 6.6, busco alguna vuln relacionada a esta version y encuentro lo siguiente.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%208.png)

Luego de probar esta vuln no me resulta ya que requiere de poder modificar el nombre del archivo, pero en el panel que se nos otorga en `templates.php` no se puede hacer esto, pero si se puede guardar.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%209.png)

Si le damos a guardar, automaticamente se nos guarda el templates que aparece debajo, pero no sabemos donde se guarda, para esto reviso el `documento.html` y encuentro que ahí se almacena el contenido del recuadro.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2010.png)

Ahora si juntamos lo que hemos encontrado, puede ser que haya una tarea cron ejecutando este document.html con php, por ende pruebo esto.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2011.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2012.png)

Pero luego al revisar me percato en que no cambia, esto puede ser a que por defecto no toma el contenido del recuadro por ende lo hago desde burpsuite para modificar directamente la petición.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2013.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2014.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2015.png)

Como observamos si existia una tarea cron por detrás. Ahora nos montamos una shell e ingresamos. (En esta parte probe con muchos payload y me fije en que los caracteres como `>` o `&` no funcionaban bien por lo que lo hice con netcat).

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2016.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2017.png)

Con esto ya estamos dentro y tenemos la primera flag.

---

## Escalada a root

Examinando los permisos, binarios suid, grupos, etc. encuentro lo siguiente con el comando sudo.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2018.png)

Podemos ejecutar un binario gitea como el usuario primavera.

### Explotación Gitea

Pruebo ejecutar el comando para ver lo que hace.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2019.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2020.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2021.png)

Nos encontramos con un servicio de Gitea versión 1.16.6 y  si buscamos una vuln asociada a esta versión encontramos un exploit de MSF.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2022.png)

Pruebo este exploit con MSF 

### Agregando un exploit a MSF

Para agregar el exploit nos dirigimos a `/root/.msf4/modules/exploits` y creamos una carpeta con el nombre del servicio o un nombre descriptivo y dejamos el exploit ahí.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2023.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2024.png)

Ahora con el exploit cargado y configurado, lo ejecutamos y obtenemos un shell.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2025.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2026.png)

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2027.png)

Con esto ya estamos como primavera, a parte me monto una revershell con netcat para trabajar más comodo.

### SSH y Root

Examinando los archivos del usuario primavera, encuentro que hay una nota.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2028.png)

Que dice que somos el admin de las sombras, cosa que supongo que es una pista sobre los permisos del `/etc/shadow` .

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2029.png)

Pero nada, luego de probar varias cosas y buscar, reviso la id_rsa y me intento conectar por ssh, pero me sale este error al conectarme como primavera.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2030.png)

La id_rsa en el directorio `.ssh` del usuario primavera no corresponde a primavera, pero si a root.

![Untitled](/images/2024-03-16-HVM-Bounty/Untitled%2031.png)

Esta parte no la entendí bien y me tuvo bastante tiempo pegado, pero es un recordatorio de que siempre vale la pena probrar todas las formas de explotar o acceder a una máquina, por último con esto estariamos con privilegios máximos y tendriamos la última flag.
