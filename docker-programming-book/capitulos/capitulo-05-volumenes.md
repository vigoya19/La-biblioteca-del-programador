# Capítulo 5: Almacenamiento y Volúmenes

> *"Los contenedores mueren. Los datos deben sobrevivir."*

---

## 5.1 El problema de la persistencia

Los contenedores Docker están diseñados para ser efímeros: nacen, cumplen su propósito y mueren. Esta filosofía es parte del ADN de los contenedores y lo que los hace tan potentes para escalar, reemplazar y destruir sin miedo. Pero hay un precio: **cuando un contenedor muere, todo lo que escribiste dentro de él desaparece para siempre**.

Imagina que despliegas una base de datos MySQL en un contenedor. Tus usuarios crean cuentas, suben fotos, generan pedidos. Millones de registros. Luego haces `docker rm -f mysql` porque necesitas actualizar la versión. Si no configuraste almacenamiento persistente, acabas de perder toda tu base de datos. Para siempre. Sin recuperación posible.

Esto no es un bug. Es una decisión de diseño. El sistema de archivos de un contenedor es una **capa writable efímera** montada sobre las capas de solo lectura de la imagen. Cuando el contenedor se elimina, esa capa writable se destruye. Docker lo advierte en su documentación:

> *"By default all files created inside a container are stored on a writable container layer. This means that the data doesn't persist when that container no longer exists."*

### 5.1.1 Demostración: la tragedia de los datos perdidos

Hagamos una prueba contundente. Creamos un contenedor, escribimos un archivo, destruimos el contenedor, lo recreamos desde la misma imagen y comprobamos que el archivo ya no existe:

```bash
# 1. Creamos un contenedor Alpine y escribimos datos importantes
$ docker run --name mi_contenedor alpine \
    sh -c 'echo "datos-criticos.txt: usuario=admin, password=secret123" > /datos/importantes.txt && \
           echo "Datos guardados. Contenido:" && \
           cat /datos/importantes.txt'

# Salida:
# Datos guardados. Contenido:
# datos-criticos.txt: usuario=admin, password=secret123

# 2. Verificamos que el contenedor terminó pero existe (estado exited)
$ docker ps -a --filter name=mi_contenedor
# CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS
# a1b2c3d4e5f6   alpine    "sh -c 'echo ...'"      10 seconds ago   Exited (0) 8 seconds ago

# 3. Eliminamos el contenedor
$ docker rm mi_contenedor
# mi_contenedor

# 4. Lanzamos un contenedor NUEVO desde la misma imagen
$ docker run --name mi_contenedor alpine \
    sh -c 'echo "Buscando datos..." && \
           cat /datos/importantes.txt 2>&1 || echo "ARCHIVO NO ENCONTRADO"'

# Salida:
# Buscando datos...
# cat: can't open '/datos/importantes.txt': No such file or directory
# ARCHIVO NO ENCONTRADO
```

El directorio `/datos/` ni siquiera existe en el nuevo contenedor. La capa writable del primer contenedor fue completamente destruida con `docker rm`.

### 5.1.2 El modelo de capas y la capa writable

Para entender por qué los datos desaparecen, necesitamos recordar cómo funciona el sistema de archivos de un contenedor:

```
┌─────────────────────────────────────────┐
│         CAPA WRITABLE (R/W)              │  ← Esta capa MUERE con el contenedor
│  Todos los cambios: nuevos archivos,     │
│  modificaciones, eliminaciones           │
├─────────────────────────────────────────┤
│  Capa 5: CMD (R/O)                       │  ← Capas de imagen
├─────────────────────────────────────────┤
│  Capa 4: COPY app (R/O)                  │     (inmutables, compartidas)
├─────────────────────────────────────────┤
│  Capa 3: RUN apt install (R/O)           │
├─────────────────────────────────────────┤
│  Capa 2: RUN apt update (R/O)            │
├─────────────────────────────────────────┤
│  Capa 1: FROM ubuntu:22.04 (R/O)         │
└─────────────────────────────────────────┘
```

La capa writable usa un **storage driver** (OverlayFS en la mayoría de sistemas modernos) que implementa **copy-on-write**: cuando modificas un archivo que existe en una capa inferior, se copia a la capa writable y se modifica allí. Pero esta capa writable pertenece exclusivamente a ESE contenedor. Si eliminas el contenedor, la capa se destruye.

El comando `docker commit` podría salvar la capa writable creando una nueva imagen, pero esto es un antipatrón para persistencia de datos (las imágenes no deben contener datos de runtime).

### 5.1.3 Los tres mecanismos de persistencia

Docker ofrece tres mecanismos para que los datos sobrevivan al contenedor:

| Mecanismo    | Dónde se almacena                | Quién lo gestiona | Persiste tras `docker rm`           |
|--------------|----------------------------------|-------------------|-------------------------------------|
| **Bind mount** | Cualquier ruta del host        | El usuario        | Sí (son archivos del host)          |
| **Volume**     | `/var/lib/docker/volumes/`    | Docker            | Sí                                  |
| **tmpfs mount**| RAM del host                   | El kernel         | No (es RAM)                         |

```
┌──────────────────────────────────────────────────────────────┐
│                       HOST (Linux)                           │
│                                                              │
│  /home/user/project ──bind mount──▶ /app  (en contenedor)    │
│                                                              │
│  /var/lib/docker/volumes/mi_vol/_data ──volume──▶ /data     │
│                                                              │
│  RAM (tmpfs) ──tmpfs mount──▶ /tmp/cache  (en contenedor)   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

A lo largo de este capítulo exploraremos los tres en profundidad, con ejemplos reales, casos de uso, problemas comunes y soluciones.

---

## 5.2 Bind Mounts — Montajes de directorios del host

### 5.2.1 Qué son los bind mounts

Un **bind mount** es exactamente lo que su nombre indica: "ata" (bind) un directorio o archivo del sistema de archivos del host a una ruta dentro del contenedor. Es el mecanismo más antiguo de Docker y el más directo. El contenedor ve y modifica los archivos del host como si fueran propios.

Piensa en un bind mount como un acceso directo (symlink potenciado) entre dos puntos del árbol de directorios. No hay copia, no hay volumen intermedio. Lo que escribes en el contenedor aparece instantáneamente en el host y viceversa.

```
┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│         HOST                    │     │         CONTENEDOR              │
│                                 │     │                                 │
│  /home/user/proyecto/           │     │  /app/                          │
│  ├── src/                       │     │  ├── src/                       │
│  │   ├── main.py                │◄───▶│  │   ├── main.py                │
│  │   └── utils.py               │     │  │   └── utils.py               │
│  ├── config/                    │     │  ├── config/                    │
│  │   └── settings.yaml          │     │  │   └── settings.yaml          │
│  └── README.md                  │     │  └── README.md                  │
│                                 │     │                                 │
│  (los mismos archivos,          │     │  (mismo inodo, misma            │
│   misma realidad física)        │     │   realidad física)              │
└─────────────────────────────────┘     └─────────────────────────────────┘
```

### 5.2.2 Sintaxis de bind mounts

Docker ofrece dos sintaxis para montar bind mounts. La sintaxis corta con `-v` y la sintaxis larga con `--mount`. Aunque `-v` es más concisa, **`--mount` es la recomendada por Docker** por ser más explícita, menos ambigua y más fácil de leer en scripts.

**Sintaxis corta (`-v`):**

```bash
docker run -v /ruta/en/host:/ruta/en/contenedor:opciones imagen

# Ejemplo:
docker run -v /home/user/proyecto:/app:ro nginx
```

La sintaxis de `-v` tiene tres partes separadas por `:`:
1. Ruta en el host (debe ser absoluta, excepto para volúmenes nombrados)
2. Ruta en el contenedor (debe ser absoluta)
3. Opciones (separadas por coma): `ro` (read-only), `rw` (read-write), `z`/`Z` (SELinux)

Si la ruta del host no existe, Docker la crea automáticamente como directorio (con `-v`). Esto puede ser peligroso: un typo crea un directorio vacío en lugar de montar el que querías.

**Sintaxis larga (`--mount`):**

```bash
docker run --mount type=bind,source=/ruta/en/host,target=/ruta/en/contenedor,readonly imagen

# Ejemplo:
docker run --mount type=bind,source=/home/user/proyecto,target=/app,readonly nginx
```

Con `--mount`, las opciones se especifican como pares `clave=valor` separados por coma. Es mucho más legible:

```bash
docker run \
  --mount type=bind,source=/home/user/config,target=/etc/app/config,readonly \
  --mount type=bind,source=/home/user/data,target=/var/lib/app/data \
  mi-app
```

La sintaxis `--mount` **no crea directorios automáticamente**. Si la ruta `source` no existe, Docker arroja un error explícito. Esto es bueno: prefieres un error claro a un directorio vacío misterioso.

Parámetros de `--mount type=bind`:

| Parámetro                  | Descripción                 | Obligatorio |
|----------------------------|-----------------------------|-------------|
| `type=bind`                | Tipo de montaje             | Sí          |
| `source` o `src`           | Ruta en el host             | Sí          |
| `target` o `dst` o `destination` | Ruta en el contenedor | Sí          |
| `readonly` o `ro`          | Montaje de solo lectura     | No          |
| `bind-propagation`         | Modo de propagación         | No          |
| `bind-nonrecursive`        | No montar submounts         | No          |

### 5.2.3 Casos de uso reales de bind mounts

#### 5.2.3.1 Desarrollo con live reload

El caso de uso más común para bind mounts es el desarrollo local. Montas tu código fuente en el contenedor para que los cambios que haces en tu editor se reflejen instantáneamente dentro del contenedor:

```bash
# Node.js con nodemon para live reload
docker run -d \
  --name dev-api \
  -p 3000:3000 \
  -v "$(pwd)/src:/app/src" \
  -v "$(pwd)/package.json:/app/package.json" \
  -e NODE_ENV=development \
  node:20-alpine \
  sh -c "npm install && npx nodemon src/index.js"
```

```bash
# Python con Flask debug mode
docker run -d \
  --name dev-flask \
  -p 5000:5000 \
  -v "$(pwd):/app" \
  -e FLASK_ENV=development \
  -e FLASK_DEBUG=1 \
  python:3.12-slim \
  sh -c "pip install -r /app/requirements.txt && flask run --host=0.0.0.0"
```

```bash
# Go con CompileDaemon
docker run -d \
  --name dev-go \
  -p 8080:8080 \
  -v "$(pwd):/app" \
  -w /app \
  golang:1.22 \
  sh -c "go install github.com/githubnemo/CompileDaemon@latest && \
         CompileDaemon -command='go run ./cmd/server'"
```

```bash
# PHP con Apache en desarrollo
docker run -d \
  --name dev-php \
  -p 80:80 \
  -v "$(pwd)/src:/var/www/html" \
  -v "$(pwd)/apache.conf:/etc/apache2/sites-available/000-default.conf" \
  php:8.3-apache
```

#### 5.2.3.2 Acceso al socket de Docker (Docker-in-Docker ligero)

Un patrón muy común es montar el socket de Docker dentro de un contenedor para que ese contenedor pueda gestionar otros contenedores. Jenkins, GitLab Runner, Traefik y Portainer usan este patrón:

```bash
# Jenkins con acceso al socket de Docker
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

```bash
# Portainer con acceso al socket de Docker
docker run -d \
  --name portainer \
  -p 9443:9443 \
  -p 8000:8000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

**Advertencia de seguridad:** Montar el socket de Docker le da al contenedor acceso root al daemon de Docker del host. El contenedor puede crear, modificar o eliminar cualquier contenedor, imagen, volumen o red. Es equivalente a dar acceso root al host. Solo debes hacerlo en entornos controlados y confiables. Alternativas más seguras incluyen Docker-in-Docker con `--privileged` (más aislado pero más pesado) o Kaniko para builds sin socket.

#### 5.2.3.3 Compartir configuraciones centralizadas

Cuando tienes múltiples contenedores que necesitan la misma configuración (ej: certificados TLS, archivos de configuración corporativos), los bind mounts son perfectos:

```bash
# Centralizar certificados TLS en el host
sudo mkdir -p /etc/docker/certs

# Un script de despliegue copia los certs una sola vez
sudo cp mi-dominio.crt mi-dominio.key /etc/docker/certs/

# Todos los contenedores montan la misma configuración
docker run -d --name nginx-proxy \
  -v /etc/docker/certs/mi-dominio.crt:/etc/nginx/certs/server.crt:ro \
  -v /etc/docker/certs/mi-dominio.key:/etc/nginx/certs/server.key:ro \
  nginx

docker run -d --name haproxy-lb \
  -v /etc/docker/certs/mi-dominio.crt:/etc/haproxy/certs/server.pem:ro \
  haproxy
```

#### 5.2.3.4 Montar dispositivos Unix y sockets

Los sockets Unix son archivos especiales en el sistema de archivos. Con bind mounts puedes compartirlos:

```bash
# Montar el socket de MySQL del host en un contenedor
docker run -it --rm \
  -v /var/run/mysqld/mysqld.sock:/var/run/mysqld/mysqld.sock \
  mysql:8.0 \
  mysql -u root -S /var/run/mysqld/mysqld.sock
```

```bash
# Compartir un socket de Redis
docker run -d --name redis-socket \
  -v /tmp/redis:/var/run/redis \
  redis:7-alpine \
  redis-server --unixsocket /var/run/redis/redis.sock --unixsocketperm 777

docker run -it --rm \
  -v /tmp/redis:/var/run/redis \
  redis:7-alpine \
  redis-cli -s /var/run/redis/redis.sock
