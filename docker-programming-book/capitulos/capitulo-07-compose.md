# Capítulo 7: Docker Compose — Orquestación declarativa de contenedores

---

Docker Compose es la herramienta que transforma la gestión de contenedores de un ejercicio manual e imperativo a una práctica declarativa, reproducible y versionable. Si `docker run` es el martillo y el clavo, Docker Compose es el plano de construcción completo. En este capítulo exploraremos cada rincón de Compose: desde su anatomía YAML hasta despliegues complejos multi-servicio con redes segmentadas, health checks y perfiles condicionales.

---

## 7.1 ¿Por qué Docker Compose? — El salto de calidad

### 7.1.1 El problema: la pesadilla del `docker run` múltiple

Imagina que necesitas levantar una aplicación web típica: frontend en React, API en Node.js, base de datos PostgreSQL, Redis para caché y Nginx como proxy inverso. Con `docker run` individual, cada servicio requiere su propia invocación:

```bash
docker network create app-net

docker run -d --name db --network app-net \
  -e POSTGRES_USER=app -e POSTGRES_PASSWORD=s3cr3t -e POSTGRES_DB=appdb \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine

docker run -d --name redis --network app-net \
  redis:7-alpine

docker run -d --name api --network app-net \
  -e DATABASE_URL=postgresql://app:s3cr3t@db:5432/appdb \
  -e REDIS_URL=redis://redis:6379 \
  -p 3000:3000 \
  myapp/api:latest

docker run -d --name frontend --network app-net \
  -p 80:80 \
  myapp/frontend:latest
```

Esto funciona para 5 servicios. Ahora escala a 15, 20 o 30 microservicios. Los problemas se multiplican exponencialmente:

- **Órdenes de arranque**: ¿el API debe esperar a que PostgreSQL esté listo? `docker run` no ofrece control de dependencias.
- **Redes**: cada contenedor debe conectarse explícitamente a las redes correctas. Un error de tipeo y el servicio queda aislado.
- **Variables de entorno**: repetir `-e` decenas de veces es propenso a errores y duplicación.
- **Volúmenes**: debes recordar qué volúmenes creaste, con qué nombres, y si son persistentes o no.
- **Reproducibilidad**: ¿cómo compartes esta configuración con tu equipo? ¿Un script bash de 200 líneas? ¿Un README que se desactualiza al día siguiente?
- **Versionado**: sin un archivo declarativo, no hay trazabilidad de cambios en la topología de servicios.

La conclusión es inevitable: `docker run` es excelente para desarrollo rápido y experimentación, pero **no escala** como herramienta de orquestación.

### 7.1.2 Compose: declarativo, versionado, reproducible

Docker Compose resuelve todos estos problemas con un único archivo YAML que describe el estado deseado de tu aplicación:

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: s3cr3t
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend

  redis:
    image: redis:7-alpine
    networks:
      - backend

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgresql://app:s3cr3t@db:5432/appdb
      REDIS_URL: redis://redis:6379
    ports:
      - "3000:3000"
    networks:
      - backend
      - frontend
    depends_on:
      - db
      - redis

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "80:80"
    networks:
      - frontend
    depends_on:
      - api

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge

volumes:
  pgdata:
    driver: local
```

Las ventajas son inmediatas:

1. **Declarativo**: describes QUÉ quieres, no CÓMO lograrlo. Compose se encarga de crear redes, volúmenes y contenedores en el orden correcto.
2. **Versionado**: el archivo YAML vive en tu repositorio Git. Cada cambio en la topología tiene trazabilidad completa.
3. **Reproducible**: cualquier miembro del equipo ejecuta `docker compose up` y obtiene exactamente el mismo entorno.
4. **Autocontenido**: redes, volúmenes, variables de entorno, políticas de reinicio — todo en un solo lugar.
5. **Portable**: el mismo archivo funciona en desarrollo local, en el CI/CD y en producción (con ajustes mínimos mediante archivos `.env`).

### 7.1.3 Comparativa: imperativo vs declarativo

| Aspecto | `docker run` (Imperativo) | `docker-compose.yml` (Declarativo) |
|---|---|---|
| **Enfoque** | Dices paso a paso qué hacer | Describes el estado deseado |
| **Reproducibilidad** | Requiere scripts bash | Un solo archivo YAML |
| **Control de dependencias** | Manual (`sleep`, `wait-for-it`) | `depends_on` + health checks |
| **Redes** | Creación manual con `docker network create` | Definidas declarativamente en `networks:` |
| **Volúmenes** | Gestión manual | Declarativos, con drivers y opciones |
| **Variables de entorno** | Flags `-e` repetitivos | Archivos `.env` + interpolación |
| **Escalado** | Scripts complejos | `docker compose up --scale api=5` |
| **Versionado** | No existe | Git: diff, blame, historia completa |
| **CI/CD** | Scripts ad-hoc por pipeline | Mismo archivo Compose en todos lados |

El paradigma declarativo de Compose no es un lujo — es una necesidad cuando tu aplicación crece más allá de 3 contenedores.

---

## 7.2 Anatomía del `docker-compose.yml` — Claves raíz

Un archivo Compose tiene 5 claves raíz posibles. Cada una define un aspecto distinto de la topología:

```yaml
# Claves raíz del archivo Compose
services:   # Contenedores que componen la aplicación
networks:   # Redes personalizadas para comunicación entre servicios
volumes:    # Volúmenes nombrados para persistencia de datos
secrets:    # Secretos (solo Docker Swarm)
configs:    # Configuraciones (solo Docker Swarm)
```

### 7.2.1 `services:` — El corazón de Compose

`services` es la clave más importante. Define cada contenedor que forma parte de la aplicación. Cada entrada bajo `services` es un nombre de servicio, y su valor es un diccionario con la configuración del contenedor:

```yaml
services:
  web:           # Nombre del servicio
    image: nginx:alpine
    ports:
      - "8080:80"

  api:           # Otro servicio
    build: ./api
    environment:
      - NODE_ENV=production
    depends_on:
      - db

  db:            # Un tercer servicio
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data
```

Cada servicio se convierte en un contenedor (o en múltiples contenedores si se escala). Compose asigna nombres automáticamente usando el patrón `<proyecto>_<servicio>_<número>`.

### 7.2.2 `networks:` — Redes personalizadas

Por defecto, Compose crea una red bridge llamada `<proyecto>_default` y conecta todos los servicios a ella. Pero puedes definir redes adicionales con configuraciones específicas:

```yaml
networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

  backend:
    driver: bridge
    internal: true     # Sin acceso a internet
    ipam:
      config:
        - subnet: 172.29.0.0/16

  monitoring:
    external: true     # Red creada fuera de Compose
```

Las redes permiten segmentar la comunicación: el frontend no debería acceder directamente a la base de datos; solo el API debería tener acceso a la red `backend`.

### 7.2.3 `volumes:` — Volúmenes nombrados

Declarar volúmenes en el nivel raíz permite configurar su comportamiento (driver, opciones, etiquetas) y garantiza que Compose los cree automáticamente:

```yaml
volumes:
  db_data:
    driver: local
    driver_opts:
      type: none
      device: /mnt/data/postgres
      o: bind

  redis_data:
    driver: local

  shared_data:
    external: true
    name: proyecto_compartido_data
```

### 7.2.4 `secrets:` — Secretos para Swarm

`secrets` solo funciona en modo Swarm. Permite definir datos sensibles que se montan como archivos en `/run/secrets/<nombre>`:

```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
```

### 7.2.5 `configs:` — Configuraciones para Swarm

Similar a `secrets`, pero para datos no sensibles (archivos de configuración). También exclusivo de Swarm:

```yaml
configs:
  nginx_config:
    file: ./nginx.conf
  app_config:
    external: true
```

---

## 7.3 Services en profundidad — Cada directiva explicada

Esta sección desmenuza cada directiva disponible dentro de un servicio en Compose. Las directivas se presentan en orden alfabético para referencia rápida, con ejemplos concretos y funcionales.

### 7.3.1 `image` vs `build`

**`image`** especifica una imagen existente (de Docker Hub o un registro privado):

```yaml
services:
  web:
    image: nginx:1.25-alpine
  redis:
    image: redis:7.2-bookworm
  api:
    image: registry.miempresa.com/app/api:v2.3.1
```

**`build`** indica que la imagen debe construirse desde un Dockerfile:

```yaml
services:
  api:
    build: ./api                     # Sintaxis corta: ruta al contexto

  frontend:
    build:
      context: ./frontend            # Sintaxis larga
      dockerfile: Dockerfile.prod    # Dockerfile alternativo
      args:
        NODE_ENV: production
        API_URL: https://api.miapp.com
      cache_from:
        - myapp/frontend:latest
      labels:
        org.opencontainers.image.source: "https://github.com/miempresa/app"
      target: production              # Stage de multi-stage build
      network: host                   # Red durante el build
      shm_size: "256m"               # Tamaño de /dev/shm
```

Puedes combinar `build` e `image` para construir y etiquetar:

```yaml
services:
  api:
    build: ./api
    image: registry.miempresa.com/api:latest
```

### 7.3.2 `container_name`

Fuerza un nombre fijo para el contenedor, en lugar del nombre generado por Compose (`<proyecto>_<servicio>_<número>`):

```yaml
services:
  db:
    image: postgres:16
    container_name: postgres_principal
```

**Precaución**: con `container_name`, no puedes escalar el servicio (los nombres deben ser únicos). Úsalo solo cuando necesites que el nombre del contenedor sea predecible para scripts externos o monitoreo.

### 7.3.3 `command`

Sobrescribe el `CMD` definido en el Dockerfile:

```yaml
services:
  web:
    image: nginx:alpine
    command: ["nginx", "-g", "daemon off;"]

  worker:
    build: ./worker
    command: celery -A tasks worker --loglevel=info --concurrency=4
```

`command` puede ser una cadena (se ejecuta con `/bin/sh -c`) o un array (ejecución directa, sin shell). Prefiere el array cuando no necesites interpolación de shell:

```yaml
# Cadena: /bin/sh -c "bundle exec thin -p 3000"
command: bundle exec thin -p 3000

# Array: ejecución directa de bundle con argumentos
command: ["bundle", "exec", "thin", "-p", "3000"]
```

### 7.3.4 `entrypoint`

Sobrescribe el `ENTRYPOINT` del Dockerfile:

```yaml
services:
  debug:
    image: myapp:latest
    entrypoint: ["/bin/bash"]

  migrator:
    build: ./api
    entrypoint: ["python", "manage.py"]
    command: ["migrate"]
```

Puedes anular completamente el entrypoint o usar `--entrypoint` en la CLI:

```bash
docker compose run --entrypoint /bin/sh api
```

### 7.3.5 `ports` — Mapeo de puertos

Expone puertos del contenedor al host. Dos sintaxis:

**Sintaxis corta** — la más común:

```yaml
ports:
  - "3000"                # Solo puerto contenedor, host aleatorio
  - "8080:80"             # host:contenedor
  - "192.168.1.100:8080:80"  # ip:host:contenedor
```

**Sintaxis larga** — más control:

```yaml
ports:
  - target: 80              # Puerto en el contenedor
    published: 8080         # Puerto en el host
    protocol: tcp
    mode: host              # host vs ingress (Swarm)
```

- **`mode: host`**: el puerto se mapea solo en el nodo donde corre el contenedor.
- **`mode: ingress`** (por defecto en Swarm): el puerto se publica en todos los nodos del clúster, y el routing mesh dirige el tráfico.

**Rangos de puertos**:

```yaml
ports:
  - "8080-8090:80"   # 11 puertos host mapeados al 80 del contenedor
```

### 7.3.6 `expose`

Expone puertos a otros servicios en la misma red, pero NO al host:

```yaml
services:
  api:
    image: myapp/api
    expose:
      - "3000"
      - "9229"    # Debugger de Node.js
```

A diferencia de `ports`, `expose` no abre el puerto en el host. Los servicios en la misma red pueden alcanzar estos puertos. En la práctica, con redes Compose todos los puertos del contenedor son accesibles desde otros servicios en la misma red — `expose` sirve principalmente como documentación explícita.

### 7.3.7 `environment` — Variables de entorno

**Formato diccionario** (recomendado para legibilidad):

```yaml
services:
  api:
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://user:pass@db:5432/mydb
      REDIS_URL: redis://redis:6379
      LOG_LEVEL: debug
```

**Formato array** (útil cuando necesitas el signo `=` en el valor):

```yaml
services:
  api:
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - ESCAPED_VAR=valor=con=iguales
```

### 7.3.8 `env_file` — Archivos de variables de entorno

Carga variables desde uno o más archivos:

```yaml
services:
  api:
    env_file:
      - .env.api
      - .env.common
