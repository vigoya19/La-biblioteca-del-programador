# Capítulo 2: Instalación y Primeros Pasos

> *"Un viaje de mil millas comienza con un solo paso."* — Lao Tzu

En este capítulo instalarás Docker en cualquier sistema operativo, comprenderás la anatomía completa del comando `docker run`, dominarás el ciclo de vida de los contenedores y ejecutarás comandos esenciales con ejemplos reales. Al final, desplegarás un stack de cuatro servicios interconectados que sentará las bases para todo lo que viene después.

---

## Tabla de Contenidos

1. [Instalación de Docker](#1-instalación-de-docker)
2. [Anatomía de `docker run`](#2-anatomía-de-docker-run)
3. [Ciclo de vida de un contenedor](#3-ciclo-de-vida-de-un-contenedor)
4. [Comandos esenciales del día a día](#4-comandos-esenciales-del-día-a-día)
5. [Gestión de contenedores](#5-gestión-de-contenedores)
6. [Imágenes básicas](#6-imágenes-básicas)
7. [Laboratorio práctico integrador](#7-laboratorio-práctico-integrador)

---

## 1. Instalación de Docker

Docker existe en dos sabores principales:

- **Docker Desktop**: incluye GUI, Kubernetes integrado, sincronización de credenciales y una VM gestionada automáticamente. Ideal para desarrollo en macOS y Windows.
- **Docker Engine**: el daemon puro que corre nativamente en Linux. Es lo que se usa en producción y servidores.

```
┌──────────────────────────────────────────────────────────────┐
│                  ¿Qué opción de instalación?                 │
├──────────────┬───────────────────┬───────────────────────────┤
│   Sistema    │    Recomendado    │       Alternativa         │
├──────────────┼───────────────────┼───────────────────────────┤
│ Linux        │ Docker Engine     │ Docker Desktop            │
│ macOS        │ Colima + CLI      │ Docker Desktop            │
│ Windows      │ Docker Desktop    │ Engine en WSL2 manual     │
└──────────────┴───────────────────┴───────────────────────────┘
```

---

### 1.1 Linux — Ubuntu / Debian (vía apt, repositorio oficial)

El método recomendado es instalar desde el repositorio oficial de Docker, NUNCA desde los repositorios de la distribución (`apt install docker.io`), que suelen estar desactualizados.

#### Paso 1: Eliminar versiones antiguas

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg 2>/dev/null
done
```

La palabra `docker` en estos paquetes antiguos puede referirse a cosas distintas. `docker.io` es el paquete de Debian/Ubuntu con versiones viejas. `podman-docker` emula el CLI de Docker con Podman.

#### Paso 2: Instalar dependencias y configurar el repositorio oficial

```bash
# Actualizar índice de paquetes e instalar dependencias necesarias para HTTPS
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Agregar la clave GPG oficial de Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Agregar el repositorio estable
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

El parámetro `$(dpkg --print-architecture)` detecta automáticamente si es amd64, arm64, etc. El `$(lsb_release -cs)` obtiene el nombre en código de tu Ubuntu (focal, jammy, noble...). El `signed-by` garantiza que solo se acepten paquetes firmados por la clave GPG de Docker.

#### Paso 3: Instalar Docker Engine, CLI, containerd y Compose

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin
```

Qué instala cada paquete:

| Paquete | Qué contiene |
|---------|-------------|
| `docker-ce` | El daemon de Docker (`dockerd`), el servicio systemd |
| `docker-ce-cli` | El binario `docker` que usas en la terminal |
| `containerd.io` | El runtime de contenedores que gestiona el ciclo de vida |
| `docker-buildx-plugin` | BuildKit integrado para builds avanzados |
| `docker-compose-plugin` | `docker compose` (plugin) en lugar de `docker-compose` standalone |

#### Paso 4: Post-instalación — grupo docker

El socket de Docker (`/var/run/docker.sock`) pertenece a `root:docker`. Para usar Docker sin `sudo` constantemente, añade tu usuario al grupo `docker`:

```bash
sudo usermod -aG docker $USER
newgrp docker
# O alternativamente, cierra sesión y vuelve a entrar
```

**Advertencia de seguridad**: el grupo `docker` otorga privilegios equivalentes a root en el host. Cualquier usuario en ese grupo puede escalar a root trivialmente montando `/` del host dentro de un contenedor:

```bash
docker run -v /:/host -it ubuntu chroot /host
# Ahora tienes root en el host
```

En producción, considera usar **rootless Docker** (Capítulo 10) o políticas de autorización.

#### Paso 5: Verificar la instalación

```bash
# Comprobar que el daemon está corriendo
sudo systemctl status docker

# Habilitar inicio automático al bootear
sudo systemctl enable docker

# Verificar que todo funciona
docker run hello-world
```

Si ves:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

tu instalación está completa.

---

#### Script de conveniencia vs instalación manual

Docker ofrece un script que automatiza todo:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**Ventajas**:
- Una sola línea, rápido.
- Detecta automáticamente la distribución.

**Desventajas**:
- Ejecutas código de internet como root sin revisarlo. **Mal hábito**.
- Riesgo de MITM si no verificas TLS.
- No sabes exactamente qué se está configurando.

**Recomendación**: usar el script SOLO en entornos efímeros (CI/CD, máquinas de prueba) o después de revisarlo manualmente (`less get-docker.sh`). En producción, siempre instalación manual paso a paso o gestión por configuración (Ansible, Puppet, Chef, Terraform).

---

### 1.2 Linux — RHEL / CentOS / Fedora (vía dnf/yum)

#### RHEL 9 / CentOS Stream 9 / Fedora

```bash
# Eliminar versiones antiguas
sudo dnf remove -y docker docker-client docker-client-latest \
    docker-common docker-latest docker-latest-logrotate \
    docker-logrotate docker-engine podman runc

# Instalar el repositorio oficial
sudo dnf config-manager --add-repo \
    https://download.docker.com/linux/rhel/docker-ce.repo

# Instalar Docker Engine
sudo dnf install -y docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin

# Iniciar y habilitar
sudo systemctl start docker
sudo systemctl enable docker

# Grupo docker
sudo usermod -aG docker $USER

# Verificar
docker run hello-world
```

#### CentOS 7 / RHEL 7 (yum)

```bash
sudo yum remove -y docker docker-client docker-client-latest \
    docker-common docker-latest docker-latest-logrotate \
    docker-logrotate docker-engine

sudo yum install -y yum-utils
sudo yum-config-manager --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install -y docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin

sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

---

### 1.3 macOS

macOS no tiene kernel Linux, por lo que Docker necesita una máquina virtual Linux para funcionar. Tienes dos opciones:

#### Opción A: Docker Desktop (oficial)

```
┌─────────────────────────────────────────────────────┐
│                  Docker Desktop en macOS             │
├─────────────────────────────────────────────────────┤
│  ┌──────────────┐    ┌────────────────────────────┐ │
│  │  CLI (docker)│◄──►│  Docker Daemon (dockerd)   │ │
│  └──────────────┘    │  dentro de VM Linux         │ │
│                      │  ┌───────────────────────┐  │ │
│  ┌──────────────┐    │  │ containerd + runc     │  │ │
│  │  GUI App     │    │  │ Contenedores Linux    │  │ │
│  └──────────────┘    │  └───────────────────────┘  │ │
│                      └────────────────────────────┘ │
│  Hypervisor: Virtualization.framework / HyperKit    │
└─────────────────────────────────────────────────────┘
```

**Instalación vía Homebrew:**

```bash
brew install --cask docker
# Abre Docker.app desde /Applications y acepta los términos de servicio
```

**Instalación manual:** descargar el `.dmg` desde [hub.docker.com](https://hub.docker.com/) y arrastrar a `/Applications`.

**Apple Silicon (M1/M2/M3/M4):** descarga la versión "Apple Chip". Docker Desktop usa `qemu` para emular imágenes `amd64` en ARM64, pero con penalización de rendimiento. Siempre prefiere imágenes multi-arch (`--platform linux/arm64` o `linux/amd64`).

**Requisitos:**
- macOS 12 (Monterey) o superior.
- Al menos 4 GB de RAM (recomendado 8 GB+).
- Virtualization.framework habilitado (por defecto en Apple Silicon; en Intel requiere Hypervisor.framework).

#### Opción B: Colima + Docker CLI (ligera, open-source)

Colima es una alternativa minimalista que ejecuta containerd en una VM de Lima (QEMU o Virtualization.framework). No trae GUI, consume menos recursos y es completamente open-source.

```bash
# Instalar CLI de Docker y Colima
brew install docker colima

# Iniciar Colima (crea la VM Linux automáticamente)
colima start

# Verificar
docker run hello-world
```

**Configuraciones útiles de Colima:**

```bash
# Especificar CPU, memoria y disco
colima start --cpu 4 --memory 8 --disk 60

# Usar arquitectura x86_64 en Apple Silicon (emulación)
colima start --arch amd64

# Activar Kubernetes
colima start --kubernetes

# Con runtime alternativo
colima start --runtime containerd    # por defecto
colima start --runtime docker        # modo legacy (dockerd)
```

**Ventajas de Colima:**
- No requiere cuenta Docker.
- Código completamente abierto.
- Arranque más rápido que Docker Desktop.
- Soporte nativo para `docker compose` (vía plugin de CLI).
- Menor consumo de RAM en idle (~200 MB vs ~1 GB de Docker Desktop).

**Desventajas:**
- Sin GUI.
- Sin soporte para extensiones de Docker Desktop.
- La sincronización de archivos host↔VM puede ser más lenta en cargas intensivas de I/O.

**Comparativa Docker Desktop vs Colima:**

| Característica | Docker Desktop | Colima |
|---|---|---|
| GUI | Sí | No |
| Kubernetes integrado | Sí | Sí (opcional) |
| Cuenta obligatoria | Sí (desde 2021) | No |
| Licencia | Propietaria (gratis para uso personal/pequeñas empresas) | MIT |
| Consumo RAM (idle) | ~1 GB | ~200 MB |
| Volúmenes montados | gRPC-FUSE (rápido) | SSHFS/9P (más lento) |
| Soporte oficial | Sí (Docker Inc.) | Comunidad |
| Rosetta 2 (x86 en ARM) | Sí | Limitado |

---

### 1.4 Windows

Docker en Windows requiere el Subsistema de Windows para Linux (WSL2), que proporciona un kernel Linux real dentro de una VM ligera gestionada por Hyper-V.

```
┌────────────────────────────────────────────────────────────┐
│            Docker Desktop en Windows con WSL2              │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────────── Windows 10/11 ────────────────────┐ │
│  │                                                        │ │
│  │  PowerShell / CMD ─── docker CLI (nativo Windows)      │ │
│  │       │                                                │ │
│  │       ▼                                                │ │
│  │  ┌──────────────┐     ┌───────────────────────────┐   │ │
│  │  │ Docker Desktop│────►│  VM WSL2 (kernel Linux)   │   │ │
│  │  │ (GUI + Service)│    │  ┌─────────────────────┐  │   │ │
│  │  └──────────────┘     │  │ dockerd              │  │   │ │
│  │                       │  │ containerd + runc    │  │   │ │
│  │  C:\> docker run ...  │  │ Contenedores Linux   │  │   │ │
│  │       │               │  └─────────────────────┘  │   │ │
│  │       │               └───────────────────────────┘   │ │
│  │       │  named pipe                                   │ │
│  │       └────────► docker.exe ──► /var/run/docker.sock  │ │
│  │                                                        │ │
│  │  También: docker CLI en WSL2 directamente              │ │
│  │  $ docker run ...                                     │ │
│  └────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

#### Requisitos previos

1. **Windows 10 versión 2004+** (Build 19041+) o **Windows 11**.
2. **Virtualización habilitada en BIOS/UEFI:**
   - Intel VT-x / AMD-V.
   - Reinicia, entra al BIOS (F2/DEL/ESC durante el boot).
   - Busca "Intel Virtualization Technology" o "SVM Mode" y actívalo.
3. **WSL2 instalado.**

#### Paso 1: Instalar WSL2

Abre PowerShell como Administrador:

```powershell
# Instalar WSL2 con un solo comando (Windows 10 2004+ o Windows 11)
wsl --install

# Si ya tienes WSL1, actualiza a WSL2
wsl --set-default-version 2

# Verificar versión
wsl --status
```

Este comando instala:
- Virtual Machine Platform (componente opcional de Windows).
- WSL2 kernel.
- Distribución Ubuntu por defecto.

#### Paso 2: Instalar Docker Desktop

```powershell
# Con winget (recomendado)
winget install Docker.DockerDesktop

# O descarga el instalador .exe desde hub.docker.com
```

Durante la instalación:
- Marca **"Use WSL 2 instead of Hyper-V"**.
- El instalador añadirá automáticamente el backend WSL2.

#### Paso 3: Verificar backend WSL2

Abre Docker Desktop → Settings → Resources → WSL Integration:

- Asegúrate de que "Enable integration with my default WSL distro" esté marcado.
- Selecciona las distribuciones WSL donde quieres que Docker funcione.

#### Paso 4: Verificar

```powershell
# En PowerShell
docker run hello-world

# En WSL2 (dentro de Ubuntu)
docker run hello-world
```

Ambos deben funcionar porque comparten el mismo daemon Docker. El CLI de Windows se comunica con el daemon en WSL2 vía named pipe.

#### Notas importantes para Windows

- **Diferencia de rutas**: en PowerShell, las rutas son `C:\Users\...`. En WSL2, son `/mnt/c/Users/...`. Al montar volúmenes desde el host usa rutas WSL2 o rutas UNC.
- **Rendimiento de archivos**: montar archivos desde `/mnt/c` (sistema Windows) es lento. Para mejor rendimiento, coloca tu código dentro del sistema de archivos nativo de WSL (`~/proyecto/` en Ubuntu, no `/mnt/c/proyecto`).
- **Firewall**: Windows Defender Firewall puede bloquear el acceso a puertos expuestos. Usa `localhost` para acceder a contenedores.
- **Contenedores Windows**: Docker Desktop también soporta contenedores nativos de Windows, pero el 99% del ecosistema usa Linux. Mantén el modo por defecto (Linux containers en WSL2).

---

### 1.5 Verificación post-instalación completa

Ejecuta estos tres comandos tras cualquier instalación:

#### `docker version`

```bash
docker version
```

Output típico (abreviado con anotaciones):

```
Client: Docker Engine - Community          # ← CLI instalado en tu máquina
 Version:           27.1.1                # ← Versión del CLI
 API version:       1.46                  # ← Versión de la API Docker
 Go version:        go1.22.5              # ← Go con el que se compiló el CLI
 Git commit:        6312585               # ← Commit exacto del build
 Built:             Tue Jul 23 18:00:00 2024  # ← Fecha de compilación
 OS/Arch:           linux/amd64           # ← Tu sistema operativo y arquitectura
 Context:           default               # ← Contexto Docker activo (default)

Server: Docker Engine - Community          # ← Daemon corriendo en el servidor
 Engine:
  Version:          27.1.1                # ← Versión del daemon
  API version:      1.46 (minimum version 1.24) # ← Compatibilidad API
  Go version:       go1.22.5
  Git commit:       cc13f95
  Built:            Tue Jul 23 18:00:00 2024
  OS/Arch:          linux/amd64           # ← Arquitectura del servidor
  Experimental:     false                 # ← Funciones experimentales desactivadas
 containerd:                              # ← Runtime de contenedores
  Version:          1.7.19
 runc:                                    # ← OCI runtime de bajo nivel
  Version:          1.1.13
 docker-init:                             # ← Proceso init (PID 1) en contenedores
  Version:          0.19.0
```

**Puntos clave:**
- Si `Server` NO aparece, el daemon no está corriendo o no tienes permisos. Revisa `sudo systemctl status docker`.
- Las APIs de Client y Server pueden diferir, pero el servidor mantiene compatibilidad hacia atrás (mínimo `1.24`).
- `containerd` gestiona el ciclo de vida de contenedores. `runc` crea y ejecuta contenedores OCI.

#### `docker info`

```bash
docker info
```

Este comando muestra la configuración global del daemon Docker. Vamos a desglosar CADA línea relevante del output:

```
Client: Docker Engine - Community
 Version:    27.1.1
 Context:    default              # Contexto activo (docker context ls)
 Debug Mode: false                # Modo debug del CLI (docker -D info)
 Plugins:
  buildx: Docker Buildx (Docker Inc.)    # Plugin de builds avanzados
  compose: Docker Compose (Docker Inc.)  # Plugin de compose v2

Server:
 Containers: 3                    # Contenedores totales (running + stopped)
  Running: 0                      # Contenedores corriendo ahora mismo
  Paused: 0                       # Contenedores pausados
  Stopped: 3                      # Contenedores detenidos
 Images: 8                        # Imágenes almacenadas localmente

 Server Version: 27.1.1           # Versión del daemon
 Storage Driver: overlay2         # ← CRÍTICO: driver de almacenamiento
                                   # overlay2: el estándar moderno
                                   # aufs: legacy (Ubuntu 16.04)
                                   # devicemapper: legacy (RHEL 7)
                                   # btrfs/zfs: para casos específicos
  Backing Filesystem: extfs       # Sistema de archivos del host (ext4/xfs/btrfs)
  Supports d_type: true           # Necesario para overlay2 (entry type en dirs)
  Using metacopy: false           # Copy-on-write optimizado (requiere kernel 4.19+)
  Native Overlay Diff: true       # Usa diff nativo del kernel (más rápido)
  userxattr: false                # Extended attributes de usuario

 Logging Driver: json-file        # Driver de logs por defecto
                                   # json-file: logs en /var/lib/docker/containers/<id>/<id>-json.log
                                   # syslog, journald, fluentd, gelf, splunk...
 Cgroup Driver: systemd           # Driver de cgroups
                                   # systemd: recomendado para systemd como init
                                   # cgroupfs: legacy, usar solo sin systemd
 Cgroup Version: 2                # cgroups v2 (unified hierarchy)

 Plugins:
  Volume: local                   # Drivers de volumen cargados
  Network: bridge host ipvlan     # Drivers de red cargados
     macvlan null overlay
  Log: awslogs fluentd gcplogs    # Drivers de logging cargados
     gelf journald json-file
     local splunk syslog

 Swarm: inactive                  # Modo Swarm (clustering) desactivado

 Runtimes: io.containerd.runc.v2  # Runtime por defecto
           runc                   # Alias compatible
 Default Runtime: runc            # Runtime usado si no se especifica

 Init Binary: docker-init         # tini, el proceso init dentro de contenedores
 containerd version: 1.7.19
 runc version: 1.1.13
 init version: 0.19.0

 Security Options:                # Opciones de seguridad del daemon
  seccomp
   Profile: builtin               # Perfil seccomp por defecto (bloquea ~60 syscalls)
  cgroupns
  apparmor                        # AppArmor activo (en Ubuntu/Debian)

 Kernel Version: 6.5.0-35-generic
 Operating System: Ubuntu 24.04 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 8                          # CPUs visibles para Docker
 Total Memory: 15.54GiB           # RAM total disponible para Docker
 Name: mi-servidor                # Hostname del daemon (importante para Swarm)
 ID: A1B2:C3D4:E5F6:...          # ID único del nodo Docker

 Docker Root Dir: /var/lib/docker # ← DIRECTORIO RAIZ de Docker
                                   # /var/lib/docker/containers/ → metadata contenedores
                                   # /var/lib/docker/image/    → imágenes y capas
                                   # /var/lib/docker/volumes/  → volúmenes
                                   # /var/lib/docker/overlay2/ → capas de overlay2

 Debug Mode: false                # Debug del daemon (cambiar en daemon.json)
 HTTP Proxy:                      # Proxy HTTP configurado (vacío = sin proxy)
 HTTPS Proxy:
 No Proxy:
 Registry: https://index.docker.io/v1/  # Registry por defecto para pulls

 Experimental: false              # Funcionalidades experimentales

 Insecure Registries:             # Registries sin TLS (no usar en producción)
  127.0.0.0/8

 Registry Mirrors:                # Mirrors para acelerar pulls
  https://mirror.gcr.io

 Live Restore Enabled: false      # Mantener contenedores vivos si el daemon muere
```

**Interpretación de campos clave:**

| Campo | Significado | Impacto |
|-------|-------------|---------|
| `Storage Driver: overlay2` | Sistema de archivos en capas. `overlay2` es el más moderno y estable. | Usa menos inodes y espacio que AUFS. Requiere kernel 4.0+. |
| `Cgroup Driver: systemd` | Docker delega cgroups a systemd. | Evita conflictos si systemd también gestiona cgroups. Recomendado para producción. |
| `Docker Root Dir` | Todo el estado de Docker vive aquí. | Particiona este directorio separado si usas Docker intensivamente. |
| `Live Restore Enabled: false` | Si lo activas, los contenedores sobreviven a un reinicio del daemon. | Útil en actualizaciones sin downtime. |
| `Security Options` | Capas de seguridad activas (seccomp, AppArmor, cgroupns). | Añaden restricciones que pueden necesitar relajarse con `--security-opt`. |

#### `docker run hello-world`

```bash
docker run hello-world
```

Output:

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete
Digest: sha256:53641cd209a4fecfc68e21a99871ce8c6920b2e7502df0a20671c6fccc73a7c6
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

**Qué acaba de pasar (diagrama de secuencia):**

```
Terminal         docker CLI         dockerd           Registry
  │                  │                  │                  │
  │  docker run      │                  │                  │
  │  hello-world     │                  │                  │
  │─────────────────►│                  │                  │
  │                  │  POST /containers/create           │
  │                  │─────────────────►│                  │
  │                  │                  │  ¿Tengo imagen?  │
  │                  │                  │  Busca en local  │
  │                  │                  │  → NO encontrada │
  │                  │                  │                  │
  │                  │                  │  GET /v2/library/│
  │                  │                  │  hello-world/    │
  │                  │                  │  manifests/latest│
  │                  │                  │─────────────────►│
  │                  │                  │     manifiesto   │
  │                  │                  │◄─────────────────│
  │                  │                  │                  │
  │                  │                  │  Descarga capas  │
  │                  │                  │─────────────────►│
  │                  │                  │◄─────────────────│
  │                  │                  │                  │
  │                  │                  │  Crea contenedor │
  │                  │                  │  (namespaces,    │
  │                  │                  │   cgroups...)    │
  │                  │                  │                  │
  │                  │                  │  Ejecuta /hello  │
  │                  │                  │  (dentro del     │
  │                  │                  │   contenedor)    │
  │                  │                  │                  │
  │                  │    stdout/stderr │                  │
  │                  │◄─────────────────│                  │
  │   "Hello from    │                  │                  │
  │    Docker!"       │                  │                  │
  │◄─────────────────│                  │                  │
  │                  │                  │                  │
  │                  │                  │  Contenedor      │
  │                  │                  │  termina (exit 0)│
  │                  │                  │  → estado stopped │
```

Resumen de los 4 pasos invisibles:
1. **El CLI se comunica con el daemon** vía socket Unix (`/var/run/docker.sock`) o TCP.
2. **El daemon busca la imagen** en su almacenamiento local. Si no la encuentra, la descarga del registry configurado (por defecto Docker Hub).
3. **El daemon crea el contenedor** asignando namespaces, cgroups, sistema de archivos aislado y red.
4. **El daemon ejecuta el binario** definido en la imagen y redirige la salida estándar al CLI, que la imprime en tu terminal.

---

## 2. Anatomía de `docker run`

`docker run` es el comando más importante de Docker. Es la suma de `docker create` + `docker start` + `docker attach` (opcional) en una sola instrucción. Entenderlo a fondo es dominar Docker.

### 2.1 Sintaxis completa

```
docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...]
           ─┬─────── ─────────┬───────── ───┬─── ───┬───
            │                 │              │       │
            │                 │              │       └── Argumentos del comando
            │                 │              └── Comando a ejecutar (opcional)
            │                 └── Imagen y versión
            └── Flags de configuración del contenedor
```

El `[COMMAND]` es opcional: si no lo especificas, se ejecuta el `CMD` o `ENTRYPOINT` definido en la imagen. Si lo especificas, **sobrescribes** el `CMD` de la imagen (no el `ENTRYPOINT`, que solo se sobrescribe con `--entrypoint`).

Ejemplo básico:

```bash
docker run ubuntu:22.04 echo "Hola desde Ubuntu"
# IMAGE=ubuntu:22.04, COMMAND=echo, ARG="Hola desde Ubuntu"
```

---

### 2.2 Flags esenciales explicadas con ejemplos

#### `-d` / `--detach` — Ejecutar en background

```bash
# Sin -d: el contenedor se "pega" a tu terminal
docker run nginx
# Presiona Ctrl+C y el contenedor se para.

# Con -d: el contenedor corre en segundo plano
docker run -d nginx
# → 7f8a3b2c1d4e...  (devuelve el ID del contenedor y libera la terminal)
```

El contenedor sigue corriendo aunque cierres la terminal (a menos que uses `--rm` combinado con `-it` sin `-d`). Para volver a "entrar" a un contenedor detachado, usas `docker attach` o `docker exec`.

#### `-it` — Terminal interactiva

```bash
# -i: interactive (mantener STDIN abierto aunque no esté attachado)
# -t: allocate a pseudo-TTY (emular una terminal con soporte de colores, señales, etc.)

docker run -it ubuntu bash
# root@a1b2c3d4e5f6:/#   ← prompt dentro del contenedor
# Ahora puedes ejecutar comandos interactivamente
# exit → sale del bash y PARA el contenedor (porque el proceso PID 1 termina)
```

**Diferencia `-it` vs solo `-i`:**

```bash
# Solo -i: STDIN abierto, pero sin TTY → sin prompt, sin colores, Ctrl+C no funciona bien
echo "ls" | docker run -i ubuntu bash
# Ejecuta ls y muestra la salida, pero no ves un prompt

# Con -it: TTY completo, colores, señales Unix, Ctrl+C, Ctrl+D, autocompletado
docker run -it ubuntu bash
```

**Al salir del bash:**
- Si escribes `exit` o presionas `Ctrl+D`: el proceso bash termina con exit code 0, por lo que el contenedor se para.
- Si presionas `Ctrl+P` seguido de `Ctrl+Q`: te "despegas" de la terminal sin parar el contenedor (solo funciona con `-it`).

#### `--name` — Nombrar contenedores

```bash
# Sin --name: Docker asigna nombres aleatorios (fervent_hopper, nostalgic_borg, ...)
docker run -d nginx
docker ps
# NAMES
# nostalgic_borg   ← nombre aleatorio: adjetivo + apellido de científico/ingeniero

# Con --name: nombre significativo
docker run -d --name mi-nginx nginx
docker run -d --name mi-nginx nginx
# Error: container name "mi-nginx" is already in use by container abc123

# Los nombres deben ser ÚNICOS en el host
```

**Buenas prácticas para nombres:**
- Usa nombres descriptivos: `db-postgres-prod`, `api-backend-v2`, `cache-redis-session`.
- Incluye entorno si es relevante: `nginx-staging`, `nginx-prod`.
- No uses nombres que parezcan IDs (hexadecimal puro).
- En scripts, usa `--name` para luego referenciar por nombre en lugar de ID.

#### `-p` / `--publish` — Mapeo de puertos

```
Sintaxis: -p [HOST_IP:]HOST_PORT:CONTAINER_PORT[/PROTOCOL]

┌──────────────────────────────────────────────┐
│                   HOST                       │
│  ┌───────────────────────────────────────┐   │
│  │ Puerto 8080                           │   │
│  │   │                                   │   │
│  │   │ docker-proxy / iptables DNAT      │   │
│  │   ▼                                   │   │
│  │  ┌──────── CONTAINER ──────────┐      │   │
│  │  │ Puerto 80                   │      │   │
│  │  │   ▲                         │      │   │
│  │  │   │ Nginx escucha en :80    │      │   │
│  │  │   │                         │      │   │
│  │  └─────────────────────────────┘      │   │
│  │                                       │   │
│  │  localhost:8080 → contenedor:80       │   │
│  └───────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

```bash
# Formato básico: HOST:CONTAINED — no es "host:container" sino "puerto_host:puerto_contenedor"
docker run -d -p 8080:80 nginx    # localhost:8080 → contenedor:80
docker run -d -p 3000:3000 node-app

# Solo especificar puerto del contenedor: Docker asigna un puerto aleatorio del host
docker run -d -P nginx            # -P publica TODOS los puertos EXPOSE de la imagen
# localhost:32768 → contenedor:80  (puerto alto aleatorio, rango 32768-60999)

# Enlazar a IP específica del host
docker run -d -p 127.0.0.1:8080:80 nginx    # Solo accesible desde localhost
docker run -d -p 0.0.0.0:8080:80 nginx      # Accesible desde cualquier IP (y red externa)

# Protocolo UDP
docker run -d -p 53:53/udp bind9-dns

# Rango de puertos
docker run -d -p 8000-8010:8000-8010 my-api

# Múltiples mapeos
docker run -d -p 80:80 -p 443:443 -p 3306:3306 mi-stack
```

**Verificación:**

```bash
docker port mi-nginx
# 80/tcp -> 0.0.0.0:8080
# 80/tcp -> [::]:8080

curl localhost:8080
# <html>... página de bienvenida de Nginx
```

#### `-v` / `--volume` — Montar volúmenes

```bash
# Sintaxis: -v [HOST_PATH:]CONTAINER_PATH[:OPTIONS]

# Bind mount: montar directorio del host
docker run -d -v /home/user/app:/usr/share/nginx/html nginx
# Lo que pongas en /home/user/app aparece en el contenedor en /.../html

# Volume gestionado por Docker (sin HOST_PATH)
docker run -d -v mi-volumen:/var/lib/mysql mysql:8
# Docker crea y gestiona el volumen en /var/lib/docker/volumes/mi-volumen/

# Solo lectura (ro)
docker run -d -v /home/user/config:/etc/nginx/conf.d:ro nginx

# Múltiples montajes
docker run -d \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  -v $(pwd)/logs:/var/log/nginx \
  -v mi-certificados:/etc/ssl:ro \
  nginx
```

**Nota importante**: en macOS y Windows (Docker Desktop), los bind mounts pueden tener problemas de rendimiento. Para desarrollo, prefiere montar directorios dentro de WSL2 (Windows) o usa `delegated`/`cached` en macOS:

```bash
docker run -d -v /Users/user/proyecto:/app:delegated node
# :delegated: el host "manda" cambios al contenedor con retraso (mejor rendimiento)
# :cached: el contenedor "ve" cambios del host con retraso
# :consistent: consistencia total (por defecto, más lento)
```

#### `-e` / `--env` — Variables de entorno

```bash
# Una variable
docker run -d -e MYSQL_ROOT_PASSWORD=secreto mysql:8

# Múltiples variables
docker run -d \
  -e MYSQL_ROOT_PASSWORD=secreto \
  -e MYSQL_DATABASE=miapp \
  -e MYSQL_USER=usuario \
  -e MYSQL_PASSWORD=pass123 \
  mysql:8

# Desde archivo (--env-file)
echo "MYSQL_ROOT_PASSWORD=secreto" > mysql.env
echo "MYSQL_DATABASE=miapp" >> mysql.env
docker run -d --env-file mysql.env mysql:8

# Combinar archivo + variables sueltas (ganan las de -e)
docker run -d --env-file base.env -e API_KEY=sobrescribe mysql:8
```

**Advertencia de seguridad**: las variables de entorno son visibles en `docker inspect`. NUNCA pases secretos de producción con `-e`. Usa Docker Secrets (Swarm) o Kubernetes Secrets (Capítulos 8 y 10).

```bash
# ¡MAL! Tu contraseña es visible para cualquier usuario con acceso al daemon
docker run -d -e DB_PASSWORD=supersecreto mysql:8
docker inspect mi-mysql | grep DB_PASSWORD
# "DB_PASSWORD=supersecreto"

# MEJOR: archivo .env en desarrollo, secrets en producción
docker run -d --env-file .env mysql:8
# .env en .gitignore (NUNCA commitear .env con secretos)
```

#### `--rm` — Auto-eliminar al parar

```bash
# Sin --rm: el contenedor queda stopped después de terminar
docker run ubuntu echo "hola"
docker ps -a
# CONTAINER ID   IMAGE    COMMAND       CREATED         STATUS                     NAMES
# a1b2c3d4e5f6   ubuntu   "echo hola"   5 seconds ago   Exited (0) 5 seconds ago   cool_galileo

# Con --rm: el contenedor desaparece al terminar
docker run --rm ubuntu echo "hola"
docker ps -a
# (vacío)

# --rm es ideal para:
# - Tareas puntuales: docker run --rm alpine wget -qO- https://api.ipify.org
# - Scripts de inicialización: docker run --rm -v ./db:/flyway/sql flyway/flyway migrate
# - Compilación temporal: docker run --rm -v $(pwd):/src -w /src golang:1.22 go build -o app .
# - Clientes efímeros: docker run --rm -it redis redis-cli -h redis-server
```

#### `--restart` — Política de reinicio

```bash
# no (por defecto): nunca reiniciar automáticamente
docker run -d --restart no nginx

# on-failure[:max-retries]: reiniciar solo si el contenedor termina con error (exit code != 0)
docker run -d --restart on-failure:3 nginx
# Si nginx crashea, Docker lo reinicia hasta 3 veces, luego desiste.

# always: reiniciar SIEMPRE (incluso si fue parado manualmente con docker stop)
docker run -d --restart always nginx
# Si haces docker stop, el contenedor NO se reinicia solo en ese momento,
# pero SÍ cuando el daemon Docker arranca (boot del sistema).

# unless-stopped: reiniciar siempre, excepto si fue parado manualmente
docker run -d --restart unless-stopped nginx
# Recomendado para la mayoría de servicios. Si paras manualmente, no se reinicia
# al bootear el sistema. Si crashea o el daemon se reinicia, SÍ se reinicia.
```

**Tabla de decisión:**

| Política | Contenedor se para por error | `docker stop` | Boot del sistema |
|---|---|---|---|
| `no` | ❌ no reinicia | ❌ | ❌ |
| `on-failure` | ✅ reinicia (hasta N veces) | ❌ | ❌ |
| `always` | ✅ reinicia | ❌ (pero sí al boot) | ✅ siempre |
| `unless-stopped` | ✅ reinicia | ❌ (respeta la parada) | ✅ excepto parados |

#### `--network` — Conectar a red específica

```bash
# Crear una red personalizada (recomendado para DNS automático)
docker network create mi-red

# Ejecutar contenedores en esa red
docker run -d --network mi-red --name db mysql:8
docker run -d --network mi-red --name app mi-api

# Dentro de 'app', puedo hacer ping a 'db' porque están en la misma red definida por usuario
docker exec app ping db
# PING db (172.18.0.2) 56(84) bytes of data.
# 64 bytes from db.mi-red (172.18.0.2): icmp_seq=1 ttl=64 time=0.089 ms

# Tipos de red:
docker run -d --network host nginx         # Comparte la red del host (sin aislamiento)
docker run -d --network none nginx         # Sin acceso a red
docker run -d --network bridge nginx       # Red bridge por defecto (NO tiene DNS automático)
docker run -d --network container:otro-nginx nginx  # Comparte la red de otro contenedor
```

#### `--cpus` y `--memory` — Límites de recursos

```bash
# Limitar CPU: --cpus (número de CPUs virtuales, puede ser decimal)
docker run -d --cpus 1.5 nginx              # Máximo 1.5 CPUs
docker run -d --cpus 0.5 nginx              # Máximo medio CPU

# Limitar memoria:
docker run -d --memory 512m nginx            # 512 MB máximo
docker run -d --memory 1g nginx              # 1 GB máximo
docker run -d --memory 256m --memory-swap 512m nginx  # 256 MB RAM + 256 MB swap

# Sin límites: el contenedor puede usar TODA la memoria y CPU del host
# → un contenedor mal programado puede tumbar todo el servidor

# Ver límites actuales:
docker stats --no-stream mi-nginx
# CONTAINER   CPU %     MEM USAGE / LIMIT   MEM %     NET I/O    BLOCK I/O
# mi-nginx    0.01%     2.5MiB / 512MiB      0.49%     1.2kB / 0B  0B / 0B
```

**Recomendación**: SIEMPRE establece límites en producción. Sin límites, un memory leak en un servicio puede causar OOM (Out Of Memory) del kernel y matar procesos aleatorios del host.

#### `--user` — Identidad del proceso

```bash
# Por defecto, los procesos en el contenedor corren como root (UID 0)
docker run -d nginx
docker top mi-nginx
# UID   PID   PPID  C  STIME  TTY  TIME     CMD
# root  1234  1200  0  10:30  ?    00:00:00 nginx: master process

# Ejecutar como usuario específico
docker run -d --user 1000:1000 nginx
# Nginx intenta correr como UID 1000, GID 1000
# Puede fallar si los archivos no son accesibles para ese usuario

docker run -d --user www-data nginx
# Usuario por nombre (debe existir en /etc/passwd de la imagen)

# Formato: --user UID:GID
docker run -d --user 1001:1001 mi-api
```

**Por seguridad**: si un atacante compromete un contenedor que corre como root y logra escapar, obtiene root en el host. Con `--user`, limita el daño (Capítulo 10).

#### `--workdir` / `-w` — Directorio de trabajo

```bash
# Establecer el directorio de trabajo dentro del contenedor
docker run --rm -w /app alpine pwd
# /app

# Comando ejecutado en ese directorio
docker run --rm -v $(pwd):/app -w /app golang:1.22 go build -o /app/binario main.go

# Equivalente a:
# cd /app && go build -o /app/binario main.go
```

---

### 2.3 Ejemplo completo: Nginx con todas las flags

Vamos a desplegar Nginx combinando todas las flags explicadas:

```bash
# Paso previo: crear directorios y archivos de configuración
mkdir -p ~/nginx-lab/{html,conf.d,logs}

cat > ~/nginx-lab/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Docker Nginx Lab</title>
    <style>
        body { font-family: sans-serif; max-width: 800px; margin: 50px auto; padding: 20px; }
        h1 { color: #009639; }  /* Verde Docker */
    </style>
</head>
<body>
    <h1>Nginx está corriendo en Docker</h1>
    <p>Servido desde el contenedor <code>nginx-lab</code>.</p>
    <p>Timestamp: <span id="time"></span></p>
    <script>document.getElementById('time').textContent = new Date().toISOString();</script>
</body>
</html>
EOF

cat > ~/nginx-lab/conf.d/default.conf << 'EOF'
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    location /api {
        return 200 '{"status":"ok","service":"nginx-lab"}\n';
        add_header Content-Type application/json;
    }
}
EOF

# Crear red
docker network create nginx-net

# Desplegar contenedor con TODAS las flags
docker run -d \
  --name nginx-lab \
  --hostname nginx-lab-server \
  --restart unless-stopped \
  --cpus 0.5 \
  --memory 128m \
  --user 101:101 \
  --workdir /usr/share/nginx/html \
  -p 8080:80 \
  -v ~/nginx-lab/html:/usr/share/nginx/html:ro \
  -v ~/nginx-lab/conf.d:/etc/nginx/conf.d:ro \
  -v ~/nginx-lab/logs:/var/log/nginx \
  -e NGINX_HOST=localhost \
  -e NGINX_PORT=80 \
  --network nginx-net \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --dns 8.8.8.8 \
  --label environment=lab \
  --label service=web \
  nginx:alpine
```

**Desglose de CADA flag en este comando:**

| Flag | Valor | Explicación |
|---|---|---|
| `-d` | — | Detached: el contenedor corre en background |
| `--name` | `nginx-lab` | Nombre del contenedor (referencia en otros comandos) |
| `--hostname` | `nginx-lab-server` | Hostname dentro del contenedor (visible en `hostname`) |
| `--restart` | `unless-stopped` | Reiniciar si crashea o al boot, excepto si fue parado manualmente |
| `--cpus` | `0.5` | Máximo medio núcleo de CPU |
| `--memory` | `128m` | Máximo 128 MB de RAM. Si lo excede, el kernel lo mata (OOM) |
| `--user` | `101:101` | Ejecutar como UID 101 (nginx en Alpine). Más seguro que root |
| `--workdir` | `/usr/share/nginx/html` | Directorio base para cualquier comando `docker exec` |
| `-p` | `8080:80` | Mapear puerto 8080 del host al puerto 80 del contenedor |
| `-v ...html:ro` | bind mount | Contenido estático. `:ro` = solo lectura |
| `-v ...conf.d:ro` | bind mount | Configuración de Nginx. `:ro` = solo lectura |
| `-v ...logs` | bind mount | Logs persistentes. Lectura/escritura |
| `-e NGINX_HOST` | `localhost` | Variable de entorno (la imagen nginx la soporta en templates) |
| `-e NGINX_PORT` | `80` | Variable de entorno |
| `--network` | `nginx-net` | Conecta a la red `nginx-net` (DNS automático) |
| `--log-driver` | `json-file` | Formato de logs (JSON, una línea por entrada) |
| `--log-opt max-size` | `10m` | Rotar logs cuando el archivo llegue a 10 MB |
| `--log-opt max-file` | `3` | Mantener máximo 3 archivos de log rotados |
| `--dns` | `8.8.8.8` | Servidor DNS para este contenedor (sobrescribe el por defecto) |
| `--label` | `environment=lab` | Metadato (visible en `docker inspect`, filtrable por `docker ps --filter`) |
| `--label` | `service=web` | Otro metadato |
| `nginx:alpine` | — | Imagen: nginx basado en Alpine Linux (~5 MB base vs ~28 MB Debian) |

**Verificar el despliegue:**

```bash
# Ver que está corriendo
docker ps --filter name=nginx-lab

# Ver logs
docker logs nginx-lab
# /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty...
# /docker-entrypoint.sh: Configuration complete; ready for start up
# 2024-07-30T10:00:00Z [notice] 1#1: nginx/1.27.0
# 2024-07-30T10:00:00Z [notice] 1#1: start worker process 31

# Acceder al contenido
curl localhost:8080
# <!DOCTYPE html>... página de prueba

# Health check
curl localhost:8080/health
# healthy

# API falsa
curl localhost:8080/api
# {"status":"ok","service":"nginx-lab"}

# Inspeccionar estado
docker inspect nginx-lab --format '{{.State.Status}}'
# running

# Stats
docker stats --no-stream nginx-lab
# CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O       BLOCK I/O
# nginx-lab   0.00%   4.2MiB / 128MiB     3.28%   1.5kB / 1kB   8.2kB / 0B

# Ver desde el navegador: http://localhost:8080
```

---

## 3. Ciclo de Vida de un Contenedor

Un contenedor no es un proceso cualquiera. Docker gestiona su ciclo de vida mediante estados bien definidos.

### 3.1 Diagrama de estados

```
                        ┌────────────┐
                        │  IMAGEN    │
                        │ (pull/cache│
                        │  local)    │
                        └─────┬──────┘
                              │
                              │ docker create
                              ▼
        ┌─────────────────────────────────────────┐
        │              CREATED                     │
        │  Sistema de archivos montado             │
        │  Red configurada                         │
        │  No hay proceso corriendo                │
        └────────────┬─────────────────────────────┘
                     │
                     │ docker start
                     ▼
        ┌─────────────────────────────────────────┐
        │              RUNNING                     │
        │  Proceso PID 1 ejecutándose              │
        │  ┌──────────────┐      ┌──────────────┐ │
        │  │  Normal      │      │  docker      │ │
        │  │  execution   │      │  pause       │ │
        │  └──────┬───────┘      └──────┬───────┘ │
        │         │                     │         │
        └─────────┼─────────────────────┼─────────┘
                  │                     │
                  │                     ▼
                  │           ┌─────────────────────┐
                  │           │      PAUSED          │
                  │           │  Proceso congelado   │
                  │           │  (SIGSTOP al PID 1)  │
                  │           │  docker unpause →    │
                  │           │  vuelve a RUNNING    │
                  │           └─────────────────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
     │ Proceso    │ docker     │ docker
     │ termina    │ stop       │ kill
     │ por sí     │ (SIGTERM   │ (SIGKILL
     │ solo       │  +grace)   │  inmediato)
     ▼            ▼            ▼
    ┌──────────────────────────────────────────────┐
    │                 STOPPED / EXITED              │
    │  Proceso terminado                           │
    │  Sistema de archivos intacto                 │
    │  Metadatos conservados (logs, exit code...)  │
    │  docker start → vuelve a RUNNING             │
    │  docker rm → pasa a DELETED                  │
    └──────────────────────┬───────────────────────┘
                           │
                           │ docker rm
                           ▼
                    ┌──────────────┐
                    │   DELETED    │
                    │  Todo borrado│
                    │  (incluyendo │
                    │   capa R/W)  │
                    └──────────────┘
                              │
                              │ (imagen ↔ capas de solo lectura)
                              ▼
                         ┌─────────────────┐
                         │  IMAGEN (capas  │
                         │  read-only)     │
                         │  PERMANECE      │
                         └─────────────────┘
```

### 3.2 Comandos para cada transición

```bash
# CREATED: asignar recursos pero NO ejecutar el proceso
docker create --name mi-contenedor ubuntu sleep 60
# → devuelve el ID del contenedor
docker ps -a
# STATUS: Created

# RUNNING (desde CREATED): ejecutar el proceso
docker start mi-contenedor
docker ps
# STATUS: Up 5 seconds

# RUNNING (directo, sin CREATED intermedio): crear + start + attach
docker run --name mi-contenedor ubuntu sleep 60

# PAUSED: congelar el proceso (SIGSTOP)
docker pause mi-contenedor
docker ps
# STATUS: Up 5 seconds (Paused)

# RUNNING (desde PAUSED): descongelar (SIGCONT)
docker unpause mi-contenedor

# STOPPED: parada graceful
docker stop mi-contenedor
# STATUS: Exited (0) 2 seconds ago

# RUNNING (desde STOPPED): volver a arrancar
docker start mi-contenedor

# DELETED: eliminación permanente
docker rm mi-contenedor
# Error: You cannot remove a running container. Stop it first or use -f

docker stop mi-contenedor && docker rm mi-contenedor
# Eliminado

# DELETED forzado (para y borra en un paso)
docker rm -f mi-contenedor
```

---

### 3.3 `docker stop` vs `docker kill` — La diferencia crucial

```
┌─────────────────────────────────────────────────────────────────┐
│                    DIFERENCIA STOP vs KILL                      │
├──────────────────┬──────────────────┬───────────────────────────┤
│   docker stop    │   docker kill    │                           │
├──────────────────┼──────────────────┼───────────────────────────┤
│ Señal: SIGTERM   │ Señal: SIGKILL   │                           │
│ (señal 15)       │ (señal 9)        │                           │
├──────────────────┼──────────────────┼───────────────────────────┤
│ El proceso PUEDE │ El proceso NO    │                           │
│ manejar la señal │ puede manejar    │                           │
│ (graceful)       │ la señal (muere  │                           │
│                  │ instantáneamente)│                           │
├──────────────────┼──────────────────┼───────────────────────────┤
│ Grace period:    │ Sin espera:      │                           │
│ 10s por defecto  │ inmediato        │                           │
│ (--stop-timeout) │                  │                           │
├──────────────────┼──────────────────┼───────────────────────────┤
│ Para producción: │ Para emergencias │                           │
│ bases de datos,  │ o contenedores   │                           │
│ apps con estado  │ que no responden │                           │
└──────────────────┴──────────────────┴───────────────────────────┘
```

#### `docker stop`: parada graceful

```bash
# Parada estándar con tiempo de espera
docker stop mi-contenedor
```

**Qué ocurre internamente:**

```
1. Docker envía SIGTERM al proceso PID 1 del contenedor.
2. El proceso recibe SIGTERM y PUEDE:
   - Cerrar conexiones de red
   - Finalizar transacciones en curso (DB)
   - Escribir datos pendientes a disco
   - Liberar recursos
   - Llamar a exit(0) o exit(1)
3. Docker espera --stop-timeout segundos (por defecto 10s).
4. Si el proceso sigue vivo después del timeout:
   Docker envía SIGKILL (fuerza la muerte).
```

**Ejemplo con timeout personalizado:**

```bash
# Dar 30 segundos para que el proceso cierre graceful
docker stop --time 30 mi-servidor

# Sin timeout, SIGKILL inmediato después del grace
docker stop --time 0 mi-servidor
# Comportamiento casi idéntico a docker kill (pero SIGTERM + 0s, no SIGKILL directo)
```

#### `docker kill`: muerte inmediata

```bash
# SIGKILL directo (por defecto)
docker kill mi-contenedor

# Señal personalizada
docker kill --signal SIGTERM mi-contenedor  # equivalente a docker stop (sin timeout)
docker kill --signal SIGHUP mi-contenedor   # recargar configuración (algunas apps)
docker kill --signal SIGUSR1 mi-contenedor  # señal definida por la app
docker kill --signal SIGINT mi-contenedor   # equivalente a Ctrl+C
```

**Señales comunes:**

| Señal | Número | Significado | Uso en contenedores |
|---|---|---|---|
| SIGTERM | 15 | Terminación graceful | `docker stop` por defecto |
| SIGKILL | 9 | Matar inmediatamente | `docker kill` por defecto |
| SIGINT | 2 | Interrupción (Ctrl+C) | `docker run -it` con Ctrl+C |
| SIGHUP | 1 | Hangup | Recargar configuración (nginx, gunicorn) |
| SIGQUIT | 3 | Quit con core dump | Depuración |
| SIGUSR1 | 10 | Definida por usuario | Rotación de logs, señales custom |
| SIGUSR2 | 12 | Definida por usuario | Señales custom |
| SIGSTOP | 19 | Pausar (no manejable) | `docker pause` |
| SIGCONT | 18 | Continuar | `docker unpause` |

#### Cuándo usar cada uno

```
docker stop  → Bases de datos (MySQL, PostgreSQL, MongoDB)
               Colas de mensajes (RabbitMQ, Redis)
               Servidores web con conexiones largas (WebSockets)
               Microservicios que deben cerrar graceful

docker kill  → Contenedor congelado (no responde a SIGTERM)
               Contenedor que debes matar YA (incidente de producción)
               Procesos batch que no tienen estado que perder
               Cuando el grace period es inaceptable (>10s es demasiado)
```

**Simulación práctica:**

```bash
# Terminal 1: Ejecutar contenedor que maneja SIGTERM
docker run --rm --name graceful -it ubuntu bash -c '
  trap "echo Recibí SIGTERM; sleep 2; echo Guardando estado...; exit 0" TERM
  echo "PID: $$, esperando señales..."
  while true; do sleep 1; done
'

# Terminal 2: Parar graceful (envía SIGTERM)
docker stop graceful
# Terminal 1 mostrará:
# Recibí SIGTERM
# Guardando estado...

# Terminal 2: Contenedor que NO maneja señales
docker run --rm --name no-graceful -it ubuntu bash -c '
  echo "PID: $$, no manejo señales..."
  while true; do sleep 1; done
'

# Terminal 2: docker stop
docker stop no-graceful
# Espera 10 segundos (timeout), luego SIGKILL

# Comparar exit codes:
docker wait graceful
# 0
docker wait no-graceful
# 137  (128 + 9; 9 = SIGKILL)
```

---

## 4. Comandos Esenciales del Día a Día

### 4.1 `docker ps` — Listar contenedores

```bash
# Contenedores corriendo
docker ps

# Todos los contenedores (incluyendo stopped, created)
docker ps -a

# Últimos N contenedores creados
docker ps -n 5

# Solo IDs (útil para scripts)
docker ps -q
docker ps -aq   # Todos los IDs

# Tamaños de disco
docker ps -s
# SIZE: tamaño de la capa R/W (archivos creados/modificados)
# SIZE virtual: tamaño de la imagen base + capa R/W

# Filtros avanzados
docker ps -a --filter "status=exited"
docker ps -a --filter "status=exited" --filter "exited=0"       # Exit code 0 (éxito)
docker ps -a --filter "status=exited" --filter "exited=137"     # Exit code 137 (SIGKILL)
docker ps --filter "name=nginx"                                  # Por nombre
docker ps --filter "label=environment=lab"                       # Por label
docker ps --filter "ancestor=nginx"                              # Hijos de una imagen
docker ps --filter "before=mi-contenedor"                        # Creados antes que...
docker ps --filter "since=mi-contenedor"                         # Creados después que...
docker ps --filter "publish=8080"                                # Que exponen puerto 8080
docker ps --filter "volume=/data"                                # Que montan /data
docker ps --filter "network=nginx-net"                           # En red específica

# Combinar filtros
docker ps -a --filter "status=exited" --filter "name=demo" -q

# Formato personalizado
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"
# table: alinear como tabla

docker ps --format '{{.ID}}: {{.Names}} - {{.Image}}'
# 7f8a3b: nginx-lab - nginx:alpine
```

#### Entendiendo el output de `docker ps`

```
CONTAINER ID   IMAGE          COMMAND                  CREATED        STATUS        PORTS                  NAMES
7f8a3b2c1d4e   nginx:alpine   "/docker-entrypoint.…"   5 hours ago    Up 5 hours    0.0.0.0:8080->80/tcp   nginx-lab
```

| Columna | Significado | Detalle |
|---|---|---|
| **CONTAINER ID** | Hash SHA256 truncado (12 caracteres) | Identificador único. El hash completo es de 64 caracteres hex. |
| **IMAGE** | Imagen usada | Incluye tag (por defecto `:latest` si se omite). |
| **COMMAND** | Comando ejecutado como PID 1 | Truncado a ~40 chars. Ver comando completo con `docker inspect`. |
| **CREATED** | Tiempo desde que se creó el contenedor | `5 hours ago`, `2 days ago`, `30 seconds ago`. |
| **STATUS** | Estado del contenedor | `Up X` (running X tiempo), `Exited (0) X ago`, `Created`, `Paused`. |
| **PORTS** | Mapeo de puertos | `HOST_IP:HOST_PORT->CONTAINER_PORT/PROTOCOL`. Vacío si no mapea. |
| **NAMES** | Nombre del contenedor | Asignado con `--name` o autogenerado por Docker. |

#### Explicación de STATUS en detalle

```
STATUS puede ser:

Up 5 hours                    → Corriendo 5 horas (saludable, sin healthcheck)
Up 5 hours (healthy)          → Corriendo y healthcheck pasa (Capítulo 4, HEALTHCHECK)
Up 5 hours (unhealthy)        → Corriendo pero healthcheck falla
Up 10 seconds (Paused)        → Corriendo pero pausado
Exited (0) 2 hours ago        → Terminó con éxito (exit code 0)
Exited (1) 2 hours ago        → Terminó con error (exit code 1)
Exited (137) 2 hours ago      → Matado por SIGKILL (9) → 128+9=137
Exited (143) 2 hours ago      → Matado por SIGTERM (15) → 128+15=143
Created                        → Creado pero nunca arrancó
Removing                       → En proceso de eliminación
Dead                           → El daemon no pudo recuperar su estado
```

La fórmula de exit codes para señales: **exit_code = 128 + signal_number**. Por ejemplo:
- `137 = 128 + 9` → SIGKILL (`docker kill`)
- `143 = 128 + 15` → SIGTERM (`docker stop` sin grace exitoso)
- `130 = 128 + 2` → SIGINT (Ctrl+C en terminal attachada)

---

### 4.2 `docker logs` — Leer los logs del contenedor

```bash
# Logs completos
docker logs nginx-lab

# Seguir logs en tiempo real (como tail -f)
docker logs -f nginx-lab

# Últimas N líneas
docker logs --tail 50 nginx-lab

# Solo logs desde cierta fecha
docker logs --since 2024-07-30T10:00:00 nginx-lab
docker logs --since 30m nginx-lab    # Últimos 30 minutos
docker logs --until 10m nginx-lab    # Hasta hace 10 minutos

# Con timestamps
docker logs -t nginx-lab
# 2024-07-30T10:00:00.123456789Z /docker-entrypoint.sh: Configuration complete

# Sin timestamps pero con fechas
docker logs --since 1h --until 30m nginx-lab
```

#### ¿Dónde se guardan los logs?

Los logs viven en el host, dentro del directorio raíz de Docker:

```bash
# Directorio raíz de Docker (se ve en docker info → Docker Root Dir)
DOCKER_ROOT=$(docker info --format '{{.DockerRootDir}}')
echo $DOCKER_ROOT
# /var/lib/docker

# Path de logs de un contenedor (json-file driver por defecto)
CONTAINER_ID=$(docker ps -aq --filter name=nginx-lab)
sudo ls -la $DOCKER_ROOT/containers/$CONTAINER_ID/
# .../containers/<container_id>/
#   <container_id>-json.log  ← AQUÍ están los logs

# Leer logs directamente del archivo
sudo tail -f $DOCKER_ROOT/containers/$CONTAINER_ID/$CONTAINER_ID-json.log
# {"log":"172.17.0.1 - - [30/Jul/2024:10:00:00 +0000] \"GET / HTTP/1.1\" 200 ...\n",...}
```

**Formato del archivo de log (json-file driver):**

```json
{"log":"/docker-entrypoint.sh: Configuration complete\n","stream":"stdout","time":"2024-07-30T10:00:00.123456789Z"}
{"log":"2024/07/30 10:00:01 [notice] 1#1: start worker process 31\n","stream":"stderr","time":"2024-07-30T10:00:01.234567890Z"}
{"log":"172.17.0.1 - - [30/Jul/2024:10:00:05] \"GET /health HTTP/1.1\" 200 8\n","stream":"stdout","time":"2024-07-30T10:00:05.345678901Z"}
```

Cada línea es un objeto JSON con:
- `log`: el contenido emitido por el proceso (incluye `\n`).
- `stream`: `stdout` o `stderr`.
- `time`: timestamp con nanosegundos en UTC.

**Limitaciones del driver `json-file`:**
- Los logs viven SOLO en el host local.
- Si el contenedor se borra (`docker rm`), los logs desaparecen.
- No hay rotación por tiempo, solo por tamaño (si configuras `--log-opt max-size`).

**Alternativas (drivers de logging):**
- `syslog`: enviar logs al syslog del sistema.
- `journald`: logs al journal de systemd.
- `fluentd`: logs a Fluentd (agregador de logs).
- `gelf`: Graylog Extended Log Format (para ELK/Graylog).
- `splunk`: directo a Splunk.
- `awslogs`: CloudWatch Logs (AWS).
- `gcplogs`: Google Cloud Logging (GCP).

Cambio de driver:

```bash
docker run -d --log-driver syslog --name mi-app nginx

# Configuración global en /etc/docker/daemon.json
{
  "log-driver": "fluentd",
  "log-opts": {
    "fluentd-address": "fluentd-host:24224",
    "tag": "docker.{{.Name}}"
  }
}
```

---

### 4.3 `docker exec` — Ejecutar comandos dentro de un contenedor

```bash
# Ejecutar bash interactivo (el caso más común)
docker exec -it nginx-lab bash
# root@nginx-lab-server:/usr/share/nginx/html#

# Ejecutar un comando y ver la salida
docker exec nginx-lab ls -la /etc/nginx/conf.d/
# -rw-r--r-- 1 root root 319 Jul 30 10:00 default.conf

# Ejecutar como usuario específico
docker exec -u www-data nginx-lab whoami
# www-data

# Con directorio de trabajo específico
docker exec -w /etc/nginx nginx-lab cat nginx.conf

# Con variables de entorno adicionales
docker exec -e MY_VAR=hola nginx-lab env | grep MY_VAR
# MY_VAR=hola

# Ejecutar un script completo (pipe a bash)
cat << 'EOF' | docker exec -i nginx-lab bash
echo "=== Diagnóstico de nginx ==="
nginx -v 2>&1
nginx -t 2>&1
ps aux 2>&1
netstat -tlnp 2>&1 || ss -tlnp
EOF
```

#### Diferencia entre `docker exec` y `docker attach`

```
┌─────────────────────────────────────────────────────────────────┐
│              docker exec     vs     docker attach               │
├─────────────────────┬───────────────────────────────────────────┤
│ docker exec         │ docker attach                             │
├─────────────────────┼───────────────────────────────────────────┤
│ Crea un PROCESO     │ Se "engancha" a la ENTRADA/SALIDA         │
│ NUEVO dentro del    │ del proceso PID 1 (el principal)          │
│ contenedor.         │                                           │
├─────────────────────┼───────────────────────────────────────────┤
│ Múltiples exec      │ Solo UN attach a la vez por cada          │
│ simultáneos.        │ STDIN/STDOUT (como pegarse a un screen).  │
├─────────────────────┼───────────────────────────────────────────┤
│ Al salir (exit),    │ Al salir (Ctrl+C o Ctrl+D), el PID 1     │
│ el proceso HIJO     │ recibe la señal → PUEDE MATAR el          │
│ termina. PID 1      │ contenedor si PID 1 reacciona a SIGTERM   │
│ sigue vivo.         │ (ejemplo: bash de ubuntu muere).          │
├─────────────────────┼───────────────────────────────────────────┤
│ Uso: inspeccionar,  │ Uso: ver la salida del proceso            │
│ debuggear, ejecutar │ principal en tiempo real (como             │
│ comandos.           │ estar viendo la consola del juego).       │
├─────────────────────┼───────────────────────────────────────────┤
│ Ejemplo:            │ Ejemplo:                                  │
│ docker exec -it     │ docker attach nginx-lab                   │
│   nginx-lab bash    │ (ves los logs de nginx en vivo)           │
└─────────────────────┴───────────────────────────────────────────┘
```

```bash
# Experimento: diferencia exec vs attach
# Terminal 1: contenedor que escribe cada segundo
docker run --name counter ubuntu bash -c 'for i in $(seq 1 100); do echo "Tick $i"; sleep 1; done'

# Terminal 2: attach → ves los ticks en vivo
docker attach counter
# Tick 1
# Tick 2
# Tick 3
# Ctrl+C → TERMINA el proceso PID 1 → contenedor se para

# Terminal 2: exec → entras con un bash nuevo
docker exec -it counter bash
# root@abc123:/# ps aux
# PID   USER   COMMAND
#   1   root   bash -c for i in ...   ← PID 1 sigue corriendo
#  15   root   bash                     ← TU bash nuevo (PID 15)
# root@abc123:/# exit
# El contenedor sigue corriendo (PID 1 intacto)
```

### 4.4 `docker inspect` — El JSON completo del contenedor

```bash
# JSON completo (enorme)
docker inspect nginx-lab

# Ver el primer nivel de claves
docker inspect nginx-lab | jq '.[0] | keys'
# [
#   "Id", "Created", "Path", "Args", "State", "Image",
#   "ResolvConfPath", "HostnamePath", "HostsPath", "LogPath",
#   "Name", "RestartCount", "Driver", "Platform", "MountLabel",
#   "ProcessLabel", "AppArmorProfile", "ExecIDs", "HostConfig",
#   "GraphDriver", "Mounts", "Config", "NetworkSettings"
# ]
```

#### Usar `--format` (Go templates) para extraer campos

Las plantillas Go te permiten extraer cualquier campo del JSON sin necesidad de `jq`:

```bash
# Campo simple
docker inspect --format '{{.State.Status}}' nginx-lab
# running

# Estado completo
docker inspect --format '{{json .State}}' nginx-lab | jq
# {
#   "Status": "running",
#   "Running": true,
#   "Paused": false,
#   "Restarting": false,
#   "OOMKilled": false,
#   "Dead": false,
#   "Pid": 45678,
#   "ExitCode": 0,
#   "Error": "",
#   "StartedAt": "2024-07-30T10:00:00.123456789Z",
#   "FinishedAt": "0001-01-01T00:00:00Z",
#   "Health": null
# }

# IP del contenedor
docker inspect --format '{{.NetworkSettings.IPAddress}}' nginx-lab
# 172.18.0.2

# Gateway y subred
docker inspect --format '{{range .NetworkSettings.Networks}}{{.Gateway}}{{end}}' nginx-lab
# 172.18.0.1

# Todos los puertos expuestos
docker inspect --format '{{json .Config.ExposedPorts}}' nginx-lab

# Montajes (bind mounts y volúmenes)
docker inspect --format '{{json .Mounts}}' nginx-lab | jq
# [
#   {
#     "Type": "bind",
#     "Source": "/home/user/nginx-lab/html",
#     "Destination": "/usr/share/nginx/html",
#     "Mode": "ro",
#     "RW": false,
#     "Propagation": "rprivate"
#   }
# ]

# Variables de entorno
docker inspect --format '{{range .Config.Env}}{{println .}}{{end}}' nginx-lab
# NGINX_HOST=localhost
# NGINX_PORT=80
# PATH=/usr/local/sbin:/usr/local/bin:...

# Labels
docker inspect --format '{{json .Config.Labels}}' nginx-lab | jq
# { "environment": "lab", "service": "web" }

# Path del archivo de log
docker inspect --format '{{.LogPath}}' nginx-lab
# /var/lib/docker/containers/7f8a3b.../7f8a3b...-json.log

# PID del proceso en el host
docker inspect --format '{{.State.Pid}}' nginx-lab
# 45678

# Timestamps (parsear Unix timestamp)
docker inspect --format '{{.Created}}' nginx-lab
# 2024-07-30T10:00:00.123456789Z

# Restart count y política
docker inspect --format 'Restarts: {{.RestartCount}} - Policy: {{.HostConfig.RestartPolicy.Name}}' nginx-lab
# Restarts: 0 - Policy: unless-stopped

# Tabla multi-contenedor
docker inspect --format 'table {{.Name}}\t{{.State.Status}}\t{{.NetworkSettings.IPAddress}}' $(docker ps -q)
# /nginx-lab       running   172.18.0.2
# /mysql-lab       running   172.18.0.3

# Verificar healthcheck (si existe)
docker inspect --format '{{.State.Health.Status}}' nginx-lab
# (vacío = sin healthcheck definido)
```

#### Receta: función alias útiles en `.bashrc`

```bash
# Agregar a ~/.bashrc o ~/.zshrc

# IP de un contenedor
dip() { docker inspect --format '{{.NetworkSettings.IPAddress}}' "$1"; }

# PID del proceso contenedor
dpid() { docker inspect --format '{{.State.Pid}}' "$1"; }

# Volúmenes montados
dvol() { docker inspect --format '{{range .Mounts}}{{.Source}} -> {{.Destination}} ({{.Mode}})\n{{end}}' "$1"; }

# Estado
dstatus() { docker inspect --format 'Container: {{.Name}} | Status: {{.State.Status}} | Started: {{.State.StartedAt}}' "$1"; }

# Puertos mapeados
dport() { docker inspect --format '{{range $p, $conf := .NetworkSettings.Ports}}{{$p}} -> {{(index $conf 0).HostPort}}\n{{end}}' "$1"; }

# Uso:
dip nginx-lab        # 172.18.0.2
dpid nginx-lab       # 45678
dvol nginx-lab       # /home/user/nginx-lab/html -> /usr/share/nginx/html (ro)
dstatus nginx-lab    # Container: /nginx-lab | Status: running | Started: 2024-07-30T...
dport nginx-lab      # 80/tcp -> 8080
```

### 4.5 `docker cp` — Copiar archivos entre host y contenedor

```bash
# HOST → CONTENEDOR
docker cp ./index.html nginx-lab:/usr/share/nginx/html/index.html
docker cp ./config/ nginx-lab:/etc/nginx/conf.d/    # Directorio completo

# CONTENEDOR → HOST
docker cp nginx-lab:/var/log/nginx/access.log ./access.log
docker cp nginx-lab:/etc/nginx/ ./nginx-backup/      # Directorio completo

# Entre contenedores (vía host temporal)
docker cp db-lab:/var/lib/mysql ./mysql-data/
docker cp ./mysql-data/ db-new:/var/lib/mysql/

# Con tar (empaquetar/desempaquetar) - útil para backups
docker cp nginx-lab:/usr/share/nginx/html - | tar -tv
# Lista el contenido sin extraer

docker cp nginx-lab:/usr/share/nginx/html - | gzip > nginx-html-backup.tar.gz
# Backup comprimido en un solo comando

gzip -d < backup.tar.gz | docker cp - nginx-new:/usr/share/nginx/html/
# Restaurar backup
```

**Limitación**: `docker cp` copia todo el archivo o directorio. No soporta wildcards (`*.log`). Para operaciones más complejas, usa `docker exec` con un comando de shell:

```bash
# Copiar solo ciertos archivos
docker exec nginx-lab find /var/log/nginx -name "access.log*" -exec cat {} \; > access.logs

# Backup de BD con compresión
docker exec mysql-lab mysqldump -uroot -psecreto miapp | gzip > backup-$(date +%Y%m%d).sql.gz
```

### 4.6 `docker diff` — Ver cambios en el sistema de archivos

```bash
# Crear contenedor y modificarlo
docker run --name test-diff ubuntu bash -c 'touch /nuevo.txt && echo "cambiado" >> /etc/hosts && rm /etc/legal'

# Ver qué archivos cambiaron
docker diff test-diff
# C /root                         ← Changed: directorio cambió
# A /root/.bash_history           ← Added: archivo nuevo
# C /etc                          ← Changed
# C /etc/hosts                    ← Changed: archivo modificado
# D /etc/legal                    ← Deleted: archivo eliminado
# A /nuevo.txt                    ← Added: archivo creado
```

**Indicadores:**

| Símbolo | Significado |
|---|---|
| **A** | Added: archivo o directorio añadido |
| **C** | Changed: archivo o directorio modificado |
| **D** | Deleted: archivo o directorio eliminado |

**Aplicaciones prácticas:**

```bash
# Auditar qué cambios hace una imagen sospechosa
docker run --rm --name audit ubuntu bash -c 'apt-get update -qq'
docker diff audit

# Ver si una app escribe archivos temporales inesperados
docker run --name app-test mi-api
sleep 10
docker diff app-test | grep "^A"     # Archivos nuevos
docker diff app-test | grep "^C"     # Archivos modificados

# Optimizar Dockerfile: ver qué escribe cada capa
docker run --rm --name step1 ubuntu bash -c 'apt-get update -qq'
docker diff step1
docker commit step1 mi-ubuntu:step1  # (no recomendado en producción)
```

### 4.7 `docker top` — Procesos corriendo en el contenedor

```bash
docker top nginx-lab
# UID    PID    PPID   C  STIME  TTY  TIME     CMD
# 101    45678  45650  0  10:00  ?    00:00:00 nginx: master process nginx -g daemon off;
# 101    45712  45678  0  10:00  ?    00:00:00 nginx: worker process
# 101    45713  45678  0  10:00  ?    00:00:00 nginx: worker process
# 101    45714  45678  0  10:00  ?    00:00:00 nginx: worker process
```

**Columnas:**

| Columna | Significado | Nota |
|---|---|---|
| UID | ID de usuario del proceso | Mapeado al UID dentro del contenedor |
| PID | ID de proceso DENTRO del contenedor | El PID 1 es el proceso principal |
| PPID | Parent PID | Quién creó este proceso |
| C | Uso de CPU (%) | Según el scheduler del kernel |
| STIME | Hora de inicio del proceso | Dentro del contenedor |
| TTY | Terminal asociada | `?` = sin terminal |
| TIME | Tiempo acumulado de CPU | En minutos:segundos |
| CMD | Comando y argumentos | Comando completo |

```bash
# Comparar con procesos en el HOST
ps aux | grep nginx
# 101    45678  0.0  0.0  ... nginx: master process  ← Mismo PID desde el host
# 101    45712  0.0  0.0  ... nginx: worker process
```

Los PIDs que ves en `docker top` son los mismos PIDs del host. Docker NO crea PID namespaces anidados por defecto; los procesos del contenedor son visibles desde el host como cualquier otro proceso.

### 4.8 `docker stats` — Monitoreo en tiempo real

```bash
# Stream en vivo de todos los contenedores
docker stats

# Sin streaming (un solo snapshot)
docker stats --no-stream

# Solo contenedores específicos
docker stats nginx-lab mysql-lab

# Formato personalizado
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"
# NAME         CPU %    MEM USAGE / LIMIT    NET I/O          BLOCK I/O
# nginx-lab    0.01%    4.2MiB / 128MiB      1.5kB / 1.2kB   8.2kB / 0B
# mysql-lab    15.3%    389MiB / 512MiB      45kB / 23kB      12MB / 0B

# Solo nombres y uso de memoria
docker stats --no-stream --format '{{.Name}}: {{.MemPerc}}'
# nginx-lab: 3.28%
# mysql-lab: 75.98%

# Filtrar y ordenar con herramientas estándar
docker stats --no-stream --format '{{.Name}}\t{{.MemPerc}}' | sort -k2 -rn
```

**Columnas de `docker stats`:**

| Columna | Significado |
|---|---|
| CONTAINER ID / Name | Identificador o nombre |
| CPU % | Porcentaje de CPU del host usado por este contenedor |
| MEM USAGE / LIMIT | Memoria actual usada / límite impuesto (`--memory`) |
| MEM % | Porcentaje del límite usado |
| NET I/O | Tráfico de red: entrada / salida (acumulado desde inicio) |
| BLOCK I/O | Lectura / escritura a disco (acumulado desde inicio) |
| PIDS | Número de procesos dentro del contenedor |

### 4.9 `docker port` — Ver mapeo de puertos

```bash
docker port nginx-lab
# 80/tcp -> 0.0.0.0:8080
# 80/tcp -> [::]:8080

# Solo el puerto del host
docker port nginx-lab 80/tcp
# 0.0.0.0:8080

# Útil para scripting
PORT=$(docker port nginx-lab 80/tcp | cut -d: -f2)
curl localhost:$PORT
```

---

## 5. Gestión de Contenedores

### 5.1 Comandos de control de vida

```bash
# Parar (SIGTERM + grace period)
docker stop nginx-lab
docker stop --time 30 nginx-lab          # Grace period de 30s

# Arrancar un contenedor parado
docker start nginx-lab

# Reiniciar (stop + start)
docker restart nginx-lab
docker restart --time 30 nginx-lab       # Con grace period

# Pausar / reanudar
docker pause nginx-lab
docker unpause nginx-lab

# Matar (SIGKILL o señal custom)
docker kill nginx-lab
docker kill --signal SIGHUP nginx-lab     # Enviar SIGHUP (recargar config)
```

### 5.2 Eliminar contenedores

```bash
# Eliminar contenedor parado
docker stop nginx-lab
docker rm nginx-lab

# Forzar eliminación (para + borra contenedor en ejecución)
docker rm -f nginx-lab

# Eliminar TODOS los contenedores parados
docker container prune
# WARNING! This will remove all stopped containers.
# Are you sure you want to continue? [y/N]

# Sin confirmación (para scripts)
docker container prune -f

# Eliminar contenedor y su volumen anónimo asociado
docker rm -v nginx-lab

# Eliminar todos los contenedores (incluso los que corren)
docker rm -f $(docker ps -aq)
# ⚠️ PELIGRO: borra TODO

# Eliminar condicionalmente
docker ps -a --filter "status=exited" --filter "exited=137" -q | xargs docker rm
# Borra todos los contenedores matados por OOM/SIGKILL

# Eliminar contenedores por label
docker rm $(docker ps -aq --filter "label=environment=lab")
```

### 5.3 Renombrar contenedores

```bash
docker rename viejo-nombre nuevo-nombre

# Útil cuando quieres "promover" un despliegue
docker stop app-blue
docker start app-green
docker rename app-blue app-old
docker rename app-green app-blue
```

### 5.4 Esperar a que un contenedor termine

```bash
# Bloquea hasta que el contenedor termine y muestra el exit code
docker run --rm -d --name worker ubuntu bash -c 'sleep 5; exit 42'
docker wait worker
# 42  (exit code del contenedor)

# Útil en scripts:
docker run -d --name backup-job backup-image
docker wait backup-job
EXIT_CODE=$?
echo "Backup terminó con código: $EXIT_CODE"
if [ $EXIT_CODE -eq 0 ]; then
    echo "Backup exitoso"
else
    echo "Backup falló!" >&2
fi
```

### 5.5 Políticas de reinicio en detalle

```
                ┌──────────────┐
                │  Contenedor  │
                │  termina     │
                └──────┬───────┘
                       │
           ¿Qué política de reinicio tiene?
           ┌───────────┼───────────┐
           │           │           │
      no (default)  on-failure  always / unless-stopped
           │           │           │
           ▼           ▼           ▼
      NUNCA        ¿Exit code    SIEMPRE
      reinicia      != 0?        reinicia
                       │
                  ┌────┴────┐
                  │  NO      │  SI
                  ▼          ▼
               No          Reinicio tras
               reinicia    1s, luego delay
                           exponencial

Delay exponencial:
  Intento 1: 100ms
  Intento 2: 200ms
  Intento 3: 400ms
  Intento 4: 800ms
  Intento 5: 1.6s
  ...
  Máximo delay: 1 minuto entre intentos
```

**Ejemplo práctico: todas las políticas**

```bash
# Política no (por defecto)
docker run -d --restart no --name test-no alpine sleep 3
# Espera 3 segundos...
docker ps -a --filter name=test-no
# STATUS: Exited (0) 5 seconds ago  ← NO se reinicia

# Política on-failure (solo errores)
docker run -d --restart on-failure --name test-onfail alpine sh -c 'echo "Voy a fallar" && sleep 2 && exit 1'
# Espera unos segundos...
docker ps --filter name=test-onfail
# STATUS: Up 10 seconds  ← Se reinició porque exit code != 0

# Ver contador de reinicios
docker inspect --format '{{.RestartCount}}' test-onfail
# 2  (se ha reiniciado 2 veces)

# on-failure con máximo de reintentos
docker run -d --restart on-failure:3 --name test-max alpine sh -c 'exit 1'
# Después de 3 intentos fallidos, Docker desiste
docker ps --filter name=test-max
# STATUS: Exited (1) X seconds ago  ← Ya no se reinicia más

# always: incluso si se para manualmente
docker run -d --restart always --name test-always alpine sleep 3600
docker stop test-always
# El contenedor se para, pero...
docker ps -a --filter name=test-always
# STATUS: Exited (137) ...  ← NO se reinicia en este momento
# PERO si el daemon Docker se reinicia (sudo systemctl restart docker),
# test-always VOLVERÁ a arrancar automáticamente

# unless-stopped: respeta la parada manual
docker run -d --restart unless-stopped --name test-unless alpine sleep 3600
docker stop test-unless
# El contenedor se para y NO se reinicia al reiniciar el daemon
# Pero si crashea por error, SÍ se reinicia antes de que lo pares manualmente
```

**Recomendaciones:**

| Entorno | Política recomendada | Razón |
|---|---|---|
| Desarrollo local | `no` (default) | Quieres control total sobre cuándo corren los contenedores |
| CI/CD | `no` o `--rm` | Contenedores efímeros, no deben reiniciarse |
| Producción (servicios) | `unless-stopped` | Balance entre resiliencia y control manual |
| Producción (crítico) | `always` | Debe estar corriendo SIEMPRE, incluso tras reinicio del host |
| Jobs batch | `on-failure:3` | Reintentar pero no indefinidamente |

---

## 6. Imágenes Básicas

### 6.1 `docker pull` — Descargar imágenes

```bash
# Imagen sin tag específico (usa :latest)
docker pull nginx
# latest: Pulling from library/nginx
# c1ec31eb5944: Pull complete
# ...
# Digest: sha256:...
# Status: Downloaded newer image for nginx:latest

# Tag específico (recomendado en producción)
docker pull nginx:1.27-alpine
docker pull nginx:1.27.0-alpine3.20   # Versión exacta

# Por digest (inmutable, más seguro)
docker pull nginx@sha256:53641cd209a4fecfc68e21a99871ce8c6920b2e7502df0a20671c6fccc73a7c6

# Otras imágenes comunes
docker pull mysql:8.4
docker pull mysql:8.4-oracle           # Basado en Oracle Linux
docker pull mysql:8.4-debian           # Basado en Debian
docker pull redis:7.4-alpine
docker pull alpine:3.20
docker pull ubuntu:24.04
docker pull python:3.12-slim           # Versión reducida de Python
docker pull node:22-alpine
docker pull golang:1.23-alpine
```

**¿Qué pasa durante un pull?**

```
docker pull nginx:1.27-alpine
                        │
                        ▼
┌────────────────────────────────────────────────┐
│  1. Resolver nombre de la imagen               │
│     library/nginx → Docker Hub oficial         │
│     tag: 1.27-alpine                           │
│     plataforma: linux/amd64 (auto-detectada)   │
└──────────────────────┬─────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────┐
│  2. Obtener manifiesto                         │
│     GET /v2/library/nginx/manifests/1.27-alpine│
│     → lista de capas (sha256 de cada layer)    │
│     → config (entrypoint, env, etc.)           │
└──────────────────────┬─────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────┐
│  3. Descargar cada capa (en paralelo)          │
│     Para cada layer sha256:                    │
│       - ¿Existe localmente? → skip (cached)    │
│       - ¿No existe? → descargar, verificar,    │
│         descomprimir en overlay2 storage       │
│     ┌────────┐ ┌────────┐ ┌────────┐           │
│     │ Layer 0 │ │ Layer 1 │ │ Layer 2 │  ...    │
│     │ (base)  │ │        │ │        │           │
│     └────┬────┘ └────┬────┘ └────┬────┘           │
│          │           │           │               │
│          ▼           ▼           ▼               │
│        /var/lib/docker/overlay2/<hash>/diff/    │
└──────────────────────┬─────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────┐
│  4. Registrar en la base de datos de imágenes   │
│     /var/lib/docker/image/overlay2/repositories.json   │
└────────────────────────────────────────────────┘
```

---

### 6.2 `docker images` — Listar imágenes locales

```bash
docker images
# REPOSITORY   TAG            IMAGE ID       CREATED        SIZE
# nginx        alpine         39286d2f6c4e   2 days ago     45.1MB
# mysql        8.4            5a57d4e1e5f2   3 days ago     582MB
# redis        7.4-alpine     8e8f2b3c4d1e   5 days ago     35.4MB
# ubuntu       24.04          bf3dc08bfed0   7 days ago     77.9MB
# alpine       latest         25fad4a56508   2 weeks ago    7.05MB
# hello-world  latest         d2c94e258dcb   3 months ago   13.3kB
```

#### Entendiendo el output

| Columna | Significado | Notas |
|---|---|---|
| **REPOSITORY** | Nombre de la imagen | `library/<nombre>` en Docker Hub. O `usuario/imagen`. |
| **TAG** | Etiqueta de versión | Si es `<none>`, es una imagen "dangling" (sin tag). |
| **IMAGE ID** | Hash SHA256 único (12 chars) | Dos tags pueden apuntar al mismo ID. |
| **CREATED** | Cuándo se creó esta capa final | No necesariamente cuándo la descargaste. |
| **SIZE** | Tamaño total descomprimido | No incluye capas compartidas con otras imágenes. |

**Filtros y formatos:**

```bash
# Solo cierta imagen
docker images nginx

# Solo por tag
docker images nginx:alpine

# Mostrar digests (inmutables)
docker images --digest
# REPOSITORY  TAG     DIGEST                                                                   IMAGE ID
# nginx       alpine  sha256:53641cd209a4fecfc68e21a99871ce8c6920b2e7502df0a20671c6fccc73a7c6  39286d2f6c4e

# Mostrar imágenes intermedias (capas sin tag, del build)
docker images -a

# Imágenes "dangling" (sin tag, <none>:<none>)
docker images --filter "dangling=true"

# Filtrar por label
docker images --filter "label=maintainer=nginx"

# Filtrar por referencia
docker images --filter "reference=nginx:*"      # Todas las versiones de nginx
docker images --filter "reference=*:alpine"     # Todo lo que sea alpine

# Formato personalizado
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
docker images --format '{{.ID}}: {{.Repository}}:{{.Tag}} ({{.Size}})'

# Solo IDs
docker images -q

# Tamaño total en disco (incluye capas compartidas)
docker system df
# TYPE           TOTAL   ACTIVE  SIZE    RECLAIMABLE
# Images         12      5       2.15GB  1.02GB (47%)
# Containers     8       3       125MB   0B (0%)
# Local Volumes  5       2       1.5GB   800MB (53%)
# Build Cache    20      0       500MB   500MB (100%)
```

---

### 6.3 `docker rmi` — Eliminar imágenes

```bash
# Eliminar por nombre:tag
docker rmi nginx:alpine

# Eliminar por ID
docker rmi 39286d2f6c4e

# Eliminar MÚLTIPLES (separados por espacio)
docker rmi nginx:alpine redis:7.4-alpine mysql:8.4

# Forzar (incluso si contenedores la usan)
docker rmi -f nginx:alpine

# Eliminar TODAS las imágenes sin usar
docker image prune -a
# ⚠️ Borra todas las imágenes que no están referenciadas por ningún contenedor

# ¿Qué imágenes tengo que ningún contenedor usa?
comm -23 \
  <(docker images -q | sort) \
  <(docker ps -a --format '{{.Image}}' | xargs -n1 docker images -q --no-trunc | sort -u)
```

**Error común:**

```
Error response from daemon: conflict: unable to remove repository reference
"nginx:alpine" (must force) - container 7f8a3b2c1d4e is using its referenced
image 39286d2f6c4e
```

→ Hay un contenedor (incluso parado) que usa esa imagen. Bórralo primero con `docker rm` o usa `-f`.

### 6.4 `docker image prune` — Limpiar imágenes sin uso

```bash
# Ver qué se borraría (dry run)
docker image prune -a --dry-run

# Borrar imágenes sin contenedor asociado
docker image prune

# Borrar TODAS las imágenes sin contenedor (incluso con tag)
docker image prune -a

# Sin confirmación
docker image prune -a -f

# Borrar imágenes creadas hace más de 24h
docker image prune -a --filter "until=24h"

# Limpieza completa del sistema
docker system prune -af --volumes
# ⚠️ Elimina: contenedores parados, imágenes sin uso, redes sin uso,
#            build cache, volúmenes sin uso
```

---

## 7. Laboratorio Práctico Integrador

Vamos a desplegar un stack completo de 4 servicios interconectados que simulan una aplicación real:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  STACK DE LABORATORIO                               │
│                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐         │
│  │   Nginx      │     │   Adminer    │     │   Redis      │         │
│  │   (web)      │     │   (DB GUI)   │     │   (cache)    │         │
│  │   :80        │     │   :8080      │     │   :6379      │         │
│  └──────┬───────┘     └──────┬───────┘     └──────────────┘         │
│         │                    │                                      │
│         │    ┌───────────────┘                                      │
│         │    │                                                      │
│         ▼    ▼                                                      │
│  ┌──────────────────────────────────────┐                           │
│  │           Red: lab-net               │                           │
│  │      (bridge user-defined)           │                           │
│  │  ┌────────────────────────────────┐  │                           │
│  │  │  DNS automático:               │  │                           │
│  │  │  mysql-lab → 172.18.0.2        │  │                           │
│  │  │  redis-lab → 172.18.0.3        │  │                           │
│  │  │  nginx-lab → 172.18.0.4        │  │                           │
│  │  │  adminer-lab → 172.18.0.5      │  │                           │
│  │  └────────────────────────────────┘  │                           │
│  └──────────────┬───────────────────────┘                           │
│                 │                                                   │
│                 ▼                                                   │
│  ┌──────────────────────────────────────┐                           │
│  │   MySQL 8.4                          │                           │
│  │   (base de datos)                    │                           │
│  │   :3306                              │                           │
│  │   ┌────────────────────────────┐     │                           │
│  │   │ Volumen: mysql-data-lab    │     │                           │
│  │   │ /var/lib/mysql (persistente│     │                           │
│  │   │ incluso si borras el       │     │                           │
│  │   │ contenedor)                │     │                           │
│  │   └────────────────────────────┘     │                           │
│  └──────────────────────────────────────┘                           │
│                                                                     │
│  Externamente expuestos:                                           │
│    localhost:8080 → Nginx (página web)                             │
│    localhost:8081 → Adminer (gestor de BD)                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Paso 1: Crear la red definida por el usuario

```bash
docker network create lab-net
# 7f8a3b2c1d4e5f6a7b8c9d0e1f2a3b4c

# Verificar
docker network inspect lab-net
```

**Por qué red definida por el usuario y no la bridge default:**
- DNS automático por nombre de contenedor (en la bridge default NO funciona).
- Aislamiento: los contenedores en otras redes no pueden alcanzar a los de `lab-net` a menos que se configure explícitamente.
- Mejor control: puedes especificar subred, gateway, IPAM.

### Paso 2: Crear volumen para MySQL

```bash
docker volume create mysql-data-lab

# Verificar
docker volume ls
# DRIVER    VOLUME NAME
# local     mysql-data-lab

docker volume inspect mysql-data-lab
# [
#   {
#     "CreatedAt": "2024-07-30T12:00:00Z",
#     "Driver": "local",
#     "Labels": {},
#     "Mountpoint": "/var/lib/docker/volumes/mysql-data-lab/_data",
#     "Name": "mysql-data-lab",
#     "Options": {},
#     "Scope": "local"
#   }
# ]
```

**Por qué volumen y no bind mount para la BD:**
- Los volúmenes gestionados por Docker tienen mejor rendimiento en macOS/Windows.
- Son portables entre sistemas (Docker gestiona la ubicación).
- Más fáciles de hacer backup (`docker run --rm -v mysql-data-lab:/data -v $(pwd):/backup alpine tar czf /backup/mysql-backup.tar.gz -C /data .`).

### Paso 3: Desplegar MySQL

```bash
docker run -d \
  --name mysql-lab \
  --network lab-net \
  --restart unless-stopped \
  --cpus 1.0 \
  --memory 512m \
  -v mysql-data-lab:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=rootpass123 \
  -e MYSQL_DATABASE=miapp \
  -e MYSQL_USER=miapp_user \
  -e MYSQL_PASSWORD=miapp_pass \
  -p 3306:3306 \
  --health-cmd='mysqladmin ping -h localhost --silent' \
  --health-interval=10s \
  --health-timeout=5s \
  --health-retries=3 \
  --health-start-period=30s \
  mysql:8.4
```

**Qué hace cada opción:**

| Opción | Explicación |
|---|---|
| `-d` | Detached: MySQL corre en background |
| `--name mysql-lab` | Nombre para referencia DNS (`mysql-lab` → resuelve a su IP) |
| `--network lab-net` | Dentro de nuestra red aislada |
| `--restart unless-stopped` | Si crashea, se reinicia. Si el host bootea, se reinicia. Si lo paras manualmente, no. |
| `--cpus 1.0` | Máximo 1 CPU |
| `--memory 512m` | Máximo 512 MB de RAM |
| `-v mysql-data-lab:/var/lib/mysql` | Persistencia de datos fuera del contenedor |
| `-e MYSQL_ROOT_PASSWORD=...` | Contraseña root (MySQL la exige) |
| `-e MYSQL_DATABASE=miapp` | BD creada automáticamente al iniciar |
| `-e MYSQL_USER=...` | Usuario con permisos solo sobre `miapp` |
| `-e MYSQL_PASSWORD=...` | Contraseña del usuario `miapp_user` |
| `-p 3306:3306` | Expone MySQL (opcional, solo para debug local) |
| `--health-cmd=...` | Healthcheck: comprueba que MySQL acepta conexiones |
| `--health-interval=10s` | Revisar cada 10 segundos |
| `--health-timeout=5s` | Timeout del healthcheck |
| `--health-retries=3` | 3 fallos consecutivos → unhealthy |
| `--health-start-period=30s` | Primeros 30s no cuentan (MySQL tarda en arrancar) |

**Verificar que MySQL está healthy:**

```bash
docker ps --filter name=mysql-lab
# STATUS: Up 30 seconds (healthy)  ← "healthy" indica que el healthcheck pasa

# Probar conexión
docker exec -it mysql-lab mysql -umiapp_user -pmiapp_pass miapp -e "SELECT 1 AS conectado;"
# +------------+
# | conectado  |
# +------------+
# |          1 |
# +------------+

# Crear una tabla y datos de prueba
docker exec mysql-lab mysql -umiapp_user -pmiapp_pass miapp << 'SQL'
CREATE TABLE IF NOT EXISTS usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO usuarios (nombre, email) VALUES
    ('Alice', 'alice@ejemplo.com'),
    ('Bob', 'bob@ejemplo.com'),
    ('Charlie', 'charlie@ejemplo.com');
SQL

# Verificar datos
docker exec mysql-lab mysql -umiapp_user -pmiapp_pass miapp -e "SELECT * FROM usuarios;"
```

### Paso 4: Desplegar Redis (caché)

```bash
docker run -d \
  --name redis-lab \
  --network lab-net \
  --restart unless-stopped \
  --cpus 0.5 \
  --memory 64m \
  --health-cmd='redis-cli ping | grep PONG' \
  --health-interval=10s \
  --health-timeout=3s \
  --health-retries=3 \
  redis:7.4-alpine redis-server --save 60 1 --loglevel warning
```

**Explicación de los argumentos de Redis:**
- `redis-server --save 60 1`: persistir a disco si al menos 1 clave cambió en 60 segundos.
- `--loglevel warning`: solo mostrar advertencias y errores (menos ruido en logs).

**Verificar Redis:**

```bash
docker ps --filter name=redis-lab
# STATUS: Up X seconds (healthy)

# Probar conexión y operaciones
docker exec redis-lab redis-cli PING
# PONG

docker exec -it redis-lab redis-cli << 'REDIS'
SET visitas 100
SET usuario:1:nombre "Alice"
SET usuario:1:plan "premium"
GET usuario:1:nombre
INCR visitas
GET visitas
EXPIRE usuario:1:nombre 3600
TTL usuario:1:nombre
REDIS
# OK
# OK
# OK
# "Alice"
# (integer) 101
# "101"
# (integer) 1
# (integer) 3599
```

### Paso 5: Desplegar Adminer (gestor web de base de datos)

Adminer es un gestor de bases de datos en un solo archivo PHP, mucho más ligero que phpMyAdmin.

```bash
docker run -d \
  --name adminer-lab \
  --network lab-net \
  --restart unless-stopped \
  -p 8081:8080 \
  -e ADMINER_DEFAULT_SERVER=mysql-lab \
  -e ADMINER_DESIGN=nette \
  adminer:latest
```

**Explicación:**
- `-p 8081:8080`: Adminer sirve en el puerto 8080 del contenedor. Lo mapeamos al 8081 del host (el 8080 lo ocupa Nginx luego).
- `ADMINER_DEFAULT_SERVER=mysql-lab`: Adminer se conecta automáticamente a MySQL por nombre (gracias al DNS de `lab-net`).
- `ADMINER_DESIGN=nette`: tema visual alternativo.

**Accede a Adminer:**
- Abre `http://localhost:8081` en tu navegador.
- Login: Servidor `mysql-lab`, Usuario `miapp_user`, Contraseña `miapp_pass`, Base de datos `miapp`.
- Verás la tabla `usuarios` con los 3 registros insertados.

### Paso 6: Configurar y desplegar Nginx

Nginx servirá como proxy inverso y servidor de contenido estático.

```bash
# Crear directorios
mkdir -p ~/lab-stack/{html,conf.d,logs}

# Página web estática con dashboard de estado embebido
cat > ~/lab-stack/html/index.html << 'HTML'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Lab Stack - Docker</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
               background: #0f172a; color: #e2e8f0; line-height: 1.6; }
        .container { max-width: 900px; margin: 0 auto; padding: 2rem; }
        h1 { color: #38bdf8; font-size: 2.5rem; margin-bottom: 0.5rem; }
        .subtitle { color: #94a3b8; margin-bottom: 2rem; font-size: 1.1rem; }
        .status-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 1rem;
            margin-bottom: 2rem;
        }
        .status-card {
            background: #1e293b;
            border: 1px solid #334155;
            border-radius: 12px;
            padding: 1.5rem;
            text-align: center;
            transition: transform 0.2s, border-color 0.2s;
        }
        .status-card:hover { transform: translateY(-2px); border-color: #38bdf8; }
        .status-card .name { font-weight: 600; color: #f1f5f9; font-size: 1.1rem; }
        .status-card .version { color: #64748b; font-size: 0.8rem; }
        .status-card .info { color: #94a3b8; font-size: 0.85rem; margin-top: 0.3rem; }
        .status-card.online { border-left: 4px solid #22c55e; }
        .status-card.offline { border-left: 4px solid #ef4444; opacity: 0.5; }
        .endpoints { background: #1e293b; border-radius: 12px; padding: 1.5rem; margin-bottom: 2rem; }
        .endpoints h2 { color: #38bdf8; margin-bottom: 1rem; }
        .endpoint-row {
            display: flex; justify-content: space-between; align-items: center;
            padding: 0.5rem 0; border-bottom: 1px solid #334155;
        }
        .endpoint-row:last-child { border-bottom: none; }
        .method { display: inline-block; padding: 2px 8px; border-radius: 4px;
                  font-size: 0.75rem; font-weight: 700; margin-right: 0.5rem; }
        .method.get { background: #22c55e33; color: #22c55e; }
        .method.post { background: #3b82f633; color: #3b82f6; }
        .endpoint-path { font-family: 'Fira Code', monospace; color: #e2e8f0; }
        .endpoint-desc { color: #94a3b8; font-size: 0.85rem; }
        footer { text-align: center; color: #475569; padding: 2rem 0;
                 border-top: 1px solid #1e293b; margin-top: 3rem; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Docker Lab Stack</h1>
        <p class="subtitle">Stack de laboratorio: Nginx + MySQL + Adminer + Redis</p>
        <div class="status-grid">
            <div class="status-card online">
                <div class="name">Nginx</div>
                <div class="version">nginx:alpine</div>
                <div class="info">Servidor web y proxy inverso</div>
            </div>
            <div class="status-card online">
                <div class="name">MySQL</div>
                <div class="version">mysql:8.4</div>
                <div class="info">Base de datos relacional</div>
            </div>
            <div class="status-card online">
                <div class="name">Redis</div>
                <div class="version">redis:7.4-alpine</div>
                <div class="info">Caché en memoria</div>
            </div>
            <div class="status-card online">
                <div class="name">Adminer</div>
                <div class="version">adminer:latest</div>
                <div class="info">Gestor de base de datos</div>
            </div>
        </div>
        <div class="endpoints">
            <h2>Endpoints disponibles</h2>
            <div class="endpoint-row">
                <span><span class="method get">GET</span><span class="endpoint-path">/</span></span>
                <span class="endpoint-desc">Esta página</span>
            </div>
            <div class="endpoint-row">
                <span><span class="method get">GET</span><span class="endpoint-path">/health</span></span>
                <span class="endpoint-desc">Health check de Nginx</span>
            </div>
            <div class="endpoint-row">
                <span><span class="method get">GET</span><span class="endpoint-path">/api/data</span></span>
                <span class="endpoint-desc">Datos del sistema (hostname, nginx versión)</span>
            </div>
            <div class="endpoint-row">
                <span><span class="method get">GET</span><span class="endpoint-path">/adminer</span></span>
                <span class="endpoint-desc">Adminer (gestor de BD proxyado)</span>
            </div>
        </div>
        <footer>
            Libro Docker: De Novato a Experto — Capítulo 2: Laboratorio Integrador
        </footer>
    </div>
</body>
</html>
HTML

# Configuración de Nginx con proxy a Adminer y logging JSON
cat > ~/lab-stack/conf.d/default.conf << 'NGINX'
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /health {
        access_log off;
        return 200 '{"status":"healthy","service":"nginx","timestamp":"$time_iso8601"}\n';
        add_header Content-Type application/json;
    }

    location /api/data {
        access_log off;
        return 200 '{"hostname":"$hostname","nginx_version":"nginx/1.27","connections":$connections_active}\n';
        add_header Content-Type application/json;
    }

    location /adminer {
        proxy_pass http://adminer-lab:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

log_format json_log escape=json '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"method":"$request_method",'
    '"uri":"$request_uri",'
    '"status":$status,'
    '"body_bytes":$body_bytes_sent,'
    '"request_time":$request_time'
'}';

access_log /var/log/nginx/access.log json_log;
NGINX

# Desplegar Nginx
docker run -d \
  --name nginx-lab \
  --network lab-net \
  --restart unless-stopped \
  --cpus 0.5 \
  --memory 128m \
  --user 101:101 \
  -p 8080:80 \
  -v ~/lab-stack/html:/usr/share/nginx/html:ro \
  -v ~/lab-stack/conf.d:/etc/nginx/conf.d:ro \
  -v ~/lab-stack/logs:/var/log/nginx \
  --health-cmd='wget --no-verbose --tries=1 --spider http://localhost:80/health || exit 1' \
  --health-interval=15s \
  --health-timeout=3s \
  --health-retries=3 \
  nginx:alpine
```

**Por qué montar `/etc/nginx/conf.d:ro` como solo lectura:**
- La configuración de Nginx nunca debería ser modificada por el contenedor.
- Si un atacante compromete Nginx, no puede modificar la configuración del servidor.
- En desarrollo, editas los archivos en tu host y recargas con `docker exec nginx-lab nginx -s reload`.

**Proxy a Adminer explicado:**
El usuario pide → `http://localhost:8080/adminer` → Nginx lo reenvía a `adminer-lab:8080` (resuelto por DNS interno de `lab-net`) → Adminer responde → Nginx lo devuelve al navegador.

### Paso 7: Verificar todo el stack

```bash
# Todos los contenedores deben estar Up y healthy
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" --filter network=lab-net
# NAMES         STATUS                       PORTS
# nginx-lab     Up 5 minutes (healthy)       0.0.0.0:8080->80/tcp
# adminer-lab   Up 5 minutes                 0.0.0.0:8081->8080/tcp
# redis-lab     Up 5 minutes (healthy)       6379/tcp
# mysql-lab     Up 5 minutes (healthy)       0.0.0.0:3306->3306/tcp

# Verificar DNS interno (desde nginx)
docker exec nginx-lab ping -c1 mysql-lab
# PING mysql-lab (172.18.0.2): 56 data bytes
# 64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.120 ms

docker exec nginx-lab ping -c1 redis-lab
# PING redis-lab (172.18.0.3): 56 data bytes
# 64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.095 ms

# Verificar conexión MySQL desde adminer
docker exec adminer-lab php -r '
try {
    $pdo = new PDO("mysql:host=mysql-lab;dbname=miapp", "miapp_user", "miapp_pass");
    $stmt = $pdo->query("SELECT COUNT(*) FROM usuarios");
    echo "Usuarios en BD: " . $stmt->fetchColumn() . "\n";
} catch (PDOException $e) {
    echo "Error: " . $e->getMessage() . "\n";
}'
# Usuarios en BD: 3

# Verificar Redis
docker exec redis-lab redis-cli GET visitas
# "101"

# Stats completos del stack
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}" \
  nginx-lab mysql-lab redis-lab adminer-lab
# NAME         CPU %    MEM USAGE / LIMIT   MEM %
# nginx-lab    0.00%    4.1MiB / 128MiB     3.20%
# mysql-lab    0.85%    395MiB / 512MiB      77.15%
# redis-lab    0.12%    3.8MiB / 64MiB       5.94%
# adminer-lab  0.01%    18.2MiB / 1.95GiB    0.91%
```

### Paso 8: Pruebas funcionales

```bash
# Test 1: Nginx sirviendo contenido estático
curl -s localhost:8080 | grep -o '<h1>.*</h1>'
# <h1>Docker Lab Stack</h1>

# Test 2: Health check
curl -s localhost:8080/health | jq .
# {
#   "status": "healthy",
#   "service": "nginx",
#   "timestamp": "2024-07-30T12:30:00+00:00"
# }

# Test 3: API de datos
curl -s localhost:8080/api/data | jq .
# {
#   "hostname": "nginx-lab-server",
#   "nginx_version": "nginx/1.27",
#   "connections": 1
# }

# Test 4: MySQL datos persistentes (reiniciar para comprobar)
docker restart mysql-lab
sleep 10  # Esperar a que MySQL esté listo
docker exec mysql-lab mysql -umiapp_user -pmiapp_pass miapp -e "SELECT * FROM usuarios;"
# Los 3 usuarios siguen ahí → el volumen funciona

# Test 5: Redis persistencia
docker restart redis-lab
sleep 3
docker exec redis-lab redis-cli GET visitas
# "101"  → el dato sobrevivió al reinicio (gracias a --save 60 1)

# Test 6: Crear un nuevo usuario via MySQL
docker exec mysql-lab mysql -umiapp_user -pmiapp_pass miapp \
  -e "INSERT INTO usuarios (nombre, email) VALUES ('Diana', 'diana@ejemplo.com');"
docker exec mysql-lab mysql -umiapp_user -pmiapp_pass miapp \
  -e "SELECT * FROM usuarios ORDER BY id;"
# +----+---------+---------------------+---------------------+
# | id | nombre  | email               | creado_en           |
# +----+---------+---------------------+---------------------+
# |  1 | Alice   | alice@ejemplo.com   | 2024-07-30 12:00:00 |
# |  2 | Bob     | bob@ejemplo.com     | 2024-07-30 12:00:00 |
# |  3 | Charlie | charlie@ejemplo.com | 2024-07-30 12:00:00 |
# |  4 | Diana   | diana@ejemplo.com   | 2024-07-30 12:30:00 |
# +----+---------+---------------------+---------------------+

# Test 7: Cache en Redis
docker exec redis-lab redis-cli INCR visitas
docker exec redis-lab redis-cli GET visitas
# "102"
```

### Paso 9: Backup y limpieza

```bash
# Backup de MySQL (dump SQL comprimido)
BACKUP_FILE="mysql-backup-$(date +%Y%m%d-%H%M%S).sql.gz"
docker exec mysql-lab mysqldump -umiapp_user -pmiapp_pass miapp | gzip > $BACKUP_FILE
echo "Backup creado: $BACKUP_FILE ($(wc -c < $BACKUP_FILE) bytes)"
# Backup creado: mysql-backup-20240730-123000.sql.gz (1234 bytes)

# Backup de Redis (BGSAVE guarda en dump.rdb dentro del contenedor)
docker exec redis-lab redis-cli BGSAVE
# Background saving started

# Backup del volumen de MySQL comprimido (todo el directorio de datos)
docker run --rm \
  -v mysql-data-lab:/data:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/mysql-volume-backup.tar.gz -C /data .

# Listar los backups creados
ls -lh *backup*
# -rw-r--r-- 1 user user 1.2K Jul 30 12:30 mysql-backup-20240730-123000.sql.gz
# -rw-r--r-- 1 user user  45M Jul 30 12:30 mysql-volume-backup.tar.gz

# Restaurar backup de volumen
# docker run --rm -v mysql-data-lab:/data -v $(pwd):/backup alpine \
#   tar xzf /backup/mysql-volume-backup.tar.gz -C /data

# Para detener todo el stack SIN borrar datos:
docker stop nginx-lab mysql-lab redis-lab adminer-lab

# Para re-arrancar todo:
docker start mysql-lab redis-lab adminer-lab nginx-lab

# Para limpiar todo (incluyendo volumen con datos):
docker stop nginx-lab mysql-lab redis-lab adminer-lab
docker rm nginx-lab mysql-lab redis-lab adminer-lab
docker volume rm mysql-data-lab
docker network rm lab-net
rm -rf ~/lab-stack
```

---

## Resumen del Capítulo

```
┌─────────────────────────────────────────────────────────────────┐
│                      CAPÍTULO 2 COMPLETADO                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ Instalaste Docker en Linux (apt/dnf), macOS (Desktop/Colima)│
│     y Windows (WSL2).                                           │
│                                                                 │
│  ✅ Verificaste la instalación con docker version, docker info  │
│     y docker run hello-world. Entendiste CADA línea de salida.   │
│                                                                 │
│  ✅ Dominaste docker run y sus 15+ flags esenciales con ejemplos│
│     reales: -d, -it, --name, -p, -v, -e, --rm, --restart,      │
│     --network, --cpus, --memory, --user, --workdir.             │
│                                                                 │
│  ✅ Comprendiste el ciclo de vida: CREATED → RUNNING → PAUSED  │
│     → STOPPED → DELETED y los comandos para cada transición.    │
│                                                                 │
│  ✅ Diferenciaste docker stop (SIGTERM + grace) vs docker kill  │
│     (SIGKILL inmediato) y supiste cuándo usar cada uno.          │
│                                                                 │
│  ✅ Ejecutaste comandos del día a día: ps, logs, exec, inspect, │
│     cp, diff, top, stats, port.                                 │
│                                                                 │
│  ✅ Gestionaste contenedores: stop, start, restart, rm, prune,  │
│     rename, wait, políticas de reinicio.                        │
│                                                                 │
│  ✅ Trabajaste con imágenes: pull, images, rmi, prune.          │
│                                                                 │
│  ✅ Desplegaste un stack completo: Nginx + MySQL + Adminer +    │
│     Redis en red personalizada con volúmenes, healthchecks      │
│     y backup automatizado.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Lo que aprendiste y cómo se conecta con los próximos capítulos

| Concepto del Capítulo 2 | Se profundiza en... |
|---|---|
| `docker run -v` (volúmenes) | Capítulo 5: Almacenamiento y Volúmenes |
| `docker run -p` (puertos) | Capítulo 6: Redes en Docker |
| `--network mi-red` | Capítulo 6: DNS, drivers de red |
| `docker pull` (imágenes) | Capítulo 3: Imágenes y Registries |
| `docker run` con muchas flags | Capítulo 7: Docker Compose (un solo YAML) |
| `--restart`, `--cpus`, `--memory` | Capítulo 8: Orquestación (Swarm/K8s) |
| `--user`, `-v :ro` | Capítulo 10: Seguridad en Docker |
| Healthchecks y logs | Capítulo 11: Monitoreo y Logging |
| Volúmenes persistentes y backups | Capítulo 12: Docker en Producción |

### Checklist de autoevaluación

Antes de pasar al Capítulo 3, asegúrate de poder responder SÍ a todas estas preguntas:

- [ ] ¿Puedo instalar Docker en Ubuntu sin usar el script de conveniencia?
- [ ] ¿Sé interpretar CADA línea del output de `docker info`?
- [ ] ¿Puedo explicar la diferencia entre `docker stop` y `docker kill`?
- [ ] ¿Sé qué hace cada una de estas flags: `-d`, `-it`, `--name`, `-p`, `-v`, `-e`, `--rm`, `--restart`, `--network`?
- [ ] ¿Puedo desplegar Nginx mapeando puertos y montando configuración personalizada?
- [ ] ¿Sé listar contenedores con filtros y formato personalizado?
- [ ] ¿Puedo extraer la IP de un contenedor usando `docker inspect --format`?
- [ ] ¿Entiendo los estados de un contenedor y qué comando causa cada transición?
- [ ] ¿Sé en qué directorio del host se almacenan los logs de un contenedor?
- [ ] ¿Puedo conectar dos contenedores por nombre usando una red definida por el usuario?
- [ ] ¿Sé hacer backup de un volumen Docker?
- [ ] ¿Puedo desplegar el stack del laboratorio (Nginx + MySQL + Adminer + Redis) sin mirar la guía?

Si respondiste SÍ a todas, estás listo para el **Capítulo 3: Imágenes y Registries**, donde explorarás el sistema de capas, Docker Hub, registries privados y la gestión avanzada de imágenes.
