# Dead Drop - TryHackMe

**Fecha de resolución:** 26 de agosto de 2026

Escenario con dos máquinas: una app web Linux (DeadDrop) y un controlador de dominio Windows (DEADDROP-DC), dominio `deaddrop.loc`.

## Reconocimiento

Contra la primera máquina (192.168.11.200):

```
nmap -sS -n -vv -p22,80 -sV -sC -oN scan.txt 192.168.11.200
```

SSH y un puerto 80 corriendo una app Node/Express, "DeadDrop - Login".

Contra el controlador de dominio (192.168.11.100):

```
nmap -Pn -sT -sV -sC -n -p 53,88,135,139,445,389,636,3268,3269,3389,5985,5986 -oN nmap-100.txt 192.168.11.100
```

Kerberos, LDAP y SMB propios de un DC, dominio `deaddrop.loc`.

## Explotación de la app web

La app DeadDrop, descompilada también en su versión móvil (`deaddrop-mobile.apk`), guarda credenciales por defecto en el código (`config.java`):

```
DEFAULT_USERNAME: j.harris
DEFAULT_PASSWORD: DropsOfJupiter2026!
```

El formulario de login además es vulnerable a un bypass de autenticación por inyección. Con eso conseguí acceso a la aplicación y, revisando su base de datos SQLite (`deaddrop.db`, consultada con `test.js` usando `better-sqlite3`), extraje una tabla de usuarios con contraseñas de `admin` y `svc-backup` en texto claro, además de un hash de `svc-drop` en `hash.txt`.

## Ejecución remota

Subí `shell.js` (reverse shell para Node) explotando una función de subida/carga de la propia app, y obtuve ejecución en el servidor Linux.

## Del servidor Linux al dominio

Con las credenciales de `j.harris` validadas contra el dominio, tenía ya un punto de apoyo válido en Active Directory. Contra el DC:

```
crackmapexec smb 192.168.11.100 -u j.harris -p 'DropsOfJupiter2026!'
```

confirma el acceso (Pwn3d!) y permite volcar el SAM/NTDS (`ntds-sam.txt`), incluyendo el hash de Administrator.

## Escalada a Domain Admin

Enumerando el dominio con BloodHound se ve que `j.harris` tiene privilegio `AddMember` sobre el grupo `ITSupport-Admins`, y ese grupo está anidado dentro de Domain Admins. Añadiendo a `j.harris` a `ITSupport-Admins`:

```
net rpc group addmem "ITSupport-Admins" "j.harris" -U deaddrop.loc/j.harris%'DropsOfJupiter2026!' -S 192.168.11.100
```

la cuenta hereda privilegios de Domain Admin, cerrando la máquina con control total del dominio.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