```

Formato del archivo `.env.api`:

```bash
# .env.api
NODE_ENV=production
DATABASE_URL=postgresql://user:pass@db:5432/mydb
REDIS_URL=redis://redis:6379
```

Si la misma variable se define en `environment` y `env_file`, gana `environment`. Si se define en múltiples archivos `env_file`, gana el último.

### 7.3.9 `volumes` — Montajes en el servicio

Define qué se monta dentro del contenedor. Referencia volúmenes nombrados o rutas del host.

**Sintaxis corta**:

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data       # Volumen nombrado
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # Bind mount (lectura)
      - /var/run/docker.sock:/var/run/docker.sock           # Bind mount
      - cache_data:/app/cache                  # Otro volumen nombrado
```

**Sintaxis larga** — más control:

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - type: volume
        source: db_data
        target: /var/lib/postgresql/data
        volume:
          nocopy: true      # No copiar contenido de la imagen al volumen

      - type: bind
        source: ./init.sql
        target: /docker-entrypoint-initdb.d/init.sql
        read_only: true

      - type: tmpfs
        target: /tmp/pg_tmp
        tmpfs:
          size: "128m"      # Tamaño máximo del tmpfs
          mode: 1777        # Permisos
```

La sintaxis larga permite configurar:
- `type`: `volume`, `bind`, `tmpfs`, o `npipe` (Windows).
- `source`: origen del montaje (ruta host para bind, nombre de volumen para volume).
- `target`: ruta dentro del contenedor.
- `read_only`: montaje de solo lectura.
- `volume.nocopy`: evita copiar datos preexistentes de la imagen al volumen.
- `tmpfs.size`: límite de tamaño para montajes en memoria.

### 7.3.10 `networks` — Conexión a redes

Conecta el servicio a una o más redes definidas en el nivel raíz `networks:`:

```yaml
services:
  api:
    networks:
      - frontend
      - backend

  db:
    networks:
      backend:
        aliases:
          - database
          - postgres
```

Los `aliases` son nombres DNS alternativos para el servicio dentro de esa red. Otros contenedores pueden resolver `database` o `postgres` para llegar a `db`:

```yaml
services:
  api:
    networks:
      backend:
        aliases:
          - api-svc
        ipv4_address: 172.28.5.10
        ipv6_address: 2001:db8::10
        priority: 100
        link_local_ips:
          - 169.254.10.10
```

### 7.3.11 `depends_on` — Dependencias de arranque

Controla el orden de inicio de los servicios:

```yaml
services:
  web:
    build: .
    depends_on:
      - db
      - redis
      - elasticsearch
```

**Comportamiento**:
1. Compose arranca los servicios en orden de dependencia: primero `db`, `redis` y `elasticsearch`; luego `web`.
2. Al detener, se invierte el orden: primero `web`, luego los demás.

**Limitación clásica**: `depends_on` solo espera a que el contenedor esté *iniciado*, no a que la aplicación dentro del contenedor esté *lista*. PostgreSQL puede estar aceptando conexiones TCP pero todavía inicializando la base de datos.

**Solución con `condition`** (disponible en Compose V2 con health checks):

```yaml
services:
  web:
    build: .
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

- `service_started`: espera a que el contenedor arranque.
- `service_healthy`: espera a que el health check pase (requiere `healthcheck` en el servicio dependido).
- `service_completed_successfully`: espera a que el contenedor termine con código 0 (útil para migraciones).

### 7.3.12 `restart` — Política de reinicio

```yaml
services:
  api:
    image: myapp/api
    restart: unless-stopped

  db:
    image: postgres:16
    restart: always
```

Valores posibles:
- `no`: no reiniciar (valor por defecto).
- `always`: reiniciar siempre, incluso si se detuvo manualmente.
- `on-failure`: reiniciar solo si el contenedor termina con código de error.
- `unless-stopped`: reiniciar siempre, a menos que se detenga explícitamente (con `docker compose stop` o `docker stop`). Es la opción más recomendada para producción.

### 7.3.13 `profiles` — Servicios condicionales

Los perfiles permiten definir servicios que solo se ejecutan bajo demanda:

```yaml
services:
  api:
    image: myapp/api
    # Sin profiles: siempre se ejecuta

  debug-tool:
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
    profiles:
      - debug

  adminer:
    image: adminer
    ports:
      - "8080:8080"
    profiles:
      - debug
      - tools
```

Para arrancar con perfiles:

```bash
# Método 1: flag --profile
docker compose --profile debug up

# Método 2: variable de entorno
COMPOSE_PROFILES=debug,tools docker compose up

# Múltiples perfiles
docker compose --profile debug --profile monitoring up
```

### 7.3.14 `healthcheck` — Verificación de salud

Define cómo Compose verifica que el servicio está funcionando correctamente:

```yaml
services:
  api:
    build: ./api
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
      start_interval: 5s

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

Parámetros:
- `test`: comando que verifica la salud. `CMD` para ejecución directa, `CMD-SHELL` para pasar por shell.
- `interval`: tiempo entre chequeos.
- `timeout`: tiempo máximo que puede tardar el test.
- `retries`: fallos consecutivos para marcar como `unhealthy`.
- `start_period`: tiempo de gracia inicial antes de contar fallos.
- `start_interval`: intervalo entre chequeos durante `start_period`.

**Nota sobre el escape de `$`**: en Compose, `$$` escapa el signo `$` para que no sea tratado como interpolación de variables. `$${VAR}` se convierte en `${VAR}` literal en el contenedor.

### 7.3.15 `logging` — Configuración de logging

Controla cómo se registran los logs del servicio:

```yaml
services:
  api:
    image: myapp/api
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
        tag: "api-{{.Name}}-{{.ID}}"

  fluentd:
    image: fluent/fluentd
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: "docker.{{.Name}}"
        fluentd-async: "true"

  loki:
    image: grafana/loki
    logging:
      driver: loki
      options:
        loki-url: "http://localhost:3100/loki/api/v1/push"
        loki-batch-size: "400"
```

El driver por defecto es `json-file`. Otros drivers disponibles: `syslog`, `journald`, `gelf`, `fluentd`, `awslogs`, `splunk`, `gcplogs`, `loki`, `etwlogs` (Windows).

### 7.3.16 `labels` — Metadatos

Añade metadatos al contenedor en forma de etiquetas:

```yaml
services:
  api:
    image: myapp/api
    labels:
      com.miempresa.departamento: "ingenieria"
      com.miempresa.version: "2.4.1"
      com.miempresa.entorno: "produccion"
      traefik.enable: "true"
      traefik.http.routers.api.rule: "Host(`api.miapp.com`)"
```

Las labels se pueden consultar con `docker inspect` y son ampliamente utilizadas por proxies inversos como Traefik y Caddy para descubrimiento dinámico:

```bash
docker inspect api --format '{{ json .Config.Labels }}' | jq
```

### 7.3.17 `extra_hosts` — Entradas en `/etc/hosts`

```yaml
services:
  api:
    image: myapp/api
    extra_hosts:
      - "host.internal:host-gateway"
      - "auth.miempresa.local:10.0.0.50"
      - "legacy-db.internal:192.168.1.100"
```

`host-gateway` es un valor especial que se resuelve a la IP del host Docker. Útil para acceder a servicios corriendo en el host desde dentro del contenedor.

### 7.3.18 `ulimits` — Límites de recursos del sistema operativo

```yaml
services:
  db:
    image: postgres:16
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
      nproc:
        soft: 4096
        hard: 8192
      memlock:
        soft: -1
        hard: -1
```

### 7.3.19 `sysctls` — Parámetros del kernel

```yaml
services:
  db:
    image: postgres:16
    sysctls:
      net.core.somaxconn: 1024
      net.ipv4.tcp_keepalive_time: 600
      kernel.shmmax: 2147483648
```

### 7.3.20 `cap_add` y `cap_drop` — Capacidades Linux

```yaml
services:
  api:
    image: myapp/api
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      - SYS_PTRACE
```

- `cap_drop: [ALL]`: elimina todas las capacidades (máxima seguridad).
- `cap_add`: añade solo las necesarias.
- Principio de mínimo privilegio: otorga solo lo indispensable.

### 7.3.21 `security_opt` — Opciones de seguridad

```yaml
services:
  api:
    image: myapp/api
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-default
      - seccomp:unconfined
```

### 7.3.22 `dns` y `dns_search` — Configuración DNS

```yaml
services:
  api:
    image: myapp/api
    dns:
      - 8.8.8.8
      - 1.1.1.1
    dns_search:
      - miempresa.local
      - svc.cluster.local
```

### 7.3.23 `init` — Proceso init (tini)

```yaml
services:
  api:
    image: myapp/api
    init: true
```

Inserta `docker-init` (tini) como PID 1. Maneja procesos zombie y reenvía señales correctamente. Esencial si tu aplicación principal no está diseñada para ser PID 1:

```bash
# Equivalente a:
docker run --init myapp/api
```

### 7.3.24 `stop_grace_period`

Tiempo que Compose espera entre enviar SIGTERM y SIGKILL al detener un servicio:

```yaml
services:
  api:
    image: myapp/api
    stop_grace_period: 30s

  worker:
    image: myapp/worker
    stop_grace_period: 2m
```

El valor por defecto es 10 segundos. Auméntalo si tu aplicación necesita más tiempo para terminar conexiones activas o finalizar tareas en curso.

### 7.3.25 `deploy` — Configuración para Swarm

Las opciones bajo `deploy` solo se aplican cuando se usa `docker stack deploy` en modo Swarm:

```yaml
services:
  api:
    image: myapp/api:v2
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.role == worker
          - node.labels.region == us-east
        preferences:
          - spread: node.labels.az
        max_replicas_per_node: 2
      resources:
        limits:
          cpus: "0.50"
          memory: 256M
          pids: 100
        reservations:
          cpus: "0.25"
          memory: 128M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        monitor: 30s
        max_failure_ratio: 0.3
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 5s
        failure_action: pause
        monitor: 30s
        max_failure_ratio: 0.3
      labels:
        com.miempresa.tier: "backend"
```

Parámetros clave:
- **`mode`**: `replicated` (réplicas distribuidas) o `global` (una por nodo).
- **`replicas`**: número de instancias.
- **`placement.constraints`**: limitar en qué nodos se ejecuta.
- **`placement.preferences`**: distribuir réplicas según criterios (zonas de disponibilidad, racks).
- **`resources.limits`**: límites máximos de CPU/memoria.
- **`resources.reservations`**: reservas mínimas garantizadas.
- **`restart_policy`**: política de reinicio en Swarm (diferente de `restart:`).
- **`update_config`**: cómo se actualiza el servicio (rolling updates).
  - `parallelism`: cuántas réplicas actualizar a la vez.
  - `delay`: tiempo entre lotes de actualización.
  - `failure_action`: `pause`, `continue`, `rollback`.
  - `order`: `start-first` (nuevo antes de matar viejo) o `stop-first` (matar viejo antes de iniciar nuevo).
- **`rollback_config`**: cómo se revierte una actualización fallida.

---
## 7.4 Configuración de `build` — Construcción de imágenes desde Compose

Cuando usas `build` en un servicio, Compose llama internamente a `docker build`. La directiva `build` admite una sintaxis corta y una larga extremadamente detallada:

### 7.4.1 Sintaxis corta

```yaml
services:
  api:
    build: ./api              # Dockerfile en ./api/Dockerfile

  worker:
    build: .                  # Dockerfile en ./Dockerfile (directorio actual)
```

### 7.4.2 Sintaxis larga completa

```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.prod
      dockerfile_inline: |
        # Comandos Dockerfile en línea (Compose V2.17+)
      args:
        - NODE_VERSION=20
        - BUILD_ENV=production
        - API_BASE_URL
      cache_from:
        - type=registry,ref=registry.miempresa.com/frontend:cache
        - type=local,src=/tmp/.buildx-cache
      cache_to:
        - type=registry,ref=registry.miempresa.com/frontend:cache,mode=max
      ssh:
        - default
        - id_custom=~/.ssh/custom_key
      platforms:
        - linux/amd64
        - linux/arm64
      labels:
        org.opencontainers.image.title: "Frontend App"
        org.opencontainers.image.version: "2.4.1"
        org.opencontainers.image.authors: "dev@miempresa.com"
      target: production
      network: host
      shm_size: "512m"
      no_cache: false
      pull: true
      additional_contexts:
        shared: ../shared
        config: /etc/app-config
      secrets:
        - id=npm_token
        - source: ./secrets/npmrc
          target: /root/.npmrc
      tags:
        - "registry.miempresa.com/frontend:latest"
        - "registry.miempresa.com/frontend:2.4.1"
