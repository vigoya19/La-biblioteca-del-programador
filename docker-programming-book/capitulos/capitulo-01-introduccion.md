# Capítulo 1: Introducción a Docker y los Contenedores

> *"Cualquier tecnología suficientemente avanzada es indistinguible de la magia."*  
> — Arthur C. Clarke

---

Bienvenido al primer capítulo de este viaje. Si alguna vez has escuchado frases como *"en mi máquina funciona"* o has pasado horas configurando entornos de desarrollo que luego fallan en producción, estás en el lugar correcto. Docker no es solo una herramienta: es un cambio de paradigma en cómo empaquetamos, distribuimos y ejecutamos software.

En este capítulo vamos a desmontar Docker pieza por pieza. No nos quedaremos en la superficie con comandos bonitos; bajaremos al kernel de Linux para entender exactamente qué sucede cuando ejecutas `docker run`. Al terminar, sabrás más sobre el funcionamiento interno de los contenedores que el 95% de los usuarios de Docker.

Empecemos.

---

## 1. ¿Qué es un contenedor?

### 1.1 El problema clásico: "en mi máquina funciona"

Imagina esta escena. Eres desarrollador. Llevas dos semanas escribiendo una aplicación en Python. En tu portátil todo va perfecto. Haces push, abres un Pull Request, y el pipeline de CI falla estrepitosamente. El mensaje de error es críptico. Tras investigar, descubres que el servidor de CI tiene Python 3.8 y tu código usa características de Python 3.11. O peor: la versión de una librería del sistema (digamos, `libssl`) es diferente y tu aplicación se niega a arrancar.

Este problema tiene nombre: **"works on my machine"** o, en español, **"en mi máquina funciona"**.

La causa raíz es simple pero profunda: **el software no vive aislado**. Cada aplicación depende de:

- Una versión específica del sistema operativo.
- Librerías del sistema (`glibc`, `openssl`, `ffmpeg`).
- El runtime del lenguaje (Python 3.11 vs 3.8, Node 20 vs 18).
- Variables de entorno, archivos de configuración, permisos.
- Otros procesos que puedan interferir (puertos ocupados, locks en archivos).

Tradicionalmente, había dos soluciones a este problema. Ninguna ideal.

**Solución 1: Máquinas Virtuales (VMs).** Empaquetas tu aplicación junto con un sistema operativo completo dentro de una VM. Funciona, pero es pesado: gigabytes de disco, minutos de arranque, overhead de CPU y RAM significativo.

**Solución 2: Entornos estandarizados.** "Todos usamos Ubuntu 22.04". Funciona hasta que alguien necesita una versión distinta de algo. Además, no escala cuando tienes 20 microservicios, cada uno con requisitos diferentes.

Los contenedores ofrecen una tercera vía. La metáfora es poderosa: en lugar de enviar tu aplicación por barco en un contenedor genérico donde puede golpearse con otras mercancías, la metes en un **contenedor estandarizado** que la aísla completamente de lo que hay alrededor. El contenedor es idéntico en tu máquina, en el CI y en producción.

### 1.2 Contenedores vs. Máquinas Virtuales

Antes de seguir, necesitamos un modelo mental claro de la diferencia. Observa este diagrama:

```
┌──────────────────────────────────┐  ┌──────────────────────────────────┐
│       MÁQUINA VIRTUAL            │  │         CONTENEDOR               │
│                                  │  │                                  │
│  ┌────────────────────────────┐  │  │  ┌────────────────────────────┐  │
│  │     App A    │   App B     │  │  │  │   App A  │   App B        │  │
│  │───────────────────────────│  │  │  │───────────────────────────│  │
│  │   Bins/Libs  │ Bins/Libs  │  │  │  │ Bins/Libs │ Bins/Libs     │  │
│  │───────────────────────────│  │  │  │───────────────────────────│  │
│  │   Guest OS   │ Guest OS   │  │  │  │     Container Engine       │  │
│  │───────────────────────────│  │  │  │───────────────────────────│  │
│  │        Hypervisor          │  │  │  │       Host OS Kernel       │  │
│  │───────────────────────────│  │  │  │───────────────────────────│  │
│  │       Host OS Kernel       │  │  │  │        Hardware             │  │
│  │───────────────────────────│  │  │  └────────────────────────────┘  │
│  │        Hardware             │  │                                  │
│  └────────────────────────────┘  │                                  │
└──────────────────────────────────┘  └──────────────────────────────────┘
```

La diferencia fundamental está en la respuesta a una pregunta: **¿cuántos kernels de sistema operativo están ejecutándose?**

- **VM**: Cada máquina virtual tiene su propio kernel completo. El hypervisor (VMware, KVM, VirtualBox) emula hardware virtual y cada VM arranca su SO desde cero. Esto implica que si tienes 10 VMs Ubuntu, tienes 10 kernels de Linux independientes en memoria. Cada uno gestionando su propia tabla de procesos, su propio scheduler, su propio sistema de archivos. Multiplica eso por los recursos que consume cada kernel.

- **Contenedor**: Todos los contenedores comparten **el mismo kernel** del sistema operativo host. No hay hypervisor. No hay kernel invitado. El aislamiento se consigue mediante mecanismos del kernel de Linux (namespaces, cgroups) que particionan la *visibilidad* de los recursos, no los recursos mismos. Los procesos dentro del contenedor son procesos normales desde la perspectiva del host, solo que con una "visión recortada" del sistema.

### 1.3 Tabla comparativa

| Característica       | Máquina Virtual                    | Contenedor                          |
|----------------------|------------------------------------|-------------------------------------|
| **Aislamiento**      | Completo (kernel independiente)    | A nivel de proceso (namespaces)     |
| **Rendimiento**      | Cercano al nativo (con paravirtualización) | Nativo (sin capa de traducción) |
| **Tamaño típico**    | GB (incluye SO completo)           | MB (solo app + dependencias)        |
| **Arranque**         | 30s - 2min (arranque de SO)        | < 1s (arranque de proceso)          |
| **Overhead CPU**     | 5-15% (hypervisor + kernel extra)  | < 1% (solo namespaces/cgroups)      |
| **Overhead RAM**     | Alto (kernel + SO por VM)          | Bajo (comparte kernel)              |
| **Densidad**         | Docenas por host                   | Cientos/miles por host              |
| **Seguridad**        | Fuerte (VM escape es muy difícil)  | Buena (con configuración adecuada)  |
| **Portabilidad**     | Depende del hypervisor             | Estándar OCI, funciona en cualquier runtime |
| **Sistema operativo**| Cualquiera (Linux, Windows, BSD)   | Mismo kernel que el host (Linux)    |

Una aclaración importante sobre la última fila: no puedes ejecutar un contenedor Windows en un host Linux (el kernel es distinto). Docker Desktop en macOS y Windows resuelve esto ejecutando una VM ligera con Linux donde corren los contenedores. Pero el contenedor en sí siempre usa el kernel del host.

### 1.4 "Un contenedor NO es una VM"

Esta frase es el mantra más repetido en el ecosistema Docker. Interiorízala. Un contenedor es **un grupo de procesos aislados con restricciones de recursos**. Punto. No hay magia. No hay virtualización de hardware. No hay un sistema operativo invitado.

Cuando ejecutas `docker run nginx`, esto es lo que realmente sucede:

1. Se descarga la imagen si no está en local.
2. Se crea un sistema de archivos aislado a partir de las capas de la imagen.
3. Se crean nuevos namespaces para el contenedor.
4. Se aplican límites de cgroups.
5. Se ejecuta `nginx` como proceso hijo de `containerd-shim`.

El proceso `nginx` aparece en la tabla de procesos del host. Puedes verlo con `ps aux | grep nginx`. Intenta hacer eso dentro de una VM tradicional: el proceso es invisible desde el host. Esta diferencia aparentemente sutil tiene implicaciones profundas en rendimiento, depuración y seguridad que exploraremos a lo largo de todo el capítulo.

### 1.5 Benchmarking práctico: VM vs Contenedor

Hablemos con números, no con palabras. Vamos a comparar el arranque de una VM y un contenedor en la misma máquina:

```bash
# ─── ARRANQUE DE VM (VirtualBox + Ubuntu 22.04) ───
$ time VBoxManage startvm ubuntu-server --type headless
# Espera hasta que SSH esté disponible...
Starting VM...
real    1m 32.450s    # Más de 90 segundos para estar operativa

# ─── ARRANQUE DE CONTENEDOR (Docker + Ubuntu 22.04) ───
$ time docker run -d --rm ubuntu:22.04 sleep 60
7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d
real    0m 0.247s     # Menos de un cuarto de segundo

# ─── CONSUMO DE RAM EN REPOSO ───
# VM Ubuntu Server recién arrancada (sin apps):
$ VBoxManage metrics query ubuntu-server Guest/RAM/Usage
RAM Usage: 245 MB

# Contenedor Ubuntu recién arrancado:
$ docker stats --no-stream ubuntu-test
CONTAINER ID   NAME          CPU %   MEM USAGE / LIMIT    MEM %   NET I/O
abc123def456   ubuntu-test   0.00%   412KiB / 15.55GiB    0.00%   0B / 0B
# Solo 412 KiB (0.4 MB) vs 245 MB de la VM

# ─── DENSIDAD ───
# ¿Cuántas instancias de Nginx puedes correr en un portátil de 16GB?
# VMs: cada VM necesita ~512MB mínimo para el SO base → unas 25 VMs máx.
# Contenedores: ~10-20MB extra por contenedor → literalmente miles.
```

Esta diferencia de órdenes de magnitud no es magia, es arquitectura. Y la arquitectura se basa en los mecanismos del kernel que vamos a estudiar a continuación.

---

## 2. Namespaces: cómo el kernel crea "cajas aisladas"

Imagina que vives en un edificio de apartamentos. Dentro de tu apartamento ves tus muebles, tu tele, tu cocina. No ves lo que hay en el apartamento de tu vecino. Pero todos comparten la estructura del edificio: las tuberías, los cables, el ascensor.

Los **namespaces** son exactamente eso para procesos en Linux:

| El edificio... | Es en Linux... |
|---|---|
| El edificio entero | El **kernel** (el host) |
| Cada apartamento | Un **namespace** |
| Tus muebles dentro | Los **procesos** del contenedor |
| Tuberías y ascensor | El kernel **compartido** |

Linux tiene 8 tipos de namespace. Cada uno aísla una cosa distinta. Vamos a ver los dos más importantes para entender Docker.

---

### 2.2 PID namespace: "cada contenedor tiene su propia lista de procesos"

#### La idea en 30 segundos

En Linux, cada proceso tiene un número (el PID). Chrome es el #847, Spotify el #1203... Es como el DNI de cada proceso. **Todos los procesos del sistema comparten la misma lista de números.**

Un PID namespace crea una "habitación aparte". Dentro de esa habitación, los procesos arrancan una lista NUEVA desde el número 1. El proceso que era el #28471 en la lista global ahora es el #1 dentro de su habitación.

**Analogía:** Imagina una oficina gigante con 300 empleados numerados del 1 al 300. Pones paredes y creas despachos pequeños. Dentro de cada despacho, el empleado se renumera: el #274 de la oficina ahora es el #1 de su despacho. Desde dentro solo ves los números de tu despacho. No sabes que fuera eres el #274.

#### Ejemplo visual

```
┌──────────────────────────────────────────┐
│           HOST (tu máquina)              │
│                                          │
│  PID 1   → systemd (el init del host)    │
│  PID 847 → Chrome                        │
│  ...                                     │
│  PID 28471 ───────────────────┐          │
│                               │          │
│  ┌────────────────────────────┘          │
│  │ ┌────────────────────────┐           │
│  │ │   CONTENEDOR alpine    │           │
│  │ │                        │           │
│  │ │  PID 1 → ps aux       │           │
│  │ │                        │           │
│  │ │  (Este proceso es el   │           │
│  │ │   PID 28471 del host,  │           │
│  │ │   pero él cree que es  │           │
│  │ │   el PID 1.)           │           │
│  │ └────────────────────────┘           │
│                                          │
│  El contenedor NO SABE que existen       │
│  Chrome, systemd, ni los otros 300       │
│  procesos. Solo se ve a sí mismo.        │
└──────────────────────────────────────────┘
```

#### Pruébalo tú mismo (30 segundos)

```bash
# ¿Cuántos procesos hay en tu máquina?
$ ps aux | wc -l
342       # Tu ordenador tiene 342 procesos

# Ahora dentro de un contenedor Alpine:
$ docker run --rm alpine ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 ps aux
# Solo ve 1 proceso. Y se cree el número 1.
```

El contenedor vive en una burbuja: cree que es el único proceso del mundo.

#### ¿Por qué le importa esto a Docker?

1. **El contenedor no puede tocar los procesos del host.** Si dentro del contenedor haces `kill`, solo matas procesos del contenedor.
2. **Cada contenedor tiene su "jefe" (PID 1).** Cuando Docker quiere apagar el contenedor, le manda una señal al PID 1 y este cierra todo ordenadamente.
3. **50 contenedores = 50 "PID 1".** Ninguno interfiere con los demás.

---

> **📚 Avanzado (sáltate esto si estás empezando):** El PID 1 en Linux tiene responsabilidades especiales: recoger procesos huérfanos (zombis) y reenviar señales. Si tu app no fue diseñada para ser PID 1, usa `docker run --init` y Docker meterá un mini-gestor (`tini`) que se encarga. Si algún día ves procesos con estado `Z` (defunct) en `docker top`, vuelve aquí.

---

### 2.3 NET namespace: "cada contenedor tiene su propia red privada"

#### La idea en 30 segundos

Tu ordenador tiene una tarjeta de red, una IP, y 65535 puertos. Si dos programas intentan usar el puerto 80, el segundo casca.

Un NET namespace le da a cada contenedor **su propia red privada virtual** completa: su propia IP, sus propios puertos, sus propias reglas. Dos contenedores pueden usar el puerto 80 sin estorbarse porque cada uno está en su "mundo de red" privado.

**Analogía:** Un hotel. El hotel tiene una dirección postal (la IP del host: `192.168.1.10`). Cada habitación tiene su propio teléfono interno. La habitación 101 tiene extensión 101, la 102 tiene extensión 102. Ambas pueden tener un contestador en la extensión 80 sin conflicto. Para llamar desde fuera, marcas el número del hotel y dices "pásame con la extensión 80 de la habitación 101" (eso es `docker run -p 8080:80`).

