# Apéndice A: Quick Reference y Cheat Sheets

> "Un buen ingeniero no memoriza comandos; sabe dónde encontrarlos cuando los necesita."

Este apéndice es tu referencia rápida para el día a día con Docker. Organizado por categorías, con los comandos que más usarás en desarrollo y producción.

---

## A.1 Comandos Esenciales por Categoría

### Contenedores

| Acción | Comando |
|--------|---------|
| Listar contenedores corriendo | `docker ps` |
| Listar todos (incluyendo parados) | `docker ps -a` |
| Listar solo IDs | `docker ps -q` |
| Crear y ejecutar | `docker run --name <nombre> -d -p 8080:80 <imagen>` |
| Ejecutar interactivo | `docker run -it <imagen> bash` |
| Ejecutar y eliminar al parar | `docker run --rm <imagen>` |
| Detener | `docker stop <contenedor>` |
| Detener con timeout | `docker stop -t 30 <contenedor>` |
| Iniciar contenedor parado | `docker start <contenedor>` |
| Reiniciar | `docker restart <contenedor>` |
| Pausar/Reanudar | `docker pause <c> / docker unpause <c>` |
| Eliminar | `docker rm <contenedor>` |
| Eliminar forzado | `docker rm -f <contenedor>` |
| Ejecutar comando dentro | `docker exec -it <contenedor> <comando>` |
| Ver logs | `docker logs <contenedor>` |
| Logs con follow | `docker logs -f <contenedor>` |
| Logs últimas N líneas | `docker logs --tail 100 <contenedor>` |
| Logs con timestamps | `docker logs -t <contenedor>` |
| Inspeccionar | `docker inspect <contenedor>` |
| Inspeccionar + jq | `docker inspect <c> \| jq '.[0].NetworkSettings.IPAddress'` |
| Ver procesos | `docker top <contenedor>` |
| Estadísticas en vivo | `docker stats` |
| Estadísticas sin stream | `docker stats --no-stream` |
| Copiar archivos host→cont | `docker cp ./archivo <c>:/ruta/` |
| Copiar cont→host | `docker cp <c>:/ruta/archivo ./` |
| Crear imagen desde contenedor | `docker commit <contenedor> <imagen>:<tag>` |
| Ver cambios en filesystem | `docker diff <contenedor>` |
| Ver eventos en tiempo real | `docker events` |
| Forzar kill (SIGKILL) | `docker kill <contenedor>` |
| Kill con señal específica | `docker kill -s SIGTERM <contenedor>` |

### Imágenes

| Acción | Comando |
|--------|---------|
| Listar imágenes | `docker images` |
| Descargar | `docker pull <imagen>:<tag>` |
| Descargar con digest | `docker pull <imagen>@sha256:abc123...` |
| Subir | `docker push <imagen>:<tag>` |
| Construir | `docker build -t <nombre>:<tag> .` |
| Construir sin caché | `docker build --no-cache -t <nombre> .` |
| Construir con BuildKit | `DOCKER_BUILDKIT=1 docker build -t <nombre> .` |
| Historial de capas | `docker history <imagen>` |
| Historial sin truncar | `docker history --no-trunc <imagen>` |
| Inspeccionar imagen | `docker inspect <imagen>` |
| Eliminar imagen | `docker rmi <imagen>` |
| Eliminar forzado | `docker rmi -f <imagen>` |
| Etiquetar | `docker tag <origen> <destino>:<tag>` |
| Guardar a tar | `docker save -o imagen.tar <imagen>` |
| Cargar desde tar | `docker load -i imagen.tar` |
| Exportar contenedor | `docker export <c> > contenedor.tar` |
| Importar como imagen | `docker import contenedor.tar <nombre>:<tag>` |
| Login a registry | `docker login <registry>` |
| Logout | `docker logout <registry>` |
| Buscar en Docker Hub | `docker search <término>` |
| Ver digest | `docker images --digests` |

### Redes

| Acción | Comando |
|--------|---------|
| Listar redes | `docker network ls` |
| Crear red bridge | `docker network create --driver bridge <nombre>` |
| Crear red overlay | `docker network create --driver overlay <nombre>` |
| Inspeccionar red | `docker network inspect <red>` |
| Conectar contenedor | `docker network connect <red> <contenedor>` |
| Desconectar | `docker network disconnect <red> <contenedor>` |
| Eliminar red | `docker network rm <red>` |
| Eliminar no usadas | `docker network prune` |
| Ver puertos expuestos | `docker port <contenedor>` |
| DNS interno: ping | `docker exec <c1> ping <c2>` |