```

### 7.4.3 `args` — Argumentos de build

Pasados al Dockerfile como `ARG`:

```dockerfile
# Dockerfile
ARG NODE_VERSION=18
ARG BUILD_ENV=development
ARG API_BASE_URL

FROM node:${NODE_VERSION}-alpine
ARG BUILD_ENV
ARG API_BASE_URL
ENV NODE_ENV=${BUILD_ENV}
ENV API_URL=${API_BASE_URL:-http://localhost:3000}
```

Los `args` pueden definirse sin valor en el compose file — en ese caso, toman el valor de la variable de entorno del shell:

```yaml
build:
  args:
    - CI           # Toma el valor de $CI del entorno
    - NODE_ENV     # Toma el valor de $NODE_ENV del entorno
```

### 7.4.4 `cache_from` — Fuentes de caché

Acelera builds usando caché remota o imágenes existentes:

```yaml
build:
  context: .
  cache_from:
    - myapp:latest
    - registry.miempresa.com/myapp:buildcache
    - type=registry,ref=registry.miempresa.com/myapp:buildcache
    - type=local,src=/tmp/build-cache
```

### 7.4.5 `labels` — Etiquetas OCI

Metadatos estándar incrustados en la imagen resultante:

```yaml
build:
  context: .
  labels:
    org.opencontainers.image.source: "https://github.com/miempresa/app"
    org.opencontainers.image.revision: "${GIT_COMMIT}"
    org.opencontainers.image.created: "${BUILD_DATE}"
```

### 7.4.6 `target` — Stage de multi-stage build

```dockerfile
# Dockerfile multi-stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --production
EXPOSE 3000
CMD ["node", "dist/server.js"]

FROM node:20-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["npm", "run", "dev"]
```

```yaml
# Producción
services:
  api-prod:
    build:
      context: .
      target: production

# Desarrollo
services:
  api-dev:
    build:
      context: .
      target: development
```

### 7.4.7 `network` — Red durante el build

```yaml
build:
  context: .
  network: host     # Usa la red del host durante RUN en el build
```

Valores: `bridge` (por defecto), `host`, `none`, o cualquier red existente.

### 7.4.8 `shm_size` — Tamaño de /dev/shm

```yaml
build:
  context: .
  shm_size: "1gb"
```

Necesario para aplicaciones que usan `/dev/shm` intensivamente (bases de datos, testing con muchos hilos).

---

## 7.5 Redes en Compose — A fondo

La comunicación entre contenedores es uno de los pilares de cualquier aplicación distribuida. Compose ofrece un modelo de redes flexible y potente.

### 7.5.1 La red por defecto

Si no defines redes explícitamente, Compose crea una red bridge llamada `<nombre_directorio>_default`. Todos los servicios se conectan a ella automáticamente:

```yaml
# Sin definición de redes: Compose crea mi-proyecto_default
services:
  web:
    image: nginx
  api:
    build: ./api
  db:
    image: postgres:16
```

Estos servicios pueden comunicarse usando los nombres de servicio como hostnames: `web` resuelve al contenedor web, `api` al contenedor api, etc.

```bash
# Desde el contenedor 'web', ping a 'api' funciona
docker compose exec web ping api
# PING api (172.18.0.3): 56 data bytes
```

### 7.5.2 Redes definidas por el usuario

Define redes personalizadas en el nivel raíz `networks:`:

```yaml
services:
  frontend:
    image: nginx
    networks:
      - front-tier

  api:
    build: ./api
    networks:
      - front-tier
      - back-tier

  db:
    image: postgres:16
    networks:
      - back-tier

  redis:
    image: redis:7-alpine
    networks:
      - back-tier

networks:
  front-tier:
    driver: bridge
  back-tier:
    driver: bridge
```

Topología resultante:

```
[frontend] --- front-tier --- [api] --- back-tier --- [db]
                                   +--- back-tier --- [redis]
```

- `frontend` solo puede comunicarse con `api` (a través de `front-tier`).
- `api` puede comunicarse con `frontend`, `db` y `redis`.
- `db` y `redis` están aislados en `back-tier`, inaccesibles desde fuera.

### 7.5.3 Drivers de red

```yaml
networks:
  default_bridge:
    driver: bridge         # Red bridge estándar (por defecto)

  isolated:
    driver: bridge
    internal: true         # Sin acceso a internet; solo comunicación interna

  host_net:
    driver: host           # Usa la red del host directamente

  no_net:
    driver: none           # Contenedor sin red (aislado)

  overlay:
    driver: overlay        # Red multi-host (requiere Swarm)
    attachable: true       # Permite conectar contenedores no-Swarm
```

### 7.5.4 `driver_opts` — Opciones del driver

```yaml
networks:
  custom_bridge:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: "br-custom"
      com.docker.network.bridge.enable_icc: "true"
      com.docker.network.bridge.enable_ip_masquerade: "true"
      com.docker.network.bridge.host_binding_ipv4: "0.0.0.0"
      com.docker.network.driver.mtu: "1500"
```

### 7.5.5 IPAM — Gestión de direcciones IP

```yaml
networks:
  backend:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
          ip_range: 172.28.5.0/24
          gateway: 172.28.5.254
          aux_addresses:
            dns_server: 172.28.5.10
            ntp_server: 172.28.5.11
```

Permite definir subredes personalizadas y reservar direcciones para servicios auxiliares.

### 7.5.6 Redes externas

Una red creada fuera de Compose (manualmente o por otro archivo Compose):

```yaml
networks:
  shared_network:
    external: true

  specific_network:
    external: true
    name: produccion_monitoring
```

Para reutilizar una red existente:

```bash
docker network create app-shared
```

```yaml
# archivo-1.yml
services:
  api:
    networks:
      - shared

networks:
  shared:
    external: true
    name: app-shared
```

### 7.5.7 Redes compartidas entre múltiples archivos Compose

Este patrón es común en microservicios donde cada equipo mantiene su propio archivo Compose pero los servicios necesitan comunicarse:

```bash
# Equipo A: API + DB
docker compose -f api-stack.yml up -d

# Equipo B: Workers + Redis
docker compose -f worker-stack.yml up -d

# La red app-shared conecta ambos stacks
```

```yaml
# api-stack.yml
services:
  api:
    networks:
      - shared
      - internal
  db:
    networks:
      - internal

networks:
  shared:
    external: true
    name: app-shared
  internal:
    driver: bridge
```

```yaml
# worker-stack.yml
services:
  worker:
    networks:
      - shared
  redis:
    networks:
      - shared

networks:
  shared:
    external: true
    name: app-shared
```

---

## 7.6 Volúmenes en Compose — Persistencia de datos

### 7.6.1 Declaración de volúmenes

Los volúmenes se declaran en dos lugares:
1. En el servicio (`services.<nombre>.volumes`) — qué se monta y dónde.
2. En el nivel raíz (`volumes:`) — configuración del volumen.

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data
      - db_logs:/var/log/postgresql
      - ./backups:/backups:ro

volumes:
  db_data:
    driver: local
    driver_opts:
      type: none
      device: /mnt/fast-ssd/postgres
      o: bind
    labels:
      com.miempresa.backup: "diario"
  db_logs:
    driver: local
```

### 7.6.2 Named volumes con configuración completa

```yaml
volumes:
  data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,nolock,soft,rw
      device: ":/exports/data"

  encrypted_data:
    driver: local
    driver_opts:
      type: btrfs
      device: /dev/sdb1

  cloud_backup:
    driver: rexray/s3fs
    driver_opts:
      accesskey: "${AWS_ACCESS_KEY}"
      secretkey: "${AWS_SECRET_KEY}"
      region: us-east-1
      bucket: my-docker-volumes
```

### 7.6.3 Volúmenes externos

Volúmenes creados fuera de Compose que quieres reutilizar:

```yaml
volumes:
  existing_volume:
    external: true

  specific_volume:
    external: true
    name: proyecto_existente_db_data
```

Útil para migraciones graduales o para compartir datos entre diferentes proyectos Compose.

### 7.6.4 Estrategia de volúmenes para producción

```yaml
services:
  postgres:
    image: postgres:16-alpine
    volumes:
      - pg_data:/var/lib/postgresql/data:rw
      - pg_backups:/backups:rw
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
    environment:
      PGDATA: /var/lib/postgresql/data/pgdata

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  elasticsearch:
    image: elasticsearch:8.11.0
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ulimits:
      nofile:
        soft: 65536
        hard: 65536

volumes:
  pg_data:
    driver: local
  pg_backups:
    driver: local
  redis_data:
    driver: local
  es_data:
    driver: local
```

---

## 7.7 Variables de entorno — Precedencia e interpolación

El manejo de variables de entorno en Compose es uno de los aspectos más potentes y, a la vez, fuente de confusión si no se entiende el orden de precedencia.

### 7.7.1 El orden de precedencia completo

Cuando una variable está definida en múltiples lugares, el orden de precedencia (de mayor a menor) es:

1. **Variables en `environment:` del compose file** (mayor prioridad)
2. **Variables en `env_file:`** (dentro del servicio)
3. **Variables del shell** (heredadas del entorno donde se ejecuta `docker compose`)
4. **Archivo `.env`** en el directorio del proyecto (menor prioridad)

### 7.7.2 Demostración de cada nivel

**Archivo `.env` en el directorio del proyecto:**

```bash
# .env
DATABASE_URL=postgresql://user:pass@prod-db:5432/proddb
LOG_LEVEL=info
MAX_WORKERS=4
```

**Variables del shell:**

```bash
export DATABASE_URL=postgresql://user:pass@staging-db:5432/stagingdb
export LOG_LEVEL=debug
```

**`env_file` en el servicio:**

```bash
# ./env/api.env
DATABASE_URL=postgresql://user:pass@dev-db:5432/devdb
APP_NAME=MiAPI
```

**`environment` en el compose file:**

```yaml
services:
  api:
    image: myapp/api
    env_file:
      - ./env/api.env
    environment:
      DATABASE_URL: postgresql://user:pass@localhost:5432/testdb
```

**Resultado final**: el contenedor recibe `DATABASE_URL=postgresql://user:pass@localhost:5432/testdb` porque `environment` tiene la máxima prioridad.

**Prioridad demostrada en tabla:**

| Fuente | DATABASE_URL | LOG_LEVEL | APP_NAME | MAX_WORKERS |
|---|---|---|---|---|
| `.env` (prioridad 4) | `...@prod-db...` | `info` | — | `4` |
| Shell (prioridad 3) | `...@staging-db...` | `debug` | — | — |
| `env_file` (prioridad 2) | `...@dev-db...` | — | `MiAPI` | — |
| `environment` (prioridad 1) | `...@localhost...` | — | — | — |
| **Resultado final** | `...@localhost...` | `debug` | `MiAPI` | `4` |

### 7.7.3 Interpolación de variables

Compose soporta sintaxis de shell para interpolar variables:

```yaml
services:
  api:
    image: "${DOCKER_REGISTRY:-registry.miempresa.com}/api:${API_VERSION:-latest}"
    environment:
      APP_ENV: "${ENVIRONMENT:-development}"
      DATABASE_URL: "postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}"
      OPTIONAL_FEATURE: "${FEATURE_FLAG:-false}"

  db:
    image: "postgres:${POSTGRES_VERSION:-16}-alpine"
    environment:
      POSTGRES_USER: "${DB_USER:-app}"
      POSTGRES_PASSWORD: "${DB_PASSWORD:?DB_PASSWORD es obligatoria}"
      POSTGRES_DB: "${DB_NAME:-appdb}"
```

Sintaxis de interpolación:
- **`${VARIABLE}`**: usa el valor de `VARIABLE`. Error si no existe y no tiene default.
- **`${VARIABLE:-default}`**: usa `VARIABLE` si existe; si no, usa `default`.
- **`${VARIABLE:?mensaje}`**: usa `VARIABLE` si existe; si no, error con `mensaje`.
- **`${VARIABLE:+alternativa}`**: usa `alternativa` si `VARIABLE` existe; si no, vacío.
- **`${VARIABLE:offset:length}`**: substring (requiere Compose V2.22+).

### 7.7.4 Uso de `.env` para diferentes entornos

El archivo `.env` es leído automáticamente por Compose desde el directorio del proyecto. Puedes tener múltiples archivos para distintos entornos:

```bash
# Estructura de archivos
.
├── docker-compose.yml
├── .env              # Valores por defecto
├── .env.dev          # Sobrescrituras para desarrollo
├── .env.staging      # Sobrescrituras para staging
├── .env.prod         # Sobrescrituras para producción
└── .env.ci           # Sobrescrituras para CI
```

```bash
# .env (valores por defecto)
COMPOSE_PROJECT_NAME=miapp
POSTGRES_VERSION=16
NODE_VERSION=20
API_PORT=3000
```

```bash
# .env.dev
ENVIRONMENT=development
API_PORT=3000
LOG_LEVEL=debug
DB_USER=devuser
DB_PASSWORD=devpass
DB_NAME=devdb
COMPOSE_PROFILES=debug,tools
```

```bash
# .env.prod
ENVIRONMENT=production
API_PORT=3000
LOG_LEVEL=warn
DB_USER=prod_user
DB_PASSWORD=
DB_NAME=proddb
DOCKER_REGISTRY=registry.miempresa.com
API_VERSION=2.4.1
```

**Uso práctico con `--env-file`:**

```bash
# Desarrollo
docker compose --env-file .env.dev up -d

# Staging
docker compose --env-file .env.staging up -d

# Producción
docker compose --env-file .env.prod up -d

# CI
docker compose --env-file .env.ci up --exit-code-from test
```

**`docker-compose.yml` parametrizado:**

```yaml
services:
  api:
    image: "${DOCKER_REGISTRY:-localhost}/api:${API_VERSION:-latest}"
    build:
      context: ./api
      args:
        NODE_ENV: "${ENVIRONMENT:-production}"
    ports:
      - "${API_PORT:-3000}:3000"
    environment:
      NODE_ENV: "${ENVIRONMENT:-production}"
      LOG_LEVEL: "${LOG_LEVEL:-info}"
      DATABASE_URL: "postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
```

**Variables obligatorias con validación:**

```yaml
services:
  api:
    environment:
      DATABASE_PASSWORD: "${DB_PASSWORD:?ERROR: DB_PASSWORD debe estar definida en .env o en el entorno}"
      SECRET_KEY: "${SECRET_KEY:?ERROR: SECRET_KEY es obligatoria para producción}"
```

### 7.7.5 Variables de entorno especiales de Compose

Algunas variables de entorno afectan el comportamiento del propio Compose:

| Variable | Descripción | Ejemplo |
|---|---|---|
| `COMPOSE_PROJECT_NAME` | Nombre del proyecto (prefijo de recursos) | `miapp` |
| `COMPOSE_FILE` | Archivo(s) Compose a usar | `docker-compose.yml:docker-compose.dev.yml` |
| `COMPOSE_PROFILES` | Perfiles activos | `debug,monitoring` |
| `COMPOSE_HTTP_TIMEOUT` | Timeout para peticiones HTTP | `120` |
| `COMPOSE_PARALLEL_LIMIT` | Límite de operaciones paralelas | `5` |
| `DOCKER_HOST` | Daemon de Docker remoto | `ssh://user@server` |
| `DOCKER_TLS_VERIFY` | Verificar TLS | `1` |
| `DOCKER_CERT_PATH` | Ruta a certificados TLS | `/home/user/.docker` |
| `COMPOSE_DOCKER_CLI_BUILD` | Usar `docker build` en lugar de buildkit interno | `1` |
| `DOCKER_BUILDKIT` | Habilitar BuildKit | `1` |
| `BUILDKIT_PROGRESS` | Estilo de progreso de build | `plain`, `auto`, `tty` |
| `COMPOSE_IGNORE_ORPHANS` | No advertir sobre servicios huérfanos | `true` |

---

## 7.8 Profiles — Servicios condicionales

Los perfiles resuelven el problema de "quiero este servicio solo a veces". En lugar de mantener múltiples archivos Compose casi idénticos, defines servicios opcionales que se activan con un perfil.

### 7.8.1 Definición de perfiles

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    # Siempre se ejecuta (no tiene profiles)

  db:
    image: postgres:16-alpine
    # Siempre se ejecuta

  adminer:
    image: adminer:latest
    ports:
      - "8080:8080"
    environment:
      ADMINER_DEFAULT_SERVER: db
    profiles:
      - debug
      - tools

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
    profiles:
      - debug
      - email

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    profiles:
      - monitoring

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    profiles:
      - monitoring
```

### 7.8.2 Activación de perfiles

```bash
# Activar un perfil
docker compose --profile debug up -d

# Activar múltiples perfiles
docker compose --profile debug --profile monitoring up -d

# Variable de entorno (útil para CI o Makefiles)
COMPOSE_PROFILES=debug,monitoring docker compose up -d

# En .env
COMPOSE_PROFILES=debug
docker compose up -d
```

### 7.8.3 Combinación con `--env-file`

```bash
# .env.dev
COMPOSE_PROFILES=debug,tools,email

# Ejecutar
docker compose --env-file .env.dev up -d
```

### 7.8.4 Casos de uso reales

**Herramientas de desarrollo:**

```yaml
services:
  # Servicios base (sin profiles)
  app:
    build: .
    volumes:
      - .:/app
    ports:
      - "3000:3000"

  db:
    image: postgres:16-alpine

  # Servicios condicionales
  adminer:         # Interfaz web para la BD
    image: adminer
    ports:
      - "8080:8080"
    profiles:
      - tools

  mailpit:         # Captura de emails
    image: axllent/mailpit
    ports:
      - "8025:8025"
      - "1025:1025"
    profiles:
      - email

  phpmyadmin:      # Alternativa a Adminer
    image: phpmyadmin
    ports:
      - "8081:80"
    profiles:
      - tools
```

**Herramientas de depuración:**

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
      - "9229:9229"   # Debugger Node.js

  debug-shell:
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
    network_mode: service:app
    profiles:
      - debug

  wireshark:
    image: lscr.io/linuxserver/wireshark
    cap_add:
      - NET_ADMIN
    network_mode: host
    profiles:
      - debug
```

**Monitoreo:**

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
      - "9091:9091"   # Métricas

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    profiles:
      - monitoring

  grafana:
    image: grafana/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    profiles:
      - monitoring

  node_exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"
    profiles:
      - monitoring

  loki:
    image: grafana/loki
    ports:
      - "3100:3100"
    profiles:
      - monitoring

volumes:
  grafana_data:
```

---
## 7.9 Extends y múltiples archivos Compose

Compose permite componer configuraciones desde múltiples archivos YAML, lo que facilita la gestión de variantes por entorno sin duplicar código.

### 7.9.1 Múltiples archivos con `-f`

La técnica moderna (Compose V2) consiste en usar varios archivos con el flag `-f`:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Los archivos se fusionan en orden. El último archivo tiene **prioridad** sobre los anteriores para claves que se solapan.

### 7.9.2 Archivo base + archivo de entorno

**Archivo base: `docker-compose.yml`**

```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    image: myapp/app:${TAG:-latest}
    ports:
      - "${APP_PORT:-3000}:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASS}@db:5432/${DB_NAME}
    depends_on:
      - db
      - redis
    restart: unless-stopped
    networks:
      - backend

  db:
    image: postgres:${PG_VERSION:-16}-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
      POSTGRES_DB: ${DB_NAME}
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    networks:
      - backend

networks:
  backend:
    driver: bridge

volumes:
  db_data:
```

**Desarrollo: `docker-compose.dev.yml`**

```yaml
services:
  app:
    build:
      target: development
    ports:
      - "3000:3000"
      - "9229:9229"
    environment:
      NODE_ENV: development
      LOG_LEVEL: debug
    volumes:
      - ./app/src:/app/src
      - ./app/package.json:/app/package.json
    command: npm run dev
    restart: "no"

  adminer:
    image: adminer
    ports:
      - "8080:8080"
    environment:
      ADMINER_DEFAULT_SERVER: db

  mailpit:
    image: axllent/mailpit
    ports:
      - "8025:8025"
      - "1025:1025"
```

**Producción: `docker-compose.prod.yml`**

```yaml
services:
  app:
    build:
      target: production
    environment:
      NODE_ENV: production
      LOG_LEVEL: warn
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M

  db:
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 1G
```

**CI: `docker-compose.ci.yml`**

```yaml
services:
  app:
    build:
      target: testing
    environment:
      NODE_ENV: test
      CI: "true"
    command: npm run test:ci
    ports: []
    restart: "no"

  db:
    tmpfs:
      - /var/lib/postgresql/data
    ports: []

  redis:
    ports: []
```

### 7.9.3 Orden de fusión en detalle

La fusión sigue estas reglas:
- Las claves de diccionario se fusionan recursivamente (las claves del último archivo sobrescriben).
- Las claves de lista (como `ports`, `environment` en formato array, `volumes` en formato corto) se **reemplazan completamente**, no se concatenan.
- `networks` y `volumes` del nivel raíz también se fusionan.

**Ejemplo de fusión:**

```yaml
# base.yml
services:
  web:
    image: nginx
    ports:
      - "80:80"
    environment:
      FOO: bar
      BAZ: qux
    volumes:
      - data:/data

# override.yml
services:
  web:
    ports:
      - "8080:80"
    environment:
      BAZ: quux
      NEW: value
    labels:
      env: dev
```

**Resultado fusionado:**

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"          # Reemplaza la lista completa
    environment:
      FOO: bar             # Se mantiene de base
      BAZ: quux            # Sobrescrito por override
      NEW: value           # Añadido por override
    volumes:
      - data:/data         # Se mantiene de base
    labels:
      env: dev             # Añadido por override
```

### 7.9.4 Visualizar la configuración fusionada

```bash
# Ver la configuración resultante de la fusión
docker compose -f base.yml -f override.yml config

# Guardarla en un archivo
docker compose -f base.yml -f override.yml config > docker-compose.resolved.yml

# Solo servicios (sin redes ni volúmenes)
docker compose -f base.yml -f override.yml config --services

# Solo volúmenes
docker compose -f base.yml -f override.yml config --volumes
```

### 7.9.5 `extends:` (obsoleto en V2)

En Compose V1 existía la directiva `extends` para heredar servicios de otros archivos. En Compose V2, la práctica recomendada es usar múltiples archivos con `-f`:

```yaml
# Antiguo (V1) — NO recomendado
services:
  web:
    extends:
      file: common-services.yml
      service: base-app
```

La migración es directa: mueve `base-app` a un archivo separado y úsalo como archivo base con `-f`.

---

## 7.10 Comandos Compose — Referencia completa

### 7.10.1 `docker compose up` — Construir, crear, arrancar

El comando principal. Orquesta todo el ciclo de vida.

```bash
# Arrancar en primer plano (logs en stdout)
docker compose up

# Arrancar en segundo plano (detached mode)
docker compose up -d

# Reconstruir imágenes antes de arrancar
docker compose up --build

# Forzar recreación de contenedores (incluso sin cambios)
docker compose up --force-recreate

# No arrancar servicios enlazados por depends_on
docker compose up --no-deps api

# Escalar servicios
docker compose up -d --scale api=3 --scale worker=5

# Arrancar solo un servicio específico (+ dependencias)
docker compose up db

# Esperar a que los health checks pasen antes de salir
docker compose up --wait

# Arrancar y salir cuando un servicio específico termine
docker compose up --exit-code-from test

# Limpiar volúmenes anónimos al arrancar
docker compose up --renew-anon-volumes

# Baja y vuelve a subir
docker compose up --force-recreate --renew-anon-volumes -d
```

### 7.10.2 `docker compose down` — Parar y eliminar

```bash
# Parar y eliminar contenedores y redes (mantiene volúmenes e imágenes)
docker compose down

# Incluir volúmenes nombrados y anónimos
docker compose down -v
docker compose down --volumes

# Incluir imágenes construidas con build
docker compose down --rmi all      # Todas las imágenes
docker compose down --rmi local    # Solo imágenes sin tag remoto

# Timeout para parada graceful
docker compose down -t 30

# Eliminar también los recursos externos que no estén en el compose file
docker compose down --remove-orphans
```

### 7.10.3 `docker compose build` — Construir imágenes

```bash
# Construir todas las imágenes
docker compose build

# Sin caché
docker compose build --no-cache

# Pull de imágenes base antes de construir
docker compose build --pull

# Construir en paralelo
docker compose build --parallel

# Construir solo un servicio
docker compose build api

# Pasar build arguments
docker compose build --build-arg NODE_ENV=production --build-arg CI=true

# Usar memoria específica para BuildKit
docker compose build --memory 2g

# Habilitar SSH agent forwarding
docker compose build --ssh default
```

### 7.10.4 `docker compose ps` — Estado de los servicios

```bash
# Listar servicios del proyecto actual
docker compose ps

# Todos los servicios (incluyendo detenidos)
docker compose ps -a

# Solo IDs de contenedores
docker compose ps -q

# Formato JSON
docker compose ps --format json

# Formato personalizado
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

### 7.10.5 `docker compose logs` — Ver logs

```bash
# Todos los servicios
docker compose logs

# Seguir logs en tiempo real
docker compose logs -f

# Últimas 100 líneas
docker compose logs --tail 100

# Con timestamps
docker compose logs -t

# Solo un servicio
docker compose logs api

# Múltiples servicios
docker compose logs api db redis

# Sin color
docker compose logs --no-color
```

### 7.10.6 `docker compose exec` — Ejecutar comandos

```bash
# Shell interactivo
docker compose exec api sh

# Con usuario específico
docker compose exec -u root api bash

# Sin TTY (para scripts)
docker compose exec -T api ls -la /app

# Con variables de entorno adicionales
docker compose exec -e DEBUG=true api node --inspect server.js

# En un directorio de trabajo específico
docker compose exec -w /app/logs api ls

# Con privilegios (peligroso en producción)
docker compose exec --privileged api bash

# Enviar stdin incluso sin TTY
docker compose exec -T api python manage.py migrate < /dev/null
```

### 7.10.7 `docker compose run` — Nuevo contenedor para un comando

A diferencia de `exec`, `run` crea un **nuevo** contenedor basado en la configuración del servicio:

```bash
# Ejecutar migraciones
docker compose run --rm api python manage.py migrate

# Shell en un contenedor efímero
docker compose run --rm api bash

# Sin mapear puertos (evita conflictos)
docker compose run --service-ports api bash

# Con variables de entorno adicionales
docker compose run -e SEED_DB=true --rm api npm run seed

# Usar un entrypoint diferente
docker compose run --entrypoint /bin/sh api

# En un perfil específico
docker compose run --rm api_test npm test
```

**Patrones comunes con `run`:**

```bash
# Migraciones de base de datos
docker compose run --rm api rails db:migrate

# Seed de datos iniciales
docker compose run --rm api python manage.py loaddata initial_data.json

# Pruebas
docker compose run --rm api pytest --cov

# Tareas programadas (en cron)
docker compose run --rm worker python -m app.tasks.cleanup

# Consola interactiva
docker compose run --rm api python manage.py shell
```

### 7.10.8 `docker compose restart` — Reiniciar servicios

```bash
# Reiniciar todos los servicios
docker compose restart

# Reiniciar servicios específicos
docker compose restart api worker

# Timeout entre stop y start
docker compose restart -t 30 api
```

### 7.10.9 `docker compose start` / `stop` / `pause` / `unpause`

```bash
# Arrancar servicios detenidos (sin recrearlos)
docker compose start

# Detener servicios (mantiene contenedores)
docker compose stop

# Timeout para stop graceful
docker compose stop -t 30

# Pausar procesos (freeze cgroup)
docker compose pause

# Reanudar procesos
docker compose unpause
```

### 7.10.10 `docker compose pull` — Descargar imágenes

```bash
# Descargar todas las imágenes
docker compose pull

# Incluir dependencias de build
docker compose pull --include-deps

# Ignorar errores de imágenes no encontradas
docker compose pull --ignore-pull-failures

# Política de pull: missing (por defecto), always, never
docker compose pull --policy always

# Paralelo
docker compose pull --parallel

# Quiet mode
docker compose pull -q
```

### 7.10.11 `docker compose push` — Subir imágenes

```bash
# Subir imágenes construidas
docker compose push

# Solo un servicio
docker compose push api

# Ignorar errores
docker compose push --ignore-push-failures
```

### 7.10.12 `docker compose top` — Procesos en ejecución

```bash
# Listar procesos de todos los servicios
docker compose top

# Solo un servicio
docker compose top api
```

### 7.10.13 `docker compose config` — Validar y mostrar configuración

```bash
# Mostrar configuración final (fusionada, con variables interpoladas)
docker compose config

# Mostrar solo los nombres de servicios
docker compose config --services

# Mostrar solo los nombres de volúmenes
docker compose config --volumes

# Mostrar solo los hashes de configuración (para detectar cambios)
docker compose config --hash="*"

# Validar sin escribir output (útil en CI)
docker compose config -q

# Mostrar el archivo resuelto (sin comentarios, formato canónico)
docker compose config > docker-compose.resolved.yml
```

### 7.10.14 `docker compose watch` — Hot reload en desarrollo

```bash
docker compose watch
```

Requiere una sección `develop` en el servicio (Compose V2.22+):

```yaml
services:
  frontend:
    build: .
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
          ignore:
            - node_modules/
            - .git/

        - action: rebuild
          path: package.json

        - action: sync+restart
          path: ./config
          target: /app/config
```

**Acciones disponibles:**
- **`sync`**: copia archivos al contenedor en caliente (sin reiniciar).
- **`rebuild`**: reconstruye la imagen y recrea el contenedor.
- **`sync+restart`**: copia archivos y reinicia el contenedor.
- **`sync+exec`**: copia archivos y ejecuta un comando.

```yaml
# Ejemplo completo de watch para desarrollo full-stack
services:
  frontend:
    build:
      context: ./frontend
      target: development
    ports:
      - "5173:5173"
    develop:
      watch:
        - action: sync
          path: ./frontend/src
          target: /app/src
        - action: rebuild
          path: ./frontend/package.json
        - action: sync+restart
          path: ./frontend/vite.config.ts
          target: /app/vite.config.ts

  backend:
    build:
      context: ./backend
      target: development
    ports:
      - "3000:3000"
    develop:
      watch:
        - action: sync+exec
          path: ./backend/src
          target: /app/src
          exec:
            command: ["npm", "run", "build"]
        - action: rebuild
          path: ./backend/package.json
```

### 7.10.15 Comandos adicionales

```bash
# Crear servicios sin arrancarlos
docker compose create

# Forzar creación de servicios
docker compose create --force-recreate

# Listar imágenes usadas por los servicios
docker compose images

# Mostrar versión de Docker Compose
docker compose version

# Ver estadísticas de uso de recursos
docker compose stats

# Copiar archivos entre host y contenedor
docker compose cp ./backup.sql db:/tmp/
docker compose cp db:/var/lib/postgresql/data/backup.sql ./

# Matar servicios inmediatamente (SIGKILL)
docker compose kill

# Eliminar contenedores detenidos
docker compose rm
```

---

## 7.11 Compose V1 vs V2 — Migración y diferencias

### 7.11.1 Historia y diferencias técnicas

| Característica | Compose V1 (`docker-compose`) | Compose V2 (`docker compose`) |
|---|---|---|
| **Binario** | `docker-compose` (Python, pip) | `docker compose` (Go, plugin de Docker) |
| **Instalación** | Independiente (pip/curl) | Integrado en Docker Desktop / plugin |
| **Lenguaje** | Python | Go |
| **Rendimiento** | Más lento (intérprete Python) | Más rápido (binario compilado) |
| **Formato de archivo** | `version:` requerido | `version:` opcional (obsoleto) |
| **Redes** | `docker-compose` crea redes con `_` | `docker compose` crea redes con `-` o `_` según configuración |
| **Contexto Docker** | No soportado | Soporta `docker context` |
| **BuildKit** | Requiere variable `DOCKER_BUILDKIT=1` | BuildKit por defecto |
| **GPU** | No soportado | Soporta `deploy.resources.reservations.devices` |
| **`docker compose watch`** | No disponible | Disponible (V2.22+) |
| **`profiles`** | No disponible | Disponible |
| **Comandos avanzados** | Limitados | `docker compose alpha`, `docker compose debug` |

### 7.11.2 Cambios en el formato de archivo

**V1 (obsoleto):**

```yaml
version: "3.8"
services:
  web:
    image: nginx
```

**V2 (moderno):**

```yaml
services:
  web:
    image: nginx
```

La clave `version:` es ignorada en Compose V2. Puedes omitirla completamente. Si tu archivo la tiene, no causa errores — simplemente se ignora.

### 7.11.3 Migración paso a paso

1. **Elimina `version:`** de todos tus archivos Compose.
2. **Reemplaza `docker-compose` por `docker compose`** en scripts, CI/CD y Makefiles.
3. **Elimina `extends:`** y usa múltiples archivos `-f`.
4. **Aprovecha `profiles`** para servicios condicionales en lugar de archivos separados.
5. **Usa `condition: service_healthy`** en `depends_on` (no disponible en V1).
6. **Adopta `docker compose watch`** para desarrollo con hot reload.

**Script de migración de CI/CD:**

```bash
# Antes (V1)
docker-compose -f docker-compose.yml -f docker-compose.test.yml up --abort-on-container-exit

# Después (V2)
docker compose -f docker-compose.yml -f docker-compose.test.yml up --exit-code-from test
```

**Alias recomendado para transición suave:**

```bash
# En ~/.bashrc o ~/.zshrc
alias docker-compose='docker compose'
```

---
## 7.12 Laboratorios

### 7.12.1 Laboratorio 1: LAMP Stack — Apache + PHP + MySQL + phpMyAdmin

**Objetivo**: Desplegar un stack LAMP completo con volúmenes persistentes, redes segmentadas, health checks y perfiles para herramientas de administración.

**Estructura del proyecto:**

```
lamp-stack/
├── docker-compose.yml
├── .env
├── php/
│   ├── Dockerfile
│   └── src/
│       └── index.php
├── mysql/
│   └── init/
│       └── 01-init.sql
└── apache/
    └── vhost.conf
```

**`.env`:**

```bash
# .env
MYSQL_ROOT_PASSWORD=root_secret_2024
MYSQL_DATABASE=lamp_db
MYSQL_USER=lamp_user
MYSQL_PASSWORD=lamp_pass_2024
PHP_VERSION=8.2
APACHE_PORT=8080
ADMINER_PORT=8081
PMA_PORT=8082
```

**`php/Dockerfile`:**

```dockerfile
FROM php:8.2-apache

RUN docker-php-ext-install pdo pdo_mysql mysqli \
    && docker-php-ext-enable pdo_mysql

RUN a2enmod rewrite

COPY vhost.conf /etc/apache2/sites-available/000-default.conf

RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/ || exit 1
```

**`apache/vhost.conf`:**

```apache
<VirtualHost *:80>
    DocumentRoot /var/www/html
    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

**`mysql/init/01-init.sql`:**

```sql
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO users (name, email) VALUES
('Alice', 'alice@example.com'),
('Bob', 'bob@example.com'),
('Charlie', 'charlie@example.com');
```

**`php/src/index.php`:**

```php
<?php
$host = getenv('MYSQL_HOST') ?: 'db';
$db   = getenv('MYSQL_DATABASE') ?: 'lamp_db';
$user = getenv('MYSQL_USER') ?: 'lamp_user';
$pass = getenv('MYSQL_PASSWORD') ?: 'lamp_pass_2024';

try {
    $pdo = new PDO("mysql:host=$host;dbname=$db;charset=utf8mb4", $user, $pass, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ]);

    $stmt = $pdo->query('SELECT id, name, email, created_at FROM users');
    $users = $stmt->fetchAll();
} catch (PDOException $e) {
    $error = $e->getMessage();
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>LAMP Stack - Docker Compose</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', system-ui, sans-serif;
            background: #0f172a;
            color: #e2e8f0;
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
        }
        .container {
            background: #1e293b;
            border-radius: 12px;
            padding: 2.5rem;
            width: 90%;
            max-width: 800px;
            box-shadow: 0 25px 50px -12px rgba(0,0,0,0.5);
        }
        h1 { color: #38bdf8; margin-bottom: 0.5rem; font-size: 2rem; }
        .subtitle { color: #94a3b8; margin-bottom: 2rem; }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 1.5rem;
        }
        th {
            text-align: left;
            padding: 0.75rem 1rem;
            background: #334155;
            color: #38bdf8;
            font-weight: 600;
        }
        td {
            padding: 0.75rem 1rem;
            border-bottom: 1px solid #334155;
        }
        tr:hover td { background: #1e3a5f; }
        .error {
            background: #7f1d1d;
            color: #fca5a5;
            padding: 1rem;
            border-radius: 8px;
            border-left: 4px solid #ef4444;
        }
        .badge {
            display: inline-block;
            padding: 0.25rem 0.75rem;
            border-radius: 20px;
            font-size: 0.8rem;
            font-weight: 600;
        }
        .badge-php { background: #4f46e5; color: #e0e7ff; }
        .badge-mysql { background: #0369a1; color: #e0f2fe; }
        .badge-docker { background: #0ea5e9; color: #e0f2fe; }
    </style>
</head>
<body>
    <div class="container">
        <h1>LAMP Stack con Docker Compose</h1>
        <p class="subtitle">
            <span class="badge badge-docker">Docker</span>
            <span class="badge badge-php">PHP <?= phpversion() ?></span>
            <span class="badge badge-mysql">MySQL 8.0</span>
        </p>

        <?php if (isset($error)): ?>
            <div class="error">
                <strong>Error de conexión:</strong> <?= htmlspecialchars($error) ?>
            </div>
        <?php else: ?>
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Nombre</th>
                        <th>Email</th>
                        <th>Creado</th>
                    </tr>
                </thead>
                <tbody>
                    <?php foreach ($users as $user): ?>
                    <tr>
                        <td><?= $user['id'] ?></td>
                        <td><?= htmlspecialchars($user['name']) ?></td>
                        <td><?= htmlspecialchars($user['email']) ?></td>
                        <td><?= $user['created_at'] ?></td>
                    </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        <?php endif; ?>
    </div>
</body>
</html>
```

**`docker-compose.yml` (LAMP Stack):**

```yaml
services:
  web:
    build:
      context: ./php
      dockerfile: Dockerfile
    container_name: lamp_web
    ports:
      - "${APACHE_PORT:-8080}:80"
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: "${MYSQL_DATABASE}"
      MYSQL_USER: "${MYSQL_USER}"
      MYSQL_PASSWORD: "${MYSQL_PASSWORD}"
    volumes:
      - ./php/src:/var/www/html:ro
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    labels:
      app: lamp-stack
      tier: web

  db:
    image: mysql:8.0
    container_name: lamp_db
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD}"
      MYSQL_DATABASE: "${MYSQL_DATABASE}"
      MYSQL_USER: "${MYSQL_USER}"
      MYSQL_PASSWORD: "${MYSQL_PASSWORD}"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d:ro
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test:
        [
          "CMD",
          "mysqladmin",
          "ping",
          "-h",
          "localhost",
          "-u",
          "root",
          "-p${MYSQL_ROOT_PASSWORD}",
        ]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    labels:
      app: lamp-stack
      tier: database

  adminer:
    image: adminer:latest
    container_name: lamp_adminer
    ports:
      - "${ADMINER_PORT:-8081}:8080"
    environment:
      ADMINER_DEFAULT_SERVER: db
      ADMINER_DESIGN: nette
    depends_on:
      - db
    networks:
      - backend
    profiles:
      - tools
    restart: "no"

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: lamp_pma
    ports:
      - "${PMA_PORT:-8082}:80"
    environment:
      PMA_HOST: db
      PMA_USER: root
      PMA_PASSWORD: "${MYSQL_ROOT_PASSWORD}"
    depends_on:
      - db
    networks:
      - backend
    profiles:
      - tools
    restart: "no"

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.25.10.0/24
  backend:
    driver: bridge
    internal: false
    ipam:
      config:
        - subnet: 172.25.20.0/24

volumes:
  mysql_data:
    driver: local
    labels:
      app: lamp-stack
      data: mysql
```

**Ejecución paso a paso:**

```bash
# 1. Navegar al directorio del proyecto
cd lamp-stack

# 2. Validar la configuración
docker compose config

# 3. Construir la imagen de PHP
docker compose build web

# 4. Arrancar el stack
docker compose up -d

# 5. Verificar que los servicios están corriendo
docker compose ps

# 6. Ver logs del servicio web
docker compose logs web -f

# 7. Verificar health checks
docker compose ps --format "table {{.Name}}\t{{.Status}}"

# 8. Acceder a la aplicación web
open http://localhost:8080

# 9. Abrir herramientas con perfiles
docker compose --profile tools up -d

# 10. Acceder a Adminer
open http://localhost:8081

# 11. Acceder a phpMyAdmin
open http://localhost:8082

# 12. Verificar conectividad desde web a db
docker compose exec web ping db

# 13. Ejecutar consulta SQL directa
docker compose exec db mysql -u lamp_user -p lamp_db -e "SELECT COUNT(*) FROM users;"

# 14. Detener todo (incluyendo perfiles)
docker compose --profile tools down

# 15. Eliminar volúmenes (datos persistentes)
docker compose down -v
```

---

### 7.12.2 Laboratorio 2: Stack de microservicios — API Gateway + Backend + PostgreSQL + Redis + RabbitMQ

**Objetivo**: Desplegar una arquitectura de microservicios con redes segmentadas, health checks en cascada y despliegue condicional de servicios de infraestructura.

**Estructura del proyecto:**

```
microservices-stack/
├── docker-compose.yml
├── docker-compose.dev.yml
├── docker-compose.prod.yml
├── .env
├── gateway/
│   ├── Dockerfile
│   └── nginx.conf
├── services/
│   ├── users/
│   │   ├── Dockerfile
│   │   └── src/
│   ├── orders/
│   │   ├── Dockerfile
│   │   └── src/
│   └── products/
│       ├── Dockerfile
│       └── src/
└── shared/
    └── healthcheck.sh
```

**`.env`:**

```bash
# .env
COMPOSE_PROJECT_NAME=microservices
DOCKER_REGISTRY=localhost:5000

# Database
PG_USER=microservice
PG_PASSWORD=S3cur3P@ss2024
PG_DB=microservices_db
PG_VERSION=16

# Redis
REDIS_VERSION=7-alpine

# RabbitMQ
RABBIT_USER=microservice
RABBIT_PASS=S3cur3P@ss2024
RABBIT_VERSION=3-management-alpine

# Services
USERS_PORT=3001
ORDERS_PORT=3002
PRODUCTS_PORT=3003
GATEWAY_PORT=80

# Logging
LOG_LEVEL=info
```

**`docker-compose.yml` (Microservicios):**

```yaml
services:
  gateway:
    build:
      context: ./gateway
      dockerfile: Dockerfile
      args:
        USERS_SERVICE: users
        ORDERS_SERVICE: orders
        PRODUCTS_SERVICE: products
    image: "${DOCKER_REGISTRY}/gateway:latest"
    container_name: ms_gateway
    ports:
      - "${GATEWAY_PORT:-80}:80"
    networks:
      - public
    depends_on:
      users:
        condition: service_healthy
      orders:
        condition: service_healthy
      products:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    logging:
      driver: json-file
      options:
        max-size: "20m"
        max-file: "5"
    labels:
      app: microservices
      tier: gateway

  users:
    build:
      context: ./services/users
      dockerfile: Dockerfile
    image: "${DOCKER_REGISTRY}/users:latest"
    container_name: ms_users
    environment:
      PORT: "3001"
      DATABASE_URL: "postgresql://${PG_USER}:${PG_PASSWORD}@postgres:5432/${PG_DB}"
      REDIS_URL: "redis://redis:6379/0"
      RABBITMQ_URL: "amqp://${RABBIT_USER}:${RABBIT_PASS}@rabbitmq:5672"
      LOG_LEVEL: "${LOG_LEVEL:-info}"
      SERVICE_NAME: users-service
    networks:
      - internal
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
        reservations:
          cpus: "0.25"
          memory: 128M
    labels:
      app: microservices
      tier: backend
      service: users

  orders:
    build:
      context: ./services/orders
      dockerfile: Dockerfile
    image: "${DOCKER_REGISTRY}/orders:latest"
    container_name: ms_orders
    environment:
      PORT: "3002"
      DATABASE_URL: "postgresql://${PG_USER}:${PG_PASSWORD}@postgres:5432/${PG_DB}"
      REDIS_URL: "redis://redis:6379/1"
      RABBITMQ_URL: "amqp://${RABBIT_USER}:${RABBIT_PASS}@rabbitmq:5672"
      LOG_LEVEL: "${LOG_LEVEL:-info}"
      SERVICE_NAME: orders-service
    networks:
      - internal
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3002/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
        reservations:
          cpus: "0.25"
          memory: 128M
    labels:
      app: microservices
      tier: backend
      service: orders

  products:
    build:
      context: ./services/products
      dockerfile: Dockerfile
    image: "${DOCKER_REGISTRY}/products:latest"
    container_name: ms_products
    environment:
      PORT: "3003"
      DATABASE_URL: "postgresql://${PG_USER}:${PG_PASSWORD}@postgres:5432/${PG_DB}"
      REDIS_URL: "redis://redis:6379/2"
      RABBITMQ_URL: "amqp://${RABBIT_USER}:${RABBIT_PASS}@rabbitmq:5672"
      LOG_LEVEL: "${LOG_LEVEL:-info}"
      SERVICE_NAME: products-service
    networks:
      - internal
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3003/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
        reservations:
          cpus: "0.25"
          memory: 128M
    labels:
      app: microservices
      tier: backend
      service: products

  postgres:
    image: "postgres:${PG_VERSION:-16}-alpine"
    container_name: ms_postgres
    environment:
      POSTGRES_USER: "${PG_USER}"
      POSTGRES_PASSWORD: "${PG_PASSWORD}"
      POSTGRES_DB: "${PG_DB}"
    volumes:
      - pg_data:/var/lib/postgresql/data
      - pg_backups:/backups
    networks:
      - internal
    restart: unless-stopped
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U ${PG_USER} -d ${PG_DB}",
        ]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
    labels:
      app: microservices
      tier: database

  redis:
    image: "redis:${REDIS_VERSION:-7-alpine}"
    container_name: ms_redis
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - internal
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s
    deploy:
      resources:
        limits:
          cpus: "0.25"
          memory: 128M
    labels:
      app: microservices
      tier: cache

  rabbitmq:
    image: "rabbitmq:${RABBIT_VERSION:-3-management-alpine}"
    container_name: ms_rabbitmq
    hostname: rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: "${RABBIT_USER}"
      RABBITMQ_DEFAULT_PASS: "${RABBIT_PASS}"
      RABBITMQ_DEFAULT_VHOST: "/"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - internal
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 384M
    labels:
      app: microservices
      tier: queue

  portainer:
    image: portainer/portainer-ce:latest
    container_name: ms_portainer
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - public
    profiles:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: ms_grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - internal
    profiles:
      - monitoring
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: ms_prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=30d"
    networks:
      - internal
    profiles:
      - monitoring
    restart: unless-stopped

networks:
  public:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.10.0/24
  internal:
    driver: bridge
    internal: false
    ipam:
      config:
        - subnet: 172.30.20.0/24

volumes:
  pg_data:
    driver: local
    labels:
      app: microservices
      tier: database
  pg_backups:
    driver: local
  redis_data:
    driver: local
  rabbitmq_data:
    driver: local
  portainer_data:
    driver: local
  grafana_data:
    driver: local
  prometheus_data:
    driver: local
```

**`gateway/nginx.conf`:**

```nginx
upstream users_backend {
    server users:3001 max_fails=3 fail_timeout=30s;
}

upstream orders_backend {
    server orders:3002 max_fails=3 fail_timeout=30s;
}

upstream products_backend {
    server products:3003 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name api.microservices.local;

    location /health {
        return 200 '{"status":"healthy","gateway":"OK"}';
        add_header Content-Type application/json;
    }

    location /api/users/ {
        proxy_pass http://users_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }

    location /api/orders/ {
        proxy_pass http://orders_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }

    location /api/products/ {
        proxy_pass http://products_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }
}
```

**`docker-compose.dev.yml`:**

```yaml
services:
  users:
    build:
      target: development
    volumes:
      - ./services/users/src:/app/src
      - ./shared:/app/shared
    environment:
      LOG_LEVEL: debug
    command: npm run dev
    ports:
      - "3001:3001"
      - "9229:9229"
    restart: "no"

  orders:
    build:
      target: development
    volumes:
      - ./services/orders/src:/app/src
      - ./shared:/app/shared
    environment:
      LOG_LEVEL: debug
    command: npm run dev
    ports:
      - "3002:3002"
    restart: "no"

  products:
    build:
      target: development
    volumes:
      - ./services/products/src:/app/src
      - ./shared:/app/shared
    environment:
      LOG_LEVEL: debug
    command: npm run dev
    ports:
      - "3003:3003"
    restart: "no"
```

**Ejecución paso a paso:**

```bash
# 1. Navegar al directorio del proyecto
cd microservices-stack

# 2. Validar la configuración
docker compose config

# 3. Construir todas las imágenes en paralelo
docker compose build --parallel

# 4. Arrancar el stack completo
docker compose up -d

# 5. Esperar a que todos los health checks pasen
docker compose up --wait

# 6. Verificar que todos los servicios están healthy
docker compose ps

# 7. Ver la topología de redes
docker network ls --filter name=microservices

# 8. Inspeccionar la red interna
docker network inspect microservices_internal

# 9. Ver logs de un servicio específico
docker compose logs users -f --tail 50

# 10. Verificar conectividad entre servicios
docker compose exec users curl -s http://orders:3002/health
docker compose exec users curl -s http://products:3003/health

# 11. Verificar la conectividad a través del gateway
curl http://localhost:80/health
curl http://localhost:80/api/users/health

# 12. Escalar un servicio (requiere eliminar container_name y port fijo)
docker compose up -d --scale users=3

# 13. Activar perfil de monitoreo
docker compose --profile monitoring up -d

# 14. Acceder a Grafana
open http://localhost:3000

# 15. Ejecutar migraciones de base de datos
docker compose run --rm users npm run db:migrate
docker compose run --rm orders npm run db:migrate
docker compose run --rm products npm run db:migrate

# 16. Ver estadísticas de recursos
docker compose stats

# 17. Reiniciar un servicio específico
docker compose restart products

# 18. Detener todo preservando volúmenes
docker compose --profile monitoring down

# 19. Recrear desde cero (incluyendo volúmenes)
docker compose down -v
docker compose up -d --build
```

---
### 7.12.3 Laboratorio 3: Entorno de desarrollo con hot reload

**Objetivo**: Configurar un entorno de desarrollo productivo con hot reload, bind mounts, perfiles para herramientas auxiliares y `docker compose watch`.

**Estructura del proyecto:**

```
dev-environment/
├── docker-compose.yml
├── docker-compose.watch.yml
├── .env
├── .env.example
├── frontend/
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   ├── package.json
│   └── vite.config.ts
├── backend/
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   └── package.json
└── database/
    └── seed.sql
```

**`docker-compose.yml` (Entorno de desarrollo):**

```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
      target: development
    container_name: dev_frontend
    ports:
      - "5173:5173"
    environment:
      VITE_API_URL: http://localhost:3000
      VITE_WS_URL: ws://localhost:3000
    volumes:
      - ./frontend/src:/app/src
      - ./frontend/public:/app/public
      - ./frontend/index.html:/app/index.html
      - frontend_node_modules:/app/node_modules
    depends_on:
      backend:
        condition: service_healthy
    restart: "no"
    stdin_open: true
    tty: true
    develop:
      watch:
        - action: sync
          path: ./frontend/src
          target: /app/src
          ignore:
            - "**/*.test.ts"
            - "**/*.spec.ts"
            - "**/__tests__/**"

        - action: rebuild
          path: ./frontend/package.json

        - action: sync+restart
          path: ./frontend/vite.config.ts
          target: /app/vite.config.ts

        - action: sync+restart
          path: ./frontend/tsconfig.json
          target: /app/tsconfig.json

    labels:
      app: dev-environment
      tier: frontend

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
      target: development
    container_name: dev_backend
    ports:
      - "3000:3000"
      - "9229:9229"
    environment:
      NODE_ENV: development
      PORT: "3000"
      DATABASE_URL: "postgresql://devuser:devpass@db:5432/devdb"
      REDIS_URL: "redis://redis:6379"
      LOG_LEVEL: debug
      DEBUG: "app:*"
    volumes:
      - ./backend/src:/app/src
      - ./backend/tests:/app/tests
      - ./backend/package.json:/app/package.json
      - backend_node_modules:/app/node_modules
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: "no"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 15s
    develop:
      watch:
        - action: sync+exec
          path: ./backend/src
          target: /app/src
          exec:
            command: ["npm", "run", "build"]

        - action: rebuild
          path: ./backend/package.json

        - action: sync+restart
          path: ./backend/tsconfig.json
          target: /app/tsconfig.json

    labels:
      app: dev-environment
      tier: backend

  db:
    image: postgres:16-alpine
    container_name: dev_db
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: devdb
    ports:
      - "5432:5432"
    volumes:
      - dev_db_data:/var/lib/postgresql/data
      - ./database/seed.sql:/docker-entrypoint-initdb.d/01-seed.sql:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devuser -d devdb"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s
    labels:
      app: dev-environment
      tier: database

  redis:
    image: redis:7-alpine
    container_name: dev_redis
    ports:
      - "6379:6379"
    volumes:
      - dev_redis_data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    labels:
      app: dev-environment
      tier: cache

  mailpit:
    image: axllent/mailpit:latest
    container_name: dev_mailpit
    ports:
      - "1025:1025"
      - "8025:8025"
    profiles:
      - debug
      - email
    labels:
      app: dev-environment
      tier: tools

  adminer:
    image: adminer:latest
    container_name: dev_adminer
    ports:
      - "8080:8080"
    environment:
      ADMINER_DEFAULT_SERVER: db
      ADMINER_DESIGN: nette
    profiles:
      - debug
      - tools
    labels:
      app: dev-environment
      tier: tools

  prisma-studio:
    image: node:20-alpine
    container_name: dev_prisma_studio
    working_dir: /app
    command: >
      sh -c "npm install prisma --save-dev &&
             npx prisma studio --port 5555"
    environment:
      DATABASE_URL: "postgresql://devuser:devpass@db:5432/devdb"
    volumes:
      - ./backend/prisma:/app/prisma
      - ./backend/package.json:/app/package.json
    ports:
      - "5555:5555"
    profiles:
      - debug
      - tools
    labels:
      app: dev-environment
      tier: tools

  test-runner:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
      target: testing
    container_name: dev_tests
    environment:
      NODE_ENV: test
      DATABASE_URL: "postgresql://devuser:devpass@db:5432/devdb_test"
      CI: "true"
    volumes:
      - ./backend/src:/app/src
      - ./backend/tests:/app/tests
      - ./backend/package.json:/app/package.json
      - ./backend/jest.config.ts:/app/jest.config.ts
      - backend_test_node_modules:/app/node_modules
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: ["npm", "run", "test:watch"]
    profiles:
      - test
      - debug
    labels:
      app: dev-environment
      tier: testing

  netshoot:
    image: nicolaka/netshoot
    container_name: dev_netshoot
    command: ["sleep", "infinity"]
    networks:
      - default
    profiles:
      - debug
    labels:
      app: dev-environment
      tier: debug

networks:
  default:
    driver: bridge

volumes:
  dev_db_data:
    driver: local
  dev_redis_data:
    driver: local
  frontend_node_modules:
    driver: local
  backend_node_modules:
    driver: local
  backend_test_node_modules:
    driver: local
```

**`.env`:**

```bash
COMPOSE_PROJECT_NAME=dev
COMPOSE_PROFILES=
```

**`backend/Dockerfile.dev`:**

```dockerfile
FROM node:20-alpine AS development

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY tsconfig.json ./
COPY . .

EXPOSE 3000 9229

CMD ["npm", "run", "dev"]

FROM node:20-alpine AS testing

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

CMD ["npm", "run", "test:ci"]
```

**`frontend/Dockerfile.dev`:**

```dockerfile
FROM node:20-alpine AS development

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY vite.config.ts tsconfig.json index.html ./
COPY public ./public
COPY src ./src

EXPOSE 5173

CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
```

**Ejecución paso a paso:**

```bash
# 1. Navegar al directorio del proyecto
cd dev-environment

# 2. Arrancar en modo desarrollo sin herramientas extra
docker compose up -d

# 3. Arrancar con herramientas de desarrollo
docker compose --profile debug up -d

# 4. Ver todos los servicios incluyendo perfiles
docker compose --profile debug --profile test ps

# 5. Ver logs con colores para distinguir servicios
docker compose logs -f --tail 20

# 6. Acceder a la aplicación frontend
open http://localhost:5173

# 7. Acceder a la API backend
curl http://localhost:3000/health

# 8. Acceder a Mailpit (emails capturados)
open http://localhost:8025

# 9. Acceder a Adminer (gestión de BD)
open http://localhost:8080

# 10. Acceder a Prisma Studio (ORM visual)
open http://localhost:5555

# 11. Usar netshoot para depurar conectividad
docker compose exec netshoot ping backend
docker compose exec netshoot curl -v http://backend:3000/health
docker compose exec netshoot dig db

# 12. Ejecutar tests en modo watch
docker compose --profile test run --rm test-runner npm run test:watch

# 13. Ejecutar migraciones de base de datos
docker compose exec backend npm run db:migrate

# 14. Seed de datos de prueba
docker compose exec backend npm run db:seed

# 15. Reiniciar solo el backend
docker compose restart backend

# 16. Reconstruir imágenes tras cambios en dependencias
docker compose build --no-cache backend

# 17. Usar docker compose watch para hot reload (V2.22+)
docker compose watch

# 18. Shell interactiva en el backend
docker compose exec backend sh

# 19. Copiar archivo del contenedor al host
docker compose cp backend:/app/logs/app.log ./logs/

# 20. Inspeccionar volúmenes anónimos de node_modules
docker volume ls --filter name=dev_

# 21. Detener el entorno preservando volúmenes
docker compose --profile debug --profile test down

# 22. Limpiar todo (incluyendo volúmenes y node_modules)
docker compose --profile debug --profile test down -v
```

---

### 7.12.4 Laboratorio 4: Entorno de testing para CI/CD

**Objetivo**: Crear un entorno efímero de testing que se ejecute en pipelines de CI, con bases de datos temporales, servicios mock y reportes de cobertura.

**Estructura del proyecto:**

```
ci-testing/
├── docker-compose.yml
├── docker-compose.ci.yml
├── .env.ci
├── app/
│   ├── Dockerfile
│   ├── Dockerfile.test
│   ├── package.json
│   ├── jest.config.ts
│   └── src/
├── e2e/
│   ├── Dockerfile
│   └── tests/
└── scripts/
    ├── wait-for-it.sh
    └── run-tests.sh
```

**`docker-compose.yml` (base):**

```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    image: myapp:${TAG:-latest}
    environment:
      NODE_ENV: production
      DATABASE_URL: "postgresql://${DB_USER}:${DB_PASS}@db:5432/${DB_NAME}"
      REDIS_URL: "redis://redis:6379"
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: "${DB_USER}"
      POSTGRES_PASSWORD: "${DB_PASS}"
      POSTGRES_DB: "${DB_NAME}"
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

volumes:
  pg_data:
```

**`docker-compose.ci.yml` (testing):**

```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile.test
      target: testing
      args:
        NODE_ENV: test
    image: myapp:test
    command: ["sh", "-c", "/app/scripts/wait-for-it.sh db:5432 -t 60 -- npm run test:ci"]
    environment:
      NODE_ENV: test
      DATABASE_URL: "postgresql://test_user:test_pass@db:5432/test_db"
      REDIS_URL: "redis://redis:6379"
      CI: "true"
      JEST_JUNIT_OUTPUT_DIR: "/app/reports/junit"
      JEST_JUNIT_OUTPUT_NAME: "results.xml"
    volumes:
      - ./app/coverage:/app/coverage
      - ./app/reports:/app/reports
    ports: []
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: "no"
    profiles:
      - ci

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_pass
      POSTGRES_DB: test_db
    tmpfs:
      - /var/lib/postgresql/data
    ports: []
    restart: "no"
    profiles:
      - ci

  redis:
    image: redis:7-alpine
    ports: []
    restart: "no"
    profiles:
      - ci

  db-migrate:
    build:
      context: ./app
      dockerfile: Dockerfile.test
      target: testing
    command: ["sh", "-c", "/app/scripts/wait-for-it.sh db:5432 -t 60 -- npm run db:migrate"]
    environment:
      DATABASE_URL: "postgresql://test_user:test_pass@db:5432/test_db"
      NODE_ENV: test
    depends_on:
      db:
        condition: service_healthy
    restart: "no"
    profiles:
      - ci

  db-seed:
    build:
      context: ./app
      dockerfile: Dockerfile.test
      target: testing
    command: ["sh", "-c", "/app/scripts/wait-for-it.sh db:5432 -t 60 -- npm run db:seed"]
    environment:
      DATABASE_URL: "postgresql://test_user:test_pass@db:5432/test_db"
      NODE_ENV: test
    depends_on:
      db-migrate:
        condition: service_completed_successfully
    restart: "no"
    profiles:
      - ci

  unit-tests:
    build:
      context: ./app
      dockerfile: Dockerfile.test
      target: testing
    command:
      [
        "sh",
        "-c",
        "/app/scripts/wait-for-it.sh db:5432 -t 60 -- npm run test:unit -- --ci --coverage --reporters=jest-junit",
      ]
    environment:
      DATABASE_URL: "postgresql://test_user:test_pass@db:5432/test_db"
      REDIS_URL: "redis://redis:6379"
      NODE_ENV: test
      CI: "true"
      JEST_JUNIT_OUTPUT_DIR: "/app/reports/junit"
      JEST_JUNIT_OUTPUT_NAME: "unit-results.xml"
    volumes:
      - ./app/coverage:/app/coverage
      - ./app/reports:/app/reports
    depends_on:
      db-seed:
        condition: service_completed_successfully
    restart: "no"
    profiles:
      - ci

  integration-tests:
    build:
      context: ./app
      dockerfile: Dockerfile.test
      target: testing
    command:
      [
        "sh",
        "-c",
        "/app/scripts/wait-for-it.sh db:5432 -t 60 -- npm run test:integration -- --ci --coverage --reporters=jest-junit",
      ]
    environment:
      DATABASE_URL: "postgresql://test_user:test_pass@db:5432/test_db"
      REDIS_URL: "redis://redis:6379"
      NODE_ENV: test
      CI: "true"
      JEST_JUNIT_OUTPUT_DIR: "/app/reports/junit"
      JEST_JUNIT_OUTPUT_NAME: "integration-results.xml"
    volumes:
      - ./app/coverage:/app/coverage
      - ./app/reports:/app/reports
    depends_on:
      db-seed:
        condition: service_completed_successfully
    restart: "no"
    profiles:
      - ci

  e2e-tests:
    build:
      context: ./e2e
      dockerfile: Dockerfile
    command:
      [
        "sh",
        "-c",
        "npx wait-on http://app:3000/health --timeout 120000 && npx playwright test --reporter=junit",
      ]
    environment:
      BASE_URL: http://app:3000
      CI: "true"
      PLAYWRIGHT_JUNIT_OUTPUT_NAME: "e2e-results.xml"
    volumes:
      - ./e2e/test-results:/app/test-results
      - ./e2e/reports:/app/reports
    depends_on:
      app:
        condition: service_healthy
    restart: "no"
    profiles:
      - ci

  lint:
    build:
      context: ./app
      dockerfile: Dockerfile.test
      target: testing
    command: ["npm", "run", "lint"]
    environment:
      NODE_ENV: test
      CI: "true"
    restart: "no"
    profiles:
      - ci

  typecheck:
    build:
      context: ./app
      dockerfile: Dockerfile.test
      target: testing
    command: ["npm", "run", "typecheck"]
    environment:
      NODE_ENV: test
      CI: "true"
    restart: "no"
    profiles:
      - ci

volumes:
  pg_data:
```

**`.env.ci`:**

```bash
COMPOSE_PROJECT_NAME=ci-tests
TAG=test
DB_USER=test_user
DB_PASS=test_pass
DB_NAME=test_db
COMPOSE_PROFILES=ci
```

**`app/Dockerfile.test`:**

```dockerfile
FROM node:20-alpine AS testing

RUN apk add --no-cache postgresql-client curl

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY jest.config.ts tsconfig.json .eslintrc.js .prettierrc ./
COPY src ./src
COPY tests ./tests
COPY scripts ./scripts

RUN chmod +x scripts/wait-for-it.sh

CMD ["npm", "run", "test:ci"]
```

**`scripts/wait-for-it.sh`:**

```bash
#!/usr/bin/env sh
# wait-for-it.sh - Espera a que un host:puerto esté disponible

TIMEOUT=15
QUIET=0

echoerr() { if [ "$QUIET" -ne 1 ]; then echo "$@" 1>&2; fi }

usage() {
    exitcode="$1"
    cat << USAGE >&2
Usage:
    $0 host:port [-s] [-t timeout] [-- command args]
    -h HOST | --host=HOST       Host o IP
    -p PORT | --port=PORT       Puerto TCP
    -s | --strict               Solo ejecutar command si el puerto está libre
    -q | --quiet                No mostrar mensajes
    -t TIMEOUT | --timeout=TIMEOUT
                                Tiempo máximo de espera en segundos (0 = infinito)
    -- COMMAND ARGS             Comando a ejecutar tras éxito
USAGE
    exit "$exitcode"
}

wait_for() {
    for i in $(seq $TIMEOUT); do
        if nc -z "$HOST" "$PORT" > /dev/null 2>&1; then
            return 0
        fi
        sleep 1
    done
    return 1
}

while [ $# -gt 0 ]; do
    case "$1" in
        *:* )
        HOST=$(echo "$1" | cut -d : -f 1)
        PORT=$(echo "$1" | cut -d : -f 2)
        shift 1
        ;;
        -q | --quiet)
        QUIET=1
        shift 1
        ;;
        -s | --strict)
        STRICT=1
        shift 1
        ;;
        -t)
        TIMEOUT="$2"
        shift 2
        ;;
        --timeout=*)
        TIMEOUT="${1#*=}"
        shift 1
        ;;
        --)
        shift
        CLI="$@"
        break
        ;;
        -h | --help)
        usage 0
        ;;
        *)
        echoerr "Unknown argument: $1"
        usage 1
        ;;
    esac
done

if [ -z "$HOST" ] || [ -z "$PORT" ]; then
    echoerr "Error: debes proporcionar host y puerto"
    usage 2
fi

echoerr "Esperando a $HOST:$PORT (timeout: ${TIMEOUT}s)..."
wait_for
RESULT=$?

if [ $RESULT -ne 0 ]; then
    echoerr "Timeout: $HOST:$PORT no disponible tras ${TIMEOUT}s"
    exit 1
fi

echoerr "$HOST:$PORT está disponible"

if [ -n "$CLI" ]; then
    exec $CLI
fi
```

**`scripts/run-tests.sh` (orquestador CI):**

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== CI Test Pipeline ==="
echo "Timestamp: $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
echo "Project: ${COMPOSE_PROJECT_NAME:-ci-tests}"
echo ""

# Colores para output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

run_stage() {
    local stage_name="$1"
    local service_name="$2"
    echo -e "${YELLOW}[$(date +%H:%M:%S)] ${stage_name}...${NC}"

    docker compose \
        -f docker-compose.yml \
        -f docker-compose.ci.yml \
        --env-file .env.ci \
        run --rm --use-aliases \
        "$service_name"

    local exit_code=$?
    if [ $exit_code -eq 0 ]; then
        echo -e "${GREEN}[OK] ${stage_name}${NC}"
    else
        echo -e "${RED}[FAIL] ${stage_name} (exit code: ${exit_code})${NC}"
        return $exit_code
    fi
}

# 1. Validar configuración Compose
echo -e "${YELLOW}[$(date +%H:%M:%S)] Validando configuración Compose...${NC}"
docker compose \
    -f docker-compose.yml \
    -f docker-compose.ci.yml \
    --env-file .env.ci \
    config -q
echo -e "${GREEN}[OK] Configuración válida${NC}"

# 2. Construir imágenes de prueba
echo -e "${YELLOW}[$(date +%H:%M:%S)] Construyendo imágenes de prueba...${NC}"
docker compose \
    -f docker-compose.yml \
    -f docker-compose.ci.yml \
    --env-file .env.ci \
    build --no-cache --pull

# 3. Arrancar servicios de infraestructura
echo -e "${YELLOW}[$(date +%H:%M:%S)] Arrancando infraestructura de pruebas...${NC}"
docker compose \
    -f docker-compose.yml \
    -f docker-compose.ci.yml \
    --env-file .env.ci \
    up -d db redis

FAILED_STAGES=()

# 4. Etapas de prueba
run_stage "Lint" "lint" || FAILED_STAGES+=("lint")
run_stage "TypeCheck" "typecheck" || FAILED_STAGES+=("typecheck")
run_stage "Migraciones BD" "db-migrate" || FAILED_STAGES+=("db-migrate")
run_stage "Seed BD" "db-seed" || FAILED_STAGES+=("db-seed")
run_stage "Unit Tests" "unit-tests" || FAILED_STAGES+=("unit-tests")
run_stage "Integration Tests" "integration-tests" || FAILED_STAGES+=("integration-tests")

# 5. Arrancar app para pruebas E2E
echo -e "${YELLOW}[$(date +%H:%M:%S)] Arrancando aplicación para E2E...${NC}"
docker compose \
    -f docker-compose.yml \
    -f docker-compose.ci.yml \
    --env-file .env.ci \
    up -d app

# Esperar a que la app esté healthy
echo -e "${YELLOW}[$(date +%H:%M:%S)] Esperando a que la app esté healthy...${NC}"
for i in $(seq 1 30); do
    if docker compose -f docker-compose.yml -f docker-compose.ci.yml ps app | grep -q "healthy"; then
        echo -e "${GREEN}[OK] App healthy${NC}"
        break
    fi
    if [ "$i" -eq 30 ]; then
        echo -e "${RED}[FAIL] Timeout esperando app healthy${NC}"
        FAILED_STAGES+=("app-healthcheck")
    fi
    sleep 2
done

run_stage "E2E Tests" "e2e-tests" || FAILED_STAGES+=("e2e-tests")

# 6. Limpiar
echo -e "${YELLOW}[$(date +%H:%M:%S)] Limpiando entorno de pruebas...${NC}"
docker compose \
    -f docker-compose.yml \
    -f docker-compose.ci.yml \
    --env-file .env.ci \
    down -v --remove-orphans

# 7. Resultados
echo ""
echo "=== Resultados ==="

# Buscar reportes JUnit
if [ -d "./app/reports/junit" ]; then
    echo "Reportes JUnit generados:"
    ls -la ./app/reports/junit/
fi

# Resumen final
if [ ${#FAILED_STAGES[@]} -eq 0 ]; then
    echo ""
    echo -e "${GREEN}=========================================${NC}"
    echo -e "${GREEN}  TODAS LAS PRUEBAS PASARON EXITOSAMENTE  ${NC}"
    echo -e "${GREEN}=========================================${NC}"
    exit 0
else
    echo ""
    echo -e "${RED}=========================================${NC}"
    echo -e "${RED}  ${#FAILED_STAGES[@]} ETAPA(S) FALLIDA(S): ${FAILED_STAGES[*]}  ${NC}"
    echo -e "${RED}=========================================${NC}"
    exit 1
fi
```

**Ejecución completa del pipeline CI:**

```bash
# 1. Navegar al directorio
cd ci-testing

# 2. Dar permisos de ejecución a los scripts
chmod +x scripts/wait-for-it.sh scripts/run-tests.sh

# 3. Ejecutar el pipeline completo
./scripts/run-tests.sh

# 4. Ejecutar solo una etapa específica
docker compose \
  -f docker-compose.yml \
  -f docker-compose.ci.yml \
  --env-file .env.ci \
  run --rm unit-tests

# 5. Ejecutar solo tests de integración
docker compose \
  -f docker-compose.yml \
  -f docker-compose.ci.yml \
  --env-file .env.ci \
  run --rm integration-tests

# 6. Ejecutar solo E2E (con app ya corriendo)
docker compose \
  -f docker-compose.yml \
  -f docker-compose.ci.yml \
  --env-file .env.ci \
  run --rm e2e-tests

# 7. Ver logs de una etapa fallida
docker compose \
  -f docker-compose.yml \
  -f docker-compose.ci.yml \
  --env-file .env.ci \
  logs unit-tests

# 8. Depurar: shell interactiva en el contenedor de test
docker compose \
  -f docker-compose.yml \
  -f docker-compose.ci.yml \
  --env-file .env.ci \
  run --rm --entrypoint sh unit-tests

# 9. Validar configuración final
docker compose \
  -f docker-compose.yml \
  -f docker-compose.ci.yml \
  --env-file .env.ci \
  config > docker-compose.ci.resolved.yml

# 10. Limpiar manualmente
docker compose \
  -f docker-compose.yml \
  -f docker-compose.ci.yml \
  down -v --remove-orphans
```

**Integración con GitHub Actions (`.github/workflows/test.yml`):**

```yaml
name: Test Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Cache Docker layers
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      - name: Run test pipeline
        run: |
          cd ci-testing
          chmod +x scripts/wait-for-it.sh scripts/run-tests.sh
          ./scripts/run-tests.sh

      - name: Upload unit test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: unit-test-results
          path: ci-testing/app/reports/junit/unit-results.xml

      - name: Upload integration test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: integration-test-results
          path: ci-testing/app/reports/junit/integration-results.xml

      - name: Upload E2E test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-test-results
          path: ci-testing/e2e/reports/

      - name: Upload coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: ci-testing/app/coverage/
```

---

---

← [Capítulo anterior](capitulo-06-redes.md) | [Inicio](README.md) | [Capítulo siguiente →](capitulo-08-orquestacion.md)
