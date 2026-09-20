# Skynet - TryHackMe

**Fecha de resolución:** 31 de agosto de 2026

Máquina Linux, IP 10.129.153.40, hostname SKYNET. Temática Terminator.

## Reconocimiento

```
nmap --privileged -sS -n -p22,80,110,139,143,445 -sV -sC -oN scan.txt 10.129.153.40
```

Puertos abiertos: 22 (SSH), 80 (Apache 2.4.18), 110/143 (Dovecot pop3/imap) y 445 (Samba 4.3.11). El SMB permite login anónimo.

## Enumeración SMB

Con acceso de invitado a los shares de Samba aparece correo interno de un empleado, Miles Dyson. Uno de los mensajes contiene una contraseña de SMB reseteada:

```
Password: )s{A&2Z=F^n_E.B`
```

y otro apunta a una ruta oculta de administración de un CMS instalado en el sitio (`important.txt` menciona "Add features to beta CMS /45kra24zxs28v3yd").

## Explotación web

Esa ruta corresponde a una instalación de Cuppa CMS vulnerable a LFI/RFI a través de `alertConfigField.php`. Confirmé la lectura de archivos locales:

```
/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../etc/passwd
```

Con el mismo parámetro apuntando a mi servidor HTTP, subí `shell.php` (reverse shell de pentestmonkey) y obtuve ejecución como `www-data`.

## Post-explotación y contraseña de milesdyson

Con shell en la máquina, generé un diccionario de variaciones sobre "terminator" (`log1.txt`) para atacar el hash/contraseña de `milesdyson`, el único usuario con home real en el sistema, y conseguí el acceso como ese usuario.

## Escalada a root

En `/home/milesdyson/backups/` hay un script `backup.sh` que un cron de root ejecuta periódicamente para comprimir el contenido de `/var/www/html` con `tar` usando un wildcard. Es vulnerable a wildcard injection: creando en ese directorio archivos con nombres tipo `--checkpoint=1` y `--checkpoint-action=exec=sh privesc.sh`, tar los interpreta como argumentos propios en lugar de nombres de archivo, y ejecuta el comando que le paso con los privilegios del cron (root). Con eso conseguí una shell como root y la flag final.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
