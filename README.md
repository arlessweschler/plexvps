# plexvps
La idea de esta guía es montar un servidor de Plex con Google Drive en un VPS con linux (ubuntu en mi caso). 
Lo acompañaré de algunos extras como: cliente de torrent (transmission), plexpy para monitorizar nuestro servidor plex y algunos scripts y consideraciones para mejorar la experiencia a la hora del visionado.

### Ventajas Plex VPS vs Plex Cloud
- No necesitas Plex Pass.
- El escaneo de directorios es muuucho más rápido porque es como si tuvieras plex instalado en local.
- Posiblidad de instalar plexpy para monitorizar tu plex server, así como otros servicios.

La mayor desventaja que veo a plex en vps es que si quieres transcodificar contenido vas a tener que tirar a por un servidor de mínimo 15-20€ / mes.


## VPS y SSH
En mi caso voy a utilizar un VPS pequeño y barato de Scaleway (https://www.scaleway.com 2.99€) ya que voy a hacer direct play de todo el contenido, pero tened en cuenta que si alguno de vuestros users va a necesitar transcoding, este server no va a poder con ello. Necesitaréis mínimo 4 cores, se recomienda una puntuación de 2000 en passmark para un sólo transcoding a 1080p.

Otras opciones económicas: https://www.kimsufi.com/es/servidores.xml 


Cuando tengamos el vps arrancado, tenemos que entrar por ssh. Seguramente en el panel nos indiquen cómo hacerlo pero lo normal es abrir un sesión en PuTTy (Windows) o Terminal (Mac OS).

En el caso de scaleway, debéis crear una ssh key antes de acceder. Aquí lo explican perfectamente: https://www.scaleway.com/docs/configure-new-ssh-key/

Accederemos con:

```
ssh root@ipvps
```

## plexdrive (alternativa rclone)
Para liberías grandes es recomendable montar la unidad con plexdrive en lugar de rclone. Plexdrive cachea el contenido de tu unidad para no realizar un exceso de peticiones a la API de google drive y de esta forma evitar los baneos.

NOTA: tras probar la v4 y la v5, me quedo con la 4 ya que me ha dado un mejor rendimiento en general. No obstante, en su github tenéis las instrucciones para instalar la v5. Utiliza otro sistema de base de datos y el comando de montaje cambia ligeramente.

Instalamos la base de datos de MongoDB.
```
apt-get install mongodb
```
Descargamos la versión más actual de su web: https://github.com/dweidenfeld/plexdrive/releases (versión amd64).
```
mkdir /home/plexdrive
cd /home/plexdrive
wget https://github.com/dweidenfeld/plexdrive/releases/download/4.0.0/plexdrive-linux-amd64
mv plexdrive-linux-amd64 plexdrive
chown root:root /home/plexdrive/plexdrive
chmod 755 /home/plexdrive/plexdrive
```

Ahora vamos a obtener nuestro client id y client secret de la API de google. Para ello hacemos lo siguiente:
- Nos logueamos en [Google API Console](https://console.developers.google.com/). 
- Creamos un nuevo proyecto.
- Vamos a Overview -> Google APIs, Google Apps APIs, Drive API y Enable.
- Vamos a Credentials en el panel izquierdo y Create Credentials, OAuth client ID.
- En tipo de aplicación seleccionamos Other y Create.
- Nos dara un client id y client secret. Lo copiamos y guardamos.

Instalamos screen para dejar el proceso de montaje corriendo en segundo plano y montamos con las opciones por defecto.
```
apt-get install screen
screen -S plexdrive
mkdir /home/plexcloud
cd /home/plexdrive
./plexdrive -o allow_other -v 3 -m localhost /home/plexcloud
```

Plexdrive empezará a cachear todo el contenido de vuestro plex y puede que tarde bastante, deberíamos dejarle hacer hasta que ponga que ha acabado o haya parado la actividad.

Para salir de la screen, simplemente hacemos Control + A + D. Si queremos volver a ella podemos hacerlo con: screen -r plexdrive.



## rclone
Utilizaremos rclone para montar Google Drive como unidad de disco en nuestro sistemaRecomiendo usar plexdrive para enlazar con Plex y rclone para hacer tareas de copia, subida y demás. Aun así, rclone tiene ahora una nueva opción para cachear el contenido de una unidad que podéis probar.

Descargarmos rclone y lo configuramos ejecutando este script realizado por el propio developer.
```
curl https://rclone.org/install.sh | sudo bash
```

Accedemos a la configuración.
```
rclone config
```
N: creamos una nueva unidad.

Le damos un nombre, por ejemplo: plexcloud.

11: Google Drive (revisa el número, puede que haya cambiado con la versión de rclone)

Dejamos client id y client secret en blanco.

En el siguiente paso le damos a No (N) y nos dará una url que debemos pegar en nuestro navegador (en tu ordenador local), loguearnos con nuestra cuenta de Google Drive que vayamos a utilizar y copiar el token que nos da y pegarlo.

Nos preguntará si esta todo bien y le decimos que sí (Y).

Nos hará falta instalar fuse también, antes de montar la unidad.
```
apt-get install fuse
```

Creamos una carpeta y montamos la unidad en ella.
```
mkdir /home/plexcloud
rclone mount --allow-other --allow-non-empty -v plexcloud: /home/plexcloud &
```

Si todo ha ido bien, listando el directorio deberéis ver vuestro contenido del drive.
```
ls /home/plexcloud
```

Para que se monte la unidad sola cuando reiniciamos el sistema tenemos que editar el crontab.
```
export EDITOR=nano
crontab -e
```

Pegamos estas líneas y guardamos:

```
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
@reboot sleep 30 && rclone mount --allow-other --allow-non-empty -v plexcloud: /home/plexcloud &
```


## Servidor plex
Vamos a instalar el plex media server y configurarlo.

En la sección de Downloads de su web (https://www.plex.tv/es/downloads/), copiamos el enlace de la versión de linux, en mi caso Ubuntu 64 bits. Podéis mirar si hay una versión más moderna disponible o incluso instalar la versión de Plex Pass que siempre es algo más avanzada.

Ponemos a descargar y esperamos a que acabe.
```
wget https://downloads.plex.tv/plex-media-server/1.5.6.3790-4613ce077/plexmediaserver_1.5.6.3790-4613ce077_amd64.deb
```
Instalamos.
```
dpkg -i plexmediaserver_1.5.6.3790-4613ce077_amd64.deb
```

Para acceder por primera vez, como estamos sin entorno gráfico en linux y por tanto no hay navegador, debemos hacer un tunneling por ssh para enlazar nuestro localhost con el VPS.

Aquí haremos dos distinciones dependiendo de si estamos en Windows o Linux/Mac OS.

- Windows

Hay que configurar PuTTy para hacer la conexión por el puerto en concreto, tendría que quedaros como en la siguiente imagen http://imgur.com/a/jceaI

- Linux/Mac OS

```
ssh root@ipvps -L 8888:localhost:32400
```

Ahora accedemos en nuestro navegador a http://localhost:8888/web y plex debería darnos la bienvenida para proceder a su configuración.
Añadimos las bibliotecas que queramos apuntando al contenido del drive, que detecta como una carpeta más.

En principio Plex debería empezar a escanear todo el contenido de los directorios que le hayamos indicado y una vez acabe ya está disponible par utilizarlo en cualquier dispositivo.

A partir de ahora, podéis acceder a vuestro servidor plex mediante http://ipvps:32400



## Optimizaciones para evitar transcoding

En los clientes/dispositivos que vayáis a utilizar hay que tener en cuenta un par de consideraciones respecto a la calidad de vídeo y los subtítulos para evitar la transcodificación.

Os dejo los parámetros a ajustar en estas imágenes http://imgur.com/a/HoWyR.



## plexpy
Podemos instalar el programa de monitorzación plexpy en nuestro vps para controlar lo que pasa en que cada momento en nuestro servidor y habilitar notificaciones personalizadas de cualquier evento que ocurra.


Instalamos git, clonamos el repositorio de plexpy y lo arrancamos.
```
sudo apt-get install git-core
cd /opt
sudo git clone https://github.com/JonnyWong16/plexpy.git
cd plexpy
python PlexPy.py &
```

Ahora vamos crearlo como servicio y hacer que arranque por defecto con el sistema.
```
cd /etc/systemd/system/
sudo nano plexpy.service
```

Copiamos lo siguiente y lo pegamos en el nuevo archivo.
```
[Unit]
Description=PlexPy - Stats for Plex Media Server usage

[Service]
ExecStart=/opt/plexpy/PlexPy.py --quiet --daemon --nolaunch --config /opt/plexpy/config.ini --datadir /opt/plexpy
GuessMainPID=no
Type=forking
User=root
Group=root

[Install]
WantedBy=multi-user.target
``` 
Guardamos con Ctrl + X y Yes (Y).

Activamos el servicio y arrancamos.
```
sudo systemctl enable plexpy
sudo service plexpy start
```

Ya podremos acceder a la interfaz de plexpy y configurarlo a traves de http://ipvps:8181



## transmission
Instalamos transmission para poder descargar torrents mediante el VPS sin tener ocupada nuestro ancho de banda.

Instalación y configuración.
```
sudo apt-get install transmission-daemon
sudo service transmission-daemon stop
sudo nano /var/lib/transmission-daemon/.config/transmission-daemon/settings.json
```

- Modificar user y pass ( si quieres, por defecto es transmission / transmission ) y quitamos la whitelist.
- Habilitamos una carpeta para que se descarguen los torrents temporalmente, y luego se copien a otra cuando ya estén completados.
```
"rpc-password": "{beb0a38055a0b658f15823f45bb3473696f690326Jhy/s2r", 
"rpc-username": "transmission", 
"rpc-whitelist-enabled": false, 
"incomplete-dir-enabled": true, 
"incomplete-dir": "/var/lib/transmission-daemon/incomplete",
```

Guardamos con Ctrl+X -> Y.

Creamos el directorio con y le damos el propietario a user de transmission:.
```
mkdir /var/lib/transmission-daemon/incomplete
chown debian-transmission:debian-transmission /var/lib/transmission-daemon/incomplete
```

Por defecto, el directorio de descargas está en '/var/lib/transmission-daemon/incomplete' pero podéis modificarlo también.

Arrancamos de nuevo el servicio
```
service transmission-daemon start
```

Accedemos a la interfaz web a través de http://ipvps:9091.



## Uso de rclone
Podemos darle otros usos interesantes a rclone a parte de para montar la unidad de Google Drive.

### Mover automáticamente los archivos descargados de transmission a una carpeta del drive.

Lo haremos con un script muy sencillito que os he dejado en el respositorio y podéis modificar a vuestro gusto para cambiar las rutas de las carpetas.

Descargamos el script en nuestro /home/, por ejemplo, y lo añadimos al crontab para que se ejecute cada 15min.

```
wget https://raw.githubusercontent.com/titelas/plexvps/master/rclonemv.sh
chmod 755 rclonemv.sh
crontab -e
```

Añadimos la línea después de lo que ya incluimos arriba. Tiene que quedar así:
```
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
@reboot sleep 30 && rclone mount --allow-other --allow-non-empty -v plexcloud: /home/plexcloud &
*/15 * * * * /home/rclonemv.sh
```


### Copiarte archivos de una carpeta compartida de Google Drive.

Primero debes añadirte esa carpeta a tu unidad de drive, desde la web.

Ahora lo que haremos será duplicar la unidad con rclone para simular la transferencia de cuenta a cuenta de drive. Para ello, abrimos la configuración.

```
rclone config
```
C: copiamos una unidad.

Seleccionamos el número de la unidad que queramos duplicar y le damos un nombre distinto, por ejemplo: plexcloud2.


Para mover pasarnos archivos de una carpeta a otra lo hacemos con el siguiente comando.
```
rclone copy -v -u --stats 30s --transfers 10 plexcloud2:ruta/hasta/los/archivos plexcloud:donde/lo/queramos/copiar
```

Respecto al parámetro --transfers, indica el número de transferencias simultáneas que podéis realizar a la vez. Deberíais alcanzar mínimo los 30MB/s sin mayor problema, podéis ir jugando con ese valor (10) para maximizar la velocidad. Otra idea es abrir otra sesión de ssh y tener 2 procesos a la vez copiando datos.

## Contacto
- Estoy en telegram: http://t.me/titelas
- [Invítame a un café](https://paypal.me/titelas)

## Enlacés de interés
- [XML para no transcoding](https://forums.plex.tv/discussion/260803/unnecessary-transcoding-of-h264)
- [Instalación plexpy](https://www.htpcbeginner.com/install-plexpy-on-ubuntu/)
- [rclone a ~300MB/s con google cloud platform](https://docs.google.com/document/d/17sOynlIKO5cgdzir4xmxzKHLGglKGGtT4zNsBklvSnQ/edit)