#### Ejemplo visual

```
┌─────────────────────────────────────────────────┐
│            HOST (192.168.1.10)                  │
│                                                 │
│  ┌───────────────────┐  ┌───────────────────┐  │
│  │  Contenedor A     │  │  Contenedor B     │  │
│  │                   │  │                   │  │
│  │  IP: 172.17.0.2   │  │  IP: 172.17.0.3   │  │
│  │  Puerto 80: nginx │  │  Puerto 80: Apache │  │
│  │                   │  │                   │  │
│  │  Red privada A    │  │  Red privada B    │  │
│  └───────────────────┘  └───────────────────┘  │
│                                                 │
│  Ambos en puerto 80. ¡Cero conflictos!          │
│                                                 │
│  Desde fuera accedes así:                       │
│  http://192.168.1.10:8080 → Contenedor A :80   │
│  http://192.168.1.10:9090 → Contenedor B :80   │
└─────────────────────────────────────────────────┘
```

#### Pruébalo tú mismo (30 segundos)

```bash
# Terminal 1: lanza un contenedor que escucha en puerto 80
$ docker run --rm -it alpine sh
/ # ip addr | grep inet
    inet 127.0.0.1/8 ...
    inet 172.17.0.2/16 ...    ← IP propia de ESTE contenedor
/ # nc -l -p 80               ← Escuchando en puerto 80
(se queda esperando)

# Terminal 2: otro contenedor, mismo puerto 80, sin conflicto
$ docker run --rm -it alpine sh
/ # ip addr | grep inet
    inet 172.17.0.3/16 ...    ← OTRA IP, distinta
/ # nc -l -p 80               ← ¡También puerto 80 y funciona!
```

Dos contenedores, mismo puerto 80, cero conflictos. Eso es el NET namespace.

#### ¿Por qué le importa esto a Docker?

1. **Sin conflictos de puertos.** 10 Nginx distintos, todos en puerto 80. Cero problemas.
2. **Seguridad.** El contenedor A no puede espiar el tráfico del contenedor B.
3. **Redes a medida.** Puedes crear una red privada donde tu backend y tu base de datos se ven entre sí, pero solo el backend está expuesto a internet.

---

> **📚 Avanzado (sáltate esto si estás empezando):** Por dentro, Docker conecta cada contenedor al host con un "cable virtual" (veth pair) enchufado a un "switch virtual" (bridge `docker0`). El mapeo de puertos (`-p 8080:80`) se hace con reglas de iptables. Si algún día `-p` no funciona tras instalar un firewall como `ufw`, el problema es que las reglas de iptables se pisan. Más adelante, en el capítulo de redes, desmontamos todo esto con detalle.

### 2.4 MNT (Mount) namespace: aislamiento del sistema de archivos

**Qué aísla:** Los puntos de montaje. Cada MNT namespace tiene su propio árbol de montajes. Un montaje realizado en un namespace no es visible en otros.

**Por qué importa:** Es la base de las imágenes de Docker y del sistema de archivos del contenedor. Permite que cada contenedor tenga su propio `/`, sus propios montajes, sin interferir con el host ni con otros contenedores.

**¿Cómo se crea un MNT namespace?**

Cuando un proceso crea un nuevo MNT namespace (vía `clone()` con `CLONE_NEWNS` o `unshare --mount`), recibe una copia exacta de la tabla de montajes del namespace padre. Es decir, **hereda** todos los montajes. Pero a partir de ese momento, cualquier cambio de montaje (mount, umount, remount) dentro del namespace no afecta a otros namespaces.

Esto tiene una implicación sutil pero fundamental: si montas `/proc` en un nuevo namespace de montaje sin antes crear un PID namespace, el nuevo `/proc` reflejará los procesos del PID namespace padre (el host). Por eso, cuando creas un contenedor completamente aislado, necesitas:

1. Crear el PID namespace primero (`--pid --fork`).
2. Crear el MNT namespace (`--mount`).
3. Montar un nuevo `/proc` dentro del namespace combinado.

El orden importa. Y Docker lo hace automáticamente.

**Eventos de propagación de montajes:**

Linux soporta propagación de eventos de montaje entre namespaces mediante los flags `shared`, `slave`, `private` y `unbindable`. Por defecto, Docker usa `private` para todos los montajes del contenedor: los cambios en el host no se propagan al contenedor y viceversa.

```bash
# Ver la propagación de un montaje
$ findmnt -o TARGET,PROPAGATION /var/lib/docker
TARGET                     PROPAGATION
/var/lib/docker/overlay2   private
```

Esto significa que si montas un disco USB en el host después de que un contenedor esté en ejecución, el contenedor no lo verá a menos que uses `--volume` o montes explícitamente.

**Verificación práctica:**

```bash
# En el host
$ mount | wc -l
45

$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       200G   80G  120G  40% /
...

# En un contenedor Alpine
$ docker run --rm -it alpine df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  58.4G     12.3G     43.1G  22% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                   989.0M         0    989.0M   0% /sys/fs/cgroup
/dev/sda1                58.4G     12.3G     43.1G  22% /etc/resolv.conf
...

$ docker run --rm -it alpine mount | wc -l
10
```

El sistema de archivos del contenedor es completamente distinto al del host. El contenedor ve su propio `/` (que es un overlay mount sobre la imagen de Alpine), sus propios montajes de `/proc`, `/sys`, etc.

**Creando un MNT namespace manualmente:**

```bash
# Crear namespace de montaje y montar algo nuevo dentro
$ sudo unshare --mount --fork /bin/bash

# Dentro del namespace: crear un directorio y montar tmpfs
$ mkdir /tmp/mi-montaje
$ mount -t tmpfs tmpfs /tmp/mi-montaje
$ echo "datos privados" > /tmp/mi-montaje/secreto.txt
$ cat /tmp/mi-montaje/secreto.txt
datos privados

# Salir y verificar desde el host
$ exit
$ ls /tmp/mi-montaje/
# (está vacío o no existe)
```

El montaje `tmpfs` que hicimos dentro del namespace no es visible desde fuera. Este es el mismo mecanismo que permite que un contenedor tenga su propio `/tmp`, sus propios volúmenes, su propio sistema de archivos raíz completamente distinto al del host.

### 2.5 UTS namespace: aislamiento de hostname y NIS domain

**Qué aísla:** El hostname (`nodename`) y el NIS domain name del sistema. Cada UTS namespace puede tener su propio hostname independiente.

**Por qué importa:** Permite que cada contenedor tenga su propio hostname, lo que es útil para identificación, logs, y aplicaciones que dependen del hostname para su configuración.

**Verificación práctica:**

```bash
# En el host
$ hostname
mi-servidor-produccion

# En un contenedor (hostname por defecto = ID del contenedor)
$ docker run --rm -it alpine hostname
f7a8b3c2d1e9

# Especificando hostname (Docker crea un UTS namespace)
$ docker run --rm -it --hostname mi-app alpine hostname
mi-app
```

**Creando un UTS namespace manualmente:**

```bash
# Cambiar hostname sin afectar al host
$ sudo unshare --uts --fork /bin/bash

# Dentro del namespace UTS
$ hostname
mi-servidor-produccion   # (hereda el hostname del host)

$ hostname contenedor-1
$ hostname
contenedor-1

# En otra terminal del host: el hostname sigue igual
$ hostname
mi-servidor-produccion
```

UTS significa "UNIX Time-sharing System". Es uno de los namespaces más antiguos y simples. Docker lo usa para asignar a cada contenedor su hostname (por defecto, los primeros 12 caracteres del ID del contenedor).

### 2.6 IPC namespace: aislamiento de comunicación entre procesos

**Qué aísla:** Mecanismos de IPC de System V (semáforos, colas de mensajes, segmentos de memoria compartida) y colas de mensajes POSIX.

**Por qué importa:** Evita que un contenedor acceda a la memoria compartida o a los semáforos de otro contenedor o del host. Esto es tanto una cuestión de seguridad como de corrección: dos aplicaciones en distintos contenedores no deberían colisionar en recursos IPC.

**Verificación práctica:**

```bash
# En el host: listar recursos IPC
$ ipcs

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status

------ Semaphore Arrays --------
key        semid      owner      perms      nsems

# En un contenedor: visión aislada, vacía
$ docker run --rm -it alpine ipcs

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status

------ Semaphore Arrays --------
key        semid      owner      perms      nsems
```

Docker aísla completamente los IPC. El contenedor no ve ningún recurso IPC del host. Puedes compartir el IPC namespace entre contenedores usando `--ipc=container:<id>` o `--ipc=host` (para acceder al IPC del host, con cuidado).

**Creando un IPC namespace manualmente:**

```bash
# Crear un segmento de memoria compartida en el host
$ ipcmk -M 1024
Shared memory id: 32768

# Verificar
$ ipcs -m | grep 32768
0x00000000 32768     andres     644        1024        0

# Dentro de un namespace IPC, no se ve
$ sudo unshare --ipc --fork /bin/bash
$ ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
# (vacío)
```

### 2.7 USER namespace: aislamiento de UID/GID

**Qué aísla:** Los identificadores de usuario y grupo. Dentro de un USER namespace, un proceso puede tener UID 0 (root) dentro del namespace, pero un UID sin privilegios fuera.

**Por qué importa:** Es la característica clave para la seguridad. Permite ejecutar contenedores con un usuario que es root *dentro* del contenedor pero que se mapea a un usuario sin privilegios en el host. Esto significa que incluso si un atacante escapa del contenedor, no obtiene acceso root en el host.

**Verificación práctica:**

```bash
# En el host
$ whoami
andres
$ id -u
1000

# En un contenedor (por defecto, root dentro del contenedor)
$ docker run --rm -it alpine whoami
root
$ docker run --rm -it alpine id -u
0

# Pero en el host, el proceso se ejecuta como root también (sin user namespace)
$ docker run --rm -d alpine sleep 3600
$ ps aux | grep "sleep 3600" | grep -v grep
root     12345  0.0  0.0   1592     4 ?  Ss   10:30   0:00 sleep 3600
```

Por defecto, Docker NO usa USER namespace (a menos que configures `userns-remap` en el daemon). Esto es una decisión deliberada de Docker por simplicidad, pero significa que el root del contenedor es el mismo root del host. Más adelante en el libro cubriremos cómo habilitar user namespaces para entornos de producción con requisitos de seguridad elevados.

**Creando un USER namespace manualmente:**

```bash
# Crear un USER namespace y mapear el usuario actual a root dentro
$ sudo unshare --user --fork /bin/bash

# Dentro: somos root
# whoami
root
# id -u
0

# Pero fuera (en otra terminal):
$ ps aux | grep unshare
root     28175  0.0  0.0   7440  4260 pts/0    S    10:31   0:00 unshare --user...
```

USER namespaces permiten el mapeo flexible de UIDs. Por ejemplo, puedes mapear UID 0 del namespace a UID 1000 del host, y UID 1000 del namespace a UID 2000 del host. Esta característica es la base del modo "rootless" de Docker y Podman, que permite ejecutar contenedores completamente sin privilegios de root en el host.

### 2.8 CGROUP namespace: aislamiento de límites de recursos

**Qué aísla:** La vista del sistema de archivos de cgroups (`/sys/fs/cgroup`). Los procesos dentro de un CGROUP namespace ven sus propios límites de recursos como si fueran los límites del sistema completo.

**Por qué importa:** Sin CGROUP namespace, un proceso en un contenedor podría ver todos los cgroups del sistema. Esto tiene implicaciones de seguridad (fuga de información) y de funcionalidad (una aplicación que intenta leer sus límites de recursos debería ver los límites del contenedor, no los del host).

**Verificación práctica:**

```bash
# En el host: la jerarquía completa de cgroups
$ ls /sys/fs/cgroup/
blkio  cpu  cpu,cpuacct  cpuacct  cpuset  devices  freezer  hugetlb  memory  net_cls  net_prio  perf_event  pids  systemd

# En un contenedor (con CGROUP namespace)
$ docker run --rm -it alpine ls /sys/fs/cgroup/
blkio  cpu  cpu,cpuacct  cpuacct  cpuset  devices  freezer  hugetlb  memory  net_cls  net_prio  perf_event  pids  systemd
```

La salida se ve similar, pero los archivos de límites dentro muestran los límites del contenedor, no los del host. Por ejemplo, si limitaste la memoria del contenedor a 512MB, el archivo `memory.limit_in_bytes` dentro del contenedor mostrará 512MB, no la RAM total del host.

**Creando un CGROUP namespace manualmente:**

```bash
# Crear un CGROUP namespace
$ sudo unshare --cgroup --fork /bin/bash

# Dentro: la vista de cgroups está limitada a nuestro propio cgroup
$ cat /sys/fs/cgroup/memory/memory.limit_in_bytes
9223372036854771712   # (máximo, sin límite)
```

### 2.9 TIME namespace (novedad)

Linux 5.6 introdujo el TIME namespace, que permite aislar los relojes monotónico y de arranque. Esto significa que un contenedor puede tener su propio "tiempo de arranque" (`CLOCK_BOOTTIME` y `CLOCK_MONOTONIC`). Es útil para migración de contenedores y checkpoint/restore. Docker no lo usa por defecto todavía, pero runc y CRI-O están incorporando soporte.

### 2.10 Tabla resumen de namespaces

| Namespace  | Constante de kernel   | Qué aísla                     | Flag de `unshare` | Año introducido |
|------------|-----------------------|-------------------------------|-------------------|-----------------|
| PID        | CLONE_NEWPID          | Process IDs                   | `--pid`           | 2008 (2.6.24)   |
| NET        | CLONE_NEWNET          | Network interfaces, routes    | `--net`           | 2009 (2.6.29)   |
| MNT        | CLONE_NEWNS           | Mount points                  | `--mount`         | 2002 (2.4.19)   |
| UTS        | CLONE_NEWUTS          | Hostname, NIS domain          | `--uts`           | 2006 (2.6.19)   |
| IPC        | CLONE_NEWIPC          | SysV IPC, POSIX message queues| `--ipc`           | 2006 (2.6.19)   |
| USER       | CLONE_NEWUSER         | UID/GID                       | `--user`          | 2013 (3.8)      |
| CGROUP     | CLONE_NEWCGROUP       | Cgroup root directory         | `--cgroup`        | 2016 (4.6)      |
| TIME       | CLONE_NEWTIME         | Boot and monotonic clocks     | `--time`          | 2020 (5.6)      |

