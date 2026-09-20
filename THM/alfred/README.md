# Alfred - TryHackMe

**Fecha de resolución:** 28 de agosto de 2026

Máquina Windows con un Jenkins expuesto. Temática Batman.

## Reconocimiento y acceso inicial

Jenkins corriendo en el puerto por defecto, accesible con credenciales por defecto (admin/admin). Jenkins trae de serie una Script Console que permite ejecutar Groovy arbitrario sobre el sistema donde corre.

Desde la Script Console ejecuté un script Groovy que descarga y lanza un ejecutable, usando `shell.exe` (msfvenom, payload meterpreter) servido por mi propio servidor HTTP, para obtener una sesión de meterpreter.

También dejé preparado `Invoke-PowerShellTcp.ps1` (Nishang) como alternativa de shell reversa vía PowerShell si el primer método fallaba.

## Escalada de privilegios

Con meterpreter, `getprivs` muestra que el usuario del servicio de Jenkins tiene `SeImpersonatePrivilege` habilitado. Usando el módulo `incognito` de Metasploit:

```
load incognito
list_tokens -u
impersonate_token "NT AUTHORITY\SYSTEM"
```

Tras impersonar el token, migré el proceso de meterpreter a uno que ya corriera con esos privilegios (el token de impersonación por sí solo no basta si el proceso actual no lo soporta), y confirmé acceso completo como SYSTEM.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
