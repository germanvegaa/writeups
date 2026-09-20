# Include - TryHackMe

**Fecha de resolución:** 12 de septiembre de 2026

Máquina Linux, IP 10.129.178.85, hostname `mail.filepath.lab`.

## Reconocimiento

```
nmap --privileged -sS -n -vvv -sV -p22,25,110,143,993,995,4000,5000 -oN scan.txt 10.129.178.85
```

Un servidor de correo completo (Postfix, Dovecot pop3/imap) junto a una aplicación Node.js/Express en el puerto 4000. El propio hostname, `mail.filepath.lab`, ya apunta a que la vulnerabilidad pasa por manejo de rutas de archivo.

## Explotación

La aplicación del puerto 4000 es vulnerable a inclusión de archivo local. Aprovechando el servidor de correo del mismo host como vector de log/mail poisoning, inyecté código PHP en un campo que terminara reflejado en un log accesible (o directamente en un correo), y usé el LFI de la app para incluir ese archivo y ejecutar el código inyectado.

## Post-explotación

Con ejecución de código en el sistema, entre los archivos locales apareció un fichero de configuración interno con credenciales de dos aplicaciones distintas del entorno:

```
ReviewAppUsername: admin
ReviewAppPassword: admin@!!!
SysMonAppUsername: administrator
SysMonAppPassword: S$9$qk6d#**LQU
```

que sirvieron para acceder a los paneles de administración de esas dos aplicaciones internas.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