### 2.11 Ver todos los namespaces de un contenedor

El comando `lsns` lista todos los namespaces del sistema:

```bash
# Listar namespaces
$ lsns
        NS TYPE   NPROCS   PID USER             COMMAND
4026531835 cgroup     295     1 root             /sbin/init
4026531836 pid        295     1 root             /sbin/init
4026531837 user       295     1 root             /sbin/init
4026531838 uts        295     1 root             /sbin/init
4026531839 ipc        295     1 root             /sbin/init
4026531840 net        294     1 root             /sbin/init
4026531992 mnt          1   384 root             /usr/sbin/haveged
4026531993 mnt          1  2301 systemd-timesync /lib/systemd/systemd-timesyncd
4026532401 mnt          2 28174 root             sleep 3600
4026532402 uts          2 28174 root             sleep 3600
4026532403 ipc          2 28174 root             sleep 3600
4026532404 pid          2 28174 root             sleep 3600
4026532405 net          2 28174 root             sleep 3600
4026532482 cgroup       2 28174 root             sleep 3600
```

Cada contenedor tiene su propia fila de namespaces para cada tipo. También puedes ver los namespaces como archivos en `/proc`:

```bash
$ docker inspect <container_id> | jq '.[0].State.Pid'
28174

$ ls -la /proc/28174/ns/
total 0
lrwxrwxrwx 1 root root 0 May 20 10:34 cgroup -> 'cgroup:[4026532482]'
lrwxrwxrwx 1 root root 0 May 20 10:34 ipc -> 'ipc:[4026532403]'
lrwxrwxrwx 1 root root 0 May 20 10:34 mnt -> 'mnt:[4026532401]'
lrwxrwxrwx 1 root root 0 May 20 10:34 net -> 'net:[4026532405]'
lrwxrwxrwx 1 root root 0 May 20 10:34 pid -> 'pid:[4026532404]'
lrwxrwxrwx 1 root root 0 May 20 10:34 pid_for_children -> 'pid:[4026532404]'
lrwxrwxrwx 1 root root 0 May 20 10:34 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 May 20 10:34 uts -> 'uts:[4026532402]'
```

Los números entre corchetes son los inode numbers que identifican de forma única cada namespace. Dos procesos que comparten el mismo inode para un tipo de namespace están en el mismo namespace de ese tipo. Fíjate en que el `user` namespace es el mismo que el del host (`4026531837`): Docker por defecto no crea USER namespaces.

---

## 3. cgroups (Control Groups)

Si los namespaces responden a la pregunta **"¿qué ven los procesos?"**, los cgroups responden a **"¿cuánto pueden usar?"**. Los namespaces proporcionan aislamiento; los cgroups proporcionan control.

### 3.1 ¿Qué son los cgroups?

Los **cgroups** (control groups) son un mecanismo del kernel de Linux para limitar, contabilizar y aislar el uso de recursos (CPU, memoria, I/O, red) entre grupos de procesos.

Un cgroup es un grupo de procesos vinculados a un conjunto de parámetros que definen límites de recursos. Los cgroups están organizados jerárquicamente: un cgroup hijo hereda los límites del cgroup padre y puede restringirlos aún más.

Cada subsistema de recursos (llamado "controller" en la jerga de cgroups v2) gestiona un tipo de recurso:

- **cpu**: controla el acceso a la CPU.
- **memory**: limita y contabiliza el uso de memoria.
- **blkio**: controla el acceso a dispositivos de bloque (discos).
- **pids**: limita el número de procesos.
- **net_cls/net_prio**: clasifica paquetes de red para QoS.
- **devices**: controla acceso a dispositivos.
- **cpuset**: asigna procesos a CPUs y nodos de memoria específicos.
- **hugetlb**: controla el uso de huge pages.
- **perf_event**: monitorización de rendimiento.

### 3.2 cgroups v1 vs cgroups v2

Linux ha tenido dos generaciones de cgroups:

**cgroups v1:** Múltiples jerarquías separadas, una por cada controller. Puedes montar cada controller en un punto diferente. Los procesos pueden estar en diferentes posiciones en cada jerarquía.

```
/sys/fs/cgroup/
├── blkio/
├── cpu/
├── cpu,cpuacct/   # (combinados)
├── memory/
├── net_cls/
├── ...
```

**cgroups v2:** Una única jerarquía unificada. Todos los controllers se gestionan desde el mismo árbol. Más simple, más coherente.

```
/sys/fs/cgroup/
├── cgroup.controllers
├── cgroup.subtree_control
├── system.slice/
├── user.slice/
├── docker/
│   ├── <container-id>/
```

Docker por defecto usa cgroups v1 en la mayoría de distribuciones (por compatibilidad), pero soporta cgroups v2. Puedes verificar qué versión estás usando:

```bash
$ mount | grep cgroup
# Si ves múltiples lineas con diferentes controllers: cgroups v1
# Si ves una sola línea "cgroup2": cgroups v2
```

### 3.3 Controlando CPU

#### CPU shares (prioridad relativa)

```bash
$ docker run --cpu-shares=512 nginx
```

Las CPU shares definen el peso relativo del contenedor cuando hay contención de CPU. Con valores por defecto de 1024:

- Contenedor A con `--cpu-shares=1024`: 50% de la CPU en contención.
- Contenedor B con `--cpu-shares=1024`: 50% de la CPU en contención.
- Contenedor C con `--cpu-shares=512`: 33% de la CPU si compite con A o B.

Pero si no hay contención (la CPU está ociosa), cualquier contenedor puede usar el 100% de la CPU. Las shares solo importan cuando varios contenedores compiten.

Internamente, esto se traduce al archivo `cpu.shares` en cgroups v1:

```bash
$ cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.shares
512
```

#### CPU quota (límite absoluto)

Para limitar de forma absoluta, Docker ofrece `--cpus`:

```bash
# Limitar a 1.5 CPUs
$ docker run --cpus=1.5 nginx
```

Esto se traduce a dos parámetros de cgroups:

- `cpu.cfs_period_us` (período): normalmente 100000 microsegundos (100ms).
- `cpu.cfs_quota_us` (cuota): microsegundos de CPU por período.

Para `--cpus=1.5`:
- `period_us = 100000`
- `quota_us = 150000` (1.5 * 100000)

Significa que el contenedor puede usar 150ms de CPU cada 100ms, lo que equivale a 1.5 CPUs en promedio.

```bash
# Verificar los valores
$ cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_period_us
100000
$ cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_quota_us
150000
```

También puedes asignar CPUs específicas:

```bash
# Usar solo los núcleos 0 y 3
$ docker run --cpuset-cpus="0,3" nginx
```

Esto se traduce a `cpuset.cpus`. Es útil para reducir interferencia con la caché de CPU en aplicaciones de alto rendimiento.

### 3.4 Controlando memoria

```bash
$ docker run --memory=512m --memory-swap=1g nginx
```

- `--memory=512m`: límite duro de memoria física (512 MB).
- `--memory-swap=1g`: límite de memoria total (física + swap). Si no se especifica, por defecto es el doble de `--memory`.

En cgroups, esto se traduce a:

```bash
$ cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
536870912   # 512 MB

$ cat /sys/fs/cgroup/memory/docker/<container-id>/memory.memsw.limit_in_bytes
1073741824  # 1 GB (swap incluido)
```

**¿Qué pasa cuando se excede el límite de memoria?**

El kernel invoca el **OOM Killer** (Out-Of-Memory Killer) para el cgroup. Mata el proceso que más memoria esté usando dentro del cgroup. El contenedor se detiene con código de salida 137 (128 + 9, donde 9 es `SIGKILL`).

Puedes ajustar la puntuación OOM de procesos dentro del contenedor:

```bash
# Hacer que este proceso sea el último en ser matado
$ docker run --oom-score-adj=-500 nginx
```

Valores del rango -1000 a 1000. Un valor más bajo significa menos probabilidad de ser matado.

También puedes deshabilitar completamente el OOM Killer para un contenedor:

```bash
$ docker run --oom-kill-disable --memory=512m nginx
```

Pero cuidado: si el contenedor excede la memoria, no será matado, simplemente se colgará o el kernel congelará sus procesos. Úsalo con extrema precaución.

### 3.5 Controlando I/O de disco (blkio)

Puedes limitar las operaciones de lectura/escritura a dispositivos de bloque:

```bash
# Limitar a 10 MB/s de lectura en /dev/sda
$ docker run --device-read-bps=/dev/sda:10mb nginx

# Limitar a 100 IOPS de escritura
$ docker run --device-write-iops=/dev/sda:100 nginx

# Limitar peso relativo de I/O (similar a cpu shares)
$ docker run --blkio-weight=500 nginx
```

En cgroups:

```bash
$ cat /sys/fs/cgroup/blkio/docker/<container-id>/blkio.throttle.read_bps_device
8:0 10485760   # /dev/sda (8:0) limitado a 10MB/s
```

### 3.6 Limitando el número de procesos (pids controller)

El controller `pids` limita cuántos procesos puede crear un contenedor. Es una protección contra fork bombs y contención de recursos:

```bash
# Limitar a 50 procesos máximo
$ docker run --pids-limit=50 nginx

# Verificar el límite creado
$ cat /sys/fs/cgroup/pids/docker/<container-id>/pids.max
50

# Intentar crear demasiados procesos dentro del contenedor
$ docker exec <container-id> sh -c 'for i in $(seq 1 100); do sleep 999 & done'
# Cuando el contenedor alcanza 50 procesos:
# sh: can't fork: Resource temporarily unavailable
```

El contador `pids.current` muestra cuántos procesos hay actualmente en el cgroup:

```bash
$ cat /sys/fs/cgroup/pids/docker/<container-id>/pids.current
12
```

### 3.7 Limitando acceso a dispositivos (devices controller)

El controller `devices` restringe qué dispositivos del host son accesibles desde el contenedor:

```bash
# Acceso a un dispositivo específico con permisos limitados
$ docker run --device=/dev/ttyUSB0:/dev/ttyUSB0:rwm --rm -it alpine sh
# r=read, w=write, m=mknod

# Denegar acceso a todos los dispositivos salvo los necesarios
$ docker run --device=/dev/null:/dev/null:rw \
             --device=/dev/zero:/dev/zero:rw \
             --device=/dev/random:/dev/random:r \
             --device=/dev/urandom:/dev/urandom:r \
             --cap-drop=ALL \
             myapp
```

Por defecto, Docker otorga acceso a un conjunto mínimo de dispositivos (`/dev/null`, `/dev/zero`, `/dev/random`, `/dev/urandom`, `/dev/tty`, `/dev/console`, `/dev/pts`). El flag `--privileged` desactiva TODOS los límites de devices y da acceso completo:

```bash
# MUY PELIGROSO: acceso completo a todos los dispositivos del host
$ docker run --privileged -it ubuntu bash
# Desde dentro: puedo montar discos, acceder a /dev/sda, etc.
```

`--privileged` es un antipatrón de seguridad. Casi siempre puedes conseguir lo mismo con `--device` y `--cap-add` específicos.

### 3.8 Cómo ver los cgroups de un contenedor

**Método 1: docker inspect**

```bash
$ docker inspect --format='{{.HostConfig.CpuShares}}' <container-id>
512

$ docker inspect --format='{{.HostConfig.Memory}}' <container-id>
536870912
```

**Método 2: Navegar por /sys/fs/cgroup**

```bash
# Encontrar el cgroup de un contenedor
$ CONTAINER_ID=$(docker ps -q --filter name=nginx)

# cgroups v1
$ ls /sys/fs/cgroup/cpu/docker/$CONTAINER_ID/
cgroup.clone_children  cpu.cfs_period_us  cpu.cfs_quota_us  cpu.shares  cpu.stat  ...

# cgroups v2 (el path puede variar)
$ ls /sys/fs/cgroup/system.slice/docker-$CONTAINER_ID.scope/
```

**Método 3: systemd-cgtop**

```bash
$ systemd-cgtop
Control Group                               Tasks  %CPU   Memory  Input/s Output/s
/                                             295  12.3    7.8G        -        -
/system.slice                                  80   4.2    3.1G        -        -
/system.slice/nginx.service                     2   0.5   45.2M        -        -
/system.slice/docker-abc123.scope               1   8.0  512.0M        -        -
```

**Método 4: docker stats para cada contenedor**

```bash
# Monitoreo continuo de uso de recursos (versión legible)
$ docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}"
NAME                CPU %     MEM USAGE / LIMIT     MEM %     NET I/O
nginx-prod          2.15%     45.2MiB / 512MiB      8.83%     1.2MB / 850kB
redis-cache         0.08%     12.1MiB / 1GiB        1.18%     340kB / 125kB
postgres-db         5.42%     384MiB / 2GiB         18.75%    2.8MB / 1.5MB

# Versión JSON para scripts de monitoreo
$ docker stats --no-stream --format json | jq '.'
{
  "CPUPerc": "2.15%",
  "MemUsage": "45.2MiB",
  "MemPerc": "8.83%",
  ...
}
```

**Método 5: Script para tracking histórico de recursos**

```bash
#!/bin/bash
# Guardar en /usr/local/bin/docker-resource-track.sh
INTERVAL=${1:-5}
CONTAINER=${2}

log_cgroup() {
    local cid=$1
    echo "$(date -Iseconds) | $cid | $(cat /sys/fs/cgroup/memory/docker/$cid/memory.usage_in_bytes 2>/dev/null || echo 0) | $(cat /sys/fs/cgroup/cpu/docker/$cid/cpuacct.usage 2>/dev/null || echo 0)"
}

if [ -n "$CONTAINER" ]; then
    while true; do
        log_cgroup $(docker inspect --format='{{.Id}}' $CONTAINER)
        sleep $INTERVAL
    done
else
    while true; do
        for cid in $(docker ps -q); do
            log_cgroup $cid
        done
        sleep $INTERVAL
    done
fi
```

### 3.10 El problema del "noisy neighbor"

Cuando varios contenedores comparten el mismo host, un contenedor que consume recursos excesivamente puede afectar a los demás. Esto se conoce como el problema del **"vecino ruidoso"** (noisy neighbor).

Sin cgroups, un contenedor podría acaparar toda la CPU o memoria del host, afectando a todos los demás. Con cgroups bien configurados, puedes garantizar isolación de recursos:

