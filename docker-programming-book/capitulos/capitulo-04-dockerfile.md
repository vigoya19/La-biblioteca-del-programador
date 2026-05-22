# Capítulo 4: El Dockerfile Completo

> *"El Dockerfile no es solo un script de construcción — es la firma del ingeniero, el
> contrato entre desarrollo y producción, y el artefacto más importante del ecosistema
> Docker."*

---

Un Dockerfile es un documento de texto que contiene una serie de instrucciones que Docker
ejecuta secuencialmente para construir una **imagen**. Cada instrucción crea una **capa**
en el sistema de archivos de la imagen, formando una pila inmutable de cambios. Dominar el
Dockerfile es dominar Docker: aquí se define cómo se comporta tu aplicación en producción,
qué tan segura es, cuánto pesa y qué tan rápido se despliega.

Este capítulo es el más extenso del libro porque cada instrucción merece un análisis
profundo. No basta con saber que `FROM` elige la imagen base; hay que entender cómo una
mala elección añade 300MB innecesarios. No basta con saber que `CMD` ejecuta un comando;
hay que entender que escribirlo en *shell form* hará que tu contenedor ignore SIGTERM y
tarde 10 segundos en detenerse en producción.

Vamos a diseccionar cada instrucción, sus trampas, sus mejores prácticas y sus
combinaciones. Al final de este capítulo, serás capaz de escribir Dockerfiles que sean:

- **Mínimos**: imágenes de 5MB en lugar de 900MB.
- **Seguros**: sin secretos filtrados, sin procesos como root.
- **Reproducibles**: builds deterministas, versiones fijadas con SHA256.
- **Eficientes**: aprovechando la caché de capas al máximo.
- **Profesionales**: multi-stage, BuildKit, health checks, señales Unix correctas.

---

## 4.1 FROM — La Base de Todo

### 4.1.1 Sintaxis

```
FROM <imagen>[:<tag>] [AS <nombre-stage>]
FROM <imagen>[@<digest>] [AS <nombre-stage>]
```

`FROM` es siempre la primera instrucción del Dockerfile (salvo `ARG` antes de `FROM`,
que tiene un comportamiento especial). Establece la **imagen base** sobre la que se
construirá todo lo demás. Puedes referenciar una imagen por tag o por digest SHA256.

```dockerfile
FROM node:20-alpine
FROM node:20-alpine@sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
FROM node:20-alpine AS builder
FROM scratch
```

### 4.1.2 Elección de Imagen Base

La imagen base es la decisión de mayor impacto en tu imagen final. Determina el tamaño,
la superficie de ataque, las dependencias disponibles y el comportamiento en runtime.
No existe una respuesta universal — cada proyecto tiene necesidades distintas.

| Imagen | Tamaño aprox. | Package Manager | libc | Caso de uso |
|---|---|---|---|---|
| `scratch` | 0 bytes | Ninguno | Ninguna | Binarios estáticos (Go, Rust) |
| `alpine:3.20` | ~5 MB | `apk` | musl | Imagen mínima con shell |
| `debian:stable-slim` | ~50 MB | `apt` | glibc | Compatibilidad total con Linux |
| `ubuntu:24.04` | ~70 MB | `apt` | glibc | Familiaridad del equipo |
| `distroless/java17` | ~120 MB | Ninguno | glibc | Solo runtime de Java |
| `centos:stream9` | ~150 MB | `dnf` | glibc | Entornos enterprise legacy |

**Tabla comparativa detallada:**

```
┌────────────────────┬──────────┬───────────┬──────────────┬──────────────────────┐
│ Imagen base        │ Tamaño   │ CVE count │ Superficie   │ Velocidad de build   │
│                    │ (aprox.) │ (approx.) │ de ataque    │ (cold start)         │
├────────────────────┼──────────┼───────────┼──────────────┼──────────────────────┤
│ scratch            │ 0 MB     │ 0         │ Nula         │ Instantánea          │
│ alpine:3.20        │ 5 MB     │ ~0-2      │ Mínima       │ ~1s                  │
│ debian:stable-slim │ 50 MB    │ ~20-40    │ Baja         │ ~3s                  │
│ ubuntu:24.04       │ 70 MB    │ ~30-60    │ Media        │ ~5s                  │
│ node:20-alpine     │ 120 MB   │ ~5-10     │ Baja         │ ~2s                  │
│ node:20-slim       │ 200 MB   │ ~40-60    │ Media        │ ~5s                  │
│ node:20 (default)  │ 1.1 GB   │ ~600-800  │ Alta         │ ~15s                 │
│ python:3.12-alpine │ 50 MB    │ ~5-10     │ Baja         │ ~2s                  │
│ python:3.12-slim   │ 130 MB   │ ~40-60    │ Media        │ ~5s                  │
│ python:3.12        │ 1.0 GB   │ ~600-800  │ Alta         │ ~15s                 │
│ openjdk:17-slim    │ 200 MB   │ ~40-60    │ Media        │ ~5s                  │
│ openjdk:17         │ 470 MB   │ ~200-300  │ Alta         │ ~10s                 │
└────────────────────┴──────────┴───────────┴──────────────┴──────────────────────┘
```

**Principios para elegir imagen base:**

1. **Empieza con `slim` o `alpine`**. La versión `:latest` o sin sufijo (`ubuntu`, `node`)
   incluye toolchains de compilación, documentación, manuales y headers innecesarios en
   producción. La variante `:slim` elimina todo eso excepto las dependencias runtime.

2. **Considera musl vs glibc**. Alpine usa musl libc, no glibc.

   > [!NOTE]
   > ### 🗣️ glibc vs. musl: El Traductor de Bolsillo vs. El Intérprete Profesional
   > 
   > Las aplicaciones que escribimos en lenguajes como Python, Node.js o Java eventualmente necesitan comunicarse con el "cerebro" del sistema operativo (el Kernel de Linux) para pedir cosas como leer un archivo o enviar datos por internet. Para lograrlo, usan una librería traductora estándar de C.
   > 
   > - **`glibc` (El Intérprete Profesional - Debian/Ubuntu)**: Es un traductor con un vocabulario gigantesco, preparado para entender cualquier dialecto, modismo o palabra técnica compleja. Es sumamente compatible con todo el software del mundo, pero pesa bastante (hace que las imágenes sean más grandes).
   > - **`musl` (El Traductor de Bolsillo - Alpine)**: Es una pequeña libreta de traducción supercompacta y ligera. Contiene solo lo indispensable para comunicarse rápidamente. Gracias a esto, las imágenes de Alpine pesan apenas 5 MB.
   > 
   > **¿La trampa?** Si tu aplicación viene con un módulo binario precompilado que usa una palabra sumamente técnica y compleja escrita específicamente para el intérprete profesional (`glibc`), el traductor de bolsillo (`musl`) no la entenderá y tu contenedor fallará.
   > 
   > **Regla de oro sencilla**: Si tu aplicación es puramente web y usa paquetes estándar, usa **Alpine** (ligero). Si usas librerías científicas pesadas de Python (como NumPy o Pandas) o extensiones nativas complejas de C, prefiere **Debian Slim** (`glibc`) para evitar dolores de cabeza.

3. **No uses `:latest` nunca**. `FROM node:latest` es una bomba de tiempo. Lo que hoy
   es Node 20 mañana será Node 22 y tu build se romperá sin cambios en tu código.
   Siempre especifica una versión concreta: `FROM node:20.11.1-alpine`.

### 4.1.3 Distroless: Solo Runtime, Nada Más

Las imágenes **distroless** de Google son la respuesta a la pregunta: "¿qué es lo
mínimo que necesita mi aplicación para ejecutarse?". No contienen package manager,
ni shell, ni siquiera `ls` o `cat`. Solo el runtime y sus dependencias de biblioteca
compartidas.

```
gcr.io/distroless/java17        → JRE 17, sin javac, sin jar, sin shell
gcr.io/distroless/nodejs20      → Node.js 20 runtime, sin npm, sin shell
gcr.io/distroless/python3       → Python 3 runtime, sin pip, sin shell
gcr.io/distroless/static        → ca-certificates, tzdata, /etc/passwd (para binarios estáticos)
gcr.io/distroless/base          → static + glibc, libssl, libcrypto (para binarios dinámicos)
gcr.io/distroless/cc            → base + libstdc++ (para binarios C++)
```

**Ejemplo: aplicación Java con distroless**

```dockerfile
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

FROM gcr.io/distroless/java17-debian12
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Beneficios de distroless:**

- **Seguridad mejorada**: sin shell, un atacante no puede ejecutar `sh` para obtener
  acceso al contenedor, incluso si compromete la aplicación.
- **Imágenes más pequeñas**: al eliminar el package manager y utilidades, reduces de
  200-400MB a 80-120MB.
- **Menos CVEs**: sin herramientas innecesarias, no hay vulnerabilidades en esas
  herramientas.
- **Menor superficie de ataque**: cada binario eliminado es un vector de ataque menos.

**Desventajas de distroless:**

- **Sin shell para debugging**: no puedes hacer `docker exec -it contenedor sh`.
  Alternativa: `kubectl debug` o añadir un sidecar de debugging.
- **Sin package manager**: si tu app necesita instalar algo en runtime, distroless
  no sirve. Replantea si realmente necesitas instalar algo en runtime.
- **Compatibilidad**: las librerías compiladas deben ser compatibles con la libc y
  arquitectura de la imagen distroless.

**Estrategia de debugging con distroless:**

```bash
# No puedes hacer esto porque no hay shell:
# docker exec -it contenedor sh

# Alternativa 1: Ephemeral debug container (Kubernetes)
kubectl debug -it mi-pod --image=busybox --target=mi-contenedor

# Alternativa 2: Lanzar imagen 'debug' equivalente
# Las imágenes distroless tienen variantes :debug con busybox
docker run --rm -it gcr.io/distroless/java17-debian12:debug sh
```

### 4.1.4 Scratch: La Imagen Vacía

`scratch` es una pseudo-imagen de **0 bytes**. No contiene nada — ni sistema de
archivos, ni shell, ni libc, ni siquiera `/bin`. Es literalmente vacío. Solo
puedes ejecutar binarios **compilados estáticamente** que no dependan de ninguna
biblioteca del sistema.

```dockerfile
FROM scratch
COPY mi-binario /
ENTRYPOINT ["/mi-binario"]
```

**¿Cuándo usar scratch?**

| Lenguaje | Compilación estática | Ejemplo |
|---|---|---|
| Go | `CGO_ENABLED=0 go build` | `FROM golang:1.22 AS builder` → `FROM scratch` |
| Rust | `target/x86_64-unknown-linux-musl` | `FROM rust:1.78 AS builder` → `FROM scratch` |
| C/C++ | `gcc -static` | `FROM gcc:13 AS builder` → `FROM scratch` |
| Zig | Por defecto estática | `FROM zig:0.11 AS builder` → `FROM scratch` |
| .NET AOT | `dotnet publish -p:PublishAot=true` | `FROM mcr.microsoft.com/dotnet/sdk:8.0 AS builder` → `FROM scratch` |

**Ejemplo completo: Go → scratch**

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-w -s" \
    -o /bin/mi-app \
    ./cmd/mi-app

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /bin/mi-app /mi-app
EXPOSE 8080
ENTRYPOINT ["/mi-app"]
```

**Notas importantes sobre scratch:**

1. **No puedes usar shell form**. `CMD /mi-app` no funciona porque no hay `/bin/sh`.
   Siempre usa exec form: `CMD ["/mi-app"]`.

2. **Necesitas certificados CA**. Si tu app hace peticiones HTTPS, necesitas los
   certificados raíz. Cópialos del stage de build.

3. **No hay `/tmp`**. Crea directorios que necesites o asegúrate de que tu app cree
   sus propios directorios temporales.

4. **Debugging es imposible**. No puedes `exec`. No hay `sh`. Si la app falla, el
   contenedor muere y solo tienes los logs. Para desarrollo, usa `alpine` o una
   imagen intermedia; en producción, scratch.

5. **Peso total ~5-15MB**. Un binario Go de 10MB + certificados = imagen final de
   ~10MB. Compáralo con los ~900MB de usar `FROM ubuntu`.

### 4.1.5 Versionado de la Imagen Base: ¿Fijar o No Fijar el SHA256?

El versionado determinista es una de las decisiones más debatidas en el ecosistema
Docker. Veamos las opciones:

**Opción 1: Solo tag (no determinista)**

```dockerfile
FROM node:20-alpine
```

- **Pros**: simple, legible, fácil de actualizar con Dependabot/Renovate.
- **Contras**: `node:20-alpine` es un tag mutable. Hoy apunta a 20.11.1-alpine3.19;
  mañana a 20.12.0-alpine3.20. Tu build de CI puede romperse sin que cambies nada.

**Opción 2: Tag específico más SHA256 (determinista)**

```dockerfile
FROM node:20.11.1-alpine3.19@sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
```

- **Pros**: absolutamente determinista. La misma imagen exacta siempre. El build
  es reproducible en cualquier máquina, en cualquier momento del tiempo.
- **Contras**: el SHA256 es ilegible. Para actualizarlo necesitas una herramienta.
  El Dockerfile parece ofuscado.

**Opción 3: Tag exacto sin SHA (compromiso razonable)**

```dockerfile
FROM node:20.11.1-alpine3.19
```

- **Pros**: razonablemente determinista (los tags de versión exacta no se reasignan
  en imágenes oficiales). Legible. Fácil de actualizar con herramientas automáticas.
- **Contras**: en teoría, el mantenedor *podría* reasignar este tag. En la práctica,
  las imágenes oficiales de Docker Hub no lo hacen.

**Recomendación práctica:**

```dockerfile
# Para desarrollo local y CI: tag semi-determinista
FROM node:20.11.1-alpine3.19

# Para producción con requisitos de cumplimiento estrictos:
FROM node:20.11.1-alpine3.19@sha256:a1b2c3d4...
```

Puedes extraer el SHA256 actual de un tag con:

```bash
docker inspect node:20.11.1-alpine3.19 \
  --format='{{index .RepoDigests 0}}'
```

### 4.1.6 Multi-Stage: FROM Múltiple en un Solo Dockerfile

Los builds multi-stage permiten usar múltiples `FROM` en un mismo Dockerfile. Cada
`FROM` inicia una nueva etapa de build. Lo fundamental es que **solo la última etapa
llega a la imagen final** — las etapas anteriores se descartan, pero sus artefactos
pueden copiarse a la etapa final.

```dockerfile
# Etapa 1: compilación (se descarta)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Etapa 2: producción (esta es la imagen final)
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

**Por qué multi-stage es revolucionario:**

Antes de multi-stage (Docker < 17.05), necesitabas dos Dockerfiles o scripts externos
para lograr lo mismo: uno para compilar, otro para empaquetar. El resultado era imágenes
con SDKs, toolchains y artefactos intermedios que jamás deberían llegar a producción.
Multi-stage resuelve esto de forma nativa, dentro del propio Dockerfile.

**Reglas de multi-stage:**

1. Puedes tener tantas etapas como quieras.
2. Nombra las etapas con `AS nombre`; si no las nombras, se indexan como 0, 1, 2...
3. Solo la última etapa produce la imagen final (a menos que uses `--target`).
4. `COPY --from=etapa` copia archivos de una etapa anterior a la actual.
5. Las etapas pueden heredar de imágenes base completamente diferentes.

---

## 4.2 RUN — Ejecutar Comandos Durante el Build

### 4.2.1 Shell Form vs Exec Form

`RUN` tiene dos sintaxis, y la diferencia es profunda:

**Shell form:**

```dockerfile
RUN apt-get update && apt-get install -y curl
```

- Se ejecuta con `/bin/sh -c "comando"` (o el shell definido por `SHELL`).
- Las variables de entorno se expanden (`$HOME`, `$VERSION`).
- Puedes usar pipes (`|`), redirecciones (`>`), operadores lógicos (`&&`, `||`).
- En RUN es temporal durante el build, así que el problema del PID 1 (que sí afecta a CMD) no aplica aquí.

**Exec form:**

```dockerfile
RUN ["apt-get", "update"]
```

- Se ejecuta directamente, sin shell intermedio.
- **No hay expansión de variables**: `RUN ["echo", "$HOME"]` imprimirá literalmente `$HOME`.
- **No puedes usar pipes ni redirecciones**.
- Útil cuando quieres evitar la sobrecarga del shell o cuando la imagen base no tiene shell.

**Recomendación**: usa siempre **shell form** para `RUN`, porque la expansión de
variables, pipes y `&&` son esenciales para escribir Dockerfiles correctos y
eficientes. El exec form para RUN es raramente útil en la práctica.

### 4.2.2 Encadenamiento con && — El Arte de Fusionar Capas

Cada instrucción `RUN` crea una **capa** en la imagen. Las capas son inmutables y se
almacenan por separado. Una capa creada por `RUN` no puede modificarse en una capa
posterior — solo puede ocultarse (lo que no reduce el tamaño total).

**Mal — capas separadas que inflan la imagen:**

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get clean
```

Cada `RUN` genera una capa. `apt-get update` actualiza la cache en la capa 1. La capa
2 instala curl. La capa 3 instala vim. La capa 4 ejecuta `apt-get clean`, pero **no
puede eliminar los archivos creados en las capas anteriores** — esos archivos ya están
en capas inmutables. El resultado: la imagen contiene toda la caché de apt aunque
"limpiaste" al final.

**Bien — todo en una capa:**

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl vim \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

Todo ocurre en la misma capa. La caché de apt se descarga, se usa para instalar y se
elimina antes de que la capa se cierre. El resultado es mucho más pequeño.

**Reglas de oro para RUN:**

```
1. Encadena comandos relacionados con &&
2. Limpia en la misma capa que instalas
3. Usa --no-install-recommends para evitar dependencias innecesarias
4. Elimina las listas de paquetes: rm -rf /var/lib/apt/lists/*
5. Elimina las cachés de los package managers
```

### 4.2.3 Limpieza Específica por Package Manager

**Apt (Debian/Ubuntu):**

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /var/cache/apt/archives/*
```

**Apk (Alpine):**

```dockerfile
RUN apk add --no-cache \
        curl \
        ca-certificates
```

El flag `--no-cache` de apk ya hace la limpieza por ti. No necesitas `rm` adicional.
Para casos donde necesitas `--update`:

```dockerfile
RUN apk add --update --no-cache curl \
    && rm -rf /var/cache/apk/*
```

**DNF/YUM (CentOS/RHEL/Fedora):**

```dockerfile
RUN dnf install -y curl \
    && dnf clean all \
    && rm -rf /var/cache/dnf/*
```

**Pip (Python):**

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

**NPM (Node.js):**

```dockerfile
RUN npm ci --omit=dev && npm cache clean --force
```

### 4.2.4 Cache Busting con BuildKit

Por defecto, Docker cachea cada capa de RUN. Si la capa no cambia, Docker reusa la
caché. Esto es bueno para la velocidad, pero problemático cuando quieres forzar la
re-ejecución de un comando (por ejemplo, `apt-get update` necesita ejecutarse para
obtener los últimos índices de paquetes).

BuildKit introduce `--mount=type=cache` que separa el contenido cacheable del
comando en sí:

```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu:24.04
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y curl
```

Esto monta directorios de caché que **persisten entre builds** pero **no se incluyen
en la imagen final**. La caché de apt sobrevive entre builds diferentes, acelerando
drásticamente las reinstalaciones.

**Caché para diferentes package managers:**

```dockerfile
# Apt
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y ...

# Apk
RUN --mount=type=cache,target=/var/cache/apk \
    apk add ...

# NPM
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Pip
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Go modules
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# Maven
RUN --mount=type=cache,target=/root/.m2 \
    mvn package

# Cargo (Rust)
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    cargo build --release
```

**Modos de compartición (sharing):**

- `shared` (default): todos los builds concurrentes comparten la misma caché.
- `locked`: solo un build a la vez accede a la caché; los demás esperan.
- `private`: cada build tiene su propia copia de la caché.

### 4.2.5 RUN --network=none (Seguridad en Build)

BuildKit permite controlar el acceso a red durante `RUN`:

```dockerfile
RUN --network=none go build -o /app ./...
```

Con `--network=none`, el comando se ejecuta sin acceso a red. Esto es útil para:

1. **Verificar que tu build es autosuficiente**: si `go build` necesita descargar
   dependencias, fallará, indicándote que deberías haber ejecutado `go mod download`
   en un paso anterior (cacheable).

2. **Seguridad**: evita que dependencias maliciosas se comuniquen con el exterior
   durante el build (supply chain attack).

3. **Reproducibilidad**: garantiza que el build no depende de recursos de red que
   podrían no estar disponibles en el futuro.

**Opciones de --network:**

| Valor | Descripción |
|---|---|
| `default` | Acceso de red normal (por defecto) |
| `none` | Sin acceso de red |
| `host` | Usa la red del host directamente |

### 4.2.6 RUN --mount=type=secret — Secretos sin Dejarlos en la Imagen

El problema clásico: necesitas un token, una clave SSH o un archivo `.npmrc` privado
durante el build, pero **bajo ninguna circunstancia** debe quedar en la imagen final.

**La forma insegura (NUNCA hagas esto):**

```dockerfile
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > ~/.npmrc \
    && npm ci \
    && rm ~/.npmrc
```

`docker build --build-arg NPM_TOKEN=secreto123 .`

El token está ahora en el historial de la imagen:

```bash
$ docker history mi-imagen --no-trunc
IMAGE          CREATED BY
sha256:abc...  /bin/sh -c echo "//registry.npmjs.org/:_authToken=secreto123" > ~/.npmrc && npm ci && rm ~/.npmrc
```

**La forma segura con BuildKit:**

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

```bash
# El archivo con el secreto existe solo durante este RUN
echo "//registry.npmjs.org/:_authToken=secreto123" > ~/.npmrc

# Se monta como archivo temporal
export DOCKER_BUILDKIT=1
docker build --secret id=npmrc,src=$HOME/.npmrc -t mi-app .
```

**¿Qué pasa realmente?** BuildKit monta el archivo de secretos en `/root/.npmrc` solo
durante la ejecución de ese `RUN`. El archivo existe en un tmpfs (memoria RAM) y se
destruye cuando el comando termina. No aparece en ninguna capa de la imagen, no está
en el historial, no puede extraerse con `docker history`.

**Ejemplos con diferentes tipos de secretos:**

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim

# Secreto: token de PyPI privado
RUN --mount=type=secret,id=pypirc,target=/root/.pypirc \
    pip install mi-paquete-privado

# Secreto: clave SSH para clonar repos privados
RUN --mount=type=ssh \
    git clone git@github.com:empresa/repo-privado.git

# Secreto: certificados para firmar código
RUN --mount=type=secret,id=signing-key,target=/tmp/key.pem \
    cosign sign --key /tmp/key.pem mi-imagen:latest
```

### 4.2.7 RUN --mount=type=bind

Permite montar archivos o directorios del sistema de archivos del host o del contexto
de build para usarlos durante el build:

```dockerfile
RUN --mount=type=bind,from=builder,source=/app/dist,target=/tmp/dist \
    cp /tmp/dist/* /var/www/html/
```

Esto es útil para compartición de datos entre stages sin usar COPY, o para montar
configuraciones del host (como mirrors de paquetes locales).

### 4.2.8 Buenas y Malas Prácticas con RUN

**Mal — instalación no reproducible:**

```dockerfile
RUN apt-get update && apt-get install -y nodejs
```

El paquete `nodejs` de los repositorios de Debian suele ser muy antiguo. Mejor usa
la imagen oficial de Node o el repositorio de NodeSource con una versión fija.

**Bien — versión fija y fuente confiable:**

```dockerfile
FROM node:20.11.1-alpine3.19
```

O, si realmente necesitas instalarlo en otra base:

```dockerfile
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs=20.11.1-1nodesource1 \
    && rm -rf /var/lib/apt/lists/*
```

**Mal — comandos no idempotentes:**

```dockerfile
RUN curl -o /tmp/script.sh https://example.com/latest-script.sh \
    && bash /tmp/script.sh
```

Si la URL cambia, tu build cambia sin que veas la diferencia en el código fuente.

**Bien — versionar el script, verificar checksum:**

```dockerfile
ADD https://example.com/v2.5/script.sh /tmp/script.sh
RUN echo "a1b2c3d4e5f6... /tmp/script.sh" | sha256sum -c - \
    && bash /tmp/script.sh
```

**Mal — dependencia de red innecesaria:**

```dockerfile
RUN go get ./... && go build -o /app ./...
```

Cada build descarga dependencias. Incluso sin cambios en el código.

**Bien — separar descarga de compilación:**

```dockerfile
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /app ./...
```

Las dependencias se cachean hasta que cambian `go.mod` o `go.sum`.

---

## 4.3 COPY vs ADD — La Diferencia Crucial

### 4.3.1 COPY: La Herramienta Correcta para el 99% de los Casos

`COPY` copia archivos y directorios desde el **contexto de build** (el directorio donde
ejecutaste `docker build`) al sistema de archivos del contenedor.

**Sintaxis:**

```dockerfile
COPY [--chown=<user>:<group>] [--chmod=<perms>] <src>... <dest>
COPY [--chown=<user>:<group>] [--chmod=<perms>] ["<src>",... "<dest>"]
```

**Reglas de COPY:**

1. `<src>` debe estar dentro del contexto de build. No puedes copiar `../algo` ni
   `/etc/passwd` del host.
2. `<dest>` puede ser ruta absoluta o relativa a `WORKDIR`.
3. Si `<dest>` no termina en `/`, se asume que es un archivo y el contenido se
   copia con ese nombre.
4. Si `<src>` es un directorio, se copia su contenido, no el directorio en sí (a
   menos que `<dest>` termine en `/nombre-directorio/`).

```dockerfile
# Copia todos los archivos del contexto al directorio /app
COPY . /app

# Copia un archivo específico
COPY package.json /app/

# Copia con cambio de propietario
COPY --chown=1000:1000 app-files/ /home/node/app/

# Copia múltiples archivos a un directorio
COPY package.json package-lock.json /app/
```

### 4.3.2 ADD: COPY con Funcionalidades Extra (y Peligrosas)

`ADD` hace todo lo que `COPY` más:

1. **Extracción automática de archivos tar**: si `<src>` es un archivo `.tar`, `.tar.gz`,
   `.tar.bz2` o `.tar.xz`, ADD lo descomprime automáticamente en `<dest>`.

2. **Descarga de URLs**: si `<src>` es una URL, ADD descarga el recurso y lo coloca en
   `<dest>`. Si además es un tar comprimido, lo descomprime.

**Sintaxis idéntica a COPY:**

```dockerfile
ADD archivo.tar.gz /app/       # Descomprime el contenido en /app/
ADD https://ejemplo.com/archivo /app/   # Descarga y coloca en /app/
ADD archivo.txt /app/          # Igual que COPY
```

**¿Por qué DEBES evitar ADD?**

1. **Magia inesperada**: `ADD archivo.tar.gz /app` descomprime. `COPY archivo.tar.gz /app`
   copia el archivo. El comportamiento de ADD es no-obvio y fuente de bugs silenciosos.

2. **URLs no son reproducibles**: `ADD https://...` descarga un recurso externo que
   puede cambiar. Tu build deja de ser determinista. Peor aún: si la URL desaparece,
   tu Dockerfile se rompe sin que nada en tu código haya cambiado.

3. **Caché impredecible**: Docker cachea basándose en el contenido. Una URL remota
   puede devolver contenido diferente sin que Docker lo detecte como cambio.

4. **La extracción de tar puede ser un vector de ataque**: si extraes un tar malicioso
   que sobrescribe archivos del sistema...

**Excepciones legítimas para ADD:**

1. **Archivos tar que necesitan extraerse**: `ADD app-dist.tar.gz /var/www/html/`.
   Aunque COPY + RUN tar es más explícito y preferible.

2. **URLs con checksum verificado**: `ADD https://releases.example.com/app-v1.2.3.tar.gz /tmp/`
   seguido de `RUN echo "sha256:abc123..." | sha256sum -c`.

**Recomendación definitiva:**

> Usa `COPY` siempre. Si necesitas descomprimir un tar: `COPY archivo.tar.gz /tmp/ && RUN tar -xzf /tmp/archivo.tar.gz -C /app && rm /tmp/archivo.tar.gz`. Si necesitas descargar algo: `RUN curl -fsSL -o /tmp/archivo URL && verificar_checksum`.

### 4.3.3 .dockerignore: El Guardián del Contexto de Build

Cuando ejecutas `docker build`, Docker empaqueta todo el directorio actual (el
**contexto de build**) y lo envía al daemon. Si tu proyecto tiene 2GB en
`node_modules`, `__pycache__`, `.git`, `build/` o `target/`, **todo eso se envía**
antes de que Docker procese siquiera la primera instrucción `FROM`.

`.dockerignore` es un archivo en la raíz del contexto de build que especifica qué
archivos y directorios **excluir** del contexto. Es funcionalmente idéntico a
`.gitignore`, pero para Docker.

**Por qué es vital:**

```
Sin .dockerignore:
  Contexto enviado: 850MB (node_modules, .git, build/, logs/, etc.)
  Tiempo de envío: 45 segundos
  Cada build: 45 segundos perdidos

Con .dockerignore:
  Contexto enviado: 50MB (solo código fuente)
  Tiempo de envío: 2 segundos
  Cada build: 2 segundos
```

**Ejemplo completo para Node.js:**

```dockerignore
# .dockerignore para aplicación Node.js

# Dependencias (se instalarán en el contenedor)
node_modules/
npm-debug.log*

# Build output (se generará en el contenedor)
dist/
build/
.next/
nuxt/

# Control de versiones
.git/
.gitignore
.gitattributes

# CI/CD y configuración local
.github/
.gitlab-ci.yml
Jenkinsfile
.dockerignore
Dockerfile
docker-compose*.yml

# IDE y editores
.vscode/
.idea/
*.swp
*.swo
*~

# Documentación (no necesaria en build)
docs/
*.md
!README.md
LICENSE

# Variables de entorno locales
.env
.env.local
.env.*.local

# Logs
*.log
logs/

# Dependencias de sistema operativo
.DS_Store
Thumbs.db

# Testing
coverage/
.nyc_output/
test/
tests/
__tests__/
*.test.js
*.spec.js

# Archivos temporales
tmp/
temp/
*.tmp
```

**Ejemplo para Python:**

```dockerignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.egg-info/
.eggs/
dist/
build/
*.whl

# Entorno virtual
venv/
.venv/
env/
.env/

# Dependencias (se instalarán en el contenedor)
site-packages/

# Testing y CI
.pytest_cache/
.coverage
htmlcov/
.tox/
.github/
.gitlab-ci.yml

# Control de versiones
.git/
.gitignore

# IDE
.vscode/
.idea/

# Configuración local
.env
.env.*

# OS
.DS_Store
```

**Ejemplo para Java:**

```dockerignore
# Java / Maven / Gradle
target/
*.class
*.jar
*.war
*.ear

# Gradle
.gradle/
build/

# IDE
.idea/
*.iml
.classpath
.project
.settings/

# Control de versiones
.git/
.gitignore

# CI/CD
.github/
Jenkinsfile

# OS
.DS_Store

# Logs
*.log
```

**Ejemplo para Go:**

```dockerignore
# Go
*.exe
*.exe~
*.dll
*.so
*.dylib
bin/
vendor/

# Build cache
*.test
*.out

# Control de versiones
.git/
.gitignore

# IDE
.vscode/
.idea/

# OS
.DS_Store

# Archivos específicos de CI
.github/
```

**Sintaxis de .dockerignore:**

```
# Comentario con #
*           # Wildcard (cualquier cadena)
?           # Comodín de un carácter
**          # Cualquier número de directorios
!           # Excepción (no ignorar)

# Ejemplos:
*.log       # Ignorar todos los .log
!app.log    # Excepto app.log
temp/       # Ignorar directorio temp/
**/temp     # Ignorar cualquier directorio temp en cualquier nivel
```

**Trampas comunes:**

```dockerignore
# MAL — esto ignora node_modules en TODOS los niveles
node_modules

