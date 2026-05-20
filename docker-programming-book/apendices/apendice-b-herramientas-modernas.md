# Apéndice B: Herramientas Modernas del Ecosistema Docker

> "Docker no es solo un motor de contenedores. Es un ecosistema en evolución constante. Las herramientas que aprendiste hace 2 años ya tienen reemplazos más rápidos, más seguros y más integrados."

Este apéndice cubre las herramientas y funcionalidades modernas que complementan —y en algunos casos reemplazan— los flujos de trabajo clásicos de Docker. Desde entornos de desarrollo hasta escaneo de vulnerabilidades, builds multi-arquitectura y alternativas a Docker Engine.

---

## B.1 Dev Containers (Development Containers)

### ¿Qué son?

Los **Dev Containers** son entornos de desarrollo completamente contenerizados y reproducibles. En lugar de instalar Node.js, Python, Go, compiladores y bases de datos en tu máquina host, defines un entorno de desarrollo como código (`.devcontainer.json` + Dockerfile) y trabajas dentro de un contenedor con todas las herramientas preconfiguradas.

### ¿Por qué usarlos?

- **Onboarding en segundos**: `git clone` + "Reopen in Container" → entorno listo. Sin guías de instalación de 20 pasos.
- **Cero "en mi máquina funciona"**: Todos los desarrolladores del equipo usan exactamente el mismo entorno.
- **Aislamiento**: Python 3.8 para un proyecto, Python 3.12 para otro. Sin conflictos.
- **CI parity**: El mismo Dockerfile que usa el dev container puede ser la base de tu imagen de CI.

### Configuración mínima

Crea `.devcontainer/devcontainer.json` en tu proyecto:

```json
{
  "name": "TasksFlow Dev",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:22",

  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },

  "forwardPorts": [3000, 8080],

  "postCreateCommand": "npm install -g npm@latest && cd backend && npm ci",

  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "ms-azuretools.vscode-docker"
      ]
    }
  }
}
```

### Usar Docker Compose como Dev Container

```json
{
  "name": "TasksFlow Full Stack",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "api",
  "workspaceFolder": "/app",
  "shutdownAction": "none",
  "customizations": {
    "vscode": {
      "extensions": ["ms-azuretools.vscode-docker"]
    }
  }
}
```

Con esta configuración, VS Code usará el servicio `api` de tu docker-compose como entorno de desarrollo. Los demás servicios (db, cache) correrán en segundo plano.

---

## B.2 Docker Compose Watch (Hot Reload Nativo)

### El problema clásico

Tradicionalmente, para desarrollo con Docker Compose usabas bind mounts (`- ./src:/app/src`) para que los cambios en el código fuente se reflejaran dentro del contenedor. Esto funciona, pero tiene limitaciones:

- Los bind mounts tienen peor rendimiento que los volúmenes (especialmente en macOS/Windows).
- El contenedor no sabe que el archivo cambió; necesitas un file watcher dentro del contenedor (`nodemon`, `--watch`, `air`).
- Si cambias `package.json`, necesitas reconstruir la imagen manualmente.

### Docker Compose Watch (v2.22+)

Compose Watch es un mecanismo nativo que monitorea cambios en archivos del host y ejecuta acciones dentro del contenedor **sin bind mounts**:

```yaml
# docker-compose.yml
services:
  api:
    build: ./backend
    ports:
      - "3000:3000"
    develop:
      watch:
        - action: sync
          path: ./backend/src
          target: /app/src
        - action: rebuild
          path: ./backend/package.json
        - action: sync+restart
          path: ./backend/package-lock.json
          target: /app/package-lock.json
```

**Acciones disponibles:**

| Acción | Qué hace | Cuándo usarla |
|--------|----------|---------------|
| `sync` | Copia el archivo modificado al contenedor | Cambios en código fuente |
| `rebuild` | Reconstruye la imagen con `docker compose build` | Cambios en Dockerfile, dependencias |
| `sync+restart` | Copia el archivo y reinicia el contenedor | Cambios en configuración que requieren reinicio |
| `sync+exec` | Copia y ejecuta un comando | Compilación o recarga específica |

### Uso

```bash
# Iniciar con watch mode
docker compose watch

# En otra terminal, o integrado:
docker compose up --watch
```

### Ventajas sobre bind mounts

- **Rendimiento**: Usa `docker cp` internamente, que es más rápido que bind mounts en sistemas no-Linux.
- **Notificaciones**: El motor de Docker notifica al contenedor del cambio.
- **Granularidad**: Puedes especificar exactamente qué hacer con cada tipo de archivo.
- **Sin sidecar**: No necesitas `nodemon` ni scripts de watch dentro del contenedor.

---

## B.3 docker init — Generación Automática de Dockerfiles

### El problema