### Volúmenes

| Acción | Comando |
|--------|---------|
| Listar volúmenes | `docker volume ls` |
| Crear | `docker volume create <nombre>` |
| Inspeccionar | `docker volume inspect <volumen>` |
| Eliminar | `docker volume rm <volumen>` |
| Eliminar no usados | `docker volume prune` |
| Backup de volumen | `docker run --rm -v <vol>:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz -C /data .` |
| Restaurar volumen | `docker run --rm -v <vol>:/data -v $(pwd):/backup alpine tar xzf /backup/backup.tar.gz -C /data` |

### Sistema y Limpieza

| Acción | Comando |
|--------|---------|
| Información del sistema | `docker info` |
| Versión | `docker version` |
| Uso de disco | `docker system df` |
| Uso de disco detallado | `docker system df -v` |
| Limpiar TODO no usado | `docker system prune -a` |
| Limpiar contenedores | `docker container prune` |
| Limpiar imágenes | `docker image prune -a` |
| Limpiar volúmenes | `docker volume prune` |
| Limpiar redes | `docker network prune` |
| Limpiar build cache | `docker builder prune` |

### Docker Compose

| Acción | Comando |
|--------|---------|
| Levantar (foreground) | `docker compose up` |
| Levantar (detached) | `docker compose up -d` |
| Reconstruir y levantar | `docker compose up -d --build` |
| Detener | `docker compose down` |
| Detener + eliminar volúmenes | `docker compose down -v` |
| Detener + eliminar imágenes | `docker compose down --rmi all` |
| Ver logs | `docker compose logs -f` |
| Ver logs de un servicio | `docker compose logs -f <servicio>` |
| Listar servicios | `docker compose ps` |
| Ejecutar comando | `docker compose exec <servicio> <cmd>` |
| Construir imágenes | `docker compose build` |
| Pull de imágenes | `docker compose pull` |
| Push de imágenes | `docker compose push` |
| Reiniciar servicio | `docker compose restart <servicio>` |
| Escalar servicio | `docker compose up -d --scale <servicio>=3` |

---

## A.2 Dockerfile Instructions — Quick Reference

| Instrucción | Propósito | Ejemplo |
|-------------|-----------|---------|
| `FROM` | Imagen base | `FROM node:20-alpine AS builder` |
| `RUN` | Ejecutar comando en build | `RUN apk add --no-cache curl` |
| `RUN --mount` | BuildKit: cache/secrets | `RUN --mount=type=cache,target=/root/.npm npm ci` |
| `COPY` | Copiar archivos (preferido) | `COPY package*.json ./` |
| `ADD` | Copiar + auto-extraer tar | `ADD app.tar.gz /app/` |
| `CMD` | Comando por defecto | `CMD ["node", "server.js"]` |
| `ENTRYPOINT` | Entrypoint del contenedor | `ENTRYPOINT ["docker-entrypoint.sh"]` |
| `CMD` + `ENTRYPOINT` | CMD como args de ENTRYPOINT | `CMD ["--config", "/etc/app.conf"]` |
| `WORKDIR` | Directorio de trabajo | `WORKDIR /app` |
| `ENV` | Variable de entorno (runtime) | `ENV NODE_ENV=production` |
| `ARG` | Variable de build | `ARG VERSION=1.0` |
| `EXPOSE` | Puerto (documentación) | `EXPOSE 3000` |
| `VOLUME` | Punto de montaje | `VOLUME /data` |
| `USER` | Usuario para ejecutar | `USER appuser` |
| `HEALTHCHECK` | Health check | `HEALTHCHECK CMD curl -f http://localhost/ || exit 1` |
| `SHELL` | Shell para RUN/CMD | `SHELL ["/bin/bash", "-c"]` |
| `STOPSIGNAL` | Señal de parada | `STOPSIGNAL SIGTERM` |
| `ONBUILD` | Trigger en builds hijos | `ONBUILD COPY . /app` |

### Shell Form vs Exec Form

```dockerfile
# ❌ Shell form: /bin/sh -c "node server.js"
# El proceso no recibe señales Unix (SIGTERM).
CMD node server.js

# ✅ Exec form: ejecuta node directamente.
# El proceso recibe señales Unix correctamente.
CMD ["node", "server.js"]
```

### Multi-Stage Build Pattern

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /bin/app