```

### 5.2.4 Riesgos de los bind mounts

#### 5.2.4.1 El contenedor puede destruir archivos del host

Este es el riesgo más importante y menos comprendido. Si montas un directorio del host en modo lectura-escritura (el modo por defecto), el proceso dentro del contenedor puede hacer cualquier cosa con esos archivos: modificarlos, borrarlos, cifrarlos (ransomware), o llenar el disco.

```bash
# ¡PELIGRO! Esto borrará todos los archivos de /home/user/documentos
docker run --rm \
  -v /home/user/documentos:/data \
  alpine \
  rm -rf /data/*
```

```bash
# ¡PELIGRO! Esto monta TODO el sistema de archivos del host
docker run --rm \
  -v /:/host \
  alpine \
  rm -rf /host/etc/ssh
```

**Regla de oro:** Si el contenedor no necesita escribir, monta SIEMPRE como solo lectura (`:ro` o `readonly`).

```bash
# Montaje seguro de solo lectura
docker run --rm \
  -v /home/user/documentos:/data:ro \
  alpine sh -c 'ls -la /data'
# Si intentas escribir: "Read-only file system"
```

#### 5.2.4.2 Path traversal y escapes de contenedor

Si tu aplicación toma rutas del usuario (query param, API input) y las usa para leer/escribir archivos sin sanitizar, un atacante podría leer archivos del host montados:

```python
# CÓDIGO VULNERABLE - NO USAR EN PRODUCCIÓN
# app.py
@app.route('/leer')
def leer():
    archivo = request.args.get('archivo')
    # Vulnerable: ../ permite escapar de /data
    with open(f'/data/{archivo}', 'r') as f:
        return f.read()
```

Un atacante podría llamar a `http://app/leer?archivo=../../../etc/passwd` y leer `/etc/passwd` del host si el directorio `/etc` está montado.

### 5.2.5 Permisos: el infierno de UID/GID en bind mounts

Este es posiblemente el problema más frustrante al trabajar con bind mounts. El kernel de Linux identifica a los usuarios y grupos por su **UID numérico**, no por su nombre. Cuando un proceso dentro del contenedor crea un archivo, lo hace con su UID. Pero el host y el contenedor tienen sus propias bases de datos de usuarios independientes (`/etc/passwd` diferentes).

#### 5.2.5.1 El escenario clásico del desastre

```bash
# Por defecto, el proceso en el contenedor Alpine corre como root (UID 0)
$ docker run --rm -v "$(pwd)/salida:/salida" alpine \
    sh -c 'echo "creado por root del contenedor" > /salida/archivo.txt'

# En el host:
$ ls -la salida/
# -rw-r--r-- 1 root root 32 May 20 10:00 archivo.txt

$ whoami
# andres  ← UID 501 en macOS, 1000 en Linux

$ echo "intentando modificar..." >> salida/archivo.txt
# bash: salida/archivo.txt: Permission denied
```

El contenedor Alpine ejecuta como root (UID 0) por defecto. Los archivos que crea pertenecen a UID 0. Tu usuario normal en el host (UID 1000) no puede modificarlos ni eliminarlos sin `sudo`.

#### 5.2.5.2 El problema es bidireccional

```bash
$ echo "creado por andres" > salida/mi-archivo.txt
$ ls -la salida/mi-archivo.txt
# -rw-r--r-- 1 andres andres 18 May 20 10:05 salida/mi-archivo.txt

$ docker run --rm -v "$(pwd)/salida:/salida" alpine \
    sh -c 'cat /salida/mi-archivo.txt && echo "modificando..." >> /salida/mi-archivo.txt'

# Salida:
# creado por andres          ← Puede leer (cualquiera puede leer world-readable)
# sh: can't create /salida/mi-archivo.txt: Permission denied   ← No puede escribir
```

Si el archivo pertenece a UID 1000 en el host y el contenedor ejecuta como UID 1001, no puede escribir en él. Este problema es especialmente doloroso en desarrollo:

```bash
# Escenario real: Rails genera assets como root, tú no puedes borrarlos
docker run -v "$(pwd):/app" ruby:3.3 rails assets:precompile
# Ahora tienes cientos de archivos owned by root en public/assets/

# Intentas borrarlos como tu usuario normal
rm -rf public/assets/*
# rm: cannot remove 'public/assets/application-abc123.js': Permission denied
```

#### 5.2.5.3 Solución 1: `--user` en docker run

La solución más inmediata es ejecutar el contenedor con el mismo UID/GID que tu usuario del host:

```bash
# Descubrir tu UID y GID
$ id -u   # 1000
$ id -g   # 1000

# Ejecutar el contenedor con tu UID/GID
docker run --user "$(id -u):$(id -g)" \
  -v "$(pwd):/app" \
  alpine sh -c 'touch /app/archivo.txt'

# En el host:
$ ls -la archivo.txt
# -rw-r--r-- 1 andres andres 0 May 20 10:10 archivo.txt  ✓
```

Pero esta solución tiene limitaciones:
- El usuario con UID 1000 podría no existir en el `/etc/passwd` del contenedor (solo existe si lo creaste en el Dockerfile). El proceso funciona pero `whoami` muestra "I have no name!".
- Archivos como `/etc/resolv.conf` requieren permisos de root para leerse. Si corres como UID 1000, DNS podría fallar.

#### 5.2.5.4 Solución 2: Crear el usuario en el Dockerfile

La solución robusta para imágenes propias es crear un usuario explícito en el Dockerfile con el UID:

```dockerfile
# Dockerfile
FROM node:20-alpine

# Crear un usuario no-root con UID fijo
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -s /bin/sh -D appuser

# Crear directorio de trabajo y asignar propiedad
WORKDIR /app
RUN chown -R appuser:appgroup /app

# Copiar código como root y luego cambiar owner
COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --only=production

COPY --chown=appuser:appgroup . .

# Cambiar a usuario no-root
USER appuser

EXPOSE 3000
CMD ["node", "index.js"]
```

**Truco para imágenes oficiales sin modificar.** Muchas imágenes oficiales usan UID no-root conocido:

| Imagen             | Usuario no-root | UID  |
|--------------------|-----------------|------|
| `nginx`            | `nginx`         | 101  |
| `httpd`            | `www-data`      | 33   |
| `postgres`         | `postgres`      | 999  |
| `redis`            | `redis`         | 999  |
| `node`             | `node`          | 1000 |
| `jenkins/jenkins`  | `jenkins`       | 1000 |

Puedes usar `--user 101` con `nginx` para que los archivos creados pertenezcan a UID 101 en el host.

#### 5.2.5.5 Solución 3: `userns-remap` (namespaces de usuario)

`userns-remap` es una configuración a nivel del daemon de Docker que mapea todos los UID del contenedor a UID diferentes en el host. Por ejemplo, el UID 0 (root) del contenedor se mapea a UID 100000 en el host.

```bash
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

```bash
# Crear el usuario de remapeo
sudo useradd -u 100000 dockremap
sudo usermod -aG docker dockremap

# Crear rangos subordinados
sudo sh -c 'echo "dockremap:100000:65536" > /etc/subuid'
sudo sh -c 'echo "dockremap:100000:65536" > /etc/subgid'

# Reiniciar Docker
sudo systemctl restart docker

# Verificar
docker info | grep -A5 "userns"
```

Con esto, si un contenedor se ejecuta como UID 0 y escribe archivos en un bind mount, en el host verás que pertenecen a UID 100000, no a root.

**Ventajas:** Aislamiento real. Root en contenedor ≠ root en host. Seguridad mejorada drásticamente.

**Desventajas:** Rompe bind mounts existentes (UIDs desplazados). No compatible con ciertos drivers de almacenamiento. Puede causar problemas con imágenes que hacen `chown` internamente. Los volúmenes nombrados funcionan porque Docker hace chown automático.

### 5.2.6 Bind propagation

Cuando montas un sistema de archivos que a su vez tiene submounts, la propagación de bind mount controla si esos submounts son visibles dentro del contenedor y si los montajes creados dentro del contenedor son visibles en el host.

**Modos de propagación:**

| Modo        | Descripción                                                                  |
|-------------|------------------------------------------------------------------------------|
| `private`   | **Por defecto.** Submounts del host NO visibles en contenedor. Aislamiento total. |
| `shared`    | Bidireccional. Cualquier montaje en host es visible en contenedor y viceversa. |
| `slave`     | Unidireccional host→contenedor. Montajes del host visibles en contenedor.    |
| `rshared`   | "Shared recursivo". Igual que shared pero propaga en jerarquía de montajes.  |
| `rslave`    | "Slave recursivo". Igual que slave pero recursivo.                           |
| `rprivate`  | "Private recursivo".                                                         |

**Cuándo usar cada uno:**

```bash
# Caso: sistema de archivos NFS compartido entre hosts Docker
# En el host, montar el NFS como shared
sudo mount --make-shared /mnt/nfs

# Contenedor puede ver y crear submounts visibles en el host y otros contenedores
docker run --mount type=bind,source=/mnt/nfs,target=/data,bind-propagation=shared alpine
```

```bash
# Caso: sistema de archivos con submounts que necesitas ver dentro del contenedor
docker run --mount type=bind,source=/var,target=/host-var,bind-propagation=rslave alpine
```

**Caso de uso real — Clúster de Docker con almacenamiento compartido:**

```bash
# Nodo 1: montar NFS como shared
sudo mkdir -p /mnt/cluster-data
sudo mount -t nfs 192.168.1.100:/exports/data /mnt/cluster-data
sudo mount --make-shared /mnt/cluster-data

# Lanzar aplicación que necesita ver el NFS y sus submounts
docker run -d --name app-node1 \
  --mount type=bind,source=/mnt/cluster-data,target=/shared,bind-propagation=rshared \
  mi-app
```

### 5.2.7 Bind mounts de solo lectura

Una práctica de seguridad fundamental: montar en modo solo lectura cuando el contenedor no necesita escribir:

```bash
# Sintaxis corta -v
docker run -v /host/config:/app/config:ro nginx

# Sintaxis --mount
docker run --mount type=bind,source=/host/config,target=/app/config,readonly nginx
```

```bash
# Verificar que es efectivo
docker run --rm \
  --mount type=bind,source="$(pwd)/readonly-test",target=/data,readonly \
  alpine sh -c 'touch /data/intento.txt'

# touch: /data/intento.txt: Read-only file system
```

### 5.2.8 Problemas comunes con bind mounts y sus soluciones

#### 5.2.8.1 El directorio del host no existe

```bash
# Con -v: Docker CREA el directorio (peligroso si es un typo)
docker run -v /home/user/proyetco:/app alpine ls /app
# Crea /home/user/proyetco (directorio vacío por typo)

# Con --mount: Docker arroja ERROR (mejor)
docker run --mount type=bind,source=/home/user/proyetco,target=/app alpine ls /app
# docker: Error response from daemon: invalid mount config for type "bind":
# bind source path does not exist: /home/user/proyetco
```

**Solución:** Siempre verifica que el directorio existe antes de lanzar el contenedor:

```bash
# Script robusto
SOURCE_DIR="/home/user/proyecto"
if [ ! -d "$SOURCE_DIR" ]; then
    echo "ERROR: El directorio fuente '$SOURCE_DIR' no existe" >&2
    exit 1
fi

docker run --mount type=bind,source="$SOURCE_DIR",target=/app mi-app
```

#### 5.2.8.2 Conflictos con node_modules (desarrollo Node.js)

Un problema clásico: montas tu directorio de proyecto en `/app`, pero `node_modules` de tu host (macOS/Windows) puede tener binarios nativos incompatibles con Linux:

```bash
# ❌ Esto romperá si tienes node_modules de macOS
docker run -v "$(pwd):/app" node:20-alpine npm start

# Error: Cannot find module 'bcrypt'
# (bcrypt tiene binarios compilados para macOS, no para Linux)
```

**Solución 1:** Usar un volumen anónimo para anular `node_modules`:

```bash
# El volumen anónimo en /app/node_modules "tapa" el node_modules del host
docker run -v "$(pwd):/app" -v /app/node_modules node:20-alpine npm start
```

**Solución 2:** No montar todo el proyecto, solo `src/`:

```bash
docker run -v "$(pwd)/src:/app/src" -w /app node:20-alpine npm start
```

#### 5.2.8.3 Case-sensitivity en macOS y Windows

Los sistemas de archivos de macOS (APFS/HFS+, por defecto case-insensitive) y Windows (NTFS, case-insensitive) no distinguen mayúsculas/minúsculas. Linux (ext4, xfs) sí. Si desarrollas en macOS con bind mounts y el contenedor es Linux:

```bash
# En macOS, estos son el mismo archivo
$ touch Archivo.txt archivo.txt

# Dentro del contenedor Linux con bind mount, son archivos diferentes
$ docker run -v "$(pwd):/data" alpine ls /data
# Archivo.txt   archivo.txt   ← ¡Linux los ve como diferentes!
```

Esto puede causar bugs misteriosos cuando importas módulos en JavaScript (`import MyModule from './MyModule'` vs `./mymodule`).

---

## 5.3 Volumes — El mecanismo nativo de persistencia

### 5.3.1 Qué son los volúmenes de Docker

Un **volumen** es un objeto Docker de primera clase: un directorio gestionado íntegramente por Docker, almacenado en una ubicación privada del daemon (`/var/lib/docker/volumes/`), y completamente desacoplado del ciclo de vida de los contenedores.

A diferencia de los bind mounts:
- Docker gestiona la ubicación física (no necesitas saber dónde está)
- Funcionan en todos los sistemas operativos (Linux, macOS con Docker Desktop, Windows con WSL2)
- Pueden ser gestionados con la CLI de Docker (`docker volume` subcomandos)
- Soportan drivers para almacenamiento externo (NFS, cloud, SAN)
- Los permisos iniciales son gestionados automáticamente por Docker
- Son la opción recomendada para datos persistentes en producción

```
┌─────────────────────────────────────────────────────────────────┐
│  HOST                                                           │
│                                                                 │
│  /var/lib/docker/volumes/                                       │
│  ├── mi_volumen/                                                │
│  │   └── _data/          ← Los datos reales viven aquí          │
│  │       ├── mysql/                                             │
│  │       │   ├── ibdata1                                       │
│  │       │   └── mi_db/                                        │
│  │       └── archivo.txt                                       │
│  │                                                              │
│  ├── postgres_data/                                             │
│  │   └── _data/          ← Otro volumen                         │
│  │       └── pgdata/                                           │
│  │                                                              │
│  └── metadata.db         ← Docker mantiene metadatos            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3.2 Anatomía de un volumen en el disco

El directorio `_data` dentro de cada volumen es donde se almacenan realmente los archivos:

```bash
# Crear un volumen
$ docker volume create datos-app
# datos-app

# Inspeccionar para ver su ubicación física
$ docker volume inspect datos-app
```

```json
[
    {
        "CreatedAt": "2026-05-20T10:30:00+02:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/datos-app/_data",
        "Name": "datos-app",
        "Options": {},
        "Scope": "local"
    }
]
```

```bash
# En Linux, puedes explorar el directorio directamente (como root)
$ sudo ls -la /var/lib/docker/volumes/datos-app/_data/
# total 0
# (vacío, el volumen recién creado no tiene datos)

# Escribir algo en el volumen usando un contenedor temporal
$ docker run --rm -v datos-app:/data alpine \
    sh -c 'echo "datos persistentes" > /data/importante.txt'

# Verificar que el archivo existe en el volumen
$ sudo ls -la /var/lib/docker/volumes/datos-app/_data/
# -rw-r--r-- 1 root root 20 May 20 10:31 importante.txt

$ sudo cat /var/lib/docker/volumes/datos-app/_data/importante.txt
# datos persistentes
```

**En macOS (Docker Desktop):** Los volúmenes se almacenan dentro de la VM de Linux que Docker Desktop ejecuta internamente. Para acceder necesitas entrar a la VM:

```bash
# Entrar a la VM de Docker Desktop (solo en macOS/Windows)
$ docker run -it --rm --privileged --pid=host justincormack/nsenter1
/ # ls /var/lib/docker/volumes/
```

### 5.3.3 Gestión de volúmenes con la CLI

```bash
# Crear un volumen nombrado
docker volume create mi-volumen

# Crear con labels y opciones de driver
docker volume create \
  --label entorno=produccion \
  --label app=mi-aplicacion \
  mi-volumen-prod

# Listar todos los volúmenes
docker volume ls
# DRIVER    VOLUME NAME
# local     datos-app
# local     mi-volumen
# local     mi-volumen-prod

# Listar con filtros
docker volume ls --filter name=mi
docker volume ls --filter label=entorno=produccion
docker volume ls --filter dangling=true  # Volúmenes no usados por ningún contenedor

# Inspeccionar un volumen
docker volume inspect mi-volumen

# Eliminar un volumen (debe no estar en uso)
docker volume rm mi-volumen

# Eliminar todos los volúmenes no usados (pide confirmación)
docker volume prune

# Eliminar volúmenes no usados con filtro por edad
docker volume prune --filter "until=72h"  # Solo los no usados en las últimas 72h
```

### 5.3.4 Usar volúmenes con contenedores

```bash
# Sintaxis -v: primer argumento es el NOMBRE del volumen (no una ruta)
docker run -d --name mysql-db \
  -v mysql_data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# Sintaxis --mount: más explícita
docker run -d --name mysql-db \
  --mount type=volume,source=mysql_data,target=/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# Si el volumen no existe, Docker lo crea automáticamente
docker run -d --name redis-cache \
  -v redis_data:/data \
  redis:7-alpine
# Crea automáticamente el volumen "redis_data" si no existe
```

### 5.3.5 Volúmenes anónimos vs volúmenes nombrados

**Volumen anónimo:** Se crea cuando especificas solo la ruta del contenedor sin nombre de volumen. Docker genera un nombre aleatorio (hash).

```bash
# Volumen anónimo: solo la ruta del contenedor
docker run -v /data alpine sh -c 'date > /data/timestamp.txt'

# Ver el volumen anónimo creado
docker volume ls
# DRIVER    VOLUME NAME
# local     8a7b3c9d2e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8

# Problema: ¿cuál es cuál? Imposible saber sin inspeccionar
docker volume ls --filter dangling=true
# Muestra los anónimos que no están asociados a ningún contenedor
```

**Volumen nombrado:** Le das un nombre significativo. Puedes reutilizarlo, hacerle backup, migrarlo.

```bash
# Volumen nombrado: nombre + ruta del contenedor
docker run -v mysql_prod_data:/var/lib/mysql mysql:8.0

# Ventajas:
# 1. Nombre descriptivo: mysql_prod_data
# 2. Reutilizable: el mismo volumen en múltiples contenedores (con cuidado)
# 3. Inspeccionable: docker volume inspect mysql_prod_data
# 4. Gestionable: backup, restore, migrate
```

**Regla de oro para producción:**
- Nunca uses volúmenes anónimos en producción.
- Siempre usa volúmenes NOMBRADOS.
- Usa convenciones de nomenclatura: `<app>-<entorno>-<dato>` (ej: `mi-app-prod-postgres`).

### 5.3.6 Poblar volúmenes: el comportamiento inicial

Cuando montas un volumen en un directorio que ya contiene archivos en la imagen, Docker **copia** esos archivos al volumen la primera vez. Esto es crucial para bases de datos:

```bash
# La imagen de MySQL tiene /var/lib/mysql con archivos de inicialización
# Cuando montas un volumen vacío en /var/lib/mysql, Docker copia esos archivos
# al volumen. Así MySQL encuentra su estructura inicial y arranca correctamente.

docker run -d --name mysql-test \
  -v mysql_fresh:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=testpass \
  mysql:8.0

# El volumen mysql_fresh ahora contiene los archivos base de MySQL
docker run --rm -v mysql_fresh:/data alpine ls -la /data
# Verás ibdata1, mysql/, sys/, performance_schema/
```

**Importante:** Este comportamiento SOLO ocurre cuando el volumen está vacío. Si el volumen ya contiene datos (de un contenedor anterior), Docker NO sobreescribe los datos existentes.

### 5.3.7 Compartir volúmenes entre contenedores

Múltiples contenedores pueden montar el mismo volumen simultáneamente:

```bash
# Contenedor 1: escritor
docker run -d --name escritor \
  -v datos-compartidos:/data \
  alpine sh -c 'while true; do echo "$(date): escribiendo" >> /data/registro.log; sleep 1; done'

# Contenedor 2: lector
docker run -d --name lector \
  -v datos-compartidos:/data \
  alpine sh -c 'tail -f /data/registro.log'

# Ver logs del lector
docker logs -f lector
# [Muestra las líneas que el escritor va añadiendo]
```

**Advertencia:** Compartir volúmenes entre contenedores requiere manejo cuidadoso de concurrencia. Dos procesos escribiendo al mismo archivo simultáneamente pueden corromper datos. Bases de datos como MySQL o PostgreSQL NO soportan acceso concurrente al mismo directorio de datos desde múltiples procesos.

### 5.3.8 Drivers de volumen

Los drivers de volumen son plugins que determinan DÓNDE y CÓMO se almacenan los datos. El driver por defecto es `local`, que guarda los datos en el sistema de archivos del host.

#### 5.3.8.1 El driver `local`

El driver `local` almacena los datos en el sistema de archivos del daemon Docker (por defecto `/var/lib/docker/volumes/`). Aunque se llama "local", puede apuntar a cualquier ruta del host e incluso a montajes de red configurados a nivel de sistema operativo.

**Opciones del driver `local`:**

| Opción   | Descripción                       | Ejemplo              |
|----------|-----------------------------------|----------------------|
| `device` | Ruta del dispositivo o directorio | `/mnt/disco-rapido/` |
| `type`   | Tipo de sistema de archivos       | `none`, `ext4`, `nfs`|
| `o`      | Opciones de montaje               | `bind`, `rw`, `noatime` |

```bash
# Crear un volumen local en un disco específico (SSD rápido)
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/mnt/ssd/mi-app \
  --opt o=bind \
  datos-rapidos

# Verificar que realmente apunta al disco SSD
$ ls -la /var/lib/docker/volumes/datos-rapidos/_data
# Muestra el contenido de /mnt/ssd/mi-app
```

#### 5.3.8.2 Volumen NFS con el driver `local`

Docker no tiene un driver nativo para NFS, pero puedes usar el driver `local` con las opciones adecuadas para montar un share NFS como volumen:

```bash
# 1. Asegúrate de que el paquete nfs-common esté instalado en el host
sudo apt-get install -y nfs-common   # Debian/Ubuntu
sudo yum install -y nfs-utils         # RHEL/CentOS

# 2. Verifica conectividad con el servidor NFS
showmount -e 192.168.1.50
# Export list for 192.168.1.50:
# /exports/data    192.168.1.0/24

# 3. Crea el volumen con opciones NFS
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.50,rw,nfsvers=4,hard,intr \
  --opt device=:/exports/data \
  nfs-datos

# 4. Verifica
docker volume inspect nfs-datos
```

```json
[
    {
        "CreatedAt": "2026-05-20T11:00:00+02:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/nfs-datos/_data",
        "Name": "nfs-datos",
        "Options": {
            "device": ":/exports/data",
            "o": "addr=192.168.1.50,rw,nfsvers=4,hard,intr",
            "type": "nfs"
        },
        "Scope": "local"
    }
]
```

```bash
# 5. Usarlo con un contenedor
docker run --rm -v nfs-datos:/data alpine sh -c 'echo "desde contenedor $(hostname)" > /data/hostname.txt && cat /data/hostname.txt'
# desde contenedor a1b2c3d4e5f6

# 6. Verificar desde otro host Docker que monte el mismo NFS
docker run --rm -v nfs-datos:/data alpine cat /data/hostname.txt
# desde contenedor a1b2c3d4e5f6   ← ¡mismo archivo!
```

**Opciones NFS importantes:**

| Opción                  | Significado                                    | Cuándo usarla                     |
|-------------------------|------------------------------------------------|-----------------------------------|
| `nfsvers=4`             | Usar NFSv4 (recomendado)                       | Siempre que el servidor lo soporte|
| `nfsvers=3`             | Usar NFSv3 (legado)                            | Compatibilidad con servidores antiguos |
| `hard`                  | El cliente reintenta indefinidamente            | Datos críticos                    |
| `soft`                  | El cliente devuelve error tras timeout          | Datos no críticos                 |
| `intr`                  | Permite interrumpir operaciones NFS colgadas    | Junto con `hard`                  |
| `noatime`               | No actualizar tiempo de acceso                  | Rendimiento                       |
| `rsize=1048576`         | Tamaño de lectura (1MB)                         | Optimización de throughput        |
| `wsize=1048576`         | Tamaño de escritura (1MB)                       | Optimización de throughput        |

```bash
# Configuración optimizada para base de datos sobre NFS (con cuidado)
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt "o=addr=192.168.1.50,rw,nfsvers=4,hard,intr,noatime,rsize=1048576,wsize=1048576" \
  --opt device=:/exports/postgres-data \
  postgres-nfs
```

#### 5.3.8.3 Volumen CIFS/SMB (Windows shares)

```bash
# Requiere cifs-utils en el host
sudo apt-get install -y cifs-utils

# Crear volumen que apunta a un share de Windows/Samba
docker volume create \
  --driver local \
  --opt type=cifs \
  --opt "o=username=usuario,password=contraseña,uid=1000,gid=1000,file_mode=0755,dir_mode=0755,vers=3.0,addr=192.168.1.100" \
  --opt device=//192.168.1.100/shared/docs \
  docs-empresa

# Usar con contenedor
docker run --rm -v docs-empresa:/documentos alpine ls /documentos
```

**Seguridad con CIFS:** La contraseña es visible en `docker volume inspect`. Para producción, usa un archivo de credenciales:

```bash
# Crear archivo de credenciales
cat > /etc/docker/cifs-credentials << 'EOF'
username=mi-usuario
password=contraseña-segura
domain=MI-DOMINIO
EOF

chmod 600 /etc/docker/cifs-credentials

# Usar credentials en lugar de username/password
docker volume create \
  --driver local \
  --opt type=cifs \
  --opt "o=credentials=/etc/docker/cifs-credentials,uid=1000,gid=1000,vers=3.0,addr=192.168.1.100" \
  --opt device=//192.168.1.100/shared/docs \
  docs-empresa
```

#### 5.3.8.4 Volúmenes en la nube (drivers de terceros)

Para entornos de producción en la nube, necesitas que los volúmenes se almacenen en el block storage del proveedor:

**REX-Ray (Dell EMC) — AWS EBS:**

```bash
# Instalar REX-Ray
docker plugin install rexray/ebs:latest \
  EBS_ACCESSKEY=AKIAIOSFODNN7EXAMPLE \
  EBS_SECRETKEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Crear volumen en AWS EBS
docker volume create --driver rexray/ebs --name datos-aws --opt size=100

# Usar con contenedor (se adjunta disco EBS a la instancia EC2 automáticamente)
docker run -d --name app-aws -v datos-aws:/data mi-app
```

**Google Cloud Persistent Disk:**

```bash
docker plugin install gce
docker volume create --driver gce --name datos-gcp --opt size=50
```

**Azure File Storage:**

```bash
docker plugin install docker4x/cloudstor:azure \
  CLOUD_PLATFORM=AZURE \
  AZURE_STORAGE_ACCOUNT=miscuenta \
  AZURE_STORAGE_ACCOUNT_KEY=clave-super-secreta

docker volume create --driver cloudstor:azure --name datos-azure
```

**CSI (Container Storage Interface) — El estándar moderno:**

Desde Kubernetes 1.13, CSI es el estándar para storage plugins. Docker Swarm también soporta CSI en versiones recientes.

```
┌──────────────────────────────────────────────┐
│        ORQUESTADOR (K8s, Swarm)              │
│                    │                         │
│              CSI Interface                    │
│                    │                         │
│     ┌──────────────┼──────────────┐          │
│     ▼              ▼              ▼          │
│  AWS EBS      GCP PD        Azure Disk      │
│  CSI Driver   CSI Driver    CSI Driver      │
└──────────────────────────────────────────────┘
```

#### 5.3.8.5 Plugins de volumen populares

| Plugin              | Proveedor      | Tipo                     | Caso de uso                   |
|---------------------|----------------|--------------------------|-------------------------------|
| **NetApp Trident**  | NetApp         | NAS/SAN (ONTAP)          | Enterprise, snapshots, clones |
| **Portworx**        | Pure Storage   | SDS                      | HA, multi-cloud, DR           |
| **Ceph RBD**        | Ceph (OS)      | Block storage distribuido| On-premise, gran escala       |
| **GlusterFS**       | Red Hat        | NAS distribuido          | On-premise, archivos compartidos |
| **Longhorn**        | Rancher/SUSE   | Block storage distribuido| Kubernetes, backup a S3       |
| **OpenEBS**         | Open Source    | Container-attached storage| Kubernetes nativo            |
| **vSphere Volume**  | VMware         | vSAN/VMFS                | Entornos VMware               |

```bash
# Ejemplo: Crear volumen con plugin NetApp
docker volume create -d netapp --name datos-oracle -o size=500G

# Ejemplo: Crear volumen con plugin Portworx (replicado en 3 nodos)
docker volume create -d pxd --name datos-replicados -o repl=3 -o size=200G
```

### 5.3.9 `docker volume inspect` en detalle

```bash
$ docker volume inspect mysql_data
```

```json
[
    {
        "CreatedAt": "2026-05-20T09:15:00+02:00",
        "Driver": "local",
        "Labels": {
            "com.example.environment": "production",
            "com.example.service": "database"
        },
        "Mountpoint": "/var/lib/docker/volumes/mysql_data/_data",
        "Name": "mysql_data",
        "Options": {},
        "Scope": "local"
    }
]
```

**Campos importantes:**
- **`Mountpoint`**: Ruta física en el host. Útil para backups a nivel de sistema.
- **`Driver`**: Qué driver gestiona este volumen. `local` es el más común.
- **`Labels`**: Metadatos que adjuntaste al crear el volumen.
- **`Scope`**: `local` (un solo host Docker) o `global` (clúster Swarm).
- **`Options`**: Argumentos pasados al driver.

---

## 5.4 tmpfs Mounts — Almacenamiento en memoria RAM

### 5.4.1 Qué son los tmpfs mounts

Un **tmpfs mount** es un sistema de archivos temporal que reside completamente en la memoria RAM del host. No toca el disco en ningún momento. Cuando el contenedor se detiene, los datos se evaporan instantáneamente.

```
┌──────────────────────────────────────────────────────────┐
│                     HOST                                 │
│                                                          │
│  ┌────────────────────────────────────────┐              │
│  │            MEMORIA RAM                │              │
│  │                                        │              │
│  │  ┌──────────────────────────┐          │              │
│  │  │  tmpfs mount             │          │              │
│  │  │  /tmp/sesiones (100MB)   │          │              │
│  │  │                          │          │              │
│  │  │  session_abc123.json     │          │              │
│  │  │  session_def456.json     │          │              │
│  │  │  cache_temp.bin          │          │              │
│  │  └──────────────────────────┘          │              │
│  └────────────────────────────────────────┘              │
│                                                          │
│  ┌────────────────────────────────────────┐              │
│  │            DISCO                      │              │
│  │  (NO se usa para tmpfs)               │              │
│  └────────────────────────────────────────┘              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 5.4.2 Sintaxis de tmpfs mounts

```bash
# Sintaxis corta (--tmpfs)
docker run --tmpfs /ruta/en/contenedor imagen

# Sintaxis larga (--mount)
docker run --mount type=tmpfs,destination=/ruta/en/contenedor imagen
```

```bash
# Ejemplo básico
docker run --rm --tmpfs /tmp:rw,noexec,nosuid,size=100m alpine \
  sh -c 'dd if=/dev/zero of=/tmp/gran-archivo bs=1M count=50 && \
         ls -lh /tmp/gran-archivo'

# 50+0 records in
# 50+0 records out
# -rw-r--r-- 1 root root 50M May 20 10:00 /tmp/gran-archivo
```

```bash
# Con opciones detalladas usando --mount
docker run --rm \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=256m,tmpfs-mode=1777 \
  alpine sh -c 'mount | grep tmpfs'

# tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noexec,relatime,size=262144k)
```

### 5.4.3 Opciones de tmpfs

| Opción        | Descripción                              | Ejemplo                        |
|---------------|------------------------------------------|--------------------------------|
| `tmpfs-size`  | Tamaño máximo en bytes                   | `tmpfs-size=256m`              |
| `tmpfs-mode`  | Permisos en octal                        | `tmpfs-mode=1770`              |
| `noexec`      | Prohibir ejecución de binarios           | Seguridad                      |
| `nosuid`      | Ignorar bits SUID y SGID                 | Seguridad                      |
| `nodev`       | Ignorar archivos de dispositivo          | Seguridad                      |
| `rw` / `ro`   | Lectura-escritura (defecto) o solo-lectura |                            |

```bash
# tmpfs con todas las restricciones de seguridad
docker run --rm \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=128m,tmpfs-mode=1777,noexec,nosuid,nodev \
  alpine sh -c 'mount | grep /tmp'

# tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noexec,relatime,size=131072k,mode=1777)
```

```bash
# Verificar que noexec funciona
docker run --rm \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=64m,noexec \
  alpine sh -c 'cp /bin/ls /tmp/ && /tmp/ls'

# sh: /tmp/ls: Permission denied  ← noexec bloquea la ejecución
```

### 5.4.4 Casos de uso de tmpfs

#### 5.4.4.1 Datos sensibles temporales (tokens, credenciales)

```bash
# Un contenedor que procesa pagos y genera tokens temporales
docker run --rm \
  --mount type=tmpfs,destination=/var/run/secrets,tmpfs-size=10m,tmpfs-mode=0700 \
  procesador-pagos

# Dentro del contenedor:
# echo "token=$(openssl rand -hex 64)" > /var/run/secrets/session-token
# El token existe solo en RAM. Si el contenedor crashea, no queda rastro.
```

#### 5.4.4.2 Archivos temporales que no deben tocar disco

```bash
# Compilación con archivos intermedios en RAM
docker run --rm \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=512m \
  -v "$(pwd):/src" \
  -w /src \
  golang:1.22 sh -c \
    'mkdir -p /tmp/build && \
     go build -o /tmp/build/mi-app ./cmd/server && \
     cp /tmp/build/mi-app /src/'
```

#### 5.4.4.3 Cachés de aplicación efímeras

```bash
# Redis con datos en RAM + tmpfs para swap/tránsito
docker run -d --name redis-ram \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=64m \
  redis:7-alpine \
  redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
```

#### 5.4.4.4 Evitar escrituras involuntarias en la capa writable

```bash
# Java escribe logs en /tmp por defecto. Con tmpfs, esas escrituras
# van a RAM y no llenan la capa writable del contenedor.
docker run -d --name java-app \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=200m \
  openjdk:21-jdk \
  java -jar /app/app.jar
```

---

## 5.5 Tabla comparativa — Bind mount vs Volume vs tmpfs

| Característica                  | Bind Mount                       | Volume                               | tmpfs                          |
|---------------------------------|-----------------------------------|--------------------------------------|---------------------------------|
| **Ubicación física**            | Cualquier ruta del host           | `/var/lib/docker/volumes/`           | Memoria RAM del host            |
| **Quién gestiona la ubicación** | El usuario                        | Docker                               | El kernel (automático)          |
| **Persiste tras `docker rm`**   | Sí (son archivos del host)        | Sí                                   | No (está en RAM)                |
| **Persiste tras reinicio**      | Sí                                | Sí                                   | No                              |
| **Soporte multiplataforma**     | Limitado (depende del SO host)    | Sí (Linux, macOS, Windows)           | Linux nativo                    |
| **Backup con herramientas Docker**| No (usa rsync, tar)            | Sí (`docker run` + tar)              | No aplica (es temporal)         |
| **Migración entre hosts**       | Copia manual (rsync, scp)         | Backup + restore                     | No aplica                       |
| **Drivers externos (NFS, cloud)**| No (montar NFS en host, luego bind)| Sí                                | No                              |
| **Rendimiento**                 | Depende del storage del host      | Depende del driver                   | El más rápido (RAM)             |
| **Aislamiento**                 | Bajo (contenedor ve host)         | Alto (directorio privado)            | Medio (comparte RAM del host)   |
| **Seguridad**                   | Riesgo: contenedor modifica host  | Seguro (Docker gestiona)             | Seguro (se destruye al parar)   |
| **Soporte en Compose**          | Sí                                | Sí                                   | Sí                              |
| **Soporte en Swarm**            | No (solo managers legacy)         | Sí (volumes + drivers)               | Parcial                         |
| **Equivalente en Kubernetes**   | hostPath                          | PersistentVolume                     | emptyDir (medium: Memory)       |
| **Ideal para**                  | Desarrollo, config files          | Bases de datos, datos de app         | Tokens, sesiones, caché temporal|
| **NO usar para**                | Producción (datos críticos)       | Archivos que editas directamente     | Datos que deben persistir       |

### 5.5.1 Árbol de decisión: ¿qué tipo de almacenamiento necesito?

```
¿Los datos deben sobrevivir al contenedor?
├── NO
│   └── ¿Deben estar en RAM (no en disco)?
│       ├── SÍ → tmpfs mount
│       └── NO → No necesitas montaje (la capa writable es suficiente
│                 para datos efímeros durante la vida del contenedor)
│
└── SÍ
    └── ¿Necesitas editar los archivos desde el host?
        ├── SÍ → Bind mount
        └── NO → Volume (nombrado)
             └── ¿Los datos deben compartirse entre múltiples hosts?
                 ├── SÍ → Volume con driver NFS, CIFS, o cloud
                 └── NO → Volume con driver local
```

### 5.5.2 Comparativa de rendimiento

```bash
#!/bin/bash
# Script de benchmark comparativo: bind vs volume vs tmpfs

echo "=== Benchmark de I/O: bind vs volume vs tmpfs ==="

# Preparar
mkdir -p /tmp/benchmark-host
echo "test data" > /tmp/benchmark-host/test.txt

# 1. Bind mount
echo "--- Bind mount ---"
docker run --rm \
  -v /tmp/benchmark-host:/data \
  alpine sh -c \
    "time dd if=/dev/zero of=/data/bench bs=1M count=500 oflag=direct 2>&1" \
  | grep real

# 2. Volume (local)
docker volume create benchmark-vol > /dev/null 2>&1
echo "--- Volume (local) ---"
docker run --rm \
  -v benchmark-vol:/data \
  alpine sh -c \
    "time dd if=/dev/zero of=/data/bench bs=1M count=500 oflag=direct 2>&1" \
  | grep real
docker volume rm benchmark-vol > /dev/null 2>&1

# 3. tmpfs
echo "--- tmpfs ---"
docker run --rm \
  --mount type=tmpfs,destination=/data,tmpfs-size=1024m \
  alpine sh -c \
    "time dd if=/dev/zero of=/data/bench bs=1M count=500 2>&1" \
  | grep real

rm -rf /tmp/benchmark-host

# Resultados típicos en un servidor con SSD NVMe:
# --- Bind mount ---
# real    0m 0.52s      (SSD rápido)
# --- Volume (local) ---
# real    0m 0.54s      (Similar, overhead mínimo)
# --- tmpfs ---
# real    0m 0.08s      (RAM es mucho más rápido)
```

---

## 5.6 Permisos y ownership en profundidad

### 5.6.1 El problema fundamental: UID ≠ UID entre host y contenedor

El kernel de Linux identifica usuarios y grupos por su **UID numérico**, no por su nombre. Los nombres (`root`, `www-data`, `postgres`) son solo alias legibles que se resuelven consultando `/etc/passwd` y `/etc/group`. Cada contenedor tiene su propio `/etc/passwd`, independiente del host.

```bash
# En el host
$ grep andres /etc/passwd
# andres:x:1000:1000:Andres:/home/andres:/bin/bash

# En un contenedor Alpine
$ docker run --rm alpine grep "x:1000:" /etc/passwd
# (vacío — no existe un usuario con UID 1000)

# En un contenedor Ubuntu
$ docker run --rm ubuntu:22.04 grep "x:1000:" /etc/passwd
# (vacío — ídem)
```

Cuando un proceso con UID 1000 crea un archivo en un volumen, el sistema de archivos registra UID 1000 como propietario. Si ese volumen se monta en un contenedor donde UID 1000 pertenece a otro usuario (o no existe), tienes un problema de permisos.

### 5.6.2 El caso de PostgreSQL: un ejemplo paradigmático

La imagen oficial de PostgreSQL ejecuta el proceso como usuario `postgres` (UID 999 dentro del contenedor):

```bash
# Iniciar PostgreSQL con volumen nombrado
docker run -d --name postgres-db \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# Ver los permisos DENTRO del contenedor
docker exec postgres-db ls -la /var/lib/postgresql/data
# drwx------ 19 postgres postgres 4096 May 20 10:00 .
# -rw-------  1 postgres postgres    3 May 20 10:00 PG_VERSION
# (todo owned by postgres:postgres, UID 999)

# Ver los permisos DESDE el host
sudo ls -la /var/lib/docker/volumes/pgdata/_data
# drwx------ 19 999 999 4096 May 20 10:00 .
# -rw-------  1 999 999    3 May 20 10:00 PG_VERSION
# (UID 999 — no tiene nombre en el host porque no existe ese UID)
```

### 5.6.3 Estrategias para manejar UID/GID

Docker ofrece varias estrategias para manejar la discrepancia de UID/GID. Ninguna es perfecta para todos los casos. La elección depende de tu escenario.

#### 5.6.3.1 Estrategia 1: Ejecutar como root y asumir el riesgo

```bash
# Archivos creados son owned by root en el host
docker run --rm -v datos:/data alpine sh -c 'touch /data/archivo'

sudo ls -la /var/lib/docker/volumes/datos/_data
# -rw-r--r-- 1 root root 0 May 20 10:00 archivo   ← root
```

**Cuándo usarlo:** Nunca en producción. Solo en desarrollo local si los permisos no importan y el trabajo es efímero.

#### 5.6.3.2 Estrategia 2: Especificar `--user` en docker run

```bash
# Ejecutar con el mismo UID/GID que el usuario del host
docker run --user "$(id -u):$(id -g)" \
  -v datos:/data \
  alpine sh -c 'touch /data/archivo'

# Ahora el archivo pertenece a tu usuario en el host
ls -la /var/lib/docker/volumes/datos/_data
# -rw-r--r-- 1 andres andres 0 May 20 10:00 archivo ✓
```

**Problemas de esta estrategia:**
1. El UID/GID debe existir en `/etc/passwd` del contenedor (no crítico pero molesto).
2. Algunos archivos del sistema requieren root para leerse.
3. Si la imagen fue diseñada para correr como usuario específico, cambiar el usuario puede romperla.

#### 5.6.3.3 Estrategia 3: Crear un usuario dedicado en el Dockerfile

Esta es la práctica recomendada para imágenes propias:

```dockerfile
# Dockerfile para una aplicación Node.js
FROM node:20-alpine

# Crear grupo y usuario con UID/GID fijos
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -s /bin/sh -D appuser

# Crear directorios de la aplicación y asignar propiedad
WORKDIR /app
RUN mkdir -p /app/data /app/logs && \
    chown -R appuser:appgroup /app

# Instalar dependencias como root
COPY package*.json ./
RUN npm ci --only=production

# Copiar código y asignar propiedad al usuario de la app
COPY --chown=appuser:appgroup . .

# El usuario final
USER appuser:appgroup

EXPOSE 3000
CMD ["node", "server.js"]
```

**Ventaja clave:** UID fijo (1000) en el Dockerfile. Si tu usuario de desarrollo también es UID 1000 (lo normal en Linux), los permisos coinciden automáticamente.

#### 5.6.3.4 Estrategia 4: `useradd` condicional en ENTRYPOINT con gosu

Para imágenes que necesitan ejecutar setup como root y luego bajar privilegios:

```dockerfile
# Dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y gosu

ARG UID=1000
ARG GID=1000
RUN groupadd -g $GID appgroup && \
    useradd -u $UID -g appgroup -m appuser

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

WORKDIR /app
COPY . .

ENTRYPOINT ["/entrypoint.sh"]
CMD ["python3", "app.py"]
```

```bash
#!/bin/sh
# entrypoint.sh
# Set up inicial como root
echo "Ejecutando setup como $(whoami) (UID: $(id -u))"

# Tareas que requieren root:
chown -R appuser:appgroup /app/data
chown -R appuser:appgroup /app/logs

# Bajar privilegios y ejecutar la aplicación
exec gosu appuser "$@"
```

#### 5.6.3.5 Estrategia 5: Script fix-permissions en ENTRYPOINT

```bash
#!/bin/bash
# fix-permissions.sh
# Ajusta los permisos de directorios montados al UID/GID de la aplicación

APP_UID=${APP_UID:-1000}
APP_GID=${APP_GID:-1000}
DATA_DIR=${DATA_DIR:-/var/lib/data}
LOG_DIR=${LOG_DIR:-/var/log/app}

echo "Fixing permissions for UID=$APP_UID GID=$APP_GID"

for dir in "$DATA_DIR" "$LOG_DIR"; do
    if [ -d "$dir" ]; then
        echo "  Setting ownership of $dir"
        chown -R "$APP_UID:$APP_GID" "$dir"
        chmod 755 "$dir"
    fi
done

echo "Permissions fixed."
```

```bash
# docker-entrypoint.sh (combinado)
/usr/local/bin/fix-permissions
exec gosu appuser "$@"
```

### 5.6.4 `userns-remap` — La opción nuclear

`userns-remap` es una configuración del daemon Docker que crea un namespace de usuario separado para todos los contenedores. El UID 0 (root) dentro del contenedor se mapea a un UID alto (ej: 100000) en el host.

```bash
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

```bash
# Configuración completa
sudo systemctl stop docker

sudo useradd -u 100000 -g docker dockremap
sudo sh -c 'echo "dockremap:100000:65536" >> /etc/subuid'
sudo sh -c 'echo "dockremap:100000:65536" >> /etc/subgid'

sudo mv /var/lib/docker /var/lib/docker.old
sudo systemctl start docker

docker info | grep -A5 "userns"
# Username: dockremap
# UID: 100000
```

```bash
# Comportamiento con userns-remap activo
docker run --rm -v datos:/data alpine sh -c 'touch /data/root-file.txt'

# En el host: el archivo pertenece a UID 100000, no a root
sudo ls -n /var/lib/docker/volumes/datos/_data
# -rw-r--r-- 1 100000 100000 0 May 20 10:00 root-file.txt
```

**Consideraciones de userns-remap:**
- Rompe bind mounts existentes (UIDs desplazados).
- No compatible con todos los drivers.
- Volúmenes nombrados funcionan porque Docker hace chown automático.
- `--net=host` y `--pid=host` no funcionan con userns-remap.

---

## 5.7 Backup y restauración de volúmenes

### 5.7.1 El principio: contenedor temporal como herramienta de backup

Docker no tiene comandos nativos para copiar datos dentro/fuera de un volumen. La estrategia estándar es lanzar un contenedor temporal que monte el volumen (origen) y un directorio del host (destino), y ejecute `tar` para transferir los datos.

```
┌─────────────┐     ┌───────────────────────────┐     ┌──────────────────┐
│   VOLUMEN    │────▶│ CONTENEDOR TEMPORAL       │────▶│  HOST            │
│  mi_volumen  │     │ alpine + tar              │     │  $(pwd)/backups/ │
│  (origen)    │     │                           │     │  (destino)       │
│              │     │ mount: mi_volumen → /data │     │                  │
│              │     │ mount: $(pwd)     → /bk   │     │  backup.tar.gz   │
│              │     │                           │     │                  │
│              │     │ tar czf /bk/backup.tar.gz │     │                  │
│              │     │   -C /data .              │     │                  │
└─────────────┘     └───────────────────────────┘     └──────────────────┘
```

### 5.7.2 Backup de un volumen

```bash
# Sintaxis genérica
docker run --rm \
  -v <NOMBRE_VOLUMEN>:/data \
  -v "$(pwd)":/backup \
  alpine \
  tar czf /backup/<nombre>-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .
```

```bash
# Ejemplo concreto: backup de volumen de MySQL
export FECHA=$(date +%Y%m%d-%H%M%S)
export BACKUP_DIR="/home/user/backups"
export VOLUMEN="mysql_data"

mkdir -p "$BACKUP_DIR"

docker run --rm \
  -v "$VOLUMEN:/data:ro" \
  -v "$BACKUP_DIR:/backup" \
  alpine \
  tar czf "/backup/mysql-backup-${FECHA}.tar.gz" -C /data .

echo "Backup creado: ${BACKUP_DIR}/mysql-backup-${FECHA}.tar.gz"
ls -lh "$BACKUP_DIR/mysql-backup-${FECHA}.tar.gz"
# -rw-r--r-- 1 root root 250M May 20 12:00 mysql-backup-20260520-120000.tar.gz
```

**Nota importante:** El montaje del volumen es `:ro` (read-only). Esto evita que el proceso de backup modifique accidentalmente los datos mientras los lee.

### 5.7.3 Restauración de un volumen

```bash
# Primero, asegúrate de que el volumen existe (puede estar vacío o contener datos viejos)
docker volume create mysql_restaurado

# Restaurar desde backup
docker run --rm \
  -v mysql_restaurado:/data \
  -v "$BACKUP_DIR:/backup:ro" \
  alpine \
  tar xzf /backup/mysql-backup-20260520-120000.tar.gz -C /data

# Verificar restauración
docker run --rm -v mysql_restaurado:/data alpine ls -la /data
```

**Restauración limpia (volumen nuevo):**

```bash
# 1. Eliminar el volumen viejo (o crear uno nuevo)
docker volume rm mysql_restaurado 2>/dev/null
docker volume create mysql_restaurado

# 2. Restaurar en volumen limpio
docker run --rm \
  -v mysql_restaurado:/data \
  -v "$BACKUP_DIR:/backup:ro" \
  alpine \
  tar xzf /backup/mysql-backup-20260520-120000.tar.gz -C /data
```

### 5.7.4 Backup de base de datos: dump lógico vs dump físico

El backup con `tar` de los archivos crudos se llama **dump físico**. Es rápido pero:
- Requiere que la base de datos esté parada (o en modo hot-backup).
- Depende de la versión exacta del motor.
- No es portable entre arquitecturas (x86 vs ARM).

El **dump lógico** (con `pg_dump` o `mysqldump`) exporta SQL, portable y restaurable en cualquier versión. Es más lento pero más seguro.

#### 5.7.4.1 Backup lógico de PostgreSQL

```bash
# Método 1: pg_dump desde el host
docker run --rm \
  --network host \
  postgres:16 \
  pg_dump -h localhost -p 5432 -U postgres -d mi_db \
  > backup-$(date +%Y%m%d).sql

# Método 2: usando docker exec sobre el contenedor existente
docker exec postgres-db pg_dump -U postgres mi_db \
  > backup-$(date +%Y%m%d).sql

# Método 3: Backup completo del cluster (todas las BBDD)
docker exec postgres-db pg_dumpall -U postgres \
  > backup-all-$(date +%Y%m%d).sql
```

```bash
# Restauración de PostgreSQL
# 1. Crear la base de datos vacía (si no existe)
docker exec postgres-db createdb -U postgres mi_db_restaurada

# 2. Restaurar
docker exec -i postgres-db psql -U postgres -d mi_db_restaurada \
  < backup-20260520.sql
```

#### 5.7.4.2 Backup lógico de MySQL/MariaDB

```bash
# Backup con mysqldump
docker exec mysql-db mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" \
  --all-databases \
  --single-transaction \
  --quick \
  --lock-tables=false \
  --routines \
  --triggers \
  --events \
  > mysql-backup-$(date +%Y%m%d-%H%M%S).sql

# Opciones importantes:
# --single-transaction: backup consistente sin bloquear tablas (InnoDB)
# --quick: no cargar todo en memoria
# --lock-tables=false: no bloquear tablas MyISAM
# --routines: incluir stored procedures
# --triggers: incluir triggers
# --events: incluir eventos programados
```

```bash
# Restauración de MySQL
docker exec -i mysql-db mysql -u root -p"$MYSQL_ROOT_PASSWORD" \
  < mysql-backup-20260520-120000.sql
```

### 5.7.5 Estrategias de backup automatizado

#### 5.7.5.1 Cron en el host (simple)

```bash
# /etc/cron.d/docker-backup
# Backup diario a las 2:00 AM
0 2 * * * root /usr/local/bin/docker-backup.sh
```

```bash
#!/bin/bash
# /usr/local/bin/docker-backup.sh
# Backup automático de volúmenes Docker

set -euo pipefail

BACKUP_DIR="/mnt/backups/docker"
RETENTION_DAYS=30
LOG_FILE="/var/log/docker-backups.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

mkdir -p "$BACKUP_DIR"

# Lista de volúmenes a respaldar
VOLUMES=("mysql_data" "postgres_data" "app_uploads" "redis_data")

for VOL in "${VOLUMES[@]}"; do
    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
    BACKUP_FILE="${BACKUP_DIR}/${VOL}-${TIMESTAMP}.tar.gz"

    log "Iniciando backup de volumen: $VOL"

    if ! docker volume inspect "$VOL" > /dev/null 2>&1; then
        log "ERROR: Volumen '$VOL' no encontrado. Saltando."
        continue
    fi

    if docker run --rm \
        -v "${VOL}:/data:ro" \
        -v "${BACKUP_DIR}:/backup" \
        alpine \
        tar czf "/backup/${VOL}-${TIMESTAMP}.tar.gz" -C /data . 2>/dev/null; then
        log "OK: Backup creado: $BACKUP_FILE ($(du -h "$BACKUP_FILE" | cut -f1))"
    else
        log "ERROR: Fallo el backup de $VOL"
        continue
    fi

    # Limpiar backups antiguos
    find "$BACKUP_DIR" -name "${VOL}-*.tar.gz" -mtime +$RETENTION_DAYS -delete
    log "Limpieza de backups de $VOL con más de $RETENTION_DAYS días"
done

log "Backup diario completado"
```

#### 5.7.5.2 Backup a almacenamiento cloud (S3)

```bash
#!/bin/bash
# backup-to-s3.sh
# Backup de volúmenes Docker y subida a S3

set -euo pipefail

S3_BUCKET="s3://mi-bucket/docker-backups"
AWS_PROFILE="produccion"
VOLUMEN="postgres_data"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
TEMP_DIR=$(mktemp -d)

# 1. Backup del volumen
echo "Creando backup de $VOLUMEN..."
docker run --rm \
  -v "${VOLUMEN}:/data:ro" \
  -v "${TEMP_DIR}:/backup" \
  alpine \
  tar czf "/backup/${VOLUMEN}-${TIMESTAMP}.tar.gz" -C /data .

# 2. Subir a S3
echo "Subiendo a S3..."
aws s3 cp \
  "${TEMP_DIR}/${VOLUMEN}-${TIMESTAMP}.tar.gz" \
  "${S3_BUCKET}/${VOLUMEN}-${TIMESTAMP}.tar.gz" \
  --profile "$AWS_PROFILE" \
  --storage-class STANDARD_IA

# 3. Verificar subida
if aws s3 ls "${S3_BUCKET}/${VOLUMEN}-${TIMESTAMP}.tar.gz" --profile "$AWS_PROFILE" > /dev/null 2>&1; then
    echo "Backup subido exitosamente a S3"
else
    echo "ERROR: La subida a S3 falló" >&2
fi

# 4. Limpiar
rm -rf "$TEMP_DIR"
```

---

## 5.8 Migración de volúmenes entre hosts

### 5.8.1 El problema: los volúmenes son locales por defecto

Los volúmenes Docker con el driver `local` residen en el disco del host. Si necesitas mover un contenedor y sus datos a otro servidor, necesitas migrar el volumen. Docker no tiene un built-in `docker volume copy` o `docker volume migrate`. Hay que hacerlo manualmente o con herramientas de terceros.

### 5.8.2 Migración manual (backup → transferir → restaurar)

```bash
#!/bin/bash
# ======= EN EL HOST ORIGEN =======

# 1. Backup del volumen
ssh user@origen "
  docker run --rm \
    -v postgres_data:/data:ro \
    -v /tmp:/backup \
    alpine \
    tar czf /backup/postgres-migracion.tar.gz -C /data .
"

# 2. Copiar el backup al host destino
scp user@origen:/tmp/postgres-migracion.tar.gz /tmp/

# ======= EN EL HOST DESTINO =======

# 3. Crear volumen nuevo
docker volume create postgres_data

# 4. Restaurar
docker run --rm \
  -v postgres_data:/data \
  -v /tmp:/backup:ro \
  alpine \
  tar xzf /backup/postgres-migracion.tar.gz -C /data

# 5. Verificar
docker run --rm -v postgres_data:/data alpine ls -la /data
```

### 5.8.3 Migración directa con pipe SSH

Para volúmenes pequeños/medianos, puedes hacer la migración en un solo paso sin archivo intermedio:

```bash
# Desde el host origen
docker run --rm \
  -v postgres_data:/data:ro \
  alpine \
  tar cz -C /data . | \
  ssh user@destino "
    docker volume create postgres_data && \
    docker run --rm -i \
      -v postgres_data:/data \
      alpine \
      tar xz -C /data
  "
```

```bash
# Con barra de progreso (pv)
docker run --rm \
  -v postgres_data:/data:ro \
  alpine \
  tar cz -C /data . | \
  pv -s $(docker run --rm -v postgres_data:/data:ro alpine du -sb /data | cut -f1) | \
  ssh user@destino "
    docker volume create postgres_data 2>/dev/null;
    docker run --rm -i \
      -v postgres_data:/data \
      alpine \
      tar xz -C /data
  "
```

### 5.8.4 Migración a la nube

```bash
# ======= PASO 1: Backup en origen =======
docker run --rm \
  -v mis_datos:/data:ro \
  -v /tmp:/backup \
  alpine \
  tar czf /backup/mis-datos-migracion.tar.gz -C /data .

# ======= PASO 2: Subir a S3 =======
aws s3 cp /tmp/mis-datos-migracion.tar.gz \
  s3://mi-bucket/migraciones/mis-datos-migracion.tar.gz

# ======= PASO 3: Descargar en destino =======
aws s3 cp s3://mi-bucket/migraciones/mis-datos-migracion.tar.gz /tmp/

# ======= PASO 4: Restaurar en destino =======
docker volume create mis_datos
docker run --rm \
  -v mis_datos:/data \
  -v /tmp:/backup:ro \
  alpine \
  tar xzf /backup/mis-datos-migracion.tar.gz -C /data

# ======= PASO 5: Verificar y limpiar =======
docker run --rm -v mis_datos:/data alpine ls -la /data
aws s3 rm s3://mi-bucket/migraciones/mis-datos-migracion.tar.gz
```

### 5.8.5 Script de migración robusto

```bash
#!/bin/bash
# docker-volume-migrate.sh
# Migra un volumen Docker de un host a otro vía SSH
# Uso: ./docker-volume-migrate.sh <volumen> <origen> <destino>

set -euo pipefail

VOLUME="$1"
SRC_HOST="$2"
DST_HOST="$3"

if [ $# -ne 3 ]; then
    echo "Uso: $0 <nombre-volumen> <host-origen> <host-destino>"
    exit 1
fi

echo "=== Migración del volumen '$VOLUME' ==="
echo "  Origen:  $SRC_HOST"
echo "  Destino: $DST_HOST"

# [1/6] Verificar conectividad
echo "[1/6] Verificando conectividad..."
ssh -o ConnectTimeout=5 "$SRC_HOST" echo ok > /dev/null 2>&1 || {
    echo "ERROR: No se puede conectar a $SRC_HOST"; exit 1; }
ssh -o ConnectTimeout=5 "$DST_HOST" echo ok > /dev/null 2>&1 || {
    echo "ERROR: No se puede conectar a $DST_HOST"; exit 1; }

# [2/6] Verificar que el volumen existe en origen
echo "[2/6] Verificando volumen en origen..."
ssh "$SRC_HOST" "docker volume inspect $VOLUME" > /dev/null 2>&1 || {
    echo "ERROR: El volumen '$VOLUME' no existe en $SRC_HOST"; exit 1; }

# [3/6] Calcular tamaño
echo "[3/6] Calculando tamaño..."
SIZE=$(ssh "$SRC_HOST" \
  "docker run --rm -v $VOLUME:/data:ro alpine du -sb /data 2>/dev/null | cut -f1")
echo "  Tamaño: $(numfmt --to=iec $SIZE 2>/dev/null || echo "${SIZE} bytes")"

# [4/6] Transferir vía pipe SSH
echo "[4/6] Transfiriendo datos..."
ssh "$SRC_HOST" "docker run --rm -v $VOLUME:/data:ro alpine tar cz -C /data ." | \
  ssh "$DST_HOST" "
    docker volume create $VOLUME 2>/dev/null || true
    docker run --rm -i -v $VOLUME:/data alpine tar xz -C /data
  "

# [5/6] Verificar integridad
echo "[5/6] Verificando integridad..."
SRC_COUNT=$(ssh "$SRC_HOST" \
  "docker run --rm -v $VOLUME:/data:ro alpine find /data -type f | wc -l")
DST_COUNT=$(ssh "$DST_HOST" \
  "docker run --rm -v $VOLUME:/data:ro alpine find /data -type f | wc -l")

echo "  Archivos en origen:  $SRC_COUNT"
echo "  Archivos en destino: $DST_COUNT"

if [ "$SRC_COUNT" -eq "$DST_COUNT" ]; then
    echo "  ✓ Verificación OK: mismo número de archivos"
else
    echo "  ⚠ ADVERTENCIA: diferente número de archivos."
fi

echo "[6/6] Migración completada"
```

### 5.8.6 Herramientas de terceros para migración

| Herramienta               | Descripción                                            |
|---------------------------|--------------------------------------------------------|
| **offen/docker-volume-backup** | CLI para backup/restore con encriptación         |
| **dobi**                  | Build automation con soporte para backup de volúmenes |
| **Velero**                | Backup y migración de volúmenes Kubernetes            |


---

## 5.9 Volúmenes en Docker Compose

### 5.9.1 Volúmenes nombrados en Compose

Docker Compose tiene soporte de primera clase para volúmenes. Los volúmenes se declaran en dos lugares dentro del archivo `docker-compose.yml`:

1. **Nivel `services`**: Referencias de montaje (qué volumen, en qué ruta)
2. **Nivel `volumes`** (top-level): Declaración de volúmenes (nombre, driver, opciones)

```yaml
# docker-compose.yml
version: "3.8"

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mi_app
    volumes:
      # Referencia al volumen declarado abajo + ruta en el contenedor
      - postgres_data:/var/lib/postgresql/data
      # Bind mount para init scripts
      - ./db/init:/docker-entrypoint-initdb.d:ro

  app:
    build: .
    volumes:
      - app_uploads:/app/uploads
      - app_logs:/app/logs
    depends_on:
      - db

# Declaración top-level de volúmenes
volumes:
  postgres_data:
    driver: local
  app_uploads:
    driver: local
  app_logs:
    driver: local
```

### 5.9.2 Volúmenes con driver options en Compose

```yaml
version: "3.8"

services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: produccion
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
    driver: local
    driver_opts:
      type: none
      device: /mnt/disco-rapido/mysql
      o: bind
```

```yaml
# NFS en Docker Compose
volumes:
  shared_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,nfsvers=4,rw,hard,intr
      device: ":/exports/shared"
```

```yaml
# CIFS en Docker Compose
volumes:
  windows_share:
    driver: local
    driver_opts:
      type: cifs
      o: "username=user,password=pass,vers=3.0,addr=192.168.1.200,file_mode=0755,dir_mode=0755"
      device: "//192.168.1.200/shared"
```

### 5.9.3 Volúmenes externos (`external: true`)

Los volúmenes externos son aquellos que ya existen en el host (creados manualmente con `docker volume create` o por otro proyecto Compose). Compose no intenta crearlos ni eliminarlos.

```yaml
version: "3.8"

services:
  app:
    image: mi-app:v2
    volumes:
      - datos_externos:/data

volumes:
  datos_externos:
    external: true   # Debe existir previamente
```

Si el volumen externo no existe, `docker compose up` falla con:

```
ERROR: Volume datos_externos declared as external, but could not be found.
Please create the volume manually using `docker volume create datos_externos` and try again.
```

```yaml
# Volumen externo con nombre diferente al que usas en el servicio
volumes:
  mi_volumen:
    external:
      name: nombre-real-del-volumen-en-docker
```

### 5.9.4 Bind mounts en Compose

```yaml
version: "3.8"

services:
  dev:
    build: .
    volumes:
      # Bind mount: ruta relativa al directorio del compose file
      - ./src:/app/src
      # Bind mount: ruta absoluta
      - /etc/app/config.yaml:/app/config.yaml:ro
      # Volumen anónimo para node_modules (anula el bind mount en ese subdirectorio)
      - /app/node_modules
    ports:
      - "3000:3000"
```

### 5.9.5 tmpfs en Compose

```yaml
version: "3.8"

services:
  cache-service:
    image: redis:7-alpine
    tmpfs:
      - /tmp:size=100M,mode=1777
      - /var/run/secrets:size=10M,mode=0700,noexec
```

```yaml
# Sintaxis larga para tmpfs en Compose (v3.6+)
services:
  app:
    image: mi-app
    tmpfs:
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 268435456   # 256 MB en bytes
          mode: 1777
```

### 5.9.6 Compose multi-entorno con volúmenes

```yaml
# docker-compose.yml (base, usado en todos los entornos)
version: "3.8"

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

```yaml
# docker-compose.prod.yml (override de producción)
version: "3.8"

services:
  db:
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./backups:/backups:ro

volumes:
  db_data:
    driver: local
    driver_opts:
      type: none
      device: /mnt/disco-empresarial/postgres
      o: bind
```

```bash
# Producción: combina base + override
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Desarrollo: combina base + override de desarrollo
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

### 5.9.7 Compose con múltiples volúmenes y labels

```yaml
version: "3.8"

services:
  wordpress:
    image: wordpress:6.5-php8.2-apache
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_content:/var/www/html/wp-content
  
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wppass
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init-scripts:/docker-entrypoint-initdb.d:ro

volumes:
  wordpress_content:
    labels:
      com.example.app: "wordpress"
      com.example.env: "production"
      com.example.backup: "daily"

  mysql_data:
    labels:
      com.example.app: "wordpress"
      com.example.env: "production"
      com.example.backup: "daily"
    driver: local
    driver_opts:
      device: /mnt/fast-storage/mysql
      type: none
      o: bind
```

---

## 5.10 Patrones de datos en Docker

### 5.10.1 Sidecar para backups

El patrón sidecar consiste en ejecutar un contenedor secundario junto al contenedor principal, compartiendo el mismo volumen, cuyo único trabajo es realizar backups periódicos.

```
┌────────────────────────────────────────────────┐
│           docker-compose stack                  │
│                                                  │
│  ┌──────────────┐       ┌──────────────────────┐ │
│  │   postgres   │       │   backup-sidecar     │ │
│  │              │       │                      │ │
│  │  Puerto 5432 │       │  Cron interno:       │ │
│  │              │       │  - pg_dump c/1h      │ │
│  │  VOLUMEN ────┼───────┤  - Rotación: 7 días  │ │
│  │  pg_data     │  R/O  │  - Sinc a S3 c/6h    │ │
│  │              │       │                      │ │
│  └──────────────┘       └──────────────────────┘ │
│                                                  │
└────────────────────────────────────────────────┘
```

```dockerfile
# Dockerfile.backup
FROM alpine:3.19

RUN apk add --no-cache \
    postgresql16-client \
    mariadb-client \
    aws-cli \
    dcron \
    bash

COPY scripts/backup.sh /usr/local/bin/backup
COPY crontab /etc/crontabs/root

RUN chmod +x /usr/local/bin/backup

ENTRYPOINT ["crond", "-f", "-d", "8"]
```

```bash
#!/bin/bash
# backup.sh — Script de backup ejecutado por cron en el sidecar

set -euo pipefail

BACKUP_DIR="/backups"
RETENTION_DAYS=7
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
LOG_FILE="/var/log/backup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

mkdir -p "$BACKUP_DIR"

# Backup de PostgreSQL (pg_dump lógico)
if [ -n "${PGHOST:-}" ]; then
    log "Iniciando backup lógico de PostgreSQL en $PGHOST..."
    DB_NAME="${PGDATABASE:-app}"
    BACKUP_FILE="${BACKUP_DIR}/postgres-${DB_NAME}-${TIMESTAMP}.sql.gz"

    PGPASSWORD="${PGPASSWORD}" pg_dump \
        -h "$PGHOST" \
        -U "${PGUSER:-postgres}" \
        -d "$DB_NAME" \
        | gzip > "$BACKUP_FILE"

    log "Backup PostgreSQL: $BACKUP_FILE ($(du -h "$BACKUP_FILE" | cut -f1))"
fi

# Backup de MySQL/MariaDB (mysqldump lógico)
if [ -n "${MYSQL_HOST:-}" ]; then
    log "Iniciando backup de MySQL en $MYSQL_HOST..."
    BACKUP_FILE="${BACKUP_DIR}/mysql-all-${TIMESTAMP}.sql.gz"

    mysqldump \
        -h "$MYSQL_HOST" \
        -u "${MYSQL_USER:-root}" \
        -p"${MYSQL_PASSWORD}" \
        --all-databases \
        --single-transaction \
        --quick \
        | gzip > "$BACKUP_FILE"

    log "Backup MySQL: $BACKUP_FILE ($(du -h "$BACKUP_FILE" | cut -f1))"
fi

# Backup de volúmenes (dump físico con tar)
if [ -d "/data" ] && [ "$(ls -A /data 2>/dev/null)" ]; then
    log "Iniciando backup físico del volumen /data..."
    BACKUP_FILE="${BACKUP_DIR}/volume-data-${TIMESTAMP}.tar.gz"

    tar czf "$BACKUP_FILE" -C /data .

    log "Backup volumen /data: $BACKUP_FILE ($(du -h "$BACKUP_FILE" | cut -f1))"
fi

# Limpiar backups antiguos
log "Limpiando backups con más de $RETENTION_DAYS días..."
find "$BACKUP_DIR" -type f -mtime +$RETENTION_DAYS -delete
log "Limpieza completada"

# Opcional: sincronizar a S3
if [ -n "${S3_BUCKET:-}" ]; then
    log "Sincronizando con S3: ${S3_BUCKET}..."
    aws s3 sync "$BACKUP_DIR" "$S3_BUCKET" --storage-class STANDARD_IA
    log "Sincronización con S3 completada"
fi

log "=== Ciclo de backup finalizado ==="
```

```cron
# crontab (formato cron estándar)
# Cada hora: backup lógico de bases de datos
0 * * * * /usr/local/bin/backup >> /var/log/backup-cron.log 2>&1

# Cada 6 horas: sincronización adicional a S3
0 */6 * * * aws s3 sync /backups s3://mi-bucket/backups --storage-class STANDARD_IA