```bash
# App web (tráfico de producción): alta prioridad de CPU, memoria garantizada
$ docker run -d --name web \
    --cpus=2 --cpu-shares=2048 \
    --memory=2g --memory-reservation=1500m \
    nginx

# Worker de background: baja prioridad, CPU limitada
$ docker run -d --name worker \
    --cpus=0.5 --cpu-shares=256 \
    --memory=512m \
    my-worker
```

Las `--cpu-shares` (2048 vs 256) significan que, en contención de CPU, `web` recibe 8 veces más CPU que `worker`. `--memory-reservation` establece un "piso" de memoria que Docker intenta garantizar (no es un límite duro, sino un soft limit).

### 3.7 Ejemplo práctico completo: limitar recursos y demostrar

Vamos a crear un contenedor con límites estrictos y demostrar que se aplican usando la herramienta `stress`:

```bash
# Paso 1: Crear contenedor limitado a 512MB RAM y 0.5 CPU
$ docker run -d \
    --name limite-demo \
    --memory=512m \
    --memory-swap=512m \
    --cpus=0.5 \
    alpine sleep 3600

# Paso 2: Verificar límites
$ docker inspect limite-demo --format='Memoria: {{.HostConfig.Memory}} bytes'
Memoria: 536870912 bytes

$ docker inspect limite-demo --format='CPU: {{.HostConfig.NanoCpus}}'
CPU: 500000000    # 0.5 CPUs = 500,000,000 nanocpus

# Paso 3: Instalar stress dentro del contenedor
$ docker exec -it limite-demo sh
/ # apk add --no-cache stress-ng
/ #

# Paso 4: Estresar la CPU (dentro del contenedor)
/ # stress-ng --cpu 2 --timeout 30s &
/ # top
# Verás que la CPU no supera el 50% aunque haya 2 workers

# Paso 5: Estresar la memoria (dentro del contenedor)
/ # stress-ng --vm 1 --vm-bytes 600M --timeout 10s
# El proceso será matado por OOM porque excede los 512MB
# Código de salida: 137 (SIGKILL)
```

Para ver el OOM en acción desde el host:

```bash
# En otra terminal del host
$ dmesg | tail -20
[12345.678] memory: usage 524288kB, limit 524288kB, failcnt 5
[12345.679] Memory cgroup out of memory: Killed process 28912 (stress-ng) ...
```

---

## 4. Union Filesystems (OverlayFS)

Las imágenes de Docker son uno de los conceptos más brillantes de la plataforma. La idea es genialmente simple: **capas de solo lectura apiladas, con una capa de escritura encima**. Para entenderlo, necesitamos entender los sistemas de archivos de unión (union filesystems).

### 4.1 El problema: compartir archivos sin duplicar

Imagina que tienes 10 contenedores ejecutando Ubuntu. Cada uno necesita los archivos base de Ubuntu (~70MB). Sin un mecanismo de compartición, necesitarías 10 * 70MB = 700MB de disco. Con un union filesystem, los 70MB base se almacenan una sola vez y los 10 contenedores los comparten. Cada contenedor solo almacena los cambios que ha hecho sobre la base (normalmente unos pocos KB o MB).

### 4.2 OverlayFS: el estándar de facto

OverlayFS es un sistema de archivos de unión incluido en el kernel de Linux desde la versión 3.18 (2014). Es el storage driver por defecto de Docker en la mayoría de distribuciones.

El modelo conceptual de OverlayFS tiene cuatro componentes:

```
┌──────────────────────────────────────────┐
│              MERGED (vista unificada)     │
│  ┌────────────────────────────────────┐   │
│  │  Esto es lo que "ve" el contenedor │   │
│  └────────────────────────────────────┘   │
│        ↑                                 │
│   ┌────┴────┐                            │
│   │ UPPERDIR │  Capa de escritura         │
│   │ (rw)    │  Los cambios van aquí       │
│   └─────────┘                            │
│        ↑                                 │
│   ┌────┴────┐                            │
│   │ LOWERDIR │ Capas de solo lectura       │
│   │ (ro)    │ Capa 3: tu aplicación      │
│   │         │ Capa 2: dependencias       │
│   │         │ Capa 1: imagen base (Ubuntu)│
│   └─────────┘                            │
│                                           │
│   ┌─────────┐                            │
│   │ WORKDIR │ Directorio de trabajo       │
│   │         │ (interno, no visible)       │
│   └─────────┘                            │
└──────────────────────────────────────────┘
```

- **lowerdir**: Una o más capas de solo lectura. Se montan en orden: la primera es la más "baja" (imagen base), la última es la más "alta" (última capa añadida).
- **upperdir**: La capa de escritura. Todos los cambios (crear, modificar, eliminar archivos) van aquí.
- **merged**: La vista unificada. El sistema de archivos que el contenedor "ve". Combina lowerdir y upperdir.
- **workdir**: Directorio interno que OverlayFS usa para operaciones atómicas. Debe estar en el mismo sistema de archivos que upperdir.

### 4.3 Copy-on-write (CoW)

El mecanismo de **Copy-on-write** es la clave de la eficiencia:

1. **Lectura**: Si un archivo existe en `upperdir`, se lee de ahí. Si no, se busca en `lowerdir` (de la capa más alta a la más baja).
2. **Escritura (archivo nuevo)**: Se crea directamente en `upperdir`.
3. **Escritura (modificar archivo existente)**: El archivo se copia de `lowerdir` a `upperdir` (la primera vez que se modifica), y luego se modifica en `upperdir`. El original en `lowerdir` permanece intacto.
4. **Eliminación**: Se crea un "whiteout file" en `upperdir` (un archivo especial de carácter 0:0) que indica que el archivo ha sido eliminado. El archivo original en `lowerdir` sigue existiendo.

### 4.4 Ejemplo manual: crear un overlay mount

```bash
# Crear las estructuras de directorios
$ mkdir -p /tmp/overlay-demo/{lower,upper,work,merged}

# Crear archivos en el lowerdir
$ echo "Archivo original" > /tmp/overlay-demo/lower/hola.txt
$ echo "Config base" > /tmp/overlay-demo/lower/config.ini
$ mkdir /tmp/overlay-demo/lower/subdir
$ echo "archivo en subdir" > /tmp/overlay-demo/lower/subdir/datos.txt

# Montar el overlay
$ sudo mount -t overlay overlay \
    -o lowerdir=/tmp/overlay-demo/lower,upperdir=/tmp/overlay-demo/upper,workdir=/tmp/overlay-demo/work \
    /tmp/overlay-demo/merged

# Ver la vista unificada
$ ls /tmp/overlay-demo/merged/
hola.txt  config.ini  subdir/

$ cat /tmp/overlay-demo/merged/hola.txt
Archivo original

# Modificar un archivo (dispara CoW)
$ echo "Modificado desde el overlay" > /tmp/overlay-demo/merged/hola.txt

# El archivo original sigue intacto
$ cat /tmp/overlay-demo/lower/hola.txt
Archivo original

# La copia modificada está en upperdir
$ cat /tmp/overlay-demo/upper/hola.txt
Modificado desde el overlay

# Crear un archivo nuevo
$ echo "Nuevo archivo" > /tmp/overlay-demo/merged/nuevo.txt
$ ls /tmp/overlay-demo/upper/
hola.txt  nuevo.txt       # Ambos están en upperdir

# Eliminar un archivo
$ rm /tmp/overlay-demo/merged/config.ini
$ ls /tmp/overlay-demo/lower/config.ini
config.ini                  # Sigue existiendo en lowerdir

$ ls /tmp/overlay-demo/upper/
hola.txt  nuevo.txt  config.ini   # Whiteout file (carácter especial)
$ stat /tmp/overlay-demo/upper/config.ini
# Verás que es un character device (0,0) — un whiteout

# Desmontar
$ sudo umount /tmp/overlay-demo/merged
```

### 4.5 Cómo Docker usa OverlayFS

Cada imagen de Docker está compuesta de múltiples capas. Cada instrucción en un Dockerfile (`FROM`, `RUN`, `COPY`, `ADD`) crea una nueva capa.

```
FROM ubuntu:22.04           → Capa 1 (base Ubuntu, ~70MB)
RUN apt-get update && \     → Capa 2 (índices de paquetes actualizados)
    apt-get install -y python3
COPY app.py .               → Capa 3 (código de la aplicación)
CMD ["python3", "app.py"]   → Capa 4 (metadatos, no ocupa espacio en disco)
```

Cuando ejecutas un contenedor, Docker crea un overlay mount:

```
merged = upperdir (rw, inicialmente vacío) + lowerdir (capas de imagen, ro)
```

El `upperdir` es efímero: cuando eliminas el contenedor, el `upperdir` se borra. Por eso los cambios dentro de un contenedor no persisten a menos que uses volúmenes.

**Dónde se almacenan las capas en el disco:**

```bash
# Ver las capas de las imágenes
$ docker image inspect ubuntu:22.04 | jq '.[0].RootFS.Layers'
[
  "sha256:abc123...",
  "sha256:def456...",
  "sha256:ghi789..."
]

# Las capas están en el storage driver
$ ls /var/lib/docker/overlay2/
abc123.../
def456.../
ghi789.../
l/          # Enlaces simbólicos acortados
```

Cada directorio de capa contiene un directorio `diff/` (el contenido real) y un archivo `link` (nombre corto).

**Capas en ejecución:**

```bash
$ docker container inspect mi-contenedor | jq '.[0].GraphDriver'
{
  "Data": {
    "LowerDir": "/var/lib/docker/overlay2/abc123/diff:/var/lib/docker/overlay2/def456/diff",
    "MergedDir": "/var/lib/docker/overlay2/xyz789/merged",
    "UpperDir": "/var/lib/docker/overlay2/xyz789/diff",
    "WorkDir": "/var/lib/docker/overlay2/xyz789/work"
  },
  "Name": "overlay2"
}
```

Puedes explorar el sistema de archivos del contenedor directamente desde el host:

```bash
$ ls /var/lib/docker/overlay2/xyz789/merged/
bin/  boot/  dev/  etc/  home/  lib/  ...
```

### 4.6 Otros storage drivers

Aunque Overlay2 es el estándar, Docker soporta otros storage drivers. Aquí una comparativa:

| Driver        | Descripción                                                  | Ventajas                        | Desventajas                         |
|---------------|--------------------------------------------------------------|---------------------------------|-------------------------------------|
| **overlay2**  | Union filesystem del kernel. Por defecto.                    | Rápido, kernel nativo, estable  | Solo Linux                          |
| **aufs**      | Union filesystem más antiguo. Patch externo al kernel.       | Muy maduro, primer driver       | No en mainline, obsoleto en Docker  |
| **devicemapper** | Usa thin provisioning y snapshots de LVM.                 | Robusto, snapshots              | Más lento, requiere configuración   |
| **btrfs**     | Sistema de archivos CoW nativo.                              | Snapshots, compresión           | Fragmentación, requiere btrfs       |
| **zfs**       | Sistema de archivos avanzado.                                | Snapshots, compresión, checksums| Licencia CDDL, requiere ZFS         |
| **vfs**       | Sin CoW. Copia completa de la imagen. Solo para testing.     | Simplicidad máxima              | Uso masivo de disco, lentísimo      |
| **fuse-overlayfs** | OverlayFS en userspace (FUSE). Para rootless.           | Rootless                        | Más lento que overlay2 del kernel   |

Para la gran mayoría de casos de uso, Overlay2 es la opción correcta. Los drivers de snapshots (btrfs, zfs) tienen sentido si ya usas esos sistemas de archivos para tus datos.

### 4.7 Cómo Docker distribuye las imágenes

Hemos visto cómo se almacenan las capas localmente, pero ¿cómo se transfieren a través de la red? El proceso es brillantemente eficiente:

```
┌──────────────────────────────────────────────────────────────────┐
│              DISTRIBUCIÓN DE IMÁGENES                            │
│                                                                   │
│  Registry (Docker Hub)                    Host local               │
│  ┌──────────────────────┐               ┌──────────────────────┐  │
│  │ Manifest (JSON)      │── descarga ──→│ docker pull nginx    │  │
│  │ ├─ Config digest     │               │                       │  │
│  │ ├─ Layer 1 digest    │               │ 1. Descarga manifest  │  │
│  │ ├─ Layer 2 digest    │               │ 2. Compara digests    │  │
│  │ └─ Layer 3 digest    │               │    con layers locales │  │
│  │                      │               │ 3. Descarga SOLO las  │  │
│  │ Layer 1 (tar.gz) ────│── descarga ──→│    capas nuevas       │  │
│  │ Layer 2 (tar.gz) ────│── (ya existe, │                       │  │
│  │ Layer 3 (tar.gz) ────│   no descarga)│ 4. Extrae y verifica  │  │
│  └──────────────────────┘               │    checksums SHA256   │  │
│                                          └──────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

Cada capa se identifica por su **content-addressable digest** (SHA256). Si dos imágenes comparten la misma capa base (por ejemplo, ambas usan `ubuntu:22.04`), la capa se descarga una sola vez y se reutiliza. Este es el mecanismo que hace `docker pull` increíblemente rápido después del primer uso.

**Verificación de integridad:**

```bash
# Ver un manifiesto de imagen
$ docker manifest inspect nginx:alpine | jq '.layers[] | .digest'
"sha256:abc123..."
"sha256:def456..."
"sha256:ghi789..."

# Cada capa se verifica contra su digest al descargarse
$ docker pull nginx:alpine
abc123: Pulling from library/nginx
abc123: Already exists      # Digest coincide → no se descarga
def456: Pull complete       # Digest nuevo → se descarga y verifica
```

**Diferencia entre Docker Image Manifest y OCI Image Manifest:**

Docker usa su propio formato de manifiesto (media type `application/vnd.docker.distribution.manifest.v2+json`), mientras que OCI define el estándar (`application/vnd.oci.image.manifest.v1+json`). Son casi idénticos en estructura pero usan diferentes media types para las capas:

```json
/* Manifiesto Docker (v2) */
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "digest": "sha256:config...",
    "size": 1500
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "digest": "sha256:layer1...",
      "size": 72593453
    }
  ]
}