Crear un buen `Dockerfile`, `.dockerignore` y `compose.yaml` desde cero requiere experiencia. Muchos desarrolladores copian y pegan de tutoriales, introduciendo malas prácticas.

### docker init (Docker Desktop 4.19+)

`docker init` analiza tu proyecto, detecta el lenguaje/framework, y genera archivos Docker profesionales adaptados a tu aplicación:

```bash
# En la raíz de tu proyecto
docker init
```

El asistente interactivo te preguntará:

```
? What application platform does your project use?
  [ ] Go
  [ ] Python
  [x] Node.js
  [ ] Rust
  [ ] ASP.NET
  [ ] PHP (Laravel)
  [ ] Java
  [ ] Other

? What version of Node.js do you want to use? (20)

? What port does your server listen on? (3000)

? What is the command to run your app? (node src/server.js)
```

Y genera automáticamente:

```
├── .dockerignore      # Adaptado a tu stack (node_modules, .git, logs...)
├── Dockerfile         # Multi-stage con alpine, non-root, health check
├── compose.yaml       # Servicio + dependencias detectadas
└── README.Docker.md   # Instrucciones para construir y ejecutar
```

### Ejemplo de lo que genera para Node.js

```dockerfile
# syntax=docker/dockerfile:1
ARG NODE_VERSION=20.11.1

FROM node:${NODE_VERSION}-alpine AS base
WORKDIR /usr/src/app

FROM base AS deps
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

FROM base AS build
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=cache,target=/root/.npm \
    npm ci

FROM base AS final
ENV NODE_ENV=production
USER node
COPY package.json .
COPY --from=deps /usr/src/app/node_modules ./node_modules
COPY --from=build /usr/src/app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["node", "src/server.js"]
```

### Limitaciones

- No genera configuraciones avanzadas (BuildKit mounts, seccomp, distroless).
- Las opciones son limitadas; para proyectos complejos siempre necesitarás ajustar manualmente.
- Solo funciona con Docker Desktop (no con Docker Engine standalone).

---

## B.4 Docker Scout — Análisis de Vulnerabilidades y SBOM

### El problema

Las imágenes Docker contienen paquetes del sistema operativo y librerías que acumulan vulnerabilidades (CVEs) con el tiempo. Sin escaneo automatizado, estás desplegando software con agujeros de seguridad conocidos.

### Docker Scout

**Docker Scout** es la herramienta de análisis de seguridad integrada en Docker. Proporciona:

- **Análisis de vulnerabilidades (CVE)**: Escanea todas las capas de tu imagen.
- **SBOM (Software Bill of Materials)**: Lista completa de todos los componentes de tu imagen.
- **Recomendaciones de remediación**: Sugiere imágenes base alternativas o versiones de paquetes.
- **Comparación de imágenes**: Muestra cómo evoluciona la superficie de ataque entre versiones.
- **Políticas de calidad**: Define reglas como "cero CRITICALs", "sin licencias GPL", etc.

### Comandos esenciales

```bash
# Vista rápida de vulnerabilidades
docker scout quickview tasksflow-api:1.0.0

# Análisis detallado por CVE
docker scout cves tasksflow-api:1.0.0

# Análisis detallado con severidad mínima
docker scout cves tasksflow-api:1.0.0 --severity high,critical

# Comparar dos versiones
docker scout compare tasksflow-api:1.0.0 tasksflow-api:2.0.0

# Recomendaciones de imagen base
docker scout recommendations tasksflow-api:1.0.0

# Generar SBOM
docker scout sbom tasksflow-api:1.0.0 --format spdx

# Ver en modo watch (actualizaciones automáticas)
docker scout watch tasksflow-api:1.0.0
```

### Alternativa Open Source: Trivy

```bash
# Escanear imagen
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image tasksflow-api:1.0.0

# Solo críticas y altas
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image --severity CRITICAL,HIGH tasksflow-api:1.0.0

# Escanear sistema de archivos
trivy fs /

# Escanear repo git
trivy repo https://github.com/usuario/repo

# Escanear IaC (Dockerfile, k8s manifests)
trivy config ./k8s/
```

### Integrar en CI/CD

```yaml
# En GitHub Actions
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ghcr.io/${{ github.repository }}/tasksflow-api:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1  # Falla el pipeline si encuentra CRITICAL o HIGH
```

---

## B.5 Builds Multi-Arquitectura con buildx

### El problema

Si construyes una imagen en una máquina AMD64 (`linux/amd64`) y la ejecutas en una Raspberry Pi o un Mac Apple Silicon (`linux/arm64`), la imagen no funcionará porque la arquitectura de CPU no coincide. Históricamente, cada equipo mantenía Dockerfiles separados para cada arquitectura.

### Docker Buildx

`buildx` es el builder de nueva generación de Docker que soporta builds multi-plataforma nativos:

