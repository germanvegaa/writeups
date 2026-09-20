# Retro - TryHackMe

**Fecha de resolución:** 5 de septiembre de 2026

Máquina Windows Server 2016, IP 10.129.183.180, hostname RetroWeb.

## Reconocimiento

```
nmap --privileged -n -vvv -sS -Pn -p80,3389 -sV -sC -oN scan.txt 10.129.183.180
```

IIS 10.0 en el puerto 80 y RDP en el 3389. El sistema es Windows Server 2016 (build 14393).

## Explotación web

El sitio en el 80 redirige a un WordPress. Enumerando usuarios y con un ataque de fuerza bruta ligero se obtienen credenciales del usuario `Wade`. Con acceso al panel de WordPress, edité la plantilla del tema (archivo 404.php) para inyectar una reverse shell PHP, y al visitar una ruta que dispara ese template obtuve ejecución de código en el sistema.

## Escalada de privilegios

Build 14393 de Windows Server 2016 es vulnerable a CVE-2017-0213, un fallo de elevación de privilegios en el manejo de objetos COM. Descargué el exploit compilado (`CVE-2017-0213_x64.exe`), lo serví por HTTP y lo ejecuté desde el contexto de la shell obtenida. El exploit abre una consola con privilegios de SYSTEM.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