# Stage 2: Runtime
FROM scratch
COPY --from=builder /bin/app /app
ENTRYPOINT ["/app"]
```

---

## A.3 docker run — Flags Esenciales

| Flag | Significado | Ejemplo |
|------|-------------|---------|
| `-d` | Ejecutar en background (detached) | `docker run -d nginx` |
| `-it` | Interactivo + TTY | `docker run -it ubuntu bash` |
| `--rm` | Eliminar al parar | `docker run --rm alpine echo "hola"` |
| `--name` | Nombre del contenedor | `docker run --name web nginx` |
| `-p` | Publicar puerto (host:cont) | `docker run -p 8080:80 nginx` |
| `-P` | Publicar todos los EXPOSE | `docker run -P nginx` |
| `-v` | Montar volumen/bind | `docker run -v /host/path:/cont/path` |
| `--mount` | Montar (sintaxis moderna) | `docker run --mount type=bind,src=/host,dst=/cont` |
| `-e` | Variable de entorno | `docker run -e MYSQL_ROOT_PASSWORD=secret mysql` |
| `--env-file` | Archivo de variables | `docker run --env-file .env app` |
| `--network` | Conectar a red | `docker run --network mynet app` |
| `--restart` | Política de reinicio | `docker run --restart unless-stopped app` |
| `--memory` | Límite de memoria | `docker run --memory=256m app` |
| `--cpus` | Límite de CPU | `docker run --cpus=1.5 app` |
| `--cap-drop` | Quitar capability | `docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE` |
| `--read-only` | FS raíz solo lectura | `docker run --read-only --tmpfs /tmp app` |
| `--user` | Usuario/UID | `docker run --user 1000:1000 app` |
| `-w` | WORKDIR | `docker run -w /app node node server.js` |
| `--entrypoint` | Sobrescribir ENTRYPOINT | `docker run --entrypoint /bin/bash app` |

---

## A.4 Troubleshooting Común

### El contenedor no arranca

```bash
# 1. Ver logs
docker logs <contenedor>

# 2. Inspeccionar exit code
docker inspect <c> --format='{{.State.ExitCode}}'
# 0 = éxito, 1 = error app, 137 = OOM/SIGKILL, 143 = SIGTERM

# 3. Sobrescribir entrypoint para debug
docker run --rm -it --entrypoint /bin/sh <imagen>

# 4. Verificar health
docker inspect <c> --format='{{.State.Health.Status}}'

# 5. Si fue OOM killed
dmesg | grep -i "out of memory"
docker inspect <c> --format='{{.State.OOMKilled}}'
```

### No puedo conectar al contenedor

```bash
# 1. Verificar mapeo de puertos
docker port <contenedor>

# 2. Verificar que el proceso escucha en 0.0.0.0 (no 127.0.0.1)
docker exec <c> netstat -tlnp

# 3. Verificar conectividad entre contenedores
docker exec <c1> ping <c2>
docker exec <c1> nslookup <c2>

# 4. Verificar iptables
sudo iptables -L -n -t nat | grep DOCKER
```

### Disco lleno

```bash
# 1. Diagnosticar
docker system df
# TYPE           TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images         45        12        12.5GB    8.2GB (65%)
# Containers     60        5         1.8GB     1.2GB (66%)
# Local Volumes  20        8         3.4GB     900MB (26%)
# Build Cache    100       0         5.2GB     5.2GB (100%)

# 2. Limpiar
docker system prune -a --volumes

# 3. Ver qué ocupa más (imágenes)
docker images --format "{{.Size}}\t{{.Repository}}:{{.Tag}}" | sort -rh | head -10