```bash
# Ver builders disponibles
docker buildx ls

# Crear un builder multi-plataforma
docker buildx create --name multiarch --driver docker-container --use

# Inspeccionar
docker buildx inspect --bootstrap

# Construir para múltiples arquitecturas
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/usuario/tasksflow-api:latest \
  --push \
  ./backend
```

### Estrategias de construcción

| Estrategia | Cómo funciona | Velocidad | Requisitos |
|------------|---------------|-----------|------------|
| **QEMU emulation** | Emula ARM en x86 (o viceversa) | Lenta (5-10x más) | Nada. Funciona out of the box. |
| **Cross-compilation** | Compila para la arquitectura destino desde la nativa | Rápida | El compilador debe soportar cross-compilation (Go, Rust, Zig: ✅; Node/Python: no necesitan) |
| **Nativa (remote builder)** | Construye en una máquina física de cada arquitectura | La más rápida | Necesitas una máquina ARM64 y otra AMD64 |

### Configuración recomendada

```yaml
# .github/workflows/ci-cd.yml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push multi-arch
  uses: docker/build-push-action@v6
  with:
    context: ./backend
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ghcr.io/usuario/tasksflow-api:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Inspeccionar arquitecturas de una imagen

```bash
# Ver qué arquitecturas soporta una imagen en el registry
docker buildx imagetools inspect ghcr.io/usuario/tasksflow-api:latest

# Output:
# Name:      ghcr.io/usuario/tasksflow-api:latest
# MediaType: application/vnd.docker.distribution.manifest.list.v2+json
# Digest:    sha256:abc123...
#
# Manifests:
#   Name:     ghcr.io/usuario/tasksflow-api:latest@sha256:def456...
#   Platform: linux/amd64
#
#   Name:     ghcr.io/usuario/tasksflow-api:latest@sha256:ghi789...
#   Platform: linux/arm64
```

---

## B.6 docker debug — Depuración Interactiva

### El problema

Cuando un contenedor falla, tradicionalmente tenías que:
1. Leer los logs (`docker logs`).
2. Sobrescribir el entrypoint (`docker run --entrypoint /bin/sh`).
3. Reconstruir una imagen de debug con herramientas adicionales.
4. Hacer port-forward para acceder a puertos internos.

Nada de esto es rápido ni cómodo.

### docker debug (Docker Desktop 4.27+ / Docker Scout)

`docker debug` te permite abrir una shell de depuración dentro de **cualquier contenedor** —incluso si el contenedor está parado, no tiene shell (`scratch`, `distroless`), o se creó sin `-it`.

```bash
# Abrir shell de debug en contenedor corriendo
docker debug <contenedor>

# Debug de contenedor parado
docker debug <contenedor-exit-code-1>

# Debug de imagen (sin crear contenedor)
docker debug <imagen>

# Con herramienta específica
docker debug --tool curl <contenedor>
```

### ¿Cómo funciona?

`docker debug` inyecta un sidecar (Nix environment) con herramientas de diagnóstico (curl, htop, strace, tcpdump, nslookup, vim, etc.) dentro del namespace del contenedor. No modifica la imagen original. Puedes depurar contenedores `FROM scratch` que no tienen ni `/bin/sh`.

### Herramientas disponibles en el shell de debug

```
curl, wget, jq, vim, nano, htop, iftop, iotop,
strace, ltrace, tcpdump, ngrep, nslookup, dig,
netcat, socat, nmap, iperf3, procps, psmisc
```

### Alternativa: nsenter (sin Docker Desktop)

```bash
# Obtener PID del contenedor
PID=$(docker inspect -f '{{.State.Pid}}' <contenedor>)

# Entrar en todos los namespaces del proceso
sudo nsenter -t $PID -a /bin/bash
# o usar una herramienta de debug:
sudo nsenter -t $PID -a strace -p 1
```

---

## B.7 Podman vs Docker — La Alternativa Sin Daemon

### ¿Qué es Podman?

**Podman** (Pod Manager) es un motor de contenedores creado por Red Hat que no requiere un daemon corriendo en segundo plano. Los contenedores se ejecutan como procesos hijos directos de Podman, igual que cualquier otro proceso del sistema.

### Diferencias clave

| Característica | Docker | Podman |
|----------------|--------|--------|
| **Arquitectura** | Cliente-servidor (daemon `dockerd`) | Sin daemon (fork/exec directo) |
| **Root requerido** | Sí (aunque existe rootless mode) | No. Rootless por defecto. |
| **Systemd integration** | Manual | Nativo (genera units de systemd) |
| **Docker Compose** | `docker compose` | `podman-compose` (compatible) |
| **Compatibilidad CLI** | `docker ...` | `podman ...` (99% idéntico) |
| **Kubernetes** | YAML manual | `podman generate kube` (desde pods existentes) |
| **Pods** | No tiene concepto de pod | Soporte nativo de pods |
| **Registries** | Docker Hub por defecto | Mismos registries, mismos protocolos |

### Uso básico (99% igual que Docker)

```bash
# Instalar (macOS: podman machine start)
brew install podman
podman machine init
podman machine start

