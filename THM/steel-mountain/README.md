# Steel Mountain - TryHackMe

**Fecha de resolución:** 28 de agosto de 2026

Máquina Windows, IP 10.130.134.238. Temática Mr. Robot.

## Reconocimiento

```
nmap --privileged -n -sS -Pn -vvv -p80,8080,135,139,445,5985,49152,49153,49155,49156 -sV -oN scan.txt 10.130.134.238
```

Puerto 8080 corriendo HttpFileServer (HFS) httpd 2.3, además de IIS 8.5 en el 80, SMB y WinRM.

## Explotación

HFS 2.3.x es vulnerable a RCE remota (CVE-2014-6287) a través del parámetro `search` con un null byte, que permite ejecutar comandos del sistema. Usé el exploit público (`exploit.py`, adaptado a `exploit2.py` para Python 3) para forzar al servidor a descargar `nc.exe` desde mi servidor HTTP y abrir una conexión de vuelta a mi listener en el puerto 443, obteniendo shell como el usuario del servicio.

## Escalada de privilegios

Subí `winPEASx64.exe` y `PowerUp.ps1` para enumerar el sistema. PowerUp señala un unquoted service path en el servicio `AdvancedSystemCareService9`, cuyo binario vive en `C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe`, dentro de una carpeta con permisos de escritura para el usuario actual.

Coloqué mi propio binario como `Advanced.exe` en esa ruta, detuve y reinicié el servicio, y al arrancar Windows interpreta la ruta sin comillas y ejecuta mi binario en lugar del original, corriendo con privilegios de NT AUTHORITY\SYSTEM. Shell de SYSTEM obtenida.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
