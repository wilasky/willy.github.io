
> Habilidades: Enumeración, análisis de archivos, decode Brainfuck, Escalada de privilegios (linux)


# RECONOCIMIENTO

## Nmap Scan

Realizamos el primer sondeo a la ip de la victimma para averiguar que puertos están abiertos:

~~~ bash
nmap --open -p- -n -sS -Pn $ip -oG FirsScan
~~~

![Primer_Escaneo](https://i.imgur.com/NssolyM.png)

- `--open`: Nos muestre los puertos abiertos.
- `-p-`: Escanear todo el rango de puertos (65535)
- `-sS`: Escaneo TCP SYN, técnica más sigilosa para determinar el estado del puerto.
- `-n`: No aplicar resolución DNS.
- `-Pn`: Deshabilitar el descubrimiento de host, asumiendo que el objetivo se encuentra activo.
- `-oG`: Exportar el escaneo a un formato `Grepable`, util para extraer información.

_____________________________________________________________________________________________________________________________

### ENUMERAR SERVICIOS
Realizamos enumaeración a los puertos abiertos.
~~~ bash
nmap -sVC -p 22,80 172.17.0.2 -oN ServicesScan
~~~
- `-sC`: Ejecuta scripts NSE (Nmap Scripting Engine) predeterminados.
- `-sV`: Detecta la versión del servicio que se está ejecutando en cada puerto abierto.

![Escaneo_Sevicios](https://i.imgur.com/6ybGn1A.png)

_____________________________________________________________________________________________________________________________

## Http Service

Vemos el puerto `80` que corresponde a un servicio `http`, veamos que hay en él. Podemos usar la herramienta `whatweb` para listar las tecnologías detectadas en el servidor. (Como alternativa, puedes usar la extension de navegador Wappalyzer)

~~~ bash
whatweb http://172.17.0.2
~~~

![whatweb](https://i.imgur.com/iz3f6TK.png)

Abrimos en navegador y vamos a dicha web, leemos y podemos ver algo interesante, una nota de `Carlota`.
Yo siempre reviso view-source de la web por si se me escapa algo interesante. Siempre hay que enumerar todo lo que se pueda.
![despido](https://i.imgur.com/JTVpkU4.png)



## Fuzzing

El siguiente paso será intentar descubrir posibles directorios, podemos usar cualquier herramienta de fuzzing, yo usaré `wfuzz` y `gobuster`

### Wfuzz
~~~ bash
wfuzz -c --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 http://172.17.0.2/FUZZ
~~~

![wfuzz](https://i.imgur.com/GEns8Jg.png)

- `-c`: Formato colorizado
- `--hc=404`: Ocultar el código de estado 404 (Not Found)
- `-w`: Especificar un diccionario de palabras
- `-t 200`: Dividir el proceso en 200 hilos, agilizando la tarea


### Gobuster

~~~ bash
gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
~~~

![gobuster](https://i.imgur.com/qsO45ms.png)

- `dir`: Modo de descubrimiento de directorios y archivos
- `-u`: Dirección URL
- `-w`: Diccionario a usar
- `-t 200`: Establecer 200 subprocesos 

Podemos ver que en ambos casos no se ha descubierto ningun directorio. :(

## Fuerza Bruta

Como obtuvimos dos nombres anteriormente, vamos a intenta hacer fuerza bruta con uno de ellos.
~~~ bash
hydra -l carlota -P /usr/share/wordlists/rockyou.txt -t4 ssh://172.17.0.2
~~~

![hydra](https://i.imgur.com/gHUFhTq.png)


- `-l`: Especificar un nombre.
- `-P`: Especificar una lista de palabras.
- `-t4`: Especificar hilos a usar. (más rapido)
- `ssh`: Especifica el protocolo.




![dir images]()

Vemos un archivo  `agua_ssh.jpg`, procedemos a traerla a nuestra máquina y ver de qué se trata


Si hacemos un análisis del archivo o imprimimos los caracteres de la imagen, no encontramos mayores pistas

## Exiftool Analysis

~~~ bash
exiftool agua_ssh.jpg
~~~

![exiftool_analysis]()


## Strings

~~~ bash
strings agua_ssh.jpg
~~~

![listar caracteres]()


# Intrusión
---
## Brainfuck Decode

Si vemos el código fuente podemos ver algo inusual al final del todo en un comentario HTML

![web codigo raro]()

Como no sabía a qué me enfrentaba, lo primero que hice fue buscar este comentario HTML en Google

![decode ]()

Podemos ver que hay un mensaje escrito en lenguaje Brainfuck, así que lo decodificaremos con ayuda de esta web `https://dcode.fr/brainfuck-language`
**No olvidemos quitar los caracteres del comentario HTML** (`<!--` y `-->`)

![decrypt]()

Vemos que el mensaje escondido se trata de la palabra `bebeaguaqueessano`, quizá sea la contraseña del usuario `agua`, por lo que probamos conectarnos por `ssh`

~~~ bash
ssh agua@aguademayo.local
~~~

![entramos por ssh]()


# Escalada de privilegios
---
## Tratamiento TTY

Una vez tenemos acceso, haremos un tratamiento de la TTY para limpiar la pantalla con `Ctrl + L`, para esto cambiaremos el valor de la variable `$TERM`

~~~ bash
export TERM=xterm
~~~

## Sudoers

Primeramente veremos si tenemos privilegios `sudo` con el siguiente comando

~~~ bash
sudo -l
~~~

![sudo -l]()

- `-l`: Enumerar los comandos permitidos (o prohibidos) invocables por el usuario en la máquina actual


Existe `bettercap` en la máquina y podemos ejecutarlo sin proporcionar contraseña, ejecutaremos el binario

~~~ bash
sudo bettercap
~~~

![exec bettercap with sudo](https://github.com/user-attachments/assets/8b2b3182-7951-491f-a10f-8ef29cd2139e)

Veamos el panel de ayuda con el comando `help`

![bettercap help](https://github.com/user-attachments/assets/3752382d-c7d9-4f66-acbb-784d1a8385f3)

![help bettercap](https://github.com/user-attachments/assets/42e3730f-aa2b-414c-8185-d124b5e4cd7b)


## SUID Privilege Escalation

Esta opción (`!`) nos permite ejecutar un comando a nivel de sistema, así que podemos asignar el privilegio SUID a la `bash` para ejecutarla como `root`, para eso, lo haremos con el comando `chmod`, dentro de la consola interactiva de `bettercap` ejecutamos el siguiente comando

~~~ bash
! chmod u+s /bin/bash
~~~

También podemos hacer esto con un solo comando con la opción `-eval` 

~~~ bash
sudo bettercap -eval "! chmod u+s /bin/bash"
~~~

![cambiar el privilegio onliner](https://github.com/user-attachments/assets/48db4ad6-d528-42c2-a14d-f0fd8c97b612)

- `-eval`: Ejecutar un comando en la máquina


## Root time

Ahora supuestamente asignamos la capacidad de ejecutar `bash` como el usuario `root`, lo podemos verificar con el comando `ls -l /bin/bash` para listar los permisos.

![bash permisions](https://github.com/user-attachments/assets/d3d00e95-9394-47b6-8d64-6bb444c6eae4)

Así que ejecutamos `bash` como el propietario, y nos convertimos en el usuario `root`

~~~ bash
bash -p
~~~

- `-p`: Ejecutar como el usuario original

![bash como root](https://github.com/user-attachments/assets/3bdbe0a3-ad67-4758-80ed-03c209b20d9b)
