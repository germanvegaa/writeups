# Windows PrivEsc Arena - TryHackMe

**Fecha de resolución:** 21 de agosto de 2026

Sala orientada exclusivamente a técnicas de escalada de privilegios en Windows. Máquina con dominio `privesc`, hostname PRIVESC, acceso inicial ya facilitado con las credenciales de un usuario estándar.

## Reconocimiento

```
nmap -Pn -sV -sC <target>
```

Puertos abiertos: 135 (msrpc), 139/445 (SMB), 3389 (RDP) y 5985 (WinRM). El propio nmap identifica el hostname vía RDP:

```
Target_Name: PRIVESC
NetBIOS_Computer_Name: PRIVESC
DNS_Domain_Name: privesc
```

## Acceso inicial

Credenciales por defecto de la sala, entregadas en el enunciado:

```
Usuario: thmuser
Contraseña: Password1!
```

Conexión por WinRM/RDP con esas credenciales para tener una sesión de usuario estándar sobre la que trabajar la escalada.

## Escalada de privilegios

La sala plantea varios vectores de privesc clásicos de Windows para practicar uno por uno (contraseñas guardadas, tareas programadas, AlwaysInstallElevated, servicios mal configurados). El que se llevó hasta el final fue el de servicio con permisos débiles sobre su binario: localicé un servicio cuyo ejecutable era escribible por el usuario actual, lo sustituí por un binario propio (`svc.exe`) y reinicié el servicio para que se ejecutara con privilegios de SYSTEM. Confirmado el acceso con la shell recibida (`shell.exe`) apuntando de vuelta a mi listener.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
