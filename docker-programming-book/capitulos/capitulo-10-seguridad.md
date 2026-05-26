# Capítulo 10: Seguridad en Contenedores Docker

> *"La seguridad no es un producto, es un proceso."* — Bruce Schneier

---

Los contenedores Docker revolucionaron el despliegue de software, pero trajeron consigo un nuevo modelo de seguridad que muchos equipos subestiman. La falsa sensación de aislamiento — "está en un contenedor, es seguro" — ha provocado brechas graves en producción. Este capítulo te llevará desde los fundamentos del modelo de amenazas en contenedores hasta un hardening completo, con exploits reales (con fines educativos) y sus mitigaciones.

---

## 10.1 El problema de seguridad en contenedores

### 10.1.1 Kernel compartido: el talón de Aquiles

A diferencia de las máquinas virtuales, que tienen su propio kernel invitado, **todos los contenedores en un host comparten el mismo kernel de Linux**. Esto significa que si un atacante logra escapar del contenedor y ejecutar código en el espacio del kernel, compromete **todos los contenedores** y el host completo.

```
┌─────────────────────────────────────────────────────────┐
│                     MÁQUINA VIRTUAL                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │
│  │ App A   │  │ App B   │  │ App C   │                 │
│  │ Bins/Lib│  │ Bins/Lib│  │ Bins/Lib│                 │
│  │Guest OS │  │Guest OS │  │Guest OS │   ← Kernels     │
│  └─────────┘  └─────────┘  └─────────┘     separados    │
│  Hypervisor                                              │
│  Host Kernel                                             │
│  Hardware                                                │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                     CONTENEDORES                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │
│  │ App A   │  │ App B   │  │ App C   │                 │
│  │ Bins/Lib│  │ Bins/Lib│  │ Bins/Lib│   ← Kernel     │
│  └─────────┘  └─────────┘  └─────────┘     compartido! │
│  Container Runtime (containerd/runc)                     │
│  Host Kernel (Linux) ← UN SOLO KERNEL                   │
│  Hardware                                                │
└─────────────────────────────────────────────────────────┘
```

**Demostración práctica: escape de contenedor vía kernel module**

```bash
# Terminal 1: Host
uname -r
# 5.15.0-91-generic

# Terminal 2: Contenedor privilegiado (simula lo que un atacante haría tras breakout)
docker run -it --privileged ubuntu:22.04 bash

# Dentro del contenedor, verificamos el kernel:
uname -r
# 5.15.0-91-generic  ← ¡El mismo kernel!

# ¿Qué puede hacer un contenedor privilegiado?
# Cargar un módulo de kernel — esto afecta al host entero
cat > /root/evil_module.c << 'KERNELCODE'
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/proc_fs.h>
#include <linux/cred.h>

static int __init evil_init(void) {
    printk(KERN_INFO "[EVIL] Modulo malicioso cargado - podriamos hacer rootkit aqui\n");
    return 0;
}

static void __exit evil_exit(void) {
    printk(KERN_INFO "[EVIL] Modulo descargado\n");
}

module_init(evil_init);
module_exit(evil_exit);
MODULE_LICENSE("GPL");
KERNELCODE

# Con --privileged, el contenedor puede cargar modulos en el kernel del HOST
# Esto es un game-over total: rootkit, keylogger, puertas traseras...
```

Este es el escenario mas extremo, pero incluso sin `--privileged`, existen vectores de ataque. La leccion fundamental: **el aislamiento de contenedores es mas debil que el de VMs por diseno**.

### 10.1.2 Imagenes de terceros: confias en el maintainer?

Cada vez que ejecutas `docker pull nginx:latest`, estas descargando codigo arbitrario que se ejecutara en tu infraestructura. El modelo de confianza es implicito pero fragil:

```bash
# Esto descarga y ejecuta codigo de un desconocido
docker pull someuser/someimage:latest
docker run someuser/someimage:latest

# Que puede hacer esa imagen?
# - Minar criptomonedas (cryptojacking)
# - Exfiltrar variables de entorno (AWS_ACCESS_KEY_ID...)
# - Actuar como pivot para atacar tu red interna
# - Instalar un reverse shell
```

**Caso real — cryptojacking en Docker Hub (2021)**:
Se descubrieron mas de 30 imagenes maliciosas en Docker Hub que minaban Monero. Habian sido descargadas mas de 20 millones de veces. Los atacantes usaban nombres como `alpine`, `nginx`, `mysql` con typosquatting.

```bash
# Inspeccionemos una imagen sospechosa
docker history suspicious-image:latest
# IMAGE          CREATED BY
# <missing>      /bin/sh -c apt-get update && apt-get install -y xmrig
# <missing>      /bin/sh -c echo "minando monero en segundo plano..."

# El entrypoint puede ser cualquier cosa
docker inspect suspicious-image:latest | jq '.[].Config.Entrypoint'
# ["/usr/local/bin/miner-start.sh"]
```

### 10.1.3 Superficie de ataque

La superficie de ataque de una instalacion Docker tipica incluye:

| Componente | Riesgo | Vector de ataque |
|---|---|---|
| **Docker Daemon** (dockerd) | Ejecuta como root | Socket Unix expuesto, TCP expuesto |
| **Socket Docker** (`/var/run/docker.sock`) | Equivale a root en el host | Montarlo en contenedores |
| **Imagenes base** | Vulnerabilidades sin parchear | CVEs en librerias del sistema |
| **Runtime (runc, containerd)** | CVEs de breakout | CVE-2019-5736, CVE-2024-21626 |
| **Red Docker** | Exposicion no intencionada | Puertos mapeados, redes bridge |
| **Capabilities por defecto** | Demasiados privilegios | Root en contenedor con caps innecesarias |

**El socket de Docker: la llave del reino**

```bash
# Montar el socket de Docker en un contenedor es peligrosisimo
docker run -it -v /var/run/docker.sock:/var/run/docker.sock alpine sh

# Dentro del contenedor, instalamos docker CLI
apk add docker-cli

# Ahora podemos controlar el host entero:
docker run -it -v /:/host alpine sh
# Estamos en el host, con acceso completo al filesystem
cat /host/etc/shadow   # Hashes de contrasenas
cat /host/etc/passwd   # Usuarios del host

# O lanzar un contenedor privilegiado que es game-over
docker run -it --privileged --pid=host alpine nsenter -t 1 -m -u -n -i sh
# Ahora estamos en el namespace del host, como root
```

**Regla de oro: NUNCA montes `/var/run/docker.sock` dentro de un contenedor a menos que sea absolutamente necesario y entiendas el riesgo.** Herramientas como Portainer o Traefik lo necesitan — pero deben ejecutarse con las maximas restricciones posibles.

### 10.1.4 CIA Triad aplicado a contenedores

La triada CIA (Confidentiality, Integrity, Availability) es el modelo clasico de seguridad de la informacion. Aplicado a contenedores:

**Confidencialidad** — Quien puede leer que?
- Secretos en variables de entorno: accesibles via `docker inspect` y `/proc/<pid>/environ`
- Imagenes con datos sensibles incrustados en capas
- Redes bridge por defecto: todos los contenedores pueden comunicarse entre si
- Filesystem expuesto: un `docker cp` puede extraer datos

**Integridad** — Podemos confiar en que el codigo no fue modificado?
- Imagenes sin firmar: cualquiera puede subir `nginx:latest` falsificado
- Registries sin autenticacion
- Contenedores con filesystem escribible: malware puede persistir
- Pipelines CI/CD sin verificacion de integridad de imagenes

**Disponibilidad** — El servicio sigue funcionando bajo ataque?
- Sin limites de recursos: un contenedor puede consumir toda la RAM/CPU
- Fork bombs: un proceso puede colapsar el host
- Ataques de denegacion de servicio entre contenedores

---

## 10.2 Principio de minimo privilegio

El principio de minimo privilegio establece que **cada proceso debe tener exactamente los permisos que necesita para funcionar, y nada mas**. En Docker, esto significa:

### 10.2.1 No ejecutar como root dentro del contenedor

Por defecto, Docker ejecuta el proceso del contenedor como **root** (UID 0). Aunque el root dentro del contenedor no es exactamente igual al root del host, sigue siendo innecesariamente peligroso.

**Demostracion: los peligros de ejecutar como root**

```dockerfile
# Dockerfile INSEGURO — ejecuta como root
FROM node:18-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
# UID = 0 (root) por defecto — peligroso!
```

Veamos que puede hacer este contenedor:

```bash
# Construir y ejecutar
docker build -t insecure-app .
docker run -it --rm insecure-app sh

# Dentro: somos root
whoami
# root
id
# uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),...

# Podemos instalar paquetes (si hay red)
apt-get update && apt-get install -y nmap curl

# Podemos cambiar configuraciones del sistema
echo 1 > /proc/sys/net/ipv4/ip_forward
# Podemos cambiar iptables
iptables -L

# Podemos bindear puertos privilegiados (<1024)
nc -l -p 80

# Si hay una vulnerabilidad en la app (RCE), el atacante obtiene root
# directamente, y desde ahi puede intentar breakout
```

**Dockerfile CORREGIDO — usuario no-root**

```dockerfile
# Dockerfile SEGURO — usuario no-root
FROM node:18-alpine

# Crear usuario y grupo de aplicacion
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Asegurar ownership correcto ANTES de copiar
COPY --chown=appuser:appgroup package.json package-lock.json ./
RUN npm ci --only=production

COPY --chown=appuser:appgroup . .

# Cambiar a usuario no-root
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
# Verifiquemos el UID
docker build -t secure-app .
docker run -it --rm secure-app sh

whoami
# appuser
id
# uid=1000(appuser) gid=1000(appgroup)

# Ya no podemos instalar paquetes
apt-get update
# E: Could not open lock file /var/lib/apt/lists/lock - open (13: Permission denied)

# No podemos escribir en /proc
echo 1 > /proc/sys/net/ipv4/ip_forward
# sh: can't create /proc/sys/net/ipv4/ip_forward: Permission denied

# No podemos bindear puertos < 1024 (sin NET_BIND_SERVICE cap)
nc -l -p 80
# nc: Permission denied
```

### 10.2.2 Crear usuario en Dockerfile: opciones por distribucion

**Alpine Linux** (recomendado por minimalista):

```dockerfile
FROM alpine:3.19
RUN addgroup -S app && adduser -S app -G app
USER app
# UID/GID por defecto: 1000/1000
```

**Debian/Ubuntu**:

```dockerfile
FROM ubuntu:22.04
RUN groupadd -r app && useradd -r -g app -m -s /bin/bash app
USER app
```

**Red Hat UBI / CentOS / Fedora**:

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi
RUN groupadd -r app && useradd -r -g app -m -d /home/app app
USER app
```

**Especificando UID/GID explicito** (mejor practica para consistencia):

```dockerfile
FROM alpine:3.19
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
USER 1001
```

### 10.2.3 UID/GID y volumenes: el infierno de los permisos

Un problema comun con usuarios no-root es la discrepancia de UIDs entre el host y el contenedor:

```bash
# El contenedor escribe como UID 1001
# Pero en el host, el directorio del volumen pertenece a root (UID 0)
docker run -v /host/data:/app/data secure-app

# Resultado: Permission denied al escribir
```

**Soluciones:**

```bash
# Opcion 1: Chown en el host
sudo chown -R 1001:1001 /host/data

# Opcion 2: Usar el UID del host en el contenedor
docker run -u $(id -u):$(id -g) -v /host/data:/app/data secure-app

# Opcion 3: Inicializacion con entrypoint que ajusta permisos
# (script entrypoint que chown antes de ejecutar la app)

# Opcion 4: Usar --userns=remap (user namespace remapping)
# Configurar en /etc/docker/daemon.json:
{
  "userns-remap": "default"
}
# Esto mapea UID 0 en contenedor -> UID no privilegiado en host
```

### 10.2.4 docker run --user — override en runtime

Puedes cambiar el usuario al arrancar el contenedor:

```bash
# Ejecutar como UID 1000 especifico
docker run --user 1000:1000 myimage

# Ejecutar como usuario existente en el contenedor
docker run --user nobody myimage

# Pero si necesitas capabilities especificas, mejor definir USER en Dockerfile
```

### 10.2.5 Capabilities: root en contenedor NO es root en el host

Aqui hay una confusion comun: **cuando ejecutas un proceso como UID 0 dentro de un contenedor, NO tienes todos los poderes del root del host**. Docker, por defecto, limita las capabilities del contenedor a un subconjunto seguro.

```bash
# Comparativa: root en el host vs root en contenedor
# En el HOST como root real:
capsh --print
# Current: =ep
# Bounding set =cap_chown,cap_dac_override,...,cap_sys_admin,...,cap_sys_time,...
# (TODAS ~40 capabilities)

# En un CONTENEDOR como root:
docker run -it alpine sh
capsh --print 2>/dev/null || cat /proc/1/status | grep Cap
# Solo 14 capabilities por defecto
# NO incluye: CAP_SYS_ADMIN, CAP_NET_ADMIN, CAP_SYS_PTRACE, CAP_SYSLOG, etc.
```

---

## 10.3 Linux Capabilities

### 10.3.1 Que son las capabilities?

Tradicionalmente, en Unix/Linux hay dos tipos de procesos: privilegiados (UID 0, root, que puede hacer todo) y no privilegiados (el resto, sujetos a controles de permisos). Esto es binario y burdo.

Desde Linux 2.2 (1999), el kernel divide los privilegios de root en **~40 capabilities atomicas e independientes**. Un proceso puede tener unas si y otras no, permitiendo granularidad fina.

```bash
# Ver todas las capabilities disponibles
man capabilities