/* Manifiesto OCI (v1) */
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:config...",
    "size": 1500
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:layer1...",
      "size": 72593453
    }
  ]
}
```

Docker puede consumir ambos formatos. BuildKit (el constructor moderno de Docker) genera imágenes OCI por defecto.

---

## 5. Arquitectura de Docker

Docker no es un monolito. Es un ecosistema de componentes que colaboran mediante APIs bien definidas.

### 5.1 Diagrama de arquitectura

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ARQUITECTURA DOCKER                          │
│                                                                      │
│  ┌──────────────┐                                                    │
│  │  Docker CLI  │  (docker build, docker run, docker ps...)         │
│  │  (cliente)   │                                                    │
│  └──────┬───────┘                                                    │
│         │ REST API sobre Unix Socket (/var/run/docker.sock)          │
│         │ o TCP (con TLS)                                            │
│         ▼                                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    dockerd (Docker Daemon)                    │   │
│  │  ┌─────────────────────────────────────────────────────────┐ │   │
│  │  │  - Orquestación de contenedores                          │ │   │
│  │  │  - Gestión de imágenes (pull, push, build)               │ │   │
│  │  │  - Gestión de redes (bridge, overlay, macvlan)           │ │   │
│  │  │  - Gestión de volúmenes                                  │ │   │
│  │  │  - Gestión de plugins (authz, volume, network)           │ │   │
│  │  │  - Swarm mode (orquestación en clúster)                  │ │   │
│  │  └─────────────────────────────────────────────────────────┘ │   │
│  └──────┬───────────────────────────────────────────────────────┘   │
│         │ gRPC                                                       │
│         ▼                                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      containerd                               │   │
│  │  ┌─────────────────────────────────────────────────────────┐ │   │
│  │  │  - Gestión del ciclo de vida de contenedores             │ │   │
│  │  │  - Transferencia de imágenes (push/pull)                 │ │   │
│  │  │  - Gestión de snapshots (storage)                        │ │   │
│  │  │  - Gestión de namespaces de containerd                   │ │   │
│  │  │  - API estándar (O CI runtime spec)                     │ │   │
│  │  └─────────────────────────────────────────────────────────┘ │   │
│  └──────┬───────────────────────────────────────────────────────┘   │
│         │ Fork + exec                                                │
│         ▼                                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   containerd-shim                             │   │
│  │  ┌─────────────────────────────────────────────────────────┐ │   │
│  │  │  - Proceso intermediario por contenedor                  │ │   │
│  │  │  - Mantiene STDIO abierto                                │ │   │
│  │  │  - Reporta código de salida a containerd                 │ │   │
│  │  │  - Permite que containerd se reinicie sin matar          │ │   │
│  │  │    los contenedores (daemonless containers)              │ │   │
│  │  └─────────────────────────────────────────────────────────┘ │   │
│  └──────┬───────────────────────────────────────────────────────┘   │
│         │ Exec (OCI Runtime)                                         │
│         ▼                                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                        runc                                   │   │
│  │  ┌─────────────────────────────────────────────────────────┐ │   │
│  │  │  - Crea namespaces de Linux                              │ │   │
│  │  │  - Aplica cgroups                                        │ │   │
│  │  │  - Configura capabilities, seccomp, AppArmor/SELinux     │ │   │
│  │  │  - Monta el rootfs                                       │ │   │
│  │  │  - Ejecuta el proceso del contenedor                     │ │   │
│  │  │  - Implementa OCI Runtime Specification                  │ │   │
│  │  └─────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              KERNEL DE LINUX                                  │   │
│  │  namespaces │ cgroups │ seccomp │ capabilities │ LSM         │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Componente por componente

#### dockerd (Docker Daemon)

Es el cerebro de Docker. Expone la API REST que el cliente `docker` consume. Es responsable de la orquestación de alto nivel:

- **Gestión de imágenes**: `docker pull`, `docker push`, `docker build`. Habla con registries (Docker Hub, ECR, GCR, registries privados).
- **Gestión de contenedores**: orquesta la creación, inicio, parada y eliminación de contenedores delegando en containerd.
- **Redes**: crea bridges, overlay networks (para Swarm), macvlan, ipvlan. Gestiona iptables y DNS interno.
- **Volúmenes**: mounts de tipo bind, volume, tmpfs.
- **Plugins**: sistema de extensiones para autenticación, redes y volúmenes.
- **Swarm mode**: orquestación nativa en clúster de Docker.

El daemon normalmente escucha en un Unix socket (`/var/run/docker.sock`) y opcionalmente en TCP con TLS.

#### containerd

Originalmente, containerd era parte de dockerd. En 2016, Docker lo extrajo como proyecto independiente y en 2017 lo donó a la CNCF (Cloud Native Computing Foundation). Hoy containerd es un proyecto graduado de la CNCF.

Containerd se centra en la **gestión del ciclo de vida del contenedor**, sin las capas de alto nivel que dockerd proporciona. Es el estándar de facto para runtimes de contenedores y lo usan Kubernetes (a través de CRI), Docker, y otras plataformas.

Características:
- API gRPC estable y bien definida.
- Cliente `ctr` y `nerdctl` (como `ctr` pero con una interfaz más amigable tipo Docker).
- Transferencia de imágenes con soporte para múltiples formatos.
- Snapshotters (storage drivers).
- Namespaces de containerd (independientes de los namespaces del kernel).

#### containerd-shim

El shim resuelve un problema concreto: ¿qué pasa si el daemon de Docker o containerd se reinicia? Sin el shim, todos los contenedores morirían porque su proceso padre (containerd) desaparecería.

El shim actúa como proceso intermediario:
1. containerd lanza `containerd-shim`.
2. El shim lanza `runc`, que crea el contenedor.
3. runc termina (su trabajo está hecho: creó el contenedor).
4. El shim permanece como padre del proceso del contenedor.
5. Si containerd se reinicia, el shim mantiene el contenedor vivo.
6. El shim reenvía stdin/stdout/stderr y reporta el código de salida cuando el contenedor termina.

Esto permite **actualizaciones sin downtime de dockerd**: puedes reiniciar el daemon de Docker sin que los contenedores se detengan.

#### runc

runc es la culminación de todo lo que hemos estudiado en este capítulo. Es un pequeño binario (unos 10MB) que implementa la **OCI Runtime Specification**. Su trabajo es:

1. Leer un archivo `config.json` (especificación OCI del contenedor).
2. Crear namespaces (PID, NET, MNT, UTS, IPC, etc.).
3. Aplicar límites de cgroups.
4. Configurar capacidades de Linux, seccomp profiles, AppArmor/SELinux.
5. Montar el rootfs usando el storage driver configurado.
6. Hacer `pivot_root` para cambiar el sistema de archivos raíz del proceso.
7. Ejecutar (`exec`) el proceso especificado en el comando del contenedor.

Podemos invocar runc directamente para entender exactamente cómo funciona la creación de contenedores:

```bash
# Paso 1: Crear un rootfs (sistema de archivos mínimo)
$ mkdir -p /tmp/mycontainer/rootfs
$ docker export $(docker create alpine) | tar -C /tmp/mycontainer/rootfs -xvf -

# Paso 2: Generar la especificación OCI (config.json)
$ cd /tmp/mycontainer
$ runc spec
# Esto crea config.json con valores por defecto

# Paso 3: Ejecutar el contenedor con runc directamente
$ sudo runc run my-container
# Esto arranca un shell dentro del rootfs, ¡sin Docker!

# En otra terminal:
$ runc list
ID             PID         STATUS      BUNDLE              CREATED
my-container   12345       running     /tmp/mycontainer    2024-01-15T10:30:00Z

# Paso 4: Matar el contenedor
$ runc kill my-container
```

Este ejemplo demuestra algo fundamental: **Docker es conveniencia, no magia**. runc puede crear contenedores sin Docker. Docker proporciona la capa de gestión, networking, volúmenes y la experiencia de usuario, pero el trabajo pesado lo hacen runc y el kernel.

### 5.3 El modelo cliente-servidor

Docker usa una arquitectura cliente-servidor. El cliente (`docker`) y el daemon (`dockerd`) pueden estar en la misma máquina o en máquinas distintas. La comunicación es vía REST API sobre Unix socket o TCP.

**Demostración con curl sobre Unix socket:**

```bash
# Listar contenedores usando la API REST directamente
$ curl --unix-socket /var/run/docker.sock \
    http://localhost/v1.44/containers/json | jq

# Inspeccionar un contenedor
$ curl --unix-socket /var/run/docker.sock \
    http://localhost/v1.44/containers/<id>/json | jq

# Listar imágenes
$ curl --unix-socket /var/run/docker.sock \
    http://localhost/v1.44/images/json | jq

# Crear un contenedor
$ curl --unix-socket /var/run/docker.sock \
    -H "Content-Type: application/json" \
    -d '{"Image": "alpine", "Cmd": ["echo", "hola desde API"]}' \
    http://localhost/v1.44/containers/create

# Iniciar el contenedor creado
$ curl --unix-socket /var/run/docker.sock \
    -X POST \
    http://localhost/v1.44/containers/<id>/start

# Ver logs
$ curl --unix-socket /var/run/docker.sock \
    http://localhost/v1.44/containers/<id>/logs?stdout=true
hola desde API
```

Cualquier herramienta que pueda hacer peticiones HTTP puede gestionar Docker. Esto es la base de las librerías cliente en Python, Go, Node.js, etc., y de herramientas como Docker Compose y Portainer.

**Docker Context: conectándose a daemons remotos**

El cliente `docker` puede conectarse a múltiples daemons usando "contextos". Esto permite gestionar contenedores en máquinas remotas o en clústeres de Swarm/Kubernetes sin cambiar de terminal:

```bash
# Listar contextos existentes
$ docker context ls
NAME        TYPE        DESCRIPTION
default *   moby        Current DOCKER_HOST based configuration

# Crear un contexto para un servidor remoto
$ docker context create remote-server \
    --docker "host=ssh://user@192.168.1.50"

# Cambiar al contexto remoto
$ docker context use remote-server

# Ahora todos los comandos docker van al servidor remoto
$ docker ps         # Muestra contenedores del servidor remoto
$ docker run nginx  # Crea un contenedor en el servidor remoto

# Volver al contexto local
$ docker context use default
```

Esto usa SSH para la conexión, sin exponer el socket Docker a la red. Es la forma recomendada (y más segura) de gestionar Docker remoto. También puedes conectarte a contextos de Kubernetes (`kubectl`), Amazon ECS, y Azure ACI con extensiones.

**Demo: conectarse a un daemon remoto con TLS**

```bash
# En el servidor: configurar dockerd para escuchar en TCP con TLS
# /etc/docker/daemon.json
{
  "tls": true,
  "tlsverify": true,
  "tlscacert": "/etc/docker/certs/ca.pem",
  "tlscert": "/etc/docker/certs/server-cert.pem",
  "tlskey": "/etc/docker/certs/server-key.pem",
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"]
}

# En el cliente
$ docker --tlsverify \
    --tlscacert=ca.pem \
    --tlscert=cert.pem \
    --tlskey=key.pem \
    -H=192.168.1.50:2376 \
    ps
```

### 5.4 ¿Por qué esta separación de componentes?

La arquitectura modular de Docker ofrece ventajas concretas:

1. **Desacoplamiento**: Cada componente tiene una responsabilidad clara. runc no sabe nada de imágenes. containerd no sabe nada de redes overlay. dockerd no implementa namespaces.

2. **Actualización sin detener contenedores**: Como el shim es el padre de los procesos, puedes reiniciar containerd o dockerd sin que tus contenedores se detengan (daemonless containers).

3. **Interoperabilidad**: Como containerd y runc implementan estándares OCI, otras herramientas pueden sustituir componentes. Por ejemplo, Kubernetes puede usar containerd directamente sin necesidad de dockerd.

4. **Reemplazo de componentes**: Puedes cambiar runc por crun (más rápido, escrito en C), kata-containers (máquinas virtuales ligeras), o gVisor (sandbox en userspace), y containerd seguirá funcionando igual.

5. **Pruebas y depuración**: Puedes probar cada componente por separado. Si un contenedor no arranca, puedes ejecutar `runc` directamente para aislar el problema.

### 5.5 Mecanismos de seguridad en Docker

Entender la arquitectura de componentes no está completo sin entender las capas de seguridad que Docker aplica. Cuando ejecutas `docker run`, no solo se crean namespaces y cgroups. Docker aplica **por defecto** varias capas de seguridad del kernel de Linux. Vamos a explorar las tres más importantes.

#### Linux Capabilities

Tradicionalmente, en Linux hay dos tipos de procesos: privilegiados (root, UID 0) y no privilegiados. Root puede hacer cualquier cosa. Pero esta división binaria es muy burda. Las **capabilities** dividen los privilegios de root en unidades pequeñas e independientes que pueden asignarse selectivamente.

**Capabilities relevantes para contenedores:**

| Capability        | Permiso                                              | ¿La necesita un contenedor típico? |
|-------------------|------------------------------------------------------|-------------------------------------|
| `CAP_CHOWN`       | Cambiar dueño de archivos                            | A veces (si ejecuta como root)     |
| `CAP_DAC_OVERRIDE`| Ignorar permisos de archivos                         | A veces                             |
| `CAP_FOWNER`      | Ignorar comprobación de dueño                        | Rara vez                            |
| `CAP_KILL`        | Enviar señales a procesos de otros usuarios          | Sí (para matar procesos hijos)     |
| `CAP_SETUID`      | Cambiar UID/GID                                      | Rara vez (peligroso)               |
| `CAP_SETPCAP`     | Modificar capabilities de otros procesos             | Nunca                               |
| `CAP_NET_BIND_SERVICE` | Escuchar en puertos < 1024                     | Sí (si necesitas puerto 80/443)    |
| `CAP_NET_RAW`     | Usar sockets RAW y PACKET                            | Rara vez                            |
| `CAP_SYS_ADMIN`   | Operaciones de administración del sistema             | Nunca para apps normales            |
| `CAP_SYS_PTRACE`  | Trazear procesos de otros                            | Sí (debugging)                     |
| `CAP_SYS_TIME`    | Cambiar el reloj del sistema                         | Nunca                               |
| `CAP_SYSLOG`      | Operaciones de syslog privilegiadas                   | Nunca                               |

Por defecto, Docker **elimina** un conjunto de capabilities peligrosas del contenedor, incluso si se ejecuta como root. El conjunto exacto incluye `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, `CAP_SYS_PTRACE`, `CAP_SYS_MODULE`, entre otras.

```bash
# Ver las capabilities de un contenedor
$ docker run --rm -it alpine sh -c 'capsh --print'
Current: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,
cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,
cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap=eip
# "=eip" significa: effective, inheritable, permitted

# Compara con las capabilities de root en el host:
$ sudo capsh --print
# La lista es mucho más larga
```

