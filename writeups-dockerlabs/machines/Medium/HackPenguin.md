

> Habilidades: Enumeración, esteganografía, fuerza bruta, escalada de privilegios
> 
![intro](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Medium/images/Intro.png?raw=true)

# RECONOCIMIENTO

## Nmap Scan

Realizamos el primer sondeo a la IP de la víctima para averiguar qué puertos están abiertos:

~~~ bash
nmap --open -p- -n -sS -Pn --min-rate=5000 $ip -oG FirsScan
~~~

![Primer_Escaneo](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Medium/images/FirsScan.png?raw=true)

- `--open`: Mostrar los puertos abiertos.
- `-p-`: Escanear todo el rango de puertos (65535)
- `-sS`: Escaneo TCP SYN, técnica más sigilosa para determinar el estado del puerto.
- `-n`: No aplicar resolución DNS.
- `-Pn`: Deshabilitar el descubrimiento de host, asumiendo que el objetivo se encuentra activo.
- `-oG`: Exportar el escaneo a un formato `Grepable`, util para extraer información.
- `--min-rate`: Tasa mínima de paquetes por segundo.

_____________________________________________________________________________________________________________________________________________________________________

### Enumerar Servicios

Realizamos enumeración a los puertos abiertos lanzando los scripts predeterminados e indicando la versión de los servicios. (-sC -sV)
~~~ bash
nmap -sVC -p 22,80 172.17.0.2 -oN ServicesScan
~~~

![ServicesScan](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Medium/images/ServicesScan.png?raw=true)


- `-sC`: Ejecuta scripts NSE (Nmap Scripting Engine) predeterminados.
- `-sV`: Detecta la versión del servicio que se está ejecutando en cada puerto abierto.

_____________________________________________________________________________________________________________________________________________________________________

## Http Service

Vemos que el puerto `80` corresponde a un servicio HTTP. Vamos a investigar qué hay en él. Podemos usar la herramienta `whatweb` para listar las tecnologías detectadas en el servidor. (Como alternativa, puedes usar la extensión de navegador `Wappalyzer`).

~~~ bash
whatweb http://172.17.0.2
~~~

![whatweb](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Medium/images/WhatWeb.png?raw=true)

Ahora nos dirigimos a reisar la web con el explorador. Vemos la pagina default del servidor Apache.

![Apache2Web](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Medium/images/UbuntuWeb.png?raw=true)

Ahora realizaremos fuzzing para ver si encontramos algun directorio.


### Gobuster
~~~ bash
gobuster dir -u http://172.17.0.2 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 200 -x php,txt,html,php.bak
~~~
![gobuster](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Medium/images/Gobuster.png?raw=true)

- `dir`: Modo de descubrimiento de directorios y archivos
- `-u`: Dirección URL
- `-w`: Diccionario a usar
- `-t 200`: Establecer 200 subprocesos 
- `-x php,txt,html,php.bak`: Busca extensiones de archivo

Encontramos un directorio interesante, penguin.html, nos dirigimos a la página a ver que podemos encontrar.

![penguinweb](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Medium/images/penguinweb.png?raw=true)

Como siempre, revisamos también el código fuente, pero no encontramos nada. Vamos a revisar la imagen por si contuviera información oculta en los píxeles menos relevantes, usaremos la herramienta `steghide`. Con las opciones `extract` y `sf`.


## Stego

~~~
steghide extract -sf imagen.jpg
~~~

![steghide_pass](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Medium/images/steghide_pass.png?raw=true)

El contenido de la imagen parece estár protegido por una contraseña. Hay herramientas para realizar fuerza bruta vamos a ello, `stegseek`.

~~~
stegseek penguin.jpg
~~~

![penguibruteforze](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Medium/images/stegkek.png?raw=true)

Encontró la contraseña muy easy, exportó un arhivo con extensión kdbx y relativo a un gestor de contraseñas: `KeepPass`.





Al parecer, `chocolate` es la contraseña para extraer el archivo kdbx.
Lo extraemos e intentamos abrir con la misma contraseña sin suerte. Intentaremos fuerza bruta con la herramienta de ´keepass2john´ y `John`

![Extract_choco]()

Primero, lo pasamos a un formato legible para john.

![Keepastohash]()

~~~
keepass2john penguin.kdbx > hash
~~~

A continuación, hacemos fuerza bruta al archivo `hash`.

![bruteforce]()

~~~
john --wordlist=/usr/share/wordlists/rockyou.txt hash
john --show hash
~~~
**
<details>
    <summary>Script para la misma tarea que John</summary>

```bash
#!/bin/bash

# Pedir rutas al usuario con ejemplos
read -p "Ruta del archivo diccionario (ej: /usr/share/wordlists/rockyou.txt): " rockyou_file
read -p "Ruta del archivo imagen (ej: /home/user/Desktop/DockerLabs/HackPenguin/penguin.jpg): " image_file
read -p "Ruta del archivo kdbx (ej: /home/user/Desktop/DockerLabs/HackPenguin/database.kdbx): " database_file

# Variables
output=""
nIntentos=0

# Verificar si los archivos existen
[[ ! -f "$rockyou_file" ]] && echo "rockyou.txt no existe en la ruta indicada" && exit 1
[[ -f "$database_file" ]] && echo "kdbx existe. Borrándolo..." && rm "$database_file"
[[ ! -f "$image_file" ]] && echo "imagen.jpg no existe en la ruta indicada" && exit 1

# Leer y probar contraseñas
while IFS= read -r password; do
    output=$(steghide --extract -sf "$image_file" -p "$password" 2>&1)
    let "nIntentos++"
    if [[ $output == *"no pude"* ]]; then
        echo -ne "Probando... : $password\rNúmero de contraseñas probadas: $nIntentos\r"
    else
        echo -e "\n------------------------------------\nContraseña encontrada: $password"
        break
    fi
done < "$rockyou_file"
```
</details>
**

Abrimos el archivo con la contraseña encontrada, para ello usamos keepassxc:

![Keepasxc]()
~~~
keepassxc penguin.kdbx
~~~

Una vez dentro obtenemos usuario y contraesña.

![pinguinpass]()


# Escalada de Privilegios

Nos conectamos al host mediante ssh con las credenciales y empezamos a enumerar el entorno.

![ssh_enum]()
~~~
ssh penguin@172.17.0.2
~~~

Lanzamos algunos comandos básicos y observamos un cronjob ejecutando un script cada cierto tiempo.

![basics_commands]()

~~~
sudo -l

~~~

