# BIEN — solo ignora en la raíz
/node_modules

# MAL — esto ignora todos los .git
.git

# BIEN — solo el .git de la raíz
/.git
```

### 4.3.4 COPY con --from (Multi-Stage)

En builds multi-stage, `COPY --from` permite copiar archivos de una etapa anterior
a la etapa actual:

```dockerfile
COPY --from=builder /app/dist /usr/share/nginx/html
COPY --from=0 /app/output /app/input
```

Donde `builder` es el nombre asignado con `AS`, o `0` es el índice de la etapa
(0 para la primera).

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.25-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 4.3.5 Permisos con --chown y --chmod

Los contenedores modernos no se ejecutan como root (o no deberían). Los archivos
copiados con `COPY` pertenecen a root (uid 0, gid 0) por defecto. Si tu aplicación
corre con un UID diferente, no podrá leer/escribir esos archivos.

```dockerfile
FROM node:20-alpine

# Crear usuario no-root
RUN addgroup -g 1000 appgroup \
    && adduser -u 1000 -G appgroup -s /bin/sh -D appuser

# Copiar archivos con el propietario correcto
COPY --chown=1000:1000 package.json package-lock.json /app/
COPY --chown=1000:1000 src/ /app/src/

# Cambiar al usuario
USER 1000:1000
WORKDIR /app

RUN npm ci
CMD ["node", "src/index.js"]
```

**Alternativa — cambiar propietario después:**

```dockerfile
COPY . /app/
RUN chown -R 1000:1000 /app
```

Pero esto añade una capa extra. `--chown` es más eficiente porque ocurre en la misma
operación de COPY.

**--chmod (requiere BuildKit):**

```dockerfile
COPY --chmod=755 script.sh /usr/local/bin/
```

---
## 4.4 CMD vs ENTRYPOINT — SECCIÓN CRÍTICA

Esta es, sin exagerar, la sección más importante después de elegir la imagen base.
La diferencia entre `CMD` y `ENTRYPOINT` y la elección entre *shell form* y *exec
form* determinan cómo tu contenedor recibe señales Unix, cómo se detiene, y qué
tan bien se integra con orquestadores como Kubernetes o Swarm.

### 4.4.1 CMD: El Comando por Defecto

`CMD` establece el comando (y argumentos) que se ejecutarán cuando el contenedor
arranque, **a menos que el usuario especifique otro comando** en `docker run`.

**Tres formas de CMD:**

```dockerfile
# 1. Exec form (RECOMENDADA)
CMD ["ejecutable", "param1", "param2"]

# 2. Shell form (EVITAR)
CMD ejecutable param1 param2

# 3. Parámetros para ENTRYPOINT (combinación)
CMD ["param1", "param2"]
```

**CMD en exec form:**

```dockerfile
FROM ubuntu:24.04
CMD ["echo", "Hola desde CMD"]
```

```bash
$ docker run mi-imagen
Hola desde CMD

$ docker run mi-imagen echo "Yo decido"
Yo decido
```

**CMD en shell form:**

```dockerfile
FROM ubuntu:24.04
CMD echo "Hola desde CMD"
```

Se ejecuta como: `/bin/sh -c "echo Hola desde CMD"`

### 4.4.2 ENTRYPOINT: El Comando Fijo

`ENTRYPOINT` define el ejecutable que será el **propósito mismo del contenedor**.
No puede ser sobrescrito por `docker run` (salvo con el flag `--entrypoint`).

Un contenedor con ENTRYPOINT define **qué es**: un proxy nginx, un servidor de
aplicaciones, un worker de colas. CMD complementa como argumentos por defecto.

```dockerfile
FROM nginx:1.25-alpine
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

```bash
# Ejecuta nginx -g 'daemon off;'
$ docker run mi-nginx

# Ejecuta nginx -t (solo test de configuración)
$ docker run mi-nginx -t

# CMD fue sobrescrito, ENTRYPOINT se mantuvo
```

### 4.4.3 El Problema de Señales Unix: Shell Form vs Exec Form

Esta es la razón técnica fundamental por la que **siempre debes usar exec form**
para CMD y ENTRYPOINT:

**Shell form — ROMPE las señales Unix:**

```dockerfile
FROM ubuntu:24.04
CMD ping localhost
```

¿Qué procesos se ejecutan realmente?

```
PID 1: /bin/sh -c "ping localhost"
PID 7: ping localhost
```

El problema: **PID 1 es el shell, no tu aplicación**. Cuando Docker envía SIGTERM
al contenedor (con `docker stop`), el kernel envía la señal a PID 1. Pero `/bin/sh`
**no reenvía señales a sus procesos hijos**. Resultado: el shell muere, pero `ping`
sigue ejecutándose como proceso huérfano. Docker espera 10 segundos (el grace period)
y luego envía SIGKILL, matando todo forzosamente.

**Demostración del problema:**

```bash
# Shell form - tarda 10 segundos en detenerse
$ time docker stop contenedor-shell
contenedor-shell
docker stop contenedor-shell  0.04s user 0.02s system 0% cpu 10.234 total

# Exec form - se detiene en milisegundos
$ time docker stop contenedor-exec
contenedor-exec
docker stop contenedor-exec  0.04s user 0.02s system 0% cpu 0.023 total
```

**Exec form — las señales llegan correctamente:**

```dockerfile
FROM ubuntu:24.04
CMD ["ping", "localhost"]
```

PID 1 es directamente `ping`. Cuando Docker envía SIGTERM, `ping` lo recibe y responde
inmediatamente. El contenedor se detiene en milisegundos.

**Tabla comparativa shell form vs exec form para CMD/ENTRYPOINT:**

| Característica | Shell form | Exec form |
|---|---|---|
| PID 1 | `/bin/sh` | Tu aplicación |
| Recibe señales Unix | NO | SÍ |
| `docker stop` | Lento (~10s) | Instantáneo |
| Variables de entorno | Expandidas | Sin expandir |
| Pipes, redirecciones | Funcionan | No funcionan |
| Shell features (&&, \|\|) | Funcionan | No funcionan |
| Recomendado para producción | NO | SÍ |

**Cómo ejecutar comandos complejos con exec form:**

Si necesitas pipes, variables o lógica de shell, encapsúlalos en un script:

```dockerfile
FROM ubuntu:24.04
COPY entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

```bash
#!/bin/sh
# entrypoint.sh
export APP_ENV="${APP_ENV:-production}"
echo "Iniciando en $APP_ENV"
exec ./mi-app --port "${PORT:-8080}"
```

Lo importante es `exec` al final: reemplaza el shell por tu aplicación, haciendo
que tu app sea PID 1 y reciba señales correctamente.

### 4.4.4 Combinación CMD + ENTRYPOINT

La combinación más poderosa: ENTRYPOINT define el ejecutable, CMD establece los
parámetros por defecto (que el usuario puede sobrescribir en `docker run`).

**Tabla de combinaciones:**

| Caso | Dockerfile | `docker run` sin args | `docker run` con args |
|---|---|---|---|
| Solo CMD | `CMD ["app"]` | `app` | Los args reemplazan CMD |
| Solo ENTRYPOINT | `ENTRYPOINT ["app"]` | `app` | `app` + args al final |
| ENTRYPOINT + CMD | `ENTRYPOINT ["app"]` `CMD ["--help"]` | `app --help` | `app` + args (CMD ignorado) |
| Shell CMD | `CMD app --port 80` | `/bin/sh -c "app --port 80"` | Reemplazado |
| Shell ENTRYPOINT | `ENTRYPOINT app` | `/bin/sh -c "app"` | Ignorado a menos que --entrypoint |

**Ejemplos prácticos:**

```dockerfile
# Nginx: ENTRYPOINT fijo, CMD como argumentos por defecto
FROM nginx:1.25-alpine
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]

# Test de configuración sin arrancar: docker run mi-nginx -t

# Python: ENTRYPOINT como wrapper de setup, CMD como app
FROM python:3.12-slim
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["gunicorn", "mi_app.wsgi:application", "--bind", "0.0.0.0:8000"]

# APP como ENTRYPOINT con subcomando por defecto
FROM node:20-alpine
ENTRYPOINT ["node"]
CMD ["dist/server.js"]

# docker run mi-app              → node dist/server.js
# docker run mi-app --version    → node --version
# docker run mi-app -e "console.log(1+1)" → node -e "console.log(1+1)"
```

### 4.4.5 Patrón Docker Entrypoint Script

El patrón más profesional: un script de entrada que hace setup de runtime (esperar
dependencias, migraciones, generación de configuración) y luego ejecuta la aplicación
real con `exec "$@"`.

```bash
#!/bin/sh
set -e

# docker-entrypoint.sh — patrón canónico

# 1. Setup de runtime (si es primera ejecución)
if [ ! -f /data/.initialized ]; then
    echo "Primera ejecución: inicializando..."
    python manage.py migrate --noinput
    python manage.py collectstatic --noinput
    touch /data/.initialized
fi

# 2. Esperar dependencias
echo "Esperando que PostgreSQL esté disponible..."
until pg_isready -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER"; do
    sleep 1
done

# 3. Ejecutar la aplicación (CMD) reemplazando este script
exec "$@"
```

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends postgresql-client \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

RUN mkdir -p /data && chown 1000:1000 /data

COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

EXPOSE 8000
USER 1000:1000
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["gunicorn", "mi_app.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

**¿Por qué `exec "$@"`?** `exec` reemplaza el proceso actual (el script) por el nuevo
proceso (la aplicación). Sin `exec`, el script seguiría siendo PID 1 y tu aplicación
no recibiría señales. Los argumentos `"$@"` expanden todos los argumentos pasados al
script (el CMD), preservando el quoting correctamente.

**Variantes del patrón:**

```bash
#!/bin/sh
# Variante 1: Entrypoint que permite modo "subcomando"
case "$1" in
    web)
        exec gunicorn mi_app.wsgi --bind 0.0.0.0:8000
        ;;
    worker)
        exec celery -A mi_app worker -l info
        ;;
    beat)
        exec celery -A mi_app beat -l info
        ;;
    shell)
        exec python manage.py shell
        ;;
    *)
        exec "$@"
        ;;
esac
```

```bash
#!/bin/sh
# Variante 2: Entrypoint con inicialización condicional por tipo de despliegue
if [ "$SERVICE_TYPE" = "migrate" ]; then
    exec python manage.py migrate --noinput
elif [ "$SERVICE_TYPE" = "collectstatic" ]; then
    exec python manage.py collectstatic --noinput
else
    exec "$@"
fi
```

### 4.4.6 Análisis Forense: Qué Comando se Está Ejecutando Realmente

Para inspeccionar qué comando y entrypoint tiene una imagen o un contenedor:

```bash
# Ver CMD y ENTRYPOINT definidos en la imagen
docker inspect mi-imagen \
  --format='CMD: {{.Config.Cmd}}'
docker inspect mi-imagen \
  --format='ENTRYPOINT: {{.Config.Entrypoint}}'

# Ver el comando completo que se ejecutó (incluye shell form desglosado)
docker inspect mi-contenedor \
  --format='Path: {{.Path}} Args: {{.Args}}'

# Ver todos los procesos dentro del contenedor
docker exec mi-contenedor ps aux

# Ver PID 1 específicamente
docker top mi-contenedor

# Inspeccionar el árbol de procesos con todas las señales
docker exec mi-contenedor cat /proc/1/status | grep -E "^(Name|Pid|Sig)"
```

---

## 4.5 WORKDIR — El Directorio de Trabajo

### 4.5.1 Qué Hace WORKDIR

`WORKDIR` establece el directorio de trabajo para todas las instrucciones que le
siguen: `RUN`, `CMD`, `ENTRYPOINT`, `COPY` y `ADD`.

```dockerfile
WORKDIR /app
```

Si el directorio no existe, **WORKDIR lo crea automáticamente**. Puedes usar
`WORKDIR` múltiples veces; las rutas relativas se resuelven respecto al `WORKDIR`
anterior.

```dockerfile
WORKDIR /app
WORKDIR src
# Equivale a /app/src

WORKDIR /otro/directorio
# Reinicia: /otro/directorio
```

### 4.5.2 Sin WORKDIR vs Con WORKDIR

**Sin WORKDIR (mala práctica):**

```dockerfile
FROM node:20-alpine
RUN mkdir -p /app && cd /app && npm ci && npm run build
COPY . /app/
RUN cd /app && npm ci && cd /app && npm run build
CMD ["node", "/app/dist/index.js"]
```

Cada `RUN` y `CMD` necesita rutas absolutas o `cd` explícito. Frágil y difícil
de mantener.

**Con WORKDIR (mejor práctica):**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
```

Todo ocurre relativo a `/app`. Si cambias la estructura de directorios, solo
cambias `WORKDIR`.

### 4.5.3 WORKDIR como Mejor Práctica de Seguridad

Usar `WORKDIR` en un directorio no-root ayuda a evitar escrituras accidentales en
directorios protegidos:

```dockerfile
FROM node:20-alpine
RUN addgroup -g 1000 appgroup \
    && adduser -u 1000 -G appgroup -s /bin/sh -D appuser

WORKDIR /home/appuser/app
COPY --chown=1000:1000 . .

USER 1000:1000
CMD ["node", "dist/index.js"]
```

---

## 4.6 ENV vs ARG — Build-Time vs Runtime

### 4.6.1 ENV: Variables de Entorno en Runtime

`ENV` establece variables de entorno que persisten **durante la construcción y en el
contenedor en ejecución**. Son accesibles en `RUN`, `CMD`, `ENTRYPOINT`, y desde
cualquier proceso dentro del contenedor.

```dockerfile
ENV NODE_ENV=production
ENV APP_HOME=/app
ENV PORT=8080
ENV DB_HOST=postgres
```

**Sintaxis con = (recomendada para una variable):**

```dockerfile
ENV MI_VAR=valor
```

**Sintaxis sin = (para múltiples variables):**

```dockerfile
ENV MI_VAR valor
ENV OTRA_VAR otro_valor
```

**Sintaxis compacta (una capa para múltiples variables):**

```dockerfile
ENV NODE_ENV=production \
    APP_HOME=/app \
    PORT=8080 \
    DB_HOST=postgres
```

**Uso en RUN:**

```dockerfile
ENV APP_VERSION=1.2.3
RUN echo "Building version $APP_VERSION" && \
    curl -o /tmp/app-${APP_VERSION}.tar.gz https://releases.example.com/app-${APP_VERSION}.tar.gz
```

**Sobrescritura en runtime:**

```bash
docker run -e NODE_ENV=development -e PORT=3000 mi-imagen
```

### 4.6.2 ARG: Variables Solo en Tiempo de Build

`ARG` define variables que solo existen durante el build. **No persisten en la
imagen final** (salvo que se usen en `ENV` u otras instrucciones que persisten).

```dockerfile
ARG VERSION=1.0.0
ARG DEBIAN_FRONTEND=noninteractive
```

**Pasar valores con --build-arg:**

```bash
docker build --build-arg VERSION=2.0.0 --build-arg NODE_ENV=staging -t mi-app .
```

**ARG con valor por defecto:**

```dockerfile
ARG VERSION=latest
# Si no se pasa --build-arg, usa "latest"
```

**ARG sin valor por defecto (obligatorio pasar):**

```dockerfile
ARG VERSION
# Requiere --build-arg VERSION=... o falla
```

### 4.6.3 Scope de ARG

**Antes de FROM (scope global, solo para FROM):**

```dockerfile
ARG IMAGE_TAG=20-alpine
FROM node:${IMAGE_TAG}
```

Este ARG está fuera de cualquier stage y solo puede usarse en `FROM`. No está
disponible dentro de los stages.

**Dentro de un stage:**

```dockerfile
FROM node:20-alpine
ARG APP_ENV=production
RUN echo "Building for $APP_ENV"
```

Este ARG solo existe en este stage. Cada stage tiene su propio scope de ARGs.

**ARG antes y dentro de FROM (requiere re-declaración):**

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine
ARG NODE_VERSION
RUN echo "Node version: $NODE_VERSION"
```

El ARG antes de `FROM` y dentro del stage son variables **diferentes**. Necesitas
re-declararlo dentro del stage para usarlo.

### 4.6.4 NUNCA Usar ARG para Secretos

Este es uno de los errores más graves y comunes en Dockerfiles:

**DEMOSTRACIÓN DEL PROBLEMA — El secreto queda expuesto:**

```dockerfile
FROM alpine:3.20
ARG SECRET_TOKEN
RUN echo "Token: $SECRET_TOKEN" > /app/token.txt && rm /app/token.txt
```

```bash
$ docker build --build-arg SECRET_TOKEN=super-secreto-12345 -t demo .
$ docker history demo --no-trunc
IMAGE                                             CREATED BY
sha256:fedcba9876543210...   /bin/sh -c echo "Token: super-secreto-12345" > /app/token.txt && rm /app/token.txt
```

El secreto está **para siempre** en el historial de la imagen. Cualquiera con acceso
al registro o a la imagen local puede extraerlo.

```bash
# Inspeccionar todas las capas y sus comandos
$ docker history demo
$ docker image inspect demo

# Con dive puedes incluso explorar el contenido de cada capa
$ dive demo
```

**La solución: BuildKit secrets (`--mount=type=secret`):**

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.20
RUN --mount=type=secret,id=token,dst=/tmp/token \
    TOKEN=$(cat /tmp/token) && echo "Usando token" && \
    echo "Procesado sin persistir"
```

```bash
$ export DOCKER_BUILDKIT=1
$ docker build --secret id=token,env=SECRET_TOKEN -t demo .
$ docker history demo   # El token NO aparece
```

### 4.6.5 ENV vs ARG: Cuándo Usar Cada Uno

| Necesidad | Usar |
|---|---|
| Configuración de runtime (DB_HOST, PORT, etc.) | `ENV` |
| Versión de dependencia para build | `ARG` |
| Modo de entorno (development/production) | `ARG` si solo afecta el build, `ENV` si afecta runtime |
| URLs de descarga durante build | `ARG` |
| Tokens, contraseñas, claves | **Ninguno** — usa BuildKit secrets |
| Flags de compilación | `ARG` |
| Configuración regional / timezone | `ENV` |

**Patrón común: pasar ARG a ENV para runtime:**

```dockerfile
ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION}
LABEL org.opencontainers.image.version=${APP_VERSION}
```

El ARG se usa durante el build, pero la variable persiste como ENV para que la app
pueda leer su versión en runtime.

### 4.6.6 Variables Predefinidas de BuildKit

BuildKit expone automáticamente variables predefinidas con información de la
plataforma de build:

```dockerfile
FROM alpine:3.20
ARG TARGETARCH
ARG TARGETOS
ARG BUILDPLATFORM
ARG BUILDARCH
ARG BUILDOS
ARG TARGETVARIANT

RUN echo "Compilando en $BUILDPLATFORM para $TARGETOS/$TARGETARCH"
```

```bash
$ docker build --platform linux/amd64 .
Compilando en linux/amd64 para linux/amd64

$ docker build --platform linux/arm64 .
Compilando en linux/amd64 para linux/arm64
```

**Tabla de variables predefinidas:**

| Variable | Descripción | Ejemplo |
|---|---|---|
| `TARGETARCH` | Arquitectura objetivo | `amd64`, `arm64`, `arm/v7` |
| `TARGETOS` | SO objetivo | `linux`, `windows` |
| `TARGETVARIANT` | Variante de arquitectura | `v7`, `v8` |
| `BUILDPLATFORM` | Plataforma del builder | `linux/amd64` |
| `BUILDARCH` | Arquitectura del builder | `amd64` |
| `BUILDOS` | SO del builder | `linux` |
| `HTTP_PROXY` | Proxy HTTP (si se configura) | `http://proxy:8080` |
| `HTTPS_PROXY` | Proxy HTTPS | `http://proxy:8080` |
| `FTP_PROXY` | Proxy FTP | `http://proxy:8080` |
| `NO_PROXY` | Excepciones de proxy | `localhost,127.0.0.1` |

### 4.6.7 ENV No Sustituye Configuración Externalizada

Un error común es meter toda la configuración en ENV. Si tu aplicación tiene 50
variables de entorno, algo está mal:

```dockerfile
# MAL: Dockerfile como repositorio de configuración
ENV DB_HOST=postgres
ENV DB_PORT=5432
ENV DB_USER=admin
ENV DB_PASS=secreto              # ¡PELIGRO!
ENV REDIS_HOST=redis
ENV REDIS_PORT=6379
ENV SMTP_HOST=smtp.example.com
ENV SMTP_PORT=587
ENV SMTP_USER=noreply@example.com
ENV SMTP_PASS=otro-secreto       # ¡PELIGRO!
# ... 40 ENV más
```

El Dockerfile debe ser el contrato de construcción, no el repositorio de
configuración. La configuración de runtime debe inyectarse externamente:

```bash
docker run \
  -e DB_HOST=postgres \
  -e DB_PORT=5432 \
  --env-file production.env \
  mi-app
```

O mejor aún, usando Docker Compose, Kubernetes ConfigMaps/Secrets, o HashiCorp Vault.

---

## 4.7 EXPOSE — Documentar Puertos

### 4.7.1 EXACTAMENTE Lo Que Hace (y No Hace) EXPOSE

`EXPOSE` **no publica puertos**. Es puramente documental. Sirve para:

1. Documentar qué puertos usa la aplicación, para que los operadores sepan qué
   puertos mapear.
2. Habilitar el flag `-P` (`--publish-all`), que mapea **todos** los puertos
   declarados con EXPOSE a puertos aleatorios del host.

```dockerfile
EXPOSE 8080
EXPOSE 8080/tcp
EXPOSE 8080/udp
EXPOSE 8080 8443
```

**Para publicar realmente un puerto, necesitas `-p`:**

```bash
# Esto SÍ publica el puerto
docker run -p 8080:8080 mi-app

# Esto mapea todos los EXPOSE a puertos aleatorios del host
docker run -P mi-app
# Equivalente a: docker run -p <host-aleatorio>:8080 mi-app
```

### 4.7.2 Protocolo

Por defecto, EXPOSE asume TCP. Para UDP debes especificarlo explícitamente:

```dockerfile
EXPOSE 8080/tcp
EXPOSE 53/udp
```

### 4.7.3 Mejores Prácticas

```dockerfile
# Expón los puertos que tu aplicación realmente usa
FROM nginx:1.25-alpine
EXPOSE 80   # HTTP
EXPOSE 443  # HTTPS

FROM node:20-alpine
EXPOSE 3000  # API
EXPOSE 9229  # Debugger (solo en imagen de desarrollo)
```

