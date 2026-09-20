# HackPark - TryHackMe

**Fecha de resolución:** 29 de agosto de 2026

Máquina Windows, IP 10.129.164.86. El sitio web es un blog de un parque de atracciones.

## Reconocimiento

```
nmap --privileged -sS -n -Pn -p 80,3389 -sV -sC -oN scan.txt 10.129.164.86
```

Puerto 80 con IIS 8.5 corriendo BlogEngine.NET, y 3389 (RDP) abierto.

## Fuerza bruta del panel admin

El panel de administración del blog acepta login por formulario. Con hydra contra el usuario `admin` (visible como autor de los posts) se consigue la contraseña por fuerza bruta.

## Explotación

BlogEngine.NET en versión <= 3.3.6 es vulnerable a directory traversal con RCE (CVE-2019-6714). Editando un post desde el panel se puede subir un archivo con el editor de contenido; subiendo `PostView.ascx` (payload de `46353.cs` con mi IP y puerto de listener) a `/App_Data/files/` y luego forzando su carga vía el parámetro `theme` con path traversal, el servidor ejecuta el código y conecta de vuelta a mi netcat, dándome shell como el pool de aplicaciones de IIS.

## Escalada de privilegios

Con `winPEAS.bat` localicé el servicio SystemScheduler, que ejecuta `Message.exe` cada 30 segundos desde una carpeta con permisos de escritura para Everyone. Sustituí ese binario por mi propio `Message.exe` (un reverse shell) y esperé al siguiente ciclo del scheduler: al ejecutarse como administrador, la shell que recibo ya tiene privilegios elevados.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