# Alias para compatibilidad
alias docker=podman
alias docker-compose=podman-compose

# Comandos idénticos
podman run -d --name web -p 8080:80 nginx:alpine
podman ps
podman images
podman build -t miapp .
podman logs web
podman exec -it web sh
```

### Pods (exclusivo de Podman)

Un **pod** es un grupo de contenedores que comparten el mismo namespace de red (y opcionalmente PID e IPC). Es el equivalente a un Pod de Kubernetes, pero local:

```bash
# Crear un pod
podman pod create --name myapp -p 8080:80 -p 3000:3000

# Añadir contenedores al pod
podman run -d --pod myapp --name api tasksflow-api:1.0.0
podman run -d --pod myapp --name web tasksflow-web:1.0.0

# Los contenedores del pod se comunican por localhost
podman exec api curl localhost:80

# Generar YAML de Kubernetes desde el pod
podman generate kube myapp > pod.yaml
# Ahora puedes desplegar en Kubernetes con:
kubectl apply -f pod.yaml
```

### ¿Cuándo usar Podman?

| Caso | Recomendación |
|------|---------------|
| CI/CD pipelines (GitHub Actions, GitLab CI) | Podman: sin daemon, sin root, más seguro |
| Desarrollo local en Linux | Podman: integración con systemd, rootless |
| Equipos que también usan Kubernetes | Podman: pods nativos, generation a K8s YAML |
| Empresas con políticas estrictas de seguridad | Podman: rootless por defecto, sin daemon privilegiado |
| Ecosistema Docker existente (Compose, Hub, herramientas) | Docker: madurez del ecosistema |
| Docker Desktop (macOS/Windows) | Docker: mejor experiencia GUI, `docker init`, `docker debug`, `docker scout` |

### Buildah y Kaniko

**Buildah** (compañero de Podman): Construye imágenes OCI sin Dockerfile, usando comandos imperativos o Dockerfiles. Ideal para scripts de CI donde quieres control total.

```bash
# Construir sin Dockerfile
buildah from alpine
buildah run alpine-working-container -- apk add --no-cache curl
buildah config --cmd "/bin/sh" alpine-working-container
buildah commit alpine-working-container my-custom-image
```

**Kaniko**: Construye imágenes Docker dentro de Kubernetes o entornos sin daemon Docker. Ideal para pipelines de CI donde no tienes acceso al socket de Docker.

```yaml
# En GitLab CI
build:
  image: gcr.io/kaniko-project/executor:debug
  script:
    - /kaniko/executor
      --context $CI_PROJECT_DIR
      --dockerfile $CI_PROJECT_DIR/Dockerfile
      --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

---

## B.8 WebAssembly (Wasm) con Docker

Docker soporta **WebAssembly (Wasm)** como runtime alternativo a los contenedores Linux tradicionales. Wasm ofrece arranques en microsegundos, seguridad reforzada (sandbox por defecto), y portabilidad total entre arquitecturas.

```bash
# Habilitar Wasm en Docker Desktop (Settings → Features in development → Enable Wasm)

# Ejecutar un módulo Wasm
docker run --runtime=io.containerd.wasmedge.v1 \
  --platform=wasi/wasm \
  wasmedge/example-wasi:latest
```

El ecosistema Wasm aún está en fase temprana para aplicaciones generales, pero es el runtime más prometedor para edge computing, plugins y serverless.

---

## B.9 Resumen: ¿Qué Herramienta Usar y Cuándo?

| Necesidad | Herramienta | Alternativa |
|-----------|-------------|-------------|
| Entorno de desarrollo reproducible | **Dev Containers** (VS Code) | Nix shell, Vagrant |
| Hot reload en desarrollo | **Docker Compose Watch** (v2.22+) | Bind mounts + nodemon |
| Generar Dockerfiles rápido | **docker init** | Copiar de proyectos similares |
| Escanear vulnerabilidades | **Docker Scout** / Trivy | Snyk, Clair, Grype |
| Construir para ARM + x86 | **docker buildx** | Buildah con qemu |
| Depurar contenedor sin shell | **docker debug** | nsenter, ephemeral debug container |
| Sin daemon, rootless | **Podman** | nerdctl (contaiNERD CTL) |
| Construir sin Docker daemon (CI) | **Kaniko** | Buildah |
| Pods locales como K8s | **Podman pods** | Minikube, Kind |
| Serverless / edge computing | **Docker + Wasm** | WasmEdge, Wasmtime |