# Limpieza de logs cada domingo a las 3 AM
0 3 * * 0 truncate -s 0 /var/log/backup-cron.log /var/log/backup.log
```

```yaml
# docker-compose.yml con sidecar de backup
version: "3.8"

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: supersecret
      POSTGRES_DB: produccion
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - backend

  app:
    image: mi-app:v3
    environment:
      DATABASE_URL: postgresql://postgres:supersecret@db:5432/produccion
    ports:
      - "3000:3000"
    depends_on:
      - db
    networks:
      - backend

  backup:
    build:
      context: .
      dockerfile: Dockerfile.backup
    environment:
      PGHOST: db
      PGUSER: postgres
      PGPASSWORD: supersecret
      PGDATABASE: produccion
      S3_BUCKET: s3://mi-bucket/backups   # Opcional
      AWS_ACCESS_KEY_ID: ${AWS_KEY}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET}
    volumes:
      - backups_data:/backups
      - pg_data:/data:ro        # Montaje de solo lectura para dump físico opcional
    networks:
      - backend
    restart: always

volumes:
  pg_data:
  backups_data:

networks:
  backend:
    driver: bridge
```

### 5.10.2 Init container (contenedor de inicialización)

El patrón init container ejecuta un contenedor temporal antes de arrancar la aplicación principal. Este contenedor inicializa datos, configura permisos, ejecuta migraciones, descarga archivos, o prepara cualquier precondición que la app necesite.

```
┌──────────────────────────────────────────────────────┐
│  Secuencia de inicio de la aplicación                │
│                                                      │
│  1. Init Container: chown, migraciones, seed data    │
│     ┌──────────┐                                     │
│     │  init    │ (se ejecuta una vez, luego muere)   │
│     └────┬─────┘                                     │
│          │ Ok                                        │
│          ▼                                           │
│  2. App Container: la aplicación principal            │
│     ┌──────────┐                                     │
│     │   app    │ (arranca y corre continuamente)     │
│     └──────────┘                                     │
│                                                      │
│  Comparten el mismo volumen                          │
└──────────────────────────────────────────────────────┘
```

```bash
#!/bin/bash
# init.sh — Script de inicialización

