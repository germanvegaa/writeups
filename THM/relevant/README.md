# Relevant - TryHackMe

**Fecha de resolución:** 1 de septiembre de 2026

Máquina Windows, IP 10.130.139.241, hostname RELEVANT.

## Reconocimiento

```
nmap --privileged -n -sS -sV -p 80,135,3389,445,139 -sC -oN scan.txt 10.130.139.241
```

IIS 10.0 en el puerto 80, SMB (139/445) con firma de mensajes deshabilitada y sesión null/guest permitida, y RDP en 3389.

## Enumeración SMB

Con acceso de invitado al share SMB apareció un archivo con credenciales codificadas en base64 (`passwords.txt`), correspondientes a dos usuarios del sistema:

```
Bob - !P@$$W0rD!123
Bill - Juw4nnaM4n420696969!$$$
```

## Explotación

El mismo share SMB es también la raíz web de IIS (accesible en lectura/escritura), así que subí directamente varias webshells ASPX (`shell.aspx`, `shellfinal.aspx`, iterando puertos y payloads hasta dar con uno que no fuera bloqueado por el AV) y las invoqué vía HTTP para obtener ejecución de código como el pool de aplicaciones de IIS.

## Escalada de privilegios

Esa cuenta de servicio tiene el privilegio `SeImpersonatePrivilege` habilitado. Subí `PrintSpoofer64.exe` al mismo share y lo ejecuté:

```
PrintSpoofer64.exe -i -c powershell.exe
```

Abusando del servicio de cola de impresión y de la impersonación de named pipes, PrintSpoofer consigue un token de NT AUTHORITY\SYSTEM y me deja una PowerShell con esos privilegios.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