No expongas puertos de bases de datos en imágenes de aplicación web.
No expongas puertos que no usa tu aplicación.
## 4.4 CMD vs ENTRYPOINT — SECCIÓN CRÍTICA

Esta es, sin exagerar, la sección más importante después de elegir la imagen base.
La diferencia entre `CMD` y `ENTRYPOINT` y la elección entre *shell form* y *exec
form* determinan cómo tu contenedor recibe señales Unix, cómo se detiene, y qué
tan bien se integra con orquestadores como Kubernetes o Swarm.

### 4.4.1 CMD: El Comando por Defecto

`CMD` establece el comando (y argumentos) que se ejecutarán cuando el contenedor
arranque, **a menos que el usuario especifique otro comando** en `docker run`.

**Tres formas de CMD:**

```dockerfile
# 1. Exec form (RECOMENDADA)
CMD ["ejecutable", "param1", "param2"]

# 2. Shell form (EVITAR)
CMD ejecutable param1 param2

# 3. Parámetros para ENTRYPOINT (combinación)
CMD ["param1", "param2"]
```

**CMD en exec form:**

```dockerfile
FROM ubuntu:24.04
CMD ["echo", "Hola desde CMD"]
```

```bash
$ docker run mi-imagen
Hola desde CMD

$ docker run mi-imagen echo "Yo decido"
Yo decido
```

**CMD en shell form:**

```dockerfile
FROM ubuntu:24.04
CMD echo "Hola desde CMD"
```

Se ejecuta como: `/bin/sh -c "echo Hola desde CMD"`

### 4.4.2 ENTRYPOINT: El Comando Fijo

`ENTRYPOINT` define el ejecutable que será el **propósito mismo del contenedor**.
No puede ser sobrescrito por `docker run` (salvo con el flag `--entrypoint`).

Un contenedor con ENTRYPOINT define **qué es**: un proxy nginx, un servidor de
aplicaciones, un worker de colas. CMD complementa como argumentos por defecto.

```dockerfile
FROM nginx:1.25-alpine
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

```bash
# Ejecuta nginx -g 'daemon off;'
$ docker run mi-nginx

# Ejecuta nginx -t (solo test de configuración)
$ docker run mi-nginx -t

# CMD fue sobrescrito, ENTRYPOINT se mantuvo
```

### 4.4.3 El Problema de Señales Unix: Shell Form vs Exec Form

Esta es la razón técnica fundamental por la que **siempre debes usar exec form**
para CMD y ENTRYPOINT:

**Shell form — ROMPE las señales Unix:**

```dockerfile
FROM ubuntu:24.04
CMD ping localhost
```

¿Qué procesos se ejecutan realmente?

```
PID 1: /bin/sh -c "ping localhost"
PID 7: ping localhost
```

El problema: **PID 1 es el shell, no tu aplicación**. Cuando Docker envía SIGTERM
al contenedor (con `docker stop`), el kernel envía la señal a PID 1. Pero `/bin/sh`
**no reenvía señales a sus procesos hijos**. Resultado: el shell muere, pero `ping`
sigue ejecutándose como proceso huérfano. Docker espera 10 segundos (el grace period)
y luego envía SIGKILL, matando todo forzosamente.

**Demostración del problema:**

```bash
# Shell form - tarda 10 segundos en detenerse
$ time docker stop contenedor-shell
contenedor-shell
docker stop contenedor-shell  0.04s user 0.02s system 0% cpu 10.234 total

# Exec form - se detiene en milisegundos
$ time docker stop contenedor-exec
contenedor-exec
docker stop contenedor-exec  0.04s user 0.02s system 0% cpu 0.023 total
```

**Exec form — las señales llegan correctamente:**

```dockerfile
FROM ubuntu:24.04
CMD ["ping", "localhost"]
```

PID 1 es directamente `ping`. Cuando Docker envía SIGTERM, `ping` lo recibe y responde
inmediatamente. El contenedor se detiene en milisegundos.

**Tabla comparativa shell form vs exec form para CMD/ENTRYPOINT:**

| Característica | Shell form | Exec form |
|---|---|---|
| PID 1 | `/bin/sh` | Tu aplicación |
| Recibe señales Unix | NO | SÍ |
| `docker stop` | Lento (~10s) | Instantáneo |
| Variables de entorno | Expandidas | Sin expandir |
| Pipes, redirecciones | Funcionan | No funcionan |
| Shell features (&&, \|\|) | Funcionan | No funcionan |
| Recomendado para producción | NO | SÍ |

**Cómo ejecutar comandos complejos con exec form:**

Si necesitas pipes, variables o lógica de shell, encapsúlalos en un script:

```dockerfile
FROM ubuntu:24.04
COPY entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

```bash
#!/bin/sh
# entrypoint.sh
export APP_ENV="${APP_ENV:-production}"
echo "Iniciando en $APP_ENV"
exec ./mi-app --port "${PORT:-8080}"
```

Lo importante es `exec` al final: reemplaza el shell por tu aplicación, haciendo
que tu app sea PID 1 y reciba señales correctamente.

### 4.4.4 Combinación CMD + ENTRYPOINT

La combinación más poderosa: ENTRYPOINT define el ejecutable, CMD establece los
parámetros por defecto (que el usuario puede sobrescribir en `docker run`).

**Tabla de combinaciones:**

| Caso | Dockerfile | `docker run` sin args | `docker run` con args |
|---|---|---|---|
| Solo CMD | `CMD ["app"]` | `app` | Los args reemplazan CMD |
| Solo ENTRYPOINT | `ENTRYPOINT ["app"]` | `app` | `app` + args al final |
| ENTRYPOINT + CMD | `ENTRYPOINT ["app"]` `CMD ["--help"]` | `app --help` | `app` + args (CMD ignorado) |
| Shell CMD | `CMD app --port 80` | `/bin/sh -c "app --port 80"` | Reemplazado |
| Shell ENTRYPOINT | `ENTRYPOINT app` | `/bin/sh -c "app"` | Ignorado a menos que --entrypoint |

**Ejemplos prácticos:**

```dockerfile
# Nginx: ENTRYPOINT fijo, CMD como argumentos por defecto
FROM nginx:1.25-alpine
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]

# Test de configuración sin arrancar: docker run mi-nginx -t

# Python: ENTRYPOINT como wrapper de setup, CMD como app
FROM python:3.12-slim
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["gunicorn", "mi_app.wsgi:application", "--bind", "0.0.0.0:8000"]

# APP como ENTRYPOINT con subcomando por defecto
FROM node:20-alpine
ENTRYPOINT ["node"]
CMD ["dist/server.js"]

# docker run mi-app              → node dist/server.js
# docker run mi-app --version    → node --version
# docker run mi-app -e "console.log(1+1)" → node -e "console.log(1+1)"
```

### 4.4.5 Patrón Docker Entrypoint Script

El patrón más profesional: un script de entrada que hace setup de runtime (esperar
dependencias, migraciones, generación de configuración) y luego ejecuta la aplicación
real con `exec "$@"`.

```bash
#!/bin/sh
set -e

# docker-entrypoint.sh — patrón canónico

# 1. Setup de runtime (si es primera ejecución)
if [ ! -f /data/.initialized ]; then
    echo "Primera ejecución: inicializando..."
    python manage.py migrate --noinput
    python manage.py collectstatic --noinput
    touch /data/.initialized
fi

# 2. Esperar dependencias
echo "Esperando que PostgreSQL esté disponible..."
until pg_isready -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER"; do
    sleep 1
done

# 3. Ejecutar la aplicación (CMD) reemplazando este script
exec "$@"
```

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends postgresql-client \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

RUN mkdir -p /data && chown 1000:1000 /data

COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

EXPOSE 8000
USER 1000:1000
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["gunicorn", "mi_app.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

**¿Por qué `exec "$@"`?** `exec` reemplaza el proceso actual (el script) por el nuevo
proceso (la aplicación). Sin `exec`, el script seguiría siendo PID 1 y tu aplicación
no recibiría señales. Los argumentos `"$@"` expanden todos los argumentos pasados al
script (el CMD), preservando el quoting correctamente.

**Variantes del patrón:**

```bash
#!/bin/sh
# Variante 1: Entrypoint que permite modo "subcomando"
case "$1" in
    web)
        exec gunicorn mi_app.wsgi --bind 0.0.0.0:8000
        ;;
    worker)
        exec celery -A mi_app worker -l info
        ;;
    beat)
        exec celery -A mi_app beat -l info
        ;;
    shell)
        exec python manage.py shell
        ;;
    *)
        exec "$@"
        ;;
esac
```

```bash
#!/bin/sh
# Variante 2: Entrypoint con inicialización condicional por tipo de despliegue
if [ "$SERVICE_TYPE" = "migrate" ]; then
    exec python manage.py migrate --noinput
elif [ "$SERVICE_TYPE" = "collectstatic" ]; then
    exec python manage.py collectstatic --noinput
else
    exec "$@"
fi
```

### 4.4.6 Análisis Forense: Qué Comando se Está Ejecutando Realmente

Para inspeccionar qué comando y entrypoint tiene una imagen o un contenedor:

```bash
# Ver CMD y ENTRYPOINT definidos en la imagen
docker inspect mi-imagen \
  --format='CMD: {{.Config.Cmd}}'
docker inspect mi-imagen \
  --format='ENTRYPOINT: {{.Config.Entrypoint}}'

# Ver el comando completo que se ejecutó (incluye shell form desglosado)
docker inspect mi-contenedor \
  --format='Path: {{.Path}} Args: {{.Args}}'

# Ver todos los procesos dentro del contenedor
docker exec mi-contenedor ps aux

# Ver PID 1 específicamente
docker top mi-contenedor

# Inspeccionar el árbol de procesos con todas las señales
docker exec mi-contenedor cat /proc/1/status | grep -E "^(Name|Pid|Sig)"
```

---

## 4.5 WORKDIR — El Directorio de Trabajo

### 4.5.1 Qué Hace WORKDIR

`WORKDIR` establece el directorio de trabajo para todas las instrucciones que le
siguen: `RUN`, `CMD`, `ENTRYPOINT`, `COPY` y `ADD`.

```dockerfile
WORKDIR /app
```

Si el directorio no existe, **WORKDIR lo crea automáticamente**. Puedes usar
`WORKDIR` múltiples veces; las rutas relativas se resuelven respecto al `WORKDIR`
anterior.

```dockerfile
WORKDIR /app
WORKDIR src
# Equivale a /app/src

WORKDIR /otro/directorio
# Reinicia: /otro/directorio
```

### 4.5.2 Sin WORKDIR vs Con WORKDIR

**Sin WORKDIR (mala práctica):**

```dockerfile
FROM node:20-alpine
RUN mkdir -p /app && cd /app && npm ci && npm run build
COPY . /app/
RUN cd /app && npm ci && cd /app && npm run build
CMD ["node", "/app/dist/index.js"]
```

Cada `RUN` y `CMD` necesita rutas absolutas o `cd` explícito. Frágil y difícil
de mantener.

**Con WORKDIR (mejor práctica):**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
```

Todo ocurre relativo a `/app`. Si cambias la estructura de directorios, solo
cambias `WORKDIR`.

### 4.5.3 WORKDIR como Mejor Práctica de Seguridad

Usar `WORKDIR` en un directorio no-root ayuda a evitar escrituras accidentales en
directorios protegidos:

```dockerfile
FROM node:20-alpine
RUN addgroup -g 1000 appgroup \
    && adduser -u 1000 -G appgroup -s /bin/sh -D appuser

WORKDIR /home/appuser/app
COPY --chown=1000:1000 . .

USER 1000:1000
CMD ["node", "dist/index.js"]
```

---

## 4.6 ENV vs ARG — Build-Time vs Runtime

### 4.6.1 ENV: Variables de Entorno en Runtime

`ENV` establece variables de entorno que persisten **durante la construcción y en el
contenedor en ejecución**. Son accesibles en `RUN`, `CMD`, `ENTRYPOINT`, y desde
cualquier proceso dentro del contenedor.

```dockerfile
ENV NODE_ENV=production
ENV APP_HOME=/app
ENV PORT=8080
ENV DB_HOST=postgres
```

**Sintaxis con = (recomendada para una variable):**

```dockerfile
ENV MI_VAR=valor
```

**Sintaxis sin = (para múltiples variables):**

```dockerfile
ENV MI_VAR valor
ENV OTRA_VAR otro_valor
```

**Sintaxis compacta (una capa para múltiples variables):**

```dockerfile
ENV NODE_ENV=production \
    APP_HOME=/app \
    PORT=8080 \
    DB_HOST=postgres
```

**Uso en RUN:**

```dockerfile
ENV APP_VERSION=1.2.3
RUN echo "Building version $APP_VERSION" && \
    curl -o /tmp/app-${APP_VERSION}.tar.gz https://releases.example.com/app-${APP_VERSION}.tar.gz
```

**Sobrescritura en runtime:**

```bash
docker run -e NODE_ENV=development -e PORT=3000 mi-imagen
```

### 4.6.2 ARG: Variables Solo en Tiempo de Build

`ARG` define variables que solo existen durante el build. **No persisten en la
imagen final** (salvo que se usen en `ENV` u otras instrucciones que persisten).

```dockerfile
ARG VERSION=1.0.0
ARG DEBIAN_FRONTEND=noninteractive
```

**Pasar valores con --build-arg:**

```bash
docker build --build-arg VERSION=2.0.0 --build-arg NODE_ENV=staging -t mi-app .
```

**ARG con valor por defecto:**

```dockerfile
ARG VERSION=latest
# Si no se pasa --build-arg, usa "latest"
```

**ARG sin valor por defecto (obligatorio pasar):**

```dockerfile
ARG VERSION
# Requiere --build-arg VERSION=... o falla
```

### 4.6.3 Scope de ARG

**Antes de FROM (scope global, solo para FROM):**

```dockerfile
ARG IMAGE_TAG=20-alpine
FROM node:${IMAGE_TAG}
```

Este ARG está fuera de cualquier stage y solo puede usarse en `FROM`. No está
disponible dentro de los stages.

**Dentro de un stage:**

```dockerfile
FROM node:20-alpine
ARG APP_ENV=production
RUN echo "Building for $APP_ENV"
```

Este ARG solo existe en este stage. Cada stage tiene su propio scope de ARGs.

**ARG antes y dentro de FROM (requiere re-declaración):**

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine
ARG NODE_VERSION
RUN echo "Node version: $NODE_VERSION"
```

El ARG antes de `FROM` y dentro del stage son variables **diferentes**. Necesitas
re-declararlo dentro del stage para usarlo.

### 4.6.4 NUNCA Usar ARG para Secretos

Este es uno de los errores más graves y comunes en Dockerfiles:

**DEMOSTRACIÓN DEL PROBLEMA — El secreto queda expuesto:**

```dockerfile
FROM alpine:3.20
ARG SECRET_TOKEN
RUN echo "Token: $SECRET_TOKEN" > /app/token.txt && rm /app/token.txt
```

```bash
$ docker build --build-arg SECRET_TOKEN=super-secreto-12345 -t demo .
$ docker history demo --no-trunc
IMAGE                                             CREATED BY
sha256:fedcba9876543210...   /bin/sh -c echo "Token: super-secreto-12345" > /app/token.txt && rm /app/token.txt
```

El secreto está **para siempre** en el historial de la imagen. Cualquiera con acceso
al registro o a la imagen local puede extraerlo.

```bash
# Inspeccionar todas las capas y sus comandos
$ docker history demo
$ docker image inspect demo

# Con dive puedes incluso explorar el contenido de cada capa
$ dive demo
```

**La solución: BuildKit secrets (`--mount=type=secret`):**

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.20
RUN --mount=type=secret,id=token,dst=/tmp/token \
    TOKEN=$(cat /tmp/token) && echo "Usando token" && \
    echo "Procesado sin persistir"
```

```bash
$ export DOCKER_BUILDKIT=1
$ docker build --secret id=token,env=SECRET_TOKEN -t demo .
$ docker history demo   # El token NO aparece
```

### 4.6.5 ENV vs ARG: Cuándo Usar Cada Uno

| Necesidad | Usar |
|---|---|
| Configuración de runtime (DB_HOST, PORT, etc.) | `ENV` |
| Versión de dependencia para build | `ARG` |
| Modo de entorno (development/production) | `ARG` si solo afecta el build, `ENV` si afecta runtime |
| URLs de descarga durante build | `ARG` |
| Tokens, contraseñas, claves | **Ninguno** — usa BuildKit secrets |
| Flags de compilación | `ARG` |
| Configuración regional / timezone | `ENV` |

**Patrón común: pasar ARG a ENV para runtime:**

```dockerfile
ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION}
LABEL org.opencontainers.image.version=${APP_VERSION}
```

El ARG se usa durante el build, pero la variable persiste como ENV para que la app
pueda leer su versión en runtime.

### 4.6.6 Variables Predefinidas de BuildKit

BuildKit expone automáticamente variables predefinidas con información de la
plataforma de build:

```dockerfile
FROM alpine:3.20
ARG TARGETARCH
ARG TARGETOS
ARG BUILDPLATFORM
ARG BUILDARCH
ARG BUILDOS
ARG TARGETVARIANT

RUN echo "Compilando en $BUILDPLATFORM para $TARGETOS/$TARGETARCH"
```

```bash
$ docker build --platform linux/amd64 .
Compilando en linux/amd64 para linux/amd64

$ docker build --platform linux/arm64 .
Compilando en linux/amd64 para linux/arm64
```

**Tabla de variables predefinidas:**

| Variable | Descripción | Ejemplo |
|---|---|---|
| `TARGETARCH` | Arquitectura objetivo | `amd64`, `arm64`, `arm/v7` |
| `TARGETOS` | SO objetivo | `linux`, `windows` |
| `TARGETVARIANT` | Variante de arquitectura | `v7`, `v8` |
| `BUILDPLATFORM` | Plataforma del builder | `linux/amd64` |
| `BUILDARCH` | Arquitectura del builder | `amd64` |
| `BUILDOS` | SO del builder | `linux` |
| `HTTP_PROXY` | Proxy HTTP (si se configura) | `http://proxy:8080` |
| `HTTPS_PROXY` | Proxy HTTPS | `http://proxy:8080` |
| `FTP_PROXY` | Proxy FTP | `http://proxy:8080` |
| `NO_PROXY` | Excepciones de proxy | `localhost,127.0.0.1` |

### 4.6.7 ENV No Sustituye Configuración Externalizada

Un error común es meter toda la configuración en ENV. Si tu aplicación tiene 50
variables de entorno, algo está mal:

```dockerfile
# MAL: Dockerfile como repositorio de configuración
ENV DB_HOST=postgres
ENV DB_PORT=5432
ENV DB_USER=admin
ENV DB_PASS=secreto              # ¡PELIGRO!
ENV REDIS_HOST=redis
ENV REDIS_PORT=6379
ENV SMTP_HOST=smtp.example.com
ENV SMTP_PORT=587
ENV SMTP_USER=noreply@example.com
ENV SMTP_PASS=otro-secreto       # ¡PELIGRO!
# ... 40 ENV más
```

El Dockerfile debe ser el contrato de construcción, no el repositorio de
configuración. La configuración de runtime debe inyectarse externamente:

```bash
docker run \
  -e DB_HOST=postgres \
  -e DB_PORT=5432 \
  --env-file production.env \
  mi-app
```

O mejor aún, usando Docker Compose, Kubernetes ConfigMaps/Secrets, o HashiCorp Vault.

---

## 4.7 EXPOSE — Documentar Puertos

### 4.7.1 EXACTAMENTE Lo Que Hace (y No Hace) EXPOSE

`EXPOSE` **no publica puertos**. Es puramente documental. Sirve para:

1. Documentar qué puertos usa la aplicación, para que los operadores sepan qué
   puertos mapear.
2. Habilitar el flag `-P` (`--publish-all`), que mapea **todos** los puertos
   declarados con EXPOSE a puertos aleatorios del host.

```dockerfile
EXPOSE 8080
EXPOSE 8080/tcp
EXPOSE 8080/udp
EXPOSE 8080 8443
```

**Para publicar realmente un puerto, necesitas `-p`:**

```bash
# Esto SÍ publica el puerto
docker run -p 8080:8080 mi-app

# Esto mapea todos los EXPOSE a puertos aleatorios del host
docker run -P mi-app
# Equivalente a: docker run -p <host-aleatorio>:8080 mi-app
```

### 4.7.2 Protocolo

Por defecto, EXPOSE asume TCP. Para UDP debes especificarlo explícitamente:

```dockerfile
EXPOSE 8080/tcp
EXPOSE 53/udp
```

### 4.7.3 Mejores Prácticas

```dockerfile
# Expón los puertos que tu aplicación realmente usa
FROM nginx:1.25-alpine
EXPOSE 80   # HTTP
EXPOSE 443  # HTTPS

FROM node:20-alpine
EXPOSE 3000  # API
EXPOSE 9229  # Debugger (solo en imagen de desarrollo)
```

No expongas puertos de bases de datos en imágenes de aplicación web.
No expongas puertos que no usa tu aplicación.

---

## 4.8 VOLUME — Volúmenes Anónimos

### 4.8.1 Qué Hace VOLUME

`VOLUME` crea un punto de montaje en el contenedor y marca ese directorio como
**externo al contenedor**. Los datos en ese directorio:

1. No se incluyen en la imagen (la capa se crea vacía, los datos existentes se
   copian a la ubicación del volumen en el host).
2. Persisten más allá de la vida del contenedor.
3. Pueden compartirse entre contenedores con `--volumes-from`.

```dockerfile
VOLUME /data
VOLUME /var/lib/mysql
VOLUME ["/var/log", "/var/lib/app"]
```

### 4.8.2 VOLUME en Dockerfile vs -v en docker run

**VOLUME en Dockerfile:**

```dockerfile
FROM postgres:16
VOLUME /var/lib/postgresql/data
```

Esto crea un volumen **anónimo** cada vez que ejecutas un contenedor. Docker genera
un ID aleatorio para el volumen:

```bash
$ docker run -d --name pg postgres:16
$ docker volume ls
DRIVER    VOLUME NAME
local     7a3b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
```

Si eliminas el contenedor, el volumen anónimo **no se elimina automáticamente**
(a menos que uses `--rm` o `-v`).

**-v en docker run:**

```bash
# Named volume (recomendado)
docker run -v pgdata:/var/lib/postgresql/data postgres:16

# Bind mount
docker run -v /host/path:/var/lib/postgresql/data postgres:16
```

### 4.8.3 El Anti-Patrón: VOLUME para node_modules

Un error clásico es usar VOLUME para directorios de dependencias:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
VOLUME /app/node_modules    # ¡ERROR!
COPY . .
CMD ["node", "dist/index.js"]
```

**El problema**: VOLUME monta un directorio externo encima de `/app/node_modules`,
ocultando las dependencias instaladas. El contenedor no tendrá las dependencias que
instalaste en el build, porque el volumen vacío las oculta.

**Alternativa correcta:** No uses VOLUME en el Dockerfile para dependencias. Usa
volúmenes en `docker run` o `docker-compose.yml` para desarrollo:

```yaml
# docker-compose.yml
services:
  app:
    build: .
    volumes:
      # Solo en desarrollo: monta código local y node_modules anónimo
      - .:/app
      - /app/node_modules  # Volumen anónimo que NO oculta las dependencias instaladas
```

El volumen anónimo declarado en Compose (sin parte izquierda) crea un volumen que
preserva el contenido de `/app/node_modules` instalado por la imagen, evitando
que el bind mount `.:/app` lo sobrescriba.

---

## 4.9 USER — Seguridad Imprescindible

### 4.9.1 El Problema de Root

Por defecto, **todos los contenedores Docker se ejecutan como root** (UID 0). Esto
significa que:

1. Si un atacante escapa del proceso de la aplicación, tiene acceso root al sistema
   de archivos del contenedor.
2. En configuraciones inseguras (con `--privileged` o sin user namespace remapping),
   root en el contenedor == root en el host.
3. Los archivos creados por la aplicación pertenecen a root, complicando permisos.
4. Incumples el principio de mínimo privilegio.

### 4.9.2 USER: Cambiar de Usuario

```dockerfile
# Por nombre (requiere que el usuario exista en /etc/passwd)
USER node

# Por UID/GID numéricos (más portable)
USER 1000:1000

# Solo UID (GID por defecto = UID)
USER 1000
```

### 4.9.3 Crear un Usuario y Cambiar a Él

**Alpine:**

```dockerfile
FROM alpine:3.20
RUN addgroup -g 1000 appgroup \
    && adduser -u 1000 -G appgroup -s /bin/sh -D appuser
USER 1000:1000
WORKDIR /home/appuser
COPY --chown=1000:1000 . .
```

**Debian/Ubuntu:**

```dockerfile
FROM debian:stable-slim
RUN groupadd -r -g 1000 appgroup \
    && useradd -r -u 1000 -g appgroup -m -s /bin/bash appuser
USER 1000:1000
WORKDIR /home/appuser
COPY --chown=1000:1000 . .
```

**Imagen con usuario predefinido (node, nginx, etc.):**

```dockerfile
FROM node:20-alpine
# node ya tiene UID/GID 1000 definidos
USER node
WORKDIR /home/node/app
COPY --chown=node:node . .
```

### 4.9.4 El Orden Importa

```dockerfile
# CORRECTO: crear usuario → copiar → cambiar permisos → cambiar a usuario
FROM alpine:3.20
RUN addgroup -g 1000 appgroup \
    && adduser -u 1000 -G appgroup -s /bin/sh -D appuser
RUN mkdir -p /app/data && chown -R 1000:1000 /app
COPY --chown=1000:1000 . /app
WORKDIR /app
USER 1000:1000
CMD ["./mi-app"]
```

```dockerfile
# INCORRECTO: USER antes de COPY sin --chown → archivos pertenecen a root
# y appuser no puede modificarlos
FROM alpine:3.20
RUN addgroup -g 1000 appgroup \
    && adduser -u 1000 -G appgroup -s /bin/sh -D appuser
USER 1000:1000
COPY . /app    # Archivos propiedad de root
               # Pero appuser no puede escribir en ellos
CMD ["./mi-app"]
```

### 4.9.5 Rootless Containers vs Usuario No-Root

**Usuario no-root en el contenedor:**

El daemon se ejecuta como root. El contenedor tiene su propio root (UID 0
mapeado al root del namespace). Dentro del contenedor, cambias a UID 1000. Si hay
un breakout del contenedor, el proceso en el host tiene el UID 1000 (no-root).

**Rootless Docker:**

El daemon mismo se ejecuta como usuario no-root en el host. Todos los contenedores
usan user namespace remapping: root en el contenedor se mapea a un UID no-root en
el host. Esto es más seguro pero tiene limitaciones (no puedes exponer puertos
privilegiados < 1024, ciertos storage drivers no funcionan).

```bash
# Verificar si tu contenedor corre como root
docker exec mi-contenedor whoami
# root ← MAL