# Lista completa en el sistema
cat /usr/include/linux/capability.h 2>/dev/null | grep '#define CAP_' || \
capsh --print 2>/dev/null
```

### 10.3.2 Las ~40 capabilities de Linux (lista completa)

| Capability | Proposito | Riesgo si se abusa |
|---|---|---|
| `CAP_CHOWN` | Cambiar owner de archivos | Escalar privilegios cambiando owner de binarios |
| `CAP_DAC_OVERRIDE` | Ignorar permisos de lectura/escritura | Leer cualquier archivo |
| `CAP_DAC_READ_SEARCH` | Ignorar permisos de lectura/directorios | Leer cualquier archivo |
| `CAP_FOWNER` | Ignorar restricciones de owner en operaciones | Modificar archivos de otros |
| `CAP_FSETID` | No limpiar bits setuid/setgid | Escalacion via binarios setuid |
| `CAP_KILL` | Enviar senales a cualquier proceso | Matar procesos de otros contenedores |
| `CAP_SETGID` | Cambiar GID arbitrariamente | Suplantar grupos |
| `CAP_SETUID` | Cambiar UID arbitrariamente | Suplantar usuarios |
| `CAP_SETPCAP` | Transferir capabilities | Escalar privilegios en otros procesos |
| `CAP_NET_BIND_SERVICE` | Bindear puertos < 1024 | Suplantar servicios del sistema |
| `CAP_NET_BROADCAST` | Broadcast en sockets | Usar broadcast de red |
| `CAP_NET_ADMIN` | Administracion de red (iptables, rutas) | Manipular red del host |
| `CAP_NET_RAW` | Sockets raw (ping, sniffing) | Sniffing de trafico |
| `CAP_IPC_LOCK` | Bloquear paginas en memoria (mlock) | DoS por agotamiento de RAM |
| `CAP_IPC_OWNER` | Ignorar permisos en System V IPC | Acceder a memoria compartida de otros |
| `CAP_SYS_MODULE` | Cargar/descargar modulos de kernel | Rootkit, kernel compromise |
| `CAP_SYS_RAWIO` | Acceso directo a I/O, puertos | Comprometer hardware |
| `CAP_SYS_CHROOT` | Usar chroot() | Escape de chroot |
| `CAP_SYS_PTRACE` | Trazear procesos arbitrarios | Leer memoria de otros procesos |
| `CAP_SYS_PACCT` | Activar accounting de procesos | Informacion sobre otros procesos |
| `CAP_SYS_ADMIN` | **Super-capability** (mil operaciones) | Practicamente root total |
| `CAP_SYS_BOOT` | Reiniciar el sistema | Denegacion de servicio |
| `CAP_SYS_NICE` | Cambiar prioridad de procesos | Starvation de otros procesos |
| `CAP_SYS_RESOURCE` | Exceder limites de recursos | DoS por consumo |
| `CAP_SYS_TIME` | Cambiar reloj del sistema | Corromper logs, certificados |
| `CAP_SYS_TTY_CONFIG` | Configurar TTYs | Keylogging |
| `CAP_MKNOD` | Crear device nodes | Acceso directo a dispositivos |
| `CAP_LEASE` | Establecer leases en archivos | Bloquear archivos |
| `CAP_AUDIT_WRITE` | Escribir registros de auditoria | Falsear logs |
| `CAP_AUDIT_CONTROL` | Controlar subsistema de auditoria | Deshabilitar auditoria |
| `CAP_SETFCAP` | Establecer capabilities en archivos | Persistir capabilities |
| `CAP_MAC_OVERRIDE` | Ignorar Mandatory Access Control | Burlar AppArmor/SELinux |
| `CAP_MAC_ADMIN` | Cambiar configuracion MAC | Deshabilitar AppArmor/SELinux |
| `CAP_SYSLOG` | Leer buffer del kernel (dmesg) | Fugas de informacion |
| `CAP_WAKE_ALARM` | Programar wakeups del sistema | DoS de bateria |
| `CAP_BLOCK_SUSPEND` | Bloquear suspension del sistema | DoS de energia |
| `CAP_PERFMON` | Usar subsistema de performance | Leer datos de rendimiento |
| `CAP_BPF` | Cargar programas BPF | Manipular trafico de red |
| `CAP_CHECKPOINT_RESTORE` | Checkpoint/restore de procesos | Secuestrar procesos |

### 10.3.3 Docker y capabilities: el conjunto por defecto

Docker, por diseno, **dropea capabilities peligrosas** del contenedor incluso cuando ejecutas como root. El conjunto por defecto que Docker otorga es:

```bash
# Ver las capabilities por defecto de Docker
docker run --rm alpine sh -c 'apk add -q libcap 2>/dev/null; capsh --print'

# O desde el host, inspeccionando /proc/<PID>/status:
CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' mycontainer)
cat /proc/$CONTAINER_PID/status | grep Cap
```

**Las 14 capabilities por defecto en Docker**:

| # | Capability | Por que se incluye? |
|---|---|---|
| 1 | `CAP_CHOWN` | Permite `chown` en el filesystem del contenedor |
| 2 | `CAP_DAC_OVERRIDE` | Ignorar permisos de archivos (necesario para muchas apps) |
| 3 | `CAP_FSETID` | Modificar archivos sin limpiar setuid |
| 4 | `CAP_FOWNER` | Operaciones que requieren ser owner |
| 5 | `CAP_MKNOD` | Crear device nodes (necesario para /dev/) |
| 6 | `CAP_NET_RAW` | Sockets raw (ping funciona en el contenedor) |
| 7 | `CAP_SETGID` | Cambiar de grupo |
| 8 | `CAP_SETUID` | Cambiar de usuario |
| 9 | `CAP_SETFCAP` | Establecer capabilities en archivos |
| 10 | `CAP_SETPCAP` | Transferir capabilities |
| 11 | `CAP_NET_BIND_SERVICE` | Bindear puertos < 1024 |
| 12 | `CAP_SYS_CHROOT` | Usar chroot (necesario en algunos setups) |
| 13 | `CAP_KILL` | Enviar senales a procesos |
| 14 | `CAP_AUDIT_WRITE` | Escribir al sistema de auditoria |

**Las capabilities PELIGROSAS que Docker NO otorga por defecto**:

```bash
# Estas NO estan disponibles en un contenedor normal:
# CAP_SYS_ADMIN      <- La mas peligrosa: montar filesystems, administrar namespaces
# CAP_NET_ADMIN       <- Configurar iptables, enrutamiento
# CAP_SYS_PTRACE      <- Trazear otros procesos
# CAP_SYSLOG          <- Leer dmesg
# CAP_SYS_MODULE      <- Cargar modulos de kernel
# CAP_SYS_RAWIO       <- Acceso directo a hardware
# CAP_SYS_BOOT        <- Reiniciar el sistema
# CAP_SYS_TIME        <- Cambiar la hora del sistema
# CAP_MAC_ADMIN       <- Modificar AppArmor/SELinux
# CAP_MAC_OVERRIDE    <- Burlar AppArmor/SELinux
```

### 10.3.4 Restringir aun mas: --cap-drop y --cap-add

La buena practica es **dropear TODAS las capabilities por defecto y anadir solo las que tu aplicacion realmente necesita**.

```bash
# Maxima restriccion: dropear todo, anadir solo lo necesario
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# Verificar que solo tiene esa capability
docker run --rm --cap-drop=ALL --cap-add=NET_BIND_SERVICE alpine \
  sh -c 'apk add -q libcap 2>/dev/null; capsh --print'
```

**Que capabilities necesita cada tipo de aplicacion:**

| Tipo de aplicacion | Capabilities necesarias | Comando |
|---|---|---|
| **App web** (Node, Python, Go) | Solo `NET_BIND_SERVICE` si usa puerto < 1024. Si usa puerto > 1024, NINGUNA! | `--cap-drop=ALL --cap-add=NET_BIND_SERVICE` |
| **API REST en puerto 8080** | **NINGUNA** (el puerto alto no requiere caps) | `--cap-drop=ALL` |
| **Agente de monitoreo** | `SYS_PTRACE` para leer /proc de otros | `--cap-drop=ALL --cap-add=SYS_PTRACE` |
| **Servidor de base de datos** | `IPC_LOCK` para locked memory, `SYS_NICE` para prioridad I/O | `--cap-drop=ALL --cap-add=IPC_LOCK --cap-add=SYS_NICE` |
| **Nginx reverse proxy** (puerto 80/443) | `NET_BIND_SERVICE` (y `CHOWN`, `DAC_OVERRIDE` si escribe logs) | `--cap-drop=ALL --cap-add=NET_BIND_SERVICE --cap-add=CHOWN --cap-add=DAC_OVERRIDE` |
| **Proceso batch** (sin red) | **NINGUNA** | `--cap-drop=ALL --network=none` |
| **Build container** | Multiples, mejor usar BuildKit | No ejecutar builds en contenedores de runtime |

### 10.3.5 Ejemplo concreto: hardening de capabilities paso a paso

**Paso 1: Identificar que capabilities usa realmente tu aplicacion**

```bash
# Iniciar contenedor con strace y todas las caps para ver que syscalls usa
docker run -d --name app-test --cap-add=ALL myapp
docker exec app-test strace -p 1 -e trace=capability 2>&1 | tee caps-needed.log
# Buscar operaciones que requieran capabilities especificas
```

**Paso 2: Empezar con todo dropeado y anadir gradualmente**

```bash
# Empezar sin nada
docker run --cap-drop=ALL myapp
# Si falla, el error indicara que falta. Ejemplo:
# "Permission denied" al bindear puerto 80 -> necesita NET_BIND_SERVICE

# Anadir una capability
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
# Si funciona, perfecto! Si no, anadir la siguiente necesaria
```

**Paso 3: Documentar y codificar en Docker Compose / Kubernetes**

```yaml
# docker-compose.yml con capabilities restrictivas
services:
  api:
    image: myapp:latest
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    ports:
      - "80:3000"
    read_only: true
    user: "1001:1001"
```

```yaml
# Kubernetes Pod con SecurityContext
apiVersion: v1
kind: Pod
metadata:
  name: secure-api
spec:
  containers:
  - name: api
    image: myapp:latest
    securityContext:
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1001
      allowPrivilegeEscalation: false
```

### 10.3.6 Verificar capabilities de un proceso en ejecucion

```bash
# Metodo 1: Desde el host con getpcaps (parte de libcap-ng-utils)
# Obtener PID del contenedor
PID=$(docker inspect -f '{{.State.Pid}}' mycontainer)
getpcaps $PID
# Output: Capabilities for '12345': = cap_chown,cap_dac_override,...,cap_net_bind_service+ep

# Metodo 2: Leyendo /proc/<PID>/status
cat /proc/$PID/status | grep -i cap
# CapInh: 0000000000000000
# CapPrm: 00000000a80425fb
# CapEff: 00000000a80425fb
# CapBnd: 00000000a80425fb
# CapAmb: 0000000000000000

# Metodo 3: Decodificar el bitmap de capabilities
# La mascara de bits 00000000a80425fb en hexadecimal se puede decodificar:
python3 << 'EOF'
caps_bits = {
    0: "CHOWN", 1: "DAC_OVERRIDE", 2: "DAC_READ_SEARCH", 3: "FOWNER",
    4: "FSETID", 5: "KILL", 6: "SETGID", 7: "SETUID",
    8: "SETPCAP", 9: "NET_BIND_SERVICE", 10: "NET_BROADCAST", 11: "NET_ADMIN",
    12: "NET_RAW", 13: "IPC_LOCK", 14: "IPC_OWNER", 15: "SYS_MODULE",
    16: "SYS_RAWIO", 17: "SYS_CHROOT", 18: "SYS_PTRACE", 19: "SYS_PACCT",
    20: "SYS_ADMIN", 21: "SYS_BOOT", 22: "SYS_NICE", 23: "SYS_RESOURCE",
    24: "SYS_TIME", 25: "SYS_TTY_CONFIG", 26: "MKNOD", 27: "LEASE",
    28: "AUDIT_WRITE", 29: "AUDIT_CONTROL", 30: "SETFCAP", 31: "MAC_OVERRIDE",
    32: "MAC_ADMIN", 33: "SYSLOG", 34: "WAKE_ALARM", 35: "BLOCK_SUSPEND",
    36: "AUDIT_READ", 37: "PERFMON", 38: "BPF", 39: "CHECKPOINT_RESTORE"
}

hex_mask = "00000000a80425fb"
mask_int = int(hex_mask, 16)
print("Capabilities efectivas:")
for bit, name in caps_bits.items():
    if mask_int & (1 << bit):
        print(f"  CAP_{name}")