Puedes añadir o quitar capabilities específicas:

```bash
# Añadir CAP_NET_ADMIN (necesario para iptables, VPNs dentro del contenedor)
$ docker run --cap-add=NET_ADMIN --rm -it alpine sh

# Quitar todas las capabilities, luego añadir solo las necesarias
$ docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Verificar: intentar cambiar el hostname (necesita CAP_SYS_ADMIN)
$ docker run --cap-drop=ALL --rm -it alpine hostname nuevo-nombre
hostname: sethostname: Operation not permitted
```

#### Seccomp (Secure Computing Mode)

seccomp permite a un proceso restringir qué syscalls (llamadas al sistema) puede invocar. Es la última línea de defensa: si un atacante compromete un proceso dentro del contenedor, seccomp limita drásticamente lo que puede hacer.

Docker usa **perfiles seccomp** por defecto. El perfil bloquea aproximadamente 44 syscalls de las ~300 disponibles, incluyendo algunas que son históricamente peligrosas:

```json
/* Fragmento del perfil seccomp por defecto de Docker */
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_AARCH64"],
  "syscalls": [
    {
      "names": ["accept", "accept4", "bind", "clone", "close", ...],
      "action": "SCMP_ACT_ALLOW"
    },
    /* Syscalls bloqueadas por defecto: */
    /* kexec_load, bpf, perf_event_open, mount (parcial), ... */
  ]
}
```

Syscalls bloqueadas y por qué:

| Syscall           | Por qué se bloquea                                               |
|-------------------|------------------------------------------------------------------|
| `kexec_load`      | Cargar y ejecutar un nuevo kernel (bypass total del aislamiento) |
| `bpf`             | Cargar programas BPF en el kernel (pueden leer memoria arbitaria) |
| `perf_event_open` | Puede filtrar información del kernel y otros procesos             |
| `ptrace`          | Trazear otros procesos (se requiere capacidad explícita)          |
| `mount` (parcial) | Restringe montajes peligrosos                                     |
| `reboot`          | Reiniciar el sistema                                              |
| `setdomainname`   | Cambiar el NIS domain name                                       |

Puedes personalizar el perfil seccomp:

```bash
# Usar un perfil seccomp personalizado
$ docker run --security-opt seccomp=/path/to/profile.json myapp

# Desactivar completamente seccomp (¡solo para debugging!)
$ docker run --security-opt seccomp=unconfined myapp
```

#### AppArmor / SELinux

Son **LSMs** (Linux Security Modules) que implementan control de acceso obligatorio (MAC - Mandatory Access Control). A diferencia del DAC (Discretionary Access Control, los permisos clásicos de usuario/grupo/otros), MAC es impuesto por el kernel y los usuarios no pueden eludirlo.

- **AppArmor**: se configura mediante perfiles por aplicación. Es el LSM por defecto en Ubuntu/Debian.
- **SELinux**: etiquetas de seguridad en cada objeto (archivos, procesos, sockets). Usado en RHEL/CentOS/Fedora.

Docker genera perfiles automáticamente:

```bash
# Ver el perfil AppArmor aplicado a un contenedor
$ docker inspect --format='{{.AppArmorProfile}}' mi-contenedor
docker-default

# Ver el perfil en detalle
$ sudo cat /etc/apparmor.d/docker
# O
$ sudo aa-status | grep docker

# Ejecutar sin AppArmor (no recomendado en producción)
$ docker run --security-opt apparmor=unconfined nginx
```

**La estratificación de seguridad:**

```
┌─────────────────────────────────────────────────────────┐
│                    CAPAS DE SEGURIDAD                    │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  6. Usuario no-root en el contenedor               │ │
│  │     ──────────────────────────────────────────────│ │
│  │  5. AppArmor / SELinux (control de acceso por LSM) │ │
│  │     ──────────────────────────────────────────────│ │
│  │  4. Seccomp (restricción de syscalls)              │ │
│  │     ──────────────────────────────────────────────│ │
│  │  3. Linux Capabilities (privilegios fragmentados)  │ │
│  │     ──────────────────────────────────────────────│ │
│  │  2. USER namespace (mapeo UID/GID aislado)         │ │
│  │     ──────────────────────────────────────────────│ │
│  │  1. Namespaces (PID, NET, MNT, IPC, UTS, CGROUP)   │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  DEFENSA EN PROFUNDIDAD: Cada capa contiene lo que       │
│  pueda escapar de la capa superior.                      │
└─────────────────────────────────────────────────────────┘
```

Cada capa se apoya en la anterior. Si un atacante escapa de un namespace, las capabilities le limitan. Si escala a más capabilities, seccomp bloquea las syscalls peligrosas. Si logra invocar una syscall bloqueada, AppArmor/SELinux impide el acceso a los archivos/dispositivos del host.

**¿Es Docker inseguro por defecto?**

No. Docker por defecto aplica namespaces + cgroups + un conjunto reducido de capabilities + un perfil seccomp restrictivo. Pero la seguridad no es absoluta. Los escapes de contenedores existen (CVEs de runc, del kernel), aunque son raros. Para workloads que requieren el máximo aislamiento, se recomiendan runtimes especializados como gVisor o Kata Containers que añaden capas adicionales (kernel en userspace en el caso de gVisor, o una VM ligera en el caso de Kata).

---

## 6. OCI (Open Container Initiative)

### 6.1 ¿Qué es la OCI?

La **Open Container Initiative** (OCI) es un proyecto de la Linux Foundation lanzado en 2015 por Docker, CoreOS, Google, Microsoft, Amazon, y otros líderes de la industria. Su objetivo: crear estándares abiertos para contenedores, evitando el vendor lock-in y garantizando que cualquier contenedor funcione en cualquier runtime compatible.

La OCI define dos especificaciones principales:

### 6.2 OCI Runtime Specification

Define cómo ejecutar un contenedor. Esto incluye:

- **Operaciones del ciclo de vida**: create, start, kill, delete.
- **Configuración**: `config.json` con namespaces, cgroups, capacidades, mounts, hooks.
- **Sistema de archivos**: el rootfs que se monta dentro del contenedor.
- **Estado**: cómo reportar el estado del contenedor.

Implementaciones de la Runtime Spec:
- **runc** (Docker/OCI): la implementación de referencia, escrita en Go.
- **crun** (Red Hat): implementación en C, más rápida y con menor uso de memoria.
- **kata-containers**: usa VMs ligeras para mayor aislamiento.
- **gVisor** (Google): sandbox en userspace que implementa el kernel de Linux en Go.
- **youki** (community): implementación en Rust.

Todas estas implementaciones leen el mismo `config.json` y producen el mismo resultado: un contenedor en ejecución conforme a la especificación.

### 6.3 OCI Image Specification

Define el formato estándar para imágenes de contenedor:

- **Manifest**: metadatos de la imagen (referencias a config y capas).
- **Config**: parámetros de ejecución (entrypoint, cmd, env, volumes, etc.).
- **Layers**: cada capa es un tar.gz con los cambios sobre la capa anterior.
- **Media Types**: tipos MIME que identifican el contenido (e.g., `application/vnd.oci.image.layer.v1.tar+gzip`).

Un ejemplo de manifiesto OCI:

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:abc...",
    "size": 1500
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:def...",
      "size": 72593453
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:ghi...",
      "size": 1024
    }
  ]
}
```

### 6.4 ¿Por qué importan los estándares OCI?

Antes de la OCI, cada plataforma de contenedores tenía su propio formato de imagen y su propio runtime. Una imagen de Docker solo funcionaba en Docker. Hoy:

- Una imagen OCI puede ejecutarse en Docker, Podman, containerd, CRI-O, o cualquier runtime OCI.
- Puedes construir imágenes con Buildah y ejecutarlas con Docker.
- Puedes construir imágenes con `docker build` y ejecutarlas con Podman.
- Kubernetes puede usar containerd, CRI-O, o Docker como runtime de contenedores, todos hablando el mismo lenguaje.

La estandarización es lo que ha permitido que el ecosistema de contenedores florezca sin quedar atrapado en la implementación de un solo proveedor.

### 6.5 La anatomía completa de `docker run`

Hemos cubierto namespaces, cgroups, OverlayFS, arquitectura de componentes, seguridad y estándares OCI. Es hora de unirlo todo. ¿Qué sucede exactamente, paso a paso, cuando ejecutas `docker run -d --cpus=1.5 --memory=256m -p 8080:80 nginx`?

```
PASO A PASO: docker run -d --cpus=1.5 --memory=256m -p 8080:80 nginx
═══════════════════════════════════════════════════════════════════════

1. DOCKER CLI parsea los argumentos.
   ├─ Valida que la sintaxis sea correcta.
   └─ Serializa la petición a JSON.

2. DOCKER CLI envía petición HTTP a dockerd
   POST /v1.44/containers/create
   {
     "Image": "nginx",
     "Cmd": null,         (usará el CMD de la imagen)
     "HostConfig": {
       "CpuPeriod": 100000,
       "CpuQuota": 150000,
       "Memory": 268435456,
       "PortBindings": {"80/tcp": [{"HostPort": "8080"}]}
     }
   }

3. DOCKERD (daemon) recibe la petición.
   ├─ Verifica que la imagen nginx existe localmente.
   │  └─ Si no, hace pull del registry (Docker Hub).
   │     ├─ Descarga el manifiesto.
   │     ├─ Compara digests de capas.
   │     └─ Descarga solo las capas nuevas.
   │
   ├─ Configura la red:
   │  ├─ Reserva IP 172.17.0.X en el bridge docker0.
   │  ├─ Crea reglas iptables DNAT para el puerto 8080→80.
   │  └─ Configura el DNS interno.
   │
   └─ Delega en containerd vía gRPC:
      CreateContainer(image="nginx", cpus=1.5, memory=256m, net=bridge, ...)

4. CONTAINERD recibe la petición gRPC.
   ├─ Configura el snapshotter (OverlayFS):
   │  ├─ Identifica las capas de la imagen nginx.
   │  ├─ Crea un directorio upperdir vacío (rw).
   │  └─ Monta merged = upperdir + lowerdir layers.
   │
   └─ Lanza containerd-shim como proceso hijo.
      └─ Fork + exec containerd-shim

5. CONTAINERD-SHIM:
   ├─ Abre pipes para stdin/stdout/stderr.
   ├─ Ejecuta runc con config.json (especificación OCI).
   └─ runc crea el contenedor.

6. RUNC (el runtime OCI):
   ├─ Lee config.json generado por containerd.
   │
   ├─ Crea NAMESPACES (unshare/clone):
   │  ├─ PID: nuevo espacio de PIDs (proceso nginx = PID 1).
   │  ├─ NET: nuevo espacio de red, con veth pair a docker0.
   │  ├─ MNT: nuevo espacio de montaje, overlay sobre rootfs.
   │  ├─ UTS: hostname = ID del contenedor.
   │  ├─ IPC: aislamiento de semáforos y colas.
   │  └─ CGROUP: vista aislada de /sys/fs/cgroup.
   │
   ├─ Aplica CGROUPS:
   │  ├─ cpu.max = "150000 100000"  (1.5 CPUs)
   │  ├─ memory.max = 268435456     (256 MB)
   │  └─ pids.max = (sin límite explícito)
   │
   ├─ Configura SEGURIDAD:
   │  ├─ Capabilities: conjunto reducido (se eliminan las peligrosas).
   │  ├─ Seccomp: aplica perfil por defecto (bloquea ~44 syscalls).
   │  └─ AppArmor/SELinux: aplica perfil docker-default.
   │
   ├─ Monta el ROOTFS:
   │  └─ pivot_root al merged del overlay mount.
   │
   ├─ Ejecuta el PROCESO:
   │  └─ exec("nginx", "-g", "daemon off;")
   │     └─ El proceso nginx se convierte en PID 1 del namespace.
   │
   └─ runc TERMINA. Su trabajo está hecho.

7. CONTENEDOR EN EJECUCIÓN:
   ├─ containerd-shim permanece como padre del proceso nginx.
   ├─ containerd-shim reenvía stdout/stderr.
   ├─ dockerd reporta el estado al cliente.
   └─ docker CLI muestra el ID del contenedor.