docker exec mi-contenedor whoami
# node ← BIEN
```

---

## 4.10 HEALTHCHECK — Contenedores Autoconscientes

### 4.10.1 El Problema Sin HEALTHCHECK

Por defecto, Docker solo sabe si el proceso principal (PID 1) está vivo. No sabe
si la aplicación está realmente funcionando. Un contenedor puede estar "running"
pero:
- El servidor web escucha pero devuelve 500.
- La base de datos acepta conexiones pero no responde queries.
- La API está en deadlock.

Sin HEALTHCHECK, los orquestadores como Swarm y Kubernetes no pueden tomar
decisiones informadas sobre reinicios o routing.

### 4.10.2 Sintaxis

```dockerfile
HEALTHCHECK [OPCIONES] CMD <comando>

OPCIONES:
  --interval=DURATION  (default: 30s)    # Cada cuánto ejecutar el check
  --timeout=DURATION   (default: 30s)    # Tiempo máximo del check
  --start-period=DURATION (default: 0s)  # Tiempo de gracia al arrancar
  --start-interval=DURATION (default: 5s) # Intervalo durante start period
  --retries=N          (default: 3)      # Fallos consecutivos → unhealthy
```

El comando debe devolver:
- **0**: healthy (saludable)
- **1**: unhealthy (no saludable)
- **2**: reserved (no usar)

### 4.10.3 Estados de Salud

```
starting   → período de gracia (start-period), no se penaliza
healthy    → el check devuelve 0
unhealthy  → el check devuelve 1 tres veces seguidas (según --retries)
```

### 4.10.4 Ejemplos por Aplicación

**Nginx:**

```dockerfile
FROM nginx:1.25-alpine
HEALTHCHECK --interval=15s --timeout=3s --retries=2 \
  CMD curl -f http://localhost/ || exit 1
```

**Node.js:**

```dockerfile
FROM node:20-alpine
# Asegúrate de que tu app tenga un endpoint /health
# que devuelva 200 si la app está bien
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
```

**MySQL:**

```dockerfile
FROM mysql:8.4
HEALTHCHECK --interval=30s --timeout=10s --retries=5 \
  CMD mysqladmin ping -h localhost -u root -p${MYSQL_ROOT_PASSWORD} || exit 1
```

**Redis:**

```dockerfile
FROM redis:7.2-alpine
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD redis-cli ping || exit 1
```

**PostgreSQL:**

```dockerfile
FROM postgres:16-alpine
HEALTHCHECK --interval=15s --timeout=5s --retries=3 \
  CMD pg_isready -U ${POSTGRES_USER:-postgres} || exit 1
```

**Aplicación Go:**

```dockerfile
FROM gcr.io/distroless/static
COPY mi-app /
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD ["/mi-app", "health"] || exit 1
# Tu app Go debe implementar un subcomando 'health' que haga la verificación
```

**Python/Django:**

```dockerfile
FROM python:3.12-slim
# ... setup de la app ...
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD python manage.py check --deploy || exit 1
```

### 4.10.5 Impacto en Orquestación

**Docker Compose:**

```yaml
services:
  app:
    image: mi-app
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
```

El `depends_on` con `condition: service_healthy` espera a que PostgreSQL esté
healthy antes de iniciar la app. Sin HEALTHCHECK, `depends_on` solo espera a
que el contenedor esté "started", no a que el servicio esté listo.

**Docker Swarm:**

```yaml
services:
  web:
    image: mi-app
    deploy:
      restart_policy:
        condition: on-failure
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## 4.11 Otras Instrucciones

### 4.11.1 ONBUILD — Triggers para Imágenes Base

`ONBUILD` define instrucciones que se ejecutarán cuando la imagen actual se use como
base para otra imagen (en el `FROM` de otro Dockerfile). Es un mecanismo de triggers
diferidos.

```dockerfile
# Dockerfile de la imagen base
FROM node:20-alpine
ONBUILD COPY package*.json /app/
ONBUILD RUN npm ci
```

```dockerfile
# Dockerfile de la imagen hija (que hereda de la base)
FROM mi-imagen-base:latest
COPY . /app
# ONBUILD ya ejecutó COPY package*.json y RUN npm ci
# No necesitas repetir esas instrucciones
```

**Cuándo usar ONBUILD:**

- Imágenes base estandarizadas para equipos: todas las apps deben copiar
  `package.json`, ejecutar `npm ci`, etc. ONBUILD lo automatiza.
- Frameworks que necesitan estructura de proyecto conocida.

**El peligro de ONBUILD:**

```
1. Oscurece el comportamiento: el desarrollador no ve en su Dockerfile
   qué instrucciones se están ejecutando realmente.

2. Acoplamiento frágil: si la imagen base cambia sus ONBUILD, las imágenes
   hijas se rompen silenciosamente en el próximo build.

3. Debugging difícil: docker build no muestra las instrucciones ONBUILD
   como parte del output normal.

4. Abandonado por la comunidad: ONBUILD fue popular en 2016-2018. Hoy se
   prefiere documentar las instrucciones explícitamente y usar templates o
   generadores de Dockerfiles.
```

**Recomendación:** Evita ONBUILD. Es preferible un Dockerfile explícito aunque
sea más largo, o usar plantillas (Jinja2, Go templates) para generar Dockerfiles.

Si decides usar ONBUILD, documéntalo claramente:

```
# ADVERTENCIA: Esta imagen usa ONBUILD.
# Se ejecutarán automáticamente:
#   1. ONBUILD COPY package*.json /app/
#   2. ONBUILD RUN npm ci
```

### 4.11.2 STOPSIGNAL — Señal de Parada Personalizada

Define qué señal se envía al proceso principal cuando el contenedor debe detenerse.

```dockerfile
STOPSIGNAL SIGTERM  # (por defecto)
STOPSIGNAL SIGQUIT
STOPSIGNAL SIGINT
STOPSIGNAL 9        # Número de señal
```

**Casos de uso:**

```dockerfile
# Nginx usa SIGQUIT para graceful shutdown
FROM nginx:1.25-alpine
STOPSIGNAL SIGQUIT

# Apache HTTPD usa SIGWINCH para graceful stop
FROM httpd:2.4
STOPSIGNAL SIGWINCH

# Algunas aplicaciones responden a SIGINT pero ignoran SIGTERM
STOPSIGNAL SIGINT
```

La mayoría de las aplicaciones responden correctamente a SIGTERM (por defecto).
Solo cambia `STOPSIGNAL` si sabes que tu aplicación espera una señal diferente.

### 4.11.3 SHELL — Shell por Defecto

Define qué shell se usa para el *shell form* de `RUN`, `CMD` y `ENTRYPOINT`.

```dockerfile
# Por defecto en Linux
SHELL ["/bin/sh", "-c"]

# Cambiar a bash
SHELL ["/bin/bash", "-c"]

# Cambiar a PowerShell (Windows)
SHELL ["powershell", "-Command"]

# Cambiar a un shell con opciones estrictas
SHELL ["/bin/sh", "-eux", "-o", "pipefail", "-c"]
```

**Ejemplo práctico:**

```dockerfile
FROM alpine:3.20
SHELL ["/bin/sh", "-e", "-c"]  # -e: falla si cualquier comando falla
RUN apk add --no-cache curl    # Si falla, el build se detiene
```

**Recomendación:** `SHELL ["/bin/sh", "-c"]` (default) es suficiente para la
mayoría de los casos. Si necesitas bash específicamente, considera si realmente
necesitas bash o si tu script puede funcionar con POSIX sh.

### 4.11.4 LABEL — Metadatos OCI

Las labels son pares clave-valor que añaden metadatos a la imagen. Son esenciales
para cumplimiento, catalogación y automatización.

**Labels OCI estándar (recomendadas):**

```dockerfile
LABEL org.opencontainers.image.title="Mi Aplicación"
LABEL org.opencontainers.image.description="Servicio de API REST para gestión"
LABEL org.opencontainers.image.version="1.2.3"
LABEL org.opencontainers.image.authors="devops@empresa.com"
LABEL org.opencontainers.image.url="https://github.com/empresa/mi-app"
LABEL org.opencontainers.image.documentation="https://docs.empresa.com/mi-app"
LABEL org.opencontainers.image.source="https://github.com/empresa/mi-app.git"
LABEL org.opencontainers.image.vendor="Empresa S.A."
LABEL org.opencontainers.image.licenses="MIT"
LABEL org.opencontainers.image.created="2024-05-20T10:30:00Z"
LABEL org.opencontainers.image.revision="a1b2c3d4e5f6"
```

**Todas en una capa:**

```dockerfile
LABEL org.opencontainers.image.title="Mi App" \
      org.opencontainers.image.description="API REST" \
      org.opencontainers.image.version="1.2.3" \
      org.opencontainers.image.authors="devops@empresa.com" \
      org.opencontainers.image.source="https://github.com/empresa/mi-app" \
      org.opencontainers.image.vendor="Empresa S.A." \
      org.opencontainers.image.licenses="MIT"
```

Las labels se pueden usar para filtrar, catalogar y automatizar:

```bash
# Ver todas las labels de una imagen
docker inspect mi-imagen --format='{{json .Config.Labels}}' | jq

# Filtrar imágenes por label
docker images --filter "label=org.opencontainers.image.vendor=Empresa S.A."
```

---

## 4.12 Multi-Stage Builds — SECCIÓN EXTENSA

### 4.12.1 El Concepto

Los builds multi-stage resuelven el problema fundamental: **necesitas herramientas
pesadas para compilar, pero no quieres que esas herramientas lleguen a producción**.

Antes de multi-stage (introducido en Docker 17.05, 2017), tenías tres opciones,
todas malas:

1. **Builder pattern con scripts externos**: compilar fuera de Docker, luego copiar
   el binario con COPY. Rompe el principio de "build inside Docker".

2. **Dos Dockerfiles**: uno para compilar (`Dockerfile.build`) y otro para runtime.
   Necesita scripts de orquestación.

3. **Un solo Dockerfile con todo**: compilar y desplegar en la misma imagen. Resultado:
   imágenes de 900MB con SDKs, toolchains y código fuente en producción.

Multi-stage unifica todo en un solo Dockerfile:

```
ETAPA 1 (builder): SDK + dependencias + compilación → binario/artefactos
ETAPA 2 (runtime): solo el binario/artefacto + runtime mínimo
```

**La etapa 1 se descarta**. Solo la última etapa llega al registro.

### 4.12.2 Ejemplo Completo: Go → Binario Estático → Scratch

El caso canónico de multi-stage: Go compila a binario estático, scratch es una
imagen vacía. Resultado: imagen de ~5-10MB con SOLO tu binario.

```dockerfile
FROM golang:1.22-alpine AS builder

# Instalar herramientas de build si es necesario
RUN apk add --no-cache git ca-certificates

WORKDIR /app

# Cache de dependencias (capa que solo cambia si go.mod/go.sum cambia)
COPY go.mod go.sum ./
RUN go mod download

# Compilación (capa que solo cambia si el código fuente cambia)
COPY . .
RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
      -ldflags="-w -s" \
      -trimpath \
      -o /bin/mi-app \
      ./cmd/mi-app

# ----------------------------------------------------------------

FROM scratch

# Copiar certificados CA (necesarios para HTTPS)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copiar el binario
COPY --from=builder /bin/mi-app /mi-app

# Si tu app necesita zona horaria
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

EXPOSE 8080
ENTRYPOINT ["/mi-app"]
```

**¿Qué hacen los flags de compilación?**

- `CGO_ENABLED=0`: deshabilita CGO para crear un binario completamente estático
  (sin dependencias de libc ni bibliotecas compartidas).

- `-ldflags="-w -s"`:
  - `-w`: elimina la tabla de símbolos DWARF (información de debugging).
  - `-s`: elimina la tabla de símbolos (strip). Reduce el binario ~30%.

- `-trimpath`: elimina rutas absolutas del sistema de archivos del binario compilado.
  Mejora la reproducibilidad.

**Tamaño del resultado:**

```
Imagen si usas golang:1.22 como base:   ~800 MB
Imagen con multi-stage → scratch:       ~8 MB
Reducción:                              99%
```

### 4.12.3 Ejemplo Completo: Node.js / TypeScript

El caso más común en desarrollo web: compilar TypeScript a JavaScript, desplegar
solo los archivos compilados en una imagen slim.

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# ----------------------------------------------------------------

FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# ----------------------------------------------------------------

FROM node:20-alpine AS runtime
WORKDIR /app

# Crear usuario no-root
RUN addgroup -g 1000 appgroup \
    && adduser -u 1000 -G appgroup -s /bin/sh -D appuser

# Copiar solo dependencias de producción del stage deps
COPY --from=deps --chown=1000:1000 /app/node_modules ./node_modules

# Copiar solo los archivos compilados
COPY --from=builder --chown=1000:1000 /app/dist ./dist

# Copiar package.json (necesario para algunas configuraciones de runtime)
COPY --chown=1000:1000 package.json ./

# Archivos de configuración de producción
COPY --chown=1000:1000 config/production.json ./config/

USER 1000:1000

ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

**Notas importantes sobre este ejemplo:**

1. **Tres stages**: `deps`, `builder`, `runtime`. El stage `deps` instala solo
   dependencias de producción con `--omit=dev`. El stage `builder` instala todas
   las dependencias (incluyendo TypeScript, ESLint, etc.) para compilar. El stage
   `runtime` solo recibe `node_modules` de producción y los archivos compilados.

2. **Optimización por capas**: `COPY package.json package-lock.json` antes de `npm ci`
   permite que Docker cachee las dependencias. Solo se reinstalan si cambian los
   archivos de configuración de npm.

3. **Usuario no-root**: creado en el stage runtime, después de todas las operaciones
   que requieren root.

4. **HEALTHCHECK**: con endpoint `/health` que la aplicación debe implementar.

### 4.12.4 Ejemplo Completo: Java (Spring Boot / Maven)

Java tiene fama de imágenes pesadas. Con multi-stage podemos pasar de una imagen
de 500MB+ a una de ~150MB, usando una JRE slim o distroless.

```dockerfile
FROM maven:3.9-eclipse-temurin-21-alpine AS builder
WORKDIR /app

# Cache de dependencias Maven (capa que solo cambia si pom.xml cambia)
COPY pom.xml ./
RUN mvn dependency:go-offline -B

# Compilación
COPY src ./src
RUN mvn package -DskipTests -B

# Extraer el JAR en capas (Spring Boot layertools)
# Esto optimiza el cacheo de capas de imagen
RUN mkdir -p /app/extracted && \
    java -Djarmode=layertools -jar target/*.jar extract \
      --destination /app/extracted

# ----------------------------------------------------------------

FROM eclipse-temurin:21-jre-alpine AS runtime

WORKDIR /app

# Crear usuario no-root
RUN addgroup -g 1000 appgroup \
    && adduser -u 1000 -G appgroup -s /bin/sh -D appuser

# Copiar capas extraídas del JAR en orden de frecuencia de cambio
COPY --from=builder --chown=1000:1000 /app/extracted/dependencies/ ./
COPY --from=builder --chown=1000:1000 /app/extracted/spring-boot-loader/ ./
COPY --from=builder --chown=1000:1000 /app/extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=1000:1000 /app/extracted/application/ ./

USER 1000:1000

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=5 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**¿Por qué extract con `jarmode=layertools`?**

Spring Boot 2.3+ soporta extraer el fat JAR en capas separadas:
- `dependencies`: librerías de terceros (cambian raramente).
- `spring-boot-loader`: clases del loader de Spring Boot (cambian raramente).
- `snapshot-dependencies`: dependencias snapshot (cambian a veces).
- `application`: tu código compilado (cambia frecuentemente).

Al copiarlas en este orden, cuando cambias tu código solo se reconstruye la capa
`application`, las demás se cachean. Esto acelera los builds subsecuentes
drásticamente.

**Alternativa con JRE distroless (aún más pequeña):**

```dockerfile
# ... stage builder idéntico ...

FROM gcr.io/distroless/java17-debian12 AS runtime
WORKDIR /app
COPY --from=builder /app/extracted/dependencies/ ./
COPY --from=builder /app/extracted/spring-boot-loader/ ./
COPY --from=builder /app/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/extracted/application/ ./
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### 4.12.5 Ejemplo Completo: Python (Django + Gunicorn)

```dockerfile
FROM python:3.12-slim AS builder

# Evitar que Python genere .pyc
ENV PYTHONDONTWRITEBYTECODE=1
# Evitar buffering de stdout/stderr
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Dependencias de sistema necesarias para compilar paquetes Python
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Instalar dependencias Python en un virtualenv
COPY requirements.txt .
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir -r requirements.txt

# ----------------------------------------------------------------

FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libpq5 \
        curl \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario no-root
RUN groupadd -r -g 1000 appgroup \
    && useradd -r -u 1000 -g appgroup -m -s /bin/bash appuser

WORKDIR /app

# Copiar virtualenv desde el builder
COPY --from=builder --chown=1000:1000 /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copiar código de la aplicación
COPY --chown=1000:1000 . .

# Directorio para archivos estáticos y media
RUN mkdir -p /app/staticfiles /app/media \
    && chown -R 1000:1000 /app/staticfiles /app/media

USER 1000:1000

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8000/health/ || exit 1

# Entrypoint script para migraciones y collectstatic
COPY docker-entrypoint.sh /usr/local/bin/
USER root
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
USER 1000:1000

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["gunicorn", "mi_app.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

**Entrypoint script:**

```bash
#!/bin/bash
set -e

echo "=== Docker Entrypoint ==="

# Esperar que PostgreSQL esté disponible
echo "Esperando PostgreSQL en $DB_HOST:$DB_PORT..."
while ! pg_isready -h "$DB_HOST" -p "${DB_PORT:-5432}" -U "$DB_USER" > /dev/null 2>&1; do
    echo "PostgreSQL no está listo - esperando..."
    sleep 2
done
echo "PostgreSQL está listo."

# Ejecutar migraciones
echo "Ejecutando migraciones..."
python manage.py migrate --noinput

# Recopilar archivos estáticos
echo "Recopilando archivos estáticos..."
python manage.py collectstatic --noinput

echo "Iniciando aplicación: $@"
exec "$@"
```

### 4.12.6 Ejemplo Completo: Nginx Sirviendo una SPA React

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# ----------------------------------------------------------------

FROM nginx:1.25-alpine AS runtime

# Eliminar configuración por defecto
RUN rm -rf /usr/share/nginx/html/*

# Copiar el build de React
COPY --from=builder /app/dist /usr/share/nginx/html

# Copiar configuración personalizada de Nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf para SPA (React Router / Vue Router):**

```nginx
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript;
    gzip_min_length 1000;

    # Security headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Cache static assets
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA: todas las rutas que no sean archivos van a index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### 4.12.7 Nombrado de Stages y COPY --from

Cada `FROM` puede nombrarse con `AS <nombre>`:

```dockerfile
FROM node:20-alpine AS dependencies
FROM node:20-alpine AS build
FROM node:20-alpine AS test
FROM node:20-alpine AS security-scan
FROM node:20-alpine AS production
```

Si no nombras una etapa, puedes referenciarla por índice (0, 1, 2...):

```dockerfile
COPY --from=0 /app/node_modules /app/
COPY --from=1 /app/dist /app/dist
```

Pero **siempre nombra las etapas**. El índice es frágil: añade una etapa al
principio y todos los índices cambian.

### 4.12.8 Construir Hasta un Stage Específico (--target)

Puedes construir solo hasta una etapa específica con `--target`. Esto es útil para:

- **Debugging**: construir solo la etapa de compilación para verificar que funciona.
- **Testing**: construir una etapa de testing que ejecuta tests y se descarta.
- **CI/CD**: separar linting, testing y building en etapas del mismo Dockerfile.

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS deps
RUN npm ci

FROM deps AS lint
COPY . .
RUN npm run lint

FROM deps AS test
COPY . .
RUN npm test

FROM deps AS build
COPY . .
RUN npm run build

FROM node:20-alpine AS production
COPY --from=build /app/dist /app/dist
COPY --from=deps /app/node_modules /app/node_modules
CMD ["node", "dist/index.js"]
```

```bash
# Solo instalar dependencias (para verificar)
docker build --target deps -t app:deps .

# Solo lint
docker build --target lint -t app:lint .

# Solo tests
docker build --target test -t app:test .

# Build completo para producción
docker build --target production -t app:latest .
```

### 4.12.9 Patrón: Etapa de Testing en el Dockerfile

Integrar testing en el Dockerfile asegura que nadie despliega sin pasar tests.
Pero añade tiempo al build, considera si es mejor en CI:

```dockerfile
FROM golang:1.22-alpine AS test
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go vet ./...
RUN go test -race -coverprofile=/tmp/coverage.txt ./...

FROM golang:1.22-alpine AS builder
# ... compilación ...

FROM scratch
# ... runtime ...
```

```bash
# En CI: construir hasta test → si pasa, construir hasta runtime
docker build --target test .
docker build -t mi-app:latest .
```

---

## 4.13 BuildKit — El Builder Moderno

### 4.13.1 ¿Qué es BuildKit?

BuildKit es el **motor de construcción de nueva generación** de Docker, reemplazando
al builder clásico. Fue introducido en Docker 18.09 (2018) y se convirtió en el
builder por defecto en Docker 23.0 (2023). Si usas Docker Desktop, BuildKit está
activo por defecto.

**Diferencias clave con el builder clásico:**

| Característica | Builder clásico | BuildKit |
|---|---|---|
| Paralelismo | Secuencial | Paralelo automático |
| Caché | Solo local, por capa | Remoto, granular, `--mount=type=cache` |
| Secretos | No soportado | `--mount=type=secret` |
| SSH agent forwarding | No soportado | `--mount=type=ssh` |
| Outputs | Solo imágenes | Múltiples: imagen, tar, directorio local |
| Rendimiento | Lento | Hasta 10x más rápido |
| Multi-platform | No nativo | `docker buildx build --platform` |

### 4.13.2 Habilitar BuildKit

```bash
# Por comando (una vez)
DOCKER_BUILDKIT=1 docker build -t mi-app .

# Por variable de entorno (permanente)
export DOCKER_BUILDKIT=1

# O en /etc/docker/daemon.json (daemon completo)
{
  "features": {
    "buildkit": true
  }
}

# Verificar que BuildKit está activo
docker buildx version
```

### 4.13.3 Habilitar Sintaxis BuildKit en el Dockerfile

Para usar características avanzadas de BuildKit, añade esta línea al principio:

```dockerfile
# syntax=docker/dockerfile:1
```

Esto activa el parser de Dockerfile de última generación, necesario para
`--mount=type=secret`, `--mount=type=cache`, `--mount=type=ssh`, `--chmod`,
heredoc, y otras características modernas.

**Versiones del frontend:**

```dockerfile
# syntax=docker/dockerfile:1        # Última versión estable (recomendado)
# syntax=docker/dockerfile:1.7      # Versión específica
# syntax=docker/dockerfile:1-labs   # Funcionalidades experimentales
```

### 4.13.4 Secretos con BuildKit

Ya cubierto en profundidad en la sección 4.2.6. Aquí un recordatorio rápido:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t mi-app .
```

### 4.13.5 Caché de Build con --mount=type=cache

Los directorios de caché de BuildKit persisten entre builds pero **no** se incluyen
en la imagen. Esto acelera builds al eliminar descargas y compilaciones repetitivas.

```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu:24.04
# Caché de APT: las listas de paquetes sobreviven entre builds
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y python3 python3-pip curl

FROM node:20-alpine
# Caché de NPM: los paquetes descargados sobreviven entre builds
RUN --mount=type=cache,target=/root/.npm \
    npm ci

FROM golang:1.22-alpine
# Caché de Go modules: los módulos descargados sobreviven entre builds
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

FROM rust:1.78-alpine
# Caché de Cargo: los crates descargados sobreviven entre builds
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    cargo build --release
```

### 4.13.6 SSH Forwarding

BuildKit puede reenviar tu agente SSH al build, permitiendo clonar repositorios
privados o instalar paquetes desde fuentes autenticadas:

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.20

# Instalar dependencia privada desde Git
RUN apk add --no-cache openssh-client
RUN --mount=type=ssh \
    mkdir -p -m 0700 ~/.ssh && \
    ssh-keyscan github.com >> ~/.ssh/known_hosts && \
    git clone git@github.com:empresa/repo-privado.git /tmp/repo
```

```bash
# Asegura que tu agente SSH esté ejecutándose y tenga la clave cargada
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa

docker build --ssh default -t mi-app .
```

### 4.13.7 Caché Remoto

BuildKit puede importar y exportar caché a un registry remoto, compartiendo la
caché entre máquinas (útil en CI/CD):

```bash
# Exportar caché a un registry (después de build)
docker buildx build \
  --cache-to type=registry,ref=registry.example.com/cache:mi-app \
  -t mi-app:latest .

# Importar caché de un registry (antes de build)
docker buildx build \
  --cache-from type=registry,ref=registry.example.com/cache:mi-app \
  --cache-to type=registry,ref=registry.example.com/cache:mi-app \
  -t mi-app:latest .
```

**Tipos de caché:**

| Tipo | Descripción | Uso |
|---|---|---|
| `type=registry,ref=...` | Almacena en OCI registry | CI/CD, equipos |
| `type=local,src=...` | Directorio local | Builds locales |
| `type=gha,url=...` | GitHub Actions cache | GitHub Actions CI |
| `type=s3,region=...,bucket=...` | Amazon S3 | AWS CI/CD |
| `type=inline` | Incrustado en la imagen | Imágenes simples |

### 4.13.8 Output Directo (--output)

BuildKit permite exportar artefactos directamente al sistema de archivos del host
sin crear una imagen:

```bash
# Exportar el stage builder a un directorio local
docker buildx build \
  --target builder \
  --output type=local,dest=./output \
  .

# Exportar un archivo específico
docker buildx build \
  --output type=tar,dest=resultado.tar \
  .

# Exportar la imagen al daemon local (por defecto)
docker buildx build \
  --output type=docker \
  -t mi-app:latest \
  .
```

---

## 4.14 Optimización de Dockerfiles — Reglas de Oro

### 4.14.1 Regla 1: Ordenar por Frecuencia de Cambio

Las capas de Docker se cachean. Cuando una capa cambia, **todas las capas siguientes
se reconstruyen**. Por tanto, debes ordenar las instrucciones de menos cambiante a
más cambiante.

```dockerfile
# BIEN: las dependencias (menos frecuentes) antes del código (más frecuente)
FROM node:20-alpine
WORKDIR /app

# Capa 1: Rara vez cambia (cuando subes de versión de Node)
# (implícito en FROM)

# Capa 2: Cambia cuando cambian las dependencias
COPY package.json package-lock.json ./
RUN npm ci

# Capa 3: Cambia frecuentemente (cada commit)
COPY . .

CMD ["node", "dist/index.js"]
```

```dockerfile
# MAL: código fuente antes que dependencias → cada cambio de código
# reinstala todas las dependencias
FROM node:20-alpine
WORKDIR /app

COPY . .
RUN npm ci    # Se ejecuta en CADA build porque COPY . . siempre cambia

CMD ["node", "dist/index.js"]
```

### 4.14.2 Regla 2: Minimizar el Número de Capas

Cada instrucción crea una capa. Aunque el límite de capas se eliminó, más capas =
más metadatos, más overhead en pull/push, y potencial degradación de rendimiento
en el sistema de archivos.

```dockerfile
# MAL: múltiples RUN, COPY y ENV separados
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get clean
COPY file1.txt /app/
COPY file2.txt /app/
ENV VAR1=valor1
ENV VAR2=valor2