EOF
```

---

## 10.4 Seccomp (Secure Computing Mode)

### 10.4.1 Que es seccomp?

Seccomp (Secure Computing Mode) es un mecanismo del kernel Linux que permite **filtrar las system calls (syscalls)** que un proceso puede invocar. Es como un firewall para syscalls: reduces la superficie del kernel expuesta al minimo indispensable.

Piensa en ello asi:
- **Namespaces**: aislan lo que el proceso *ve* (recursos)
- **Cgroups**: limitan lo que el proceso *consume* (recursos)
- **Capabilities**: restringen lo que root puede *hacer* (privilegios)
- **Seccomp**: controla que syscalls puede *llamar* (interfaz con el kernel)

```
┌──────────────────────────────────────────────────┐
│                 APLICACION                        │
│                                                    │
│  syscall: read, write, open, connect, ...         │
│              ↓                                     │
│  ┌─────────────────────────┐                      │
│  │   SECCOMP FILTER        │ ← Bloquea aqui      │
│  │   (allowlist/denylist)  │   syscalls no        │
│  └────────────┬────────────┘   permitidas          │
│               ↓ (syscall permitida)                │
│  ┌─────────────────────────┐                      │
│  │   KERNEL DE LINUX       │                      │
│  └─────────────────────────┘                      │
└──────────────────────────────────────────────────┘
```

### 10.4.2 Perfil seccomp por defecto de Docker

Docker incluye un perfil seccomp por defecto que **bloquea ~44 syscalls consideradas peligrosas o innecesarias** para la mayoria de las aplicaciones en contenedores. Este perfil se aplica automaticamente a todos los contenedores (a menos que lo deshabilites).

```bash
# Ver el perfil por defecto que Docker esta usando
# El perfil esta embebido en el codigo de containerd/runc
# Puedes verlo aqui (en el host):
wget -qO- https://raw.githubusercontent.com/moby/moby/master/profiles/seccomp/default.json | python3 -m json.tool | head -50
```

**Syscalls bloqueadas por el perfil por defecto de Docker** (lista parcial):

| Syscall | Por que se bloquea | Riesgo |
|---|---|---|
| `reboot` | Reiniciar sistema | Denegacion de servicio |
| `kexec_load` | Cargar nuevo kernel | Compromiso total del host |
| `kexec_file_load` | Cargar kernel desde archivo | Compromiso total del host |
| `mount` | Montar filesystems | Escape de contenedor, pivot |
| `umount2` | Desmontar filesystems | Inestabilidad |
| `pivot_root` | Cambiar root del sistema | Escape de contenedor |
| `nfsservctl` | Llamadas NFS del kernel | Compartir filesystem no deseado |
| `setdomainname` | Cambiar nombre de dominio | Suplantacion |
| `sethostname` | Cambiar hostname | Suplantacion |
| `settimeofday` | Cambiar hora del sistema | Corromper logs y TLS |
| `stime` | Cambiar hora (obsoleto) | Corromper logs y TLS |
| `create_module` | Crear modulos kernel | Rootkit |
| `init_module` | Inicializar modulos kernel | Rootkit |
| `delete_module` | Eliminar modulos kernel | Persistencia de ataque |
| `swapon`/`swapoff` | Activar/desactivar swap | Manipulacion de memoria |
| `clock_settime` | Cambiar relojes del sistema | Corromper datos temporales |
| `acct` | Activar accounting | Fuga de informacion |
| `add_key` | Anadir claves al kernel | Persistencia no deseada |
| `request_key` | Solicitar claves del kernel | Fuga de secretos |
| `keyctl` | Operaciones de keyring | Manipulacion de secretos |
| `bpf` | Cargar programas BPF | Manipulacion de red |
| `perf_event_open` | Monitoreo de performance | Fuga de informacion |
| `ptrace` | Trazear procesos | Robo de datos de otros procesos |
| `personality` | Cambiar personalidad de ejecucion | Ejecutar binarios extranjeros |

### 10.4.3 Crear un perfil seccomp personalizado

El perfil por defecto es un buen punto de partida, pero puedes crear un perfil mas restrictivo que solo permita las syscalls que tu aplicacion necesita.

**Estructura de un perfil seccomp JSON:**

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_AARCH64"
  ],
  "syscalls": [
    {
      "names": [
        "read",
        "write",
        "open",
        "close",
        "fstat",
        "mmap",
        "mprotect",
        "munmap",
        "brk",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "ioctl",
        "pread64",
        "pwrite64",
        "sched_getaffinity",
        "set_tid_address",
        "set_robust_list",
        "getrlimit",
        "prlimit64",
        "arch_prctl",
        "futex",
        "nanosleep",
        "clock_gettime",
        "epoll_create1",
        "epoll_ctl",
        "epoll_wait",
        "epoll_pwait",
        "socket",
        "bind",
        "listen",
        "accept",
        "accept4",
        "connect",
        "getsockname",
        "getpeername",
        "setsockopt",
        "getsockopt",
        "sendto",
        "recvfrom",
        "sendmsg",
        "recvmsg",
        "shutdown",
        "fcntl",
        "clone",
        "execve",
        "exit",
        "exit_group",
        "wait4",
        "kill",
        "uname",
        "getpid",
        "gettid",
        "getppid",
        "getuid",
        "geteuid",
        "getgid",
        "getegid",
        "tgkill",
        "sched_yield",
        "readlink",
        "stat",
        "lstat",
        "access",
        "getdents64",
        "getrandom",
        "madvise",
        "sysinfo",
        "statfs",
        "getcwd",
        "chdir",
        "openat",
        "newfstatat",
        "restart_syscall"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

**Perfil minimalista para un servidor web Node.js tipico:**

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    {
      "names": [
        "accept", "accept4", "access", "arch_prctl", "bind",
        "brk", "chdir", "clock_gettime", "clone", "close",
        "connect", "epoll_create1", "epoll_ctl", "epoll_pwait",
        "epoll_wait", "execve", "exit", "exit_group", "fchmod",
        "fchown", "fcntl", "fstat", "fsync", "ftruncate",
        "futex", "getcwd", "getdents64", "getegid", "geteuid",
        "getgid", "getpeername", "getpid", "getppid", "getrandom",
        "getrlimit", "getsockname", "getsockopt", "gettid",
        "getuid", "ioctl", "kill", "listen", "lseek", "lstat",
        "madvise", "mkdir", "mmap", "mprotect", "munmap",
        "nanosleep", "newfstatat", "openat", "pipe2",
        "pread64", "prlimit64", "pwrite64", "read", "readlink",
        "recvfrom", "recvmsg", "rename", "restart_syscall",
        "rmdir", "rt_sigaction", "rt_sigprocmask", "rt_sigreturn",
        "sched_getaffinity", "sched_yield", "sendfile", "sendmsg",
        "sendto", "set_robust_list", "set_tid_address", "setsockopt",
        "shutdown", "sigaltstack", "socket", "stat", "statfs",
        "sysinfo", "tgkill", "uname", "unlink", "utimensat",
        "wait4", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### 10.4.4 Aplicar un perfil seccomp

```bash
# Guardar el perfil como archivo
# Aplicarlo al ejecutar el contenedor
docker run --security-opt seccomp=/path/to/profile.json myapp

# Verificar que perfil esta activo (desde el host)
docker inspect mycontainer | jq '.[].HostConfig.SecurityOpt'
# ["seccomp=/path/to/profile.json"]
```

### 10.4.5 Deshabilitar seccomp (NO recomendado)

```bash
# Deshabilitar completamente el filtrado seccomp
docker run --security-opt seccomp=unconfined myapp

# Esto expone TODAS las syscalls del kernel
# Solo justificable en debugging o contenedores altamente confiables
```

**Cuando podrias necesitar `--security-opt seccomp=unconfined`?** Casi nunca. Algunas herramientas de debugging como `strace`, `perf`, `gdb` pueden necesitar syscalls bloqueadas. En esos casos, mejor crear un perfil especifico que anada solo las syscalls necesarias en lugar de deshabilitar todo.

### 10.4.6 Depuracion de bloqueos de seccomp

```bash
# Si tu aplicacion falla misteriosamente en un contenedor, seccomp puede ser la causa
# Verifica con strace (requiere SYS_PTRACE o unconfined temporal)
docker run --rm -it --security-opt seccomp=unconfined alpine sh -c '
  apk add strace
  strace -f your-binary
'

# En el host, puedes ver syscalls bloqueadas por seccomp en los logs del kernel:
dmesg | grep -i seccomp
# audit: type=1326 audit(...): auid=1000 ... syscall=165 comm="your-app"
# Buscar el numero de syscall en la tabla de tu arquitectura
```

---

## 10.5 AppArmor / SELinux — Mandatory Access Control (MAC)

Mientras que seccomp filtra syscalls y capabilities restringen lo que root puede hacer, **AppArmor y SELinux** anaden una capa adicional: **Mandatory Access Control (MAC)**. Son sistemas que el administrador configura y que el usuario no puede deshabilitar (a diferencia de DAC — Discretionary Access Control, que son los permisos Unix tradicionales).

### 10.5.1 AppArmor (Ubuntu, Debian, SUSE)

AppArmor funciona con **perfiles** que definen que archivos, capacidades y recursos de red puede acceder cada aplicacion.

```bash
# Verificar si AppArmor esta activo
aa-status

# Ver perfiles cargados
ls /etc/apparmor.d/

# Ver el perfil de Docker
cat /etc/apparmor.d/docker-default 2>/dev/null || \
  echo "AppArmor esta instalado pero docker-default puede estar en otro lugar"
```

**Perfil por defecto de Docker (`docker-default`):**

Cuando ejecutas un contenedor, Docker aplica automaticamente el perfil `docker-default` de AppArmor. Este perfil:

- Bloquea el montaje de filesystems
- Niega acceso a `/proc/sysrq-trigger`, `/proc/irq`, `/proc/bus`
- Restringe `ptrace` a solo el proceso mismo
- Bloquea `pivot_root`
- Niega acceso a capabilities peligrosas

```bash
# Verificar el perfil activo de un contenedor
docker inspect mycontainer | jq '.[].AppArmorProfile'
# "docker-default"

# Ejecutar con un perfil AppArmor personalizado
docker run --security-opt apparmor=my-custom-profile myapp

# Deshabilitar AppArmor para un contenedor (NO recomendado)
docker run --security-opt apparmor=unconfined myapp
```

**Crear un perfil AppArmor personalizado para tu aplicacion:**

```bash
# /etc/apparmor.d/containers/docker-nginx
cat > /etc/apparmor.d/containers/docker-nginx << 'APPARMOR'
#include <tunables/global>

