---
title: HackTheBox - Authority - Writeup
author: lukas
date: 2023-12-26 12:00:00 +0800
categories: [HackTheBox]
tags: [Windows,Active Directory,AD Enumeration,Cracking Hash,ACL,Ansible,Windows ADCS,PassTheCert,PassTheHash,Responder,Scripting]
pin: true
image:
    path: /images/2023-12-26-HTB-Authority/authority.png
---
## Reconocimiento

### Escaneo de puertos

Para iniciar la resolución de esta máquina hacemos un escaneo de puertos para detectar cuales están abiertos.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled.png)

En un principio detectamos muchos puertos abiertos y además estos parecen corresponder a puertos típicos de servicios como smb, ldap, winrm, kerberos, etc. Por lo que podríamos decir que nos enfrentamos a un entorno de Directorio Activo o en su defecto a un IIS de Windows.

`nmap -p- -sS --min-rate 5000 -Pn -n -oG allPorts 10.10.11.222`

### Escaneo de servicios

Para esta fase lanzo un conjunto de script básicos de reconocimiento con nmap.

```bash
nmap -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,8443,9389,47001,49664,49665,49666,49667,49679,49688,49689,49691,49692,49700,49708,49712,49731 -sCV -oN targeted 10.10.11.222
```

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%201.png)

Con este escaneo confirmamos que nos enfrenteamos a un entorno de directorio activo, por lo que ahora añadiré los dominios encontrados al `/etc/hosts`.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%202.png)

Ahora con los dominios podemos examinar cada servicio más detenidamente.

---

## Enumeración

### Web

Del escaneo de servicios me fijo en un que parece corresponde una pagina web por  `HTTPS`, por lo que le echo un vistazo 

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%203.png)

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%204.png)

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%205.png)

Interesante, ya que podemos ver un panel de login y al acceder los paneles de editor podemos observar inicios de sesión anterior los cuales corresponder al usuario `svc_pwm` y al dominio `htb.corp`. Después de investigar un poco llego al siguiente repositorio de github.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%206.png)

Parece ser que este servicio funciona junto con el servicio de LDAP y es un servicio el cual permite cambiar la contraseña del usuario en un directorio de LDAP.

En cuanto al usuario usuario encontrado compruebo si este es un usuario válido en el sistema.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%207.png)

Dado que si lo es, pruebo bruteforcear, enumerar, conseguir un TGT con un ASREPRoast, etc. No logro encontrar nada, por ende paso ver si puedo obtener más información en otro servicio.

### Smb

Enumero smb para obtener información del sistema al que me enferento y para ver los recursos compartidos a nivel de red.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%208.png)

Al parecer no puedo listar recursos a nivel de red con crackmapexec, por lo que como es habitual en estos casos siempre es recomendable probar con herramientas alternativas.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%209.png)

como podemos observar con `smbclient` si se listan los recursos a nivel de red, a pesar del error que nos muestra, de estos recursos me llaman la atención 2 de ellos `Department Shares` y `Development`.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2010.png)

En el recurso de `Department Shares` no tenemos permisos, pero en el de `Development` si, entonces me traere este recuros a mi equipo jugando con monturas del tipo `cifs`.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2011.png)

### Ansible Playbook

Al revisar el contenido del recuros me encuentro con que se está utilizando un `Ansible Playbook`, el cual sirve para organizar procesos de TI.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2012.png)

Luego reviso los archivos `.yml`, para ahorrar tiempo utilizo una busqueda recursiva con grep para ver si hay archivos que contengan la palabra pass.

`grep -ri 'pass' | awk '{print $1}' FS=':' | sort -u`

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2013.png)

Ahora revisando estos archivo encuentro lo siguiente en `Automation/Ansible/PWM/defaults/main.yml`

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2014.png)

Hay vaults con contenido encriptado en este archivo, por lo que con `ansible2john` puedo extraer un hash para luego intentar crackear.

1. Primero separo los conenidos de cada vault.
    
    ![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2015.png)
    
2.  Luego extraigo lo hashes de estos vault y los guardo en un archivos `hashes`.
    
    ![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2016.png)
    
3. Ahora intento crackear estos hashes con `john`.
    
    ![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2017.png)
    
4. Por último visualizamos su contenido con `ansible-vault`.
    
    ![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2018.png)
    

de acá obtenemos 1 usuario y la contraseña de svc_pwm al parecer y svc_ldap.

## Explotación

### Responder

Ahora si recordamos el servicio del puerto 8443 teniamos un intento de login con el usuario svc_pwm.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2019.png)

Pero si probamos con la opción de `Configuration Editor` accedemos al siguiente panel, notar que la web se intenta conectar utilizando el usuario `svc_ldap` y nosotros teniamos una contraseña en `ldap_admin_password` pero esta no es válida.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2020.png)

Investigando y fijandonos en el error que nos daba cuando nos intentamos loguear este servicio se intenta conectar a otro servicio LDAP remoto, luego si nos fijamos en panel de settings → default → connections.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2021.png)