# BIEN: fusionar comandos relacionados
RUN apt-get update \
    && apt-get install -y curl vim \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
COPY file1.txt file2.txt /app/
ENV VAR1=valor1 \
    VAR2=valor2
```

**Sin embargo, no fusiones ciegamente. Balancea:**

- Capas **lógicas**: agrupa comandos relacionados semánticamente.
- Capas **de caché**: separa lo que cambia raramente de lo que cambia frecuentemente.
- Capas **de tamaño**: fusiona para reducir el tamaño final, especialmente `RUN`
  de instalación + limpieza.

### 4.14.3 Regla 3: Minimizar el Contexto de Build

El contexto de build es TODO el directorio desde donde ejecutas `docker build`.
Docker lo envía completo al daemon antes de procesar el Dockerfile. La regla
`.dockerignore` (sección 4.3.3) es vital aquí.

```bash
# Ver el tamaño del contexto (sin hacer build)
docker build --no-cache -t test . 2>&1 | head -5
```

### 4.14.4 Regla 4: Usar Imágenes Slim/Alpine

Ya cubierto en la sección 4.1.2. La regla simple: si tu lenguaje/framework lo
soporta, usa `:alpine` o `:slim`. Solo usa la imagen completa si necesitas
herramientas de compilación o compatibilidad glibc que no puedes resolver.

### 4.14.5 Regla 5: Limpiar en la Misma Capa

```dockerfile
# MAL: limpieza en capa separada (no reduce el tamaño)
RUN apt-get update && apt-get install -y build-essential
# ... compilar ...
RUN apt-get remove -y build-essential && apt-get autoremove -y

