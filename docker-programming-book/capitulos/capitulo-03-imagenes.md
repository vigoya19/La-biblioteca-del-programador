# Capítulo 3: Imágenes Docker — La Piedra Angular de la Contenerización

> *"Una imagen no es una foto. Es un plano. Y de ese plano nacen todos los contenedores."*

---

## Índice

1. [Anatomía de una imagen Docker](#1-anatomía-de-una-imagen-docker)
2. [Sistema de capas (layers) en profundidad](#2-sistema-de-capas-layers-en-profundidad)
3. [Docker Hub](#3-docker-hub)
4. [Registries privados](#4-registries-privados)
5. [Gestión de imágenes](#5-gestión-de-imágenes)
6. [Transporte y backup de imágenes](#6-transporte-y-backup-de-imágenes)
7. [Firmado y verificación](#7-firmado-y-verificación)
8. [Laboratorio: Estrategia de versionado para un proyecto real](#8-laboratorio-estrategia-de-versionado-para-un-proyecto-real)

---

## 1. Anatomía de una imagen Docker

### 1.1 La imagen NO es un solo archivo

Muchos principiantes imaginan una imagen Docker como un `.iso`, un `.tar.gz` o un binario monolítico. Nada más lejos de la realidad.

Una imagen Docker es una **estructura compuesta** formada por tres componentes fundamentales:

```
┌─────────────────────────────────────────────────────────┐
│                    IMAGEN DOCKER                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  MANIFEST    │  │   CONFIG     │  │   LAYERS     │  │
│  │              │  │              │  │              │  │
│  │  - schemaVer │  │  - ENV       │  │  ┌────────┐  │  │
│  │  - mediaType │  │  - CMD       │  │  │ Layer  │  │  │
│  │  - config    │  │  - ENTRYPOINT│  │  │ sha256 │  │  │
│  │  - layers[]  │  │  - EXPOSE    │  │  │  diff  │  │  │
│  │              │  │  - VOLUME    │  │  └────────┘  │  │
│  │              │  │  - WORKDIR   │  │  ┌────────┐  │  │
│  │              │  │  - USER      │  │  │ Layer  │  │  │
│  │              │  │  - ...       │  │  │ sha256 │  │  │
│  │              │  │              │  │  │  diff  │  │  │
│  │              │  │              │  │  └────────┘  │  │
│  │              │  │              │  │  ┌────────┐  │  │
│  │              │  │              │  │  │ Layer  │  │  │
│  │              │  │              │  │  │ sha256 │  │  │
│  │              │  │              │  │  │  diff  │  │  │
│  │              │  │              │  │  └────────┘  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

Cada uno de estos tres componentes reside en el registry como un **blob independiente**, identificado por su hash SHA256. Cuando ejecutas `docker pull`, lo que realmente descargas son estos blobs. Ninguno de ellos es una "imagen" por sí solo; la imagen es la suma de las partes.

### 1.2 El Manifest: El índice maestro

El manifest es el **documento JSON** que describe qué capas componen una imagen, para qué arquitectura está construida y dónde encontrar cada pieza. Es el punto de entrada para cualquier operación de pull o push.

#### Anatomía de un manifest (Image Manifest v2, schema 2)

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 7023,
    "digest": "sha256:b5b2d2e4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4"
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 3265430,
      "digest": "sha256:abc123def456abc123def456abc123def456abc123def456abc123def456"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 456789,
      "digest": "sha256:def789abc123def789abc123def789abc123def789abc123def789abc123"
    }
  ]
}
```

Cada campo tiene una función precisa:

| Campo | Significado |
|-------|-------------|
| `schemaVersion` | Versión del formato de manifest. `2` es el estándar moderno (desde Docker 1.10). |
| `mediaType` | Tipo MIME que identifica este documento como un manifest de imagen OCI/Docker. |
| `config.digest` | Hash SHA256 del blob de configuración. Es el "puntero" al ADN de la imagen. |
| `config.size` | Tamaño en bytes del blob de configuración (antes de comprimir). |
| `layers[].digest` | Hash SHA256 de cada capa del filesystem (comprimida como tar.gz). |
| `layers[].size` | Tamaño comprimido de cada capa en bytes. |
| `layers[].mediaType` | Formato de compresión. Normalmente `tar.gzip`; Docker 23+ soporta `tar.zstd`. |
| `layers[].urls` | (Opcional) URLs alternativas para descargar la capa (usado en registries distribuidos). |

#### El orden de las capas en el manifest

El orden en el array `layers` ES el orden de apilamiento: la primera capa es la BASE (la más baja en el filesystem), y la última es la más alta (la última instrucción `RUN`/`COPY`/`ADD` del Dockerfile). Este orden es fundamental para reconstruir el filesystem.

#### Multi-arch manifests (Manifest List u OCI Image Index)

Cuando una imagen soporta múltiples arquitecturas, Docker usa un **Manifest List** (también llamado "fat manifest" o "OCI Image Index"):

```
┌──────────────────────────────────────────────────────────┐
│                   MANIFEST LIST (OCI Image Index)        │
│  (application/vnd.oci.image.index.v1+json)              │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  linux/amd64  ──► Image Manifest ──► Config + Layers  │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  linux/arm64  ──► Image Manifest ──► Config + Layers  │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  linux/arm/v7 ──► Image Manifest ──► Config + Layers  │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  windows/amd64──► Image Manifest ──► Config + Layers  │
│  └─────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

Ejemplo de un manifest list real:

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 2145,
      "digest": "sha256:aaa111bbb222...",
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 2145,
      "digest": "sha256:ccc333ddd444...",
      "platform": {
        "architecture": "arm64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 2100,
      "digest": "sha256:eee555fff666...",
      "platform": {
        "architecture": "arm",
        "os": "linux",
        "variant": "v7"
      }
    }
  ],
  "annotations": {
    "org.opencontainers.image.description": "Nginx multi-arch image"
  }
}
```

Cuando haces `docker pull nginx:alpine` en una máquina ARM64 (como una Mac M1/M2/M3), el daemon:

1. Descarga el manifest list (si el registry lo ofrece).
2. Lee la arquitectura y SO locales (`uname -m` y `uname -s`).
3. Selecciona el manifest que coincide exactamente con `linux/arm64`.
4. Si no hay coincidencia exacta, busca la más cercana (o falla).
5. Descarga el config y las capas correspondientes a ese manifest.

Esto es lo que hace que `nginx:alpine` funcione tanto en tu Mac M1 como en tu servidor x86 sin que tengas que preocuparte por arquitecturas.

#### El flujo de negociación del manifest

```
Cliente Docker                           Registry
     │                                      │
     │  GET /v2/nginx/manifests/alpine      │
     │  Accept: application/vnd.oci.image.  │
     │    index.v1+json,                    │
     │    application/vnd.docker.distribu-  │
     │    tion.manifest.list.v2+json,       │
     │    application/vnd.oci.image.        │
     │    manifest.v1+json,                 │
     │    application/vnd.docker.distribu-  │
     │    tion.manifest.v2+json             │
     │─────────────────────────────────────►│
     │                                      │
     │  ← El registry elige el tipo de      │
     │    mayor prioridad que soporta       │
     │                                      │
     │  200 OK                              │
     │  Content-Type: application/          │
     │    vnd.oci.image.index.v1+json       │
     │  Body: { manifests: [...] }          │
     │◄─────────────────────────────────────│
     │                                      │
     │  El cliente selecciona el manifest   │
     │  que coincide con su plataforma      │
     │                                      │
     │  GET /v2/nginx/manifests/            │
     │    sha256:aaa111bbb...               │
     │  Accept: application/vnd.oci.image.  │
     │    manifest.v1+json                  │
     │─────────────────────────────────────►│
     │                                      │
     │  200 OK                              │
     │  Content-Type: application/          │
     │    vnd.oci.image.manifest.v1+json    │
     │  Body: { config: {...}, layers: [...]}│
     │◄─────────────────────────────────────│
```

#### Inspeccionar manifests

```bash
# Ver el manifest de una imagen (multi-arch)
docker manifest inspect nginx:alpine

# Output resumido (una sola arquitectura, la tuya)
# Ver TODAS las arquitecturas disponibles
docker manifest inspect --verbose nginx:alpine

# Con docker buildx imagetools (más moderno)
docker buildx imagetools inspect nginx:alpine

# Output típico:
# Name:      docker.io/library/nginx:alpine
# MediaType: application/vnd.oci.image.index.v1+json
# Digest:    sha256:aaa111bbb222...
#
# Manifests:
#   Name:        docker.io/library/nginx:alpine@sha256:xxx...
#   MediaType:   application/vnd.oci.image.manifest.v1+json
#   Platform:    linux/amd64
#
#   Name:        docker.io/library/nginx:alpine@sha256:yyy...
#   MediaType:   application/vnd.oci.image.manifest.v1+json
#   Platform:    linux/arm64

# Inspeccionar manualmente desde un docker save
docker pull nginx:alpine
docker save nginx:alpine -o /tmp/nginx.tar
mkdir -p /tmp/nginx-inspect && tar xf /tmp/nginx.tar -C /tmp/nginx-inspect
cat /tmp/nginx-inspect/manifest.json | python3 -m json.tool
```

Dentro del archivo `manifest.json` que extraes de un `docker save` encontrarás:

```json
[
  {
    "Config": "blobs/sha256/a1b2c3d4e5f6...",
    "RepoTags": ["nginx:alpine"],
    "Layers": [
      "blobs/sha256/layer0.tar.gz",
      "blobs/sha256/layer1.tar.gz",
      "blobs/sha256/layer2.tar.gz",
      "blobs/sha256/layer3.tar.gz",
      "blobs/sha256/layer4.tar.gz"
    ]
  }
]
```

### 1.3 El Config: El "ADN" de la imagen

El blob de configuración contiene los **metadatos** que definen cómo se comportará un contenedor creado a partir de esta imagen. Sin este archivo, tendrías un filesystem pero sin saber qué comando ejecutar, qué puertos exponer ni qué variables de entorno definir.

#### Estructura completa del archivo de configuración

```json
{
  "architecture": "amd64",
  "os": "linux",
  "created": "2024-01-15T12:00:00.000000000Z",
  "author": "NGINX Docker Maintainers",
  "config": {
    "Hostname": "",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "ExposedPorts": {
      "80/tcp": {},
      "443/tcp": {}
    },
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "NGINX_VERSION=1.25.3",
      "NJS_VERSION=0.8.2",
      "PKG_RELEASE=1"
    ],
    "Cmd": [
      "nginx",
      "-g",
      "daemon off;"
    ],
    "Healthcheck": {
      "Test": ["CMD", "curl", "-f", "http://localhost/"]
    },
    "ArgsEscaped": true,
    "Image": "",
    "Volumes": {
      "/var/cache/nginx": {}
    },
    "WorkingDir": "/usr/share/nginx/html",
    "Entrypoint": ["/docker-entrypoint.sh"],
    "OnBuild": null,
    "Labels": {
      "maintainer": "NGINX Docker Maintainers <docker-maint@nginx.com>",
      "com.nginx.version": "1.25.3"
    },
    "StopSignal": "SIGQUIT",
    "Shell": null
  },
  "container_config": {
    "Hostname": "",
    "Domainname": "",
    "User": "",
    ...
  },
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:abc123...",
      "sha256:def456...",
      "sha256:ghi789..."
    ]
  },
  "history": [
    {
      "created": "2024-01-15T12:00:00Z",
      "created_by": "ADD file:abc123... /",
      "comment": "",
      "empty_layer": false
    },
    {
      "created": "2024-01-15T12:01:00Z",
      "created_by": "RUN /bin/sh -c set -x && apk add --no-cache nginx",
      "comment": "",
      "empty_layer": false
    },
    {
      "created": "2024-01-15T12:02:00Z",
      "created_by": "CMD [\"nginx\", \"-g\", \"daemon off;\"]",
      "comment": "",
      "empty_layer": true
    }
  ]
}
```

#### Explicación detallada de cada campo del config

| Campo | Origen (Dockerfile) | Propósito | ¿Crea capa? |
|-------|---------------------|-----------|-------------|
| `Env` | `ENV` | Variables de entorno inyectadas en el contenedor. Se heredan en builds multi-stage? No entre stages. | No |
| `Cmd` | `CMD` | Comando por defecto al iniciar el contenedor. Se puede sobrescribir en `docker run`. | No |
| `Entrypoint` | `ENTRYPOINT` | Ejecutable principal no sobrescribible (salvo con `--entrypoint`). `CMD` se pasa como argumento. | No |
| `ExposedPorts` | `EXPOSE` | Puertos que la aplicación escucha. Informativo (no publica). `docker run -P` los publica automáticamente. | No |
| `Volumes` | `VOLUME` | Directorios marcados para montaje externo o volumen. Docker crea un volumen anónimo si no especificas uno. | No |
| `WorkingDir` | `WORKDIR` | Directorio de trabajo al iniciar el contenedor. Si no existe, Docker lo crea. | Sí (si no existe) |
| `User` | `USER` | UID/GID con el que se ejecuta el proceso principal. Por defecto `root` (0:0). | No |
| `Labels` | `LABEL` | Metadatos arbitrarios clave-valor. Útiles para organización, CI/CD, tooling. | No |
| `StopSignal` | `STOPSIGNAL` | Señal enviada al detener el contenedor. Por defecto `SIGTERM`. Nginx usa `SIGQUIT` para graceful shutdown. | No |
| `Healthcheck` | `HEALTHCHECK` | Comando para verificar salud. Docker engine usa esto para `docker ps` STATUS y Swarm/K8s readiness. | No |
| `Shell` | `SHELL` | Shell usada en formato shell de RUN/CMD/ENTRYPOINT. Por defecto `["/bin/sh", "-c"]` en Linux. | No |

#### El campo `rootfs.diff_ids`: La distinción crítica

Aquí hay una sutileza que casi ningún tutorial explica y es FUNDAMENTAL para entender la integridad de las imágenes:

```
┌─────────────────────────────────────────────────────────────────────┐
│                DOS TIPOS DE HASH PARA CADA CAPA                     │
│                                                                     │
│  Manifest JSON:                                                     │
│    layers[0].digest = sha256:AAAA  ← Hash de la capa COMPRIMIDA     │
│                                      (layer.tar.gz)                 │
│                                                                     │
│  Config JSON:                                                       │
│    rootfs.diff_ids[0] = sha256:BBBB  ← Hash de la capa DESCOMPRIMIDA│
│                                        (layer.tar, contenido real)  │
│                                                                     │
│  ¡AAAA ≠ BBBB! Son dos hashes diferentes para la misma capa.       │
│                                                                     │
│  AAAA se calcula sobre los bytes de layer.tar.gz (en tránsito).     │
│  BBBB se calcula sobre los bytes del tar descomprimido (en reposo). │
│                                                                     │
│  Docker verifica AAAA al descargar (integridad en tránsito).        │
│  Docker usa BBBB para content-addressable storage local.            │
└─────────────────────────────────────────────────────────────────────┘
```

Esta distinción es fundamental para entender la verificación de integridad:
- Al hacer `docker pull`, Docker verifica que la capa descargada coincida con `layers[].digest`.
- Al almacenar localmente, Docker descomprime la capa y calcula su hash descomprimido (`diff_id`).
- El `diff_id` DEBE coincidir con el que declara el config (`rootfs.diff_ids[]`).
- Si no coincide, la capa está corrupta y Docker la rechaza.

#### El campo `container_config`

Es un artefacto histórico del proceso de build. Cuando construyes una imagen, Docker crea un contenedor temporal por cada instrucción `RUN`. La configuración de ese contenedor intermedio se guarda en `container_config`. En imágenes modernas, suele ser una copia de `config` o estar vacío. Su existencia se debe a compatibilidad hacia atrás con el formato v1 de imágenes (pre-Docker 1.10).

```bash
# La diferencia entre config y container_config en una imagen construida localmente:
docker image inspect mi-app:v1 --format='Config.Cmd: {{.Config.Cmd}}'
docker image inspect mi-app:v1 --format='ContainerConfig.Cmd: {{.ContainerConfig.Cmd}}'

# En imágenes descargadas (pull), container_config suele ser similar a config
# pero en imágenes construidas localmente puede tener valores diferentes
# que reflejan el último contenedor intermedio durante el build.
```

#### Inspeccionar el config

```bash
# Ver toda la configuración en JSON
docker image inspect nginx:alpine

# Extraer campos específicos con --format (Go templates)
docker image inspect --format='{{json .Config.Env}}' nginx:alpine | python3 -m json.tool
docker image inspect --format='{{.Config.Cmd}}' nginx:alpine
docker image inspect --format='{{range .Config.ExposedPorts}}{{println .}}{{end}}' nginx:alpine

# Ver todos los diff_ids (capas descomprimidas)
docker image inspect --format='{{range .RootFS.Layers}}{{println .}}{{end}}' nginx:alpine

# Ver la arquitectura y SO
docker image inspect --format='{{.Architecture}}/{{.Os}}' nginx:alpine

# Ver el historial de construcción completo
docker image inspect --format='{{range .History}}{{.Created}} | {{.CreatedBy}}{{"\n"}}{{end}}' nginx:alpine

# Ver etiquetas (labels)
docker image inspect --format='{{range $k, $v := .Config.Labels}}{{$k}}={{$v}}{{"\n"}}{{end}}' nginx:alpine
```

### 1.4 Layers (Capas): El filesystem por diffs

Las capas son el corazón del modelo de almacenamiento. Cada capa es un **diff del filesystem**: una colección de archivos que se añaden, modifican o eliminan respecto a la capa inmediatamente inferior. No son copias completas del filesystem.

#### Cómo se construyen las capas desde un Dockerfile

```dockerfile
FROM ubuntu:22.04              # Capa 0: filesystem base de Ubuntu (~77 MB comprimido)
RUN apt-get update             # Capa 1: actualiza índices de paquetes (~5 MB)
RUN apt-get install -y nginx   # Capa 2: instala nginx y dependencias (~15 MB)
COPY index.html /var/www/html/ # Capa 3: añade archivo de la aplicación (~1 KB)
CMD ["nginx", "-g", "daemon off;"] # NO crea capa (solo metadatos)
```

Cada instrucción `RUN`, `COPY` y `ADD` produce una capa nueva. Las instrucciones `CMD`, `ENTRYPOINT`, `ENV`, `EXPOSE`, `VOLUME`, `WORKDIR`, `USER`, `LABEL`, `STOPSIGNAL`, `HEALTHCHECK` y `SHELL` solo modifican los metadatos (el config), **no** crean capas del filesystem.

```
                     ┌─────────────────────────┐
                     │  Capa 3 (COPY)           │
                     │  /var/www/html/index.html│  ← 1 archivo nuevo
                     └───────────┬─────────────┘
                                 │ diff (lo añadido/modificado respecto a capa 2)
                     ┌───────────▼─────────────┐
                     │  Capa 2 (apt install)   │
                     │  /usr/sbin/nginx        │
                     │  /etc/nginx/nginx.conf  │  ← ~500 archivos nuevos
                     │  /var/log/nginx/        │
                     └───────────┬─────────────┘
                                 │ diff
                     ┌───────────▼─────────────┐
                     │  Capa 1 (apt update)    │
                     │  /var/lib/apt/lists/*   │  ← ~200 archivos modificados
                     └───────────┬─────────────┘
                                 │ diff
                     ┌───────────▼─────────────┐
                     │  Capa 0 (ubuntu:22.04)  │
                     │  /bin, /usr, /etc, ...  │  ← ~15000 archivos base
                     └─────────────────────────┘

                     ┌─────────────────────────┐
                     │  VISTA UNIFICADA (merge) │  ← El contenedor ve TODO esto como
                     │  Todos los archivos de   │     un solo filesystem coherente
                     │  todas las capas juntos  │
                     └─────────────────────────┘
```

#### Content-addressable storage: El hash COMO dirección

Docker identifica cada capa por el hash SHA256 de su contenido **descomprimido**. Esto se llama **content-addressable storage**: la dirección (el nombre del archivo/directorio) ES el contenido. Las implicaciones son profundas:

1. **Dos capas idénticas = mismo directorio en disco.** Si dos imágenes producen la misma capa (mismo diff), comparten el mismo espacio físico. No se duplica.
2. **Verificación instantánea.** Para verificar integridad, solo necesitas recalcular el SHA256 y compararlo con el nombre.
3. **Deduplicación automática.** Si descargas una imagen que comparte capas con una que ya tienes, esas capas no se vuelven a descargar.
4. **Inmutabilidad forzada.** Si modificas un solo byte de una capa, su hash cambia, se almacena en un directorio diferente, y se convierte en una capa nueva.

```
/var/lib/docker/overlay2/
├── 8a7c9b3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c/
│   ├── diff/           ← La capa en sí (solo los archivos añadidos aquí)
│   ├── link            ← Nombre corto del ID (el que ves en docker ps)
│   ├── lower           ← Punteros a capas inferiores (si las hay)
│   ├── merged/         ← Vista unificada de esta capa y todas debajo
│   └── work/           ← Directorio de trabajo para operaciones overlay atómicas
├── l/                  ← Enlaces simbólicos con nombres cortos
│   ├── ABC123DEF456... → ../8a7c9b3d.../diff
│   └── GHI789JKL012... → ../7b8c9d0e.../diff
```

> [!NOTE]
> **¿Cómo funciona esto en palabras simples?**
> ¿Cómo hace Docker para apilar 10 capas diferentes y hacer que parezcan un único disco duro coherente? Usa un mecanismo del kernel de Linux llamado **OverlayFS** (a través de su driver de almacenamiento `overlay2`).
> 
> 📽️ **La Analogía del Proyector de Diapositivas de Vidrio**
> Imagina un proyector de transparencias o diapositivas antiguo:
> - **`lower` (Las diapositivas de abajo)**: Son diapositivas de vidrio transparente apiladas en la base. Contienen los archivos del sistema operativo base (como Ubuntu). Son de **solo lectura**: no puedes pintar encima de ellas.
> - **`diff` (Tu diapositiva de arriba)**: Es una diapositiva en blanco que pones en la cima de la pila. Si instalas un programa nuevo (como Node.js), solo se escribe en esta diapositiva superior.
> - **`merged` (La pantalla del proyector)**: Cuando enciendes la luz, todas las diapositivas se fusionan y en la pared ves **una única imagen unificada**. Para ti es un único sistema de archivos en capas, pero por detrás está compuesto por piezas independientes.
> - **`work` (El borrador)**: Es un espacio de trabajo intermedio que usa el proyector para preparar y realizar operaciones atómicas de escritura antes de fijar la diapositiva en la pila.
> 
> **¿Qué pasa al borrar un archivo base?**
> Como no puedes modificar ni raspar una diapositiva inferior de solo lectura, Docker dibuja una "mancha de corrector blanco" (un archivo especial *whiteout* `.wh.nombre`) en tu diapositiva superior (`diff`). Al proyectarse en la pantalla unificada (`merged`), el archivo original parece haber desaparecido por completo, aunque la diapositiva de abajo siga intacta.


### 1.5 Diagrama ASCII: Anatomía completa de la imagen

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          REGISTRY (Docker Hub, ECR, etc.)                   │
│                                                                              │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌────────────────────┐ │
│  │   MANIFEST LIST      │  │   IMAGE MANIFEST     │  │   BLOBS            │ │
│  │   (fat manifest)     │  │   (schema v2)        │  │                    │ │
│  │                      │  │                      │  │  Config.json       │ │
│  │  "manifests": [      │  │  "config": {digest}  │  │  (metadatos: ENV,  │ │
│  │    {platform:        │  │  "layers": [         │  │   CMD, EXPOSE...)  │ │
│  │     linux/amd64,     │  │    {digest: A},      │  │                    │ │
│  │     digest: M1},     │  │    {digest: B},      │  │  Layer_A.tar.gz    │ │
│  │    {platform:        │  │    {digest: C}       │  │  Layer_B.tar.gz    │ │
│  │     linux/arm64,     │  │  ]                   │  │  Layer_C.tar.gz    │ │
│  │     digest: M2}      │  │                      │  │  (comprimidos)     │ │
│  │  ]                   │  │                      │  │                    │ │
│  └──────────────────────┘  └──────────────────────┘  └────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
           │                              │
           │ docker pull                  │
           ▼                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DAEMON DOCKER (disco local)                        │
│                                                                              │
│  /var/lib/docker/                                                           │
│  │                                                                           │
│  ├── image/overlay2/                    ← Metadatos de imágenes y capas      │
│  │   ├── imagedb/content/sha256/        ← Config blobs indexados por ID      │
│  │   │   └── <image-id>                 ← JSON completo (config + rootfs)    │
│  │   ├── layerdb/sha256/<diff-id>/      ← Metadatos por capa (descompr.)    │
│  │   │   ├── cache-id                   ← ID corto de la capa en overlay2/  │
│  │   │   ├── diff                       ← diff-id de esta capa              │
│  │   │   ├── parent                     ← diff-id de la capa padre          │
│  │   │   ├── size                       ← Tamaño en bytes de la capa        │
│  │   │   └── tar-split.json.gz          ← Metadatos de tar para push eficiente│
│  │   └── repositories.json              ← Mapeo: nombre:tag → image-id      │
│  │                                                                           │
│  └── overlay2/                          ← Capas físicas en disco            │
│      ├── <cache-id-1>/                                                       │
│      │   ├── diff/        ← Archivos DE ESTA capa (solo los añadidos aquí)   │
│      │   ├── link         ← Nombre corto del ID                              │
│      │   ├── lower        ← IDs de capas inferiores (enlaces simbólicos)     │
│      │   ├── merged/      ← Vista unificada (lower + diff)                   │
│      │   └── work/        ← Directorio de trabajo overlay                    │
│      ├── <cache-id-2>/                                                       │
│      │   └── ...                                                             │
│      └── l/               ← Enlaces simbólicos (nombres cortos → dirs)       │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  FLUJO DE DATOS:                                                             │
│                                                                              │
│  Registry Blob (comprimido) ──► Descarga ──► Verificación SHA256             │
│         │                                                                     │
│         ▼                                                                     │
│  Descompresión ──► Cálculo diff_id ──► Validación contra rootfs.diff_ids     │
│         │                                                                     │
│         ▼                                                                     │
│  Almacenamiento en overlay2/<cache-id>/diff/                                 │
│         │                                                                     │
│         ▼                                                                     │
│  Registro en layerdb/sha256/<diff-id>/ (size, parent, cache-id)              │
│         │                                                                     │
│         ▼                                                                     │
│  La capa está lista para ser montada por contenedores                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!TIP]
> **¿Por qué hay dos tipos de hashes para la misma capa (Digest de Transporte vs. DiffID Local)?**
> Esta es una duda muy común. ¿Por qué el manifest del registry muestra un hash `digest` y luego en nuestro disco local (`layerdb`) vemos un hash `diff-id` totalmente diferente?
> 
> 📦 **La Analogía del Mueble de IKEA**
> Imagina que compras un gran escritorio de madera por internet:
> 1. **El Digest de Transporte (La caja sellada de IKEA)**: Es el escritorio desarmado, empaquetado plano y muy comprimido dentro de una caja de cartón. La empresa de mensajería le pone una etiqueta de código de barras a la caja de cartón para rastrearla por el camión de reparto. Ese es el **Digest** (en el manifest del registry): el hash de la capa *comprimida* en formato `.tar.gz` tal y como viaja por la red.
> 2. **El DiffID Local (El escritorio armado en tu habitación)**: Cuando te llega la caja, la abres, sacas las maderas y armas el escritorio. El mueble armado ahora ocupa un volumen real y tiene un aspecto totalmente diferente. Si mides y etiquetas el mueble terminado, esa etiqueta es el **DiffID**: el hash de la capa *descomprimida* tal y como reside instalada físicamente en tu disco local bajo `overlay2/`.
> 
> **¿Por qué son diferentes hashes?** Porque una cosa es calcular la firma digital de una caja de cartón plana sellada, y otra es calcular la firma del escritorio de madera armado en tu cuarto. Son el mismo objeto, pero en estados físicos y de compresión totalmente diferentes.


---

## 2. Sistema de capas (layers) en profundidad

> **ESTA ES LA SECCIÓN MÁS IMPORTANTE DEL CAPÍTULO.**  
> *Entender las capas es entender Docker. No hay atajos. Cada detalle aquí explicado es una palanca de optimización en tus manos.*

### 2.1 Cada instrucción Dockerfile crea una capa

Regla fundamental del modelo de build de Docker: toda instrucción que modifica el filesystem (`RUN`, `COPY`, `ADD`) produce una capa. Las instrucciones de metadatos (`ENV`, `CMD`, `ENTRYPOINT`, `EXPOSE`, `VOLUME`, `WORKDIR`, `USER`, `LABEL`, `STOPSIGNAL`, `HEALTHCHECK`, `SHELL`) modifican solamente el config JSON, sin tocar el filesystem.

```dockerfile
FROM ubuntu:22.04                          # Capa 0 (heredada de ubuntu:22.04)
LABEL maintainer="yo@ejemplo.com"          # Metadato, 0 bytes en filesystem
ENV APP_HOME=/app                          # Metadato, 0 bytes en filesystem
WORKDIR $APP_HOME                          # Metadato, 0 bytes en filesystem
RUN apt-get update && apt-get install -y \ # Capa 1 (~150 MB descomprimido)
    build-essential python3 python3-pip
COPY requirements.txt .                    # Capa 2 (~1 KB)
RUN pip install -r requirements.txt        # Capa 3 (~200 MB)
COPY . .                                   # Capa 4 (~5 MB)
EXPOSE 8000                                # Metadato
CMD ["python", "app.py"]                   # Metadato
```

Este Dockerfile produce **5 capas de filesystem** (capas 0 a 4) + **7 entradas de metadatos en el config**.

#### ¿Qué hay DENTRO de cada capa? (Anatomía microscópica)

Siguiendo el ejemplo anterior, si pudiéramos "abrir" cada capa y listar su contenido:

```
Capa 0 (ubuntu:22.04) - FROM ubuntu:22.04:
  /bin/bash, /usr/lib/python3.10/, /etc/passwd, /var/log/, /dev/, /proc/, ...
  → El sistema operativo base completo (~77 MB comprimido, ~200 MB descomprimido)
  → Contiene TODO lo necesario para funcionar como Linux mínimo

Capa 1 (apt-get install) - RUN apt-get update && apt-get install -y ...:
  /usr/bin/gcc, /usr/include/stdio.h, /usr/lib/libc.a, /usr/bin/python3, ...
  → Solo los archivos NUEVOS o MODIFICADOS que instaló apt-get (~150 MB)
  → No incluye los archivos de la capa 0, solo el diff

Capa 2 (COPY requirements.txt) - COPY requirements.txt .:
  /app/requirements.txt
  → Un solo archivo (~200 bytes)
  → La capa más pequeña posible

Capa 3 (pip install) - RUN pip install -r requirements.txt:
  /usr/local/lib/python3.10/dist-packages/flask/*
  /usr/local/lib/python3.10/dist-packages/sqlalchemy/*
  /usr/local/lib/python3.10/dist-packages/requests/*
  ...
  → Solo los paquetes instalados por pip (~200 MB)
  → Si requirements.txt tiene 20 paquetes, esta capa contiene ~2000 archivos nuevos

Capa 4 (COPY . .) - COPY . .:
  /app/app.py, /app/models/, /app/views/, /app/static/, /app/templates/
  → El código fuente de la aplicación (~5 MB)
  → Contiene solo los archivos del proyecto
```

**OBSERVACIÓN FUNDAMENTAL PARA OPTIMIZACIÓN:** Observa los tamaños. La capa 3 (dependencias) ocupa ~200 MB, la capa 4 (código) solo ~5 MB. Si modificas `app.py` y reconstruyes, solo la capa 4 se vuelve a crear (5 MB a subir al registry, 5 segundos de build). Las capas 0-3 vienen del caché de build. Pero si modificas `requirements.txt` (capa 2), Docker DEBE reconstruir las capas 2, 3 y 4 (200+ MB a subir, varios minutos de build). Por eso el orden de las instrucciones en el Dockerfile es una decisión de OPTIMIZACIÓN ECONÓMICA, no solo estética.

```
                   ┌─────────────────────────────────────────┐
                   │  FRECUENCIA DE CAMBIO                   │
                   │                                         │
  (rara vez)       │  FROM ubuntu:22.04         ← Capa 0    │
                   │  RUN apt-get install       ← Capa 1    │
                   │  COPY requirements.txt     ← Capa 2    │
                   │  RUN pip install           ← Capa 3    │
  (muy frecuente)  │  COPY . .                  ← Capa 4    │
                   │                                         │
                   │  REGLA DE ORO:                          │
                   │  Lo que cambia MENOS va PRIMERO.        │
                   │  Lo que cambia MÁS va ÚLTIMO.           │
                   └─────────────────────────────────────────┘
```

#### La trampa de la primera capa heredada

La capa 0 no la "creas" tú; la heredas de `FROM`. Esta capa es la imagen base completa. Es importante entender que el tamaño de la imagen base cuenta como tu primera capa:

```bash
# Comparación de tamaños de imágenes base
docker pull ubuntu:22.04        # ~77 MB
docker pull alpine:3.19         # ~7 MB
docker pull debian:bookworm     # ~130 MB
docker pull scratch             # 0 MB (imagen vacía, sin filesystem)

# scratch es perfecta para binarios compilados estáticamente (Go, Rust)
```

### 2.2 Inmutabilidad de las capas

Las capas de una imagen son **inmutables** (solo lectura). Una vez que una capa se crea (durante `docker build`), nunca se modifica. Esta es una propiedad fundamental del modelo de almacenamiento.

Cuando inicias un contenedor con `docker run`, Docker crea una **capa de contenedor** (container layer) fina, de lectura-escritura, encima de la pila de capas de la imagen. TODOS los cambios que hace el contenedor durante su ejecución (escribir logs, crear archivos temporales, modificar configuraciones, instalar paquetes, etc.) van EXCLUSIVAMENTE a esta capa efímera.

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   CAPA DE CONTENEDOR  (read-write)                               │
│   upperdir                                                       │  ← Creada al hacer
│   - /var/log/app.log (nuevo)                                     │    docker run
│   - /tmp/cache.dat (nuevo)                                       │    Se destruye al
│   - /etc/nginx/nginx.conf (modificado, copy-up desde capa 3)     │    eliminar el contenedor
│   - .wh.archivo_borrado (whiteout, archivo borrado)              │    (docker rm)
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CAPA 4 (imagen)      readonly   ← COPY . .                    │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CAPA 3 (imagen)      readonly   ← RUN pip install              │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CAPA 2 (imagen)      readonly   ← COPY requirements.txt       │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CAPA 1 (imagen)      readonly   ← RUN apt-get install          │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CAPA 0 (imagen)      readonly   ← FROM ubuntu:22.04           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Consecuencia práctica de la inmutabilidad:** Si ejecutas 10 contenedores de la misma imagen y cada uno escribe 100 MB de logs, consumirás 1 GB de espacio en disco EN CAPAS DE CONTENEDOR (efímeras). Las capas de la imagen no se tocan. Por eso es crucial configurar rotación de logs (`--log-opt max-size=10m --log-opt max-file=3`) o, mejor aún, enviar los logs a stdout/stderr y dejar que el runtime los gestione.

**Docker commit: la excepción que confirma la regla**

```bash
# docker commit CREA una nueva capa de imagen a partir de la capa de contenedor
docker run -d --name temp nginx:alpine
docker exec temp apk add curl           # Se instala en la capa del contenedor
docker commit temp nginx-with-curl:v1   # Crea una NUEVA capa de imagen
# Ahora nginx-with-curl:v1 tiene: capas de nginx:alpine + capa con curl
docker rm temp
```

`docker commit` "solidifica" la capa de contenedor en una capa de imagen inmutable. Es útil para debugging rápido, pero NO para producción (pierdes el Dockerfile, el historial de cambios, y la reproducibilidad).

### 2.3 Layer sharing (compartición de capas)

La compartición de capas es la razón por la que Docker puede almacenar decenas de imágenes en el espacio que ocuparían dos. Varias imágenes pueden referenciar las mismas capas, y Docker las almacena UNA SOLA VEZ en disco.

#### Ejemplo concreto: 5 imágenes Node.js en un solo servidor

```
Imagen A: node:20-alpine          Capas: [alpine:3.19] [node:20] [npm:tools]
Imagen B: node:20-slim            Capas: [debian:slim] [node:20] [npm:tools]
Imagen C: api-rest:v1             Capas: [alpine:3.19] [node:20] [npm:tools] [app:deps] [app:src]
Imagen D: worker:v1               Capas: [alpine:3.19] [node:20] [npm:tools] [worker:deps] [worker:src]
Imagen E: frontend:v1             Capas: [alpine:3.19] [node:20] [npm:tools] [frontend:deps] [frontend:src]
```

```
                     ┌──────────────┐
                     │ alpine:3.19  │◄──────────────┬──────────────┬──────────────┐
                     │ 7 MB         │               │              │              │
                     └──────┬───────┘               │              │              │
                            │                       │              │              │
                     ┌──────▼───────┐               │              │              │
                     │   node:20    │◄──────────┬───┤              │              │
                     │   50 MB      │           │   │              │              │
                     └──────┬───────┘           │   │              │              │
                            │                   │   │              │              │
                     ┌──────▼───────┐           │   │              │              │
                     │  npm:tools   │◄──────┐   │   │              │              │
                     │   10 MB      │       │   │   │              │              │
                     └──────┬───────┘       │   │   │              │              │
                            │               │   │   │              │              │
                     ┌──────▼───────┐       │   │   │              │              │
                     │  app:deps    │       │   │   │              │              │
                     │  200 MB      │       │   │   │              │              │
                     └──────┬───────┘       │   │   │              │              │
                            │               │   │   │              │              │
              ┌─────────────┼──────────────┐│   │   │              │              │
              │             │              ││   │   │              │              │
     ┌────────▼───┐  ┌──────▼───┐  ┌──────▼▼───┐│   │              │              │
     │ app:src    │  │worker:src│  │front:src  ││   │              │              │
     │ 5 MB       │  │ 3 MB     │  │ 8 MB      ││   │              │              │
     │ api-rest:v1│  │worker:v1 │  │frontend:v1││   │              │              │
     └────────────┘  └──────────┘  └───────────┘│   │              │              │
                                                 │   │              │              │
┌────────────────────────────────────────────────┘   │              │              │
│  Imagen B: node:20-slim                              │              │              │
│  Capas: [debian:slim 60MB] [node:20 50MB] [npm 10MB]│              │              │
│  Comparte node:20 y npm:tools con las otras imágenes!│              │              │
└─────────────────────────────────────────────────────┘   │              │
                                                           │              │
┌──────────────────────────────────────────────────────────┘              │
│  Imagen A: node:20-alpine                                               │
│  Capas: [alpine:3.19 7MB] [node:20 50MB] [npm:tools 10MB]              │
└─────────────────────────────────────────────────────────────────────────┘

ESPACIO TOTAL EN DISCO:
  Sin compartición: 7+50+10 + 60+50+10 + (7+50+10+200+5)*3 = 1,128 MB
  Con compartición:  7+50+10 + 60       + 200 + 5+3+8      =   343 MB
  AHORRO: 785 MB (70%)
```

#### Evidencia empírica en tu sistema

```bash
# Ver cuánto espacio REAL ocupan las imágenes (contando compartición)
docker system df

# Output típico:
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          25        12        5.31GB    3.2GB (60%)
# Containers      10        3         500MB     400MB (80%)
# Local Volumes   15        8         2.3GB     1.1GB (47%)
# Build Cache     50        0         3.5GB     3.5GB (100%)

# Ver detalle de compartición por imagen
docker system df -v

# Output muestra:
# Images space usage:
# REPOSITORY    TAG       IMAGE ID       SIZE    SHARED   UNIQUE
# api-rest      v1        abc123def456   462MB   410MB    52MB
# worker        v1        def789abc123   463MB   410MB    53MB
# frontend      v1        ghi456def789   468MB   420MB    48MB
# node          20-alpine jkl012ghi345   410MB   410MB    0MB    ← 100% compartida
# nginx         alpine    mno345jkl678   42MB    0MB      42MB
```

Aquí `api-rest`, `worker` y `frontend` comparten 410-420 MB de capas comunes (`alpine`, `node`, `npm`, `deps`). Sus capas únicas (código fuente) ocupan solo ~50 MB cada una. Y `node:20-alpine` tiene 0 MB únicos porque TODAS sus capas son compartidas por las imágenes de aplicación derivadas.

### 2.4 Storage Drivers: El cómo físico de las capas

Docker abstrae el sistema de capas mediante **storage drivers** (también llamados graph drivers). El driver determina el mecanismo concreto por el cual las capas se almacenan en disco y se combinan para formar un filesystem unificado.

#### Comparativa exhaustiva de storage drivers

| Driver | Estado (2024) | Mecanismo | Rendimiento | Uso de inodos | Kernel requerido |
|--------|--------------|-----------|-------------|---------------|-----------------|
| `overlay2` | **Estándar** | OverlayFS (kernel) | Excelente | Eficiente (reutiliza inodos inferiores) | >= 4.0, recomendado >= 5.4 |
| `fuse-overlayfs` | **Alternativa rootless** | OverlayFS en userspace vía FUSE | Bueno | Eficiente | Cualquiera con FUSE |
| `devicemapper` | **Deprecado** | thin provisioning + snapshots | Medio | Consume inodos por bloque | Configuración específica (LVM) |
| `aufs` | **Deprecado/eliminado** | Union mount (parche kernel) | Bueno | Eficiente | Nunca en mainline, Ubuntu/Debian parche |
| `btrfs` | Mantenido | Subvolúmenes + snapshots Btrfs | Bueno | Nativo de Btrfs | Sistema de archivos Btrfs |
| `zfs` | Mantenido | ZFS datasets + clones | Bueno | Nativo de ZFS | Sistema de archivos ZFS |
| `vfs` | Solo testing | Copia completa de directorios | Muy pobre | Duplica todo | Cualquiera |

#### ¿Por qué overlay2 es el estándar indiscutible?

1. **Está en el kernel mainline** desde Linux 3.18 (overlay) y 4.0 (overlay2/multiple lower layers). No requiere parches externos ni módulos de terceros.
2. **Copy-on-write a nivel de ARCHIVO** (no de bloque como devicemapper). Más granular y eficiente para cargas de trabajo de contenedores.
3. **Soporta hasta 500 capas** de profundidad (overlay2) — el overlay original solo 2 capas.
4. **Reutilización de inodos**: cuando un archivo no se modifica, overlay2 usa el inodo real de la capa inferior en lugar de duplicarlo. Esto ahorra metadatos y cache del kernel.
5. **Operaciones atómicas en workdir**: rename, link y unlink son atómicos en overlay2 gracias a workdir, evitando estados inconsistentes durante escrituras concurrentes.
6. **Soporte para selinux/apparmor**: overlay2 funciona correctamente con sistemas de seguridad obligatoria.
7. **Compatible con rootless**: fuse-overlayfs proporciona la misma interfaz para usuarios sin privilegios.

```bash
# Ver qué driver usa tu sistema
docker info --format '{{.Driver}}'
# Output: overlay2

# Ver detalles del storage driver
docker info | grep -A 15 "Storage Driver"

# Ver la versión del kernel (overlay2 requiere >= 4.0)
uname -r

# Ver si overlayfs está soportado en el kernel
grep overlay /proc/filesystems
# Output: nodev   overlay
```

### 2.5 Cómo OverlayFS implementa las capas (anatomía interna)

OverlayFS monta un sistema de archivos virtual que une múltiples directorios, presentando una vista unificada al proceso (el contenedor).

```
┌──────────────────────────────────────────────────────────────────┐
│                     MONTAJE OVERLAYFS                            │
│                                                                  │
│  mount -t overlay overlay \                                      │
│    -o lowerdir=/capaA:/capaB:/capaC:/capaD, \   ← Solo lectura  │
│       upperdir=/capa-contenedor, \               ← Lectura/Escritura │
│       workdir=/work, \                           ← Operaciones atómicas │
│       /punto/de/montaje                          ← Vista unificada │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

#### Los cuatro directorios del overlay

| Directorio | Permisos | ¿Persiste tras docker rm? | Propósito |
|-----------|----------|--------------------------|-----------|
| `lowerdir` | Solo lectura | Sí (es la imagen) | Capas de la imagen, separadas por `:`. La primera tiene mayor prioridad (está más arriba). |
| `upperdir` | Lectura/escritura | **No** (se borra con el contenedor) | Capa del contenedor. Todos los cambios de escritura van aquí. |
| `workdir` | Interno (no se accede) | **No** | Directorio de trabajo atómico. Debe estar en el MISMO sistema de archivos que upperdir. |
| `merged` | Vista unificada | **No** | El punto de montaje que el contenedor ve como su filesystem raíz. |

#### lowerdir: la cadena de capas

El formato de `lowerdir` es una lista separada por `:` donde cada entrada es un directorio:

```
lowerdir=/var/lib/docker/overlay2/ABC123/diff:/var/lib/docker/overlay2/DEF456/diff:/var/lib/docker/overlay2/GHI789/diff
```

La PRIMERA entrada (ABC123) es la capa MÁS ALTA (la última capa de la imagen, la más cercana al contenedor). La ÚLTIMA entrada (GHI789) es la capa MÁS BAJA (la capa base de la imagen, el `FROM`).

```
  Prioridad de búsqueda (de mayor a menor):
  
  upperdir         ← Capa del contenedor (máxima prioridad)
      │
  lowerdir[0]      ← Última capa de la imagen (COPY . .)
      │
  lowerdir[1]      ← RUN pip install
      │
  lowerdir[2]      ← COPY requirements.txt
      │
  lowerdir[3]      ← RUN apt-get install
      │
  lowerdir[4]      ← FROM ubuntu:22.04 (mínima prioridad)
```

Cuando el contenedor busca un archivo (por ejemplo `/etc/nginx/nginx.conf`), OverlayFS recorre los directorios en este orden y devuelve la primera coincidencia. Por eso se llama "overlay": las capas superiores "tapan" a las inferiores.

#### Inspeccionar las capas de un contenedor real

```bash
# 1. Obtener el ID de un contenedor en ejecución
CONTAINER_ID=$(docker ps -q --filter "name=mi-app" | head -1)

# 2. Ver los mounts del contenedor (datos del graph driver)
docker inspect $CONTAINER_ID --format='{{json .GraphDriver.Data}}' | python3 -m json.tool
# Output:
# {
#   "LowerDir": "/var/lib/docker/overlay2/abc123.../diff:/var/lib/docker/overlay2/def456.../diff:...",
#   "MergedDir": "/var/lib/docker/overlay2/xyz789.../merged",
#   "UpperDir": "/var/lib/docker/overlay2/xyz789.../diff",
#   "WorkDir": "/var/lib/docker/overlay2/xyz789.../work"
# }

# 3. Examinar las capas de la imagen (lowerdir) - requieren sudo
LOWER_DIRS=$(docker inspect $CONTAINER_ID --format='{{index .GraphDriver.Data "LowerDir"}}')
echo "$LOWER_DIRS" | tr ':' '\n' | while read dir; do
  echo "=== Capa: $dir ==="
  sudo ls "$dir" 2>/dev/null | head -20
  echo ""
done

# 4. Ver el filesystem como lo ve el contenedor (merged)
MERGED=$(docker inspect $CONTAINER_ID --format='{{index .GraphDriver.Data "MergedDir"}}')
sudo ls "$MERGED"

# 5. Ver SOLO los archivos modificados por ESTE contenedor (upperdir)
UPPER=$(docker inspect $CONTAINER_ID --format='{{index .GraphDriver.Data "UpperDir"}}')
sudo find "$UPPER" -type f 2>/dev/null | head -20
```

#### El archivo `lower`: la cadena de dependencias

Dentro de cada directorio de capa en `/var/lib/docker/overlay2/<cache-id>/`, hay un archivo llamado `lower` que contiene la cadena de capas inferiores:

```bash
# Ver la cadena de dependencias de una capa (requiere sudo)
sudo cat /var/lib/docker/overlay2/<cache-id>/lower

# Output: l/ABC123:l/DEF456:l/GHI789
# Significado: Esta capa (cache-id) se monta ENCIMA de ABC123,
#              que está encima de DEF456, que está encima de GHI789.
#              Cada 'l/XXX' es un enlace simbólico a la capa real.
```

Este archivo permite a Docker reconstruir instantáneamente el orden de capas sin escanear el filesystem. Al iniciar un contenedor, Docker lee este archivo para construir el `lowerdir` del montaje OverlayFS.

#### Navegación completa por el sistema de archivos de Docker

```bash
# === EXPLORACIÓN GUIADA DEL SISTEMA DE CAPAS (requiere sudo) ===

# 1. Ver todas las capas almacenadas
sudo ls /var/lib/docker/overlay2/ | head -20

# 2. Para una capa específica, ver su contenido
CACHE_ID=$(sudo ls /var/lib/docker/overlay2/ | head -1)
echo "Explorando capa: $CACHE_ID"

# 3. Ver los archivos en esta capa (solo el diff)
sudo ls -la /var/lib/docker/overlay2/$CACHE_ID/diff/

# 4. Ver el nombre corto (el que usa docker internamente)
sudo cat /var/lib/docker/overlay2/$CACHE_ID/link
# Output: ABC123DEF456 (nombre corto, no SHA completo)

# 5. Ver la cadena de capas inferiores
sudo cat /var/lib/docker/overlay2/$CACHE_ID/lower 2>/dev/null || echo "(no tiene capas inferiores - es la capa base)"

# 6. Encontrar qué imágenes usan esta capa
# (Las capas no guardan referencia a las imágenes que las usan,
#  pero podemos buscar en la base de datos de imágenes)
sudo cat /var/lib/docker/image/overlay2/layerdb/sha256/*/cache-id | grep -l "$CACHE_ID"

# 7. Ver el mapeo imagen → capas
# Primero, obtener el ID de una imagen
IMAGE_ID=$(docker images -q nginx:alpine)
echo "Image ID: $IMAGE_ID"

# Luego, ver su configuración
sudo cat /var/lib/docker/image/overlay2/imagedb/content/sha256/$IMAGE_ID | python3 -m json.tool | grep -A 20 rootfs

# 8. Ver el mapeo tag → image ID
sudo cat /var/lib/docker/image/overlay2/repositories.json | python3 -m json.tool
```

### 2.6 Copy-on-write (CoW) explicado paso a paso

Copy-on-write es el mecanismo que permite a múltiples contenedores compartir las mismas capas de imagen sin interferir. Cuando un contenedor necesita modificar un archivo que reside en una capa inferior (de solo lectura), OverlayFS **copia** ese archivo al `upperdir` (capa del contenedor) y luego aplica la modificación allí. El archivo original en la capa de la imagen permanece **intacto**.

#### Escenario 1: LEER un archivo existente (operación más común)

```
El proceso en el contenedor quiere leer /etc/nginx/nginx.conf

Búsqueda (top-down):
             ┌───────────────────────────────────────┐
             │     CAPA CONTENEDOR (upperdir)        │
             │     /etc/nginx/nginx.conf?            │
             │     ❌ No está aquí                   │
             ├───────────────────────────────────────┤
             │     CAPA 4 (imagen) - COPY . .        │
             │     /etc/nginx/nginx.conf?            │
             │     ❌ No está aquí                   │
             ├───────────────────────────────────────┤
             │     CAPA 3 (imagen) - RUN pip install │
             │     /etc/nginx/nginx.conf?            │
             │     ❌ No está aquí                   │
             ├───────────────────────────────────────┤
             │     CAPA 2 (imagen) - apt install     │
             │     /etc/nginx/nginx.conf ✅          │  ← ¡Encontrado!
             │     Se lee directamente desde aquí.   │
             │     NO se copia nada a upperdir.      │
             └───────────────────────────────────────┘

COSTE: O(1) búsqueda en el overlay, lectura directa del inodo original.
       La capa del contenedor no crece. Zero overhead.
```

#### Escenario 2: ESCRIBIR (modificar) un archivo existente → COPY-UP

```
El proceso en el contenedor quiere modificar /etc/nginx/nginx.conf

PASO 1: Búsqueda top-down (igual que en lectura)
  upperdir     → ❌
  capa 4       → ❌
  capa 3       → ❌
  capa 2       → ✅ ¡Encontrado!

PASO 2: COPY-UP (la magia del CoW)
  ┌──────────────────────────────────────────────────────────────────┐
  │  OverlayFS detecta que es una ESCRITURA en un archivo de una    │
  │  capa inferior (solo lectura).                                   │
  │                                                                  │
  │  Acciones:                                                       │
  │  1. Crear los directorios padres en upperdir si no existen:      │
  │       mkdir -p upperdir/etc/nginx/                               │
  │  2. Copiar el archivo COMPLETO a upperdir:                       │
  │       cp capa2/etc/nginx/nginx.conf upperdir/etc/nginx/nginx.conf│
  │  3. Aplicar la modificación en la copia de upperdir:            │
  │       echo "nueva_linea" >> upperdir/etc/nginx/nginx.conf       │
  │                                                                  │
  │  La PRÓXIMA vez que alguien lea /etc/nginx/nginx.conf:           │
  │  upperdir → ✅ (se encuentra aquí primero, versión modificada)   │
  │  La copia en capa 2 sigue intacta pero queda "tapada".          │
  └──────────────────────────────────────────────────────────────────┘

RESULTADO:
  upperdir/etc/nginx/nginx.conf → archivo COMPLETO + modificación
  capa2/etc/nginx/nginx.conf    → archivo original INTACTO

  El espacio ocupado en upperdir = tamaño del archivo copiado
  (solo la primera vez; escrituras subsecuentes no copian de nuevo)
```

#### Escenario 3: CREAR un archivo nuevo → Escritura directa

```
El proceso crea /var/log/app/access.log (no existe en ninguna capa)

Búsqueda top-down:
  upperdir  → ❌
  capa 4    → ❌
  capa 3    → ❌
  capa 2    → ❌
  capa 1    → ❌
  capa 0    → ❌

Acción:
  mkdir -p upperdir/var/log/app/
  echo "log entry" > upperdir/var/log/app/access.log

Las capas inferiores no se tocan.
En merged, el archivo aparece como si siempre hubiera estado allí.
```

#### Escenario 4: BORRAR un archivo → Whiteout files

```
El proceso borra /usr/share/doc/README (existe en capa 1)

Problema: OverlayFS no puede "borrar" un archivo de una capa inferior
         porque las capas son readonly.

Solución: Crear un WHITEOUT FILE en upperdir.

  upperdir/usr/share/doc/.wh.README  ← Whiteout file
    Es un character device con major=0, minor=0.
    El nombre '.wh.<nombre_original>' le dice a OverlayFS:
    "El archivo <nombre_original> NO EXISTE, aunque esté en capas inferiores."

  capa1/usr/share/doc/README  ← Sigue existiendo
    Pero OverlayFS lo OCULTA porque hay un whiteout en upperdir.

Para borrar un DIRECTORIO ENTERO:
  upperdir/usr/share/doc/.wh..wh..opq  ← "Opaque whiteout"
  Indica que TODO el contenido del directorio está oculto.
  No importa qué archivos haya en capas inferiores: no se ven.
```

#### Whiteout files: Anatomía técnica

```bash
# Ver whiteout files en la capa de un contenedor
sudo find /var/lib/docker/overlay2/<container-cache-id>/diff/ -name ".wh.*" -o -name ".wh..wh..opq"

# Ejemplo de salida:
# /var/lib/docker/overlay2/abc123.../diff/etc/nginx/.wh.nginx.conf
# /var/lib/docker/overlay2/abc123.../diff/tmp/.wh..wh..opq

# Inspeccionar un whiteout
sudo stat /var/lib/docker/overlay2/abc123.../diff/etc/nginx/.wh.nginx.conf
# Output:
#   File: .wh.nginx.conf
#   Size: 0          Blocks: 0          IO Block: 4096   character special file
#   Device: 0,0      Inode: 123456      Links: 1
#   Device type: 0,0
#                     │  │
#                     │  └── Minor device number (0)
#                     └───── Major device number (0)
```

**Los whiteout files y `docker commit`:** Si haces `docker commit` sobre un contenedor que tiene whiteout files, estos se incluyen en la nueva capa de imagen. La siguiente imagen derivada NACE con esos archivos "borrados" (los whiteouts se convierten en parte de la capa inmutable). Si eliminas el contenedor sin commit, los whiteouts desaparecen y el archivo original "reaparece" en el siguiente contenedor.

#### Demostración interactiva de Copy-on-Write

```bash
# ============================================================
# DEMOSTRACIÓN PRÁCTICA DE COPY-ON-WRITE
# ============================================================

# 1. Crear un contenedor de nginx
docker run -d --name cow-demo nginx:alpine

# 2. Obtener las rutas de overlay2
UPPER=$(docker inspect cow-demo --format='{{index .GraphDriver.Data "UpperDir"}}')
MERGED=$(docker inspect cow-demo --format='{{index .GraphDriver.Data "MergedDir"}}')
echo "=== Rutas del contenedor ==="
echo "Upper:  $UPPER"
echo "Merged: $MERGED"

# 3. ANTES de modificar: el archivo NO existe en upperdir
echo ""
echo "=== ANTES: ¿Existe nginx.conf en upperdir? ==="
sudo test -f "$UPPER/etc/nginx/nginx.conf" && echo "SÍ (copy-up ya ocurrió)" || echo "NO (se lee de la imagen)"

# 4. Ver el archivo desde merged (lectura desde capa de imagen)
echo ""
echo "=== Contenido de nginx.conf (primeras líneas) ==="
sudo head -3 "$MERGED/etc/nginx/nginx.conf"

# 5. MODIFICAR el archivo desde el contenedor
echo ""
echo "=== Modificando nginx.conf... ==="
docker exec cow-demo sh -c "echo '# LINEA AÑADIDA EN CONTENEDOR' >> /etc/nginx/nginx.conf"

# 6. DESPUÉS de modificar: ¡Copy-up ocurrió!
echo ""
echo "=== DESPUÉS: ¿Existe nginx.conf en upperdir? ==="
sudo test -f "$UPPER/etc/nginx/nginx.conf" && echo "SÍ (copy-up ocurrió)" || echo "NO"

# 7. El archivo en upperdir contiene el archivo COMPLETO + la línea añadida
echo ""
echo "=== Últimas 3 líneas de nginx.conf en upperdir ==="
sudo tail -3 "$UPPER/etc/nginx/nginx.conf"

# 8. CREAR un archivo completamente nuevo
echo ""
echo "=== Creando archivo nuevo... ==="
docker exec cow-demo sh -c "echo 'nuevo contenido' > /tmp/archivo-nuevo.txt"
sudo ls -la "$UPPER/tmp/"
sudo cat "$UPPER/tmp/archivo-nuevo.txt"

# 9. BORRAR un archivo que existe en la imagen
echo ""
echo "=== Borrando /etc/nginx/fastcgi.conf... ==="
docker exec cow-demo rm /etc/nginx/fastcgi.conf
sudo ls -la "$UPPER/etc/nginx/" | grep ".wh."
# Verás .wh.fastcgi.conf

# 10. Verificar que el archivo ya no se ve en merged
echo ""
echo "=== ¿Existe fastcgi.conf en merged? ==="
sudo test -f "$MERGED/etc/nginx/fastcgi.conf" && echo "SÍ (fallo)" || echo "NO (correcto, el whiteout lo oculta)"

# 11. Crear un SEGUNDO contenedor de la misma imagen
docker run -d --name cow-demo-2 nginx:alpine
UPPER2=$(docker inspect cow-demo-2 --format='{{index .GraphDriver.Data "UpperDir"}}')

echo ""
echo "=== Segundo contenedor: ¿nginx.conf en upperdir? ==="
sudo test -f "$UPPER2/etc/nginx/nginx.conf" && echo "SÍ" || echo "NO (se lee de la imagen, copy-up aún no ocurrió)"
echo "=== Segundo contenedor: ¿fastcgi.conf en merged? ==="
MERGED2=$(docker inspect cow-demo-2 --format='{{index .GraphDriver.Data "MergedDir"}}')
sudo test -f "$MERGED2/etc/nginx/fastcgi.conf" && echo "SÍ (correcto, no se borró en este contenedor)" || echo "NO"

# 12. Limpiar
docker rm -f cow-demo cow-demo-2
```

### 2.7 Estrategias de optimización del Dockerfile basadas en capas

El conocimiento profundo de cómo funcionan las capas te permite tomar decisiones de diseño en el Dockerfile que tienen un impacto directo en:
- Velocidad de build (cache hits vs cache misses)
- Tamaño de la imagen final
- Ancho de banda consumido en CI/CD
- Tiempo de despliegue (pull de imágenes)

#### Principio 1: Ordena de menos cambiante a más cambiante

```
┌──────────────────────────────────────────────────────────────┐
│  ESTRUCTURA ÓPTIMA DEL DOCKERFILE                            │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 1. FROM           ← Base OS (cambia cada meses)         ││
│  │ 2. RUN paquetes   ← Dependencias del sistema (semanas)  ││
│  │ 3. RUN runtime    ← Runtimes (Node, Python, etc.)       ││
│  │ 4. COPY deps      ← Archivos de dependencias (días)     ││
│  │ 5. RUN install    ← Instalar dependencias (días)        ││
│  │ 6. COPY source    ← Código fuente (horas/minutos)       ││
│  │ 7. CMD/ENTRYPOINT ← Metadatos (no afecta cache)         ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  REGLA MNEMOTÉCNICA: "Lo que menos cambia, más arriba"      │
└──────────────────────────────────────────────────────────────┘
```

#### Ejemplo concreto ANTES y DESPUÉS

```dockerfile
# ==========================================
# ❌ MAL: El caché se invalida constantemente
# ==========================================
FROM node:20-alpine
WORKDIR /app
COPY . .                        # ← CUALQUIER cambio en src/ invalida ESTA capa
RUN npm ci --only=production    # ← ...y ESTA también (se descarta el cache)
CMD ["node", "server.js"]
# Resultado: cada cambio en app.js → reconstruir npm ci (2-5 minutos)

# ==========================================
# ✅ BIEN: Separar dependencias y código fuente
# ==========================================
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./   # ← Solo cambia cuando cambian las deps
RUN npm ci --only=production             # ← Cache hit el 99% de las veces
COPY src/ ./src/                         # ← Solo se recrea esta capa (1 segundo)
CMD ["node", "server.js"]
# Resultado: cada cambio en app.js → solo COPY src/ (segundos)
```

#### Principio 2: Agrupa comandos relacionados y LIMPIA en la misma capa

```dockerfile
# ❌ MAL: Muchas capas con basura
RUN apt-get update
RUN apt-get install -y curl vim git
RUN apt-get install -y build-essential
RUN rm -rf /var/lib/apt/lists/*    # ← ¡TARDE! La basura ya está en capas anteriores
# Cada RUN crea una capa. La capa del rm NO reduce el tamaño:
# las capas anteriores contienen /var/lib/apt/lists/* para siempre.

# ❌ TAMBIÉN MAL: Instalar y luego eliminar en capas separadas
RUN apt-get update && apt-get install -y build-essential python3-dev
RUN pip install somepackage
RUN apt-get remove -y build-essential python3-dev  # ← Inútil para reducir tamaño
# La capa 1 contiene build-essential (~200 MB).
# La capa 3 contiene whiteout files para los archivos removidos.
# La imagen sigue ocupando ~200 MB de más.

# ✅ BIEN: Todo en una capa, limpieza incluida
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        vim \
        git \
        build-essential && \
    pip install somepackage && \
    apt-get remove -y build-essential && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
# Todo en UNA capa. Los archivos añadidos y luego borrados en la misma
# capa no forman parte del diff neto. La capa solo contiene el resultado final.
```

**ACLARACIÓN IMPORTANTE:** `rm -rf` en la MISMA capa donde creaste los archivos SÍ reduce el tamaño de esa capa (el diff neto es correcto). `rm -rf` en una capa POSTERIOR crea whiteout files pero NO libera el espacio ocupado en la capa anterior.

#### Principio 3: Minimiza el número de capas (pero sin perder cacheabilidad)

Docker tiene un límite de 127 capas en la imagen final (herencia de aufs). Aunque overlay2 soporta más, cada capa adicional añade overhead de metadatos y complejidad.

```dockerfile
# ❌ MUCHAS capas: 1 por paquete
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get install -y git
RUN apt-get install -y wget
# → 5 capas RUN. Mala práctica, metadatos innecesarios.

# ✅ Pocas capas, agrupación lógica
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl vim git wget ca-certificates \
        && rm -rf /var/lib/apt/lists/*
# → 1 capa RUN. Excelente.
```

**Número recomendado de capas para una imagen de aplicación típica:** 5-10 capas es un buen equilibrio entre cacheabilidad y eficiencia.

### 2.8 Multi-stage builds: El patrón definitivo para imágenes mínimas

Las multi-stage builds resuelven el conflicto entre "necesito herramientas para compilar" y "no quiero herramientas en producción". Cada `FROM` en el Dockerfile inicia un nuevo stage, y puedes copiar archivos entre stages con `COPY --from=<stage>`.

```dockerfile
# ==========================================
# GO: El caso canónico de multi-stage
# ==========================================
# Stage 1: BUILD (con SDK de Go completo)
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server .

# Stage 2: RUNTIME (solo el binario)
FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /app/server /usr/local/bin/server
USER 1000:1000
ENTRYPOINT ["/usr/local/bin/server"]
```

```
┌──────────────────────────────────────┐  ┌──────────────────────────┐
│  STAGE 1: builder (golang:1.22)      │  │  STAGE 2: alpine:3.19    │
│                                      │  │                          │
│  Capa 0: golang base (~350 MB)       │  │  Capa 0: alpine (~7 MB)  │
│  Capa 1: go.mod/go.sum (~1 KB)       │  │  Capa 1: ca-certificates │
│  Capa 2: go mod download (~50 MB)    │  │          (~3 MB)          │
│  Capa 3: source code (~2 MB)         │  │  Capa 2: COPY --from=    │
│  Capa 4: go build (~10 MB binary)    │  │          builder /app/   │
│                                      │  │          server (~10 MB) │
│  TOTAL builder: ~412 MB              │  │                          │
└──────────────────────────────────────┘  │  TOTAL imagen final:     │
        ↓                                 │  ~20 MB                  │
        └── COPY --from=builder ────────► │                          │
           (SOLO el binario, no el SDK)   └──────────────────────────┘

¡La imagen final NO contiene el SDK de Go, ni go mod cache, ni código fuente!
```

#### Multi-stage builds para Node.js

```dockerfile
# ==========================================
# NODE.JS: Producción optimizada con multi-stage
# ==========================================
# Stage 1: Dependencias (solo producción)
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Stage 2: Build (si necesitas compilar assets, TypeScript, etc.)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# Stage 3: Producción (imagen final mínima)
FROM node:20-alpine AS runtime
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./
USER app
EXPOSE 3000
CMD ["node", "dist/server.js"]
# Imagen final: Node runtime + node_modules (prod) + dist compilado
# Sin TypeScript, sin node_modules de desarrollo, sin herramientas de build
```

#### Multi-stage para Python

```dockerfile
# ==========================================
# PYTHON: FastAPI con multi-stage
# ==========================================
# Stage 1: Builder
FROM python:3.12-slim-bookworm AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime
FROM python:3.12-slim-bookworm AS runtime
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app/ ./app/
ENV PATH=/root/.local/bin:$PATH
USER 1000:1000
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Dato clave sobre multi-stage builds:** Las capas del stage `builder` se almacenan en el BUILD CACHE pero NO se incluyen en la imagen final ni se descargan al hacer `docker pull`. Solo el último stage produce la imagen que se etiqueta y se empuja. Los stages intermedios solo existen durante el build y en el caché local.

---

## 3. Docker Hub

Docker Hub (`docker.io`) es el registry público por defecto. Cuando haces `docker pull nginx`, el daemon resuelve esto como `docker pull docker.io/library/nginx:latest`.

### 3.1 Buscar imágenes: `docker search`

```bash
# Búsqueda básica
docker search nginx

# Output típico:
# NAME              DESCRIPTION                                     STARS     OFFICIAL
# nginx             Official build of Nginx.                        19000+    [OK]
# bitnami/nginx     Bitnami nginx Docker Image                      200+
# linuxserver/nginx An Nginx container, brought to you by Linu...   150+
# ubuntu/nginx      Nginx on Ubuntu                                 100+

# Limitar resultados
docker search --limit 5 nginx

# Filtrar por estrellas mínimas
docker search --filter stars=100 nginx

# Filtrar solo imágenes oficiales
docker search --filter is-official=true nginx

# Buscar solo imágenes automatizadas (automated builds, deprecado)
docker search --filter is-automated=true nginx

# Formato personalizado
docker search --format "table {{.Name}}\t{{.StarCount}}\t{{.IsOfficial}}" nginx
docker search --format "{{.Name}}: {{.Description}}" --limit 3 nginx
```

#### Entender los metadatos de búsqueda

| Columna | Significado | Cómo interpretarlo |
|---------|-------------|-------------------|
| `NAME` | Nombre de la imagen. `nginx` (sin `/`) = imagen oficial. `usuario/imagen` = repositorio de usuario/org. | Las oficiales son mantenidas por Docker en colaboración con el proyecto upstream. |
| `STARS` | Número de estrellas (similar a GitHub stars). | Indicador de POPULARIDAD, no de calidad ni seguridad. Una imagen con 10K estrellas puede tener vulnerabilidades. |
| `OFFICIAL` | `[OK]` = imagen mantenida por Docker, Inc. Pasa por revisión de seguridad y buenas prácticas. | Mayor confianza, pero no garantiza ausencia de vulnerabilidades. |
| `AUTOMATED` | `[OK]` = construida automáticamente desde un repo GitHub/Bitbucket. | Deprecado para cuentas gratuitas (2021). Ya no es un indicador confiable. |
| `DESCRIPTION` | Texto del mantenedor. | Las imágenes oficiales suelen tener descripciones detalladas con ejemplos. |

**ADVERTENCIA IMPORTANTE DE SEGURIDAD:** Un alto número de estrellas NO garantiza seguridad. Antes de usar una imagen en producción, verifica:
1. La fecha de la última actualización (¿se mantiene activamente?).
2. El número de vulnerabilidades reportadas (`docker scout` o `trivy`).
3. El Dockerfile (si está disponible) — revisa qué hace realmente.
4. La reputación del mantenedor.

### 3.2 Pull: Anatomía completa de una descarga

```bash
docker pull nginx:1.25-alpine
```

Cuando este comando se ejecuta, ocurre una secuencia compleja de operaciones. Vamos a desglosarla paso a paso:

```
┌──────────────────────────────────────────────────────────────────────┐
│  PASO 1: RESOLUCIÓN DEL NOMBRE                                       │
│                                                                       │
│  docker.io/library/nginx:1.25-alpine                                  │
│  │         │         │     │                                          │
│  │         │         │     └── tag                                   │
│  │         │         └──────── nombre de la imagen                   │
│  │         └────────────────── namespace (library = imágenes oficiales)│
│  └──────────────────────────── registry index (docker.io por defecto)│
│                                                                       │
│  Si el registry no se especifica, Docker asume docker.io.             │
│  Si el namespace no se especifica, Docker asume library (oficiales).  │
│  Si el tag no se especifica, Docker asume latest.                     │
└──────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PASO 2: AUTENTICACIÓN                                               │
│                                                                       │
│  Docker verifica si tiene credenciales para el registry:              │
│    - Busca en ~/.docker/config.json                                  │
│    - Para imágenes públicas en Docker Hub, obtiene un token anónimo   │
│    - Para imágenes privadas, usa credenciales de docker login         │
│    - El token se incluye en el header Authorization de cada request  │
└──────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PASO 3: DESCARGAR EL MANIFEST LIST (multi-arch)                     │
│                                                                       │
│  GET /v2/library/nginx/manifests/1.25-alpine                          │
│  Headers:                                                             │
│    Accept: application/vnd.oci.image.index.v1+json,                  │
│            application/vnd.docker.distribution.manifest.list.v2+json, │
│            application/vnd.oci.image.manifest.v1+json,               │
│            application/vnd.docker.distribution.manifest.v2+json       │
│                                                                       │
│  El registry responde con el tipo de mayor prioridad que soporta.    │
│  Si soporta OCI Image Index, responde con eso.                        │
│  Si no, responde con el manifest de imagen directamente.              │
│                                                                       │
│  Docker inspecciona la respuesta:                                      │
│  - Si es un manifest list: buscar la entrada que coincide con          │
│    la arquitectura local (uname -m) y el SO (uname -s).                │
│  - Si no hay coincidencia exacta: error "no match for platform".      │
│  - Si hay coincidencia: anotar el digest del manifest seleccionado.   │
└──────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PASO 4: DESCARGAR EL IMAGE MANIFEST                                  │
│                                                                       │
│  GET /v2/library/nginx/manifests/sha256:<manifest-digest>             │
│                                                                       │
│  El registry devuelve el manifest JSON con:                           │
│    - config.digest: SHA del blob de configuración                     │
│    - layers[]: Array de capas con sus digests y tamaños               │
│                                                                       │
│  Docker ahora sabe exactamente qué blobs necesita.                    │
└──────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PASO 5: DESCARGAR LAS CAPAS (en paralelo, hasta 3 simultáneas)      │
│                                                                       │
│  Para cada layer.digest en manifest.layers:                           │
│                                                                       │
│  ┌── VERIFICACIÓN LOCAL ──────────────────────────────────┐         │
│  │                                                         │         │
│  │  ¿Existe /var/lib/docker/image/overlay2/layerdb/       │         │
│  │   sha256/<layer-digest>?                                │         │
│  │                                                         │         │
│  │  SI → Mostrar "<digest>: Already exists"                │         │
│  │        Saltar esta capa (no descargar)                   │         │
│  │                                                         │         │
│  │  NO → Descargar: GET /v2/<repo>/blobs/<digest>          │         │
│  │        ┌──────────────────────────────────┐            │         │
│  │        │ 1. Recibir stream de datos        │            │         │
│  │        │ 2. Verificar SHA256 en tránsito   │            │         │
│  │        │ 3. Descomprimir (tar.gz → tar)    │            │         │
│  │        │ 4. Calcular diff_id (SHA descompr)│            │         │
│  │        │ 5. Validar diff_id contra config  │            │         │
│  │        │ 6. Extraer archivos en overlay2/  │            │         │
│  │        │ 7. Registrar en layerdb/          │            │         │
│  │        │ 8. Mostrar "<digest>: Pull complete"│          │         │
│  │        └──────────────────────────────────┘            │         │
│  └───────────────────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PASO 6: DESCARGAR EL CONFIG BLOB                                    │
│                                                                       │
│  GET /v2/<repo>/blobs/<config-digest>                                 │
│  Verificar checksum                                                   │
│  Guardar en imagedb/content/sha256/<image-id>                        │
│  El image-id se calcula como SHA256 del config JSON completo.        │
└──────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PASO 7: REGISTRAR LA IMAGEN EN LA BASE DE DATOS LOCAL              │
│                                                                       │
│  - Actualizar repositories.json: tag (1.25-alpine) → image-id       │
│  - La imagen está lista para usar                                     │
│  - Output final: Digest: sha256:abc123...                             │
└──────────────────────────────────────────────────────────────────────┘
```

#### Observar el pull en detalle

```bash
# Pull normal
docker pull nginx:1.25-alpine

# Output típico:
# 1.25-alpine: Pulling from library/nginx
# abc123def456: Pull complete               ← Capa descargada
# def789abc123: Already exists              ← Capa ya en caché local
# ghi456jkl789: Pull complete
# mno123pqr456: Pull complete
# Digest: sha256:d5b2b2e4f4...             ← Digest de la imagen
# Status: Downloaded newer image for nginx:1.25-alpine
# docker.io/library/nginx:1.25-alpine

# Ver el digest de la imagen descargada (referencia canónica)
docker image inspect nginx:1.25-alpine --format='{{range .RepoDigests}}{{.}}{{"\n"}}{{end}}'
# Output: nginx@sha256:d5b2b2e4f4...

# Este digest ES inmutable: aunque cambien el tag "1.25-alpine",
# el digest @sha256:... siempre apuntará a esta versión exacta.
```

### 3.3 Push: Publicar imágenes en un registry

```bash
# 1. Autenticarse
docker login
# Username: tuusuario
# Password: (usa un token de acceso personal, NO tu contraseña real)
# Recomendación de seguridad: crear token en hub.docker.com → Settings → Security

# Alternativa: usar token por stdin (más seguro para CI/CD)
echo "$DOCKER_HUB_TOKEN" | docker login --username tuusuario --password-stdin

# 2. Etiquetar la imagen con el namespace destino
docker tag mi-app:v1 tuusuario/mi-app:v1

# 3. Empujar
docker push tuusuario/mi-app:v1

# Output:
# The push refers to repository [docker.io/tuusuario/mi-app]
# abc123: Pushed               ← Capa nueva, se subió
# def456: Pushed
# ghi789: Mounted from library/alpine  ← Capa que YA existe en otro repo del registry
# v1: digest: sha256:xyz... size: 1234
```

#### Anatomía del push

```
┌──────────────────────────────────────────────────────┐
│  PASO 1: DETERMINAR QUÉ CAPAS HAY QUE SUBIR          │
│                                                      │
│  Docker consulta al registry: ¿existe este blob?     │
│  HEAD /v2/<repo>/blobs/<digest>                      │
│                                                      │
│  200 OK → La capa YA existe → "Mounted from"         │
│  404 Not Found → Hay que subirla → "Pushed"          │
│                                                      │
│  Optimización: cross-repository blob mount            │
│  POST /v2/<repo>/blobs/uploads/?mount=<digest>       │
│       &from=<source-repo>                             │
│  Si el blob existe en el source repo del mismo        │
│  registry, se "monta" en el nuevo repo sin transferir.│
└──────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  PASO 2: SUBIR CAPAS NUEVAS (paralelo, hasta 5)      │
│                                                      │
│  Para cada capa que no existe en el registry:         │
│                                                      │
│  1. Iniciar upload:                                   │
│     POST /v2/<repo>/blobs/uploads/                    │
│     ← 202 Accepted, Location: <upload-url>           │
│                                                      │
│  2. Enviar datos (streaming):                         │
│     PATCH <upload-url>                                │
│     Body: capa comprimida (tar.gz)                    │
│     ← 202 Accepted, Location: <upload-url>           │
│                                                      │
│  3. Finalizar upload:                                 │
│     PUT <upload-url>?digest=sha256:<hash>             │
│     ← 201 Created                                     │
│                                                      │
│  El registry verifica el digest al finalizar.         │
│  Si el digest no coincide, rechaza el upload.         │
└──────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  PASO 3: SUBIR EL CONFIG BLOB                         │
│                                                      │
│  Mismo proceso de 3 pasos (POST/PATCH/PUT).           │
└──────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│  PASO 4: SUBIR EL MANIFEST                            │
│                                                      │
│  PUT /v2/<repo>/manifests/<tag>                       │
│  Body: manifest JSON                                  │
│  Content-Type: application/vnd.oci.image.manifest...  │
│                                                      │
│  El registry verifica:                                │
│    - Todos los blobs referenciados existen            │
│    - El manifest es válido (schema correcto)          │
│                                                      │
│  ← 201 Created                                        │
│     Location: /v2/<repo>/manifests/<digest>           │
│                                                      │
│  ¡La imagen está publicada!                           │
└──────────────────────────────────────────────────────┘
```

### 3.4 Tags en profundidad: La guía definitiva

Los tags son la interfaz más visible y más malentendida de Docker. Veamos cada aspecto en detalle.

#### `latest`: El tag más peligroso de Docker

```
MITO:    latest = "la versión más reciente"
REALIDAD: latest = "la última imagen construida SIN tag explícito"

┌─────────────────────────────────────────────────────────────┐
│  COMPORTAMIENTO REAL DE latest:                             │
│                                                             │
│  docker build -t mi-app .         → mi-app:latest           │
│  docker build -t mi-app:v2 .      → mi-app:v2               │
│                                     (latest NO se actualiza) │
│  docker build -t mi-app:v2 \                               │
│               -t mi-app:latest .   → mi-app:v2 + latest     │
│                                                             │
│  docker pull mi-app                → mi-app:latest          │
│    (Si no especificas tag, Docker asume latest)             │
│    (Esto es CONVENCIÓN del daemon, no del registry)         │
└─────────────────────────────────────────────────────────────┘
```

`latest` es simplemente el tag por defecto cuando no especificas ninguno. En Docker Hub, los mantenedores DEBEN asignar manualmente el tag `latest` a la versión que quieran que los usuarios obtengan por defecto. Si un mantenedor no actualiza `latest`, los usuarios obtienen una versión antigua sin saberlo.

**Por qué `latest` es peligroso en producción:**

```yaml
# ❌ PELIGROSO: imagen flotante
services:
  web:
    image: nginx:latest
# Problemas:
# 1. Irreproducible: cada docker pull puede traer una versión diferente
# 2. Rollback imposible: no sabes qué versión tenías antes
# 3. Divergencia en cluster: nodos diferentes pueden tener versiones distintas
# 4. Debugging imposible: "¿qué versión de nginx está corriendo?"
# 5. El registry puede cachear el manifest y entregar versiones diferentes
#    a diferentes nodos durante una actualización gradual

# ✅ SEGURO: versión explícita
services:
  web:
    image: nginx:1.25.3-alpine
# Ventajas:
# 1. Reproducible: siempre la misma versión
# 2. Rollback trivial: cambiar el tag y redesplegar
# 3. Consistencia en cluster: todos los nodos tienen la misma
# 4. Auditable: sabes exactamente qué está corriendo
# 5. Actualizaciones controladas: tú decides cuándo cambiar de versión
```

#### Versionado semántico con tags

El convenio más común es usar versionado semántico (SemVer) para los tags:

```
┌─────────────────────────────────────────────────────────────┐
│                   ÁRBOL DE TAGS SEMVER                       │
│                                                             │
│  nginx:1          ← Major version más reciente (1.x.x)      │
│  nginx:1.25       ← Minor version más reciente (1.25.x)     │
│  nginx:1.25.3     ← Patch version exacta (1.25.3)           │
│                                                             │
│  nginx:1.25.3-alpine    ← Patch + SO específico             │
│  nginx:1.25-alpine      ← Minor + SO                        │
│  nginx:1-alpine         ← Major + SO                        │
│  nginx:alpine           ← Última versión en Alpine (flotante)│
│                                                             │
│  nginx:stable        ← Rama LTS (long-term support)         │
│  nginx:mainline      ← Rama principal (últimas features)    │
└─────────────────────────────────────────────────────────────┘
```

**Recomendación de versionado para producción:**

```
✅ USA tags de patch exacta (1.25.3) para producción
✅ USA tags de minor (1.25) para staging
✅ USA tags de major (1) para desarrollo local
❌ NUNCA uses latest ni tags flotantes en producción
```

#### Sufijos de sistema operativo: ¿Cuál elegir?

```
┌───────────────────────────────────────────────────────────────────┐
│                 VARIANTES DE SISTEMA OPERATIVO                     │
│                                                                   │
│  TAG               │ BASE       │ TAMAÑO   │ GESTOR PKG │ USO    │
│  ──────────────────┼────────────┼──────────┼────────────┼────────│
│  nginx             │ Debian     │ ~150 MB  │ apt        │ Genérico│
│  nginx:alpine      │ Alpine     │ ~7 MB    │ apk        │ Rápido │
│  nginx:slim        │ Debian(min)│ ~80 MB   │ apt        │ Medio  │
│  nginx:bookworm    │ Debian 12  │ ~130 MB  │ apt        │ Específico│
│  nginx:bullseye    │ Debian 11  │ ~130 MB  │ apt        │ Específico│
│  nginx:buster      │ Debian 10  │ ~120 MB  │ apt        │ Legacy │
│  python:slim-bullseye│ Combinación slim + Debian específico    │
│                                                                   │
│  REGLA GENERAL:                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ alpine → Imágenes más pequeñas, arranque más rápido.         │ │
│  │          Recomendado para: Go, Rust, Node.js simple,         │ │
│  │          microservicios, CI/CD, serverless.                  │ │
│  │          Contra: musl libc (no glibc), menos paquetes.      │ │
│  │                                                              │ │
│  │ slim   → Término medio. Debian sin herramientas de build.    │ │
│  │          Recomendado para: Python, Ruby, Node.js con         │ │
│  │          dependencias nativas.                               │ │
│  │                                                              │ │
│  │ debian → Máxima compatibilidad. Todos los paquetes estándar. │ │
│  │          Recomendado para: aplicaciones legacy,              │ │
│  │          dependencias complejas, entornos empresariales.     │ │
│  └──────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

#### Diferencias técnicas entre Alpine (musl) y Debian (glibc)

```
┌─────────────────────────────────────────────────────────────────┐
│  Alpine usa musl libc en lugar de glibc. Esto tiene implicaciones:│
│                                                                 │
│  1. Binarios compilados contra glibc NO funcionan en Alpine.    │
│     → Solución: compilar estáticamente o en Alpine.             │
│                                                                 │
│  2. Resolución DNS diferente: musl usa el resolver de forma     │
│     secuencial, glibc en paralelo. En Kubernetes, esto puede    │
│     causar timeouts si no configuras bien el DNS.               │
│     → Solución: usar ndots:1 en dnsConfig de Kubernetes.       │
│                                                                 │
│  3. Zona horaria: Alpine no trae tzdata por defecto.            │
│     → Solución: RUN apk add --no-cache tzdata                   │
│     → Y ENV TZ=America/Mexico_City                              │
│                                                                 │
│  4. Alpine usa BusyBox: los comandos de shell (awk, sed, grep)   │
│     pueden tener flags diferentes a GNU coreutils.               │
│                                                                 │
│  CUÁNDO EVITAR Alpine:                                          │
│  - Paquetes Python con ruedas C precompiladas para manylinux    │
│  - Binarios distribuidos solo para glibc                        │
│  - Aplicaciones que requieren GNU coreutils específicos         │
│  - Si no puedes compilar estáticamente y necesitas glibc        │
└─────────────────────────────────────────────────────────────────┘
```

### 3.5 Construir imágenes multi-arch con buildx

```bash
# 1. Verificar que buildx está disponible
docker buildx version

# 2. Crear un builder multi-plataforma
docker buildx create --name multiarch --use
# --use lo establece como builder por defecto

# 3. Verificar e inicializar el builder
docker buildx inspect --bootstrap
# Output: muestra las plataformas soportadas (linux/amd64, linux/arm64, etc.)

# 4. Construir para múltiples arquitecturas y empujar
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  --tag tuusuario/mi-app:v1 \
  --push \
  .

# Flags importantes:
# --platform: Una o más arquitecturas (separadas por coma)
# --push:     Empuja directamente al registry (crea el manifest list)
# --load:     Carga en el daemon local (SOLO funciona con UNA arquitectura)
# --output:   type=oci,dest=archivo.tar (exporta a OCI layout)
# --cache-to/--cache-from: Gestión de caché entre CI/CD

# Para desarrollo local (solo la arquitectura actual):
docker buildx build --load -t mi-app:dev .
```

#### ¿Cómo funciona buildx internamente?

```
┌───────────────────────────────────────────────────────────────┐
│                     docker buildx build                        │
│                                                               │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ Builder ARM64   │  │ Builder AMD64│  │ Builder ARM/v7   │ │
│  │                 │  │              │  │                  │ │
│  │  QEMU user-mode │  │  Nativo      │  │  QEMU user-mode  │ │
│  │  (emulación)    │  │  (host)      │  │  (emulación)     │ │
│  │                 │  │              │  │                  │ │
│  │  FROM alpine    │  │  FROM alpine │  │  FROM alpine     │ │
│  │  RUN apk add... │  │  RUN apk ... │  │  RUN apk add...  │ │
│  │  COPY app .     │  │  COPY app .  │  │  COPY app .      │ │
│  │                 │  │              │  │                  │ │
│  │  → Imagen ARM64 │  │  → Imagen    │  │  → Imagen ARM/v7 │ │
│  │     única       │  │     AMD64    │  │     única        │ │
│  └────────┬────────┘  └──────┬───────┘  └───────┬──────────┘ │
│           │                  │                   │             │
│           └──────────────────┼───────────────────┘             │
│                              ▼                                 │
│              ┌───────────────────────────────┐                │
│              │       MANIFEST LIST           │                │
│              │                               │                │
│              │  linux/amd64  → sha256:aaa... │                │
│              │  linux/arm64  → sha256:bbb... │                │
│              │  linux/arm/v7 → sha256:ccc... │                │
│              └───────────────────────────────┘                │
│                              │                                 │
│                              ▼                                 │
│              ┌───────────────────────────────┐                │
│              │    REGISTRY (Docker Hub)      │                │
│              │                               │                │
│              │  tuusuario/mi-app:v1          │                │
│              │  → Manifest List +            │                │
│              │    3 conjuntos de capas       │                │
│              └───────────────────────────────┘                │
└───────────────────────────────────────────────────────────────┘
```

---
## 4. Registries Privados

No todas las imágenes deben ser públicas. Para código propietario, compliance empresarial, o simplemente para reducir latencia, necesitas un registry privado. Veamos las opciones principales.

### 4.1 Docker Hub: repositorios privados

Docker Hub ofrece repositorios privados incluso en su plan gratuito (1 repositorio privado). Para equipos, los planes de pago ofrecen más repos, escaneo de vulnerabilidades y builds automáticas.

```bash
# 1. Login
docker login --username tuusuario

# 2. Push a repositorio privado (debe existir en Docker Hub)
docker tag mi-app:v1 tuusuario/mi-app-privada:v1
docker push tuusuario/mi-app-privada:v1

# 3. Pull desde otro lugar (requiere login)
docker pull tuusuario/mi-app-privada:v1
```

### 4.2 AWS Elastic Container Registry (ECR)

ECR es el registry de imágenes totalmente gestionado de AWS. Se integra con IAM, ofrece escaneo de vulnerabilidades, y funciona dentro de VPCs sin tráfico a internet.

#### Crear repositorio

```bash
# Crear repositorio con escaneo automático
aws ecr create-repository \
  --repository-name mi-app \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

# Output:
# {
#   "repository": {
#     "repositoryUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/mi-app",
#     "registryId": "123456789012",
#     ...
#   }
# }

# Ver URI del repositorio
aws ecr describe-repositories --repository-name mi-app \
  --query 'repositories[0].repositoryUri' --output text
```

#### Autenticación con ECR

```bash
# Método 1: get-login-password (recomendado, token de 12 horas)
aws ecr get-login-password --region us-east-1 | \
  docker login \
    --username AWS \
    --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com

# Método 2: ECR Docker Credential Helper (transparente)
# Instalar: brew install docker-credential-helper-ecr (macOS)
# O: https://github.com/awslabs/amazon-ecr-credential-helper/releases
# Configurar ~/.docker/config.json:
# {
#   "credHelpers": {
#     "123456789012.dkr.ecr.us-east-1.amazonaws.com": "ecr-login"
#   }
# }
# Ahora docker pull/push funciona sin login explícito.

# Método 3: Script en CI/CD para renovación automática
# Cada 12 horas en el pipeline:
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ${ECR_URI}
```

#### Push y Pull

```bash
# Taggear con la URI completa
docker tag mi-app:v1 123456789012.dkr.ecr.us-east-1.amazonaws.com/mi-app:v1

# Push
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/mi-app:v1

# Pull
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/mi-app:v1
```

#### Características avanzadas de ECR

**1. Lifecycle Policies (limpieza automática):**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Mantener solo 30 imágenes con tag prod",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod"],
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Eliminar imágenes sin tag tras 14 días",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 14
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 3,
      "description": "Limpiar imágenes dev cada 7 días",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["dev"],
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
```

```bash
# Aplicar la política
aws ecr put-lifecycle-policy \
  --repository-name mi-app \
  --lifecycle-policy-text file://lifecycle-policy.json \
  --region us-east-1
```

**2. Escaneo de vulnerabilidades:**

```bash
# Ver resultados del escaneo
aws ecr describe-image-scan-findings \
  --repository-name mi-app \
  --image-id imageTag=v1 \
  --region us-east-1

# Ver resumen de severidad
aws ecr describe-image-scan-findings \
  --repository-name mi-app \
  --image-id imageTag=v1 \
  --query 'imageScanFindings.findingSeverityCounts'

# Output: {"CRITICAL": 0, "HIGH": 2, "MEDIUM": 5, "LOW": 10}
```

**3. Replicación cross-region:**

```bash
# Configurar replicación (desde la consola o CloudFormation)
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [{
      "destinations": [{
        "region": "us-west-2",
        "registryId": "123456789012"
      },
      {
        "region": "eu-west-1",
        "registryId": "123456789012"
      }]
    }]
  }'
```

#### Ventajas comparativas de ECR

| Característica | Descripción |
|----------------|-------------|
| **IAM Integration** | Políticas granulares: quién puede pull/push/qué imágenes. Ej: solo pods del namespace `prod` en EKS. |
| **Image Scanning** | Escaneo automático al push. Usa la base de datos de vulnerabilidades (CVEs) de AWS Inspector. |
| **Lifecycle Policies** | Limpieza automática basada en reglas: antigüedad, conteo, tags. |
| **Cross-Region Replication** | Replica imágenes a otras regiones para despliegues globales de baja latencia. |
| **Immutable Tags** | Un tag no puede ser sobrescrito (configurable). Evita accidentes en producción. |
| **VPC Endpoints** | Acceso privado sin pasar por internet público (VPC Endpoint para ECR). |
| **KMS Encryption** | Encriptación con claves gestionadas por el cliente (CMK). |
| **Pull-Through Cache** | Cachea imágenes de registries públicos (Docker Hub, GHCR) en ECR privado. |

### 4.3 Google Artifact Registry (GAR)

Google Artifact Registry es la evolución moderna de Google Container Registry (GCR). Soporta no solo imágenes Docker, sino también paquetes de lenguajes (Maven, npm, Python, Apt, Yum).

```bash
# 1. Habilitar la API
gcloud services enable artifactregistry.googleapis.com

# 2. Crear un repositorio Docker
gcloud artifacts repositories create mi-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Repositorio de imágenes de producción"

# 3. Configurar autenticación
gcloud auth configure-docker us-central1-docker.pkg.dev
# Esto añade el credential helper a ~/.docker/config.json

# 4. Formato de la URI
# LOCATION-docker.pkg.dev/PROJECT-ID/REPO-NAME/IMAGE-NAME:TAG
# Ejemplo: us-central1-docker.pkg.dev/mi-proyecto/mi-repo/mi-app:v1

# 5. Taggear y push
docker tag mi-app:v1 us-central1-docker.pkg.dev/mi-proyecto/mi-repo/mi-app:v1
docker push us-central1-docker.pkg.dev/mi-proyecto/mi-repo/mi-app:v1

# 6. Pull
docker pull us-central1-docker.pkg.dev/mi-proyecto/mi-repo/mi-app:v1
```

#### Características de GAR

- **Artifact Analysis:** Escaneo automático de vulnerabilidades con resultados accesibles vía API.
- **IAM granular:** `roles/artifactregistry.reader`, `roles/artifactregistry.writer`.
- **VPC Service Controls:** Acceso restringido por perímetro VPC.
- **Multi-region:** Repositorios multi-región para alta disponibilidad global.
- **Cleanup policies:** Eliminación automática por antigüedad o prefijo de tag.
- **Integración con Cloud Build y GKE:** Sin necesidad de imagePullSecrets.

### 4.4 Azure Container Registry (ACR)

```bash
# 1. Crear un ACR
az acr create \
  --resource-group mi-grupo \
  --name MiRegistroACR \
  --sku Standard \
  --admin-enabled true

# 2. Login
az acr login --name MiRegistroACR
# Esto usa el credential helper de Azure

# 3. Push
docker tag mi-app:v1 miregistroacr.azurecr.io/mi-app:v1
docker push miregistroacr.azurecr.io/mi-app:v1

# 4. Pull
docker pull miregistroacr.azurecr.io/mi-app:v1
```

#### SKUs de ACR

| SKU | Almacenamiento | Throughput | Características principales | Precio aprox. |
|-----|---------------|------------|----------------------------|---------------|
| **Basic** | 10 GB incluido | Bajo | Desarrollo y pruebas | ~$5/mes |
| **Standard** | 100 GB incluido | Medio | Producción básica | ~$20/mes |
| **Premium** | Ilimitado | Alto | Geo-replicación, Private Link, escaneo de vuln., Content Trust | ~$100/mes |

#### Características Premium de ACR

- **Content Trust:** Firmado digital de imágenes integrado con Docker Notary.
- **Tasks:** Construir imágenes en la nube sin CI/CD propio (`az acr build`).
- **Geo-replicación:** Un solo registry con réplicas en múltiples regiones.
- **Private Link:** Acceso privado vía Azure Private Endpoint.
- **CMK Encryption:** Encriptación con claves gestionadas por el cliente.

### 4.5 Harbor: Registry empresarial open source

Harbor es el registry de código abierto más completo. Es un proyecto graduado de la CNCF y se usa en entornos empresariales que requieren control total sobre sus imágenes.

#### Características principales

| Característica | Descripción |
|----------------|-------------|
| **Escaneo de vulnerabilidades** | Integrado con Trivy (predeterminado) o Clair. Escaneo automático al push y programado. |
| **RBAC** | Proyectos, roles (admin, master, developer, guest), miembros. |
| **Replicación** | Push-based y pull-based entre instancias Harbor. Filtros por proyecto/repositorio/tag. |
| **Firmado de imágenes** | Integración con Notary para Docker Content Trust. |
| **Proxy Cache** | Cachea imágenes de Docker Hub, GCR, GHCR, etc. Reduce tráfico externo y acelera pulls. |
| **Garbage Collection** | Automático y programable. Elimina blobs huérfanos. |
| **Webhooks** | Notificaciones HTTP para integración con CI/CD. |
| **API REST** | API completa para automatización de todo. |
| **OIDC/LDAP/AD** | Autenticación con proveedores de identidad empresariales. |
| **Helm Charts** | Soporte para repositorio de Helm charts (via Harbor chartmuseum). |
| **P2P Distribution** | Dragonfly integrado para distribución eficiente en clusters grandes de Kubernetes. |

#### Desplegar Harbor con Docker Compose

```bash
# 1. Descargar Harbor (offline installer incluye todas las dependencias)
HARBOR_VERSION=v2.10.0
wget https://github.com/goharbor/harbor/releases/download/${HARBOR_VERSION}/harbor-offline-installer-${HARBOR_VERSION}.tgz
tar xzf harbor-offline-installer-${HARBOR_VERSION}.tgz
cd harbor

# 2. Configurar
cp harbor.yml.tmpl harbor.yml

# Editar harbor.yml (valores mínimos):
# hostname: harbor.mi-empresa.com
# http:
#   port: 80
# https:
#   port: 443
#   certificate: /ruta/al/cert.pem
#   private_key: /ruta/al/key.pem
# harbor_admin_password: ContraseñaSegura123!
# database:
#   password: OtraContraseñaSegura456!

# 3. Instalar con Trivy (escaneo de vulnerabilidades)
sudo ./install.sh --with-trivy --with-chartmuseum

# 4. Acceder vía web
# https://harbor.mi-empresa.com
# Usuario: admin
# Password: ContraseñaSegura123!

# 5. Configurar Docker para confiar en el registry (si es self-signed)
sudo mkdir -p /etc/docker/certs.d/harbor.mi-empresa.com
sudo cp cert.pem /etc/docker/certs.d/harbor.mi-empresa.com/ca.crt
sudo systemctl restart docker

# 6. Login y uso
docker login harbor.mi-empresa.com
docker tag mi-app:v1 harbor.mi-empresa.com/mi-proyecto/mi-app:v1
docker push harbor.mi-empresa.com/mi-proyecto/mi-app:v1
```

#### Estructura de proyectos y roles en Harbor

```
┌───────────────────────────────────────────────────────────────┐
│                         HARBOR                                │
│                                                               │
│  ┌─────────────────────────┐  ┌─────────────────────────┐    │
│  │ Proyecto: prod          │  │ Proyecto: dev           │    │
│  │                         │  │                         │    │
│  │ 👤 admin (admin)        │  │ 👤 developer (develop)  │    │
│  │ 👤 equipo-infra (master)│  │ 👤 pasante (guest)     │    │
│  │ 👤 pipeline (developer) │  │                         │    │
│  │ 👤 auditor (guest)     │  │ 📦 api-backend          │    │
│  │                         │  │ 📦 api-frontend         │    │
│  │ 📦 api-backend          │  │ 📦 worker               │    │
│  │ 📦 api-frontend         │  │                         │    │
│  │ 📦 worker               │  │ Política:               │    │
│  │ 📦 migrations           │  │ - Auto-scan: ON        │    │
│  │                         │  │ - Preventar CVEs HIGH+ │    │
│  │ Política:               │  │ - Retención: 20 tags   │    │
│  │ - Auto-scan: ON         │  │ - Inmutabilidad: OFF   │    │
│  │ - Preventar CVEs: HIGH+│  │                         │    │
│  │ - Retención: 30 tags   │  │                         │    │
│  │ - Inmutabilidad: ON     │  │                         │    │
│  └─────────────────────────┘  └─────────────────────────┘    │
│                                                               │
│  Roles:                                                       │
│  ┌───────────┬──────────────────────────────────────────────┐ │
│  │ admin     │ Control total del proyecto                   │ │
│  │ master    │ Push/Pull + Gestión de miembros + Escaneos   │ │
│  │ developer │ Push/Pull (sin gestión administrativa)       │ │
│  │ guest     │ Solo Pull                                   │ │
│  └───────────┴──────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

#### Configurar replicación entre instancias Harbor

```bash
# En Harbor primario (prod):
# Administration → Replications → New Endpoint
# Provider: Harbor
# Name: harbor-dr
# Endpoint URL: https://harbor-dr.mi-empresa.com
# Access ID: admin
# Access Secret: <token o contraseña>

# Luego, en Projects → mi-proyecto → Replications → New Replication Rule:
# Name: replicar-a-dr
# Replication mode: Push-based  (empuja al DR)
# Source resource filter: All resources
# Trigger mode: Event-based  (cada push replica)
# Delete remote resources when locally deleted: ON
```

### 4.6 GitLab Container Registry

GitLab integra un registry de contenedores en CADA proyecto. Es automático: no necesitas crear repositorios separados.

```bash
# 1. Login
docker login registry.gitlab.com
# Usa un Personal Access Token (PAT) con scope read_registry + write_registry

# 2. Taggear con el formato de GitLab
docker tag mi-app:v1 registry.gitlab.com/mi-usuario/mi-proyecto/mi-app:v1

# 3. Push
docker push registry.gitlab.com/mi-usuario/mi-proyecto/mi-app:v1
```

#### Integración con GitLab CI/CD

```yaml
# .gitlab-ci.yml
variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

stages:
  - build
  - test
  - push

build:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN $CI_REGISTRY
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  # $CI_REGISTRY_USER y $CI_JOB_TOKEN son variables predefinidas
  # $CI_JOB_TOKEN es un token temporal para este job específico

push-to-prod:
  stage: push
  only:
    - tags
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN $CI_REGISTRY
    - docker pull $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker tag  $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:prod
    - docker push $CI_REGISTRY_IMAGE:prod
```

#### Variables predefinidas de GitLab Registry

| Variable | Valor | Descripción |
|----------|-------|-------------|
| `$CI_REGISTRY` | `registry.gitlab.com` | Host del registry |
| `$CI_REGISTRY_USER` | `gitlab-ci-token` | Usuario para autenticación |
| `$CI_JOB_TOKEN` | Token temporal | Contraseña para autenticación (válida solo durante el job) |
| `$CI_REGISTRY_IMAGE` | `registry.gitlab.com/grupo/proyecto` | URI base de la imagen |
| `$CI_COMMIT_SHA` | SHA del commit | Identificador único del commit |
| `$CI_COMMIT_TAG` | Nombre del tag | Solo cuando el pipeline se ejecuta por un tag de Git |

### 4.7 GitHub Container Registry (ghcr.io)

```bash
# 1. Autenticarse con Personal Access Token (PAT) de GitHub
# Scopes necesarios: read:packages, write:packages, delete:packages
echo $GITHUB_PAT | docker login ghcr.io -u tuusuario --password-stdin

# 2. Taggear
docker tag mi-app:v1 ghcr.io/tuusuario/mi-app:v1
# Para repos de organizaciones:
docker tag mi-app:v1 ghcr.io/mi-organizacion/mi-app:v1

# 3. Push
docker push ghcr.io/tuusuario/mi-app:v1

# 4. Hacer pública la imagen (por defecto son privadas)
# Vía GitHub UI: Settings → Packages → Seleccionar paquete → Change visibility
```

#### Integración con GitHub Actions

```yaml
# .github/workflows/build-push.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*.*.*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

### 4.8 Tabla comparativa completa de registries

| Registry | Costo típico (GB/mes) | Escaneo vuln. | RBAC granular | Replicación | CI/CD integrado | Auto-build | Caso de uso ideal |
|----------|------------------------|---------------|---------------|-------------|-----------------|------------|-------------------|
| **Docker Hub** | $0 (1 privado) / $5+ (Pro) | Pro/Team | Equipos | No | No (webhooks) | Sí | Individuos, equipos pequeños OSS |
| **AWS ECR** | $0.10/GB + transfer | Inspector (basic) | IAM policies | Cross-region | CodeBuild | No | AWS ecosystem (ECS, EKS, Lambda) |
| **GCP GAR** | $0.10/GB + transfer | Artifact Analysis | IAM | Multi-region | Cloud Build | No | GCP ecosystem (GKE, Cloud Run) |
| **Azure ACR** | $0.167/hora (Basic) | Premium tier | Azure AD RBAC | Premium only | Azure Pipelines | Sí (ACR Tasks) | Azure ecosystem (AKS, App Service) |
| **Harbor** | $0 (open source) | Trivy/Clair | Sí (proyectos) | Sí (push/pull) | Webhooks | No | Empresas on-premise, multi-nube, compliance |
| **GitLab Registry** | Incluido en GitLab | Ultimate tier | Roles GitLab | Geo | Sí (nativo) | Sí | Equipos con GitLab CI/CD |
| **GitHub GHCR** | Gratis (público) / Incluido | Dependabot | Equipos/orgs | No | Actions (nativo) | Sí | Equipos con GitHub Actions |
| **JFrog Artifactory** | Licencia empresarial | Xray | Completo | Sí | Plugins | No | Grandes empresas, multi-tecnología (Docker, Maven, npm, PyPI...) |
| **Red Hat Quay** | Open source / RH Subscription | Clair | Sí (equipos) | Sí | Webhooks | Sí | OpenShift ecosystem, entornos regulados |

---

## 5. Gestión de imágenes

### 5.1 `docker images`: Listar, filtrar y formatear

```bash
# Listar todas las imágenes
docker images

# Output:
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# nginx        alpine    abc123def456   2 weeks ago    42MB
# node         20-slim   def456abc789   3 weeks ago    250MB
# mi-app       v1        ghi789def012   5 days ago     450MB
# mi-app       latest    ghi789def012   5 days ago     450MB
# <none>       <none>    jkl012ghi345   1 week ago     400MB

# Explicación de las columnas:
# REPOSITORY:  Nombre del repositorio. <none> = imagen "dangling"
# TAG:         Etiqueta. <none> = dangling (sin tag)
# IMAGE ID:    Hash único. Mismo ID = misma imagen con diferentes tags
# CREATED:     Cuándo se construyó la imagen (no cuándo se descargó)
# SIZE:        Tamaño acumulado (sin contar compartición de capas)
```

#### Filtrado avanzado

```bash
# Filtros por referencia
docker images --filter "reference=nginx"              # Nombre exacto
docker images --filter "reference=mi-app:v1"          # Nombre:tag
docker images --filter "reference=*:alpine"           # Wildcard en tag
docker images --filter "reference=mi-app*"            # Wildcard en nombre

# Filtros por etiqueta (label)
docker images --filter "label=maintainer=yo@ejemplo.com"
docker images --filter "label=com.example.version=1.0"

# Filtros por estado
docker images --filter "dangling=true"                # Solo dangling
docker images --filter "dangling=false"               # Solo con tag

# Filtros temporales
docker images --filter "before=mi-app:v2"             # Creadas antes que este tag
docker images --filter "since=nginx:alpine"           # Creadas después que esta

# Combinar filtros
docker images --filter "reference=mi-app*" --filter "dangling=false"

# Solo IDs (muy útil para scripts)
docker images -q                                    # Todos los IDs
docker images -q --filter "dangling=true"          # IDs de dangling
docker images -q mi-app                            # IDs de un repositorio
```

#### Formato personalizado con Go templates

```bash
# Formato tabular personalizado
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedAt}}"

# Solo nombres con tag
docker images --format "{{.Repository}}:{{.Tag}}"

# Tamaño en formato legible
docker images --format "{{.Repository}}:{{.Tag}} -> {{.Size}}"

# ID + nombre
docker images --format "{{.ID}}: {{.Repository}}:{{.Tag}}"

# Output en JSON para scripting
docker images --format '{{json .}}'
docker images --format '{{json .}}' | while read line; do
  repo=$(echo $line | jq -r .Repository)
  tag=$(echo $line | jq -r .Tag)
  echo "Imagen: $repo:$tag"
done
```

### 5.2 `docker rmi`: Eliminar imágenes correctamente

```bash
# Eliminar por tag
docker rmi mi-app:v1

# Eliminar por ID (se pueden especificar varios)
docker rmi abc123def456 def789abc123

# Eliminar por ID abreviado (los primeros 12 caracteres son suficientes)
docker rmi abc123def456

# Forzar eliminación (incluso si está en uso)
docker rmi -f mi-app:v1

# Eliminar TODAS las imágenes de un repositorio
docker rmi $(docker images -q mi-app)

# Eliminar TODAS las imágenes del sistema
docker rmi $(docker images -q)
```

#### Errores comunes y soluciones

```bash
# Error: "image is referenced in multiple repositories"
# Causa: Misma imagen con varios tags en diferentes repositorios
# Solución: eliminar por tag específico en lugar de por ID
docker rmi mi-app:v1
docker rmi mi-app:latest

# Error: "image is being used by running container <hash>"
# Causa: Hay un contenedor (activo o detenido) usando esta imagen
# Solución 1: detener y eliminar el contenedor
docker stop CONTAINER_ID
docker rm CONTAINER_ID
docker rmi mi-app:v1

# Solución 2: forzar (el contenedor queda huérfano)
docker rmi -f mi-app:v1

# Error: "conflict: unable to remove repository reference"
# Causa: La imagen tiene capas hijas (otras imágenes dependen de ella)
# Solución: eliminar las imágenes hijas primero
docker image inspect --format='{{.Parent}}' IMAGE_ID   # Encontrar la imagen hija
docker rmi HIJA_ID
docker rmi IMAGEN_PADRE_ID
```

### 5.3 `docker image prune`: Limpieza automática

```bash
# Eliminar SOLO imágenes dangling (<none>:<none>)
docker image prune

# Eliminar TODAS las imágenes no usadas por al menos UN contenedor
docker image prune -a

# Eliminar sin pedir confirmación
docker image prune -a -f

# Filtrar por antigüedad
docker image prune -a --filter "until=24h"             # > 24 horas
docker image prune -a --filter "until=2024-01-01"      # Antes de esta fecha
docker image prune -a --filter "until=168h"            # > 7 días

# Ver qué se eliminaría sin eliminar realmente
docker image prune -a --dry-run

# Diferencia entre los modos:
# docker image prune      → Solo <none>:<none> (dangling)
# docker image prune -a   → Todas las no referenciadas por contenedores
# docker image prune -a -f → Lo mismo sin confirmación
```

#### Tipos de "basura" de imágenes

```
┌─────────────────────────────────────────────────────────────┐
│  IMAGEN DANGLING: <none>:<none>                              │
│                                                              │
│  REPOSITORY: <none>    TAG: <none>                           │
│                                                              │
│  Causa típica: Reconstruiste una imagen con el mismo tag.    │
│  La nueva imagen toma el tag, la vieja queda sin tag.        │
│                                                              │
│  Ejemplo:                                                    │
│    docker build -t mi-app:v1 .      → Imagen ID: AAAA       │
│    # Modificas algo...                                       │
│    docker build -t mi-app:v1 .      → Imagen ID: BBBB       │
│    # AAAA queda como <none>:<none>                           │
│                                                              │
│  Se eliminan con: docker image prune                         │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  IMAGEN NO USADA (UNUSED):                                   │
│                                                              │
│  Tiene tag pero ningún contenedor (activo ni detenido) la    │
│  referencia.                                                 │
│                                                              │
│  SOLO se eliminan con: docker image prune -a                 │
└──────────────────────────────────────────────────────────────┘
```

### 5.4 `docker system df`: Análisis de uso de disco

```bash
# Resumen compacto
docker system df

# Output:
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          25        12        12.5GB    8.2GB (65%)
# Containers      10        3         500MB     400MB (80%)
# Local Volumes   15        8         2.3GB     1.1GB (47%)
# Build Cache     50        0         3.5GB     3.5GB (100%)

# COLUMNAS:
# TOTAL:       Cantidad total de objetos
# ACTIVE:      En uso actualmente (contenedores activos, imágenes usadas, etc.)
# SIZE:        Espacio total ocupado
# RECLAIMABLE: Espacio que se puede liberar (no activo = basura)

# Versión verbose (detalle por imagen y build cache)
docker system df -v

# Output detallado para imágenes:
# Images space usage:
# REPOSITORY    TAG       IMAGE ID       SHARED   UNIQUE   SIZE
# mi-app        v1        abc123def456   410MB    50MB     460MB
# nginx         alpine    8a9b0c1d2e3f   0MB      42MB     42MB
#
# SHARED: Espacio compartido con otras imágenes (capas comunes)
# UNIQUE: Espacio que solo ocupa esta imagen (capas propias)
# SIZE = SHARED + UNIQUE
#
# NOTA: SHARED se cuenta en cada imagen, pero en disco SOLO ocupa
#        espacio UNA VEZ. Por eso la suma de SIZE > espacio real.
```

#### Estrategia de limpieza automatizada

```bash
#!/bin/bash
# cleanup-docker.sh — Ejecutar en cron semanal

# 1. Eliminar contenedores detenidos hace más de 7 días
docker container prune -f --filter "until=168h"

# 2. Eliminar imágenes no usadas (conserva las de los últimos 14 días)
docker image prune -a -f --filter "until=336h"

# 3. Eliminar volúmenes no usados (¡cuidado con los datos!)
docker volume prune -f --filter "until=168h"

# 4. Eliminar build cache completo
docker builder prune -a -f

# 5. Eliminar redes no usadas
docker network prune -f

# 6. Informe de espacio liberado
docker system df
```

### 5.5 `docker image history`: Ver el historial de capas

```bash
# Ver cómo se construyó una imagen
docker image history nginx:alpine

# Output:
# IMAGE          CREATED       CREATED BY                          SIZE      COMMENT
# abc123def456   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]    0B
# <missing>      2 weeks ago   STOPSIGNAL SIGQUIT                  0B
# <missing>      2 weeks ago   EXPOSE map[80/tcp:{}]               0B
# <missing>      2 weeks ago   ENV NJS_VERSION=0.8.2               0B
# <missing>      2 weeks ago   ENV NGINX_VERSION=1.25.3            0B
# <missing>      2 weeks ago   RUN /bin/sh -c set -x && ...        10MB
# <missing>      2 weeks ago   ADD file:abc123... /                 32MB

# COLUMNAS:
# IMAGE:       ID de la capa. <missing> = no disponible localmente (imagen descargada)
# CREATED:     Cuándo se creó esta capa
# CREATED BY:  El comando/comentario que generó esta capa
# SIZE:        Tamaño de ESTA capa específica (NO acumulado)
#              0B = instrucción de metadatos (no produce capa de filesystem)
# COMMENT:     Comentario opcional del mantenedor

# Sin truncar los comandos (se ve la instrucción completa)
docker image history --no-trunc nginx:alpine

# Formato personalizado
docker image history --format "table {{.CreatedBy}}\t{{.Size}}" nginx:alpine

# Solo las capas con tamaño > 0 (capas reales, no metadatos)
docker image history --format '{{if ne .Size "0B"}}{{.Size}} {{.CreatedBy}}{{"\n"}}{{end}}' nginx:alpine
```

### 5.6 `dive`: Analizar la eficiencia de las capas

`dive` es una herramienta de código abierto que te muestra EXACTAMENTE qué archivos contiene cada capa, cuánto espacio desperdiciado hay (archivos creados y luego eliminados en capas posteriores), y la eficiencia general de tu imagen.

```bash
# Instalar dive
# macOS:
brew install dive

# Linux (AMD64):
wget https://github.com/wagoodman/dive/releases/download/v0.12.0/dive_0.12.0_linux_amd64.deb
sudo apt install ./dive_0.12.0_linux_amd64.deb

# Usar dive interactivamente
dive nginx:alpine
dive mi-app:v1

# Usar dive en modo CI (no interactivo)
CI=true dive mi-app:v1

# Analizar con reglas de calidad configurables
cat > .dive.yaml << 'EOF'
rules:
  lowestEfficiency: 0.95        # Fallar si eficiencia < 95%
  highestWastedBytes: "10MB"    # Fallar si desperdicio > 10 MB
  highestUserWastedPercent: 0.20  # Fallar si > 20% es espacio de usuario
EOF
dive mi-app:v1 --ci-config .dive.yaml
```

#### Lo que dive te muestra

```
┌──────────────────────────────────────────────────────────────────────┐
│  DIVE: mi-app:v1                                                      │
│                                                                       │
│  ┌───────────────┬──────────────────────────────────────────────────┐ │
│  │ CAPAS          │  CONTENIDO DE LA CAPA SELECCIONADA              │ │
│  │                │                                                  │ │
│  │ ▼ Capa 0      │  Tree:                                           │ │
│  │   32 MB       │  /usr/...                                        │ │
│  │   (base)      │  /bin/...                                        │ │
│  │                │  /etc/...                                       │ │
│  │ ▼ Capa 1      │                                                  │ │
│  │   10 MB       │  [+] /usr/local/bin/nginx       (NEW)            │ │
│  │   (nginx)     │  [+] /etc/nginx/nginx.conf     (NEW)            │ │
│  │  Eff: 95%     │  [+] /var/log/nginx            (NEW)            │ │
│  │                │                                                  │ │
│  │ ▼ Capa 2      │  [+] /app/index.html           (NEW)            │ │
│  │   2 KB        │  [+] /app/style.css            (NEW)            │ │
│  │  Eff: 98%     │                                                  │ │
│  └───────────────┴──────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ ESTADÍSTICAS:                                                     │ │
│  │ Total Image size: 44 MB                                           │ │
│  │ Wasted space:     2.1 MB (4.8%)  ← Archivos creados y luego borrados │
│  │ Image efficiency: 95.2%                                           │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.7 `docker tag`: Crear y gestionar etiquetas

```bash
# Sintaxis básica
docker tag FUENTE[:TAG] DESTINO[:TAG]

# Crear un tag adicional para la misma imagen
docker tag mi-app:v1 mi-app:latest
# Ahora mi-app:v1 y mi-app:latest → mismo IMAGE ID

# Mover imagen entre namespaces
docker tag mi-app:v1 tuusuario/mi-app:v1
docker tag mi-app:v1 123456789012.dkr.ecr.us-east-1.amazonaws.com/mi-app:v1

# Promover entre entornos
docker tag mi-app:staging mi-app:prod
# Esto SOLO cambia el tag LOCAL. Para promover realmente:
docker tag mi-app:staging registry/mi-app:prod
docker push registry/mi-app:prod
```

#### Pipeline de promoción (dev → staging → prod)

```
┌───────────────────────────────────────────────────────────────┐
│                    PIPELINE DE PROMOCIÓN                       │
│                                                               │
│  ┌─────────┐     ┌──────────────┐     ┌────────────────┐     │
│  │  BUILD  │ ──► │   STAGING    │ ──► │     PROD       │     │
│  │         │     │              │     │                │     │
│  │ docker  │     │ docker pull  │     │ docker pull    │     │
│  │ build   │     │ mi-app:      │     │ mi-app:        │     │
│  │ -t sha  │     │   staging    │     │   prod         │     │
│  └────┬────┘     └──────────────┘     └────────────────┘     │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  REGISTRY                                            │    │
│  │                                                      │    │
│  │  mi-app@sha256:abc123  ← Immutable, el "ground truth"│    │
│  │  mi-app:dev     → abc123                             │    │
│  │  mi-app:staging → abc123                             │    │
│  │  mi-app:prod    → abc123                             │    │
│  │  mi-app:v1.2.3  → abc123                             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  La promoción NO cambia la imagen.                            │
│  Cambia SOLO el tag que apunta a ella.                       │
│  dev, staging y prod apuntan a la MISMA imagen (abc123).     │
└───────────────────────────────────────────────────────────────┘
```

**Script de promoción automatizado:**

```bash
#!/bin/bash
# promote-image.sh — Promover una imagen entre entornos
# Uso: ./promote-image.sh mi-app abc123 staging prod

IMAGE=$1
SHA=$2
FROM_ENV=$3
TO_ENV=$4
REGISTRY="registry.mi-empresa.com"

echo "Promoviendo ${IMAGE}:${SHA} de ${FROM_ENV} a ${TO_ENV}"

# Pull de la imagen con el sha inmutable
docker pull ${REGISTRY}/${IMAGE}@sha256:${SHA}

# Etiquetar para el nuevo entorno
docker tag ${REGISTRY}/${IMAGE}@sha256:${SHA} ${REGISTRY}/${IMAGE}:${TO_ENV}

# Push solo del nuevo tag (las capas ya existen en el registry)
docker push ${REGISTRY}/${IMAGE}:${TO_ENV}

echo "Promoción completada: ${IMAGE}:${TO_ENV} ahora apunta a ${SHA}"
```

---

## 6. Transporte y backup de imágenes

### 6.1 `docker save` y `docker load`: El formato canónico

```bash
# Exportar una imagen a un archivo tar
docker save nginx:alpine -o nginx-alpine.tar

# Exportar a stdout
docker save nginx:alpine > nginx-alpine.tar

# Exportar MÚLTIPLES imágenes a un único archivo tar
docker save nginx:alpine node:20-slim mi-app:v1 -o todas.tar

# Exportar con compresión al vuelo
docker save nginx:alpine | gzip > nginx-alpine.tar.gz
docker save nginx:alpine | zstd > nginx-alpine.tar.zst
docker save nginx:alpine | bzip2 > nginx-alpine.tar.bz2

# Ver el contenido del tar (sin extraer)
tar tf nginx-alpine.tar
```

#### Contenido de un archivo `docker save`

```
nginx-alpine.tar
├── manifest.json          ← Índice: qué capas forman qué imagen
├── repositories            ← Mapeo nombre:tag → capas
└── blobs/
    └── sha256/
        ├── a1b2c3d4.../config.json     ← Config blob (metadatos)
        ├── e5f6g7h8.../layer.tar.gz    ← Capa 0 (comprimida)
        ├── i9j0k1l2.../layer.tar.gz    ← Capa 1
        └── m3n4o5p6.../layer.tar.gz    ← Capa 2
```

#### Cargar desde tar

```bash
# Cargar una imagen desde archivo
docker load -i nginx-alpine.tar

# Cargar desde stdin
docker load < nginx-alpine.tar

# Cargar desde archivo comprimido
gunzip -c nginx-alpine.tar.gz | docker load
zstd -d -c nginx-alpine.tar.zst | docker load
```

#### Caso de uso: Ambientes air-gapped (sin conexión a internet)

```bash
# ============================================================
# PREPARACIÓN (en máquina con internet)
# ============================================================

# 1. Crear lista de imágenes necesarias (images.txt)
cat > images.txt << 'EOF'
nginx:1.25.3-alpine
node:20.11-alpine
python:3.12-slim-bookworm
redis:7.2-alpine
postgres:16-alpine
mi-app:v1.2.3
mi-worker:v1.2.3
EOF

# 2. Pull de todas las imágenes
while IFS= read -r image; do
  echo "Descargando $image..."
  docker pull "$image"
done < images.txt

# 3. Exportar todas a un tar comprimido
# Con barra de progreso (requiere pv: apt install pv)
docker save $(cat images.txt | tr '\n' ' ') | pv | gzip > imagenes-airgap.tar.gz

# 4. Transferir al ambiente aislado
scp imagenes-airgap.tar.gz usuario@servidor-airgap:/tmp/
# O usar USB, SFTP interno, etc.

# ============================================================
# CARGA (en ambiente air-gapped)
# ============================================================

# 5. Cargar todas las imágenes
pv /tmp/imagenes-airgap.tar.gz | gunzip | docker load

# 6. Verificar
docker images
```

### 6.2 `docker export` / `docker import`: El caso de los CONTENEDORES

```bash
# IMPORTANTE: export/import trabajan con CONTENEDORES, no imágenes.
# El resultado es una imagen PLANA (una sola capa, sin historial).

# Exportar el filesystem COMPLETO de un contenedor
docker export mi-contenedor -o mi-contenedor.tar

# Importar como una nueva imagen
docker import mi-contenedor.tar mi-imagen:plana

# Importar desde URL
docker import http://ejemplo.com/filesystem.tar.gz mi-imagen:remota

# Añadir metadatos al importar (CMD, ENV, etc.)
docker import --change 'CMD ["python", "app.py"]' \
              --change 'EXPOSE 8000' \
              --change 'ENV APP_ENV=prod' \
              mi-contenedor.tar mi-app:importada
```

#### ¿Qué contiene un export vs un save?

```
┌───────────────────────────────────────┐  ┌────────────────────────────┐
│  docker EXPORT                        │  │  docker SAVE               │
│                                       │  │                            │
│  ┌───────────────────────────────────┐│  │ ┌────────────────────────┐ │
│  │  / (filesystem COMPLETO del       ││  │ │ manifest.json          │ │
│  │     contenedor: imagen + cambios) ││  │ │ config_blob.json       │ │
│  │                                   ││  │ │ capa_0.tar.gz (diff)   │ │
│  │  /bin/bash                        ││  │ │ capa_1.tar.gz (diff)   │ │
│  │  /usr/local/bin/mi-app            ││  │ │ capa_2.tar.gz (diff)   │ │
│  │  /var/log/app.log  ← incluye logs ││  │ │ capa_3.tar.gz (diff)   │ │
│  │  /tmp/datos_modificados.txt       ││  │ └────────────────────────┘ │
│  │  /root/.bash_history              ││  │                            │
│  │                                   ││  │  INCLUYE:                  │
│  │  NO incluye:                      ││  │  - Capas individuales      │
│  │  - Historial de capas             ││  │  - Metadatos completos     │
│  │  - Metadatos (ENV, CMD...)       ││  │  - Historial               │
│  │  - Tags                           ││  │  - Tags originales        │
│  │  - Config.json                    ││  │                            │
│  │                                   ││  │  Tamaño ≈ imagen original  │
│  │  Tamaño = filesystem completo    ││  └────────────────────────────┘
│  │           (más grande que el      ││
│  │            tamaño de la imagen)   ││
│  └───────────────────────────────────┘│
└───────────────────────────────────────┘
```

### 6.3 Tabla comparativa: save/load vs export/import

| Característica | `docker save` / `docker load` | `docker export` / `docker import` |
|---------------|------------------------------|-----------------------------------|
| **Fuente** | Imagen (`docker save imagen`) | Contenedor (`docker export contenedor`) |
| **Destino** | Imagen idéntica (`docker load`) | Imagen NUEVA sin historial (`docker import`) |
| **Capas** | Conserva todas las capas individuales | Imagen plana: TODO en una sola capa |
| **Metadatos** | Conserva ENV, CMD, ENTRYPOINT, EXPOSE, etc. | PIERDE TODO (requiere `--change` para redefinir) |
| **Historial** | Conserva el historial (`docker image history`) | Sin historial (comando de creación = "Imported from -") |
| **Tags** | Conserva los tags originales | No conserva tags (se asigna uno nuevo al importar) |
| **IMAGE ID** | Mismo ID que la imagen original | NUEVO ID (imagen diferente) |
| **Compartición** | Las capas se comparten con otras imágenes | NO comparte capas (es plana) |
| **Tamaño** | Similar al original (capas comprimidas) | Potencialmente mayor (filesystem completo) |
| **Uso típico** | Backup, transferencia, air-gap, migración | Snapshot de contenedor, aplanar imagen, "máquina virtual guardada" |
| **Equivalente a** | Clone exacto de la imagen | `docker commit` + `docker save` (aproximadamente) |

#### Ejemplo comparativo práctico

```bash
# ============================================================
# SAVE/LOAD: Imagen idéntica bit a bit
# ============================================================
docker pull nginx:alpine
ORIGINAL_ID=$(docker images -q nginx:alpine)
ORIGINAL_DIGEST=$(docker image inspect nginx:alpine --format='{{range .RepoDigests}}{{.}}{{"\n"}}{{end}}')

docker save nginx:alpine -o nginx.tar
docker rmi nginx:alpine
docker load -i nginx.tar

NEW_ID=$(docker images -q nginx:alpine)
NEW_DIGEST=$(docker image inspect nginx:alpine --format='{{range .RepoDigests}}{{.}}{{"\n"}}{{end}}')

echo "Original ID: $ORIGINAL_ID"
echo "Nuevo ID:    $NEW_ID"
echo "¿Son iguales? $([ "$ORIGINAL_ID" = "$NEW_ID" ] && echo SÍ || echo NO)"

# ============================================================
# EXPORT/IMPORT: Imagen PLANA, todo diferente
# ============================================================
docker create --name temp-export nginx:alpine
docker export temp-export -o nginx-flat.tar
docker import nginx-flat.tar nginx:flat
docker rm temp-export

# Comparar capas
echo "Capas de nginx:alpine:"
docker image inspect nginx:alpine --format='{{range .RootFS.Layers}}{{println .}}{{end}}'
# Output: 5 capas (sha256:...)

echo "Capas de nginx:flat:"
docker image inspect nginx:flat --format='{{range .RootFS.Layers}}{{println .}}{{end}}'
# Output: 1 capa (sha256:...)

# Metadatos perdidos
echo "CMD de nginx:alpine:"
docker image inspect nginx:alpine --format='{{.Config.Cmd}}'
# Output: [nginx -g daemon off;]

echo "CMD de nginx:flat:"
docker image inspect nginx:flat --format='{{.Config.Cmd}}'
# Output: []  ← Vacío, no ejecutará nada sin --change
```

### 6.4 Compresión eficiente

```bash
# gzip: balance compresión/velocidad (el más común)
docker save mi-app:v1 | gzip > mi-app.tar.gz

# bzip2: mejor compresión, más lento
docker save mi-app:v1 | bzip2 > mi-app.tar.bz2

# xz: máxima compresión, muy lento
docker save mi-app:v1 | xz > mi-app.tar.xz

# zstd: compresión moderna (rápida y eficiente)
docker save mi-app:v1 | zstd -o mi-app.tar.zst

# Comparación de tamaños para una imagen de 450 MB:
# raw:      450 MB
# gzip:     180 MB  (60% compresión, rápido)
# bzip2:    165 MB  (63% compresión, medio)
# xz:       140 MB  (69% compresión, lento)
# zstd -3:  160 MB  (64% compresión, muy rápido)
# zstd -19: 135 MB  (70% compresión, lento)

# Con barra de progreso (pv)
docker save mi-app:v1 | pv -s 450M | gzip > mi-app.tar.gz
```

---

## 7. Firmado y verificación de imágenes

### 7.1 Docker Content Trust (DCT) con Notary

Docker Content Trust usa el framework Notary (basado en TUF — The Update Framework) para firmar criptográficamente imágenes y verificar su integridad y procedencia.

#### Arquitectura de DCT

```
┌───────────────────────────────────────────────────────────────────┐
│                     DOCKER CONTENT TRUST                           │
│                                                                   │
│  ┌─────────────────────┐              ┌─────────────────────┐    │
│  │  PUBLICADOR          │              │  CONSUMIDOR          │    │
│  │                      │              │                      │    │
│  │  docker push         │              │  docker pull          │    │
│  │  (DOCKER_CONTENT_    │              │  (DOCKER_CONTENT_     │    │
│  │   TRUST=1)           │              │   TRUST=1)            │    │
│  │       │              │              │       │               │    │
│  │       ▼              │              │       ▼               │    │
│  │  ┌───────────────┐   │              │  ┌───────────────┐    │    │
│  │  │ Firmar con    │   │              │  │ Verificar     │    │    │
│  │  │ clave privada │   │              │  │ firma contra  │    │    │
│  │  │ (repository   │   │              │  │ Notary server │    │    │
│  │  │  key)         │   │              │  │               │    │    │
│  │  └───────┬───────┘   │              │  └───────┬───────┘    │    │
│  │          │           │              │          │            │    │
│  └──────────┼───────────┘              └──────────┼───────────┘    │
│             │                                      │                │
│             └────────────────┬─────────────────────┘                │
│                              ▼                                      │
│               ┌──────────────────────────────┐                     │
│               │       NOTARY SERVER          │                     │
│               │                              │                     │
│               │  Almacena metadatos firmados │                     │
│               │  usando jerarquía de claves  │                     │
│               │  TUF (The Update Framework): │                     │
│               │                              │                     │
│               │  ┌────────────────────────┐  │                     │
│               │  │  Root Key (offline)    │  │                     │
│               │  │  ├─ Snapshot Key        │  │                     │
│               │  │  ├─ Targets Key         │  │                     │
│               │  │  │  └─ Repository Key   │  │                     │
│               │  │  └─ Timestamp Key       │  │                     │
│               │  └────────────────────────┘  │                     │
│               └──────────────────────────────┘                     │
└───────────────────────────────────────────────────────────────────┘
```

#### Jerarquía de claves en Notary

```
┌───────────────────────────────────────────────────────────┐
│              JERARQUÍA DE CLAVES TUF (NOTARY)             │
│                                                           │
│  ┌──────────────────┐                                     │
│  │    ROOT KEY      │  ← Clave MAESTRA offline            │
│  │    (offline)     │     Se usa SOLO para firmar las     │
│  │                  │     claves de nivel inferior.        │
│  │  ~/.docker/      │     DEBE guardarse en HSM/vault.    │
│  │  trust/private/  │     Si se compromete: desastre.     │
│  └────────┬─────────┘                                     │
│           │                                               │
│     ┌─────┴────────┐                                      │
│     ▼              ▼                                      │
│  ┌──────────┐  ┌──────────┐                               │
│  │ TARGETS  │  │ SNAPSHOT │  ← Firmadas por la root key   │
│  │  KEY     │  │   KEY    │    Online en el Notary server │
│  └────┬─────┘  └──────────┘                               │
│       │                                                   │
│       ▼                                                   │
│  ┌───────────────┐                                        │
│  │ REPOSITORY    │  ← Una por cada repositorio            │
│  │    KEY        │    Firma los tags de ESE repo.         │
│  └───────┬───────┘    Es la que usas al hacer push.       │
│          │                                                │
│          ▼                                                │
│  ┌───────────────┐                                        │
│  │  TIMESTAMP    │  ← Previene ataques de replay          │
│  │     KEY       │    Se renueva frecuentemente.          │
│  └───────────────┘                                        │
└───────────────────────────────────────────────────────────┘
```

#### Usar DCT

```bash
# Habilitar DCT globalmente (recomendado en CI/CD estrictos)
export DOCKER_CONTENT_TRUST=1

# Ahora docker pull VERIFICA las firmas automáticamente
docker pull nginx:alpine

# Si la imagen NO está firmada:
# Error: remote trust data does not exist for docker.io/library/nginx:alpine

# Si la imagen ESTÁ firmada y la verificación es exitosa:
# El pull procede normalmente, mostrando el digest verificado

# Hacer push CON firma (la primera vez genera claves)
docker push tuusuario/mi-app:v1
# Docker te pedirá crear contraseñas para las claves root y repository

# Deshabilitar DCT para una operación específica
docker pull --disable-content-trust nginx:alpine

# Gestionar claves de firma
docker trust key generate nombre-clave
docker trust key ls
docker trust signer add --key cert.pem tuusuario tuusuario/mi-app
docker trust inspect --pretty tuusuario/mi-app:v1
docker trust revoke tuusuario/mi-app:v1
```

### 7.2 Cosign (Sigstore): Firmado sin servidor central

Cosign firma imágenes usando OCI artifacts: las firmas se almacenan EN EL MISMO REGISTRY, junto a la imagen, sin necesidad de servidores externos como Notary.

#### ¿Cosign vs DCT?

| Característica | DCT (Notary) | Cosign (Sigstore) |
|---------------|--------------|-------------------|
| **Infraestructura** | Requiere servidor Notary | Sin servidor (las firmas son OCI artifacts) |
| **Dónde se guardan las firmas** | En el servidor Notary | En el registry, junto a la imagen |
| **Identidad** | Claves auto-gestionadas | OIDC (Google, GitHub, Microsoft) para keyless |
| **Transparencia** | No | Rekor transparency log (público, inmutable) |
| **Revocación** | Sí (TUF) | En desarrollo |
| **Integración Kubernetes** | Limitada | Admission controller (sigstore policy-controller) |
| **SLSA compliance** | No | Sí (para attestations) |
| **SBOM** | No | Sí (cosign attest) |

#### Instalar Cosign

```bash
# macOS
brew install cosign

# Linux
curl -sSL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 \
  -o /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign
cosign version
```

#### Firmar con par de claves (modo básico)

```bash
# 1. Generar par de claves
cosign generate-key-pair
# Enter password: *************
# Private key written to cosign.key   ← ¡GUARDAR SEGURO!
# Public key written to cosign.pub    ← Distribuir a consumidores

# 2. Firmar una imagen
cosign sign --key cosign.key tuusuario/mi-app:v1
# Enter password for private key:
# Pushing signature to: index.docker.io/tuusuario/mi-app

# 3. Verificar una imagen
cosign verify --key cosign.pub tuusuario/mi-app:v1

# Output:
# Verification for index.docker.io/tuusuario/mi-app:v1 --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key
# [{"critical":{"identity":{"docker-reference":"index.docker.io/tuusuario/mi-app"},
#   "image":{"docker-manifest-digest":"sha256:abc123..."},
#   "type":"cosign container image signature"},
#   "optional":null}]
```

#### Keyless signing (sin gestionar claves, vía OIDC)

```bash
# Firmar usando OIDC (se abre navegador para autenticación)
cosign sign tuusuario/mi-app:v1
# Se autentica con Google/GitHub/Microsoft
# La firma se registra en Rekor transparency log automáticamente

# Verificar firma keyless (especificando identidad OIDC)
cosign verify \
  --certificate-identity tuusuario@ejemplo.com \
  --certificate-oidc-issuer https://accounts.google.com \
  tuusuario/mi-app:v1

# Para GitHub Actions:
cosign verify \
  --certificate-identity https://github.com/mi-org/mi-repo/.github/workflows/build.yml@refs/heads/main \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/mi-org/mi-repo:v1
```

#### Cosign en CI/CD (GitHub Actions)

```yaml
# .github/workflows/build-sign.yml
name: Build, Sign and Verify

on:
  push:
    tags: ['v*.*.*']

permissions:
  id-token: write       # Necesario para OIDC keyless signing
  packages: write
  contents: read

jobs:
  build-and-sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign image with keyless signing
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}

      - name: Verify the signature
        run: |
          cosign verify \
            --certificate-identity https://github.com/${{ github.repository }}/.github/workflows/build-sign.yml@${{ github.ref }} \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
```

#### Firmar attestations (SBOM, escaneos de vulnerabilidades)

```bash
# Firmar un SBOM junto con la imagen
cosign attest --key cosign.key \
  --predicate sbom.spdx.json \
  --type spdx \
  tuusuario/mi-app:v1

# Firmar resultado de escaneo de vulnerabilidades
trivy image --format cosign-vuln --output vuln.json tuusuario/mi-app:v1
cosign attest --key cosign.key \
  --predicate vuln.json \
  --type vuln \
  tuusuario/mi-app:v1

# Verificar attestation
cosign verify-attestation --key cosign.pub \
  --type spdx \
  tuusuario/mi-app:v1

cosign verify-attestation --key cosign.pub \
  --type vuln \
  tuusuario/mi-app:v1
```

---

## 8. Laboratorio: Estrategia de versionado de imágenes para un proyecto real

### 8.1 Contexto del proyecto

La empresa **CloudAPI S.A.** desarrolla una plataforma de APIs con estos componentes:

| Componente | Tecnología | Imagen base | Tamaño aprox. |
|-----------|-----------|-------------|---------------|
| **api-backend** | FastAPI (Python 3.12) | `python:3.12-slim-bookworm` | ~250 MB |
| **api-frontend** | React (Node.js 20) | `node:20-alpine` | ~180 MB |
| **worker** | Celery + Redis (Python 3.12) | `python:3.12-slim-bookworm` | ~280 MB |
| **db-migrations** | Alembic (Python 3.12) | `python:3.12-slim-bookworm` | ~200 MB |

Cada componente se despliega en **3 entornos**: dev, staging, prod.

### 8.2 Requisitos de la estrategia

1. **Trazabilidad completa:** Saber qué commit de código produjo qué imagen en todo momento.
2. **Reproducibilidad:** Poder reconstruir exactamente la misma imagen en cualquier momento futuro.
3. **Promoción segura:** La MISMA imagen que pasó las pruebas en staging DEBE ser la que va a producción.
4. **Seguridad:** Escanear cada imagen antes de que llegue a producción.
5. **Limpieza automática:** No acumular imágenes obsoletas.
6. **Rollback rápido:** Volver a la versión anterior en menos de 2 minutos.
7. **Multi-arch:** Soportar ARM64 (desarrolladores con Mac M1/M2) y AMD64 (servidores en la nube).

### 8.3 Estrategia de tags

```
┌───────────────────────────────────────────────────────────────────────┐
│                    ESTRATEGIA DE TAGS                                 │
│                                                                       │
│  ┌────────────────────────────┐  ┌──────────────────────────────┐    │
│  │ TAGS INMUTABLES            │  │ TAGS MUTABLES (se mueven)    │    │
│  │ (referencia canónica)     │  │                              │    │
│  ├────────────────────────────┤  ├──────────────────────────────┤    │
│  │ <componente>:<sha256>      │  │ <componente>:dev             │    │
│  │  Ej: api-backend:abc123d   │  │  → Último build exitoso en   │    │
│  │                            │  │    rama dev                  │    │
│  │ <componente>:<git-sha>     │  │                              │    │
│  │  Ej: api-backend:a1b2c3d   │  │ <componente>:staging         │    │
│  │                            │  │  → Último build promovido    │    │
│  │ <componente>:<semver>      │  │    a staging                 │    │
│  │  Ej: api-backend:v1.2.3    │  │                              │    │
│  │                            │  │ <componente>:prod            │    │
│  │                            │  │  → Último build promovido    │    │
│  │                            │  │    a producción              │    │
│  │                            │  │                              │    │
│  │                            │  │ <componente>:latest          │    │
│  │                            │  │  → Último build de main      │    │
│  │                            │  │    (SOLO para desarrollo)    │    │
│  └────────────────────────────┘  └──────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────┘
```

### 8.4 Dockerfile optimizado para api-backend

```dockerfile
# ============================================================
# api-backend/Dockerfile
# CloudAPI S.A. — Multi-stage build optimizado
# ============================================================

# ---- Stage 1: Builder (dependencias) ----
FROM python:3.12-slim-bookworm AS builder

ARG POETRY_VERSION=1.7.1
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    POETRY_VIRTUALENVS_CREATE=false

WORKDIR /app

# Instalar poetry para gestión de dependencias
RUN pip install --no-cache-dir poetry==${POETRY_VERSION}

# Copiar solo los archivos de dependencias (cache optimization)
COPY pyproject.toml poetry.lock ./

# Instalar dependencias de producción SOLAMENTE
RUN poetry install --only main --no-interaction --no-ansi --no-root

# ---- Stage 2: Runtime (imagen final) ----
FROM python:3.12-slim-bookworm AS runtime

# Crear usuario no-root
RUN groupadd --system app && \
    useradd --system --gid app --no-create-home --home-dir /app app

WORKDIR /app

# Copiar dependencias desde el builder
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# Copiar código fuente (última capa: cambia más frecuentemente)
COPY --chown=app:app api_backend/ ./api_backend/
COPY --chown=app:app alembic/ ./alembic/
COPY --chown=app:app alembic.ini pyproject.toml ./

# Variables de entorno
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Seguridad: usuario no-root
USER app

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

EXPOSE 8000

CMD ["uvicorn", "api_backend.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 8.5 Script de CI/CD con versionado completo

```bash
#!/bin/bash
# ============================================================
# build-and-push.sh
# CloudAPI S.A. — Build, tag, scan, and push Docker images
# ============================================================
set -euo pipefail

# === Configuración ===
COMPONENT="${1:?Uso: $0 <componente> <entorno>}"
ENVIRONMENT="${2:?Uso: $0 <componente> <entorno>}"
REGISTRY="${DOCKER_REGISTRY:-123456789012.dkr.ecr.us-east-1.amazonaws.com}"
GIT_SHA=$(git rev-parse --short=8 HEAD)
GIT_SHA_FULL=$(git rev-parse HEAD)
BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
SEMVER=$(git describe --tags --always --dirty 2>/dev/null || echo "0.0.0-${GIT_SHA}")

# === Tags ===
IMAGE_BASE="${REGISTRY}/${COMPONENT}"
IMMUTABLE_TAG="${GIT_SHA_FULL}"
SHA_TAG="${GIT_SHA}"
SEMVER_TAG="${SEMVER}"
ENV_TAG="${ENVIRONMENT}"
BUILD_META_TAG="${BUILD_DATE}-${GIT_SHA}"

echo "=== Build ${COMPONENT} for ${ENVIRONMENT} ==="
echo "Git SHA: ${GIT_SHA}"
echo "SemVer:  ${SEMVER}"
echo "Registry: ${REGISTRY}"

# === Step 1: Build ===
echo ""
echo "--- Step 1: Build docker image ---"
docker build \
  --platform linux/amd64,linux/arm64 \
  --build-arg BUILD_DATE="${BUILD_DATE}" \
  --build-arg GIT_SHA="${GIT_SHA_FULL}" \
  --build-arg VERSION="${SEMVER}" \
  --label "org.opencontainers.image.created=${BUILD_DATE}" \
  --label "org.opencontainers.image.revision=${GIT_SHA_FULL}" \
  --label "org.opencontainers.image.version=${SEMVER}" \
  --label "org.opencontainers.image.title=${COMPONENT}" \
  -t "${IMAGE_BASE}:${IMMUTABLE_TAG}" \
  -t "${IMAGE_BASE}:${SHA_TAG}" \
  -t "${IMAGE_BASE}:${SEMVER_TAG}" \
  -t "${IMAGE_BASE}:${ENV_TAG}" \
  -t "${IMAGE_BASE}:${BUILD_META_TAG}" \
  -f "${COMPONENT}/Dockerfile" \
  "${COMPONENT}/"

# === Step 2: Scan (Trivy) ===
echo ""
echo "--- Step 2: Scan for vulnerabilities ---"
trivy image --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --no-progress \
  "${IMAGE_BASE}:${IMMUTABLE_TAG}"

# Guardar resultados del scan como artifact
trivy image --format json \
  --output "trivy-${COMPONENT}-${GIT_SHA}.json" \
  "${IMAGE_BASE}:${IMMUTABLE_TAG}"

# === Step 3: Push ===
echo ""
echo "--- Step 3: Push to registry ---"
docker push "${IMAGE_BASE}:${IMMUTABLE_TAG}"
docker push "${IMAGE_BASE}:${SHA_TAG}"
docker push "${IMAGE_BASE}:${SEMVER_TAG}"

# Para dev: push inmediato del tag mutable
if [ "${ENVIRONMENT}" = "dev" ]; then
  docker push "${IMAGE_BASE}:dev"
  echo "→ Tag dev actualizado"
fi

# Para staging/prod: requiere aprobación
if [ "${ENVIRONMENT}" = "staging" ] || [ "${ENVIRONMENT}" = "prod" ]; then
  echo "→ Tag ${ENVIRONMENT} requiere aprobación manual para ser actualizado"
  echo "→ Imagen disponible como: ${IMAGE_BASE}:${IMMUTABLE_TAG}"
  echo "→ Para promover: ./promote.sh ${COMPONENT} ${IMMUTABLE_TAG} ${ENVIRONMENT}"
fi

# === Step 4: Firmar con Cosign (keyless) ===
echo ""
echo "--- Step 4: Sign image with Cosign ---"
cosign sign --yes "${IMAGE_BASE}:${IMMUTABLE_TAG}"

# === Step 5: Resumen ===
echo ""
echo "==========================================="
echo "  BUILD COMPLETADO"
echo "==========================================="
echo "Componente: ${COMPONENT}"
echo "Entorno:    ${ENVIRONMENT}"
echo "Imagen:     ${IMAGE_BASE}:${IMMUTABLE_TAG}"
echo "Digest:     $(docker image inspect ${IMAGE_BASE}:${IMMUTABLE_TAG} --format='{{index .RepoDigests 0}}')"
echo "Tags:"
echo "  ${IMAGE_BASE}:${IMMUTABLE_TAG}  (immutable)"
echo "  ${IMAGE_BASE}:${SHA_TAG}"
echo "  ${IMAGE_BASE}:${SEMVER_TAG}"
echo "  ${IMAGE_BASE}:${BUILD_META_TAG}"
echo "==========================================="
```

### 8.6 Script de promoción entre entornos

```bash
#!/bin/bash
# ============================================================
# promote.sh
# CloudAPI S.A. — Promote image from one environment to another
# Uso: ./promote.sh <componente> <sha256|git-sha> <entorno-destino>
# ============================================================
set -euo pipefail

COMPONENT="${1:?Uso: $0 <componente> <sha> <entorno>}"
IMAGE_SHA="${2:?Uso: $0 <componente> <sha> <entorno>}"
TO_ENV="${3:?Uso: $0 <componente> <sha> <entorno>}"
REGISTRY="${DOCKER_REGISTRY:-123456789012.dkr.ecr.us-east-1.amazonaws.com}"
IMAGE="${REGISTRY}/${COMPONENT}"

echo "=== Promoviendo ${COMPONENT}:${IMAGE_SHA} → ${TO_ENV} ==="

# Verificar que la imagen existe
docker pull "${IMAGE}@sha256:${IMAGE_SHA}" || {
  echo "ERROR: Imagen ${IMAGE}@sha256:${IMAGE_SHA} no encontrada"
  exit 1
}

# Si es prod, verificar firma de Cosign
if [ "${TO_ENV}" = "prod" ]; then
  echo "Verificando firma Cosign en imagen de producción..."
  cosign verify \
    --certificate-oidc-issuer https://token.actions.githubusercontent.com \
    "${IMAGE}@sha256:${IMAGE_SHA}" || {
    echo "ERROR: La imagen no está firmada o la firma no es válida"
    exit 1
  }

  # Verificar que no tiene vulnerabilidades críticas
  echo "Escaneando imagen antes de promover a prod..."
  trivy image --severity CRITICAL --ignore-unfixed \
    "${IMAGE}@sha256:${IMAGE_SHA}" || {
    echo "ERROR: Vulnerabilidades CRÍTICAS encontradas. Promoción abortada."
    exit 1
  }
fi

# Etiquetar y empujar
docker tag "${IMAGE}@sha256:${IMAGE_SHA}" "${IMAGE}:${TO_ENV}"
docker push "${IMAGE}:${TO_ENV}"

echo "=== Promoción completada ==="
echo "${COMPONENT}:${TO_ENV} → sha256:${IMAGE_SHA}"
```

### 8.7 Pipeline de CI/CD (GitHub Actions)

```yaml
# .github/workflows/docker-pipeline.yml
name: Docker Build and Deploy Pipeline

on:
  push:
    branches: [main, dev]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

env:
  REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com

jobs:
  # -------------------------------
  # Job 1: Build + Scan + Push (DEV)
  # -------------------------------
  build-dev:
    if: github.ref == 'refs/heads/dev'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    strategy:
      matrix:
        component: [api-backend, api-frontend, worker, db-migrations]
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-ecr
          aws-region: us-east-1

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ./${{ matrix.component }}
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ matrix.component }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ matrix.component }}:dev
          labels: |
            org.opencontainers.image.revision=${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ matrix.component }}:${{ github.sha }}
          format: sarif
          output: trivy-results-${{ matrix.component }}.sarif
          severity: HIGH,CRITICAL

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results-${{ matrix.component }}.sarif

      - name: Sign image with Cosign
        run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ matrix.component }}@${{ github.sha }}

  # -------------------------------
  # Job 2: Build + Scan (STAGING)
  # -------------------------------
  build-staging:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    strategy:
      matrix:
        component: [api-backend, api-frontend, worker, db-migrations]
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-ecr
          aws-region: us-east-1
      - uses: aws-actions/amazon-ecr-login@v2
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v5
        with:
          context: ./${{ matrix.component }}
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ matrix.component }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ matrix.component }}:staging
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ matrix.component }}:${{ github.sha }}
          format: table
          severity: CRITICAL,HIGH
          exit-code: 1
      - run: cosign sign --yes ${{ env.REGISTRY }}/${{ matrix.component }}@${{ github.sha }}

  # -------------------------------
  # Job 3: Promote to PROD (manual)
  # -------------------------------
  promote-to-prod:
    if: startsWith(github.ref, 'refs/tags/v')
    needs: [build-staging]
    runs-on: ubuntu-latest
    environment: production  # Requiere aprobación manual en GitHub
    permissions:
      id-token: write
      contents: read
    strategy:
      matrix:
        component: [api-backend, api-frontend, worker, db-migrations]
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-ecr-prod
          aws-region: us-east-1
      - uses: aws-actions/amazon-ecr-login@v2

      - name: Verify signature before promoting
        run: |
          cosign verify \
            --certificate-identity https://github.com/${{ github.repository }}/.github/workflows/docker-pipeline.yml@${{ github.ref }} \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            ${{ env.REGISTRY }}/${{ matrix.component }}@${{ github.sha }}

      - name: Promote to prod tag
        run: |
          docker pull ${{ env.REGISTRY }}/${{ matrix.component }}@${{ github.sha }}
          docker tag ${{ env.REGISTRY }}/${{ matrix.component }}@${{ github.sha }} \
            ${{ env.REGISTRY }}/${{ matrix.component }}:prod
          docker push ${{ env.REGISTRY }}/${{ matrix.component }}:prod
```

### 8.8 Estrategia de limpieza (Lifecycle Policy en ECR)

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Mantener últimas 50 imágenes de prod",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod"],
        "countType": "imageCountMoreThan",
        "countNumber": 50
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Limpiar imágenes dev y staging tras 14 días",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["dev", "staging"],
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 14
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 3,
      "description": "Eliminar imágenes sin tag (dangling) tras 7 días",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
```

### 8.9 Plan de rollback

```bash
#!/bin/bash
# ============================================================
# rollback.sh
# CloudAPI S.A. — Rollback a versión anterior en < 2 minutos
# Uso: ./rollback.sh <componente> [n-versiones-atras]
# ============================================================
set -euo pipefail

COMPONENT="${1:?Uso: $0 <componente>}"
VERSIONS_BACK="${2:-1}"
REGISTRY="${DOCKER_REGISTRY}"

echo "=== Rollback de ${COMPONENT} ==="

# 1. Listar las últimas imágenes de prod (ordenadas por fecha de push)
echo "Buscando versiones anteriores de prod..."
PREVIOUS_IMAGES=$(aws ecr describe-images \
  --repository-name "${COMPONENT}" \
  --region us-east-1 \
  --query "reverse(sort_by(imageDetails,&imagePushedAt))[?contains(imageTags, 'prod')]" \
  --output json)

# 2. Obtener el digest de la versión N-anterior
TARGET_DIGEST=$(echo "${PREVIOUS_IMAGES}" | jq -r ".[${VERSIONS_BACK}].imageDigest")

if [ "${TARGET_DIGEST}" = "null" ] || [ -z "${TARGET_DIGEST}" ]; then
  echo "ERROR: No hay versión N-${VERSIONS_BACK} disponible para rollback"
  exit 1
fi

echo "Target digest: ${TARGET_DIGEST}"

# 3. Verificar la firma de la imagen de rollback
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  "${REGISTRY}/${COMPONENT}@${TARGET_DIGEST}" || {
  echo "ERROR: La imagen de rollback no tiene firma válida"
  exit 1
}

# 4. Mover el tag prod a la versión anterior
docker pull "${REGISTRY}/${COMPONENT}@${TARGET_DIGEST}"
docker tag "${REGISTRY}/${COMPONENT}@${TARGET_DIGEST}" "${REGISTRY}/${COMPONENT}:prod"
docker push "${REGISTRY}/${COMPONENT}:prod"

# 5. Forzar actualización en Kubernetes
kubectl rollout restart deployment "${COMPONENT}" -n production
kubectl rollout status deployment "${COMPONENT}" -n production --timeout=120s

echo "=== Rollback completado ==="
echo "${COMPONENT}:prod → ${TARGET_DIGEST}"
```

### 8.10 Resumen de la estrategia

```
┌───────────────────────────────────────────────────────────────────────────┐
│                 ESTRATEGIA COMPLETA DE VERSIONADO                          │
│                                                                           │
│  ┌────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐          │
│  │  DEV   │───►│  STAGING  │───►│    PROD   │    │  ROLLBACK │          │
│  │        │    │           │    │           │    │           │          │
│  │ Push a │    │ Merge a   │    │ Git tag   │    │ Re-apuntar│          │
│  │ dev    │    │ main      │    │ vX.Y.Z    │    │ tag prod  │          │
│  │ activa │    │ activa el │    │ activa el │    │ a digest  │          │
│  │ pipeline│   │ pipeline  │    │ pipeline  │    │ anterior  │          │
│  └───┬────┘    └─────┬─────┘    └─────┬─────┘    └─────┬─────┘          │
│      │               │               │                 │                 │
│      ▼               ▼               ▼                 ▼                 │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                         REGISTRY (ECR)                            │   │
│  │                                                                  │   │
│  │  api-backend@sha256:abc123  ← Imagen inmutable (ground truth)    │   │
│  │  api-backend:dev     → sha256:abc123                              │   │
│  │  api-backend:staging → sha256:abc123                              │   │
│  │  api-backend:prod    → sha256:abc123                              │   │
│  │  api-backend:v1.2.3  → sha256:abc123                              │   │
│  │                                                                  │   │
│  │  Lifecycle Policies:                                             │   │
│  │    - prod:   mantener últimas 50 imágenes                         │   │
│  │    - dev/staging: eliminar tras 14 días                           │   │
│  │    - untagged: eliminar tras 7 días                               │   │
│  │                                                                  │   │
│  │  Firmado: Cosign keyless en cada push                             │   │
│  │  Escaneo: Trivy (HIGH+CRITICAL bloquean promoción a prod)        │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Conclusión del Capítulo

Las imágenes Docker no son archivos misteriosos. Son estructuras bien definidas compuestas por un **manifest** (el índice), un **config** (los metadatos), y **capas** (los diffs del filesystem). Dominar estos tres componentes te da el poder de:

- **Optimizar builds** entendiendo el orden de las capas y el sistema de caché.
- **Depurar problemas** inspeccionando manifests, configs y capas directamente en disco.
- **Reducir costos** aprovechando la compartición de capas y multi-stage builds.
- **Garantizar seguridad** firmando imágenes con Cosign o DCT.
- **Diseñar pipelines** de promoción robustos con tags inmutables y versionado semántico.
- **Gestionar el ciclo de vida** de tus imágenes desde el build hasta el garbage collection.

El sistema de capas con OverlayFS y copy-on-write es la innovación clave que hace posible Docker: permite que 100 contenedores compartan un solo conjunto de capas base mientras cada uno tiene su propia capa de escritura efímera. Sin este mecanismo, la contenerización a escala sería económicamente inviable.

En el próximo capítulo, pondremos este conocimiento en práctica construyendo nuestros propios Dockerfiles avanzados y optimizando imágenes para producción.

---

*Capítulo 3 — Imágenes Docker*  
*Libro: Docker de Novato a Experto*

---

← [Capítulo anterior](capitulo-02-instalacion.md) | [Inicio](../README.md) | [Capítulo siguiente →](capitulo-04-dockerfile.md)