Podemos modificar al punto en donde se está conectando, por lo que podriamos intentar capturar las credenciales que utiliza este para intentar conectarse, esto lo podriamos hacer con netcat, wireshark, responder, etc. En este caso lo haré con responder ya que es más cómodo y reporta de una forma más clara las credenciales obtenidas.

1. Lanzar responder.
    
    ![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2022.png)
    
2. Modificar la configuración con nuestra ip y puerto de ldap, y recordar guardar los cambios.
    
    ![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2023.png)
    
3. Ahora nos intentamos loguear para que se intente conectar a nuestro equipo.
    
    ![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2024.png)
    

Y listo capturamos la credencial del usuario `svc_ldap`, esta correponde a `svc_ldap:lDaP_1n_th3_cle4r!`

Ahora con estas credenciales probamos en que servicio son válidas y obtenemos esto.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2025.png)

Son válidas para el servicio de `winrm` por lo que con `evil-winrm` me conecto a la máquina victima para obtener una powershell como este usuario.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2026.png)

---

## Escalada

Para esta parte contemplo 3 formas de elevar privilegios, esto con el fin de practicar.

### ADCS

Para esta parte como el usuario svc_ldap examino los permisos del usuario, archivos, grupos, etc. Encuentro lo siguiente.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2027.png)

Al parecer por el grupo en está este usuario puede que exista un ADCS (Active Directory Certificate Services), dado esto intento enumerar este servicio utilizando [certipy](https://github.com/ly4k/Certipy).

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2028.png)

Ahora reviso lo que logró recolectar de los templates existentes en el sistema.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2029.png)

Encontró un template con la vulnerabilidad `ESC1`, lo que quiere decir que podremos explotarla probablemente, esto lo haré siguiente los pasos del mismo repositorio de certipy.

Pero existe un problema, solamente los usuarios pertenecientes a `Domain Computers` pueden hacer enroll de este template, por ende debemos obtener un usuario perteneciente a este grupo. Ahora bien si recordamos nosotros teniamos el privilegio de `SeMachineAccountPrivilege` el cual nos permite añadir workstations al dominio.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2030.png)

Esto quiere decir que puede abusar de este privilegio para añadir un nuevo computador al dominio, lo cual se puede hacer utilizando `impacket-addcomputer`.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2031.png)

Ahora podemos solicitar el certificado con el UPN (User Principal Name) de administrator.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2032.png)

Una vez obtenido el certificado del administrador podemos obtener el hash NTLM del usuario administrator, pero ocurre lo siguiente.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2033.png)

Obtenemos un error `Kerberos SessionError: KDC_ERR_PADATA_TYPE_NOSUPP(KDC has no support for padata type)` este error quiere decir que el PKINIT (el mecanismo de pre autenticación de kerberos) no está correctamente configurado por lo tanto la autenticación falla.

Por lo tanto para resolver este problema se puede hacer de las siguientes formas.

### Escalada con ldap shell con certipy

Para esta escalada simplemente utilizamos la herrramienta certipy la cual nos permite directamente spawnear una ldap shell como administrator utilizando el certificado obtenido.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2034.png)

Ya dentro de esta consola podemos ejecutar los siguientes comandos para añadir al usuario `svc_ldap` al grupo de `Domain Admin`.

```bash
# el nombre del usuario lo elegimos nosotros
add_user svc_admin
change_password svc_admin 'svc_admin@svc_admin123!$'
add_user_to_group svc_admin 'Domain Admin'
```

Con esto ya creariamos el usuarioc svc_admin luego nos podriamos conectar al equipo y ejecutar comandos como administrador.

### Escalada con ldap shell de forma manual

Para esta forma manual necesitamos generar la llave y el certificado en archivos por separados, lo cual podemos hacer con certipy.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2035.png)

Ahora con la herramienta [PassTheCert](https://github.com/AlmondOffSec/PassTheCert) podemos spawnear una ldap shell como administrator. 

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2036.png)

Ya en este punto hacemos los mismo pasos para crear el usuario y añadirlo al grupo `Domain Admin`.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2037.png)

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2038.png)

### Escalada con PassTheCert a través de un TGT

Para esta ultima forma de escalar privilegios debemos delegar los permisos del DC sobre el computador fake que creamos, en este caso llamado `$TEST`.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2039.png)

Luego debemos sincronizar nuestro reloj (recomendable siempre que se trabaja con directorio activo).

`timedatectl set-ntp 0` → Ejecutar esto antes en caso de estar en kali linux.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2040.png)

Ahora debemos obtener un Service Ticket.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2041.png)

Con este Service Ticket podemos dumpear las credenciales del dominio con `secretsdump`

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2042.png)

Por último podemos hacer PassTheHash como administrator.

![Untitled](/images/2023-12-26-HTB-Authority/Untitled%2043.png)