# 4. Ver qué ocupa más (volúmenes)
docker volume ls -q | xargs docker volume inspect | jq '.[] | {Name: .Name, Mountpoint: .Mountpoint}' 
du -sh /var/lib/docker/volumes/*/_data | sort -rh | head -10
```

### Memoria / CPU elevada

```bash
# 1. Ver estadísticas en vivo
docker stats

# 2. Ver procesos dentro del contenedor
docker top <contenedor>

# 3. Limitar recursos
docker update --memory=512m --cpus=1.0 <contenedor>

# 4. Identificar contenedor problemático
docker stats --no-stream --format "{{.Name}}\t{{.MemPerc}}\t{{.CPUPerc}}" | sort -k2 -rh
```

### Imagen demasiado grande

```bash
# 1. Analizar capas
docker history <imagen> --no-trunc
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive <imagen>

# 2. Errores comunes:
#    - No usar .dockerignore (node_modules entran en el contexto)
#    - No limpiar caché de apt/yum/apk en el mismo RUN
#    - Usar imagen base default en vez de alpine/slim
#    - No usar multi-stage builds

# 3. Solución típica:
#    RUN apt-get update && apt-get install -y pkg && \
#        rm -rf /var/lib/apt/lists/*
```

### Build lento

```bash
# 1. Usar BuildKit
DOCKER_BUILDKIT=1 docker build -t app .

# 2. Orden de capas correcto (deps primero, código después)
# MAL: COPY . .    → invalida caché en cada cambio
# BIEN: COPY package*.json ./ → RUN npm ci → COPY src/ ./src/

# 3. Usar caché de BuildKit
RUN --mount=type=cache,target=/root/.npm npm ci

# 4. Cache remoto en CI
docker buildx build --cache-from=type=registry,ref=myimage:cache \
                    --cache-to=type=registry,ref=myimage:cache,mode=max .
```

---

## A.5 Señales Unix y Contenedores

| Señal | Número | Significado | Cuándo se usa |
|-------|--------|-------------|---------------|
| `SIGTERM` | 15 | Terminación educada | `docker stop` (por defecto). La app debe capturarla y hacer graceful shutdown. |
| `SIGKILL` | 9 | Terminación forzada | `docker kill`, o después del timeout de `docker stop` (10s). |
| `SIGINT` | 2 | Interrupción (Ctrl+C) | `docker stop` envía SIGTERM, no SIGINT. Solo en modo interactivo. |
| `SIGHUP` | 1 | Hang up | Recarga de configuración en Nginx, HAProxy, etc. |
| `SIGUSR1` | 10 | Definido por el usuario | Señal personalizada para acciones específicas. |
| `SIGUSR2` | 12 | Definido por el usuario | Señal personalizada (ej: activar debug mode). |
| `SIGSTOP` | 19 | Pausar (no capturable) | `docker pause` usa cgroups freezer, no señales. |
| `SIGCONT` | 18 | Reanudar | `docker unpause`. |

### El problema del Shell Form

```dockerfile
# ❌ Shell form: CMD /bin/sh -c "node server.js"
# Cuando docker stop envía SIGTERM → /bin/sh lo recibe, pero NO lo reenvía a node.
# Node no hace graceful shutdown → timeout 10s → SIGKILL.

# ✅ Exec form: CMD ["node", "server.js"]
# Cuando docker stop envía SIGTERM → node lo recibe directamente.
# Node puede cerrar conexiones, guardar estado, etc.
```

---

## A.6 Chmod Numérico (Referencia Rápida)

```
r = 4, w = 2, x = 1

chmod 755 = rwxr-xr-x  (directorio, binario ejecutable)
chmod 644 = rw-r--r--  (archivo normal)
chmod 600 = rw-------  (archivo sensible, clave SSH)
chmod 400 = r--------  (solo lectura, certificados)
chmod 700 = rwx------  (script privado)
chmod 777 = rwxrwxrwx  (⚠️ NUNCA en producción)
```

---

## A.7 Puertos por Defecto de Servicios Comunes

| Servicio | Puerto | Imagen Docker |
|----------|--------|---------------|
| HTTP (Nginx/Apache) | 80 | `nginx`, `httpd` |
| HTTPS | 443 | `nginx`, `traefik` |
| MySQL | 3306 | `mysql` |
| PostgreSQL | 5432 | `postgres` |
| MongoDB | 27017 | `mongo` |
| Redis | 6379 | `redis` |
| RabbitMQ | 5672, 15672 | `rabbitmq` |
| Elasticsearch | 9200 | `elasticsearch` |
| Prometheus | 9090 | `prom/prometheus` |
| Grafana | 3000 | `grafana/grafana` |
| Node Exporter | 9100 | `prom/node-exporter` |
| cAdvisor | 8080 | `gcr.io/cadvisor/cadvisor` |
| Registry (Docker) | 5000 | `registry` |
| Adminer | 8080 | `adminer` |
| phpMyAdmin | 80 | `phpmyadmin` |
| Jenkins | 8080, 50000 | `jenkins/jenkins` |
| GitLab | 80, 443, 22 | `gitlab/gitlab-ce` |
| MinIO | 9000, 9001 | `minio/minio` |

---

← [Capítulo anterior](capitulo-13-ejercicios.md) | [Inicio](README.md) | [Capítulo siguiente →](apendice-b-herramientas-modernas.md)