```

Este flujo completo, de principio a fin, es lo que distingue a un ingeniero que *entiende* Docker de uno que solo *usa* Docker. Cada componente de este diagrama corresponde a un concepto que hemos estudiado en este capítulo.

---

## 7. Historia de Docker

> *"Los que no conocen la historia están condenados a repetirla."*  
> — George Santayana

### 7.1 Los precursores: antes de Docker

Los contenedores no los inventó Docker. La tecnología subyacente existía desde hacía años en el kernel de Linux:

- **1979**: `chroot` en Unix V7. Cambia el directorio raíz de un proceso.
- **2000**: FreeBSD Jails. Aislamiento completo: filesystem, procesos, red.
- **2001**: Linux-VServer. Parches para virtualización a nivel de SO en Linux.
- **2005**: OpenVZ. Contenedores en Linux con kernel parcheado.
- **2008**: LXC (LinuX Containers). Primer uso de namespaces y cgroups del kernel mainline.
- **2011**: CloudFoundry Warden. Aislamiento de aplicaciones en entornos PaaS.

El problema era que ninguna de estas tecnologías era accesible para desarrolladores. Configurar LXC requería conocimientos profundos de Linux. No había portabilidad: un contenedor LXC creado en Ubuntu no funcionaba fácilmente en CentOS.

### 7.2 2013: El nacimiento de Docker

**Solomon Hykes**, fundador de una empresa llamada **dotCloud** (un PaaS), presentó Docker en PyCon 2013 en Santa Clara, California. Su demo de 5 minutos cambió la industria.

DotCloud tenía un problema interno: su plataforma PaaS necesitaba una forma de empaquetar aplicaciones de forma portable y ligera. Hykes y su equipo construyeron Docker como herramienta interna. En PyCon 2013, liberaron el código como open source bajo licencia Apache 2.0.

Lo que hizo Docker diferente:

1. **Imágenes portables**: Un `Dockerfile` simple + `docker build` producía una imagen que funcionaba en cualquier sitio.
2. **Docker Hub**: Un registry público donde compartir imágenes (como GitHub para código).
3. **Experiencia de usuario**: `docker run` vs. docenas de comandos LXC con flags crípticas.
4. **Union filesystem**: Las capas de imágenes hacían que el almacenamiento y la transferencia fueran eficientes.

En cuestión de meses, Docker pasó de ser el proyecto interno de una startup a ser adoptado por empresas como eBay, Spotify, y Baidu.

### 7.3 2014: Consolidación

- **Junio 2014**: Docker 1.0. La API se declara estable.
- **Docker Hub** se lanza oficialmente como marketplace de imágenes.
- **Docker Compose** (originalmente "Fig", adquirido por Docker) permite definir aplicaciones multi-contenedor.
- **Docker Machine** facilita la creación de hosts Docker en la nube.
- **Red Hat** anuncia soporte oficial para Docker en RHEL.
- **Microsoft** anuncia soporte para Docker en Azure y Windows Server.

Al final de 2014, Docker había recaudado $40M en financiación y la comunidad de desarrolladores era masiva.

### 7.4 2015: La era de los estándares

El crecimiento explosivo trajo fragmentación. Google (con Kubernetes), CoreOS (con rkt y Tectonic), Red Hat y otros competían en el espacio de orquestación y contenedores. Cada uno tenía su propio formato de imagen y runtime.

Para evitar una guerra de formatos, Docker y CoreOS anunciaron la **Open Container Initiative** (OCI) bajo la Linux Foundation. Docker donó un subconjunto de su código, que se convirtió en **runc** y **containerd**. La OCI estandarizó:

- El formato de imagen (image-spec).
- El runtime de contenedores (runtime-spec).

Esta decisión estratégica evitó un cisma tipo "Unix wars" y consolidó el formato de imagen de Docker como estándar de facto de la industria.

### 7.5 2016-2017: La guerra de orquestación

Docker lanzó **Docker Swarm** como solución nativa de orquestación, integrada en el daemon de Docker. Docker Swarm era simple: convertir un cluster de Docker Engines en un orquestador con `docker swarm init` y `docker swarm join`.

Mientras tanto, Google, basándose en su experiencia con Borg, donó **Kubernetes** a la recién creada CNCF (Cloud Native Computing Foundation).

La comunidad debatió durante dos años cuál prevalecería. Al final, Kubernetes ganó:

- Soporte de todos los grandes proveedores cloud (GKE, AKS, EKS).
- Ecosistema de herramientas (Helm, Istio, Prometheus).
- Arquitectura más flexible y extensible que Swarm.
- Comunidad masiva y rápida iteración.

En **DockerCon 2017**, Docker anunció soporte nativo para Kubernetes en Docker EE (Enterprise Edition), reconociendo la victoria de Kubernetes. Los problemas de gobernanza interna llevaron a Hykes a dejar la compañía en 2018.

**¿Por qué ganó Kubernetes?**

La respuesta corta: API extensible y comunidad. Kubernetes construyó una API declarativa con Custom Resource Definitions (CRDs), lo que permitió a la comunidad crear un ecosistema de operadores para todo tipo de aplicaciones (bases de datos, message queues, service meshes). Docker Swarm ofrecía una experiencia más simple, pero carecía de la extensibilidad que las empresas necesitaban para gestionar infraestructura compleja a escala.

### 7.6 2018-presente: Madurez y especialización

- **2018**: Docker vende su negocio empresarial (Docker EE) a Mirantis por ~$35M. La compañía se enfoca en desarrollo y experiencia de usuario.
- **2019**: Docker se reestructura completamente. Se lanza Docker Desktop Enterprise. containerd alcanza el estatus de "graduado" en la CNCF.
- **2020**: Docker Desktop WSL 2 backend se lanza para Windows, ofreciendo rendimiento casi nativo en Windows. La pandemia acelera la adopción de entornos de desarrollo reproducibles.
- **2021**: Cambios en la licencia de Docker Desktop (requiere suscripción para empresas de más de 250 empleados o $10M de ingresos). Esto genera controversia y acelera la adopción de alternativas open source: Podman, nerdctl, Rancher Desktop, Colima.
- **2022**: Docker Extensions permite plugins en Docker Desktop (seguridad, observabilidad, debugging). Docker Inc. recauda $105M en financiación Serie C.
- **2023**: BuildKit se convierte en el constructor por defecto (reemplazando al antiguo builder). Docker Scout para análisis de vulnerabilidades. `docker compose watch` para desarrollo con hot reload.
- **2024-2025**: Docker se centra en la experiencia del desarrollador con herramientas como `docker init` (genera automáticamente Dockerfiles, compose files), mejoras en Docker Desktop para Apple Silicon (M1/M2/M3/M4), y funcionalidades de IA (Docker AI).

**Lecciones de la historia de Docker:**

1. **La simplicidad gana**: Docker triunfó sobre LXC porque era más fácil de usar.
2. **Los estándares abiertos importan**: Donar runc y containerd a la OCI y CNCF solidificó a Docker como estándar, incluso cuando Kubernetes ganó la guerra de orquestación.
3. **El ecosistema es más grande que una empresa**: Docker perdió la guerra de orquestación, pero el contenedor como unidad de empaquetado es omnipresente gracias a la estandarización.
4. **La experiencia de desarrollo es el campo de batalla actual**: Después de la consolidación en producción (Kubernetes), la innovación se ha desplazado a mejorar la experiencia del desarrollador.

---

## 8. Ecosistema actual

El ecosistema de contenedores en 2025 es rico, diverso y a veces confuso. Aquí tienes una guía para navegarlo.

### 8.1 Tabla de herramientas del ecosistema

| Herramienta           | Tipo                    | Descripción breve                                     | Mejor para...                                    |
|-----------------------|-------------------------|-------------------------------------------------------|--------------------------------------------------|
| **Docker Engine**     | Runtime + gestión       | El original. Contenedores, imágenes, redes, volúmenes | Desarrollo, CI/CD, uso general                    |
| **Docker Desktop**    | Aplicación de escritorio| Docker Engine + GUI + extensiones + Kubernetes        | Desarrolladores en macOS/Windows                  |
| **Podman**            | Runtime + CLI           | Alternativa daemonless, rootless por defecto          | Seguridad, entornos sin daemon, compatibilidad systemd |
| **Buildah**           | Constructor de imágenes | Construye imágenes OCI sin daemon                     | Pipelines CI/CD, builds rootless                 |
| **Kaniko**            | Constructor en contenedor| Construye imágenes dentro de Kubernetes sin Docker   | CI/CD en Kubernetes                               |
| **containerd**        | Runtime (bajo nivel)    | Gestión del ciclo de vida, usado por Kubernetes       | Infraestructura, runtimes personalizados          |
| **CRI-O**             | Runtime para Kubernetes | Implementación ligera de CRI (Container Runtime Interface) | Nodos Kubernetes, entornos OpenShift    |
| **nerdctl**           | CLI tipo Docker         | CLI compatible con Docker para containerd             | Reemplazo directo de Docker CLI con containerd    |
| **Finch**             | Herramienta CLI         | nerdctl + Lima (VM) empaquetado para macOS            | Desarrolladores macOS que quieren evitar Docker Desktop |
| **Rancher Desktop**   | Aplicación de escritorio| Alternativa open source a Docker Desktop              | Desarrolladores que prefieren open source         |
| **Colima**            | Runtime en VM           | Containerd + Lima, reemplazo ligero de Docker Desktop | Desarrolladores macOS/Linux minimalistas          |
| **gVisor**            | Sandbox                 | Kernel en userspace (Go)                              | Aislamiento fuerte para entornos multi-tenant     |
| **Kata Containers**   | Runtime con VM          | Contenedores en VMs ligeras                           | Seguridad máxima, cargas de trabajo no confiables |
| **Firecracker**       | MicroVM                 | VMs ligeras (AWS Lambda/Fargate)                      | Serverless, alto aislamiento                      |
| **Skopeo**            | Manipulación de imágenes| Inspecciona, copia, firma imágenes entre registries   | Inspección y movimiento de imágenes               |
| **Docker Compose**    | Orquestación local      | Define y ejecuta aplicaciones multi-contenedor        | Desarrollo local, entornos de prueba              |
| **Docker Scout**      | Seguridad               | Análisis de vulnerabilidades en imágenes              | Seguridad de cadena de suministro                 |
| **Cosign**            | Firma de imágenes       | Firma y verificación de imágenes OCI con Sigstore      | Integridad de imágenes en producción              |

### 8.2 ¿Cuándo usar cada una?

**Desarrollo local (tu portátil):**
- macOS/Windows: Docker Desktop (lo más pulido) o Rancher Desktop / Colima (open source).
- Linux: Docker Engine o Podman.

**CI/CD pipelines:**
- GitHub Actions/GitLab CI: Docker Engine (está disponible por defecto).
- Pipelines en Kubernetes: Kaniko (construye sin requerir acceso al socket de Docker).

**Producción (orquestación):**
- Kubernetes con containerd o CRI-O.

**Entornos con altos requisitos de seguridad:**
- Podman en modo rootless.
- gVisor o Kata Containers para aislamiento fuerte.
- Usar `--security-opt` con seccomp y AppArmor/SELinux.

**Reemplazo ligero de Docker sin daemon:**
- Podman (CLI casi idéntica a Docker, pero sin daemon).
- nerdctl (misma CLI de Docker, sobre containerd).

### 8.3 Comparativa detallada: Docker vs Podman vs nerdctl

Para que tomes una decisión informada sobre qué herramienta usar, aquí tienes una comparativa detallada desde múltiples ángulos:

| Aspecto               | Docker                    | Podman                        | nerdctl                       |
|-----------------------|---------------------------|-------------------------------|-------------------------------|
| **Arquitectura**      | Cliente-servidor (daemon) | Daemonless (fork/exec directo)| Cliente de containerd (daemon)|
| **Permisos**          | Requiere root (o grupo docker) | Rootless sin configuración | Rootless con containerd configurado |
| **Compatibilidad CLI**| 100% (referencia)         | ~95% (alias docker=podman)    | ~90% (compatible con Docker)  |
| **Orquestación**      | Docker Compose, Swarm     | Podman Compose, pods, systemd | nerdctl compose (experimental)|
| **Kubernetes**        | No nativo (dockershim EOL)| Genera YAML de pods           | Kubernetes nativo (CRI)       |
| **Construcción**      | BuildKit integrado        | Buildah integrado             | BuildKit (opcional)            |
| **Registries**        | Docker Hub por defecto    | Múltiples (configurable)      | Múltiples (configurable)      |
| **Sistema operativo** | Linux, macOS (VM), Windows (VM) | Linux nativo, macOS (VM), Windows (WSL) | Linux nativo        |
| **Gestión de pods**   | No (necesita Kubernetes)  | Sí (pods nativos)             | No                             |
| **Integración systemd**| No                        | Genera unidades systemd       | No                             |

**¿Qué significan pods en Podman?**

Un pod es un grupo de contenedores que comparten ciertos namespaces (NET, IPC, a veces PID). Es el equivalente al Pod de Kubernetes. Podman puede crear pods nativamente, lo que facilita la transición de desarrollo local a producción en Kubernetes:

```bash
# Crear un pod con Podman
$ podman pod create --name mi-pod -p 8080:80

# Añadir contenedores al pod (comparten NET namespace)
$ podman run -d --pod mi-pod --name web nginx
$ podman run -d --pod mi-pod --name cache redis

# El contenedor web puede acceder a redis en localhost:6379
$ podman exec web curl localhost:6379
```

### 8.4 Docker Desktop en macOS: ¿cómo funciona realmente?

Docker Desktop en macOS no ejecuta contenedores nativamente (macOS usa el kernel XNU, no Linux). En su lugar, ejecuta una máquina virtual Linux ligera donde corren los contenedores. El flujo es:

```
┌──────────────────────────────────────────────────────────┐
│                 DOCKER DESKTOP EN macOS                   │
│                                                           │
│  ┌─────────────────┐        ┌──────────────────────────┐ │
│  │   macOS (Host)   │        │  VM Linux (invitada)     │ │
│  │                  │        │                          │ │
│  │  Docker CLI ─────┼──REST──→ dockerd                  │ │
│  │                  │  unix  │  ↓                       │ │
│  │  /var/run/       │ socket │  containerd              │ │
│  │  docker.sock     │ (virt) │  ↓                       │ │
│  │                  │        │  containerd-shim         │ │
│  │  Navegador ──────┼──:80──→│  ↓                       │ │
│  │  localhost:8080  │  mapeo │  runc                    │ │
│  │                  │  puerto│  ↓                       │ │
│  │  ~/mi-app/ ──────┼─bind──→│  Contenedor (nginx)      │ │
│  │                  │  mount │                          │ │
│  └─────────────────┘        └──────────────────────────┘ │
│                                                           │
│  Hypervisor: Virtualization.framework (Apple)             │
│  o QEMU (versiones antiguas)                              │
└──────────────────────────────────────────────────────────┘
```

El socket de Docker (`/var/run/docker.sock`) que ves en macOS es en realidad un proxy que reenvía las peticiones al daemon dentro de la VM. El sistema de archivos se comparte mediante virtiofs o, en versiones antiguas, mediante gRPC FUSE (osxfs).

**Alternativas a Docker Desktop en macOS:**

| Herramienta        | Hipervisor          | Runtime          | Lo mejor de...                          |
|---------------------|---------------------|------------------|----------------------------------------|
| **Rancher Desktop** | Lima + QEMU/VZ     | containerd       | GUI completa, open source              |
| **Colima**          | Lima + QEMU/VZ     | Docker o containerd | Ligero, CLI, open source            |
| **Podman Desktop**  | Podman Machine     | Podman           | Pods nativos, rootless, open source     |
| **OrbStack**       | Propio (muy rápido) | Docker Engine   | Velocidad, bajo consumo de recursos    |

Para instalar Colima como reemplazo ligero de Docker Desktop en macOS:

```bash
# Instalar Colima y Docker CLI
$ brew install colima docker docker-compose

# Iniciar la VM
$ colima start --cpu 4 --memory 8 --disk 60

# Verificar
$ docker version
# Client: Docker Engine (macOS)
# Server: Docker Engine (Linux, dentro de la VM de Lima)

# Listo: puedes usar docker run, docker compose, etc.
$ docker run --rm hello-world
```

### 8.5 BuildKit: el constructor de imágenes moderno

BuildKit es el motor de construcción de imágenes de nueva generación. Reemplazó al builder antiguo de Docker y ofrece:

- **Construcción concurrente**: las etapas de un Dockerfile que no dependen entre sí se ejecutan en paralelo.
- **Caché avanzada**: caché remota en registries, montajes de caché para directorios como `~/.npm` o `~/.cache/pip`.
- **Secretos**: pasar secretos al build sin que queden en la imagen (`--secret`).
- **Salida múltiple**: exportar a imagen, tar, directorio local, o registry directamente.
- **Ejecución sin root ni daemon**: BuildKit puede ejecutarse como proceso independiente.

```bash
# Verificar que BuildKit está activo (por defecto desde Docker 23)
$ docker buildx version
github.com/docker/buildx v0.12.0