set -e

echo "[INIT] Iniciando tareas de inicialización..."

# 1. Esperar a que la base de datos esté disponible
echo "[INIT] Esperando a PostgreSQL..."
while ! pg_isready -h db -p 5432 -U postgres; do
    echo "[INIT] PostgreSQL no está listo. Reintentando en 2s..."
    sleep 2
done
echo "[INIT] PostgreSQL está listo."

# 2. Ejecutar migraciones
echo "[INIT] Ejecutando migraciones de base de datos..."
cd /app && npm run migrate

# 3. Insertar datos semilla si la tabla está vacía
echo "[INIT] Verificando datos semilla..."
ROW_COUNT=$(psql -h db -U postgres -d mi_app -t -c "SELECT count(*) FROM usuarios;")
if [ "$ROW_COUNT" -eq 0 ]; then
    echo "[INIT] Insertando datos semilla..."
    psql -h db -U postgres -d mi_app < /seed/seed.sql
fi

# 4. Ajustar permisos del volumen de datos
echo "[INIT] Ajustando permisos de /data..."
chown -R appuser:appuser /data
chmod 755 /data

# 5. Descargar archivos necesarios
if [ ! -f /data/model.bin ]; then
    echo "[INIT] Descargando modelo..."
    curl -o /data/model.bin https://storage.example.com/models/v3/model.bin