profile docker-nginx flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # Permitir acceso a binarios de nginx
  /usr/sbin/nginx mr,
  /usr/sbin/nginx-debug mr,

  # Permitir lectura de configuracion
  /etc/nginx/** r,

  # Permitir escritura de logs
  /var/log/nginx/*.log w,

  # Permitir acceso a archivos de la web
  /usr/share/nginx/html/** r,
  /var/www/** r,

  # Denegar acceso a todo lo demas
  deny /** w,
  deny /root/** rwxl,
  deny /home/** rwxl,

  # Networking
  network inet stream,
  network inet6 stream,

  # Denegar capabilities peligrosas
  deny @{PROC}/sys/kernel/shm* wxl,
}
APPARMOR

# Cargar el perfil
apparmor_parser -r /etc/apparmor.d/containers/docker-nginx

# Usarlo
docker run --security-opt apparmor=docker-nginx nginx
```

### 10.5.2 SELinux (RHEL, CentOS, Fedora, Amazon Linux)

SELinux funciona con **etiquetas de seguridad** (contextos) que cada archivo, proceso y recurso tiene. Las politicas definen que interacciones estan permitidas entre etiquetas.

```bash
# Verificar estado de SELinux
sestatus
# SELinux status:                 enabled
# Current mode:                   enforcing

# Ver etiquetas de un contenedor
docker inspect mycontainer | jq '.[].ProcessLabel'
# "system_u:system_r:container_t:s0:c1,c2"

# Etiquetado de archivos en el host para volumenes
ls -Z /var/lib/docker/volumes/
# system_u:object_r:container_var_lib_t:s0
```

**Usar SELinux con Docker:**

```bash
# Habilitar SELinux en el daemon de Docker
# /etc/docker/daemon.json
{
  "selinux-enabled": true,
  "userns-remap": "default"
}

# Al montar volumenes, usar :Z o :z para reetiquetar
docker run -v /host/data:/app/data:Z myapp
# :Z = etiqueta privada (solo este contenedor)
# :z = etiqueta compartida (multiples contenedores)
```

**Crear una politica SELinux personalizada para un contenedor:**

```bash
# Generar modulo de politica para un contenedor especifico
mkdir -p /tmp/selinux-policy
cd /tmp/selinux-policy

cat > mycontainer.te << 'SELINUX'
policy_module(mycontainer, 1.0)

# Tipo para el proceso del contenedor
virt_sandbox_domain(mycontainer_t)
# Permitir acceso a archivos etiquetados como container_file_t
allow mycontainer_t container_file_t:file { read write open getattr };
# Permitir bind a puertos tcp
allow mycontainer_t self:tcp_socket { bind create listen accept };
# Permitir acceso a red
allow mycontainer_t self:netlink_route_socket { create bind read write };
SELINUX

# Compilar e instalar
make -f /usr/share/selinux/devel/Makefile mycontainer.pp
semodule -i mycontainer.pp

# Ejecutar contenedor con el contexto personalizado
docker run --security-opt label=type:mycontainer_t myapp
```

### 10.5.3 Comparativa AppArmor vs SELinux

| Aspecto | AppArmor | SELinux |
|---|---|---|
| **Distribuciones** | Ubuntu, Debian, SUSE, OpenSUSE | RHEL, CentOS, Fedora, Amazon Linux |
| **Modelo** | Basado en rutas (path-based) | Basado en etiquetas (label-based) |
| **Complejidad** | Mas simple, mas facil de aprender | Mas complejo, curva de aprendizaje alta |
| **Granularidad** | Por archivo/ruta | Por tipo/contexto (mas granular) |
| **Rendimiento** | Menor overhead | Mayor overhead |
| **Configuracion** | Archivos de perfil en `/etc/apparmor.d/` | Politicas compiladas, booleanos |
| **Modo aprendizaje** | `aa-complain` | `permissive` mode |
| **Docker default** | Perfil `docker-default` | `container_t` con transiciones automaticas |
| **Madurez MLS/MCS** | No | Si (Multi-Level Security) |

**Recomendacion practica**: Si estas en Ubuntu/Debian, usa AppArmor. Si estas en RHEL/CentOS, usa SELinux. Ambos proporcionan un nivel excelente de MAC cuando se configuran correctamente. No intentes cambiar el sistema MAC de tu distribucion; trabaja con el.

---

## 10.6 Rootless Docker

### 10.6.1 Que es Rootless Docker?

Rootless Docker permite ejecutar **el demonio de Docker y los contenedores sin privilegios de root en el host**. Cada usuario puede tener su propio daemon Docker ejecutandose completamente en espacio de usuario, sin acceso a root en el sistema.

Esto se logra mediante:
- **User namespaces**: mapean el UID 0 dentro del contenedor a un UID no privilegiado en el host
- **FUSE (Filesystem in Userspace)** para el storage driver (en lugar de overlay2 con kernel)
- **slirp4netns** para networking sin privilegios (en lugar de iptables/veth)

### 10.6.2 Ventajas de Rootless Docker

1. **Elimina el riesgo de escalacion a root en el host**: Si un atacante explota runc/containerd y escapa del contenedor, aterriza como un usuario no privilegiado, no como root.
2. **Aislamiento entre usuarios**: Cada usuario tiene su propio daemon, sus propias imagenes y contenedores, sin interferencia.
3. **Ideal para entornos multi-tenant**: CI/CD, trabajo compartido, entornos educativos.
4. **Compatible con Kubernetes**: Puedes ejecutar kind (Kubernetes in Docker) en rootless.

### 10.6.3 Instalacion de Rootless Docker

```bash
# Requisitos previos (Ubuntu/Debian)
# newuidmap y newgidmap deben estar instalados
sudo apt-get install -y uidmap dbus-user-session

# Si ya tienes Docker instalado, deten el servicio de sistema
sudo systemctl disable --now docker.service docker.socket
# O simplemente no uses sudo con rootless

# Descargar e instalar el script rootless
# (Requiere que el binario de Docker ya este instalado)
dockerd-rootless-setuptool.sh install

# Respuesta tipica:
# [INFO] Creating /home/user/.local/share/docker
# [INFO] systemd unit file: /home/user/.config/systemd/user/docker.service
# [INFO] Installed docker.service successfully
# [INFO] To control docker, run: systemctl --user start docker

# Iniciar el servicio rootless
systemctl --user start docker
systemctl --user enable docker

# Configurar variables de entorno
export DOCKER_HOST=unix:///run/user/1000/docker.sock
# Agregar a ~/.bashrc o ~/.zshrc para persistencia:
echo 'export DOCKER_HOST=unix:///run/user/1000/docker.sock' >> ~/.bashrc

# Verificar
docker version
# Server: Docker Engine - Community
#  Version: 24.0.x
#  ...
# rootless: true

docker info | grep -i rootless
# rootless: true

docker run hello-world
# Hello from Docker!
```

### 10.6.4 Arquitectura interna de Rootless Docker

```
┌──────────────────────────────────────────────────┐
│              HOST (root)                          │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │         USER NAMESPACE (uid=1000)             │ │
│  │                                                │ │
│  │  ┌──────────────────────┐                     │ │
│  │  │ dockerd (user)       │                     │ │
│  │  │ containerd (user)    │                     │ │
│  │  │ runc (user)          │                     │ │
│  │  └──────────────────────┘                     │ │
│  │           │                                    │ │
│  │  ┌────────┴──────────┐                       │ │
│  │  │   containers      │                       │ │
│  │  │  (UID 0 dentro    │                       │ │
│  │  │   -> UID 1000+    │                       │ │
│  │  │    en el host)    │                       │ │
│  │  └───────────────────┘                       │ │
│  │                                                │ │
│  │  Storage: fuse-overlayfs (userspace)          │ │
│  │  Network:  slirp4netns (userspace TCP/IP)     │ │
│  │  DNS:      built-in resolver                   │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
│  Kernel: overlayfs, netfilter (NO USADO)           │
└──────────────────────────────────────────────────┘
```

### 10.6.5 Limitaciones de Rootless Docker

| Caracteristica | Disponible en Rootless | Notas |
|---|---|---|
| `docker run` basico | Si | Funcionalidad completa |
| `docker build` | Si | BuildKit en modo rootless |
| `docker compose` | Si | Requiere `docker-compose` o `docker compose` |
| `--network host` | **No** | No se puede compartir la red del host sin root |
| Ping (ICMP) | **No** | Requiere `CAP_NET_RAW` y sockets raw |
| Puertos < 1024 | **No** sin configuracion | `/etc/sysctl.conf`: `net.ipv4.ip_unprivileged_port_start=80` |
| `overlay2` storage driver | **No** | Usa `fuse-overlayfs` en su lugar |
| `--privileged` | **No** | Obviamente incompatible con rootless |
| Acceso a dispositivos | Limitado | Solo dispositivos con permisos de usuario |
| `docker0` bridge | **No** | Usa slirp4netns (NAT en userspace) |
| `cgroups v1` | Parcial | cgroups v2 recomendado |

```bash
# Habilitar puertos bajos sin root (Linux 5.4+)
sudo sysctl net.ipv4.ip_unprivileged_port_start=80

# Para ping, usar en su lugar: curl, wget, o netcat
# No es ideal para health checks, pero funcional
```

### 10.6.6 Rootless vs Rootful: tabla de decision

| Caso de uso | Rootless? |
|---|---|
| Desarrollo local | **Si** — recomendado |
| CI/CD (GitHub Actions, GitLab CI) | **Si** — recomendado si el runner lo soporta |
| Entornos multi-tenant | **Si** — imprescindible |
| Produccion en servidor single | **No** — usa rootful con hardening |
| Kubernetes nodes | **No** — usa containerd/CRI-O con hardening |
| Herramientas de red avanzadas | **No** — necesitas host network, ping |
| Demos y educacion | **Si** — ideal para seguridad |

---

## 10.7 Escaneo de Vulnerabilidades de Imagenes

### 10.7.1 Por que escanear imagenes?

Las imagenes de contenedores son como una caja negra: heredan todas las vulnerabilidades del sistema operativo base, las librerias del sistema, los paquetes instalados, y las dependencias de la aplicacion. Segun estudios recientes:

- El **60-80%** de las imagenes en Docker Hub contienen al menos una vulnerabilidad conocida.
- Una imagen `node:latest` puede tener **mas de 400 CVEs** (la mayoria en el SO base).
- Las vulnerabilidades CRITICAL permiten en muchos casos RCE (Remote Code Execution).

### 10.7.2 Docker Scout

Docker Scout es la herramienta integrada de Docker para analisis de vulnerabilidades, disponible en Docker Desktop y Docker Hub.

```bash
# Requiere Docker Desktop o login en Docker Hub
docker login

# Escaneo rapido de una imagen local
docker scout quickview nginx:latest
# Output:
#   nginx:latest
#     SBOM of image already cached, 253 packages indexed
#     4 vulnerabilities

# Analisis detallado
docker scout cves nginx:latest
# Muestra cada CVE con severidad, paquete afectado, version fix

# Comparar dos versiones
docker scout compare nginx:1.25 nginx:1.26

# Recomendaciones de actualizacion
docker scout recommendations nginx:latest
# Sugerencias para cambiar a versiones sin vulnerabilidades
```

**Docker Scout en CI/CD (GitHub Actions):**

```yaml
# .github/workflows/docker-scout.yml
name: Docker Scout Security Scan

on:
  push:
    branches: [main]
  pull_request:

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Docker Scout
        uses: docker/scout-action@v1
        with:
          command: cves
          image: myapp:${{ github.sha }}
          only-fixed: true
          ignore-base: false
          # Falla si hay CRITICAL
          exit-code: true
          severity-cutoff: critical
```

### 10.7.3 Trivy (Aqua Security)

**Trivy** es el escaner open source mas popular. Es rapido, integral y escanea tanto el SO como las dependencias de aplicacion (npm, pip, maven, gem, etc.).

```bash
# Instalacion (Linux/macOS)
# Opcion 1: curl
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# Opcion 2: brew (macOS)
brew install trivy

# Opcion 3: Docker
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/Library/Caches:/root/.cache/ aquasec/trivy image nginx:latest
```

**Escaneo de imagenes:**

```bash
# Escaneo basico
trivy image nginx:latest
# Output:
# nginx:latest (debian 12.4)
# Total: 4 (UNKNOWN: 0, LOW: 0, MEDIUM: 2, HIGH: 2, CRITICAL: 0)

# Solo mostrar CRITICAL y HIGH
trivy image --severity CRITICAL,HIGH nginx:latest

# Salida en formato tabla con detalles
trivy image --format table nginx:latest

# Salida en JSON para automatizacion
trivy image --format json -o trivy-report.json nginx:latest

# Salida SARIF para GitHub Code Scanning
trivy image --format sarif -o trivy-results.sarif nginx:latest

# Escanear archivos .tar de imagenes
docker save nginx:latest -o nginx.tar
trivy image --input nginx.tar

# Ignorar vulnerabilidades sin fix disponible
trivy image --ignore-unfixed nginx:latest

# Filtrar por tipo de vulnerabilidad
trivy image --vuln-type os,library nginx:latest
```

**Interpretacion de resultados de Trivy:**

```
nginx:latest (debian 12.4)
============================
Total: 142 (UNKNOWN: 0, LOW: 68, MEDIUM: 54, HIGH: 18, CRITICAL: 2)

┌──────────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────────────┬──────────────────────────────────────────┐
│   Library    │ Vulnerability  │ Severity │ Status │ Installed Version │    Fixed Version      │                 Title                    │
├──────────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────────────┼──────────────────────────────────────────┤
│ libssl3      │ CVE-2024-0727  │ CRITICAL │ fixed  │ 3.0.11-1~deb12u2 │ 3.0.13-1~deb12u1      │ openssl: denial of service via null...   │
│ libcrypto3   │ CVE-2024-0727  │ CRITICAL │ fixed  │ 3.0.11-1~deb12u2 │ 3.0.13-1~deb12u1      │ openssl: denial of service via null...   │
│ libnghttp2-14│ CVE-2023-44487 │ HIGH     │ fixed  │ 1.52.0-1         │ 1.52.0-1+deb12u1      │ HTTP/2 Rapid Reset Attack                 │
└──────────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────────────┴──────────────────────────────────────────┘
```

**Columna Severity**: CRITICAL > HIGH > MEDIUM > LOW > UNKNOWN

**Columna Status**:
- `fixed`: Hay una version que corrige la vulnerabilidad — ACTUALIZA YA.
- `will_not_fix`: El vendor no planea corregirla — evalua mitigaciones alternativas.
- `end_of_life`: El software ya no recibe soporte — CAMBIA DE IMAGEN BASE.

### 10.7.4 Trivy en CI/CD — bloquear builds con vulnerabilidades criticas

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on: [push, pull_request]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:ci-test .

      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:ci-test'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'
          # Falla el build si encuentra CRITICAL o HIGH sin fix

      - name: Upload scan results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: trivy-results
          path: trivy-results.json
```

```yaml
# GitLab CI con Trivy
trivy-scan:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image --severity CRITICAL,HIGH --exit-code 1 --no-progress myapp:$CI_COMMIT_SHA
  allow_failure: false  # Bloquear el pipeline si falla
```

### 10.7.5 Snyk (docker scan)

```bash
# docker scan usa Snyk en el backend
# Requiere Docker Desktop (incluido) o instalar plugin
docker scan nginx:latest

# Output:
# Testing nginx:latest...
# Low severity vulnerability found in libgnutls30
#   Description: ...
#   Info: https://snyk.io/vuln/SNYK-DEBIAN12-GNUTLS28-...
#   Introduced through: libgnutls30@3.8.0-1
#   From: libgnutls30@3.8.0-1
#   Fixed in: 3.8.3-1

# Escanear e ignorar unfixed
docker scan --accept-license --dependency-tree --exclude-base myapp:latest

# JSON output
docker scan --json nginx:latest > scan-results.json

# En CI con Snyk CLI standalone
npm install -g snyk
snyk auth
snyk container test nginx:latest
```

### 10.7.6 Clair (Red Hat / Quay)

Clair es el escaner open source desarrollado por Quay/CoreOS. Es mas complejo de configurar ya que requiere PostgreSQL, pero ofrece analisis estatico muy completo.

```bash
# Clair se ejecuta tipicamente como parte de Quay Registry
# Para usarlo standalone con Docker:

# docker-compose.yml para Clair + PostgreSQL
cat > docker-compose-clair.yml << 'YAML'
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: clair
      POSTGRES_USER: clair
      POSTGRES_PASSWORD: clairpass
    volumes:
      - pgdata:/var/lib/postgresql/data

  clair:
    image: quay.io/projectquay/clair:latest
    ports:
      - "6060:6060"
      - "6061:6061"
    environment:
      CLAIR_MODE: combo
      CLAIR_CONF: /clair/config.yaml
    volumes:
      - ./clair-config.yaml:/clair/config.yaml
    depends_on:
      - postgres

volumes:
  pgdata:
YAML

# Escanear con clairctl
clairctl analyze nginx:latest
clairctl report nginx:latest
```

### 10.7.7 Automatizar el escaneo y mantenerse actualizado

**Monitorizacion continua con Renovate:**

```json
// renovate.json — actualiza imagenes base automaticamente
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ],
  "dockerfile": {
    "enabled": true,
    "fileMatch": ["(^|/)Dockerfile$", "(^|/)Dockerfile\\.[^/]*$"]
  },
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "matchDatasources": ["docker"],
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "labels": ["security-major-update"]
    }
  ]
}
```

**Dependabot para Docker:**

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 5
    labels:
      - "dependencies"
      - "security"
    allow:
      - dependency-type: "direct"
    ignore:
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]
```

### 10.7.8 Comparativa de escaneres

| Herramienta | Tipo | Velocidad | SO + App Deps | CI/CD | Precio |
|---|---|---|---|---|---|
| **Trivy** | Open Source | Muy rapida | Si (completo) | Facil (GitHub Action) | Gratis |
| **Docker Scout** | Comercial/Free tier | Rapida | Si | GitHub Action | Freemium |
| **Snyk** | Comercial | Media | Si (excelente) | Nativo | Freemium |
| **Clair** | Open Source | Lenta (config requerida) | SO + limitado | Manual/API | Gratis |
| **Grype** (Anchore) | Open Source | Rapida | Si | GitHub Action | Gratis |
| **AWS ECR Scan** | Integrado AWS | Rapida | SO | Automatico | Por uso |
| **GCP Artifact Analysis** | Integrado GCP | Rapida | SO + lenguajes | Automatico | Por uso |

---

## 10.8 Content Trust / Firma de imagenes

### 10.8.1 El problema: es esta imagen realmente lo que dice ser?

Cuando haces `docker pull nginx:latest`, estas confiando en:
1. Que Docker Hub te da la imagen correcta (no un MITM)
2. Que la imagen no fue manipulada en el registry
3. Que el maintainer es quien dice ser

La **firma de imagenes** resuelve estos tres problemas mediante criptografia asimetrica.

### 10.8.2 Docker Content Trust (DCT)

DCT usa **The Update Framework (TUF)** y **Notary** para firmar y verificar imagenes. Al activarlo, Docker verifica que cada imagen que descargas este firmada por una clave confiable.

```bash
# Activar Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Ahora, docker pull verifica firmas
docker pull library/nginx:latest
# Si la imagen no esta firmada (o la firma no es valida):
# Error: remote trust data does not exist for docker.io/library/nginx:
#   notary.docker.io does not have trust data

# Si esta firmada correctamente, el pull funciona normalmente

# Para hacer permanente:
echo 'export DOCKER_CONTENT_TRUST=1' >> ~/.bashrc

# Firmar y subir tu propia imagen (requiere DCT activado)
docker trust key generate mykey
docker trust signer add --key mykey.pub myname docker.io/myuser/myimage
docker trust sign docker.io/myuser/myimage:latest
docker push docker.io/myuser/myimage:latest
```

**Gestion de claves con Notary:**

```bash
# Listar signers de una imagen
docker trust inspect --pretty library/nginx:latest

# Output:
# Signatures for library/nginx:latest
# SIGNED TAG   DIGEST                                     SIGNERS
# latest       1234567890abcdef...                        maintainer1, maintainer2

# Revocar confianza
docker trust revoke docker.io/myuser/myimage:latest
```

### 10.8.3 Cosign (Sigstore) — firma sin servidor central

A diferencia de DCT/Notary que requiere un servidor Notary, **Cosign** (parte del proyecto Sigstore) firma imagenes de forma descentralizada, almacenando la firma directamente en el registry OCI.

```bash
# Instalacion
# macOS
brew install cosign
# Linux
curl -O -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign-linux-amd64 && sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# Generar par de claves
cosign generate-key-pair
# Crea cosign.key (privada) y cosign.pub (publica)
# Se te pedira una contrasena para proteger la clave privada

# Firmar una imagen
cosign sign --key cosign.key myregistry.com/myimage:latest

# Verificar una imagen
cosign verify --key cosign.pub myregistry.com/myimage:latest
# Output:
# Verification for myregistry.com/myimage:latest --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key

# Firmar con OIDC (keyless, usando identidad de GitHub/Gmail)
cosign sign myregistry.com/myimage:latest

# Verificar con keyless (la identidad del firmante)
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity "https://github.com/myorg/myrepo/.github/workflows/build.yml@refs/heads/main" \
  myregistry.com/myimage:latest
```

**Cosign en CI/CD (GitHub Actions):**

```yaml
# .github/workflows/build-and-sign.yml
name: Build and Sign

on:
  push:
    branches: [main]

jobs:
  build-sign:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Necesario para OIDC keyless signing
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/myorg/myimage:${{ github.sha }}

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign image with keyless
        run: |
          cosign sign \
            --yes \
            ghcr.io/myorg/myimage:${{ github.sha }}

      - name: Verify signature
        run: |
          cosign verify \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            --certificate-identity "https://github.com/${{ github.repository }}/.github/workflows/build-and-sign.yml@${{ github.ref }}" \
            ghcr.io/myorg/myimage:${{ github.sha }}
```

### 10.8.4 Verificacion de firmas en despliegue (Kubernetes + Cosign)

```yaml
# Kubernetes Admission Controller con Cosign (via Kyverno o Connaisseur)
# Politica de Kyverno que verifica firmas Cosign:

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-cosign-signatures
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: check-image-signature
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - imageReferences:
        - "ghcr.io/myorg/*"
        attestors:
        - entries:
          - keys:
              publicKeys: |-
                -----BEGIN PUBLIC KEY-----
                MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                -----END PUBLIC KEY-----
```

### 10.8.5 Por que firmar tus imagenes

1. **Integridad**: Garantiza que la imagen no fue manipulada entre el build y el deploy.
2. **Autenticidad**: Prueba quien construyo la imagen (util en compliance/SOC2).
3. **Supply chain security**: Si un atacante compromete tu registry, no puede inyectar imagenes maliciosas sin la clave de firma.
4. **Cumplimiento**: Muchas regulaciones (FedRAMP, HIPAA, PCI-DSS) requieren verificacion de integridad de software.

---

## 10.9 Gestion de Secretos

### 10.9.1 El anti-patron: secretos en el Dockerfile

NUNCA pongas secretos en un Dockerfile. Aunque uses `ARG` (que supuestamente es solo build-time), los valores quedan en el historial de la imagen.

**Demostracion del problema:**

```dockerfile
# Dockerfile INSEGURO
FROM alpine:3.19

# NUNCA HAGAS ESTO!
ENV DATABASE_PASSWORD=SuperSecret123
# O peor aun:
ARG API_KEY=sk-1234567890abcdef

RUN echo "La API key es $API_KEY" > /tmp/config
CMD ["cat", "/tmp/config"]
```

```bash
# Construir la imagen
docker build -t insecure-secrets .

# El secreto queda expuesto en el historial
docker history insecure-secrets
# IMAGE          CREATED BY
# <missing>      /bin/sh -c echo "La API key es sk-1234567890abcdef" > /tmp/config
# <missing>      |2 API_KEY=sk-1234567890abcdef /bin/sh -c ...
# <missing>      ENV DATABASE_PASSWORD=SuperSecret123

# Incluso sin --no-trunc, cualquiera con acceso a la imagen puede verlo
docker history --no-trunc insecure-secrets

# Y el secreto tambien esta en las capas de la imagen
docker save insecure-secrets -o insecure.tar
mkdir /tmp/extract && tar -xf insecure.tar -C /tmp/extract
# Buscar en todas las capas
grep -r "SuperSecret123" /tmp/extract/
# Encontrado!
```

### 10.9.2 El anti-patron: secretos en variables de entorno visibles

```bash
# Esto es inseguro: docker inspect expone las variables de entorno
docker run -d -e API_KEY=sk-1234567890abcdef --name insecure-app myapp
docker inspect insecure-app | jq '.[].Config.Env'
# [
#   "API_KEY=sk-1234567890abcdef",
#   "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
# ]

# Tambien visible en /proc/<PID>/environ (si tienes acceso al PID)
# Y en docker-compose logs, etc.
```

### 10.9.3 BuildKit Secrets — la forma correcta de usar secretos en el build

BuildKit (el builder moderno de Docker) permite montar secretos durante el build sin que queden en la imagen final.

```dockerfile
# Dockerfile SEGURO con BuildKit secrets
# Sintaxis: # syntax=docker/dockerfile:1
FROM node:18-alpine

WORKDIR /app
COPY package.json package-lock.json ./

# El secreto se monta en /run/secrets/npmrc durante el build SOLAMENTE
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --only=production

COPY . .
USER 1001
EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
# Construir con el secreto proporcionado en tiempo de build
# El secreto NO queda en la imagen
export DOCKER_BUILDKIT=1

# Crear el archivo .npmrc con el token (fuera del contexto de build)
echo "//registry.npmjs.org/:_authToken=npm_xxxxxxxxxxxx" > /tmp/npmrc

# Build con el secreto
docker build \
  --secret id=npmrc,src=/tmp/npmrc \
  -t myapp:secure .

# Verificar que el secreto NO esta en la imagen
docker history myapp:secure | grep -i token
# (No deberia encontrar nada)
```

**Otros ejemplos de BuildKit secrets:**

```dockerfile
# Usar SSH key privada para clonar repos
RUN --mount=type=ssh git clone git@github.com:org/private-repo.git

# Usar secreto para instalar dependencias privadas de pip
RUN --mount=type=secret,id=pip-conf,target=/etc/pip.conf \
    pip install -r requirements.txt

# Usar secreto para descargar de AWS S3
RUN --mount=type=secret,id=aws-creds,target=/root/.aws/credentials \
    aws s3 cp s3://bucket/config.yaml /app/config.yaml

# Build con SSH agent
docker build --ssh default --secret id=aws-creds,src=$HOME/.aws/credentials -t myapp .
```

### 10.9.4 Docker Secrets (Swarm)

En Docker Swarm, los secretos se almacenan encriptados en el Raft log y se montan como archivos temporales (tmpfs) en `/run/secrets/`.

```bash
# Crear un secreto (desde stdin)
echo "MySuperSecretPassword123" | docker secret create db_password -

# Crear un secreto desde archivo
docker secret create db_password ./password.txt

# Listar secretos
docker secret ls

# Usar el secreto en un servicio
docker service create \
  --name web \
  --secret db_password \
  --secret api_key \
  nginx

# El secreto esta disponible dentro del contenedor como archivo:
docker exec $(docker ps -q -f name=web) cat /run/secrets/db_password
# MySuperSecretPassword123
```

**Docker Compose con secrets:**

```yaml
# docker-compose.yml (modo Swarm o Compose v3+)
version: '3.8'

services:
  api:
    image: myapp:latest
    secrets:
      - db_password
      - api_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
    # La app lee el secreto del archivo, no de variable de entorno

secrets:
  db_password:
    external: true  # Creado con docker secret create
  api_key:
    file: ./secrets/api_key.txt  # Desde archivo local
```

### 10.9.5 HashiCorp Vault

Vault es el estandar de la industria para gestion de secretos. Ofrece secretos dinamicos, rotacion automatica, auditoria, y multiples backends de autenticacion.

```bash
# Iniciar Vault en desarrollo (NO para produccion)
docker run -d --name vault-dev -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=root \
  vault:latest

# Configurar Vault
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

# Almacenar un secreto
vault kv put secret/myapp/db username=admin password=SuperSecret123

# Leer un secreto
vault kv get secret/myapp/db
# Output:
# === Data ===
# Key        Value
# ---        -----
# username   admin
# password   SuperSecret123

# Generar credenciales dinamicas de base de datos (PostgreSQL)
# Vault crea un usuario temporal con TTL, lo rota automaticamente
vault secrets enable database

vault write database/config/postgres \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db:5432/mydb" \
  allowed_roles="readonly" \
  username="vault_admin" \
  password="vault_admin_password"

vault write database/roles/readonly \
  db_name=postgres \
  creation_statements="CREATE USER \"{{name}}\" WITH PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Obtener credenciales dinamicas
vault read database/creds/readonly
# username: v-token-readonly-xyz
# password: A1b2C3d4...
# Lease duration: 1h (se revocan automaticamente)
```

**Integrar la aplicacion con Vault:**

```python
# Python app usando Vault para obtener secretos
import hvac  # pip install hvac

client = hvac.Client(url='http://vault:8200')

# Autenticacion por Kubernetes Service Account
client.auth.kubernetes.login(
    role='myapp-role',
    jwt=open('/var/run/secrets/kubernetes.io/serviceaccount/token').read()
)

# Leer secreto
secret = client.secrets.kv.v2.read_secret_version(path='myapp/db')
db_password = secret['data']['data']['password']

# O para credenciales dinamicas de BD:
creds = client.secrets.database.generate_credentials(name='readonly')
db_user = creds['data']['username']
db_password = creds['data']['password']
```

### 10.9.6 SOPS (Secrets OPerationS)

SOPS (de Mozilla) permite **encriptar archivos de secretos y guardarlos en Git de forma segura**. Soporta AWS KMS, GCP KMS, Azure Key Vault, PGP, y age.

```bash
# Instalacion
brew install sops  # macOS
# o: go install go.mozilla.org/sops/v3/cmd/sops@latest

# Crear un archivo de secretos encriptado con age (recomendado)
# Primero, generar clave age
age-keygen -o key.txt
# Public key: age1ql3z7hjy54p3...

# Encriptar un archivo YAML
cat > secrets.yaml << 'EOF'
database:
  host: db.internal
  port: 5432
  username: admin
  password: SuperSecret123
api:
  key: sk-abcdefghijklmnop
EOF

sops --encrypt --age age1ql3z7hjy54p3... secrets.yaml > secrets.enc.yaml

# El archivo encriptado se puede guardar en Git
# Contenido de secrets.enc.yaml:
cat secrets.enc.yaml
# database:
#     host: ENC[AES256_GCM,data:...]
#     password: ENC[AES256_GCM,data:...]
# sops:
#     kms: []
#     age:
#         - recipient: age1ql3z7hjy54p3...
#           enc: |
#             -----BEGIN AGE ENCRYPTED FILE-----
#             ...
#             -----END AGE ENCRYPTED FILE-----
#     ...

# Desencriptar en CI/CD (solo quien tenga la clave privada)
export SOPS_AGE_KEY_FILE=/path/to/key.txt
sops --decrypt secrets.enc.yaml > secrets.yaml

# Usar en Kubernetes con sops+kustomize o Helm secrets
```

### 10.9.7 Sealed Secrets (Kubernetes)

Sealed Secrets permite **guardar secretos de Kubernetes encriptados en Git**. Solo el controlador Sealed Secrets en el cluster puede desencriptarlos.

```bash
# Instalar kubeseal
brew install kubeseal  # macOS

# Instalar el controlador en el cluster
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# Crear un secreto normal y sellarlo
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123 \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-secret.yaml

# El archivo sealed-secret.yaml esta encriptado y es seguro para Git
cat sealed-secret.yaml
# apiVersion: bitnami.com/v1alpha1
# kind: SealedSecret
# metadata:
#   name: db-credentials
# spec:
#   encryptedData:
#     username: AgBy3x8...
#     password: AgCk1p2...

# Guardar en Git
git add sealed-secret.yaml
git commit -m "Add encrypted database credentials"

# Aplicar en el cluster (el controlador lo desencripta automaticamente)
kubectl apply -f sealed-secret.yaml

# Verificar que se creo el secreto
kubectl get secret db-credentials -o yaml
# El secreto existe desencriptado en el cluster
```

### 10.9.8 Tabla comparativa de opciones de secretos

| Metodo | Entorno | Seguridad | Rotacion | Complejidad |
|---|---|---|---|---|
| **BuildKit Secrets** | Build-time | Alta (no persiste) | N/A (build-time) | Baja |
| **Docker Secrets** | Swarm | Alta (Raft encriptado) | Manual | Baja |
| **Kubernetes Secrets** | K8s | Media (etcd base64) | Manual | Baja |
| **Vault** | Cualquiera | Muy alta (HSM) | Automatica | Alta |
| **SOPS** | GitOps | Alta (KMS/age) | Manual (re-encriptar) | Media |
| **Sealed Secrets** | K8s + Git | Alta (cluster-side decrypt) | Manual | Media |
| **AWS Secrets Manager** | AWS | Alta (KMS) | Automatica posible | Baja-Media |
| **GCP Secret Manager** | GCP | Alta (KMS) | Automatica posible | Baja-Media |
| **Variables ENV** | Cualquiera | **MUY BAJA** | N/A | Trivial |

---

## 10.10 Filesystem de Solo Lectura

### 10.10.1 docker run --read-only

Ejecutar contenedores con filesystem raiz de solo lectura es una de las medidas de hardening mas efectivas y simples. Bloquea una de las tacticas mas comunes post-explotacion: escribir binarios maliciosos, modificar configuraciones, o establecer persistencia.

```bash
# Contenedor con filesystem inmutable
docker run --read-only nginx

# Pero Nginx necesita escribir en ciertos directorios
# La solucion: tmpfs para directorios especificos
docker run --read-only \
  --tmpfs /var/cache/nginx \
  --tmpfs /var/run \
  nginx

# Verificar
docker exec mynginx touch /tmp/test
# touch: /tmp/test: Read-only file system
docker exec mynginx touch /var/cache/nginx/test
# (funciona — /var/cache/nginx esta en tmpfs)
```

### 10.10.2 Directorios que tipicamente necesitan escritura

| Directorio | Proposito | Solucion |
|---|---|---|
| `/tmp` | Archivos temporales | `--tmpfs /tmp` |
| `/var/run` | PID files, sockets | `--tmpfs /var/run` |
| `/var/log` | Logs de aplicacion | `--tmpfs /var/log` o stdout/stderr |
| `/var/cache` | Cache de aplicacion | Volume o tmpfs |
| `/var/lib/app/data` | Datos persistentes | Volume nombrado |
| `/dev/shm` | Memoria compartida | Ya es tmpfs por defecto |

### 10.10.3 Ejemplo completo: Nginx con filesystem read-only

```bash
# docker-compose.yml con read-only optimizado para Nginx
cat > docker-compose-readonly.yml << 'YAML'
version: '3.8'

services:
  nginx:
    image: nginx:1.25-alpine
    read_only: true
    tmpfs:
      - /var/cache/nginx
      - /var/run
      - /tmp
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro  # Config tambien read-only
      - ./html:/usr/share/nginx/html:ro          # Contenido estatico read-only
    ports:
      - "80:80"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    user: "101:101"  # UID de nginx en Alpine
YAML
```

### 10.10.4 Kubernetes: readOnlyRootFilesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1001
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/app
    - name: config
      mountPath: /etc/app/config.yaml
      subPath: config.yaml
      readOnly: true
  volumes:
  - name: tmp
    emptyDir:
      medium: Memory    # tmpfs en RAM
      sizeLimit: 128Mi
  - name: cache
    emptyDir: {}
  - name: config
    configMap:
      name: app-config
```

### 10.10.5 Casos de uso ideales para read-only

- **APIs REST/GraphQL stateless**: No necesitan estado local
- **Workers de colas**: Procesan jobs y envian respuestas
- **Procesos batch**: Leen entrada, procesan, escriben salida a volumen/red
- **Servidores web estaticos**: Solo sirven archivos
- **Proxies y API Gateways**: Nginx, HAProxy, Envoy
- **Funciones (FaaS/serverless)**: Efimeras por definicion

---

## 10.11 Limites de Recursos — Seguridad es tambien Estabilidad

### 10.11.1 Denegacion de servicio accidental (o intencionada)

Sin limites de recursos, un solo contenedor puede consumir toda la memoria, CPU o procesos del host, causando una denegacion de servicio para todos los demas contenedores y servicios.

**Demostracion: fork bomb dentro de un contenedor**

```bash
# Contenedor SIN limite de procesos
docker run -it --rm ubuntu:22.04 bash

# Fork bomb clasica
:(){ :|:& };:

# En segundos, el host entero deja de responder
# Todos los contenedores y servicios se ven afectados
# Requiere reinicio forzado del host
```

Con `--pids-limit`:

```bash
# Contenedor CON limite de procesos
docker run -it --rm --pids-limit 100 ubuntu:22.04 bash

# Fork bomb
:(){ :|:& };:
# bash: fork: retry: Resource temporarily unavailable
# bash: fork: retry: Resource temporarily unavailable
# El contenedor se limita a si mismo — el host sigue funcionando
```

### 10.11.2 Limites de CPU

```bash
# Limitar a 1.5 CPUs
docker run --cpus=1.5 myapp

# Limitar a CPUs especificas (CPU affinity)
docker run --cpuset-cpus=0,1 myapp  # Solo usar CPU 0 y 1

# Cuota de CPU relativa (estilo antiguo, peso relativo)
docker run --cpu-shares=512 myapp  # 512 de 1024 por defecto = 1/2 CPU

# Limitar a fracciones
docker run --cpus=0.5 myapp  # Medio nucleo
```

```yaml
# Docker Compose con limites
services:
  api:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 128M
```

### 10.11.3 Limites de memoria

```bash
# Memoria maxima (con swap ilimitado por defecto)
docker run --memory=256m myapp

# Memoria + limite de swap
docker run --memory=256m --memory-swap=512m myapp
# memory-swap: total (RAM + swap). Si es igual a memory, no hay swap
docker run --memory=256m --memory-swap=256m myapp  # Sin swap
```

```bash
# Demostracion: acaparar memoria
docker run -it --rm --memory=128m ubuntu:22.04 bash

# Dentro del contenedor, instalar stress
apt-get update && apt-get install -y stress

# Intentar consumir 1 GB de RAM (limite: 128 MB)
stress --vm 1 --vm-bytes 1024M --timeout 10s
# stress: FAIL: ... <-- El proceso es OOM-killed
# El contenedor puede ser matado (OOM), pero no afecta al host

# Ver eventos OOM
docker events --filter event=oom
```

### 10.11.4 Limites de procesos (anti fork-bomb)

```bash
# Limite de PIDs
docker run --pids-limit=50 myapp

# Verificar
docker inspect mycontainer | jq '.[].HostConfig.PidsLimit'
# 50

# Demostracion
docker run -it --rm --pids-limit=20 ubuntu:22.04 bash

# Dentro:
for i in $(seq 1 30); do sleep 999 & done
# bash: fork: retry: Resource temporarily unavailable
# bash: fork: retry: Resource temporarily unavailable
# Solo se crean ~20 procesos (incluyendo bash y los sleeps exitosos)
```

### 10.11.5 ulimits — limites clasicos de Unix

```bash
# Limitar archivos abiertos
docker run --ulimit nofile=1024:2048 myapp
# soft limit: 1024, hard limit: 2048

# Limitar procesos por usuario
docker run --ulimit nproc=100:200 myapp

# Limitar memoria bloqueada (previene mlock bombing)
docker run --ulimit memlock=65536:65536 myapp

# Limitar tamano de core dump
docker run --ulimit core=0:0 myapp  # Sin core dumps

# Todos los ulimits relevantes:
docker run \
  --ulimit nofile=1024:2048 \
  --ulimit nproc=50:100 \
  --ulimit memlock=65536:65536 \
  --ulimit core=0:0 \
  --ulimit fsize=100000000 \
  --ulimit sigpending=100 \
  myapp
```

### 10.11.6 Blkio — limites de I/O

```bash
# Limitar I/O de disco (pesos relativos)
docker run --blkio-weight=100 myapp  # 100-1000, default 500

# Limitar I/O absoluto (bytes por segundo)
docker run \
  --device-read-bps=/dev/sda:1mb \
  --device-write-bps=/dev/sda:1mb \
  myapp

# Limitar IOPS
docker run \
  --device-read-iops=/dev/sda:100 \
  --device-write-iops=/dev/sda:100 \
  myapp
```

### 10.11.7 Kubernetes resource limits y QoS

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:       # Reserva minima garantizada
        memory: "128Mi"
        cpu: "250m"
      limits:         # Maximo permitido
        memory: "256Mi"
        cpu: "500m"
  # QoS Class: Burstable (requests < limits)
  # Guaranteed: requests == limits (mejor para produccion)
  # BestEffort: sin requests ni limits (primero en ser matado)
```

---

## 10.12 Aislamiento Adicional

### 10.12.1 --security-opt no-new-privileges

Esta opcion impide que el proceso del contenedor **gane nuevos privilegios** a traves de binarios setuid/setgid o capabilities adicionales. Incluso si hay un binario `sudo` dentro del contenedor, no funcionara.

```bash
# Sin no-new-privileges
docker run -it --rm --user 1000 ubuntu:22.04 bash
# Dentro: un binario setuid podria escalar a root
# (si existiera uno vulnerable como pkexec)
find / -perm -4000 -type f 2>/dev/null
# /usr/bin/passwd
# /usr/bin/su
# /usr/bin/pkexec
# ...

# Con no-new-privileges
docker run -it --rm --user 1000 --security-opt no-new-privileges ubuntu:22.04 bash
# Dentro:
/usr/bin/passwd
# passwd: Authentication token manipulation error
# No puede escalar privilegios aunque el binario sea setuid
```

```yaml
# Docker Compose
services:
  app:
    image: myapp:latest
    security_opt:
      - no-new-privileges:true
```

```yaml
# Kubernetes
securityContext:
  allowPrivilegeEscalation: false
```

### 10.12.2 --network none — contenedor sin red

Para procesos que no necesitan conectividad de red (procesos batch, calculos, transformaciones de datos), eliminar la interfaz de red reduce drasticamente la superficie de ataque.

```bash
# Contenedor completamente aislado de la red
docker run --network none my-batch-job

# Verificar
docker run --rm --network none alpine ip addr
# 1: lo: <LOOPBACK,UP,LOWER_UP> ...
# Solo loopback, ninguna interfaz externa

# Incluso con loopback, no se puede llegar a otros contenedores
```

### 10.12.3 --ipc private — memoria compartida aislada

Por defecto en Linux, los contenedores ya tienen su propio namespace IPC (System V IPC, POSIX message queues). Pero en Kubernetes, los pods comparten IPC entre contenedores del mismo pod.

```bash
# Host IPC namespace (inseguro, solo para debugging)
docker run --ipc=host myapp

# IPC privado (por defecto, recomendado)
docker run --ipc=private myapp

# Compartir IPC entre contenedores (cuando es necesario)
docker run --ipc=container:existing-container myapp
```

### 10.12.4 No compartir el namespace PID del host

```bash
# Peligroso: el contenedor ve todos los procesos del host
docker run --pid=host alpine ps aux
# PID   USER     TIME  COMMAND
# 1     root     0:00  /sbin/init
# 2     root     0:00  [kthreadd]
# ...

# Seguro (por defecto): namespace PID aislado
docker run alpine ps aux
# PID   USER     TIME  COMMAND
# 1     root     0:00  ps aux
# Solo ve sus propios procesos

# Compartir PID con otro contenedor (para debugging)
docker run --pid=container:target-container alpine ps aux
```

### 10.12.5 No compartir UTS namespace

```bash
# El UTS namespace controla hostname y domainname
# Por defecto esta aislado, pero cuidado con:
docker run --uts=host myapp
# El contenedor comparte el hostname del host
```

### 10.12.6 Resumen de flags de aislamiento

```bash
# Maximo aislamiento para un contenedor no privilegiado
docker run \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --security-opt seccomp=my-profile.json \
  --network none \
  --pids-limit 50 \
  --memory 256m \
  --cpus 0.5 \
  --user 1001:1001 \
  myapp:hardened
```

---

## 10.13 CIS Docker Benchmarks

### 10.13.1 Que es CIS Benchmark?

El **Center for Internet Security (CIS)** publica guias de hardening (benchmarks) para docenas de tecnologias, incluyendo Docker. El CIS Benchmark for Docker es la referencia de la industria para configurar Docker de forma segura.

El benchmark cubre categorias como:

1. **Host Configuration**: Configuracion del host Linux donde corre Docker.
2. **Docker Daemon Configuration**: Parametros de `dockerd` y `daemon.json`.
3. **Docker Daemon Configuration Files**: Permisos y ownership de archivos de configuracion.
4. **Container Images and Build Files**: Seguridad de imagenes y Dockerfiles.
5. **Container Runtime**: Configuracion de contenedores en ejecucion.
6. **Docker Security Operations**: Operaciones de seguridad y monitoreo.
7. **Docker Swarm Configuration**: Seguridad en clusteres Swarm.
8. **Docker Enterprise Edition**: Caracteristicas especificas de Docker EE.

### 10.13.2 Docker Bench Security

**Docker Bench Security** es un script oficial que audita tu instalacion Docker contra el CIS Benchmark.

```bash
# Ejecutar Docker Bench Security
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security

# Ejecutar auditoria
sudo sh docker-bench-security.sh

# Output tipico (resumen parcial):
# [INFO] 1 - Host Configuration
# [PASS] 1.1.1 - Ensure a separate partition for containers has been created
# [WARN] 1.1.2 - Ensure only trusted users are allowed to control Docker daemon
# [INFO] 2 - Docker daemon configuration
# [PASS] 2.1 - Ensure network traffic is restricted between containers
# [WARN] 2.2 - Ensure the logging level is set to 'info'
# ...
# [INFO] Checks: 100
# [INFO] Score: 72 (PASS: 72, WARN: 18, FAIL: 10, NOTE: 10, INFO: 0)
```

```bash
# Ejecutar con Docker (con acceso al socket y al sistema)
docker run --rm -it \
  --net host \
  --pid host \
  --userns host \
  --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /etc:/etc:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --label docker_bench_security \
  docker/docker-bench-security
```

### 10.13.3 Checks clave y como solucionarlos

| Check | Lo que evalua | Fallo tipico | Solucion |
|---|---|---|---|
| **2.1** | Restriccion de trafico inter-contenedor (ICC) | `"icc": true` en daemon.json | `{"icc": false}` — usar user-defined networks |
| **2.5** | Docker daemon no expuesto en TCP | `dockerd -H tcp://0.0.0.0:2375` | Solo escuchar en socket Unix |
| **2.8** | User namespace remapping habilitado | `"userns-remap": ""` | `{"userns-remap": "default"}` |
| **2.13** | Live restore habilitado | `"live-restore": false` | `{"live-restore": true}` |
| **2.14** | Userland proxy deshabilitado | `"userland-proxy": true` | `{"userland-proxy": false}` (usar iptables) |
| **3.x** | Permisos de archivos de config | `docker.service` world-readable | `chmod 644` (o mas restrictivo) |
| **4.4** | Imagenes verificadas (firma DCT) | `DOCKER_CONTENT_TRUST=0` | `export DOCKER_CONTENT_TRUST=1` |
| **4.6** | Imagenes escaneadas | Sin escaneo automatico | Integrar Trivy/Scout en CI |
| **5.1** | AppArmor/SELinux habilitado | Perfil "unconfined" | `--security-opt apparmor=docker-default` |
| **5.3** | Kernel capabilities restringidas | `--privileged` o `--cap-add=ALL` | Usar `--cap-drop=ALL --cap-add=solo-necesarias` |
| **5.7** | Puertos privilegiados mapeados dentro | `-p 80:80` (puerto < 1024 en host) | Usar reverse proxy, puertos > 1024 |
| **5.9** | Network namespace del host no compartido | `--network host` | Usar bridge/overlay (salvo excepcion justificada) |
| **5.15** | CPU priority (shares) configurada | Sin `--cpu-shares` | Establecer shares para contenedores criticos |
| **5.16** | Limite de memoria configurado | Sin `--memory` | Siempre establecer `--memory` |
| **5.24** | Healthcheck configurado | Dockerfile sin HEALTHCHECK | Anadir HEALTHCHECK |
| **5.28** | PIDs cgroup limit | Sin `--pids-limit` | Establecer un limite razonable |
| **7.x** | Swarm secrets y TLS | Swarm sin TLS | Activar TLS mutuo en Swarm |

### 10.13.4 daemon.json hardening de referencia

```json
{
  "icc": false,
  "log-driver": "json-file",
  "log-level": "info",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  },
  "userland-proxy": false,
  "userns-remap": "default",
  "live-restore": true,
  "no-new-privileges": true,
  "selinux-enabled": true,
  "iptables": true,
  "ip6tables": true,
  "ip-forward": false,
  "ip-masq": false,
  "experimental": false,
  "metrics-addr": "127.0.0.1:9323",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 1024,
      "Soft": 512
    },
    "nproc": {
      "Name": "nproc",
      "Hard": 2048,
      "Soft": 1024
    }
  },
  "default-runtime": "runc",
  "runtimes": {
    "runc": {
      "path": "runc"
    }
  },
  "default-seccomp-profile": "/etc/docker/seccomp/default.json",
  "seccomp-profile": "/etc/docker/seccomp/default.json"
}
```

---

## 10.14 Laboratorio: Hardening Completo Paso a Paso

### 10.14.1 Objetivo

Tomar una aplicacion web insegura ejecutada en Docker y aplicar **todas las tecnicas de hardening** aprendidas en este capitulo, midiendo la mejora de seguridad en cada paso.

### 10.14.2 La aplicacion inicial (insegura)

Vamos a crear una aplicacion Node.js simple con una vulnerabilidad de RCE (Remote Code Execution) deliberada (solo con fines educativos!).

**Paso 0: Estructura del proyecto**

```bash
mkdir hardening-lab && cd hardening-lab

# Archivo: app.js — aplicacion web vulnerable
cat > app.js << 'JAVASCRIPT'
const express = require('express');
const { execSync } = require('child_process');
const app = express();
const port = 80;

// Simulamos acceso a logs (con vulnerabilidad RCE deliberada)
app.get('/logs', (req, res) => {
    const filter = req.query.filter || '';
    // VULNERABILIDAD: ejecucion de comandos sin sanitizar
    const output = execSync('cat /var/log/app.log | grep "' + filter + '"', {
        encoding: 'utf-8'
    });
    res.send('<pre>' + output + '</pre>');
});

app.get('/', (req, res) => {
    res.send('OK - App running');
});

app.listen(port, () => {
    console.log('App listening on port ' + port);
});
JAVASCRIPT

# package.json
cat > package.json << 'JSON'
{
  "name": "insecure-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
JSON
```

**Dockerfile INSEGURO original:**

```dockerfile
# Dockerfile — VERSION INSEGURA
FROM node:18-slim

WORKDIR /app
COPY package.json ./
RUN npm install

COPY . .

# Inseguridades:
# 1. Ejecuta como ROOT
# 2. Todas las capabilities por defecto
# 3. Filesystem escribible
# 4. Sin limites de recursos
# 5. Puerto 80 (requiere NET_BIND_SERVICE innecesariamente)
EXPOSE 80
CMD ["node", "app.js"]
```

```bash
# Construir y ejecutar la version insegura
docker build -t insecure-app:v1 .
docker run -d --name insecure-app -p 8080:80 insecure-app:v1

# Verificar que funciona
curl http://localhost:8080/
# OK - App running

# Explotar la vulnerabilidad RCE (CON FINES EDUCATIVOS)
curl "http://localhost:8080/logs?filter=%24(cat%20/etc/passwd)"
# Muestra /etc/passwd del contenedor

curl "http://localhost:8080/logs?filter=%24(whoami)"
# root <-- Somos root!

curl "http://localhost:8080/logs?filter=%24(apt-get%20update)"
# Podemos instalar paquetes

# Destruir el contenedor inseguro
docker rm -f insecure-app
```

### 10.14.3 Paso 1: Cambiar a usuario no-root

```dockerfile
# Dockerfile — VERSION MEJORADA #1
FROM node:18-slim

# Crear usuario no-root
RUN groupadd -r appgroup && useradd -r -g appgroup -m -d /home/appuser appuser

WORKDIR /app
COPY package.json ./
RUN npm install

COPY . .

# Cambiar ownership de los archivos al usuario app
RUN chown -R appuser:appgroup /app

# Cambiar a usuario no-root
USER appuser

EXPOSE 3000  # Cambiamos a puerto no privilegiado (>1024)
CMD ["node", "app.js"]
```

```bash
docker build -t secure-app:v1 .
docker run -d --name secure-app-v1 -p 3000:3000 secure-app:v1

# Verificar el usuario
docker exec secure-app-v1 whoami
# appuser

# Intentar el mismo ataque
curl "http://localhost:3000/logs?filter=%24(whoami)"
# appuser <-- Ya no somos root

docker rm -f secure-app-v1
```

### 10.14.4 Paso 2: Dropear capabilities

```bash
# Dropear TODAS las capabilities (puerto > 1024, no necesita ninguna)
docker run -d --name secure-app-v2 \
  --cap-drop=ALL \
  -p 3000:3000 \
  secure-app:v1

# Verificar capabilities
PID=$(docker inspect -f '{{.State.Pid}}' secure-app-v2)
getpcaps $PID 2>/dev/null || cat /proc/$PID/status | grep CapEff
# CapEff: 0000000000000000 (ninguna capability)

# El ataque RCE sigue funcionando dentro del contenedor, pero con UID no-root
# y sin capabilities, el dano esta mucho mas limitado

docker rm -f secure-app-v2
```

### 10.14.5 Paso 3: Filesystem read-only + tmpfs

```bash
docker run -d --name secure-app-v3 \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /home/appuser/.npm \
  --tmpfs /var/log \
  --cap-drop=ALL \
  -p 3000:3000 \
  secure-app:v1

# Verificar
docker exec secure-app-v3 touch /app/test
# touch: /app/test: Read-only file system

docker exec secure-app-v3 touch /tmp/test
# (funciona — /tmp esta en tmpfs)

# El atacante no puede escribir archivos maliciosos ni modificar binarios
curl "http://localhost:3000/logs?filter=%24(echo%20owned%20%3E%20/tmp/pwned)"
# Funciona (escribe en /tmp porque esta en tmpfs)

curl "http://localhost:3000/logs?filter=%24(cat%20/tmp/pwned)"
# owned  <- Confirmado que escribio

# Pero no puede persistir tras reinicio (tmpfs es volatil)

docker rm -f secure-app-v3
```

### 10.14.6 Paso 4: Limitar recursos

```bash
docker run -d --name secure-app-v4 \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /home/appuser/.npm \
  --cap-drop=ALL \
  --memory=128m \
  --memory-swap=128m \
  --cpus=0.5 \
  --pids-limit=20 \
  --ulimit nofile=1024:2048 \
  --ulimit nproc=50:100 \
  -p 3000:3000 \
  secure-app:v1

# Verificar limites
docker stats --no-stream secure-app-v4
# MEM USAGE / LIMIT   MEM %   CPU %
# 28.5MiB / 128MiB   22.28%   0.00%

docker inspect secure-app-v4 | jq '.[].HostConfig | {Memory, PidsLimit}'
# {
#   "Memory": 134217728,
#   "PidsLimit": 20
# }

# Prueba de fork bomb:
docker exec secure-app-v4 bash -c ':(){ :|:& };:'
# bash: fork: retry: Resource temporarily unavailable
# El atacante no puede hacer fork bomb ni consumir toda la RAM

docker rm -f secure-app-v4
```

### 10.14.7 Paso 5: Escanear con Trivy

```bash
# Escanear la imagen antes de desplegar
trivy image --severity CRITICAL,HIGH secure-app:v1

# Output tipico:
# secure-app:v1 (debian 12.x)
# Total: 42 (CRITICAL: 2, HIGH: 12, MEDIUM: 18, LOW: 10)
#
# node:18-slim hereda vulnerabilidades de Debian
# CRITICAL: CVE-2023-44487 en libnghttp2 (HTTP/2 Rapid Reset)
# HIGH: CVE-2023-38545 en curl

# Mejorar cambiando a una imagen mas segura (Alpine tiene menos CVEs)
cat > Dockerfile.secure << 'DOCKERFILE'
FROM node:18-alpine

RUN addgroup -g 1001 -S appgroup && adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

COPY --chown=appuser:appgroup package.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --chown=appuser:appgroup . .

USER appuser

EXPOSE 3000
CMD ["node", "app.js"]
DOCKERFILE
```

```bash
# Comparar vulnerabilidades entre imagenes
docker build -f Dockerfile.secure -t secure-app:v5 .

echo "=== Vulnerabilidades en node:18-slim ==="
trivy image --severity CRITICAL,HIGH --ignore-unfixed --format table secure-app:v1 2>/dev/null | tail -20

echo ""
echo "=== Vulnerabilidades en node:18-alpine ==="
trivy image --severity CRITICAL,HIGH --ignore-unfixed --format table secure-app:v5 2>/dev/null | tail -20

# Alpine tipicamente tiene MUCHAS menos vulnerabilidades que Debian-based
```

### 10.14.8 Paso 6: Aplicar perfil seccomp personalizado

```bash
# Crear perfil seccomp para Node.js
cat > node-seccomp.json << 'SECCOMP'
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    {
      "names": [
        "accept", "accept4", "access", "arch_prctl", "bind",
        "brk", "capget", "chdir", "clock_gettime", "clone",
        "close", "connect", "dup", "dup2", "epoll_create1",
        "epoll_ctl", "epoll_pwait", "epoll_wait", "eventfd2",
        "execve", "exit", "exit_group", "fchdir", "fchmod",
        "fchown", "fcntl", "fdatasync", "fstat", "fsync",
        "ftruncate", "futex", "getcwd", "getdents64",
        "getegid", "geteuid", "getgid", "getpeername",
        "getpid", "getppid", "getrandom", "getrlimit",
        "getsockname", "getsockopt", "gettid", "getuid",
        "ioctl", "kill", "listen", "lseek", "lstat",
        "madvise", "mincore", "mkdir", "mmap",
        "mprotect", "munmap", "nanosleep", "newfstatat",
        "openat", "pipe2", "pread64", "prlimit64",
        "pwrite64", "read", "readlink", "readv",
        "recvfrom", "recvmsg", "rename", "restart_syscall",
        "rmdir", "rt_sigaction", "rt_sigpending",
        "rt_sigprocmask", "rt_sigreturn", "sched_getaffinity",
        "sched_yield", "sendfile", "sendmsg", "sendto",
        "set_robust_list", "set_tid_address", "setsockopt",
        "shutdown", "sigaltstack", "socket", "stat",
        "statfs", "symlink", "sysinfo", "tgkill", "uname",
        "unlink", "utimensat", "wait4", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
SECCOMP

# Ejecutar con perfil seccomp
docker run -d --name secure-app-v6 \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --memory=128m \
  --cpus=0.5 \
  --pids-limit=20 \
  --security-opt no-new-privileges \
  --security-opt seccomp=$(pwd)/node-seccomp.json \
  -p 3000:3000 \
  secure-app:v5

# Verificar que la app funciona
curl http://localhost:3000/
# OK - App running

docker rm -f secure-app-v6
```

### 10.14.9 Paso 7: Docker Compose final con todas las medidas

**Dockerfile FINAL con HEALTHCHECK:**

```dockerfile
# Dockerfile FINAL HARDENED
FROM node:18-alpine

RUN addgroup -g 1001 -S appgroup && adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

COPY --chown=appuser:appgroup package.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --chown=appuser:appgroup . .

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/ || exit 1

CMD ["node", "app.js"]
```

**Docker Compose final con todas las medidas de seguridad:**

```yaml
# docker-compose.hardened.yml
version: '3.8'

services:
  webapp:
    build:
      context: .
      dockerfile: Dockerfile.secure
    image: secure-app:hardened-v7
    read_only: true
    tmpfs:
      - /tmp:size=64M,noexec,nosuid
      - /var/log:size=32M,noexec
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
      - seccomp:./node-seccomp.json
    ulimits:
      nofile:
        soft: 1024
        hard: 2048
      nproc:
        soft: 100
        hard: 200
    pids_limit: 20
    mem_limit: 128m
    memswap_limit: 128m
    cpus: 0.5
    restart: unless-stopped
    ports:
      - "3000:3000"
    networks:
      - frontend
    user: "1001:1001"
    # NO montar /var/run/docker.sock
    # NO usar --privileged
    # NO usar pid: host
    # NO usar network_mode: host

networks:
  frontend:
    driver: bridge
```

### 10.14.10 Comparativa: Antes vs Despues

| Aspecto de seguridad | Version INSEGURA | Version HARDENED |
|---|---|---|
| **Usuario** | root (UID 0) | appuser (UID 1001) |
| **Capabilities** | 14 por defecto | 0 (todas dropeadas) |
| **Seccomp** | Perfil por defecto Docker | Perfil personalizado allowlist |
| **Filesystem** | Escribible completamente | Read-only + tmpfs especificos |
| **Limite memoria** | Ilimitado | 128 MB |
| **Limite PIDs** | Ilimitado | 20 procesos |
| **Limite CPU** | Ilimitado | 0.5 nucleos |
| **no-new-privileges** | No | Si |
| **AppArmor/SELinux** | docker-default | docker-default (mantenido) |
| **Vulnerabilidades (CVE)** | ~42 CRITICAL+HIGH (node:18-slim) | ~0-2 CRITICAL (node:18-alpine) |
| **Escaneo en CI** | No | Bloquea builds con CRITICAL |
| **Firma de imagenes** | No | Verificacion con Cosign |
| **Secretos** | Podrian estar en codigo | Via Vault/BuildKit secrets |
| **HEALTHCHECK** | No | Si, auto-recuperacion |
| **Explotable via RCE** | root, instalar paquetes, breakout potencial | appuser, sin capabilities, sin escritura en disco |

### 10.14.11 Prueba de penetracion: comparativa

```bash
# ============================================
# Ataque contra version INSEGURA
# ============================================
docker run -d --name insecure-test -p 8888:80 insecure-app:v1

# 1. Reconocimiento
echo "=== RECONOCIMIENTO ==="
curl "http://localhost:8888/logs?filter=%24(whoami)"
# root <- CRITICO!
curl "http://localhost:8888/logs?filter=%24(id)"
# uid=0(root) gid=0(root) groups=0(root)
curl "http://localhost:8888/logs?filter=%24(uname%20-a)"
# Informacion del kernel del host
curl "http://localhost:8888/logs?filter=%24(capsh%20--print%202%3E/dev/null%20%7C%7C%20cat%20/proc/1/status%20%7C%20grep%20Cap)"
# Capabilities disponibles

# 2. Persistencia
curl "http://localhost:8888/logs?filter=%24(apt-get%20update%20%26%26%20apt-get%20install%20-y%20nmap)"
# Instalamos herramientas de ataque
curl "http://localhost:8888/logs?filter=%24(nmap%20-sP%20172.17.0.0/24)"
# Escaneamos la red interna de Docker

# 3. Escalacion / breakout potencial
# Si encontrara una vulnerabilidad en runc (CVE-2019-5736), podria escapar

docker rm -f insecure-test

# ============================================
# Ataque contra version HARDENED
# ============================================
docker run -d --name secure-test \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --memory=128m \
  --pids-limit=20 \
  --security-opt no-new-privileges \
  --security-opt seccomp=$(pwd)/node-seccomp.json \
  -p 9999:3000 \
  secure-app:v5

# 1. Reconocimiento — mismo ataque
echo "=== RECONOCIMIENTO ==="
curl "http://localhost:9999/logs?filter=%24(whoami)"
# appuser <- No es root
curl "http://localhost:9999/logs?filter=%24(id)"
# uid=1001(appuser) gid=1001(appgroup)
curl "http://localhost:9999/logs?filter=%24(cat%20/proc/1/status%20%7C%20grep%20Cap)"
# CapEff: 0000000000000000 <- Sin capabilities

# 2. Intento de instalar herramientas
curl "http://localhost:9999/logs?filter=%24(apt-get%20update)"
# Error: Permission denied (sin root, sin CAP_SYS_ADMIN)

# 3. Intento de escribir malware
curl "http://localhost:9999/logs?filter=%24(echo%20'malware'%20%3E%20/app/malware.sh)"
# Error: Read-only file system (filesystem read-only)

# 4. Intento de fork bomb
# Limitado por --pids-limit=20

# 5. Intento de escanear red
curl "http://localhost:9999/logs?filter=%24(ping%20-c%201%20172.17.0.1)"
# Error: socket: Operation not permitted (sin CAP_NET_RAW)

docker rm -f secure-test
```

### 10.14.12 Resumen del laboratorio

Tras aplicar el hardening:

- El mismo ataque RCE que daba **root con capabilities** ahora da un **usuario sin privilegios, sin capacidades, sin acceso de escritura, con red restringida**.
- El atacante no puede instalar herramientas, no puede escribir en disco, no puede hacer sniffing de red, no puede hacer fork bomb.
- Si el atacante lograra un breakout del runtime, aterrizaria como UID 1001 (no root) en el host si user namespaces estan habilitados.
- La imagen base Alpine reduce la superficie de vulnerabilidades del SO de ~42 CVEs criticos/altos a practicamente 0.

### 10.14.13 Script de hardening automatizado

```bash
#!/bin/bash
# hardening-scan.sh — Auditoria rapida de seguridad de contenedores
# Uso: ./hardening-scan.sh <container_name>

set -euo pipefail

CONTAINER="${1:?Usage: $0 <container_name>}"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

check() {
    local desc="$1" result="$2"
    if [ "$result" = "PASS" ]; then
        echo -e "${GREEN}[PASS]${NC} $desc"
    elif [ "$result" = "WARN" ]; then
        echo -e "${YELLOW}[WARN]${NC} $desc"
    else
        echo -e "${RED}[FAIL]${NC} $desc"
    fi
}

echo "=== Auditoria de Seguridad del Contenedor: $CONTAINER ==="
echo ""

# 1. Usuario no-root
USER=$(docker inspect -f '{{.Config.User}}' "$CONTAINER")
if [ -z "$USER" ] || [ "$USER" = "root" ] || [ "$USER" = "0" ] || [ "$USER" = "0:0" ]; then
    check "Usuario ejecutandose como root (Config.User='${USER:-vacio=root}')" "FAIL"
else
    check "Usuario no-root configurado: $USER" "PASS"
fi

# 2. Privileged mode
PRIV=$(docker inspect -f '{{.HostConfig.Privileged}}' "$CONTAINER")
if [ "$PRIV" = "true" ]; then
    check "Modo privilegiado activado" "FAIL"
else
    check "Modo privilegiado: desactivado" "PASS"
fi

# 3. Read-only rootfs
RO=$(docker inspect -f '{{.HostConfig.ReadonlyRootfs}}' "$CONTAINER")
if [ "$RO" = "true" ]; then
    check "Filesystem raiz: read-only" "PASS"
else
    check "Filesystem raiz: escribible" "WARN"
fi

# 4. Capabilities
CAPS=$(docker inspect -f '{{range .HostConfig.CapDrop}}{{.}} {{end}}' "$CONTAINER")
if echo "$CAPS" | grep -q "ALL"; then
    CAPADD=$(docker inspect -f '{{range .HostConfig.CapAdd}}{{.}} {{end}}' "$CONTAINER")
    check "Capabilities: ALL dropeadas (anadidas: ${CAPADD:-ninguna})" "PASS"
else
    check "Capabilities: no se dropearon todas" "WARN"
fi

# 5. Limites de memoria
MEM=$(docker inspect -f '{{.HostConfig.Memory}}' "$CONTAINER")
if [ "$MEM" != "0" ]; then
    MEM_MB=$((MEM / 1048576))
    check "Limite de memoria: ${MEM_MB}MB" "PASS"
else
    check "Sin limite de memoria configurado" "WARN"
fi

# 6. PIDs limit
PIDS=$(docker inspect -f '{{.HostConfig.PidsLimit}}' "$CONTAINER")
if [ -n "$PIDS" ] && [ "$PIDS" -gt 0 ] 2>/dev/null; then
    check "Limite de PIDs: $PIDS" "PASS"
else
    check "Sin limite de PIDs configurado" "WARN"
fi

# 7. Network mode
NET=$(docker inspect -f '{{.HostConfig.NetworkMode}}' "$CONTAINER")
if [ "$NET" = "host" ]; then
    check "Network mode: host (comparte red del host)" "FAIL"
elif [ "$NET" = "none" ]; then
    check "Network mode: none (sin red — puede ser correcto para batch)" "PASS"
else
    check "Network mode: $NET (aislado)" "PASS"
fi

# 8. Docker socket montado
SOCK_MOUNT=$(docker inspect -f '{{range .Mounts}}{{if eq .Source "/var/run/docker.sock"}}MOUNTED{{end}}{{end}}' "$CONTAINER")
if [ -n "$SOCK_MOUNT" ]; then
    check "Docker socket montado dentro del contenedor" "FAIL"
else
    check "Docker socket NO montado" "PASS"
fi

# 9. no-new-privileges
NNP=$(docker inspect -f '{{range .HostConfig.SecurityOpt}}{{if eq . "no-new-privileges"}}ENABLED{{end}}{{end}}' "$CONTAINER")
if [ -n "$NNP" ]; then
    check "no-new-privileges: activado" "PASS"
else
    check "no-new-privileges: no configurado" "WARN"
fi

# 10. Healthcheck
HC=$(docker inspect -f '{{if .Config.Healthcheck}}CONFIGURED{{else}}MISSING{{end}}' "$CONTAINER")
if [ "$HC" = "CONFIGURED" ]; then
    check "HEALTHCHECK: configurado" "PASS"
else
    check "HEALTHCHECK: ausente" "WARN"
fi

echo ""
echo "=== Verificacion de Imagen ==="

# CVEs con Trivy (si esta instalado)
if command -v trivy &>/dev/null; then
    IMAGE=$(docker inspect -f '{{.Config.Image}}' "$CONTAINER")
    echo "Escaneando $IMAGE con Trivy..."
    trivy image --severity CRITICAL --ignore-unfixed --quiet "$IMAGE" 2>/dev/null || \
        echo "(Trivy no pudo escanear — imagen local?)"
else
    echo "Trivy no instalado — instalalo para escaneo de CVEs"
fi

echo ""
echo "=== Recomendaciones ==="
echo "- Usa --cap-drop=ALL --cap-add=<solo las necesarias>"
echo "- Usa --read-only con --tmpfs para directorios writables"
echo "- Nunca montes /var/run/docker.sock en contenedores"
echo "- Siempre establece --memory y --pids-limit"
echo "- Usa imagenes Alpine cuando sea posible (menos CVEs)"
echo "- Integra Trivy o Docker Scout en tu pipeline CI/CD"
echo "- Activa DOCKER_CONTENT_TRUST o usa Cosign para verificar firmas"
echo "- NUNCA pongas secretos en ENV, ARG, o COPY en el Dockerfile"
```

### 10.14.14 Conclusiones del capitulo

La seguridad en contenedores no es un checkbox que marcas una vez y olvidas. Es una disciplina continua que abarca:

1. **Build seguro**: Imagenes base minimas, no-root, multi-stage, BuildKit secrets.
2. **Runtime hardening**: Capabilities minimas, seccomp, AppArmor/SELinux, read-only filesystem.
3. **Supply chain**: Firma de imagenes, escaneo de CVEs, renovacion automatica de dependencias.
4. **Operaciones**: Limites de recursos, monitoreo, health checks, CIS benchmarks.
5. **Secretos**: Nunca en la imagen, siempre externalizados (Vault, SOPS, Sealed Secrets).

Cada capa de seguridad que anades hace el trabajo del atacante exponencialmente mas dificil. Un contenedor que ejecuta como root con capabilities por defecto es un regalo para un atacante. Un contenedor con todas las medidas de hardening aplicadas requiere no solo un exploit de aplicacion (RCE), sino tambien un exploit de escalacion de privilegios, un bypass de seccomp, y posiblemente un breakout del runtime — una cadena de exploits que pocos atacantes pueden ejecutar.

**Recuerda**: La seguridad no es un producto, es un proceso. Escanea tus imagenes regularmente, mantente al dia con los CVEs, actualiza tus imagenes base, y revisa tus configuraciones de hardening periodicamente.

---

**Fin del Capitulo 10**

---

← [Capítulo anterior](capitulo-09-cicd.md) | [Inicio](../README.md) | [Capítulo siguiente →](capitulo-11-monitoreo.md)