# BIEN: limpieza en la misma capa que la compilación
RUN apt-get update \
    && apt-get install -y build-essential \
    && make all \
    && make install \
    && apt-get remove -y build-essential \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/*
```

### 4.14.6 Regla 6: Usar Multi-Stage en Lugar de Limpiar

La regla anterior es un parche. La solución real es multi-stage:

```dockerfile
# BIEN: compilar en un stage, runtime en otro → nada que limpiar
FROM gcc:13 AS builder
RUN make all

FROM debian:stable-slim
COPY --from=builder /app/output /app/
CMD ["/app/mi-binario"]
```

### 4.14.7 Regla 7: Squash de Capas

`docker build --squash` fusiona todas las capas en una sola, eliminando archivos
borrados en capas posteriores y reduciendo el tamaño final.

```bash
# Requiere modo experimental habilitado
docker build --squash -t mi-app:squashed .
```

**Pros del squash:**
- Imagen más pequeña (elimina archivos "ocultos" en capas inferiores).
- Imagen más fácil de inspeccionar (una sola capa de filesystem).
- Puede limpiar secretos que accidentalmente quedaron en capas intermedias.

**Contras del squash:**
- Pierdes el cacheo de capas (la imagen final es una sola capa). Los pulls no
  se benefician de capas compartidas con otras imágenes.
- No es compatible con multi-stage (cada stage tiene su propio squash implícito).
- No está disponible en todos los builders.

**Recomendación:** Con multi-stage, el squash es raramente necesario. La etapa
final solo contiene lo que explícitamente copiaste, que debería ser mínimo.

### 4.14.8 Regla 8: .dockerignore Adecuado

Cubierto en detalle en la sección 4.3.3. Aquí un recordatorio de patrones clave:

```dockerignore
# Siempre excluir
.git/
node_modules/
__pycache__/
*.pyc
*.pyo
.pytest_cache/
vendor/
target/
build/
dist/

# Excluir según tu stack
.env
*.log
.DS_Store
```

### 4.14.9 Regla 9: No Instalar Paquetes Innecesarios

Cada paquete instalado aumenta la superficie de ataque y el tamaño. Pregúntate:
¿realmente necesito `vim`, `curl`, `ping`, `telnet` en producción?

```dockerfile
# MAL: debug tools en producción
RUN apt-get install -y curl vim netcat telnet htop strace

# BIEN: solo lo necesario
RUN apt-get install -y --no-install-recommends ca-certificates
```

Si necesitas debuggear, crea una imagen de debug separada o usa `kubectl debug`.

### 4.14.10 Regla 10: Versionar TODO

```dockerfile
# MAL
FROM node:alpine
RUN apk add --no-cache curl
RUN npm install mi-paquete

# BIEN
FROM node:20.11.1-alpine3.19
RUN apk add --no-cache curl=8.6.0-r0
RUN npm install mi-paquete@2.1.3
```

---

## 4.15 Anti-Patrones en Dockerfiles

### 4.15.1 Anti-Patrón 1: La Imagen "Kitchen Sink"

```dockerfile
# HORRIBLE: imagen que intenta hacerlo todo
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y \
    nodejs npm \
    python3 python3-pip \
    openjdk-17-jdk \
    golang \
    nginx \
    redis-server \
    postgresql \
    vim curl wget telnet htop \
    && apt-get clean
CMD ["/bin/bash"]
```

**Problemas:** 2.5GB, cientos de CVEs, completamente innavegable, imposible de
auditar.

**Solución:** Un contenedor = un proceso = un propósito. Si necesitas múltiples
servicios, usa Docker Compose.

### 4.15.2 Anti-Patrón 2: `apt-get upgrade` o `apk upgrade`

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get upgrade -y
```

`apt-get upgrade` actualiza TODOS los paquetes del sistema a sus últimas versiones,
haciendo tu build no determinista. Si la imagen base tiene una vulnerabilidad, usa
`docker scout` o `trivy` para detectarla y actualiza la imagen base.

### 4.15.3 Anti-Patrón 3: Dependencia del Tiempo de Build

```dockerfile
RUN curl -fsSL https://example.com/install.sh | sh
```

- La URL podría desaparecer mañana.
- El contenido podría cambiar sin que lo notes.
- `curl | sh` es un vector de ataque (script remoto ejecutándose en tu build).

**Solución:** versionar el script, verificar checksum.

```dockerfile
ADD https://example.com/v1.2.3/install.sh /tmp/install.sh
RUN echo "sha256:abc123... /tmp/install.sh" | sha256sum -c - \
    && bash /tmp/install.sh \
    && rm /tmp/install.sh
```

### 4.15.4 Anti-Patrón 4: Secretos en ARG/ENV

Ya demostrado en la sección 4.6.4. Nunca:

```dockerfile
ARG DATABASE_URL=postgres://user:password@host/db
ENV API_KEY=sk-1234567890abcdef
COPY .aws/credentials /root/.aws/
COPY id_rsa /root/.ssh/
```

### 4.15.5 Anti-Patrón 5: `FROM latest`

```dockerfile
FROM node:latest
```

Ya cubierto en 4.1.2. Nunca uses `:latest` en producción.

### 4.15.6 Anti-Patrón 6: Mapear Puertos en el Dockerfile

```dockerfile
# MAL: esto NO publica puertos. Es confuso.
EXPOSE 8080
CMD ["docker", "run", "-p", "8080:8080"]  # Esto no funciona aquí
```

`EXPOSE` no publica. `docker run` no pertenece al Dockerfile.

### 4.15.7 Anti-Patrón 7: Volumen para Código en Producción

```dockerfile
FROM node:20-alpine
COPY . /app
VOLUME /app    # ¿Por qué? El código debe ser inmutable
```

En producción, el código es parte de la imagen. Los volúmenes son para datos,
no para código. Si necesitas cambiar código, reconstruye la imagen.

### 4.15.8 Anti-Patrón 8: Procesos Múltiples sin Supervisor

```dockerfile
CMD node server.js & nginx -g 'daemon off;'
```

Docker solo monitorea PID 1. Si `node server.js` muere, Docker no lo detecta porque
el shell (PID 1) sigue vivo. Si el shell muere, ambos procesos mueren abruptamente.

**Solución:** Un proceso por contenedor. Si realmente necesitas múltiples procesos,
usa un proceso init como `tini` o replantéate la arquitectura primero.

```dockerfile
# Con tini como init
RUN apk add --no-cache tini
COPY start.sh /start.sh
ENTRYPOINT ["/sbin/tini", "--", "/start.sh"]
```

### 4.15.9 Anti-Patrón 9: `RUN cd` en Lugar de WORKDIR

```dockerfile
RUN cd /app && npm ci    # cd solo afecta a esa línea
RUN npm run build        # Se ejecuta en /, NO en /app
```

`cd` en shell solo persiste durante ese comando. Cada `RUN` es un shell nuevo.

### 4.15.10 Anti-Patrón 10: Root en Todo

```dockerfile
FROM node:20-alpine
# Sin USER → el contenedor corre como root
# Sin HEALTHCHECK → Docker no sabe si funciona
# Sin STOPSIGNAL → SIGTERM, pero shell form no lo recibe
CMD npm start
```

---
## 4.16 Laboratorio: Dockerfiles Completos para Stacks Reales

### 4.16.1 Aplicación Node.js (Express + TypeScript) en Producción

Estructura del proyecto:

```
mi-api/
├── .dockerignore
├── Dockerfile
├── docker-entrypoint.sh
├── package.json
├── package-lock.json
├── tsconfig.json
├── src/
│   ├── index.ts
│   ├── routes/
│   ├── middleware/
│   └── config/
└── tests/
```

**.dockerignore:**

```dockerignore
node_modules/
dist/
.git/
.env
.env.*
*.log
.DS_Store
.github/
.vscode/
.idea/
tests/
coverage/
README.md
```

**Dockerfile completo:**

```dockerfile
# syntax=docker/dockerfile:1

# ============================================================
# Stage 1: Instalar dependencias de producción
# ============================================================
FROM node:20.11.1-alpine3.19 AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

# ============================================================
# Stage 2: Compilar TypeScript
# ============================================================
FROM node:20.11.1-alpine3.19 AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# ============================================================
# Stage 3: Runtime de producción
# ============================================================
FROM node:20.11.1-alpine3.19 AS runtime

# Instalar dependencias de sistema necesarias
RUN apk add --no-cache \
    ca-certificates \
    tzdata \
    curl

# Crear usuario no-root
RUN addgroup -g 1000 -S appgroup \
    && adduser -u 1000 -S appuser -G appgroup

WORKDIR /app

# Copiar dependencias de producción
COPY --from=deps --chown=1000:1000 /app/node_modules ./node_modules

# Copiar código compilado
COPY --from=builder --chown=1000:1000 /app/dist ./dist

# Copiar package.json (metadatos de runtime)
COPY --chown=1000:1000 package.json ./

# Copiar y preparar entrypoint
COPY --chown=1000:1000 docker-entrypoint.sh ./
RUN chmod +x docker-entrypoint.sh

# Variables de entorno por defecto
ENV NODE_ENV=production
ENV PORT=3000

# Cambiar a usuario no-root
USER 1000:1000

# Documentar puerto
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s \
            --timeout=5s \
            --start-period=15s \
            --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Entrypoint (setup de runtime) + CMD (aplicación)
ENTRYPOINT ["./docker-entrypoint.sh"]
CMD ["node", "dist/index.js"]
```

**docker-entrypoint.sh:**

```bash
#!/bin/sh
set -e

echo "=== $(date) - Starting container ==="

# Log de configuración (sin secretos)
echo "NODE_ENV: ${NODE_ENV:-production}"
echo "PORT: ${PORT:-3000}"

# Verificar conectividad a dependencias si las URLs están definidas
if [ -n "$DATABASE_URL" ]; then
    echo "Verificando conectividad a la base de datos..."
    # Un simple test de conectividad (ajusta según tu stack)
fi

# Ejecutar migraciones si está configurado
if [ "${RUN_MIGRATIONS:-false}" = "true" ]; then
    echo "Ejecutando migraciones..."
    node dist/migrate.js
fi

echo "Iniciando aplicación..."
exec "$@"
```

### 4.16.2 Aplicación Python (Django + Gunicorn) en Producción

Estructura:

```
django-app/
├── .dockerignore
├── Dockerfile
├── docker-entrypoint.sh
├── requirements.txt
├── manage.py
├── mi_app/
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   └── production.py
│   ├── urls.py
│   └── wsgi.py
└── apps/
```

**Dockerfile:**

```dockerfile
# syntax=docker/dockerfile:1

# ============================================================
# Stage 1: Instalar dependencias Python
# ============================================================
FROM python:3.12.3-slim-bookworm AS builder

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PIP_NO_CACHE_DIR=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Dependencias de sistema para compilar paquetes Python
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        libpq-dev \
        libjpeg-dev \
        zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

# Instalar dependencias en virtualenv
COPY requirements.txt .
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir -r requirements.txt

# ============================================================
# Stage 2: Runtime de producción
# ============================================================
FROM python:3.12.3-slim-bookworm AS runtime

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app
ENV DJANGO_SETTINGS_MODULE=mi_app.settings.production

# Dependencias de sistema solo para runtime
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libpq5 \
        libjpeg62 \
        mime-support \
        curl \
        postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario no-root
RUN groupadd -r -g 1000 appgroup \
    && useradd -r -u 1000 -g appgroup -m -d /home/appuser -s /bin/bash appuser

# Copiar virtualenv desde el builder
COPY --from=builder --chown=1000:1000 /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Copiar código de la aplicación
COPY --chown=1000:1000 . .

# Directorios para archivos estáticos, media, y socket
RUN mkdir -p /app/staticfiles /app/media /app/run \
    && chown -R 1000:1000 /app/staticfiles /app/media /app/run

# Copiar y preparar entrypoint
COPY --chown=1000:1000 docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

USER 1000:1000

EXPOSE 8000

HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=45s \
            --retries=5 \
  CMD curl -f http://localhost:8000/health/ || exit 1

STOPSIGNAL SIGTERM

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["gunicorn", "mi_app.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--worker-tmp-dir", "/dev/shm", \
     "--max-requests", "1000", \
     "--max-requests-jitter", "50", \
     "--keep-alive", "5", \
     "--access-logfile", "-", \
     "--error-logfile", "-", \
     "--log-level", "info"]
```

**docker-entrypoint.sh para Django:**

```bash
#!/bin/bash
set -e

echo "=== $(date) - Docker Entrypoint ==="

# Esperar que PostgreSQL esté disponible
echo "Esperando PostgreSQL en ${DB_HOST:-postgres}:${DB_PORT:-5432}..."
while ! pg_isready -h "${DB_HOST:-postgres}" -p "${DB_PORT:-5432}" -U "${DB_USER:-postgres}" > /dev/null 2>&1; do
    echo "PostgreSQL no está listo - esperando..."
    sleep 2
done
echo "PostgreSQL está listo."

# Ejecutar migraciones
echo "Ejecutando migraciones..."
python manage.py migrate --noinput

# Recopilar archivos estáticos
echo "Recopilando archivos estáticos..."
python manage.py collectstatic --noinput

echo "Iniciando aplicación: $@"
exec "$@"
```

### 4.16.3 Aplicación Go — API REST

Estructura:

```
go-api/
├── .dockerignore
├── Dockerfile
├── go.mod
├── go.sum
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── handlers/
│   ├── models/
│   └── database/
└── migrations/
```

**Dockerfile:**

```dockerfile
# syntax=docker/dockerfile:1

ARG VERSION=dev

# ============================================================
# Stage 1: Testing
# ============================================================
FROM golang:1.22.4-alpine3.20 AS test
WORKDIR /app
RUN apk add --no-cache git ca-certificates
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go vet ./...
RUN go test -v -race -coverprofile=/tmp/coverage.out -covermode=atomic ./...

# ============================================================
# Stage 2: Compilación
# ============================================================
FROM golang:1.22.4-alpine3.20 AS builder
WORKDIR /app
RUN apk add --no-cache git ca-certificates

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

ARG VERSION
RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
      -ldflags="-w -s -X main.version=${VERSION} \
                -X main.buildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                -X main.gitCommit=$(git rev-parse --short HEAD 2>/dev/null || echo unknown)" \
      -trimpath \
      -o /bin/api \
      ./cmd/api

# ============================================================
# Stage 3: Runtime
# ============================================================
FROM gcr.io/distroless/static-debian12:nonroot AS runtime

# El usuario nonroot tiene UID 65532
# Copiar certificados CA para HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copiar el binario
COPY --from=builder /bin/api /api

# Copiar zona horaria
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/api", "health"]

ENTRYPOINT ["/api"]
```

### 4.16.4 Aplicación Java (Spring Boot + Maven)

```dockerfile
# syntax=docker/dockerfile:1

# ============================================================
# Stage 1: Build con Maven
# ============================================================
FROM maven:3.9.6-eclipse-temurin-21-alpine AS builder
WORKDIR /app

# Cache de dependencias
COPY pom.xml ./
RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:go-offline -B -q

# Compilación
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn package -DskipTests -B -q

# Extraer capas del JAR para optimizar cacheo de imagen
RUN java -Djarmode=layertools -jar target/*.jar extract \
      --destination /app/extracted

# ============================================================
# Stage 2: Runtime
# ============================================================
FROM eclipse-temurin:21-jre-alpine AS runtime

RUN apk add --no-cache curl

# Crear usuario no-root
RUN addgroup -g 1000 -S appgroup \
    && adduser -u 1000 -S appuser -G appgroup

WORKDIR /app

# Copiar capas en orden de frecuencia de cambio
COPY --from=builder --chown=1000:1000 /app/extracted/dependencies/ ./
COPY --from=builder --chown=1000:1000 /app/extracted/spring-boot-loader/ ./
COPY --from=builder --chown=1000:1000 /app/extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=1000:1000 /app/extracted/application/ ./

USER 1000:1000

# Opciones de JVM para producción
ENV JAVA_OPTS="-XX:+UseG1GC \
               -XX:MaxRAMPercentage=75.0 \
               -XX:InitialRAMPercentage=25.0 \
               -XX:+ExitOnOutOfMemoryError \
               -Djava.security.egd=file:/dev/urandom"

EXPOSE 8080

HEALTHCHECK --interval=30s \
            --timeout=5s \
            --start-period=60s \
            --retries=5 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", \
  "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

### 4.16.5 Nginx Sirviendo una SPA (React) en Producción

```dockerfile
# syntax=docker/dockerfile:1

# ============================================================
# Stage 1: Build de la SPA
# ============================================================
FROM node:20.11.1-alpine3.19 AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

COPY . .

# Variables de build de la aplicación
ARG VITE_API_URL
ENV VITE_API_URL=${VITE_API_URL:-https://api.example.com}

RUN npm run build

# ============================================================
# Stage 2: Runtime con Nginx
# ============================================================
FROM nginx:1.25.3-alpine AS runtime

# Instalar curl para healthcheck
RUN apk add --no-cache curl

# Eliminar archivos por defecto
RUN rm -rf /usr/share/nginx/html/* \
    && rm -f /etc/nginx/conf.d/default.conf

# Copiar el build de React
COPY --from=builder /app/dist /usr/share/nginx/html

# Copiar configuración de Nginx
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Configurar permisos
RUN chown -R nginx:nginx /usr/share/nginx/html \
    && chown -R nginx:nginx /var/cache/nginx \
    && chown -R nginx:nginx /var/log/nginx \
    && chown -R nginx:nginx /etc/nginx/conf.d \
    && touch /var/run/nginx.pid \
    && chown -R nginx:nginx /var/run/nginx.pid

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf para la SPA con optimizaciones de producción:**

```nginx
# Configuración de caché para archivos estáticos basada en MIME type
map $sent_http_content_type $expires {
    default                    off;
    text/html                  epoch;
    text/css                   max;
    application/javascript     max;
    application/json           max;
    image/svg+xml              max;
    image/png                  max;
    image/jpeg                 max;
    image/gif                  max;
    image/x-icon               max;
    font/woff                  max;
    font/woff2                 max;
}

server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    expires $expires;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json
               application/javascript application/xml+rss
               application/atom+xml image/svg+xml;

    # Security headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    # Cache estático inmutable (hashed filenames)
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # SPA fallback: todas las rutas → index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Health check
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Denegar acceso a archivos ocultos
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

---

## 4.17 Resumen del Capítulo

### 4.17.1 Checklist para un Dockerfile de Producción

```
□  Usa imagen base específica (no :latest)
□  Considera alpine/slim/distroless/scratch según necesidad
□  Multi-stage: separa build de runtime
□  RUN: comandos encadenados con &&
□  Limpieza de cachés en la misma capa que la instalación
□  COPY en lugar de ADD (salvo tar con checksum)
□  .dockerignore bien configurado
□  CMD y ENTRYPOINT en exec form (no shell form)
□  Patrón docker-entrypoint.sh con exec "$@"
□  WORKDIR para todas las operaciones
□  ARG para build-time, ENV para runtime
□  NUNCA secretos en ARG/ENV — usa BuildKit secrets
□  EXPOSE para documentar puertos (no publica)
□  VOLUME solo para datos persistentes, no para código/deps
□  USER no-root (UID/GID 1000+)
□  HEALTHCHECK con intervalos razonables
□  STOPSIGNAL correcto para tu aplicación
□  LABEL OCI con metadatos de la imagen
□  Capas ordenadas por frecuencia de cambio
□  Versiones fijas para paquetes instalados
```

### 4.17.2 La Pirámide de Calidad del Dockerfile

```
                          ┌─────────┐
                          │  PROD   │  HEALTHCHECK + USER + STOPSIGNAL
                          │ READY   │  LABEL OCI + entrypoint script
                          ├─────────┤
                          │ SECURE  │  non-root + BuildKit secrets
                          │         │  distroless/scratch cuando sea posible
                          ├─────────┤
                          │   OPT   │  multi-stage + .dockerignore
                          │         │  capas ordenadas + limpieza en capa
                          ├─────────┤
                          │ CORRECT │  exec form CMD/ENTRYPOINT
                          │         │  COPY > ADD, WORKDIR, versión fija
                          ├─────────┤
                          │  BASE   │  FROM imagen:versión-específica
                          │         │  RUN encadenados, slim/alpine
                          └─────────┘
```

### 4.17.3 Tabla de Decisión Rápida por Instrucción

| Instrucción | Haz esto | No hagas esto |
|---|---|---|
| `FROM` | `node:20.11.1-alpine3.19` | `node:latest` |
| `RUN` | `&&` encadenado con limpieza | Capas separadas sin limpiar |
| `COPY` | `--chown=1000:1000` | `ADD` para archivos normales |
| `ADD` | Solo tar con checksum de URL | `ADD https://...` sin checksum |
| `CMD` | `CMD ["app"]` (exec form) | `CMD app` (shell form) |
| `ENTRYPOINT` | `ENTRYPOINT ["script.sh"]` | `ENTRYPOINT script.sh` |
| `WORKDIR` | Siempre usarlo | `RUN cd /app && ...` |
| `ENV` | Configuración de runtime | Secretos, contraseñas |
| `ARG` | Versiones, flags de build | Secretos, contraseñas |
| `EXPOSE` | `EXPOSE 8080` (documental) | Pensar que publica puertos |
| `VOLUME` | `/data`, `/var/lib/mysql` | `/app/node_modules` |
| `USER` | `USER 1000:1000` | Dejar root por defecto |
| `HEALTHCHECK` | Check semántico (curl, wget) | No poner ninguno |

### 4.17.4 Errores Más Comunes que Vemos en Producción

1. **Shell form en CMD/ENTRYPOINT** — el contenedor tarda 10s en detenerse y no
   recibe SIGTERM. Es el error más frecuente y el más grave.

2. **Argumentos de build con secretos** — tokens de npm, claves de API, contraseñas
   de base de datos pasadas con `--build-arg`. Quedan expuestos en `docker history`.

3. **`FROM node:latest`** — un día el build funciona, al siguiente no. Peor: funciona
   pero con una versión diferente sin que nadie lo note.

4. **Sin HEALTHCHECK** — el orquestador no sabe si la app funciona. Reinicia
   contenedores que están bien y no reinicia contenedores que están mal.

5. **Sin USER** — root en producción. El 90% de las imágenes en Docker Hub corren
   como root. No seas parte del 90%.

6. **COPY . . al principio** — cada cambio en cualquier archivo invalida la caché
   de dependencias. Las dependencias se reinstalan en cada build.

7. **Instalar todo en una imagen** — nginx + node + redis + postgres en un solo
   contenedor porque "así es más fácil". Es una pesadilla de seguridad y
   mantenimiento.

8. **Sin .dockerignore** — enviando 2GB de `node_modules` y `.git` al daemon en
   cada build. 45 segundos de espera que se repiten cientos de veces.

9. **`docker build` sin BuildKit** — perdiendo `--mount=type=secret`, caché
   granular, SSH forwarding y builds 10x más rápidos.

10. **`apt-get upgrade` en el build** — el build deja de ser reproducible. Una
    actualización de un paquete del sistema puede romper la aplicación sin que
    el código fuente haya cambiado.

### 4.17.5 El Dockerfile Mínimo Universal

Este Dockerfile de 12 líneas es el esqueleto del que deberías partir para cualquier
proyecto nuevo:

```dockerfile
FROM <lenguaje>:<version>-alpine AS builder
WORKDIR /app
COPY <archivos-deps> ./
RUN <instalar-deps>
COPY . .
RUN <comando-compilacion>

FROM <lenguaje>:<version>-alpine
WORKDIR /app
COPY --from=builder /app/<artefactos> ./
USER 1000:1000
CMD ["<ejecutable>"]
```

Sobre este esqueleto, añade: HEALTHCHECK, .dockerignore, docker-entrypoint.sh,
BuildKit cache mounts, LABEL OCI, y las optimizaciones específicas de tu stack.

### 4.17.6 Conclusión

El Dockerfile es la **interfaz entre tu código y la infraestructura**. Un Dockerfile
mal escrito produce imágenes infladas, inseguras y lentas. Un Dockerfile bien escrito
produce artefactos mínimos, deterministas y mantenibles que se despliegan en segundos.

Las reglas no son muchas, pero son estrictas:
- Usa exec form para CMD y ENTRYPOINT.
- Nombra tus stages.
- Limpia en la misma capa.
- Usa .dockerignore.
- No seas root.
- No filtres secretos.

No necesitas memorizar todo este capítulo. Vuelve a él cuando escribas un Dockerfile
nuevo y verifica cada instrucción contra la checklist de la sección 4.17.1. En pocos
meses, estas prácticas se convertirán en memoria muscular y escribirás Dockerfiles de
calidad profesional sin pensarlo.

---

*"Un Dockerfile no se escribe. Se diseña."*

---

## 4.16 Debugging de builds: cuando el Dockerfile miente

Un Dockerfile de 15 líneas puede fallar en producción de formas que jamás
imaginarías inspeccionando el código fuente. Lo que ves en el editor no es lo que
ejecuta el daemon. Las capas intermedias cuentan historias que el build exitoso
oculta. Esta sección es la caja de herramientas forenses que todo ingeniero
de Docker necesita cuando el build "funciona en mi máquina" pero explota en CI.

### 4.16.1 Inspeccionar capas intermedias con --target

El 90% de los problemas de build están en una capa intermedia que ni siquiera
llega a la imagen final. La técnica fundamental es detener el build en una etapa
específica y abrir un shell para inspeccionar el filesystem resultante.

```bash
# El Dockerfile:
# FROM node:20-alpine AS deps
# RUN npm ci
# FROM node:20-alpine AS builder
# COPY . .
# RUN npm run build          # <-- falla aquí
# FROM nginx:1.25-alpine
# COPY --from=builder /app/dist /usr/share/nginx/html

# Detenerse en la etapa builder para inspeccionar:
docker build --target builder -t debug:builder .

# Ahora lanzar un shell en esa imagen intermedia:
docker run --rm -it debug:builder sh

# Dentro, puedes inspeccionar:
ls -la /app/
cat /app/tsconfig.json
ls /app/node_modules/
npm run build -- --verbose   # ejecutar el paso problemático manualmente
```

**Patrón sistemático de debugging por etapas:**

```bash
# 1. Construir cada etapa por separado para aislar el fallo
docker build --target deps -t debug:deps .
docker run --rm -it debug:deps sh

docker build --target builder -t debug:builder .
docker run --rm -it debug:builder sh

docker build --target runtime -t debug:runtime .
docker run --rm -it debug:runtime sh

# 2. Comparar los filesystems de dos etapas
docker run --rm debug:deps ls -R /app > /tmp/stage-deps.txt
docker run --rm debug:builder ls -R /app > /tmp/stage-builder.txt
diff /tmp/stage-deps.txt /tmp/stage-builder.txt
```

**Truco avanzado: inspeccionar una etapa sin nombre:**

```bash
# Los stages sin nombre se indexan como 0, 1, 2...
docker build --target 1 -t debug:stage1 .
```

### 4.16.2 Inspeccionar el filesystem de cualquier capa

Cada imagen es una pila de capas identificadas por su hash. Puedes ejecutar un
contenedor desde cualquier capa intermedia, no solo desde la imagen final.

```bash
# Listar todas las capas con sus hashes y tamaños
docker image history --no-trunc --human=false mi-app:latest

# Output típico:
# IMAGE          CREATED BY                                     SIZE
# sha256:abc123  /bin/sh -c npm run build                       52428800
# sha256:def456  /bin/sh -c npm ci                              104857600
# sha256:ghi789  /bin/sh -c apk add --no-cache curl             2097152

# Ejecutar un shell desde una capa intermedia:
docker run --rm -it sha256:abc123 sh
# Esto arranca la imagen HASTA esa capa (incluyéndola)

# Inspeccionar qué archivos introdujo una capa específica:
docker run --rm sha256:def456 find / -newer /proc/1 -type f 2>/dev/null
```

**Dive: la herramienta definitiva para inspección de capas:**

```bash
# Instalar dive (macOS)
brew install dive

# Inspeccionar una imagen capa por capa
dive mi-app:latest

# Dentro de dive:
# - Tecla Tab: cambiar entre vista de capas y vista de archivos
# - Ctrl+U: filtrar por archivos añadidos en esta capa
# - Ctrl+A: filtrar por archivos modificados
# - Ctrl+D: filtrar por archivos eliminados
# - /: buscar archivo por nombre
# - Ctrl+Space: colapsar/expandir directorios

# También funciona con imágenes remotas y en CI
dive node:20-alpine --ci --lowestEfficiency 0.95
# --ci: modo no-interactivo para CI
# --lowestEfficiency: falla si la eficiencia (bytes útiles / bytes totales) es baja
```

**Dive para encontrar exactamente qué capa introdujo un archivo:**

```bash
# Supón que encuentras un archivo /etc/secrets/db_password en tu imagen
dive mi-app:latest

# 1. Navegar con /etc/secrets/db_password
# 2. Dive mostrará en qué capa se introdujo (o modificó por última vez)
# 3. Anotar el hash de esa capa
# 4. Volver al Dockerfile y localizar el RUN/COPY que generó esa capa
```

### 4.16.3 El caso real: "la imagen funciona en mi PC pero no en CI"

Este es el bug más desgastante del ecosistema Docker. La causa raíz nunca es magia,
siempre es una diferencia medible entre los dos entornos. Aquí el protocolo forense
completo:

**Paso 1: Verificar que la imagen es idéntica bit a bit**

```bash
# En tu PC:
docker pull mi-app:ci-1234
docker inspect mi-app:ci-1234 --format='{{.Id}}'
# Output: sha256:abc123...

# En CI:
docker inspect mi-app:ci-1234 --format='{{.Id}}'
# Output: sha256:def456...  <-- DIFERENTE, el build no es determinista

# Comparar los hashes de todas las capas:
docker inspect mi-app:ci-1234 --format='{{json .RootFS.Layers}}' | jq
```

**Paso 2: Comparar metadatos de imagen (no solo el hash)**

```bash
# Extraer todos los metadatos relevantes
docker image inspect mi-app:ci-1234 > /tmp/inspect-local.json
docker image inspect mi-app:ci-1234 > /tmp/inspect-ci.json

# Comparar campos clave
diff <(jq -S '.[0].Config' /tmp/inspect-local.json) \
     <(jq -S '.[0].Config' /tmp/inspect-ci.json)

# Diferencias típicas que rompen el build:
# - Env: variables de entorno diferentes entre entornos
# - Entrypoint: diferente shell en CI vs local
# - Labels: build metadata que difiere por timestamp
```

**Paso 3: Comparar los contextos de build**

```bash
# El contexto de build es el directorio que Docker empaqueta y envía al daemon.
# Diferencias aquí producen imágenes diferentes con el mismo Dockerfile.

# Generar tarball del contexto local:
docker build --no-cache -t dummy . 2>&1 | grep "Sending build context"
# Sending build context to Docker daemon  542.3MB

# En CI, revisar el equivalente:
# - ¿El .dockerignore es el mismo?
# - ¿El checkout de git incluye archivos extra (submódulos, LFS)?
# - ¿Hay archivos generados en local que no están en CI (.env, certificates)?

# Inspección exhaustiva del contexto:
tar czf /tmp/build-context.tar.gz .
ls -lh /tmp/build-context.tar.gz
tar tzf /tmp/build-context.tar.gz | sort > /tmp/context-files.txt

# Verificar que .dockerignore está filtrando correctamente:
docker build --no-cache -t test 2>&1 | head -3
# El output muestra el tamaño exacto del contexto enviado
```

**Paso 4: Comparar build args y variables de entorno**

```bash
# Build args pueden cambiar drásticamente el resultado
# Extraer todos los ARG usados del Dockerfile:
grep -n 'ARG ' Dockerfile

# Comparar con lo que CI pasa realmente:
# Revisar el pipeline log o el comando docker build exacto

# Exportar las variables del build para comparar:
docker run --rm mi-app:local env | sort > /tmp/env-local.txt
docker run --rm mi-app:ci env | sort > /tmp/env-ci.txt
diff /tmp/env-local.txt /tmp/env-ci.txt
```

**Paso 5: Inspeccionar arquitectura y plataforma**

```bash
# En CI (que típicamente es linux/amd64):
docker image inspect mi-app:ci --format '{{.Os}}/{{.Architecture}}'
# linux/amd64

# En tu Mac M1/M2/M3 (linux/arm64):
docker image inspect mi-app:local --format '{{.Os}}/{{.Architecture}}'
# linux/arm64  <-- ¡DIFERENTE! La imagen local es para ARM, CI para AMD64

# Esto explica muchos "funciona en mi PC" — la arquitectura es diferente.
# Solución: siempre especificar plataforma en desarrollo:
docker build --platform linux/amd64 -t mi-app:local-amd64 .

# O configurar buildx para multi-plataforma (sección 4.18)
```

**Paso 6: Verificar versión del builder**

```bash
# BuildKit vs builder clásico producen resultados diferentes en edge cases
docker buildx version
# github.com/docker/buildx v0.14.0

# En CI, revisar la versión y asegurar consistencia:
export DOCKER_BUILDKIT=1
docker build --progress=plain -t mi-app:debug .
```

### 4.16.4 Build con --progress=plain para ver TODO

El formato fancy de BuildKit oculta información crítica. Para debugging, siempre
usa `--progress=plain`:

```bash
# El output fancy esconde líneas, timing y errores intermedios:
docker build -t mi-app .

# Con --progress=plain ves CADA comando, su output completo, y timing:
docker build --progress=plain -t mi-app:debug . 2>&1 | tee build.log

# El output incluye:
# - Momento exacto en que empieza y termina cada capa
# - Tiempo de ejecución de cada instrucción
# - STDOUT y STDERR sin truncar
# - Hashes de caché exactos que se están usando
# - Mensajes de error de herramientas dentro del contenedor que el formato
#   fancy normalmente te oculta

# Ejemplo de output de --progress=plain:
# #1 [internal] load .dockerignore
# #1 transferring context: 2.34kB done
# #1 DONE 0.0s
# #2 [internal] load build definition from Dockerfile
# #2 transferring dockerfile: 1.2kB done
# #2 DONE 0.0s
# #3 [deps 1/3] FROM docker.io/library/node:20-alpine@sha256:abc...
# #3 resolve docker.io/library/node:20-alpine@sha256:abc... done
# #3 DONE 0.3s
# #4 [deps 2/3] WORKDIR /app
# #4 DONE 0.1s
# #5 [deps 3/3] RUN npm ci
# #5 1.234 npm WARN deprecated ...        <-- ESTO no se ve en modo fancy
# #5 5.678 added 342 packages in 4.3s
# #5 DONE 5.7s
```

### 4.16.5 Debugging de BuildKit secrets: ¿llegó el secreto?

El segundo bug más frustrante después de "funciona en mi PC/CI": pasaste el secreto
pero el build falla con autenticación. Verificar si el secreto realmente se montó:

```bash
# El Dockerfile:
# RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci

# Técnica 1: Insertar un paso de debugging temporal en el Dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    sh -c '
    echo "=== Verificando secreto ===" && \
    echo "Existe: $(test -f /root/.npmrc && echo SI || echo NO)" && \
    echo "Tamaño: $(wc -c < /root/.npmrc 2>/dev/null || echo 0) bytes" && \
    echo "Contenido (primeros 100 chars): $(head -c 100 /root/.npmrc 2>/dev/null || echo VACIO)" && \
    echo "============================" && \
    npm ci
'

# Técnica 2: Construir con --progress=plain y buscar errores de autenticación
docker build --secret id=npmrc,src=$HOME/.npmrc --progress=plain -t test . 2>&1 | tee build.log
grep -i "auth\|401\|403\|E401\|E403\|token\|npmrc" build.log

# Técnica 3: Verificar que el archivo fuente del secreto existe y tiene contenido
ls -la $HOME/.npmrc
wc -l $HOME/.npmrc
# Si está vacío o no existe, BuildKit montará un archivo vacío sin error

# Técnica 4: Probar con un secreto inline para validar que la sintaxis funciona
echo "//registry.npmjs.org/:_authToken=test-token" > /tmp/test-npmrc
docker build --secret id=npmrc,src=/tmp/test-npmrc -t test .
# Si este build funciona (llega a instalar hasta el punto del token),
# el mecanismo de secretos funciona — el problema es el contenido del archivo
```

**Diagnóstico de fallos comunes con secrets:**

```bash
# Problema: el secreto se monta como directorio en vez de archivo
# Causa: el archivo fuente es un directorio
ls -ld ~/mi-secreto
# drwxr-xr-x  2 user  wheel  64 20 May 10:30 /Users/user/mi-secreto
# ERROR: BuildKit crea un directorio en el target, no monta el archivo

# Problema: permisos del archivo secreto
# BuildKit respeta los permisos del archivo fuente
ls -la ~/mi-secreto
# -rw-------  1 user  wheel  256 20 May 10:30 /Users/user/mi-secreto
# El proceso en el contenedor debe poder leerlo (el build corre como root por defecto,
# así que esto raramente es un problema, pero si usas USER en el builder...)

# Problema: ID del secreto incorrecto
# En el Dockerfile: id=npmrc
# En la línea de comandos: --secret id=npm-token,src=...
# El ID debe coincidir exactamente
```

### 4.16.6 Inspección de builds fallidos: docker buildx imagetools

`docker buildx imagetools inspect` es la herramienta para inspeccionar imágenes en
registries remotos sin descargarlas. Revela metadatos que no obtienes con `docker inspect`.

```bash
# Ver todas las arquitecturas y plataformas de una imagen multi-arch
docker buildx imagetools inspect node:20-alpine

# Output típico:
# Name:      docker.io/library/node:20-alpine
# MediaType: application/vnd.oci.image.index.v1+json
# Digest:    sha256:abc123...
#
# Manifests:
#   Name:        docker.io/library/node:20-alpine@sha256:def456...
#   MediaType:   application/vnd.oci.image.manifest.v2+json
#   Platform:    linux/amd64
#
#   Name:        docker.io/library/node:20-alpine@sha256:ghi789...
#   MediaType:   application/vnd.oci.image.manifest.v2+json
#   Platform:    linux/arm64
#
#   Name:        docker.io/library/node:20-alpine@sha256:jkl012...
#   MediaType:   application/vnd.oci.image.manifest.v2+json
#   Platform:    linux/arm/v7

# Ver información de procedencia y SBOM (si están atestiguadas)
docker buildx imagetools inspect node:20-alpine \
  --format '{{json .Provenance}}' | jq .

# Ver las anotaciones del manifiesto
docker buildx imagetools inspect mi-app:latest \
  --format '{{json .Manifest.Annotations}}' | jq .

# Ver el historial de build completo (si está embedido en la imagen)
docker buildx imagetools inspect mi-app:latest \
  --format '{{json .BuildHistory}}' | jq .
```

### 4.16.7 Comparación de imágenes con container-diff

`container-diff` de Google es un diff semántico entre dos imágenes. Te dice
exactamente qué archivos, paquetes y metadatos cambiaron:

```bash
# Instalar container-diff
brew install container-diff

# Comparar dos imágenes:
container-diff diff mi-app:v1.0 mi-app:v1.1 --type=file

# Output:
# -----File-----
# These entries have been added to mi-app:v1.1:
#   FILE                 SIZE
#   /app/dist/server.js  245K
#   /app/config/v2.json  1.2K
#
# These entries have been deleted from mi-app:v1.1:
#   FILE                 SIZE
#   /app/dist/old.js     240K
#
# These entries have been changed between mi-app:v1.0 and mi-app:v1.1:
#   FILE                 SIZE1  SIZE2
#   /app/package.json    1.2K   1.4K

# Comparar paquetes instalados (APT, PIP, NPM, etc.):
container-diff diff mi-app:v1.0 mi-app:v1.1 --type=pip --type=apt --type=node

# Comparar metadatos de la imagen:
container-diff diff mi-app:v1.0 mi-app:v1.1 --type=history

# Comparar tamaño de capas:
container-diff diff mi-app:v1.0 mi-app:v1.1 --type=size
```

### 4.16.8 Técnicas avanzadas de debugging en runtime

Cuando el problema no está en el build sino en cómo se comporta la imagen:

```bash
# 1. Entrypoint bypass: ejecutar un shell en vez de la aplicación
docker run --rm -it --entrypoint sh mi-app:debug

# 2. Sobrescribir CMD para evitar que la app arranque y así inspeccionar
docker run --rm -it mi-app:debug /bin/sh

# 3. Inspeccionar variables de entorno resueltas (incluyendo las de la imagen)
docker run --rm --entrypoint env mi-app:debug

# 4. Ver el filesystem completo de la imagen sin iniciar el entrypoint
docker run --rm --entrypoint ls mi-app:debug -laR /app

# 5. Simular el entrypoint real paso a paso
docker run --rm -it --entrypoint sh mi-app:debug
# Dentro del contenedor:
/app $ ./docker-entrypoint.sh
# → Ahora puedes ver el output del entrypoint en tiempo real

# 6. Capturar la señal de parada para debugging
docker run --rm --name debug-app mi-app:debug &
PID=$!
sleep 3
# Enviar SIGTERM manualmente y ver cómo reacciona la app:
kill -TERM $PID
sleep 10  # Observar si muere antes del grace period
```

**Strace dentro del contenedor para debugging profundo:**

```bash
# Ver qué syscalls está haciendo el proceso (requiere --cap-add=SYS_PTRACE)
docker run --rm --cap-add=SYS_PTRACE --name debug mi-app:debug &
PID=$(docker inspect debug --format '{{.State.Pid}}')
sudo strace -p $PID -f -tt -o /tmp/strace.log
# En otra terminal:
docker stop debug
# Analizar:
grep SIGTERM /tmp/strace.log
# Esto muestra si el proceso recibió SIGTERM y qué hizo con él
```

### 4.16.9 Checklist de debugging sistemático

Cuando una imagen falla, ejecuta esto en orden:

```
□ 1. docker build --progress=plain --no-cache → ¿falla el build o el runtime?
□ 2. docker build --target <stage> → ¿en qué etapa exacta falla?
□ 3. docker run --rm -it <stage-image> sh → inspeccionar el filesystem
□ 4. docker image history --no-trunc → ver comandos de cada capa
□ 5. dive <imagen> → inspección visual de capas
□ 6. Comparar contextos de build entre entornos
□ 7. Comparar build args y variables de entorno
□ 8. Verificar arquitectura: docker image inspect --format '{{.Os}}/{{.Architecture}}'
□ 9. Verificar buildx version y compatibilidad de BuildKit
□ 10. Para secretos: insertar paso de verificación temporal
```

---

## 4.17 Supply chain con Docker: SBOM, procedencia, confianza

El Dockerfile produce una imagen, pero ¿cómo demuestras que esa imagen contiene
exactamente lo que crees que contiene? ¿Cómo sabes que nadie inyectó código malicioso
entre el `git push` y el `docker push`? La respuesta es la **cadena de suministro**
(supply chain) para imágenes de contenedor: SBOM, procedencia, firmas y atestaciones
que convierten una imagen opaca en un artefacto verificable criptográficamente.

### 4.17.1 SBOM: Software Bill of Materials

Un SBOM es una lista estructurada de **todos** los componentes de software que
componen tu imagen: paquetes del sistema operativo, librerías de lenguaje, binarios
copiados. Es el equivalente a una etiqueta de ingredientes, pero verificable
automáticamente.

**¿Por qué necesitas un SBOM?**

- **Cumplimiento normativo**: US Executive Order 14028 exige SBOM para software
  vendido al gobierno federal. ISO 27001, SOC 2 y PCI DSS están siguiendo el mismo
  camino.
- **Gestión de vulnerabilidades**: cuando se descubre un CVE, el SBOM te dice en
  segundos si tu imagen está afectada, sin tener que escanear.
- **Licencias**: saber qué licencias tienen tus dependencias transitivas evita
  demandas por incumplimiento de licencias copyleft como GPL.
- **Trazabilidad**: si un paquete malicioso se cuela en tu supply chain, el SBOM
  registra exactamente qué versión estaba presente.

**Generar un SBOM con syft:**

```bash
# Instalar syft (macOS)
brew install syft

# Generar SBOM para una imagen local
syft mi-app:latest -o spdx-json > sbom.spdx.json

# Generar SBOM para una imagen en un registry
syft registry.example.com/mi-app:v1.2.3 -o cyclonedx-json > sbom.cdx.json

# Generar SBOM en formato tabla (legible para humanos)
syft mi-app:latest -o table

# Output de ejemplo:
# NAME                    VERSION              TYPE
# alpine-baselayout       3.4.3-r2             apk
# alpine-baselayout-data  3.4.3-r2             apk
# busybox                 1.36.1-r29           apk
# busybox-binsh           1.36.1-r29           apk
# libcrypto3              3.1.7-r1             apk
# libssl3                 3.1.7-r1             apk
# nodejs                  20.11.1-r0           apk
# ...paquetes npm...
# express                 4.18.2               npm
# typescript              5.4.5                npm
```

**Formatos de SBOM:**

| Formato | Estándar | Uso principal |
|---------|----------|--------------|
| SPDX 2.3 | ISO/IEC 5962:2021 | Cumplimiento normativo, gobierno |
| CycloneDX 1.5 | OWASP | Seguridad, integración con herramientas |
| Syft JSON | Anchore | Formato nativo de syft, máxima información |
| SPDX tag-value | ISO/IEC 5962:2021 | Legible por humanos, procesable por máquinas |

```bash
# Comparativa: diferentes formatos muestran diferente granularidad
syft mi-app:latest -o spdx-json | jq '.packages | length'    # 847 paquetes
syft mi-app:latest -o cyclonedx-json | jq '.components | length'  # 312 paquetes
# CycloneDX puede agrupar paquetes de forma diferente a SPDX
```

**Firmar un SBOM con cosign:**

El SBOM sin firma es solo un archivo JSON que cualquiera podría haber generado.
La firma criptográfica ata el SBOM a una identidad verificable.

```bash
# Generar par de claves para firmar (una vez)
cosign generate-key-pair
# Crea: cosign.key (privada) y cosign.pub (pública)

# Firmar la imagen Y adjuntar el SBOM firmado como atestación
cosign attest --key cosign.key \
  --predicate sbom.spdx.json \
  --type spdxjson \
  mi-app:latest

# Verificar la atestación (el SBOM está firmado y ligado a la imagen)
cosign verify-attestation --key cosign.pub \
  --type spdxjson \
  mi-app:latest | jq -r '.payload' | base64 -d | jq .

# Esto verifica:
# 1. Que la firma es válida (la clave privada corresponde a cosign.pub)
# 2. Que el SBOM no ha sido modificado desde que se firmó
# 3. Que el SBOM está ligado al digest exacto de la imagen (no se puede reutilizar
#    con otra imagen)
```

**Política de admission con SBOM:**

En Kubernetes, puedes exigir que toda imagen desplegada tenga un SBOM firmado:

```yaml
# Ejemplo de ClusterImagePolicy con Sigstore Policy Controller
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-sbom
spec:
  images:
    - glob: "registry.example.com/production/**"
  authorities:
    - key:
        data: |
          -----BEGIN PUBLIC KEY-----
          MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
          -----END PUBLIC KEY-----
      attestations:
        - predicateType: "https://spdx.dev/Document"
          name: "sbom"
          policy:
            type: "cue"
            data: |
              predicate: {
                // SBOM debe contener al menos los paquetes principales
              }
```

**Integración de SBOM en CI/CD:**

```bash
# En tu pipeline:
#!/bin/bash
set -e

IMAGE="registry.example.com/mi-app:${CI_COMMIT_SHA}"

# 1. Construir la imagen
docker build -t "$IMAGE" .

# 2. Generar SBOM
syft "$IMAGE" -o spdx-json > sbom-${CI_COMMIT_SHA}.spdx.json

# 3. Firmar el SBOM y adjuntarlo a la imagen
cosign attest --key env://COSIGN_PRIVATE_KEY \
  --predicate sbom-${CI_COMMIT_SHA}.spdx.json \
  --type spdxjson \
  "$IMAGE"

# 4. Opcional: subir SBOM al registry como artefacto separado
oras push "registry.example.com/sbom/mi-app:${CI_COMMIT_SHA}" \
  sbom-${CI_COMMIT_SHA}.spdx.json:application/spdx+json

# 5. Push de la imagen
docker push "$IMAGE"
```

### 4.17.2 SLSA Framework: niveles de madurez de supply chain

SLSA (Supply-chain Levels for Software Artifacts, pronunciado "salsa") es un
framework de Google que define 4 niveles de madurez para la cadena de suministro
de software. Cada nivel añade garantías de integridad y procedencia.

**Nivel 1: Build Scripted (builds automatizados)**

El requisito mínimo: los builds no son manuales, hay un script o pipeline que
ejecuta el build de forma repetible.

```yaml
# .github/workflows/build.yml — Nivel 1
name: Build Image
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t mi-app:${{ github.sha }} .
      - name: Push
        run: |
          docker tag mi-app:${{ github.sha }} registry.example.com/mi-app:${{ github.sha }}
          docker push registry.example.com/mi-app:${{ github.sha }}
```

**Limitación Nivel 1:** No hay forma de verificar que el build en CI corresponde
al código fuente que se subió. Un atacante que comprometa el runner de CI puede
modificar el build sin que el código fuente lo refleje.

**Nivel 2: Build Service (CI/CD gestionado)**

El build se ejecuta en un servicio gestionado (GitHub Actions, GitLab CI, Tekton)
con versiones fijas de las herramientas. Se mantiene un registro de builds.

```yaml
# Nivel 2: versiones fijas, toolchain declarado
name: SLSA Level 2 Build
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      # Versiones fijas SHA para inmutabilidad
      - uses: docker/login-action@v3.0.0
        with:
          registry: registry.example.com
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_PASS }}
      - uses: docker/build-push-action@v5.1.0
        with:
          context: .
          push: true
          tags: registry.example.com/mi-app:${{ github.sha }}
          provenance: true  # Nivel 2+: generar metadata de procedencia
```

**Nivel 3: Auditable (procedencia verificable)**

Los artefactos vienen con **procedencia firmada** que demuestra:
- Qué código fuente se usó (repositorio, commit exacto)
- Qué receta de build (Dockerfile, configuración)
- Quién ejecutó el build (identidad del builder)
- Cuándo se ejecutó el build

La procedencia permite responder preguntas como: "¿Esta imagen de producción se
construyó realmente del commit `abc123` en la rama `main`?"

**Nivel 4: Hermético (builds reproducibles)**

Dos personas diferentes, en máquinas diferentes, ejecutando la misma receta de build
con el mismo código fuente, obtienen **exactamente el mismo hash de imagen**. Esto
requiere:

- Todas las dependencias declaradas con hash (no tags flotantes)
- Compiladores deterministas
- Sin timestamps en los artefactos
- Sin acceso a red durante el build (todo pre-declarado)
- Mismo contexto de build bit a bit

```dockerfile
# Ejemplo de Dockerfile con aspiraciones de Nivel 4
FROM debian:stable-slim@sha256:abc123def456...
# Digest exacto, no tag

# Todas las dependencias con versión exacta
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl=8.5.0-1~bpo12+1 \
    ca-certificates=20230311

# Compilación determinista sin timestamps
RUN gcc -static \
    -Wl,--build-id=none \
    -fno-ident \
    -Wdate-time \
    -D__DATE__=\"redacted\" \
    -D__TIME__=\"redacted\" \
    -o /app/binary \
    main.c
```

### 4.17.3 Procedencia con BuildKit

BuildKit puede generar procedencia automáticamente como metadata adjunta a la
imagen. Esta procedencia sigue el estándar SLSA v1.0.

```bash
# Construir con procedencia (requiere BuildKit y buildx v0.10+)
docker buildx build \
  --provenance=true \
  --sbom=true \
  -t registry.example.com/mi-app:v1.2.3 \
  --push \
  .

# --provenance=true genera una atestación de procedencia SLSA
# --sbom=true genera un SBOM integrado en la imagen

# Verificar la procedencia generada
docker buildx imagetools inspect registry.example.com/mi-app:v1.2.3 \
  --format '{{json .Provenance}}' | jq .
```

**Estructura de la procedencia generada:**

```json
{
  "_type": "https://in-toto.io/Statement/v0.1",
  "subject": [
    {
      "name": "pkg:docker/registry.example.com/mi-app@v1.2.3",
      "digest": {
        "sha256": "abc123def456..."
      }
    }
  ],
  "predicateType": "https://slsa.dev/provenance/v1.0",
  "predicate": {
    "buildDefinition": {
      "buildType": "https://mobyproject.org/buildkit@v1",
      "externalParameters": {
        "source": "https://github.com/empresa/mi-app.git",
        "ref": "refs/tags/v1.2.3",
        "dockerfile": "Dockerfile.production"
      },
      "resolvedDependencies": [
        { "uri": "docker.io/library/node:20-alpine@sha256:...", "digest": { "sha256": "..." } }
      ]
    },
    "runDetails": {
      "builder": {
        "id": "https://github.com/actions/runner"
      },
      "metadata": {
        "invocationId": "https://github.com/empresa/mi-app/actions/runs/1234567890/attempts/1"
      }
    }
  }
}
```

**Firmar la procedencia:**

```bash
# La procedencia de BuildKit viene sin firma por defecto.
# Puedes firmarla con cosign o configurar buildx para que firme automáticamente.

# Opción 1: Firmar después del build con cosign
cosign attest --key cosign.key \
  --type slsaprovenance \
  registry.example.com/mi-app:v1.2.3

# Opción 2: BuildKit firma automáticamente (buildx v0.14+)
docker buildx build \
  --provenance=true \
  --attest type=provenance,mode=max \
  --sbom=true \
  --push \
  -t registry.example.com/mi-app:v1.2.3 \
  .
```

### 4.17.4 SLSA GitHub Actions Generator

El proyecto `slsa-github-generator` implementa SLSA Nivel 3 automáticamente para
GitHub Actions. Es la forma más rápida de alcanzar SLSA 3 sin implementar la
infraestructura desde cero.

```yaml
# .github/workflows/slsa-build.yml
name: SLSA Build and Push

on:
  push:
    tags:
      - 'v*'

permissions: read-all

jobs:
  # Job 1: Construir la imagen con procedencia SLSA L3
  build:
    outputs:
      image: ${{ steps.image.outputs.image }}
      digest: ${{ steps.image.outputs.digest }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}

  # Job 2: Generar procedencia SLSA firmada
  provenance:
    needs: build
    permissions:
      actions: read
      id-token: write
      contents: read
    uses: slsa-framework/slsa-github-generator/.github/workflows/
          generator_container_slsa3.yml@v2.0.0
    with:
      image: ${{ needs.build.outputs.image }}
      digest: ${{ needs.build.outputs.digest }}
    secrets:
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}

  # Job 3: Verificar la procedencia generada
  verify:
    needs: [build, provenance]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: slsa-framework/slsa-verifier/actions/installer@v2
      - run: |
          slsa-verifier verify-image \
            "${{ needs.build.outputs.image }}" \
            --source-uri github.com/${{ github.repository }} \
            --source-tag ${{ github.ref_name }}
```

**Resultado después del pipeline:**

```
ghcr.io/empresa/mi-app@sha256:abc... ← imagen
ghcr.io/empresa/mi-app:provenance    ← procedencia SLSA L3 firmada
ghcr.io/empresa/mi-app:sbom          ← SBOM firmado

# Verificar antes de desplegar:
cosign verify-attestation \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github.com/empresa/mi-app/.github/workflows/' \
  ghcr.io/empresa/mi-app@sha256:abc...
```

### 4.17.5 Firma de imágenes con cosign: cadena de confianza completa

La cadena de confianza completa se compone de tres elementos firmados:

1. **Imagen firmada**: `cosign sign` → demuestra que la imagen fue aprobada por una
   clave específica.
2. **SBOM firmado**: `cosign attest --type spdxjson` → demuestra qué contiene la
   imagen exactamente.
3. **Procedencia firmada**: `cosign attest --type slsaprovenance` → demuestra de
   dónde viene la imagen.

```bash
# Paso 1: Firmar la imagen misma
cosign sign --key cosign.key mi-app:v1.2.3
# Esto crea una firma en el registry: mi-app:sha256-abc123.sig

# Paso 2: Verificar la firma de la imagen
cosign verify --key cosign.pub mi-app:v1.2.3
# Output:
# Verification for mi-app:v1.2.3 --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key

# Paso 3: Firmar y adjuntar SBOM
cosign attest --key cosign.key \
  --predicate sbom.spdx.json \
  --type spdxjson \
  mi-app:v1.2.3

# Paso 4: Firmar y adjuntar procedencia
cosign attest --key cosign.key \
  --predicate provenance.json \
  --type slsaprovenance \
  mi-app:v1.2.3

# Paso 5: Verificación completa antes de desplegar
IMAGE="registry.example.com/mi-app@sha256:definitive-digest"

# 5a: Verificar firma de imagen
cosign verify --key cosign.pub "$IMAGE"

# 5b: Verificar atestación SBOM
cosign verify-attestation --key cosign.pub \
  --type spdxjson \
  "$IMAGE" | jq -r '.payload' | base64 -d | jq .

# 5c: Verificar atestación de procedencia
cosign verify-attestation --key cosign.pub \
  --type slsaprovenance \
  "$IMAGE"

# Si los tres pasos pasan, la imagen tiene una cadena de confianza completa.
```

**Keyless signing con OIDC:**

No necesitas gestionar claves privadas. cosign puede firmar usando tu identidad
OIDC (GitHub, Google, Microsoft) a través de Fulcio (CA de Sigstore):

```bash
# Firmar con identidad OIDC de GitHub Actions (en CI):
cosign sign \
  --oidc-issuer https://token.actions.githubusercontent.com \
  --identity-token $ACTIONS_ID_TOKEN_REQUEST_TOKEN \
  ghcr.io/empresa/mi-app:v1.2.3

# La firma está ligada a tu identidad de GitHub Actions, no a una clave estática.
# Verificación con la identidad OIDC:
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github.com/empresa/mi-app/.github/workflows/build.yml@refs/tags/v.*' \
  ghcr.io/empresa/mi-app:v1.2.3
```

### 4.17.6 Política de admission en Kubernetes

Con la cadena de confianza completa, puedes bloquear el despliegue de cualquier
imagen que no tenga firma, procedencia y SBOM verificados:

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: production-image-policy
spec:
  images:
    - glob: "registry.example.com/production/**"
  authorities:
    # Verificar firma de la imagen (cosign sign)
    - key:
        data: |
          -----BEGIN PUBLIC KEY-----
          MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEhPj1...
          -----END PUBLIC KEY-----
      # Verificar atestación de procedencia
      attestations:
        - predicateType: "https://slsa.dev/provenance/v1.0"
          name: "provenance"
      # Verificar atestación SBOM
        - predicateType: "https://spdx.dev/Document"
          name: "sbom"
          policy:
            type: "cue"
            data: |
              // No permitir imágenes con licencias GPL (o cualquier política definida)
              import "list"
              predicate: {
                packages: [...{licenseDeclared: string}]
                violations: [ for p in packages if p.licenseDeclared =~ "GPL" {p.name}]
                valid: len(violations) == 0
              }
```

### 4.17.7 Reproducibilidad: el santo grial del Nivel 4

La reproducibilidad (dos builds del mismo código producen el mismo hash) es
extremadamente difícil con Dockerfiles estándar. Aquí las técnicas para acercarse:

```dockerfile
# syntax=docker/dockerfile:1
# Dockerfile con alta reproducibilidad

# 1. Imagen base con digest exacto
FROM debian:stable-slim@sha256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0

# 2. No usar apt-get upgrade (no determinista)
# RUN apt-get update && apt-get upgrade -y    ← NUNCA

# 3. Versiones exactas con = en apt
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl=8.5.0-1~bpo12+1 \
    ca-certificates=20230311 \
    && rm -rf /var/lib/apt/lists/*

# 4. Fijar timezone (los paquetes pueden tener timestamps de la zona horaria)
ENV TZ=UTC
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 5. Eliminar timestamps de archivos copiados con COPY
#    COPY preserva los timestamps del host → no determinista
#    Solución: reescribir timestamps al copiar
COPY . /app/
RUN find /app -exec touch -t 202401010000.00 {} \; 2>/dev/null || true

# 6. Compilación determinista (Go)
RUN CGO_ENABLED=0 go build \
    -ldflags="-w -s -buildid=" \
    -trimpath \
    -mod=readonly \
    -o /app/binary .

# 7. Construir con SOURCE_DATE_EPOCH
ARG SOURCE_DATE_EPOCH=0
ENV SOURCE_DATE_EPOCH=${SOURCE_DATE_EPOCH}

# 8. Modo hermético con BuildKit
# RUN --network=none go build ...

# Verificar reproducibilidad:
# Build 1:
docker build -t app:v1 --build-arg SOURCE_DATE_EPOCH=0 .
HASH1=$(docker inspect app:v1 --format='{{.Id}}')

# Build 2:
docker build -t app:v2 --build-arg SOURCE_DATE_EPOCH=0 .
HASH2=$(docker inspect app:v2 --format='{{.Id}}')

echo "HASH1: $HASH1"
echo "HASH2: $HASH2"
# Si son iguales, el build es reproducible.
```

**Checklist de SLSA para equipos:**

```
Nivel 1 □ Builds en CI, no manuales
Nivel 2 □ CI/CD gestionado (GitHub Actions/GitLab CI)
         □ Versiones fijas de acciones y herramientas
         □ Registro de builds (logs de CI)
Nivel 3 □ Procedencia firmada en cada build
         □ Verificación de procedencia antes de desplegar
         □ SBOM generado y firmado
         □ Política de admission en Kubernetes
Nivel 4 □ Builds reproducibles (mismo hash)
         □ Builds herméticos (sin red externa)
         □ Dependencias cacheadas y verificadas con hash
         □ Dos revisores para cada cambio en el build
```

---

## 4.18 Builds multi-plataforma avanzados

Construir imágenes que funcionen en AMD64 (PCs, servidores x86) y ARM64 (Mac
Apple Silicon, AWS Graviton, Raspberry Pi) es una necesidad cada vez más común.
El ecosistema de herramientas ha madurado, pero las trampas de rendimiento y
compatibilidad requieren conocimiento experto.

### 4.18.1 Arquitectura del builder: docker-container driver

El driver por defecto (`docker`) no soporta builds multi-plataforma. Necesitas
crear un builder con el driver `docker-container`:

```bash
# Listar builders actuales
docker buildx ls

# Output típico:
# NAME/NODE       DRIVER/ENDPOINT             STATUS  BUILDKIT   PLATFORMS
# default *       docker
#   default       default                     running v0.13.2    linux/amd64, linux/arm64
# El builder default solo soporta la arquitectura nativa

# Crear un builder multi-plataforma
docker buildx create \
  --name multiarch \
  --driver docker-container \
  --driver-opt network=host \
  --bootstrap \
  --use

# Verificar capacidades
docker buildx inspect --bootstrap multiarch

# Output:
# Name:          multiarch
# Driver:        docker-container
# Last Activity: 2024-05-20 10:30:00 +0000 UTC
#
# Nodes:
# Name:          multiarch0
# Endpoint:      unix:///var/run/docker.sock
# Status:        running
# BuildKit:      v0.14.0
# Platforms:     linux/amd64, linux/arm64, linux/arm/v7, linux/arm/v6,
#                linux/386, linux/ppc64le, linux/s390x
```

**¿Por qué `docker-container` y no `docker`?**

El driver `docker` usa el BuildKit integrado en el daemon de Docker, que está
limitado a la arquitectura del host. El driver `docker-container` ejecuta BuildKit
en un contenedor separado con soporte completo para emulación QEMU y builders
remotos.

### 4.18.2 QEMU emulation vs builders nativos

QEMU permite ejecutar binarios de una arquitectura en otra (por ejemplo, compilar
para ARM64 desde una máquina AMD64). El precio: **rendimiento masivamente degradado**.

**Benchmarks comparativos (compilación de binario Go ~50MB):**

```
Escenario                    | Tiempo   | Factor
-----------------------------|----------|--------
AMD64 nativo en AMD64        | 23s      | 1x
ARM64 nativo en ARM64        | 25s      | 1.1x
ARM64 vía QEMU en AMD64      | 890s     | 38.7x ← 38 VECES MÁS LENTO
ARM64 vía QEMU en AMD64 (Go) | 145s     | 6.3x  ← Go es más liviano
AMD64 vía QEMU en ARM64      | 720s     | 31.3x
```

```bash
# Construir para ARM64 usando QEMU (emulación, en tu laptop AMD64):
docker buildx build \
  --platform linux/arm64 \
  --builder multiarch \
  -t mi-app:arm64-emulated \
  .

# El mismo build en una Mac M1/M2/M3 (ARM64 nativo):
docker buildx build \
  --platform linux/arm64 \
  -t mi-app:arm64-native \
  .
# Mucho más rápido porque no hay emulación

# Construir para múltiples plataformas simultáneamente:
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --builder multiarch \
  -t registry.example.com/mi-app:latest \
  --push \
  .
```

**Cuándo QEMU es aceptable y cuándo NO:**

| Escenario | QEMU viable? | Nota |
|-----------|-------------|------|
| Copiar archivos (COPY, ADD) | Sí | No involucra CPU |
| Ejecutar scripts shell (echo, mkdir) | Sí | Operaciones triviales |
| Instalar paquetes (apt/apk) | Tal vez | La extracción de archivos es rápida, los scripts de post-instalación son lentos |
| Compilar C/C++/Rust | NO | Horas en vez de minutos |
| Compilar Go | Tal vez | 6x a 38x más lento, aceptable para builds pequeños |
| npm ci / pip install | NO | Miles de paquetes = horas |
| Tests unitarios | NO | Inviable |

### 4.18.3 Estrategia: un builder nativo por arquitectura

La solución de producción para multi-plataforma es tener runners/builders nativos
para cada arquitectura y combinarlos con buildx:

```bash
# En un servidor AMD64 (runner de CI):
docker buildx create --name multiarch-amd64 \
  --driver docker-container \
  --platform linux/amd64

# En un servidor ARM64 (AWS Graviton, Mac M1 en CI):
docker buildx create --name multiarch-arm64 \
  --driver docker-container \
  --platform linux/arm64

# Conectar ambos builders en uno solo (desde cualquier nodo):
docker buildx create --name multiarch \
  --append --node arm64-builder \
  --platform linux/arm64 \
  ssh://user@arm64-host

docker buildx create --name multiarch \
  --append --node amd64-builder \
  --platform linux/amd64 \
  --driver docker-container

# Ahora buildx distribuye el trabajo al builder nativo de cada plataforma
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --builder multiarch \
  -t registry.example.com/mi-app:latest \
  --push \
  .
```

**Configuración de CI/CD con runners por arquitectura:**

```yaml
# GitHub Actions con runners self-hosted por arquitectura
# .github/workflows/build-multiarch.yml
name: Multi-Arch Build

on:
  push:
    tags: ['v*']

jobs:
  # Job 1: Construir AMD64 en un runner x86_64
  amd64-build:
    runs-on: [self-hosted, linux, amd64]
    steps:
      - uses: actions/checkout@v4
      - name: Build AMD64
        run: |
          docker buildx build \
            --platform linux/amd64 \
            -t registry.example.com/mi-app:${{ github.ref_name }}-amd64 \
            --push .

  # Job 2: Construir ARM64 en un runner aarch64 (AWS Graviton)
  arm64-build:
    runs-on: [self-hosted, linux, arm64]
    steps:
      - uses: actions/checkout@v4
      - name: Build ARM64
        run: |
          docker buildx build \
            --platform linux/arm64 \
            -t registry.example.com/mi-app:${{ github.ref_name }}-arm64 \
            --push .

  # Job 3: Crear el manifest list que une ambas arquitecturas
  manifest:
    needs: [amd64-build, arm64-build]
    runs-on: ubuntu-latest
    steps:
      - name: Create multi-arch manifest
        run: |
          docker manifest create \
            registry.example.com/mi-app:${{ github.ref_name }} \
            registry.example.com/mi-app:${{ github.ref_name }}-amd64 \
            registry.example.com/mi-app:${{ github.ref_name }}-arm64
          docker manifest push \
            registry.example.com/mi-app:${{ github.ref_name }}
```

### 4.18.4 Node.js native modules: compilar para cada plataforma

Los módulos nativos de Node.js (bcrypt, sharp, node-sass, protobufjs, better-sqlite3)
compilan código C/C++ durante `npm install`. Cuando construyes para múltiples
arquitecturas, necesitas compilarlos para cada una por separado.

```dockerfile
# syntax=docker/dockerfile:1

# El Dockerfile usa TARGETARCH para condicionales de compilación
FROM node:20-alpine AS builder

ARG TARGETARCH

WORKDIR /app
COPY package.json package-lock.json ./

# Instalar herramientas de compilación necesarias para módulos nativos
RUN apk add --no-cache python3 make g++

# npm install compila los módulos nativos para la arquitectura actual
RUN npm ci

# Para módulos especialmente problemáticos como sharp:
# Sharp pre-compila binarios por plataforma. A veces necesitas forzar la descarga:
ENV npm_config_platform=linux
ENV npm_config_arch=${TARGETARCH}
ENV npm_config_sharp_libvips_local_prebuilds=false

# Verificar que los módulos nativos se compilaron correctamente:
RUN node -e "require('bcrypt')" && \
    node -e "require('sharp')" && \
    echo "Módulos nativos OK para ${TARGETARCH}"

# ----------------------------------------------------------------

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
USER 1000:1000
CMD ["node", "dist/index.js"]
```

**El problema específico de `bcrypt`:**

```dockerfile
# bcrypt requiere node-gyp para compilar código C++ contra OpenSSL.
# En Alpine, necesitas las headers de desarrollo durante la compilación.

FROM node:20-alpine AS builder
RUN apk add --no-cache --virtual .build-deps \
    python3 \
    make \
    g++ \
    openssl-dev

COPY package.json package-lock.json ./
RUN npm ci  # Compila bcrypt nativo

# Limpiar dependencias de build en el builder (no llegan al runtime)
RUN apk del .build-deps

FROM node:20-alpine AS runtime
# Solo necesitas las librerías runtime de OpenSSL, no las headers
RUN apk add --no-cache openssl
COPY --from=builder /app/node_modules ./node_modules
COPY . .
```

**El problema específico de `sharp`:**

```bash
# Sharp tiene binarios pre-compilados para cada plataforma.
# El problema: cuando construyes en CI (AMD64) para ARM64 con QEMU,
# el proceso de instalación detecta AMD64 y descarga binarios AMD64,
# que luego fallan al ejecutarse en ARM64.

# Solución: forzar la arquitectura objetivo durante npm install:
docker buildx build \
  --platform linux/arm64 \
  --build-arg TARGETARCH=arm64 \
  -t mi-app:arm64 \
  .

# Y en el Dockerfile:
ARG TARGETARCH
ENV npm_config_arch=${TARGETARCH}
ENV npm_config_target_arch=${TARGETARCH}
RUN npm ci
```

### 4.18.5 Python wheels nativos: compilar extensiones C

Las extensiones C de Python (numpy, scipy, psycopg2, cryptography, Pillow) deben
compilarse o descargarse como wheels precompilados para la arquitectura objetivo.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder

ARG TARGETARCH
ARG TARGETVARIANT

# Instalar herramientas de compilación
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    python3-dev \
    && rm -rf /var/lib/apt/lists/*

# Descargar wheels para la plataforma correcta
# Usar --platform para forzar la arquitectura de los wheels
RUN pip install --no-cache-dir \
    --platform linux_${TARGETARCH} \
    --target=/install \
    numpy \
    psycopg2-binary \
    cryptography

# Verificar que los .so son para la arquitectura correcta
RUN file /install/numpy/core/_multiarray_umath.cpython-*.so \
    | grep "${TARGETARCH}"

# ----------------------------------------------------------------

FROM python:3.12-slim AS runtime
COPY --from=builder /install /usr/local/lib/python3.12/site-packages/
COPY . /app
WORKDIR /app
CMD ["python", "app.py"]
```

**Problema común: wheels no disponibles para ARM64:**

```bash
# Algunos paquetes Python no tienen wheels precompilados para ARM64.
# Debes compilarlos desde el código fuente (pip intentará hacerlo automáticamente
# pero necesita build-essential y headers C).

# Estrategia de fallback: compilar desde fuente en el builder
FROM python:3.12-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    python3-dev \
    libpq-dev \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# --no-binary fuerza compilación desde fuente para TODOS los paquetes
# (más lento pero funciona en cualquier arquitectura)
RUN pip wheel --no-cache-dir --wheel-dir=/wheels \
    --no-binary :all: \
    numpy psycopg2 cryptography

# Para builds multi-arch, puedes ser selectivo:
ARG TARGETARCH
RUN if [ "$TARGETARCH" = "arm64" ]; then \
      pip wheel --no-cache-dir --wheel-dir=/wheels \
        --no-binary psycopg2 \
        psycopg2; \
    else \
      pip wheel --no-cache-dir --wheel-dir=/wheels \
        psycopg2; \
    fi
```

### 4.18.6 Java: compilar una vez, ejecutar en todas

La gran ventaja de la JVM en el mundo multi-plataforma: el bytecode es portable.
Compilas una vez en cualquier arquitectura y ejecutas en todas las que tengan JVM.

```dockerfile
# El build de Maven es independiente de la arquitectura objetivo
FROM maven:3.9-eclipse-temurin-21-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

# Extraer capas para optimizar el cacheo en todas las arquitecturas
RUN java -Djarmode=layertools -jar target/*.jar extract \
      --destination /app/extracted

# ----------------------------------------------------------------
# El runtime stage NO especifica plataforma — buildx lo resuelve

FROM eclipse-temurin:21-jre-alpine AS runtime
# Esta imagen existe para amd64 Y arm64 en Docker Hub.
# BuildKit selecciona automáticamente la variante correcta.
WORKDIR /app
COPY --from=builder /app/extracted/dependencies/ ./
COPY --from=builder /app/extracted/spring-boot-loader/ ./
COPY --from=builder /app/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/extracted/application/ ./
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Excepción: librerías nativas en Java (JNI, Netty Epoll):**

```dockerfile
# Netty tiene transporte nativo (epoll en Linux) que se compila por plataforma.
# Asegúrate de que la dependencia nativa esté declarada para todas las plataformas
# o usa el clasificador correcto en Maven.

# pom.xml:
# <dependency>
#   <groupId>io.netty</groupId>
#   <artifactId>netty-transport-native-epoll</artifactId>
#   <classifier>linux-x86_64</classifier>   <!-- AMD64 -->
# </dependency>
# <dependency>
#   <groupId>io.netty</groupId>
#   <artifactId>netty-transport-native-epoll</artifactId>
#   <classifier>linux-aarch_64</classifier>  <!-- ARM64 -->
# </dependency>

# Dockerfile para incluir ambos clasificadores:
FROM maven:3.9-eclipse-temurin-21 AS builder
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:copy-dependencies \
      -DincludeArtifactIds=netty-transport-native-epoll \
      -Dclassifier=linux-x86_64 \
      -DoutputDirectory=/app/native/amd64
RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:copy-dependencies \
      -DincludeArtifactIds=netty-transport-native-epoll \
      -Dclassifier=linux-aarch_64 \
      -DoutputDirectory=/app/native/arm64

FROM eclipse-temurin:21-jre AS runtime
ARG TARGETARCH
# Seleccionar los binarios nativos correctos según la arquitectura
COPY --from=builder /app/native/${TARGETARCH} /app/lib/
```

### 4.18.7 Construir para plataformas específicas con condicionales

BuildKit expone variables `TARGETARCH`, `TARGETOS`, `TARGETVARIANT` que permiten
condicionales en el Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1

ARG TARGETARCH
ARG TARGETOS
ARG TARGETVARIANT

FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder

RUN echo "Builder ejecutándose en $BUILDPLATFORM, compilando para $TARGETOS/$TARGETARCH"

# Seleccionar compilador o flags según arquitectura
RUN if [ "$TARGETARCH" = "arm64" ]; then \
      echo "Compilando para ARM64 con optimizaciones específicas"; \
      GOARM=8; \
    elif [ "$TARGETARCH" = "amd64" ]; then \
      echo "Compilando para AMD64"; \
    fi

# Compilación condicional que usa las variables nativas de Go
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -o /app/binary .

# Etapa de runtime: BuildKit selecciona la imagen base para la plataforma objetivo
FROM --platform=$TARGETPLATFORM alpine:3.20 AS runtime
# Al especificar --platform=$TARGETPLATFORM, Docker Hub resolverá automáticamente
# la imagen alpine:3.20 para la arquitectura correcta (tiene manifiestos para
# amd64, arm64, arm/v7, etc.)

COPY --from=builder /app/binary /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/binary"]
```

**Variables de plataforma en detalle:**

| Variable | Descripción | Build en AMD64 → target ARM64 |
|----------|-------------|-------------------------------|
| `BUILDPLATFORM` | Plataforma donde se ejecuta el builder | `linux/amd64` |
| `BUILDOS` | SO del builder | `linux` |
| `BUILDARCH` | Arquitectura del builder | `amd64` |
| `BUILDVARIANT` | Variante del builder | `` (vacío para amd64) |
| `TARGETPLATFORM` | Plataforma para la que se construye | `linux/arm64` |
| `TARGETOS` | SO objetivo | `linux` |
| `TARGETARCH` | Arquitectura objetivo | `arm64` |
| `TARGETVARIANT` | Variante objetivo | `v8` para arm64 |

### 4.18.8 Optimizaciones cross-compilation con Rust y Go

**Go: cross-compilation nativa sin QEMU**

Go tiene cross-compilation nativa. No necesitas QEMU para compilar binarios ARM64
desde AMD64:

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder
ARG TARGETARCH
ARG TARGETOS

# Cross-compile nativo de Go: compila para ARM64 en builder AMD64
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -o /app/binary .

FROM --platform=$TARGETPLATFORM scratch
COPY --from=builder /app/binary /binary
ENTRYPOINT ["/binary"]
```

**Rust: cross-compilation con cross**

```dockerfile
# cross compila Rust para arquitecturas objetivo mediante contenedores Docker
# preconfigurados con el toolchain correcto

# Instalar cross y compilar:
cargo install cross
cross build --target aarch64-unknown-linux-musl --release

# Dockerfile que aprovecha cross-compilation de Rust:
FROM rust:1.78-alpine AS builder
RUN rustup target add aarch64-unknown-linux-musl
RUN apt-get update && apt-get install -y musl-tools

ARG TARGETARCH
RUN if [ "$TARGETARCH" = "arm64" ]; then \
      rustup target add aarch64-unknown-linux-musl && \
      cargo build --target aarch64-unknown-linux-musl --release && \
      cp target/aarch64-unknown-linux-musl/release/app /app/binary; \
    else \
      cargo build --target x86_64-unknown-linux-musl --release && \
      cp target/x86_64-unknown-linux-musl/release/app /app/binary; \
    fi

FROM scratch
COPY --from=builder /app/binary /binary
ENTRYPOINT ["/binary"]
```

### 4.18.9 Comandos útiles para multi-plataforma

```bash
# Crear y empujar un manifest list manualmente
docker manifest create registry.example.com/mi-app:latest \
  registry.example.com/mi-app:latest-amd64 \
  registry.example.com/mi-app:latest-arm64

docker manifest annotate registry.example.com/mi-app:latest \
  registry.example.com/mi-app:latest-arm64 --arch arm64 --os linux

docker manifest push registry.example.com/mi-app:latest

# Inspeccionar un manifest list
docker manifest inspect registry.example.com/mi-app:latest

# Ver qué arquitecturas tiene una imagen
docker buildx imagetools inspect registry.example.com/mi-app:latest \
  --format '{{range .Manifest.Manifests}}{{.Platform.OS}}/{{.Platform.Architecture}}{{"\n"}}{{end}}'

# Probar que la imagen multi-arch funciona
docker run --rm --platform linux/amd64 registry.example.com/mi-app:latest
docker run --rm --platform linux/arm64 registry.example.com/mi-app:latest
```

---

## 4.19 Crear un builder enterprise-grade

Una cosa es escribir un Dockerfile decente. Otra es construir un sistema de build
de imágenes de contenedor que escale a una organización con 20 microservicios, 50
ingenieros, y requisitos de compliance. Esta sección detalla la estructura de
repositorio, automatización, linting, testing, escaneo y firma que convierten el
build de imágenes en un proceso industrial.

### 4.19.1 Estructura de repositorio estandarizada

La inconsistencia en la ubicación y formato de los Dockerfiles es el primer enemigo
de la estandarización. Un patrón de proyecto que he iterado en docenas de equipos:

```
mi-servicio/
├── .dockerignore                   # Siempre en la raíz, común a todos los Dockerfiles
├── Dockerfile                      # Producción: slim, multi-stage, no-root
├── Dockerfile.dev                  # Desarrollo: hot reload, debug ports, root OK
├── Dockerfile.test                 # Testing: incluye herramientas de test, coverage
├── Makefile                        # Orquestación de todas las tareas de build
├── Taskfile.yml                    # Alternativa moderna a Makefile (Go Task)
├── .hadolint.yaml                  # Configuración de linter de Dockerfiles
├── docker-compose.yml              # Entorno local
├── docker-compose.ci.yml           # Entorno para CI (testing, integración)
├── build/                          # Scripts y configuraciones auxiliares de build
│   ├── entrypoint.sh
│   ├── healthcheck.sh
│   └── nginx/
│       ├── nginx.conf
│       └── mime.types
├── k8s/                            # Manifiestos de Kubernetes (si aplica)
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── .github/
│   └── workflows/
│       ├── build.yml               # Build + test + push
│       └── security-scan.yml       # Escaneo periódico de CVEs
└── src/                            # Código fuente
```

**Principios de la estructura:**

1. **Un `.dockerignore` canónico por repositorio**. No lo dupliques en subdirectorios.
   La raíz manda.

2. **Dockerfiles separados por propósito**, no por entorno. `Dockerfile.dev` tiene
   herramientas de debugging, mounts de código, puertos de debugger. `Dockerfile`
   (sin sufijo) es el de producción. `Dockerfile.test` tiene dependencias de test
   y comandos de test.

3. **Configuración de build junto al código**. `build/` contiene todo lo que el
   Dockerfile necesita y que no es código fuente (entrypoints, configs de nginx,
   healthchecks).

4. **CI/CD junto al código**. `.github/workflows/` está en el repositorio, visible
   para todo el equipo, versionado con el código.

### 4.19.2 .dockerignore generado automáticamente según el lenguaje

En vez de escribir `.dockerignore` a mano para cada proyecto, mantén un script
que lo genere basado en el lenguaje detectado:

```bash
#!/bin/bash
# generate-dockerignore.sh
# Detecta el lenguaje y genera un .dockerignore optimizado

LANG=${1:-$(detect-language)}

detect-language() {
  if [ -f "package.json" ]; then echo "node"; return; fi
  if [ -f "requirements.txt" ] || [ -f "pyproject.toml" ]; then echo "python"; return; fi
  if [ -f "go.mod" ]; then echo "go"; return; fi
  if [ -f "pom.xml" ] || [ -f "build.gradle" ]; then echo "java"; return; fi
  if [ -f "Cargo.toml" ]; then echo "rust"; return; fi
  echo "unknown"
}

# Cabecera común a todos los proyectos
cat << 'EOF'
# === AUTO-GENERATED .dockerignore ===
# Generado por generate-dockerignore.sh
# Lenguaje detectado: LANG_PLACEHOLDER

# ---- Control de versiones ----
.git/
.gitignore
.gitattributes
.gitmodules

# ---- CI/CD y configuración local ----
.github/
.gitlab-ci.yml
Jenkinsfile
.azure-pipelines/
.circleci/

# ---- Docker ----
.dockerignore
Dockerfile*
docker-compose*.yml
.docker/

# ---- IDE y editores ----
.vscode/
.idea/
*.swp
*.swo
*~
.project
.classpath
.settings/
*.iml

# ---- OS ----
.DS_Store
Thumbs.db
ehthumbs.db
Desktop.ini

# ---- Variables de entorno ----
.env
.env.*
!.env.example

# ---- Logs y temporales ----
*.log
logs/
*.tmp
tmp/
temp/

# ---- Documentación ----
*.md
!README.md
docs/
EOF

# Sección específica del lenguaje
case "$LANG" in
  node)
    cat << 'EOF'

# ---- Node.js ----
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.pnpm-debug.log*

# Build output
dist/
build/
.next/
nuxt/
.output/
.nuxt/

# Testing
coverage/
.nyc_output/
*.test.js
*.spec.js
__tests__/
__mocks__/

# Yarn
.yarn/
.pnp.*
EOF
    ;;
  python)
    cat << 'EOF'

# ---- Python ----
__pycache__/
*.py[cod]
*$py.class
*.egg-info/
.eggs/
dist/
build/
*.whl

# Virtual environment
venv/
.venv/
env/
.env/

# Testing
.pytest_cache/
.coverage
htmlcov/
.tox/
.hypothesis/
EOF
    ;;
  go)
    cat << 'EOF'

# ---- Go ----
*.exe
*.exe~
*.dll
*.so
*.dylib
bin/
vendor/

# Testing
*.test
*.out
*.prof
EOF
    ;;
  java)
    cat << 'EOF'

# ---- Java ----
target/
*.class
*.jar
*.war
*.ear
.gradle/
build/
!gradle-wrapper.jar

# Maven wrapper
.mvn/wrapper/maven-wrapper.jar
EOF
    ;;
esac

# Usar sed para reemplazar el placeholder con el lenguaje real
sed -i '' "s/LANG_PLACEHOLDER/$LANG/" .dockerignore
echo ".dockerignore generado para lenguaje: $LANG"
```

### 4.19.3 Makefile enterprise para Docker builds

Un Makefile es el pegamento que une linting, building, testing, escaneo y
despliegue. Debe ser el punto de entrada único para cualquier operación con
la imagen.

```makefile
# ============================================================
# Makefile para gestión de imágenes Docker (enterprise grade)
# ============================================================

# --- Variables de configuración ---
REGISTRY     ?= registry.example.com
IMAGE_NAME   ?= $(shell basename $$(pwd))
TAG          ?= $(shell git rev-parse --short HEAD 2>/dev/null || echo "dev")
VERSION      ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
FULL_IMAGE   := $(REGISTRY)/$(IMAGE_NAME):$(TAG)
PLATFORMS    ?= linux/amd64,linux/arm64
BUILDKIT_PROGRESS ?= auto

# Variables para CI (se sobrescriben en el pipeline)
CI            ?= false
PUSH          ?= false
SIGN          ?= false

# Dockerfiles por propósito
DOCKERFILE         := Dockerfile
DOCKERFILE_DEV     := Dockerfile.dev
DOCKERFILE_TEST    := Dockerfile.test

# Archivos de configuración
HADOLINT_CONFIG    := .hadolint.yaml
CST_CONFIG         := container-structure-test.yaml
TRIVY_SEVERITY     := HIGH,CRITICAL

# Colores para output
GREEN  := \033[0;32m
RED    := \033[0;31m
YELLOW := \033[0;33m
NC     := \033[0m # No Color

.PHONY: help
help: ## Muestra esta ayuda
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "$(GREEN)%-20s$(NC) %s\n", $$1, $$2}'

# ============================================================
# LINTING
# ============================================================

.PHONY: lint
lint: lint-dockerfile lint-shell lint-dockerignore ## Linting completo

.PHONY: lint-dockerfile
lint-dockerfile: ## Lint del Dockerfile con hadolint
	@echo "$(YELLOW)[LINT] Analizando $(DOCKERFILE)...$(NC)"
	@if command -v hadolint > /dev/null 2>&1; then \
		hadolint --config $(HADOLINT_CONFIG) $(DOCKERFILE); \
	elif docker image inspect hadolint/hadolint > /dev/null 2>&1 || docker pull hadolint/hadolint > /dev/null 2>&1; then \
		docker run --rm -i \
			-v $(shell pwd)/$(HADOLINT_CONFIG):/root/.config/hadolint.yaml \
			hadolint/hadolint < $(DOCKERFILE); \
	else \
		echo "$(RED)[ERROR] hadolint no encontrado. Instala con: brew install hadolint$(NC)"; \
		exit 1; \
	fi
	@echo "$(GREEN)[LINT] Dockerfile OK$(NC)"

.PHONY: lint-shell
lint-shell: ## Lint de los scripts shell (entrypoint, healthcheck)
	@echo "$(YELLOW)[LINT] Analizando scripts shell...$(NC)"
	@if command -v shellcheck > /dev/null 2>&1; then \
		find build/ -name "*.sh" -exec shellcheck {} +; \
	else \
		echo "$(YELLOW)[WARN] shellcheck no instalado. Saltando.$(NC)"; \
	fi

.PHONY: lint-dockerignore
lint-dockerignore: ## Verificar que .dockerignore existe y tiene contenido
	@if [ ! -f .dockerignore ]; then \
		echo "$(RED)[ERROR] .dockerignore no existe$(NC)"; \
		exit 1; \
	fi
	@if [ ! -s .dockerignore ]; then \
		echo "$(RED)[ERROR] .dockerignore está vacío$(NC)"; \
		exit 1; \
	fi
	@echo "$(GREEN)[LINT] .dockerignore OK$(NC)"

# ============================================================
# BUILD
# ============================================================

.PHONY: build
build: lint ## Build de la imagen de producción (con linting previo)
	@echo "$(YELLOW)[BUILD] Construyendo $(FULL_IMAGE)...$(NC)"
	DOCKER_BUILDKIT=1 docker buildx build \
		--progress=$(BUILDKIT_PROGRESS) \
		--build-arg VERSION=$(VERSION) \
		--build-arg BUILD_DATE=$(shell date -u +%Y-%m-%dT%H:%M:%SZ) \
		--build-arg VCS_REF=$(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown") \
		--cache-from type=registry,ref=$(REGISTRY)/cache/$(IMAGE_NAME):latest \
		--cache-to type=inline \
		-f $(DOCKERFILE) \
		-t $(FULL_IMAGE) \
		-t $(REGISTRY)/$(IMAGE_NAME):latest \
		.
	@echo "$(GREEN)[BUILD] Imagen construida: $(FULL_IMAGE)$(NC)"

.PHONY: build-dev
build-dev: ## Build de la imagen de desarrollo
	@echo "$(YELLOW)[BUILD- DEV] Construyendo imagen de desarrollo...$(NC)"
	DOCKER_BUILDKIT=1 docker build \
		-f $(DOCKERFILE_DEV) \
		-t $(IMAGE_NAME):dev \
		.
	@echo "$(GREEN)[BUILD- DEV] OK$(NC)"

.PHONY: build-multiarch
build-multiarch: lint ## Build multi-plataforma
	@echo "$(YELLOW)[BUILD] Construyendo para $(PLATFORMS)...$(NC)"
	DOCKER_BUILDKIT=1 docker buildx build \
		--platform $(PLATFORMS) \
		--build-arg VERSION=$(VERSION) \
		--build-arg BUILD_DATE=$(shell date -u +%Y-%m-%dT%H:%M:%SZ) \
		--cache-from type=registry,ref=$(REGISTRY)/cache/$(IMAGE_NAME):latest \
		--cache-to type=registry,ref=$(REGISTRY)/cache/$(IMAGE_NAME):latest,mode=max \
		-f $(DOCKERFILE) \
		-t $(FULL_IMAGE) \
		-t $(REGISTRY)/$(IMAGE_NAME):latest \
		--push \
		.
	@echo "$(GREEN)[BUILD] Multi-arch build completado$(NC)"

.PHONY: build-no-cache
build-no-cache: ## Build sin caché (para verificar desde cero)
	DOCKER_BUILDKIT=1 docker build --no-cache \
		--progress=plain \
		-f $(DOCKERFILE) \
		-t $(FULL_IMAGE) \
		.

# ============================================================
# TESTING
# ============================================================

.PHONY: test
test: test-structure test-security ## Testing completo

.PHONY: test-structure
test-structure: build ## Test estructural de la imagen (container-structure-test)
	@echo "$(YELLOW)[TEST] Ejecutando tests estructurales...$(NC)"
	@if [ -f $(CST_CONFIG) ]; then \
		docker run --rm \
			-v /var/run/docker.sock:/var/run/docker.sock \
			-v $(shell pwd)/$(CST_CONFIG):/test-config.yaml \
			gcr.io/gcp-runtimes/container-structure-test:v1.16.0 test \
			--image $(FULL_IMAGE) \
			--config /test-config.yaml; \
	else \
		echo "$(YELLOW)[TEST] $(CST_CONFIG) no encontrado. Saltando tests estructurales.$(NC)"; \
	fi

.PHONY: test-security
test-security: build ## Escaneo de vulnerabilidades con Trivy
	@echo "$(YELLOW)[SEC] Escaneando $(FULL_IMAGE) con Trivy...$(NC)"
	@if command -v trivy > /dev/null 2>&1; then \
		trivy image --severity $(TRIVY_SEVERITY) \
			--exit-code 1 \
			--no-progress \
			$(FULL_IMAGE); \
	else \
		docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
			aquasec/trivy:latest image --severity $(TRIVY_SEVERITY) \
			--exit-code 1 \
			--no-progress \
			$(FULL_IMAGE); \
	fi
	@echo "$(GREEN)[SEC] Sin vulnerabilidades $(TRIVY_SEVERITY)$(NC)"

.PHONY: test-integration
test-integration: build ## Tests de integración con docker-compose
	@echo "$(YELLOW)[TEST] Ejecutando tests de integración...$(NC)"
	docker compose -f docker-compose.yml -f docker-compose.ci.yml up --build -d
	# Esperar que los servicios estén healthy
	sleep 10
	# Ejecutar tests contra los servicios
	npm run test:integration || (docker compose down; exit 1)
	docker compose down

# ============================================================
# SCAN
# ============================================================

.PHONY: scan
scan: scan-cves scan-secrets scan-sbom ## Escaneo completo

.PHONY: scan-cves
scan-cves: build ## Escaneo de CVEs
	trivy image --format json --output trivy-$(IMAGE_NAME)-$(TAG).json $(FULL_IMAGE)
	@echo "$(GREEN)[SCAN] Reporte CVEs: trivy-$(IMAGE_NAME)-$(TAG).json$(NC)"

.PHONY: scan-secrets
scan-secrets: build ## Buscar secretos filtrados en la imagen
	@echo "$(YELLOW)[SCAN] Buscando secretos en $(FULL_IMAGE)...$(NC)"
	@if command -v ggshield > /dev/null 2>&1; then \
		ggshield secret scan docker $(FULL_IMAGE); \
	else \
		echo "$(YELLOW)[WARN] ggshield no instalado. Saltando.$(NC)"; \
	fi

.PHONY: scan-sbom
scan-sbom: build ## Generar SBOM
	@echo "$(YELLOW)[SBOM] Generando SBOM para $(FULL_IMAGE)...$(NC)"
	syft $(FULL_IMAGE) -o spdx-json > sbom-$(IMAGE_NAME)-$(TAG).spdx.json
	syft $(FULL_IMAGE) -o cyclonedx-json > sbom-$(IMAGE_NAME)-$(TAG).cdx.json
	@echo "$(GREEN)[SBOM] SBOM generado$(NC)"

# ============================================================
# PUSH AND SIGN
# ============================================================

.PHONY: push
push: test ## Push de la imagen al registry
	@echo "$(YELLOW)[PUSH] Subiendo $(FULL_IMAGE)...$(NC)"
	docker push $(FULL_IMAGE)
	@echo "$(GREEN)[PUSH] Imagen subida: $(FULL_IMAGE)$(NC)"

.PHONY: sign
sign: push ## Firmar la imagen con cosign
	@echo "$(YELLOW)[SIGN] Firmando $(FULL_IMAGE)...$(NC)"
	cosign sign --key env://COSIGN_PRIVATE_KEY $(FULL_IMAGE)
	@echo "$(GREEN)[SIGN] Imagen firmada$(NC)"

.PHONY: verify-signature
verify-signature: ## Verificar firma de la imagen
	cosign verify \
		--key cosign.pub \
		$(FULL_IMAGE)

.PHONY: release
release: lint build test scan push sign ## Pipeline completo de release
	@echo "$(GREEN)=== RELEASE COMPLETADO ===$(NC)"
	@echo "$(GREEN)Imagen: $(FULL_IMAGE)$(NC)"

# ============================================================
# UTILIDADES
# ============================================================

.PHONY: shell
shell: build ## Abrir shell en la imagen construida
	docker run --rm -it --entrypoint sh $(FULL_IMAGE)

.PHONY: inspect
inspect: build ## Inspeccionar la imagen con dive
	dive $(FULL_IMAGE)

.PHONY: size
size: build ## Mostrar tamaño de la imagen
	@docker image inspect $(FULL_IMAGE) --format='Tamaño: {{.Size}}' | \
		awk '{printf "%.2f MB\n", $$2/1048576}'

.PHONY: history
history: ## Ver historial de capas
	docker image history --human --no-trunc $(FULL_IMAGE)

.PHONY: clean
clean: ## Limpiar imágenes locales y caché de build
	docker image rm $(FULL_IMAGE) $(REGISTRY)/$(IMAGE_NAME):latest 2>/dev/null || true
	docker builder prune -af
	rm -f sbom-*.json trivy-*.json

.PHONY: all
all: release ## Alias para release (build + test + scan + push + sign)
```

### 4.19.4 Container Structure Test: verificar que la imagen es correcta

`container-structure-test` valida la imagen contra un contrato declarativo:
archivos esperados, comandos exitosos, metadatos correctos.

```yaml
# container-structure-test.yaml
schemaVersion: "2.0.0"

# Metadata tests: verificar labels, entrypoint, user, etc.
metadataTest:
  # El usuario no-root
  exposedPorts: ["8080"]
  volumes: ["/data"]
  entrypoint: ["./docker-entrypoint.sh"]
  cmd: ["node", "dist/index.js"]
  workdir: "/app"
  user: "1000:1000"

# File existence tests: verificar que los archivos correctos existen
fileExistenceTests:
  - name: 'Entrypoint script exists and is executable'
    path: '/app/docker-entrypoint.sh'
    shouldExist: true
    permissions: '-rwxr-xr-x'
    uid: 1000
    gid: 1000

  - name: 'Application bundle exists'
    path: '/app/dist/index.js'
    shouldExist: true

  - name: 'Node modules exist'
    path: '/app/node_modules'
    shouldExist: true

  - name: 'No shell history (security)'
    path: '/root/.bash_history'
    shouldExist: false

  - name: 'No .git directory leaked'
    path: '/app/.git'
    shouldExist: false

  - name: 'No credentials files'
    path: '/app/.env'
    shouldExist: false

  - name: 'No npm debug log'
    path: '/app/npm-debug.log'
    shouldExist: false

# File content tests: verificar contenido de archivos
fileContentTests:
  - name: 'Node.js version in package.json'
    path: '/app/package.json'
    expectedContents: ['.*"node".*']

# Command tests: verificar que comandos funcionan
commandTests:
  - name: 'Node is available and correct version'
    command: 'node'
    args: ['--version']
    expectedOutput: ['v20\.*']

  - name: 'Application health check responds'
    command: 'wget'
    args: ['--quiet', '--tries=1', '--spider', 'http://localhost:3000/health']
    # Este test requiere que el contenedor esté corriendo (se levanta automáticamente)
    exitCode: 0

  - name: 'Working directory is /app'
    command: 'pwd'
    expectedOutput: ['/app']

  - name: 'Running as non-root user'
    command: 'whoami'
    expectedOutput: ['appuser']

  - name: 'No root-owned files in app directory'
    setup: [['sh', '-c', 'find /app -user root -print']]
    command: ''
    expectedOutput: ['']
    # Si find no encuentra nada, el output es vacío → el test pasa

# License tests
licenseTests:
  - debian: true
    files: []
```

**Ejecutar los tests:**

```bash
# Ejecutar container-structure-test
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(pwd)/container-structure-test.yaml:/test-config.yaml \
  gcr.io/gcp-runtimes/container-structure-test:v1.16.0 test \
  --image mi-app:latest \
  --config /test-config.yaml

# Output exitoso:
# =============================================
# ====== Test file: /test-config.yaml ======
# =============================================
# === RUN: File Existence: Entrypoint script exists
# --- PASS
# === RUN: File Existence: Application bundle exists
# --- PASS
# === RUN: Command Test: Node.js version
# --- PASS
# === RUN: Metadata Test
# --- PASS
#
# RESULTS:
# PASS: 14
# FAIL: 0
# Total: 14
```

### 4.19.5 Integración con pre-commit hooks

Automatizar el linting de Dockerfiles antes de cada commit evita que el código
incorrecto llegue siquiera al CI:

```yaml
# .pre-commit-config.yaml
repos:
  # Linting de Dockerfiles con hadolint
  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint-docker
        args:
          - --config
          - .hadolint.yaml
        files: Dockerfile.*

  # Formateo y validación de shell scripts (entrypoint, healthcheck)
  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.10.0.1
    hooks:
      - id: shellcheck
        files: 'build/.*\.sh$'

  # Verificar que .dockerignore existe y no está vacío
  - repo: local
    hooks:
      - id: check-dockerignore
        name: Verify .dockerignore exists
        entry: bash -c 'test -s .dockerignore || (echo "ERROR: .dockerignore is missing or empty"; exit 1)'
        language: system
        files: ''
        pass_filenames: false

  # Validar formato YAML de docker-compose
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.1.0
    hooks:
      - id: prettier
        files: 'docker-compose.*\.ya?ml$'
        types: [yaml]

  # Verificar que no se filtren secretos
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
```

### 4.19.6 Configuración de hadolint

hadolint permite personalizar reglas. No todas las reglas aplican a todos los
proyectos. Configúralo explícitamente:

```yaml
# .hadolint.yaml
# Configuración de hadolint para proyectos de producción

# Ignorar reglas específicas con justificación documentada
ignored:
  # DL3008: "Pin versions in apt-get install"
  # En algunos proyectos usamos imágenes base con paquetes ya instalados
  # en versiones conocidas. Justificación: tenemos escaneo de CVE en CI.
  # - DL3008

  # DL3059: "Multiple consecutive RUN instructions"
  # Permitido en multi-stage donde cada RUN tiene propósito semántico claro
  # - DL3059

# Reglas con umbral personalizado
override:
  # DL4006: Set pipefail. Nivel: error
  # - DL4006

# Reglas adicionales habilitadas por defecto
trustedRegistries:
  - docker.io
  - gcr.io
  - ghcr.io
  - registry.example.com

# Reglas de seguridad estrictas (nivel error)
strict-labels: true
require-label:
  - 'org.opencontainers.image.title'
  - 'org.opencontainers.image.version'
  - 'org.opencontainers.image.source'
```

### 4.19.7 Integración completa en CI/CD

El pipeline final que orquesta todo:

```yaml
# .github/workflows/docker-full-pipeline.yml
name: Docker Enterprise Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============================================================
  # Job 1: Linting (rápido, falla temprano)
  # ============================================================
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          config: .hadolint.yaml

  # ============================================================
  # Job 2: Build y tests estructurales
  # ============================================================
  build-and-test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true  # Cargar en daemon local para tests
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Container Structure Test
        uses: plexsystems/container-structure-test-action@v0.3.0
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:test
          config: container-structure-test.yaml

  # ============================================================
  # Job 3: Escaneo de seguridad
  # ============================================================
  security-scan:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trivy vulnerability scan
        uses: aquasecurity/trivy-action@v0.20.0
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:test
          format: 'sarif'
          output: 'trivy-results.sarif'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Upload SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Generate SBOM
        uses: anchore/sbom-action@v0.15.10
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:test
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json

  # ============================================================
  # Job 4: Push y firma (solo en main)
  # ============================================================
  release:
    if: github.ref == 'refs/heads/main'
    needs: [security-scan]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          provenance: mode=max
          sbom: true
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Sign image with cosign
        run: |
          cosign sign \
            --oidc-issuer https://token.actions.githubusercontent.com \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}

      - name: Verify signature
        run: |
          cosign verify \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            --certificate-identity-regexp '^https://github.com/${{ github.repository }}/.github/workflows/' \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
```

### 4.19.8 Métricas y auditoría del proceso de build

Un build enterprise-grade no está completo sin visibilidad. Instrumenta tu pipeline
para capturar métricas:

```bash
#!/bin/bash
# build-metrics.sh
# Recolecta métricas de cada build para detectar degradación

IMAGE=$1

echo "=== Build Metrics Report ==="

# Tamaño de la imagen
SIZE_BYTES=$(docker image inspect "$IMAGE" --format='{{.Size}}')
SIZE_MB=$(echo "scale=2; $SIZE_BYTES / 1048576" | bc)
echo "Image Size: ${SIZE_MB} MB"

# Número de capas
LAYERS=$(docker image inspect "$IMAGE" --format='{{len .RootFS.Layers}}')
echo "Layers: $LAYERS"

# Número de CVEs (requiere trivy)
CVE_COUNT=$(trivy image --severity HIGH,CRITICAL --format json "$IMAGE" 2>/dev/null | jq '.Results[].Vulnerabilities | length' | paste -sd+ | bc)
echo "High/Critical CVEs: ${CVE_COUNT:-0}"

# Eficiencia de la imagen (bytes útiles vs totales, via dive)
EFFICIENCY=$(dive "$IMAGE" --ci 2>/dev/null | grep 'efficiency' | awk '{print $NF}' | tr -d '%')
echo "Efficiency: ${EFFICIENCY:-N/A}%"

# Tiempo de build (desde el último build log)
# ... implementación específica del CI ...

echo "=== End Report ==="
```

### 4.19.9 Checklist final del builder enterprise-grade

```
□ Estructura de proyecto estandarizada con Dockerfiles por propósito
□ .dockerignore mantenido automáticamente o documentado
□ Makefile/Taskfile como interfaz única de build
□ Linting de Dockerfiles con hadolint + .hadolint.yaml
□ Linting de scripts shell con shellcheck
□ Testing estructural con container-structure-test.yaml
□ Escaneo de vulnerabilidades con Trivy en CI
□ Generación de SBOM con syft
□ Firmado de imágenes con cosign
□ Procedencia con BuildKit attestations
□ Pre-commit hooks para linting automático
□ Pipeline de CI/CD con etapas: lint → build → test → scan → sign → push
□ Métricas de build: tamaño, capas, CVEs, eficiencia
□ Caché remota compartida entre builds y entre desarrolladores
```

---

← [Capítulo anterior](capitulo-03-imagenes.md) | [Inicio](README.md) | [Capítulo siguiente →](capitulo-05-volumenes.md)