fi

echo "[INIT] Inicialización completada exitosamente."
```

```yaml
# docker-compose.yml con init container
version: "3.8"

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mi_app
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  init:
    build:
      context: .
      dockerfile: Dockerfile.init
    environment:
      DATABASE_URL: postgresql://postgres:secret@db:5432/mi_app
    volumes:
      - app_data:/data
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend

  app:
    image: mi-app:v3
    environment:
      DATABASE_URL: postgresql://postgres:secret@db:5432/mi_app
    volumes:
      - app_data:/data
    depends_on:
      init:
        condition: service_completed_successfully
      db:
        condition: service_healthy
    ports:
      - "3000:3000"
    networks:
      - backend

volumes:
  pg_data:
  app_data:

networks:
  backend:
```

**Nota:** `depends_on` con `condition: service_completed_successfully` solo funciona en Compose v3.8+ y requiere `docker compose` (no el antiguo `docker-compose` de Python). Esto asegura que `app` no arranca hasta que `init` termina con éxito (exit code 0).

### 5.10.3 Data-only container (patrón legacy)

El patrón del data-only container fue popular en los primeros años de Docker pero hoy está mayormente obsoleto. Consistía en crear un contenedor que nunca se ejecutaba (`docker create`, no `docker run`), solo para mantener referencias a volúmenes.

```bash
# Crear un contenedor "data-only" (no se ejecuta, solo existe)
docker create \
  -v /var/lib/mysql \
  -v /var/log/app \
  --name datos-mysql \
  mysql:8.0 \
  /bin/true