$ docker buildx ls
NAME/NODE      DRIVER/ENDPOINT  STATUS  PLATFORMS
default *      docker
  default      default          running linux/amd64, linux/arm64

# Construir con secretos (sin dejar secretos en la imagen)
$ docker build --secret id=aws_creds,src=$HOME/.aws/credentials -t myapp .

# Construir multi-plataforma
$ docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

En el Dockerfile, los secretos se montan temporalmente:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.11

# Montar secreto DURANTE la construcción (no queda en la imagen)
RUN --mount=type=secret,id=aws_creds \
    AWS_SHARED_CREDENTIALS_FILE=/run/secrets/aws_creds \
    pip install -r requirements.txt

# Montar caché para dependencias (acelera builds subsiguientes)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

---

## 9. Troubleshooting: conectando los conceptos

Cuando algo falla en Docker, la comprensión de los conceptos de este capítulo te permite diagnosticar con precisión. Aquí tienes una guía de mapeo síntoma → causa probable:

| Síntoma                                          | Causa probable                  | Diagnóstico                                     |
|--------------------------------------------------|--------------------------------|--------------------------------------------------|
| Contenedor muere con exit code 137               | OOM Killer (límite de memoria) | `dmesg \| grep -i oom`                          |
| Contenedor no puede escribir en disco             | Capa overlay llena o permisos  | `df -h` dentro del contenedor                    |
| Mapeo de puertos no funciona                      | iptables interferido por firewall | `iptables -t nat -L -n`                      |
| `docker run` se queda colgado                     | containerd o runc bloqueado    | `ctr --address /run/containerd/containerd.sock containers list` |
| El contenedor ve menos procesos de los esperados  | PID namespace limitado por pids.max | `cat /sys/fs/cgroup/pids/docker/<id>/pids.max` |
| El contenedor no puede escuchar en puerto <1024   | Falta capability NET_BIND_SERVICE | Añadir `--cap-add=NET_BIND_SERVICE`           |
| Error `cannot mount block device`              | Storage driver incorrecto o capa corrupta | `docker system prune -af`, revisar `docker info \| grep Storage` |
| Contenedor no ve un archivo del host              | No montado, o problema de propagación de mounts | `docker inspect --format '{{.Mounts}}' contenedor` |
| Proceso dentro del contenedor no recibe SIGTERM   | PID 1 no es el proceso real (script shell) | Usar `exec` en CMD o `["ejecutable"]`          |
| `cannot create container` por espacio en disco    | `overlay2` o `/var/lib/docker` lleno | `docker system df`, limpiar imágenes/containers |

**Script de diagnóstico rápido:**

```bash
#!/bin/bash
# guardar como docker-diag.sh
CID=$1
echo "=== DIAGNÓSTICO CONTENEDOR: $CID ==="

echo -e "\n--- Configuración de recursos (cgroups) ---"
docker inspect --format='
CPU:     {{.HostConfig.NanoCpus}} nanos
Memory:  {{.HostConfig.Memory}} bytes
CPUShares: {{.HostConfig.CpuShares}}
' $CID

echo -e "\n--- Namespaces (desde /proc) ---"
PID=$(docker inspect --format='{{.State.Pid}}' $CID)
if [ -n "$PID" ] && [ "$PID" != "0" ]; then
    echo "Host PID: $PID"
    ls -la /proc/$PID/ns/ 2>/dev/null || echo "No se puede acceder a /proc/$PID (¿permisos?)"
    echo -e "\n--- Límites de memoria (cgroup) ---"
    CGROUP_PATH=$(cat /proc/$PID/cgroup 2>/dev/null | grep memory | cut -d: -f3)
    if [ -n "$CGROUP_PATH" ]; then
        cat /sys/fs/cgroup/memory$CGROUP_PATH/memory.limit_in_bytes 2>/dev/null || \
        cat /sys/fs/cgroup$CGROUP_PATH/memory.max 2>/dev/null
    fi
fi

echo -e "\n--- Capas de almacenamiento ---"
docker inspect --format='{{json .GraphDriver.Data}}' $CID | jq '.'

echo -e "\n--- Últimas líneas de log ---"
docker logs --tail 20 $CID

echo -e "\n--- Estado de salud ---"
docker inspect --format='{{json .State.Health}}' $CID | jq '.'

echo -e "\n--- Puertos expuestos y mapeados ---"
docker port $CID
```

Este script aplica directamente los conceptos de namespaces (`/proc/PID/ns`), cgroups (`memory.limit_in_bytes`) y storage (`GraphDriver`) para diagnosticar problemas reales.

---

## 10. Resumen

En este capítulo hemos desmontado Docker desde sus cimientos:

1. **Un contenedor no es una VM.** Es un grupo de procesos aislados con restricciones de recursos. No hay kernel invitado, no hay hypervisor. Los procesos del contenedor son visibles desde el host.

2. **Los namespaces** (PID, NET, MNT, UTS, IPC, USER, CGROUP, TIME) son la tecnología del kernel que proporciona aislamiento. Cada namespace envuelve un recurso global del sistema, haciendo que los procesos dentro del namespace tengan su propia vista privada.

3. **Los cgroups** limitan recursos (CPU, memoria, I/O, procesos). Mientras los namespaces responden a "¿qué ven los procesos?", los cgroups responden a "¿cuánto pueden usar?".

4. **OverlayFS** y otros union filesystems permiten el sistema de capas de imágenes. Copy-on-write hace que compartir imágenes sea eficiente. Las capas inferiores son de solo lectura, los cambios van a la capa superior del contenedor.

5. **La arquitectura de Docker** es modular: CLI → dockerd → containerd → containerd-shim → runc. Cada componente tiene una responsabilidad clara y puede ser reemplazado. Incluye mecanismos de seguridad por capas: capabilities, seccomp, AppArmor/SELinux.

6. **La OCI** estandariza el formato de imágenes y el runtime de contenedores, garantizando interoperabilidad entre Docker, Podman, Kubernetes, y cualquier otra plataforma.

7. **La historia** de Docker es la historia de cómo una herramienta interna de una startup se convirtió en un estándar de la industria, pasando por la guerra de orquestación y la reestructuración hacia el desarrollo.

8. **El ecosistema** actual es rico y diverso: Docker para desarrollo, containerd/CRI-O para Kubernetes, Podman para entornos rootless, y muchas herramientas especializadas como BuildKit, Kaniko, Finch y Colima.

9. **El troubleshooting** efectivo requiere conectar los conceptos teóricos con herramientas de diagnóstico: `/proc`, `lsns`, `dmesg`, `iptables`, `docker inspect`.

En el próximo capítulo, pondremos manos a la obra con la instalación y configuración de Docker, crearemos nuestros primeros contenedores y exploraremos los comandos fundamentales.

---

## Ejercicios del Capítulo 1

1. **Comparación práctica VM vs Contenedor:** Crea una VM con VirtualBox/UTM (Ubuntu Server 22.04, 1GB RAM) y un contenedor con Docker (ubuntu:22.04). Mide el tiempo de arranque y el uso de RAM en ambos. Documenta los resultados.

2. **Namespaces en acción:** Ejecuta `docker run -d --name ns-demo alpine sleep 3600`. Usa `lsns` y `/proc/<pid>/ns/` para inspeccionar todos los namespaces del contenedor. Identifica el PID del contenedor en el host.

3. **NET namespace manual:** Sigue el ejemplo de la sección 2.3 para crear un NET namespace manual con `ip netns`, crear un veth pair, asignar IPs, y demostrar conectividad entre el namespace y el host.

4. **Crear un contenedor sin Docker:** Sigue el ejemplo de runc (sección 5.2) para crear y ejecutar un contenedor usando solo `runc`, sin Docker.

5. **Explorar OverlayFS:** Crea un overlay mount (sección 4.4) y demuestra el mecanismo de copy-on-write: crea un archivo en `lowerdir`, modifícalo a través del `merged`, verifica que el original permanece intacto y que la copia modificada está en `upperdir`.

6. **Cgroups en acción:** Limita un contenedor a 256MB de RAM y 0.5 CPUs. Usa `stress-ng` dentro del contenedor para intentar exceder los límites. Documenta qué sucede (OOM Killer, throttling de CPU).

7. **API REST de Docker:** Usa `curl --unix-socket` para listar imágenes, crear un contenedor, iniciarlo y obtener los logs — todo sin usar el comando `docker`.

8. **Explorar el ecosistema:** Instala Podman (o nerdctl) y reproduce los mismos comandos de Docker (`run`, `ps`, `images`, `logs`). Compara la experiencia y documenta las diferencias.

9. **Investigación:** ¿Qué combinación de namespaces, cgroups y capacidades de Linux crees que necesita un contenedor que ejecuta una base de datos (PostgreSQL o MySQL)? Justifica cada mecanismo en función de los requisitos de la base de datos.

10. **Dockerfile de una capa:** Crea una imagen Docker que tenga una sola capa (más allá de la capa base). Pista: usa `docker export` e `docker import`. Explica por qué las imágenes de múltiples capas son más eficientes.

### Pistas y soluciones parciales

**Ejercicio 1 — VM vs Contenedor:**
```bash
# Medir tiempo de arranque de contenedor
$ time docker run --rm ubuntu:22.04 echo "listo"
# Tiempo total vs tiempo de arranque puro: usa --rm -d para medir solo arranque

# Medir RAM con docker stats
$ docker run -d --name ram-test ubuntu:22.04 sleep 3600
$ docker stats --no-stream ram-test
# Columna MEM USAGE: memoria real usada, no la asignada
```

**Ejercicio 2 — Inspeccionar namespaces:**
```bash
# Obtener el PID real del contenedor
$ PID=$(docker inspect --format='{{.State.Pid}}' ns-demo)
$ echo $PID

# Listar namespaces del proceso
$ sudo ls -la /proc/$PID/ns/
# Cada symlink apunta a un namespace. El número entre [] es el inode.
# Compáralo con lsns | grep $PID
$ lsns -p $PID
```

**Ejercicio 4 — runc sin Docker:**
```bash
# El truco está en usar el rootfs de una imagen Docker existente
$ mkdir -p /tmp/runc-demo/rootfs
$ docker export $(docker create alpine:latest) | tar -C /tmp/runc-demo/rootfs -xvf -
$ cd /tmp/runc-demo
$ runc spec
# Edita config.json si es necesario (por ejemplo, cambia "terminal": true)
$ sudo runc run demo-container
```

**Ejercicio 5 — OverlayFS CoW:**
```bash
# Preparar directorios
$ mkdir -p /tmp/cow-demo/{lower,upper,work,merged}

# Crear archivo original e inmutable
$ echo "VERSION 1.0 - ORIGINAL" > /tmp/cow-demo/lower/app.conf
$ echo "config { debug = false }" > /tmp/cow-demo/lower/settings.ini

# Montar overlay
$ sudo mount -t overlay overlay \
    -o lowerdir=/tmp/cow-demo/lower,upperdir=/tmp/cow-demo/upper,workdir=/tmp/cow-demo/work \
    /tmp/cow-demo/merged

# Demostrar CoW
$ cat /tmp/cow-demo/merged/app.conf        # VE el original
$ echo "VERSION 2.0 - MODIFICADO" > /tmp/cow-demo/merged/app.conf  # MODIFICA
$ cat /tmp/cow-demo/lower/app.conf         # Original intacto: "VERSION 1.0 - ORIGINAL"
$ cat /tmp/cow-demo/upper/app.conf         # Copia en upper: "VERSION 2.0 - MODIFICADO"

# Demostrar eliminación (whiteout)
$ rm /tmp/cow-demo/merged/settings.ini
$ ls /tmp/cow-demo/lower/settings.ini      # Sigue existiendo
$ ls -la /tmp/cow-demo/upper/settings.ini  # Es un character device (0,0)
$ stat /tmp/cow-demo/upper/settings.ini | grep "Device type"
# Device type: 0,0
```

**Ejercicio 6 — Cgroups y stress:**
```bash
# Crear contenedor limitado
$ docker run -d --name stress-lab \
    --memory=256m --memory-swap=256m \
    --cpus=0.5 \
    alpine sleep 3600

# Instalar stress-ng
$ docker exec stress-lab apk add --no-cache stress-ng

# Probar límite de CPU (2 workers en 0.5 CPU = throttling)
$ docker exec stress-lab stress-ng --cpu 2 --timeout 10s --metrics-brief

# Monitorizar desde fuera
$ docker stats --no-stream stress-lab

# Probar límite de memoria (intentar usar 300MB, OOM Killer)
$ docker exec stress-lab stress-ng --vm 1 --vm-bytes 300M --timeout 10s
# El contenedor será matado. Verifica:
$ dmesg | grep -i "out of memory"
$ docker inspect stress-lab --format='{{.State.ExitCode}}'  # 137 = OOM killed
```

**Ejercicio 9 — Base de datos:**
```sql
-- Una base de datos necesita:
-- PID namespace: gestión de procesos (postmaster + workers)
-- NET namespace: puerto 5432/3306 aislado
-- MNT namespace: sistema de archivos para datos (/var/lib/postgresql/data)
-- IPC namespace: comunicación entre procesos internos
-- UTS namespace: hostname en logs
-- cgroups:
--   memory: límite para evitar OOM del host
--   cpu: shares altas para queries intensivas
--   blkio: prioritario si es una BD de producción
--   pids: limitar conexiones máximas
-- Capabilities:
--   CAP_CHOWN: postgres cambia dueño del data directory
--   CAP_SYS_RESOURCE: aumentar ulimits
--   CAP_SYS_NICE: ajustar prioridades
-- Riesgos:
--   CAP_SYS_ADMIN: NUNCA para un contenedor de BD
--   CAP_SYS_PTRACE: NUNCA salvo debugging
```

---

*En el próximo capítulo: Instalación de Docker, primeros comandos, y creación de nuestro primer entorno de desarrollo completo.*

---

[Inicio](README.md) | [Capítulo siguiente →](capitulo-02-instalacion.md)