# Otros contenedores montan sus volúmenes desde este
docker run -d --name mysql \
  --volumes-from datos-mysql \
  mysql:8.0

docker run --rm --volumes-from datos-mysql alpine ls /var/lib/mysql
```

**Por qué este patrón está obsoleto:**
- Docker introdujo `docker volume create` y volúmenes nombrados como objetos de primera clase.
- Los volúmenes nombrados son más fáciles de gestionar, inspeccionar y respaldar.
- `--volumes-from` tiene comportamiento sorprendente (monta TODOS los volúmenes del contenedor fuente, no puedes elegir).
- Los volúmenes nombrados funcionan con `docker volume ls`, `inspect`, `prune`, etc.

**Regla moderna:** Usa volúmenes nombrados (`docker volume create`). No uses el patrón data-only container.

---

## 5.11 Laboratorio práctico

### 5.11.1 Objetivo del laboratorio

Vamos a construir un stack real con persistencia:

1. **MySQL** con volumen nombrado para datos persistentes
2. **Contenedor sidecar de backup** con cron para backups automáticos
3. **Script de restauración** para probar recuperación ante desastres
4. **Volumen NFS** opcional para compartir backups entre hosts

### 5.11.2 Estructura del laboratorio

```
lab-volumenes/
├── docker-compose.yml
├── Dockerfile.backup
├── scripts/
│   ├── backup.sh
│   ├── restore.sh
│   └── simulate-disaster.sh
├── crontab
├── init-scripts/
│   └── 01-create-db.sql
└── README.md
```

### 5.11.3 Paso 1: Crear el script de inicialización de MySQL

```sql
-- init-scripts/01-create-db.sql
CREATE DATABASE IF NOT EXISTS libreria
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE libreria;

CREATE TABLE IF NOT EXISTS libros (
    id INT AUTO_INCREMENT PRIMARY KEY,
    titulo VARCHAR(255) NOT NULL,
    autor VARCHAR(255) NOT NULL,
    isbn VARCHAR(20) UNIQUE,
    precio DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    fecha_publicacion DATE,
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    actualizado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_autor (autor),
    INDEX idx_titulo (titulo)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS ventas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    libro_id INT NOT NULL,
    cantidad INT NOT NULL,
    precio_unitario DECIMAL(10,2) NOT NULL,
    fecha_venta TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (libro_id) REFERENCES libros(id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- Datos de prueba
INSERT INTO libros (titulo, autor, isbn, precio, stock, fecha_publicacion) VALUES
('Cien Años de Soledad', 'Gabriel García Márquez', '978-0307474728', 19.99, 150, '1967-06-05'),
('Don Quijote de la Mancha', 'Miguel de Cervantes', '978-8424116191', 24.99, 200, '1605-01-16'),
('1984', 'George Orwell', '978-0451524935', 14.99, 300, '1949-06-08'),
('El Principito', 'Antoine de Saint-Exupéry', '978-0156012195', 9.99, 500, '1943-04-06'),
('Orgullo y Prejuicio', 'Jane Austen', '978-0141439518', 12.99, 175, '1813-01-28');

INSERT INTO ventas (libro_id, cantidad, precio_unitario) VALUES
(1, 2, 19.99),
(3, 1, 14.99),
(1, 5, 18.99),
(2, 3, 24.99),
(4, 10, 9.49);
```

### 5.11.4 Paso 2: Dockerfile del sidecar de backup

```dockerfile
# Dockerfile.backup
FROM alpine:3.19

LABEL description="Sidecar de backup automático para MySQL"

RUN apk add --no-cache \
    mariadb-client \
    bash \
    dcron \
    coreutils \
    curl

COPY scripts/backup.sh /usr/local/bin/backup
RUN chmod +x /usr/local/bin/backup

COPY crontab /etc/crontabs/root
RUN chmod 0644 /etc/crontabs/root

RUN mkdir -p /backups /var/log

ENTRYPOINT ["crond", "-f", "-d", "8"]
```

### 5.11.5 Paso 3: Script de backup del sidecar

```bash
#!/bin/bash
# scripts/backup.sh
# Script de backup ejecutado por cron en el sidecar

set -euo pipefail

BACKUP_DIR="/backups"
RETENTION_DAYS=7
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
LOG_FILE="/var/log/backup.log"
MYSQL_HOST="${MYSQL_HOST:-db}"
MYSQL_USER="${MYSQL_USER:-root}"
MYSQL_PASSWORD="${MYSQL_PASSWORD:-rootpass}"
DB_NAME="${DB_NAME:-libreria}"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

mkdir -p "$BACKUP_DIR"

# 1. Esperar a que MySQL esté disponible
log "Verificando conectividad con MySQL en $MYSQL_HOST..."
for i in $(seq 1 30); do
    if mysqladmin ping -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" --silent 2>/dev/null; then
        log "MySQL está disponible"
        break
    fi
    if [ "$i" -eq 30 ]; then
        log "ERROR: MySQL no disponible tras 30 intentos"
        exit 1
    fi
    sleep 2
done

# 2. Backup lógico (mysqldump)
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}-${TIMESTAMP}.sql.gz"
log "Iniciando backup lógico de la base de datos '$DB_NAME'..."

if mysqldump \
    -h "$MYSQL_HOST" \
    -u "$MYSQL_USER" \
    -p"$MYSQL_PASSWORD" \
    --single-transaction \
    --quick \
    --routines \
    --triggers \
    --events \
    "$DB_NAME" | gzip > "$BACKUP_FILE"; then

    BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log "Backup creado: $BACKUP_FILE ($BACKUP_SIZE)"
else
    log "ERROR: Falló el backup de la base de datos"
    exit 1
fi

# 3. Backup de esquema solamente
SCHEMA_FILE="${BACKUP_DIR}/${DB_NAME}-schema-${TIMESTAMP}.sql"
log "Exportando esquema de base de datos..."
mysqldump \
    -h "$MYSQL_HOST" \
    -u "$MYSQL_USER" \
    -p"$MYSQL_PASSWORD" \
    --no-data \
    --routines \
    --triggers \
    --events \
    "$DB_NAME" > "$SCHEMA_FILE"
log "Esquema exportado: $SCHEMA_FILE"

# 4. Rotación: eliminar backups antiguos
log "Rotación de backups: eliminando archivos con más de $RETENTION_DAYS días..."
DELETED=$(find "$BACKUP_DIR" -type f -name "${DB_NAME}-*.sql.gz" -mtime +$RETENTION_DAYS -print -delete | wc -l)
log "Eliminados $DELETED backups antiguos"

# 5. Estadísticas de backups
TOTAL_BACKUPS=$(find "$BACKUP_DIR" -type f -name "${DB_NAME}-*.sql.gz" | wc -l)
TOTAL_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
log "Total backups almacenados: $TOTAL_BACKUPS ($TOTAL_SIZE)"

log "=== Ciclo de backup finalizado exitosamente ==="
```

### 5.11.6 Paso 4: Configuración de cron para el sidecar

```cron
# crontab para el sidecar de backup
# Formato: minuto hora día-mes mes día-semana comando

# Backup cada hora (en el minuto 0)
0 * * * * /usr/local/bin/backup >> /var/log/backup-cron.log 2>&1

# Backup diario adicional a medianoche
0 0 * * * echo "=== BACKUP DIARIO $(date) ===" >> /var/log/backup-cron.log && \
           /usr/local/bin/backup >> /var/log/backup-cron.log 2>&1

# Limpieza de logs cada domingo a las 3 AM
0 3 * * 0 truncate -s 0 /var/log/backup-cron.log /var/log/backup.log
```

### 5.11.7 Paso 5: Script de restauración

```bash
#!/bin/bash
# scripts/restore.sh
# Script de restauración de MySQL desde backup
# Uso: ./restore.sh <archivo_backup.sql.gz> [--force]

set -euo pipefail

if [ $# -lt 1 ]; then
    echo "Uso: $0 <archivo-backup.sql.gz> [--force]"
    echo "Ejemplo: $0 /backups/libreria-20260520-120000.sql.gz"
    echo ""
    echo "Backups disponibles:"
    docker compose exec backup ls -lh /backups/*.sql.gz 2>/dev/null || \
      echo "  (no se encontraron backups)"
    exit 1
fi

BACKUP_FILE="$1"
MYSQL_HOST="${MYSQL_HOST:-db}"
MYSQL_USER="${MYSQL_USER:-root}"
MYSQL_PASSWORD="${MYSQL_PASSWORD:-rootpass}"
DB_NAME="${DB_NAME:-libreria}"

echo "==========================================="
echo " RESTAURACIÓN DE BASE DE DATOS"
echo "==========================================="
echo "Base de datos: $DB_NAME"
echo "Backup: $BACKUP_FILE"
echo "Host MySQL: $MYSQL_HOST"
echo ""

# Verificar que el backup existe
if [ ! -f "$BACKUP_FILE" ]; then
    echo "ERROR: El archivo de backup no existe: $BACKUP_FILE"
    exit 1
fi

echo "Tamaño del backup: $(du -h "$BACKUP_FILE" | cut -f1)"

# Confirmar (a menos que se pase --force)
if [ "${2:-}" != "--force" ]; then
    echo ""
    echo "ADVERTENCIA: Esto SOBREESCRIBIRÁ la base de datos '$DB_NAME'."
    echo "   Todos los datos actuales se PERDERÁN."
    read -r -p "Escribe 'RESTAURAR' para continuar: " CONFIRM

    if [ "$CONFIRM" != "RESTAURAR" ]; then
        echo "Restauración cancelada."
        exit 0
    fi
fi

echo ""
echo "[1/3] Verificando conectividad con MySQL..."
if ! mysqladmin ping -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" --silent 2>/dev/null; then
    echo "ERROR: No se puede conectar a MySQL"
    exit 1
fi
echo "  Conectividad OK"

echo "[2/3] Eliminando base de datos existente..."
mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" \
    -e "DROP DATABASE IF EXISTS \`$DB_NAME\`;"
echo "  Base de datos eliminada"

echo "  Creando base de datos vacía..."
mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" \
    -e "CREATE DATABASE \`$DB_NAME\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
echo "  Base de datos creada"

echo "[3/3] Restaurando datos desde backup..."
gunzip -c "$BACKUP_FILE" | mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" "$DB_NAME"
RESTORE_EXIT=$?

if [ $RESTORE_EXIT -eq 0 ]; then
    echo "  Restauración completada exitosamente"
else
    echo "ERROR: Falló la restauración (código $RESTORE_EXIT)"
    exit $RESTORE_EXIT
fi

echo ""
echo "=== Verificación post-restauración ==="
TABLES=$(mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" \
    -N -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = '$DB_NAME';")
echo "Número de tablas restauradas: $TABLES"

BOOK_COUNT=$(mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" \
    -N -e "SELECT COUNT(*) FROM $DB_NAME.libros;")
echo "Libros en la base de datos: $BOOK_COUNT"

echo ""
echo "✓ Restauración completada exitosamente"
echo "   Base de datos: $DB_NAME"
echo "   Tablas: $TABLES"
echo "   Libros: $BOOK_COUNT"
```

### 5.11.8 Paso 6: docker-compose.yml completo del laboratorio

```yaml
# docker-compose.yml — Stack completo del laboratorio
version: "3.8"

services:
  # ─────────────────────────────────────────
  # MySQL: base de datos principal
  # ─────────────────────────────────────────
  db:
    image: mysql:8.0
    container_name: lab-mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: libreria
    volumes:
      # Volumen nombrado para datos persistentes
      - mysql_data:/var/lib/mysql
      # Scripts de inicialización (crean tablas y datos de prueba)
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
    ports:
      - "3306:3306"
    networks:
      - lab-net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-prootpass"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  # ─────────────────────────────────────────
  # Sidecar de backup automático
  # ─────────────────────────────────────────
  backup:
    build:
      context: .
      dockerfile: Dockerfile.backup
    container_name: lab-backup
    environment:
      MYSQL_HOST: db
      MYSQL_USER: root
      MYSQL_PASSWORD: rootpass
      DB_NAME: libreria
    volumes:
      # Dónde se almacenan los backups generados
      - backups_data:/backups
      # Acceso R/O al volumen de MySQL (para dump físico opcional)
      - mysql_data:/mysql-data:ro
    depends_on:
      db:
        condition: service_healthy
    networks:
      - lab-net
    restart: unless-stopped
    # El sidecar no expone puertos. Solo hace su trabajo silenciosamente.

  # ─────────────────────────────────────────
  # Adminer: interfaz web para MySQL
  # ─────────────────────────────────────────
  adminer:
    image: adminer:latest
    container_name: lab-adminer
    environment:
      ADMINER_DEFAULT_SERVER: db
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - lab-net
    restart: unless-stopped

volumes:
  mysql_data:
    labels:
      com.lab.app: "libreria"
      com.lab.backup: "hourly"
  backups_data:
    labels:
      com.lab.app: "backups"
      com.lab.retention: "7d"

networks:
  lab-net:
    driver: bridge
```

### 5.11.9 Paso 7: Script de simulación de desastre

```bash
#!/bin/bash
# scripts/simulate-disaster.sh
# Simula un desastre: elimina el volumen de MySQL y lo recrea.
# Luego restaura desde backup para probar la recuperación.

set -euo pipefail

COMPOSE_FILE="${COMPOSE_FILE:-docker-compose.yml}"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
LAB_DIR="$(dirname "$SCRIPT_DIR")"

cd "$LAB_DIR"

echo "=========================================="
echo " SIMULACRO DE DESASTRE Y RECUPERACIÓN"
echo "=========================================="

# 1. Mostrar estado actual
echo ""
echo "[FASE 1] Estado actual de la base de datos"
echo "------------------------------------------"
docker compose exec db mysql -u root -prootpass libreria \
    -e "SELECT COUNT(*) AS total_libros FROM libros; \
        SELECT COUNT(*) AS total_ventas FROM ventas;" 2>/dev/null || \
    echo "  (no se pudo consultar - puede que la BD no esté disponible)"

# 2. Listar backups disponibles
echo ""
echo "[FASE 2] Backups disponibles"
echo "-----------------------------"
docker compose exec backup ls -lh /backups/*.sql.gz 2>/dev/null || \
    echo "  No hay backups aún. Espera al primer ciclo de cron."

LATEST_BACKUP=$(docker compose exec backup sh -c 'ls -t /backups/*.sql.gz 2>/dev/null | head -1' 2>/dev/null || echo "")

if [ -z "$LATEST_BACKUP" ]; then
    echo ""
    echo "ERROR: No hay backups para restaurar."
    echo "Ejecuta primero: docker compose exec backup /usr/local/bin/backup"
    exit 1
fi

echo ""
echo "Backup más reciente: $LATEST_BACKUP"

# 3. ¡DESASTRE! Eliminar el volumen
echo ""
echo "[FASE 3] SIMULANDO DESASTRE"
echo "----------------------------"
echo "Deteniendo servicios..."
docker compose stop db adminer

echo "Eliminando el volumen de datos de MySQL..."
docker compose down -v db 2>/dev/null || true
docker volume rm lab-volumenes_mysql_data 2>/dev/null || true

echo "Volumen eliminado. Los datos de MySQL han sido destruidos."
echo ""

# 4. Recrear el volumen y los servicios
echo "[FASE 4] Recreando infraestructura"
echo "----------------------------------"
docker volume create lab-volumenes_mysql_data
docker compose up -d db
echo ""

# 5. Esperar a que MySQL esté healthy
echo "[FASE 5] Esperando a que MySQL arranque..."
echo "------------------------------------------"
for i in $(seq 1 30); do
    if docker compose exec db mysqladmin ping -u root -prootpass --silent 2>/dev/null; then
        echo "MySQL está listo."
        break
    fi
    if [ "$i" -eq 30 ]; then
        echo "ERROR: MySQL no arrancó tras 60 segundos."
        exit 1
    fi
    sleep 2
done

# 6. Verificar que la BD está vacía (sin datos)
echo ""
echo "[FASE 6] Verificando que la BD está vacía..."
echo "--------------------------------------------"
TABLES_AFTER_DISASTER=$(docker compose exec db mysql -u root -prootpass \
    -N -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'libreria';" 2>/dev/null || echo "0")
echo "Tablas tras el desastre: $TABLES_AFTER_DISASTER"
echo ""

# 7. Restaurar desde backup
echo "[FASE 7] Restaurando desde backup más reciente..."
echo "--------------------------------------------------"
BACKUP_FILENAME=$(basename "$LATEST_BACKUP")

docker compose exec backup sh -c "
    gunzip -c /backups/$BACKUP_FILENAME | mysql -h db -u root -prootpass
"

echo "Restauración completada."
echo ""

# 8. Verificar recuperación
echo "[FASE 8] Verificando recuperación"
echo "---------------------------------"
TABLES_AFTER_RESTORE=$(docker compose exec db mysql -u root -prootpass \
    -N -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'libreria';")
BOOKS_AFTER_RESTORE=$(docker compose exec db mysql -u root -prootpass \
    -N -e "SELECT COUNT(*) FROM libreria.libros;")
SALES_AFTER_RESTORE=$(docker compose exec db mysql -u root -prootpass \
    -N -e "SELECT COUNT(*) FROM libreria.ventas;")

echo "  Tablas: $TABLES_AFTER_RESTORE"
echo "  Libros: $BOOKS_AFTER_RESTORE"
echo "  Ventas: $SALES_AFTER_RESTORE"

if [ "$BOOKS_AFTER_RESTORE" -ge 5 ] && [ "$TABLES_AFTER_RESTORE" -ge 2 ]; then
    echo ""
    echo "=========================================="
    echo " ✓ ¡RECUPERACIÓN EXITOSA!"
    echo "=========================================="
    echo "  Los datos fueron restaurados correctamente desde el backup."
    echo "  Tablas recuperadas: $TABLES_AFTER_RESTORE"
    echo "  Libros recuperados: $BOOKS_AFTER_RESTORE"
    echo "  Ventas recuperadas: $SALES_AFTER_RESTORE"
    echo ""
    echo "Muestra de datos recuperados:"
    docker compose exec db mysql -u root -prootpass libreria \
        -e "SELECT id, titulo, autor, precio FROM libros LIMIT 3;"
else
    echo ""
    echo "=========================================="
    echo " ✗ LA RECUPERACIÓN FALLÓ"
    echo "=========================================="
    echo "  Revisa los logs del sidecar de backup."
    exit 1
fi

# Reactivar adminer
docker compose up -d adminer
echo ""
echo "Adminer disponible en http://localhost:8080"
echo "  Servidor: db"
echo "  Usuario: root"
echo "  Contraseña: rootpass"
echo "  Base de datos: libreria"
```

### 5.11.10 Paso 8: Script para generar el primer backup manual

```bash
#!/bin/bash
# scripts/first-backup.sh
# Genera el primer backup manualmente (sin esperar al cron)

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
LAB_DIR="$(dirname "$SCRIPT_DIR")"
cd "$LAB_DIR"

echo "Ejecutando primer backup manual..."

docker compose exec backup /usr/local/bin/backup

echo ""
echo "Verificando que el backup se creó:"
docker compose exec backup ls -lh /backups/
```

### 5.11.11 Paso 9: Instrucciones de uso del laboratorio

```bash
# 1. Crear la estructura de directorios
mkdir -p lab-volumenes/{scripts,init-scripts}
cd lab-volumenes

# 2. Copiar todos los archivos mencionados arriba a sus ubicaciones:
#    - docker-compose.yml → lab-volumenes/
#    - Dockerfile.backup → lab-volumenes/
#    - crontab → lab-volumenes/
#    - init-scripts/01-create-db.sql → lab-volumenes/init-scripts/
#    - scripts/backup.sh → lab-volumenes/scripts/
#    - scripts/restore.sh → lab-volumenes/scripts/
#    - scripts/simulate-disaster.sh → lab-volumenes/scripts/
#    - scripts/first-backup.sh → lab-volumenes/scripts/

# 3. Dar permisos de ejecución
chmod +x scripts/*.sh

# 4. Iniciar todos los servicios
docker compose up -d

# 5. Verificar que todo arrancó
docker compose ps
# Deberías ver: db (healthy), backup (running), adminer (running)

# 6. Verificar los datos iniciales
docker compose exec db mysql -u root -prootpass libreria \
  -e "SELECT id, titulo, autor FROM libros;"

# 7. Generar el primer backup manual (sin esperar al cron)
./scripts/first-backup.sh

# 8. Listar los backups
docker compose exec backup ls -lh /backups/

# 9. Insertar más datos para verificar que el backup incremental funciona
docker compose exec db mysql -u root -prootpass libreria \
  -e "INSERT INTO libros (titulo, autor, isbn, precio, stock) \
      VALUES ('Clean Code', 'Robert C. Martin', '978-0132350884', 39.99, 50);"

# 10. Verificar que se insertó
docker compose exec db mysql -u root -prootpass libreria \
  -e "SELECT COUNT(*) as total FROM libros;"

# 11. SIMULACRO DE DESASTRE
./scripts/simulate-disaster.sh

# 12. Verificar que los datos se restauraron
docker compose exec db mysql -u root -prootpass libreria \
  -e "SELECT id, titulo, autor FROM libros;"
# ¡El libro 'Clean Code' debería aparecer si el último backup lo incluyó!

# 13. Acceder a Adminer para gestión visual
# Abrir http://localhost:8080
#   Sistema: MySQL
#   Servidor: db
#   Usuario: root
#   Contraseña: rootpass
#   Base de datos: libreria
```

### 5.11.12 Paso 10: NFS Volume para compartir backups entre múltiples hosts

Si tienes un servidor NFS disponible, puedes modificar el `docker-compose.yml` para que los backups se almacenen en NFS, accesibles desde múltiples hosts Docker:

```yaml
# Fragmento de docker-compose.yml con volumen NFS para backups
volumes:
  mysql_data:
    driver: local
  backups_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.50,nfsvers=4,rw,hard,intr
      device: ":/exports/docker-backups"
```

```bash
# En el servidor NFS, exportar el directorio
# /etc/exports
/exports/docker-backups  192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)

# Aplicar cambios en el servidor NFS
sudo exportfs -ra
sudo systemctl restart nfs-server

# En el host Docker, verificar conectividad
showmount -e 192.168.1.50
```

Con esto, los backups de todos tus hosts Docker se centralizan en un solo sistema de archivos NFS. Si un host muere, los backups sobreviven en el servidor NFS y puedes restaurarlos en otro host.

---

## Resumen del capítulo

En este capítulo has aprendido todo lo necesario sobre persistencia en Docker:

1. **El problema**: Los contenedores son efímeros por diseño. Sin almacenamiento externo, los datos mueren con el contenedor.
2. **Bind mounts**: Montan directorios del host dentro del contenedor. Ideales para desarrollo, peligrosos en producción. Sintaxis `-v` y `--mount`.
3. **Volumes**: El mecanismo nativo de Docker para persistencia. Gestionados por Docker, soportan múltiples drivers (local, NFS, cloud). Siempre usa volúmenes nombrados en producción.
4. **tmpfs mounts**: Almacenamiento en RAM. Para datos sensibles temporales que no deben tocar disco.
5. **Comparativa**: Tabla de decisión bind vs volume vs tmpfs según caso de uso.
6. **Permisos**: El problema del UID entre host y contenedor. Soluciones: `--user`, crear usuario en Dockerfile, gosu, userns-remap.
7. **Backup**: Contenedor temporal con `tar` para backup físico. `pg_dump`/`mysqldump` para backup lógico. Automatización con cron o sidecar.
8. **Migración**: Backup + transferir + restaurar. Pipe SSH para migración directa.
9. **Compose**: Volúmenes nombrados, bind mounts, tmpfs, driver options, volúmenes externos.
10. **Patrones**: Sidecar de backup, init container para inicialización, data-only container (obsoleto).
11. **Laboratorio**: Stack MySQL + sidecar de backup + simulación de desastre + recuperación real.

**Reglas de oro:**
- Usa volúmenes nombrados en producción, nunca anónimos.
- Monta en solo lectura siempre que sea posible (`:ro` o `readonly`).
- Haz backups regularmente y PRUÉBALOS con simulacros de restauración.
- Crea usuarios no-root en tus Dockerfiles (`USER appuser`).
- Usa `--mount` en lugar de `-v` en scripts para mayor claridad y seguridad.
- La estrategia 3-2-1 de backups: 3 copias, 2 medios diferentes, 1 fuera del sitio.

---
