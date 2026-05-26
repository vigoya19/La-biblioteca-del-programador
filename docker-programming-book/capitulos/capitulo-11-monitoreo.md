# Capítulo 11: Monitoreo, Logging y Troubleshooting de Contenedores

> *"No puedes arreglar lo que no puedes ver. No puedes mejorar lo que no puedes medir."*
>
> El monitoreo no es opcional en producción. Es la diferencia entre enterarte de una caída
> por tu dashboard de Grafana o enterarte por un tuit de un cliente furioso.

---

Los contenedores son efímeros por naturaleza. Nacen, viven segundos o meses, y mueren
llevándose consigo todo su estado interno. Si no tienes una estrategia de observabilidad
—logs centralizados, métricas históricas, alertas proactivas— estás volando a ciegas.

Este capítulo te lleva desde `docker logs` hasta un stack completo de monitoreo con
Prometheus + Grafana + cAdvisor + AlertManager, pasando por troubleshooting avanzado
con `nsenter` y herramientas de debug como `netshoot`.

Al terminar este capítulo sabrás:

- Capturar, rotar y centralizar logs de contenedores con ELK.
- Emitir logs estructurados (JSON) desde tus aplicaciones.
- Monitorear CPU, memoria, red y disco con `docker stats` y cAdvisor.
- Construir dashboards en Grafana con métricas de Prometheus.
- Configurar alertas que te despierten a las 3 AM (solo cuando sea necesario).
- Diagnosticar y reparar contenedores rotos con herramientas de bajo nivel.
- Hacer benchmark de rendimiento de contenedores.

---

## 11.1 Logging con Docker

### 11.1.1 El modelo de logging de Docker

Docker captura todo lo que un proceso escribe a **stdout** (file descriptor 1) y **stderr**
(file descriptor 2). Esa es la **única** fuente de logs que Docker conoce. No monta
directorios de log del contenedor. No lee archivos en `/var/log/`. Solo stdout y stderr.

Esta decisión de diseño es deliberada y poderosa: obliga a que las aplicaciones sean
*12-Factor Apps* en lo que respecta a logging. Trata los logs como streams de eventos,
no como archivos que gestionar.

Cuando ejecutas `docker logs <container>`, el daemon de Docker lee los logs desde el
*logging driver* configurado para ese contenedor y te los devuelve.

### 11.1.2 Logging drivers disponibles

Docker soporta múltiples *logging drivers*. Cada uno decide **dónde y cómo** se almacenan
o envían los logs capturados de stdout/stderr.

| Driver | Destino | Caso de uso |
|---|---|---|
| `json-file` | Archivos JSON en disco del host | Desarrollo, single-host, default |
| `syslog` | Syslog del sistema | Entornos tradicionales con syslog centralizado |
| `journald` | systemd journal | Hosts con systemd |
| `gelf` | Graylog Extended Log Format (UDP/TCP) | Centralización con Graylog |
| `fluentd` | Fluentd (forward protocol) | Pipelines de logging avanzados |
| `awslogs` | AWS CloudWatch Logs | ECS, EC2 en AWS |
| `splunk` | Splunk HTTP Event Collector | Empresas con Splunk |
| `gcplogs` | Google Cloud Logging | GKE, GCE en GCP |
| `etwlogs` | ETW (Event Tracing for Windows) | Windows containers |
| `logentries` | Logentries (Rapid7) | SaaS de logging |
| `none` | Descartar todos los logs | Cuando no quieres logs |

### 11.1.3 Configuración global en daemon.json

Puedes configurar el logging driver por defecto para todo el daemon de Docker editando
`/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "production-status",
    "env": "os,customer"
  }
}
```

Después de editar `daemon.json`, reinicia Docker:

```bash
sudo systemctl restart docker
```

**Nota importante**: La configuración en `daemon.json` es el **default** que se aplica a
todos los contenedores que no especifiquen su propio driver. Un contenedor siempre puede
sobrescribir esta configuración.

### 11.1.4 Configuración por contenedor

Cada contenedor puede especificar su propio logging driver y opciones al arrancar:

```bash
docker run -d \
  --name mi-app \
  --log-driver syslog \
  --log-opt syslog-address=udp://192.168.1.100:514 \
  --log-opt syslog-facility=daemon \
  --log-opt tag="mi-app/{{.Name}}" \
  nginx:alpine
```

Las opciones disponibles varían según el driver. Las más comunes:

| Driver | Opciones |
|---|---|
| `json-file` | `max-size`, `max-file`, `labels`, `env`, `env-regex`, `compress` |
| `syslog` | `syslog-address`, `syslog-facility`, `syslog-tls-*`, `tag` |
| `gelf` | `gelf-address`, `gelf-compression-type`, `gelf-compression-level`, `tag` |
| `fluentd` | `fluentd-address`, `fluentd-async`, `fluentd-buffer-limit`, `tag` |
| `awslogs` | `awslogs-region`, `awslogs-group`, `awslogs-stream`, `awslogs-create-group` |
| `splunk` | `splunk-token`, `splunk-url`, `splunk-source`, `splunk-index`, `tag` |

Las variables de plantilla (`{{.Name}}`, `{{.ID}}`, `{{.ImageName}}`, etc.) son
reemplazadas por Docker al enviar los logs, permitiendo identificar la fuente.

### 11.1.5 `json-file` en detalle

Es el driver por defecto. Cada línea de stdout/stderr se escribe como un objeto JSON
en un archivo dentro del host.

**Ubicación de los archivos de log:**

```
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

Cada entrada tiene esta estructura:

```json
{
  "log": "192.168.1.10 - - [20/May/2026:10:15:30 +0000] \"GET / HTTP/1.1\" 200 612\n",
  "stream": "stdout",
  "time": "2026-05-20T10:15:30.123456789Z"
}
```

**Rotación de logs (crítica para producción):**

Sin rotación, el archivo JSON crece indefinidamente hasta llenar tu disco. Configura
límites explícitos:

En `daemon.json` (global):

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Por contenedor:

```bash
docker run -d \
  --name api \
  --log-opt max-size=10m \
  --log-opt max-file=5 \
  my-api:latest
```

Esto mantiene hasta 5 archivos de 10 MB cada uno (50 MB total máximo). Cuando se
alcanza el límite, el archivo más antiguo se rota y eventualmente se elimina.

### 11.1.6 El comando `docker logs`

Tu primer aliado cuando algo falla. Docker logs lee los logs almacenados por el
logging driver y los imprime.

**Sintaxis básica:**

```bash
docker logs <container>
```

**Flags esenciales:**

```bash
# Seguir en tiempo real (como tail -f)
docker logs -f <container>

# Mostrar timestamps
docker logs -t <container>
# Output: 2026-05-20T10:15:30.123456789Z Server started on port 3000

# Últimas N líneas
docker logs --tail 50 <container>
docker logs --tail 100 -f <container>    # seguir desde las últimas 100 líneas

# Filtrar por rango temporal
docker logs --since 2026-05-20T10:00:00 <container>
docker logs --until 2026-05-20T11:00:00 <container>
docker logs --since 5m <container>       # últimos 5 minutos
docker logs --since 1h <container>       # última hora

# Filtrar logs de un servicio específico en Compose
docker compose logs -f api
docker compose logs --tail 50 --timestamps api db redis
```

**Ejemplo de troubleshooting con logs:**

```bash
# El contenedor se muere inmediatamente
docker run -d --name test my-app && docker logs -f test
# Output: "Error: Cannot connect to database at postgres:5432"

# Ver solo las últimas líneas antes del crash
docker logs --tail 20 test
```

### 11.1.7 La regla de oro de logging en contenedores

> **La aplicación debe escribir sus logs a stdout y stderr. Nunca a archivos
> dentro del contenedor.**

**Por qué:**

1. Docker **solo captura stdout y stderr**. Si tu app escribe a `/var/log/app.log`,
   ese archivo vive dentro del filesystem del contenedor y Docker no lo ve.

2. Cuando el contenedor muere, los archivos internos desaparecen con él. A menos
   que hayas montado un volumen, pierdes los logs.

3. Rotar archivos de log dentro del contenedor es complejo (logrotate, señales, etc.).
   Docker ya provee rotación nativa con `max-size`/`max-file`.

4. Centralizar logs desde archivos requiere sidecars, volúmenes compartidos o agentes
   dentro del contenedor. Con stdout/stderr, el daemon de Docker se encarga.

**Cómo loguear correctamente por lenguaje:**

**Node.js — console.log / console.error:**

```javascript
// Bien: stdout
console.log(JSON.stringify({
  level: 'info',
  message: 'User logged in',
  userId: 123,
  timestamp: new Date().toISOString()
}));

// Bien: stderr
console.error(JSON.stringify({
  level: 'error',
  message: 'Database connection failed',
  error: err.message,
  stack: err.stack
}));
```

**Python — print / logging a stdout:**

```python
import sys
import json
import logging

# Configurar logging para stdout
logging.basicConfig(
    level=logging.INFO,
    stream=sys.stdout,
    format='%(message)s'
)

logger = logging.getLogger(__name__)

# Emitir log estructurado
logger.info(json.dumps({
    "level": "info",
    "message": "Server started",
    "port": 8000
}))
```

**Java — System.out / System.err:**

```java
// Spring Boot: application.properties
// logging.file.name=   <-- DEJAR VACÍO para que use stdout

// O programáticamente con SLF4J + Logback configurado a stdout:
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

Logger logger = LoggerFactory.getLogger(MyClass.class);
logger.info("{\"level\":\"info\",\"message\":\"Request processed\",\"duration_ms\":42}");
```

**Go — fmt.Println / log.Println:**

```go
import (
    "encoding/json"
    "fmt"
    "log"
    "os"
)

func main() {
    // Por defecto, log escribe a stderr (correcto)
    log.SetOutput(os.Stderr)

    entry := map[string]interface{}{
        "level":   "info",
        "message": "Server listening",
        "port":    8080,
    }
    data, _ := json.Marshal(entry)
    fmt.Println(string(data)) // stdout
}
```

**Ruby — puts / logger con STDOUT:**

```ruby
require 'json'
require 'logger'

logger = Logger.new(STDOUT)
logger.info({ level: 'info', message: 'Worker started', pid: Process.pid }.to_json)
```

**.NET / C# — Console.WriteLine:**

```csharp
using System;
using System.Text.Json;

var logEntry = new { level = "info", message = "API started", port = 5000 };
Console.WriteLine(JsonSerializer.Serialize(logEntry));
```

---

## 11.2 Centralización de Logs con ELK

Cuando tienes 1 contenedor, `docker logs` es suficiente. Cuando tienes 100 contenedores
en 10 hosts, necesitas un sistema centralizado.

### 11.2.1 El stack ELK

ELK es el acrónimo de **Elasticsearch**, **Logstash** y **Kibana**. Con el tiempo se
añadió **Filebeat** (o Beats) como agente ligero de recolección.

**Arquitectura típica:**

```
┌─────────────┐     ┌──────────┐     ┌───────────────┐     ┌────────┐
│ Contenedores │────▶│ Filebeat │────▶│   Logstash    │────▶│ Elastic│
│ (stdout)     │     │ (recolecta)│   │ (procesa/filtra)│   │ search │
└─────────────┘     └──────────┘     └───────────────┘     └───┬────┘
                                                               │
                                                               ▼
                                                          ┌────────┐
                                                          │ Kibana │
                                                          │  (UI)  │
                                                          └────────┘
```

**Responsabilidad de cada componente:**

| Componente | Rol |
|---|---|
| **Filebeat** | Lee archivos de log de Docker (los JSON en `/var/lib/docker/containers/`) y los envía a Logstash o directamente a Elasticsearch |
| **Logstash** | Pipeline de procesamiento: parsea, filtra, enriquece y transforma logs antes de indexar |
| **Elasticsearch** | Motor de búsqueda y analítica. Almacena los logs y permite consultas full-text y agregaciones |
| **Kibana** | Interfaz web para visualizar, buscar y crear dashboards con los datos de Elasticsearch |

### 11.2.2 Desplegar ELK con Docker Compose

Este stack de ELK es funcional y usa la versión básica gratuita (Basic license).

**Archivo `docker-compose.elk.yml`:**

```yaml
version: "3.8"

services:
  # ==================== ELASTICSEARCH ====================
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
      - xpack.security.http.ssl.enabled=false
      - xpack.security.transport.ssl.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - elk
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health | grep -q 'green\|yellow'"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ==================== LOGSTASH ====================
  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m
      - XPACK_MONITORING_ENABLED=false
    ports:
      - "5044:5044"      # Beats input
      - "5000:5000/tcp"  # TCP input (opcional)
      - "9600:9600"      # API de monitoreo
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    networks:
      - elk
    depends_on:
      elasticsearch:
        condition: service_healthy

  # ==================== KIBANA ====================
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - XPACK_SECURITY_ENABLED=false
      - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=minimal-32-byte-key-for-dev
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      elasticsearch:
        condition: service_healthy

  # ==================== FILEBEAT ====================
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - elk
    depends_on:
      - logstash
    command: filebeat -e --strict.perms=false

networks:
  elk:
    driver: bridge
    name: elk

volumes:
  elasticsearch_data:
```

**Archivo `filebeat/filebeat.yml`:**

```yaml
filebeat.inputs:
  - type: container
    enabled: true
    paths:
      - /var/lib/docker/containers/*/*.log
    json.keys_under_root: false
    json.add_error_key: true
    json.message_key: log

    # Procesadores para enriquecer los logs con metadatos de Docker
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"
      - decode_json_fields:
          fields: ["message"]
          target: "json"
          overwrite_keys: true
          process_array: false
          max_depth: 2
      - drop_fields:
          fields: ["json.log"]

output.logstash:
  hosts: ["logstash:5044"]
  loadbalance: true

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

**Archivo `logstash/pipeline/docker-logs.conf`:**

```
input {
  beats {
    port => 5044
  }
}

filter {
  if [container][name] {
    mutate {
      add_field => { "[docker][container]" => "%{[container][name]}" }
    }
  }

  if [json] {
    json {
      source => "json"
      target => "parsed"
      skip_on_invalid_json => true
    }

    if [parsed][level] {
      mutate {
        add_field => { "severity" => "%{[parsed][level]}" }
      }
    }

    if [parsed][message] {
      mutate {
        add_field => { "log_message" => "%{[parsed][message]}" }
      }
    }
  }

  # Parsear fecha del log si está presente
  if [parsed][timestamp] {
    date {
      match => [ "[parsed][timestamp]", "ISO8601" ]
      target => "@timestamp"
    }
  }

  mutate {
    remove_field => ["json", "parsed", "stream", "log", "input", "agent", "ecs"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "docker-logs-%{+YYYY.MM.dd}"
    manage_template => false
  }
}
```

### 11.2.3 Levantar el stack

```bash
# Crear directorios necesarios
mkdir -p logstash/pipeline filebeat

# Crear los archivos de configuración (filebeat.yml y docker-logs.conf)
# Levantar
docker compose -f docker-compose.elk.yml up -d

# Verificar que todo está corriendo
docker compose -f docker-compose.elk.yml ps

# Ver logs del stack
docker compose -f docker-compose.elk.yml logs -f

# Acceder a Kibana: http://localhost:5601
# Elasticsearch: http://localhost:9200
```

### 11.2.4 Logs estructurados (JSON)

Los logs en texto plano son difíciles de consultar. Si un log dice:

```
2026-05-20 10:15:30 ERROR UserService: Failed to create user: duplicate email
```

Necesitas regex para extraer timestamp, nivel, servicio y mensaje. Con cada formato
diferente de cada aplicación, esto escala mal.

**Los logs estructurados (JSON) resuelven esto de raíz.** Cada línea es un objeto JSON
que los sistemas de logging pueden parsear automáticamente.

**Ejemplo de log estructurado (una línea por evento):**

```json
{"level":"error","message":"Failed to create user","service":"UserService","userId":null,"error":"duplicate email","duration_ms":12,"timestamp":"2026-05-20T10:15:30.123Z"}
```

**Ventajas:**

- **Consultas precisas**: `level:error AND service:UserService` en Kibana.
- **Agregaciones**: errores por endpoint, latencia p99 por servicio.
- **Sin regex**: cada campo tiene tipo (string, number, date).
- **Dashboards**: gráficas de errores/minuto, top 10 endpoints lentos, etc.

**Cómo emitir logs estructurados por lenguaje:**

**Node.js con Pino (el logger más rápido para Node):**

```javascript
const pino = require('pino');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level(label) {
      return { level: label };
    }
  },
  timestamp: pino.stdTimeFunctions.isoTime
});

logger.info({ userId: 42, action: 'login' }, 'User logged in');
logger.error({ err: new Error('DB connection failed'), retryCount: 3 }, 'Database error');
// Output (stdout): {"level":"info","time":"2026-05-20T10:15:30.123Z","userId":42,"action":"login","msg":"User logged in"}
```

**Python con structlog:**

```python
import structlog
import logging
import sys

structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.dev.ConsoleRenderer() if sys.stdout.isatty()
        else structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

logger = structlog.get_logger()

logger.info("User logged in", user_id=42, ip="192.168.1.10")
logger.error("Payment failed", order_id=999, reason="insufficient_funds")
# Output: {"event": "User logged in", "level": "info", "timestamp": "2026-05-20T10:15:30.123Z", "user_id": 42}
```

**Java con logstash-logback-encoder (Spring Boot):**

Añade la dependencia en `pom.xml`:

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

Archivo `src/main/resources/logback-spring.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeContext>false</includeContext>
            <timeZone>UTC</timeZone>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

Cada log en stdout será un JSON con campos como `@timestamp`, `level`, `logger_name`,
`message`, `stack_trace`, y cualquier campo que añadas con MDC.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

Logger log = LoggerFactory.getLogger(UserController.class);

MDC.put("userId", "42");
log.info("User profile updated");
MDC.clear();
// Output: {"@timestamp":"2026-05-20T10:15:30.123Z","level":"INFO","message":"User profile updated","userId":"42"}
```

**Go con zerolog:**

```go
import (
    "os"
    "github.com/rs/zerolog"
)

func main() {
    log := zerolog.New(os.Stderr).With().Timestamp().Logger()

    log.Info().
        Int("user_id", 42).
        Str("action", "login").
        Msg("User logged in")
    // Output: {"level":"info","user_id":42,"action":"login","time":"2026-05-20T10:15:30Z","message":"User logged in"}
}
```

### 11.2.5 Crear un dashboard en Kibana

Después de que los logs fluyan a Elasticsearch, puedes construir dashboards.

**Ejemplo de dashboard: Logs de Aplicación:**

1. Ve a **Kibana** > **Dashboard** > **Create dashboard**.
2. **Gráfica de logs por nivel** (bar chart):
   - Index pattern: `docker-logs-*`
   - X-axis: `@timestamp` (Date Histogram)
   - Y-axis: Count, split by `severity.keyword`
3. **Tabla de errores recientes**:
   - Filtro: `severity: "error"`
   - Columnas: `@timestamp`, `docker.container`, `log_message`
   - Sort by: `@timestamp` descending
4. **Logs por servicio** (pie chart):
   - Slice by: `docker.container.keyword`
   - Size: Count
5. **Top 10 mensajes de error**:
   - Split by: `log_message.keyword`
   - Size: 10
   - Limit: errores en últimas 24h

Guarda el dashboard. Ahora tienes visibilidad centralizada de todos los logs de tus
contenedores.

---

## 11.3 Métricas del Sistema

Conocer el estado de tus contenedores en tiempo real es fundamental. Docker incluye
varias herramientas integradas para esto.

### 11.3.1 `docker stats` — el monitor integrado

`docker stats` muestra un stream en vivo de uso de recursos de todos los contenedores
(o de uno específico):

```bash
docker stats
# Output:
# CONTAINER ID   NAME      CPU %   MEM USAGE / LIMIT     MEM %   NET I/O         BLOCK I/O     PIDS
# a1b2c3d4e5f6   api       2.15%   128.5MiB / 512MiB    25.10%  1.2MB / 850kB   0B / 0B       12
# b2c3d4e5f6a1   redis     0.50%   15.3MiB / 256MiB     5.98%   450kB / 320kB   2.5MB / 0B    4
# c3d4e5f6a1b2   postgres  1.80%   245.7MiB / 1GiB      24.00%  3.1MB / 2.8MB   45MB / 10MB   28
```

**Opciones útiles:**

```bash
# Una sola lectura (no stream)
docker stats --no-stream

# Formato personalizado (útil para scripts)
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}\t{{.PIDs}}"

# Solo un contenedor específico
docker stats mi-api

# Filtrar por nombre con --format para scripts de monitoreo
docker stats --no-stream --format "{{ json . }}" mi-api | jq .
```

**Campos disponibles para `--format`:**

| Placeholder | Descripción |
|---|---|
| `{{.Container}}` | Nombre o ID del contenedor |
| `{{.Name}}` | Nombre del contenedor |
| `{{.ID}}` | ID del contenedor |
| `{{.CPUPerc}}` | Porcentaje de CPU |
| `{{.MemUsage}}` | Uso de memoria (usado / límite) |
| `{{.MemPerc}}` | Porcentaje de memoria |
| `{{.NetIO}}` | I/O de red (recibido / enviado) |
| `{{.BlockIO}}` | I/O de bloque (lectura / escritura) |
| `{{.PIDs}}` | Número de procesos dentro del contenedor |

### 11.3.2 Entendiendo el output de `docker stats`

**CPU %:**
- Es el porcentaje de uso de CPU **del host** que consume el contenedor.
- En un host con 4 CPUs, si un contenedor muestra 100%, está usando 1 CPU completa.
- Si muestra 200%, está usando 2 CPUs completas.
- Si el contenedor tiene un límite de CPU (`--cpus=1.5`), el porcentaje sigue siendo
  relativo al host, no al límite.

**MEM USAGE / LIMIT:**
- `USAGE`: memoria actualmente en uso por el contenedor (RSS + cache de páginas).
- `LIMIT`: memoria máxima permitida. Si no se especificó con `--memory`, muestra la
  memoria total del host.
- Monitorear la diferencia entre USAGE y LIMIT es clave para detectar memory leaks.

**NET I/O:**
- Bytes recibidos (RX) / bytes enviados (TX) desde que el contenedor arrancó.
- Es acumulativo. Para ver la tasa, compara dos lecturas.

**BLOCK I/O:**
- Lecturas/escrituras a disco desde que el contenedor arrancó.
- Útil para detectar contenedores que escriben excesivamente a disco.

**PIDs:**
- Número de procesos en ejecución dentro del contenedor.
- Útil para detectar fork bombs o procesos zombies.

### 11.3.3 `htop` y `top` dentro de un contenedor

Para ver qué procesos dentro del contenedor consumen recursos:

```bash
# Ejecutar top dentro del contenedor
docker exec -it mi-api top

# Si htop está instalado en la imagen
docker exec -it mi-api htop

# Si no está instalado, puedes instalarlo temporalmente (no recomendado en producción)
docker exec -it mi-api sh -c "apt-get update && apt-get install -y htop && htop"
```

**Mejor práctica**: Usa un contenedor sidecar con privilegios para inspeccionar:

```bash
# Entrar al namespace de procesos del contenedor objetivo
docker run --rm -it \
  --pid container:mi-api \
  --net container:mi-api \
  --cap-add SYS_PTRACE \
  alpine sh -c "apk add htop && htop"
```

### 11.3.4 Métricas del host

Relacionar las métricas del contenedor con las del host es esencial para entender
el impacto global.

**Métricas del host:**

```bash
# CPU del host
top -bn1 | head -5
mpstat 1 5     # sysstat package

# Memoria del host
free -h
cat /proc/meminfo

# Disco del host
df -h
iostat -x 1 5  # sysstat package

# Red del host
iftop          # o nethogs
sar -n DEV 1 5 # sysstat package
```

**El cálculo correcto**: Si tu host tiene 8 CPUs y un contenedor usa 200% de CPU
(según `docker stats`), ese contenedor está consumiendo 2 CPUs completas, o el 25%
de la capacidad total del host.

---

## 11.4 Prometheus + Grafana — Stack de Monitoreo

`docker stats` es reactivo (tú miras). Prometheus + Grafana es proactivo: almacena
series temporales, te alerta antes de que algo explote, y visualiza tendencias.

### 11.4.1 Arquitectura del stack

```
┌───────────┐     scrape     ┌─────────────┐     query     ┌─────────┐
│  cAdvisor │───────────────▶│             │◀──────────────│ Grafana │
│  (metrics)│                │  Prometheus │               │  (dash) │
└───────────┘                │  (tsdb)     │               └─────────┘
                             │             │
┌───────────┐     scrape     │             │     alerts    ┌──────────────┐
│   Node    │───────────────▶│             │──────────────▶│ AlertManager │──▶ Email
│  Exporter │                └─────────────┘               └──────┬───────┘   Slack
└───────────┘                                                    │           PagerDuty
                                                                  │
┌───────────┐     scrape                                          │
│  App      │───────────────▶ (métricas de negocio:               │
│ /metrics  │                    requests, errores, latencia)     │
└───────────┘                                                     │
                                                                  ▼
                                                            Teams / OpsGenie
```

**Componentes:**

| Componente | Rol | Puerto por defecto |
|---|---|---|
| **Prometheus** | Recolecta métricas (pull vía HTTP), almacena en TSDB, evalúa reglas de alerta | 9090 |
| **Grafana** | Visualización: dashboards, gráficas, paneles | 3000 |
| **cAdvisor** | Expone métricas de uso de recursos de cada contenedor | 8080 |
| **Node Exporter** | Expone métricas del host (CPU, RAM, disco, red) | 9100 |
| **AlertManager** | Recibe alertas de Prometheus, las agrupa, deduplica y envía notificaciones | 9093 |

### 11.4.2 cAdvisor (Container Advisor)

cAdvisor es un proyecto de Google que corre como contenedor privilegiado y expone
métricas detalladas de todos los contenedores del host en formato Prometheus.

**Ejecutar cAdvisor standalone:**

```bash
docker run -d \
  --name cadvisor \
  --volume /:/rootfs:ro \
  --volume /var/run:/var/run:ro \
  --volume /sys:/sys:ro \
  --volume /var/lib/docker/:/var/lib/docker:ro \
  --volume /dev/disk/:/dev/disk:ro \
  --publish 8080:8080 \
  --privileged \
  gcr.io/cadvisor/cadvisor:v0.47.2
```

**Métricas clave expuestas por cAdvisor:**

| Métrica | Descripción |
|---|---|
| `container_cpu_usage_seconds_total` | CPU total consumida en segundos (contador acumulativo). Usa `rate()` para obtener uso actual |
| `container_memory_usage_bytes` | Memoria en uso en bytes |
| `container_memory_working_set_bytes` | Working set (excluye cache que puede ser reclamada) |
| `container_network_receive_bytes_total` | Bytes recibidos por red (acumulativo) |
| `container_network_transmit_bytes_total` | Bytes transmitidos por red (acumulativo) |
| `container_fs_reads_bytes_total` | Bytes leídos de disco (acumulativo) |
| `container_fs_writes_bytes_total` | Bytes escritos a disco (acumulativo) |
| `container_last_seen` | Timestamp de la última vez que se vio el contenedor |

### 11.4.3 Node Exporter

Métricas del host donde corre Docker: CPU, memoria, disco, red, system load, etc.

```bash
docker run -d \
  --name node-exporter \
  --network host \
  --pid host \
  --volume /proc:/host/proc:ro \
  --volume /sys:/host/sys:ro \
  --volume /:/rootfs:ro \
  prom/node-exporter:v1.7.0 \
  --path.procfs /host/proc \
  --path.sysfs /host/sys \
  --path.rootfs /rootfs
```

### 11.4.4 Configurar prometheus.yml

Prometheus necesita saber a qué *targets* hacer scrape (recolectar métricas).

**Archivo `prometheus/prometheus.yml`:**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'docker-monitor'

# Configuración de AlertManager (comentada si no está configurado aún)
# alerting:
#   alertmanagers:
#     - static_configs:
#         - targets: ['alertmanager:9093']

rule_files:
  - 'alerts.yml'

scrape_configs:
  # ====== Prometheus mismo ======
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # ====== cAdvisor: métricas de contenedores ======
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    metric_relabel_configs:
      # Filtrar solo contenedores en ejecución (ignorar los que ya no existen)
      - source_labels: [container_label_com_docker_compose_service]
        regex: '.+'
        action: keep

  # ====== Node Exporter: métricas del host ======
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # ====== Aplicación demo (métricas de negocio) ======
  - job_name: 'demo-app'
    static_configs:
      - targets: ['demo-app:4000']
    metrics_path: '/metrics'
```

**Archivo `prometheus/alerts.yml`:**

```yaml
groups:
  - name: container_alerts
    rules:
      # Alerta: contenedor caído
      - alert: ContainerDown
        expr: absent(container_last_seen{name=~".+"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Contenedor {{ $labels.name }} está caído"
          description: "El contenedor {{ $labels.name }} no reporta métricas desde hace 1 minuto"

      # Alerta: alto uso de CPU (> 80% por 5 minutos)
      - alert: HighCPUUsage
        expr: |
          sum(rate(container_cpu_usage_seconds_total{name!=""}[5m])) 
          / 
          sum(machine_cpu_cores) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU del host > 80%"
          description: "El host {{ $labels.instance }} está usando {{ $value | humanize }}% de CPU"

      # Alerta: alto uso de CPU por un contenedor específico
      - alert: HighContainerCPU
        expr: |
          sum(rate(container_cpu_usage_seconds_total{name!="",name=~"demo.+|api.+"}[5m])) by (name) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Contenedor {{ $labels.name }} con CPU > 80%"
          description: "El contenedor {{ $labels.name }} ha usado > 80% de CPU por más de 5 minutos"

      # Alerta: alto uso de memoria (> 90%)
      - alert: HighMemoryUsage
        expr: |
          container_memory_usage_bytes{name!=""}
          / 
          container_spec_memory_limit_bytes{name!=""} * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Contenedor {{ $labels.name }} con memoria > 90%"
          description: "Contenedor {{ $labels.name }} está usando {{ $value | humanize }}% de su límite de memoria"

      # Alerta: memory limit alcanzado (riesgo de OOM kill)
      - alert: ContainerOOMKillRisk
        expr: |
          container_memory_usage_bytes{name!=""}
          / 
          container_spec_memory_limit_bytes{name!=""} > 0.95
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Contenedor {{ $labels.name }} cerca de OOM"
          description: "El contenedor {{ $labels.name }} está al {{ $value | humanize }}% de su límite. Docker puede matarlo (OOM kill)"

      # Alerta: disco del host > 85%
      - alert: HighDiskUsage
        expr: |
          (node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"})
          /
          node_filesystem_size_bytes{mountpoint="/"} * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disco del host > 85%"
          description: "El disco del host {{ $labels.instance }} está al {{ $value | humanize }}%"

      # Alerta: demasiados contenedores reiniciándose
      - alert: HighRestartRate
        expr: rate(engine_daemon_container_actions_seconds_count{action="restart"}[5m]) > 1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Alta tasa de reinicios de contenedores"
          description: "Se están reiniciando más de 1 contenedor por segundo en promedio"
```

### 11.4.5 Desplegar el stack completo con Docker Compose

**Archivo `docker-compose.monitoring.yml`:**

```yaml
version: "3.8"

services:
  # ==================== PROMETHEUS ====================
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus_data:/prometheus
    networks:
      - monitoring
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ==================== GRAFANA ====================
  grafana:
    image: grafana/grafana:10.2.2
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource
      - GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH=/etc/grafana/dashboards/docker-monitoring.json
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ==================== cAdvisor ====================
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring
    restart: unless-stopped
    command:
      - '--housekeeping_interval=10s'
      - '--docker_only=true'
      - '--disable_metrics=percpu,sched,tcp,udp,disk,diskIO,accelerator,hugetlb,referenced_memory,cpu_topology,resctrl,process_metrics'

  # ==================== NODE EXPORTER ====================
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    pid: host
    network_mode: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  # ==================== ALERTMANAGER ====================
  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    networks:
      - monitoring
    restart: unless-stopped

  # ==================== DEMO APP ====================
  demo-app:
    build:
      context: ./demo-app
      dockerfile: Dockerfile
    container_name: demo-app
    ports:
      - "4000:4000"
    environment:
      - PORT=4000
      - NODE_ENV=production
    networks:
      - monitoring
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:4000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 10s

networks:
  monitoring:
    driver: bridge
    name: monitoring

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:
```

### 11.4.6 Configurar AlertManager

**Archivo `alertmanager/alertmanager.yml`:**

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@miempresa.com'
  smtp_auth_username: 'alerts@miempresa.com'
  smtp_auth_password: 'TU_APP_PASSWORD'
  smtp_require_tls: true

  slack_api_url: 'https://hooks.slack.com/services/TU/WEBHOOK/URL'

route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

  routes:
    - match:
        severity: critical
      receiver: 'critical-receiver'
      repeat_interval: 1h

    - match:
        severity: warning
      receiver: 'warning-receiver'

receivers:
  - name: 'default-receiver'
    slack_configs:
      - channel: '#monitoring'
        title: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        text: "{{ range .Alerts }}\n*Alert:* {{ .Annotations.summary }}\n*Description:* {{ .Annotations.description }}\n*Severity:* {{ .Labels.severity }}\n{{ end }}"

  - name: 'critical-receiver'
    email_configs:
      - to: 'oncall@miempresa.com'
        headers:
          Subject: '[CRITICAL] {{ .GroupLabels.alertname }}'
    slack_configs:
      - channel: '#oncall'
        title: '🚨 [CRITICAL] {{ .GroupLabels.alertname }}'
        text: "{{ range .Alerts }}\n*Alert:* {{ .Annotations.summary }}\n*Description:* {{ .Annotations.description }}\n{{ end }}"

  - name: 'warning-receiver'
    slack_configs:
      - channel: '#monitoring'
        title: '⚠️ [WARNING] {{ .GroupLabels.alertname }}'
        text: "{{ range .Alerts }}\n*Alert:* {{ .Annotations.summary }}\n{{ end }}"
```

### 11.4.7 Configurar Grafana — datasources y dashboards

**Archivo `grafana/datasources/prometheus.yml`:**

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: "15s"
```

### 11.4.8 Dashboard de Docker Monitoring en Grafana

Creamos un dashboard predefinido para monitoreo de Docker. Se puede cargar por
provisioning o importar manualmente con el ID 193 (Docker Monitoring) de la galería
de Grafana.

**Archivo `grafana/dashboards/docker-monitoring.json`** (dashboard funcional con paneles
clave de CPU, memoria, red y errores):

```json
{
  "title": "Docker Containers Monitoring",
  "tags": ["docker", "prometheus"],
  "refresh": "10s",
  "panels": [
    {
      "type": "stat",
      "title": "Contenedores Activos",
      "targets": [
        {
          "expr": "count(count(container_last_seen{name=~\".+\"}) by (name))",
          "legendFormat": "Containers"
        }
      ],
      "gridPos": { "h": 4, "w": 4, "x": 0, "y": 0 }
    },
    {
      "type": "stat",
      "title": "CPU Total (Host)",
      "targets": [
        {
          "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\"}[1m])) * 100)",
          "legendFormat": "CPU %"
        }
      ],
      "gridPos": { "h": 4, "w": 4, "x": 4, "y": 0 },
      "fieldConfig": {
        "defaults": {
          "unit": "percent"
        }
      }
    },
    {
      "type": "stat",
      "title": "Memoria Total (Host)",
      "targets": [
        {
          "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
          "legendFormat": "RAM %"
        }
      ],
      "gridPos": { "h": 4, "w": 4, "x": 8, "y": 0 },
      "fieldConfig": {
        "defaults": {
          "unit": "percent"
        }
      }
    },
    {
      "type": "stat",
      "title": "Disco (Host)",
      "targets": [
        {
          "expr": "(1 - node_filesystem_avail_bytes{mountpoint=\"/\"} / node_filesystem_size_bytes{mountpoint=\"/\"}) * 100",
          "legendFormat": "Disk %"
        }
      ],
      "gridPos": { "h": 4, "w": 4, "x": 12, "y": 0 },
      "fieldConfig": {
        "defaults": {
          "unit": "percent"
        }
      }
    },
    {
      "type": "stat",
      "title": "Network RX (Host)",
      "targets": [
        {
          "expr": "sum(rate(node_network_receive_bytes_total{device!=\"lo\"}[1m]))",
          "legendFormat": "RX"
        }
      ],
      "gridPos": { "h": 4, "w": 4, "x": 16, "y": 0 },
      "fieldConfig": {
        "defaults": {
          "unit": "Bps"
        }
      }
    },
    {
      "type": "stat",
      "title": "Network TX (Host)",
      "targets": [
        {
          "expr": "sum(rate(node_network_transmit_bytes_total{device!=\"lo\"}[1m]))",
          "legendFormat": "TX"
        }
      ],
      "gridPos": { "h": 4, "w": 4, "x": 20, "y": 0 },
      "fieldConfig": {
        "defaults": {
          "unit": "Bps"
        }
      }
    },
    {
      "type": "graph",
      "title": "CPU por Contenedor",
      "targets": [
        {
          "expr": "sum(rate(container_cpu_usage_seconds_total{name!=\"\"}[1m])) by (name)",
          "legendFormat": "{{name}}"
        }
      ],
      "gridPos": { "h": 12, "w": 12, "x": 0, "y": 4 },
      "fieldConfig": {
        "defaults": {
          "unit": "short"
        }
      },
      "yaxes": [
        { "format": "short", "label": "CPU Cores" },
        { "format": "short" }
      ]
    },
    {
      "type": "graph",
      "title": "Memoria por Contenedor",
      "targets": [
        {
          "expr": "container_memory_usage_bytes{name!=\"\"}",
          "legendFormat": "{{name}}"
        }
      ],
      "gridPos": { "h": 12, "w": 12, "x": 12, "y": 4 },
      "fieldConfig": {
        "defaults": {
          "unit": "bytes"
        }
      },
      "yaxes": [
        { "format": "bytes", "label": "Memory" },
        { "format": "short" }
      ]
    },
    {
      "type": "graph",
      "title": "Network RX por Contenedor",
      "targets": [
        {
          "expr": "sum(rate(container_network_receive_bytes_total{name!=\"\"}[1m])) by (name)",
          "legendFormat": "{{name}} RX"
        }
      ],
      "gridPos": { "h": 10, "w": 8, "x": 0, "y": 16 },
      "fieldConfig": {
        "defaults": {
          "unit": "Bps"
        }
      }
    },
    {
      "type": "graph",
      "title": "Network TX por Contenedor",
      "targets": [
        {
          "expr": "sum(rate(container_network_transmit_bytes_total{name!=\"\"}[1m])) by (name)",
          "legendFormat": "{{name}} TX"
        }
      ],
      "gridPos": { "h": 10, "w": 8, "x": 8, "y": 16 },
      "fieldConfig": {
        "defaults": {
          "unit": "Bps"
        }
      }
    },
    {
      "type": "graph",
      "title": "Bloques I/O por Contenedor",
      "targets": [
        {
          "expr": "sum(rate(container_fs_writes_bytes_total{name!=\"\"}[1m])) by (name)",
          "legendFormat": "{{name}} write"
        },
        {
          "expr": "sum(rate(container_fs_reads_bytes_total{name!=\"\"}[1m])) by (name)",
          "legendFormat": "{{name}} read"
        }
      ],
      "gridPos": { "h": 10, "w": 8, "x": 16, "y": 16 },
      "fieldConfig": {
        "defaults": {
          "unit": "Bps"
        }
      }
    },
    {
      "type": "graph",
      "title": "Requests por Segundo (demo-app)",
      "targets": [
        {
          "expr": "rate(http_requests_total[1m])",
          "legendFormat": "rps"
        }
      ],
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 26 }
    },
    {
      "type": "graph",
      "title": "Errores (5xx) por Endpoint",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status=~\"5..\"}[1m])) by (endpoint)",
          "legendFormat": "{{endpoint}}"
        }
      ],
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 26 }
    },
    {
      "type": "graph",
      "title": "Latencia p50/p95/p99 (demo-app)",
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[1m])) by (le))",
          "legendFormat": "p50"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m])) by (le))",
          "legendFormat": "p95"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[1m])) by (le))",
          "legendFormat": "p99"
        }
      ],
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 34 },
      "yaxes": [
        { "format": "s", "label": "Latency" },
        { "format": "short" }
      ]
    },
    {
      "type": "graph",
      "title": "Errores por Minuto (Tasa)",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status=~\"5..\"}[1m]))",
          "legendFormat": "5xx errors/min"
        },
        {
          "expr": "sum(rate(http_requests_total{status=~\"4..\"}[1m]))",
          "legendFormat": "4xx errors/min"
        }
      ],
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 34 }
    }
  ]
}
```

### 11.4.9 Consultas Prometheus útiles (PromQL)

```promql
# CPU utilizada (cores) por contenedor
sum(rate(container_cpu_usage_seconds_total{name!=""}[1m])) by (name)

# CPU % del total del host por contenedor
(sum(rate(container_cpu_usage_seconds_total{name!=""}[1m])) by (name) / scalar(sum(machine_cpu_cores))) * 100

# Memoria en uso (MB) por contenedor
container_memory_usage_bytes{name!=""} / 1024 / 1024

# Memoria % respecto al límite por contenedor
(container_memory_usage_bytes{name!=""} / container_spec_memory_limit_bytes{name!=""}) * 100

# Network throughput (bytes/s) por contenedor
sum(rate(container_network_receive_bytes_total{name!=""}[1m])) by (name)

# Top 5 contenedores por uso de CPU
topk(5, sum(rate(container_cpu_usage_seconds_total{name!=""}[5m])) by (name))

# Top 5 contenedores por uso de memoria
topk(5, container_memory_usage_bytes{name!=""})

# Tasa de requests HTTP por endpoint
sum(rate(http_requests_total[1m])) by (endpoint)

# Latencia p99 de HTTP requests
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Número de contenedores en ejecución
count(container_last_seen{name=~".+"})

# Contenedores que se han reiniciado en la última hora
changes(container_last_seen{name!=""}[1h]) > 0

# Tasa de errores 5xx como porcentaje del total
sum(rate(http_requests_total{status=~"5.."}[1m])) 
/ 
sum(rate(http_requests_total[1m])) * 100
```

---

## 11.5 Health Checks y Recuperación Automática

Un health check le dice a Docker (y a orquestadores como Kubernetes) si tu contenedor
está realmente funcionando. No basta con que el proceso exista; debe ser capaz de
responder correctamente.

### 11.5.1 HEALTHCHECK en Dockerfile

La instrucción `HEALTHCHECK` define un comando que Docker ejecuta periódicamente dentro
del contenedor para verificar su salud. El comando debe retornar 0 (éxito, healthy) o 1
(fracaso, unhealthy).

**Dockerfile con HEALTHCHECK:**

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

EXPOSE 4000

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget -q --spider http://localhost:4000/health || exit 1

USER node
CMD ["node", "server.js"]
```

**Parámetros de HEALTHCHECK:**

| Parámetro | Descripción | Default |
|---|---|---|
| `--interval` | Tiempo entre chequeos | 30s |
| `--timeout` | Tiempo máximo para que el comando responda | 30s |
| `--start-period` | Tiempo de gracia durante el arranque (los fallos no cuentan) | 0s |
| `--retries` | Fallos consecutivos necesarios para marcar unhealthy | 3 |

### 11.5.2 Health checks efectivos por tipo de aplicación

**API REST / Web:**

```dockerfile
HEALTHCHECK --interval=15s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q --spider http://localhost:3000/health || exit 1
```

**Base de datos (PostgreSQL):**

```dockerfile
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD pg_isready -U postgres || exit 1
```

**Base de datos (MySQL/MariaDB):**

```dockerfile
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD mysqladmin ping -h localhost -u root --password=$MYSQL_ROOT_PASSWORD || exit 1
```

**Redis:**

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD redis-cli ping || exit 1
```

**Worker (sin HTTP):**

```dockerfile
# Verificar que el proceso existe y no está zombie
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD pgrep -f "python worker.py" || exit 1
```

**Aplicación con endpoint /health dedicado:**

El endpoint `/health` debe verificar dependencias críticas, no solo devolver 200:

```javascript
// Node.js - Health check real
app.get('/health', async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    diskSpace: checkDiskSpace()
  };

  const allHealthy = Object.values(checks).every(v => v === true);

  if (allHealthy) {
    res.status(200).json({ status: 'healthy', checks, uptime: process.uptime() });
  } else {
    res.status(503).json({ status: 'unhealthy', checks });
  }
});
```

### 11.5.3 Consultar el estado de salud

```bash
# Ver el estado de salud de un contenedor
docker inspect --format='{{json .State.Health}}' mi-api | jq .

# Output:
# {
#   "Status": "healthy",
#   "FailingStreak": 0,
#   "Log": [
#     {
#       "Start": "2026-05-20T10:00:00.123456789Z",
#       "End": "2026-05-20T10:00:00.234567890Z",
#       "ExitCode": 0,
#       "Output": ""
#     }
#   ]
# }

# Solo el status
docker inspect --format='{{.State.Health.Status}}' mi-api
# Output: healthy
```

**Estados posibles:**

| Estado | Significado |
|---|---|
| `starting` | El contenedor está en `start-period` o aún no ha completado suficientes chequeos |
| `healthy` | El health check pasa correctamente |
| `unhealthy` | El health check ha fallado `retries` veces consecutivas |

### 11.5.4 Relación con restart policies

Un contenedor marcado como `unhealthy` **no** se reinicia automáticamente en Docker
standalone. El health check informa el estado, pero no toma acción. Sin embargo, en
Docker Swarm y Kubernetes, los health checks **sí** disparan reinicios o reemplazo
de réplicas.

**En Docker Swarm:**

```yaml
services:
  api:
    image: mi-api:latest
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
      update_config:
        order: start-first
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:4000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 10s
```

En Swarm, los health checks fallidos impactan el rolling update (se detiene si una
nueva réplica no se vuelve healthy).

### 11.5.5 Kubernetes: Liveness, Readiness y Startup Probes

Kubernetes extiende el concepto de health checks con tres tipos de probes:

**Liveness Probe:** ¿Está vivo el contenedor? Si falla, Kubernetes lo reinicia.
Ideal para detectar deadlocks, loops infinitos o estados irrecuperables.

**Readiness Probe:** ¿Está listo para recibir tráfico? Si falla, se remueve del
Service (no recibe requests). Ideal para dependencias externas o warmup.

**Startup Probe:** ¿Ya arrancó? Protege aplicaciones de arranque lento. Mientras
la startup probe no pase, liveness y readiness se deshabilitan.

**Ejemplo en Deployment de Kubernetes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mi-api
  template:
    metadata:
      labels:
        app: mi-api
    spec:
      containers:
        - name: api
          image: mi-api:latest
          ports:
            - containerPort: 4000
          # Startup probe: hasta 2 minutos para arrancar (app lenta)
          startupProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 12    # 12 x 10s = 120s máximo para arrancar
          # Liveness: ¿sigue vivo?
          livenessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 0
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3      # 3 fallos = reiniciar
          # Readiness: ¿puede recibir tráfico?
          readinessProbe:
            httpGet:
              path: /ready
              port: 4000
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3      # 3 fallos = fuera de Service
          resources:
            limits:
              cpu: "1"
              memory: "512Mi"
            requests:
              cpu: "250m"
              memory: "256Mi"
```

**Diferencias clave entre los tres probes:**

| Probe | ¿Qué pasa si falla? | ¿Cuándo usarlo? | Config típica |
|---|---|---|---|
| **Startup** | No pasan liveness ni readiness | Apps que tardan > 30s en arrancar | `failureThreshold: 12`, `periodSeconds: 10` (2 min) |
| **Liveness** | Reiniciar el contenedor | Detectar deadlocks, crashes silenciosos | `periodSeconds: 30`, `failureThreshold: 3` |
| **Readiness** | Fuera del balanceador | Dependencias no disponibles, warmup, overload | `periodSeconds: 10`, `failureThreshold: 3` |

---

## 11.6 Troubleshooting — Guía Completa

### 11.6.1 Contenedor no arranca

Es el problema #1. Ejecutas `docker run` y el contenedor se muere inmediatamente.

**Paso 1: Ver los logs (lo que emitio antes de morir)**

```bash
docker logs <container> --tail 50 -t
```

**Paso 2: Inspeccionar el estado de salida**

```bash
docker inspect <container> --format='{{json .State}}' | jq .
```

Campos clave en el output:

```json
{
  "Status": "exited",
  "Running": false,
  "Paused": false,
  "Restarting": false,
  "OOMKilled": false,
  "Dead": false,
  "Pid": 0,
  "ExitCode": 1,
  "Error": "",
  "StartedAt": "2026-05-20T10:00:00Z",
  "FinishedAt": "2026-05-20T10:00:02Z"
}
```

**Códigos de salida comunes:**

| ExitCode | Significado |
|---|---|
| 0 | Éxito (el proceso terminó normalmente) |
| 1 | Error genérico de la aplicación |
| 2 | Error de uso (argumentos incorrectos) |
| 126 | Comando encontrado pero no ejecutable (permisos) |
| 127 | Comando no encontrado |
| 128+n | Señal fatal (128+9=137 = SIGKILL; 128+15=143 = SIGTERM) |
| 137 | SIGKILL. Normalmente OOM kill o `docker kill`. Revisa `OOMKilled` |
| 139 | SIGSEGV (segmentation fault). Bug en la aplicación |
| 143 | SIGTERM. Recibió señal de terminación graciosa |

**Paso 3: Arrancar con un comando alternativo para debuggear**

```bash
# Sobrescribir entrypoint y comando para lanzar una shell
docker run --rm -it --entrypoint sh <image>

# O si quieres conservar el entrypoint pero cambiar el comando
docker run --rm -it <image> sh

# Inspeccionar el filesystem
docker run --rm -it --entrypoint sh <image> -c "ls -la /app && cat /app/package.json"
```

**Paso 4: Verificar variables de entorno**

```bash
docker run --rm -it --entrypoint sh <image> -c "env | sort"
```

**Paso 5: Verificar que el puerto no está ocupado**

```bash
# En el host
sudo lsof -i :<puerto>
sudo ss -tlnp | grep <puerto>

# Si está ocupado, usa otro puerto
docker run -d -p 8080:80 nginx
```

### 11.6.2 Contenedor arranca pero no responde

**Verificar puertos:**

```bash
# ¿Qué puertos expone y mapea?
docker port <container>

# ¿Responde en localhost?
curl http://localhost:<mapped-port>

# ¿Responde dentro del contenedor?
docker exec <container> wget -q -O- http://localhost:<container-port>
docker exec <container> curl -s http://localhost:<container-port>
```

**Verificar que la aplicación está escuchando en 0.0.0.0, no en 127.0.0.1:**

Un error clásico: la app escucha en `127.0.0.1` y solo es accesible desde dentro del
contenedor, no desde el host vía port mapping.

```bash
# Verificar interfaces de escucha
docker exec <container> netstat -tlnp
docker exec <container> ss -tlnp

# Output correcto: 0.0.0.0:3000
# Output incorrecto: 127.0.0.1:3000
```

En Node.js: `app.listen(3000, '0.0.0.0')` o simplemente `app.listen(3000)` (por defecto
es 0.0.0.0).

En Flask: `app.run(host='0.0.0.0', port=5000)`.

**Verificar health checks:**

```bash
docker inspect <container> --format='{{json .State.Health}}' | jq .
```

### 11.6.3 Alto consumo de CPU

**Paso 1: Identificar el contenedor problemático**

```bash
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemPerc}}\t{{.PIDs}}" | sort -k2 -rn
```

**Paso 2: Identificar el proceso culpable dentro del contenedor**

```bash
# Opción A: top dentro del contenedor
docker exec -it <container> top -bn1 | head -20

# Opción B: htop dentro del contenedor
docker exec -it <container> sh -c "apt-get update -qq && apt-get install -y -qq htop && htop"

# Opción C: desde el host, encontrar el PID del proceso
PID=$(docker inspect --format='{{.State.Pid}}' <container>)
ps -p $PID -o pid,ppid,pcpu,pmem,rss,vsz,comm --forest

# Todos los procesos del contenedor desde el host
ps -eo pid,ppid,pcpu,pmem,rss,comm --forest | grep -A 20 $PID
```

**Paso 3: Limitar CPU**

Si el consumo es normal para la carga de trabajo, pero afecta a otros contenedores,
establece límites:

```bash
# Actualizar un contenedor existente (requiere recrear)
docker update --cpus 1.5 --cpu-shares 512 <container>

# O mejor, recrear con límites
docker run -d --cpus=1.5 --cpu-shares=512 --name api mi-api:latest
```

### 11.6.4 Alto consumo de memoria

**Paso 1: Monitorear uso de memoria**

```bash
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}"
```

**Paso 2: Verificar si fue OOMKilled**

```bash
docker inspect <container> --format='{{.State.OOMKilled}}'
# true = el kernel mató el contenedor por falta de memoria
```

**Paso 3: Identificar memory leak**

Revisar logs de OOM del kernel:

```bash
dmesg | grep -i oom | tail -20
# o
journalctl -k | grep -i oom
```

**Paso 4: Verificar el working set real**

```bash
docker exec <container> cat /sys/fs/cgroup/memory/memory.usage_in_bytes
docker exec <container> cat /sys/fs/cgroup/memory/memory.stat
```

**Paso 5: Limitar memoria**

```bash
docker update --memory 512m --memory-swap 1g <container>
# O recrear
docker run -d --memory 512m --memory-swap 1g --name api mi-api:latest
```

### 11.6.5 Problemas de disco

El disco del host se llena con imágenes, volúmenes y cachés de Docker.

**Diagnóstico:**

```bash
# Resumen de uso de disco por Docker
docker system df

# Output:
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          25        8         12.5GB    8.2GB (65%)
# Containers      15        8         1.2GB     200MB (16%)
# Local Volumes   10        5         3.5GB     1.8GB (51%)
# Build Cache     50        0         2.1GB     2.1GB (100%)

# Detalle por componente
docker system df -v
```

**Limpieza:**

```bash
# Limpiar todo lo no usado (contenedores stopped, imágenes dangling, redes, build cache)
docker system prune -a --volumes

# Solo imágenes dangling
docker image prune

# Solo imágenes no usadas (todas)
docker image prune -a

# Solo contenedores detenidos
docker container prune

# Solo volúmenes no usados
docker volume prune

# Solo build cache
docker builder prune

# Solo redes no usadas
docker network prune

# Limpiar todo agresivamente (incluye imágenes sin contenedor asociado)
docker system prune -a -f --volumes
```

**Encontrar qué ocupa espacio:**

```bash
# Imágenes más grandes
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | sort -k3 -hr

# Contenedores más grandes (incluyendo volúmenes montados)
docker ps -s --format "table {{.Names}}\t{{.Size}}"

# Volúmenes por tamaño
sudo du -sh /var/lib/docker/volumes/*/_data | sort -hr | head -10
```

### 11.6.6 Problemas de red

**Diagnóstico de red del contenedor:**

```bash
# Inspeccionar la red a la que está conectado
docker network inspect <network-name>

# Ver IP del contenedor
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container>

# Ver puertos mapeados
docker port <container>

# Probar conectividad entre contenedores en la misma red
docker exec <container1> ping <container2>
docker exec <container1> nslookup <container2>
```

**Problema común: no hay resolución DNS entre contenedores**

Esto solo funciona en redes definidas por el usuario (`docker network create`), no en
la red `bridge` por defecto.

```bash
# Crear una red y conectar contenedores (DNS automático)
docker network create mi-red
docker run -d --name api --network mi-red mi-api
docker run -d --name db --network mi-red postgres:16-alpine

# Ahora api puede resolver db por nombre
docker exec api ping db
docker exec api nslookup db
```

**Inspeccionar tráfico de red:**

```bash
# Capturar paquetes en la interfaz del host
sudo tcpdump -i docker0 -n port 80

# Capturar paquetes dentro del namespace de red del contenedor
sudo nsenter -t $(docker inspect --format='{{.State.Pid}}' <container>) -n tcpdump -i eth0 -n
```

### 11.6.7 Depuración avanzada con `nsenter`

`nsenter` te permite entrar a los namespaces del contenedor desde el host, sin necesidad
de tener herramientas dentro del contenedor.

```bash
# Obtener PID del contenedor
CONTAINER_PID=$(docker inspect --format='{{.State.Pid}}' <container>)

# Entrar al namespace de red (ejecutar tcpdump, ip, ss, etc.)
sudo nsenter -t $CONTAINER_PID -n ss -tlnp
sudo nsenter -t $CONTAINER_PID -n ip addr
sudo nsenter -t $CONTAINER_PID -n tcpdump -i eth0 -n port 80

# Entrar al namespace de procesos
sudo nsenter -t $CONTAINER_PID -p -m ps aux

# Entrar al namespace de mounts (ver el filesystem del contenedor)
sudo nsenter -t $CONTAINER_PID -m ls -la /app

# Entrar a todos los namespaces (equivale a "estar dentro")
sudo nsenter -t $CONTAINER_PID -m -u -i -n -p -C sh
```

**Flags de nsenter:**

| Flag | Namespace |
|---|---|
| `-m` | Mount (filesystem) |
| `-u` | UTS (hostname) |
| `-i` | IPC |
| `-n` | Network |
| `-p` | PID |
| `-C` | Cgroup |

---

## 11.7 Herramientas de Debug

### 11.7.1 nicolaka/netshoot: Navaja Suiza de red

`netshoot` es un contenedor con todas las herramientas de troubleshooting de red
preinstaladas: `tcpdump`, `nmap`, `iperf`, `curl`, `wget`, `dig`, `netcat`, `iptables`,
`iproute2`, `ethtool`, `mtr`, `socat`, `strace`, y más.

```bash
# Inspeccionar el namespace de red de otro contenedor
docker run --rm -it --network container:<target> nicolaka/netshoot

# Ejemplos dentro de netshoot:
ip addr                        # Ver interfaces
ss -tlnp                       # Ver puertos en escucha
nslookup postgres              # Resolver DNS
ping redis                     # Probar conectividad
curl http://api:3000/health    # Probar HTTP
tcpdump -i eth0 -n port 80     # Capturar tráfico
iperf3 -c other-container      # Probar ancho de banda
nmap -sV api                   # Escanear puertos
dig api                        # DNS detallado
mtr redis                      # Traceroute combinado con ping
```

### 11.7.2 Debug con namespaces compartidos

Entrar al espacio de procesos y red de un contenedor sin instalar nada dentro de él:

```bash
# Unirte a los namespaces de PID y red de un contenedor
docker run --rm -it \
  --pid container:<target> \
  --net container:<target> \
  --cap-add SYS_PTRACE \
  alpine sh

# Ahora puedes hacer:
ps aux                         # Ver procesos del contenedor objetivo
top                            # Ver uso de recursos
netstat -tlnp                  # Ver puertos
kill -9 <pid>                  # Matar un proceso (si SYS_PTRACE está presente)
```

### 11.7.3 Debug de procesos con strace

`strace` rastrea llamadas al sistema. Es invaluable para diagnosticar por qué un
proceso falla silenciosamente o se cuelga.

```bash
# Strace en un contenedor existente desde el host
sudo strace -p $(docker inspect --format='{{.State.Pid}}' <container>) -f -e trace=network

# O dentro de un contenedor sidecar
docker run --rm -it \
  --pid container:<target> \
  --cap-add SYS_PTRACE \
  alpine sh -c "apk add strace && strace -p 1 -f"
```

### 11.7.4 Debug de señales

Ver si tu aplicación está manejando correctamente SIGTERM (esencial para graceful shutdown):

```bash
# Enviar SIGTERM
docker kill --signal=SIGTERM <container>

# Verificar que el proceso recibió la señal
docker logs <container> --tail 20

# Si el contenedor no se detiene, Docker espera el grace period (por defecto 10s)
# y luego envía SIGKILL. Puedes cambiar el timeout:
docker stop --time 30 <container>   # 30 segundos de gracia
```

---

## 11.8 Docker Events

`docker events` es un stream en tiempo real de todo lo que sucede en el daemon de Docker.
Cada acción (start, stop, kill, die, health_status, oom, create, destroy, etc.) genera
un evento con metadatos.

### 11.8.1 Monitorear eventos en tiempo real

```bash
# Todos los eventos
docker events

# Formato: 2026-05-20T10:15:30.123456789Z container create image=nginx:alpine, name=web
# 2026-05-20T10:15:31.234567890Z container start image=nginx:alpine, name=web
# 2026-05-20T10:15:32.345678901Z container health_status: healthy image=nginx:alpine, name=web
# 2026-05-20T10:20:00.456789012Z container die image=nginx:alpine, name=web, exitCode=137
# 2026-05-20T10:20:01.567890123Z container destroy image=nginx:alpine, name=web
```

### 11.8.2 Filtrar eventos

```bash
# Solo eventos de un contenedor específico
docker events --filter container=mi-api

# Solo eventos de un tipo
docker events --filter event=die
docker events --filter event=health_status
docker events --filter event=oom

# Solo eventos de imágenes
docker events --filter type=image

# Combinar filtros
docker events --filter type=container --filter event=die

# Filtrar por label
docker events --filter label=com.docker.compose.service=api

# Formato JSON para scripts
docker events --format '{{json .}}' | jq .
```

### 11.8.3 Usar Docker Events para respuesta automática

Un script simple que reacciona a eventos OOM:

```bash
#!/bin/bash
# oom-watcher.sh

docker events --filter event=oom --format '{{.Actor.Attributes.name}}' | while read container; do
  echo "[$(date -Iseconds)] ALERT: Container $container suffered OOM!"
  # Enviar alerta
  curl -X POST -H 'Content-Type: application/json' \
    -d "{\"text\":\"OOM en contenedor: $container\"}" \
    https://hooks.slack.com/services/TU/WEBHOOK/URL

  # Guardar estado para análisis post-mortem
  docker logs "$container" --tail 200 > "/var/log/oom-logs/$container-$(date +%s).log"
  docker inspect "$container" > "/var/log/oom-logs/$container-$(date +%s).json"
done
```

### 11.8.4 Eventos útiles para monitorizar

| Evento | Significado | Acción sugerida |
|---|---|---|
| `die` | Contenedor terminó | Verificar exitCode. Si != 0, alertar |
| `oom` | Contenedor matado por OOM killer | Aumentar memory limit o investigar leak |
| `health_status: unhealthy` | Health check falló | Revisar dependencias, logs |
| `destroy` | Contenedor eliminado | Verificar si fue intencional |
| `kill` | Contenedor recibió kill | ¿Quién y por qué? |
| `pause` / `unpause` | Contenedor pausado/reanudado | Poco común, revisar |

---

## 11.9 Benchmark y Profiling

### 11.9.1 Benchmark de CPU

```bash
# Ejecutar sysbench dentro de un contenedor para evaluar CPU
docker run --rm -it \
  --cpus=1 \
  severalnines/sysbench \
  sysbench cpu --cpu-max-prime=20000 run

# Comparar con nativo
sysbench cpu --cpu-max-prime=20000 run

# Resultados típicos: la diferencia debe ser < 2%
# Eventos por segundo contenedorizado: ~2800
# Eventos por segundo nativo: ~2850
```

### 11.9.2 Benchmark de memoria

```bash
docker run --rm -it \
  --memory=512m \
  severalnines/sysbench \
  sysbench memory --memory-block-size=1M --memory-total-size=10G run

# Esto mide velocidad de acceso a memoria (transferencia)
```

### 11.9.3 Benchmark de I/O

```bash
# Dentro del contenedor, benchmark de escritura
docker run --rm -it \
  --memory=512m \
  severalnines/sysbench \
  sysbench fileio --file-test-mode=rndrw prepare

docker run --rm -it \
  --memory=512m \
  -v sysbench-test:/tmp \
  severalnines/sysbench \
  sysbench fileio --file-test-mode=rndrw run
```

### 11.9.4 Profiling de aplicación dentro del contenedor

**Node.js con perf y flamegraphs:**

```bash
# Ejecutar node con --perf-basic-prof para habilitar perf
docker run -d --name api --security-opt seccomp=unconfined \
  -e NODE_ARGS="--perf-basic-prof" mi-api:latest

# Desde el host, perf con el PID del proceso node
sudo perf record -F 99 -p $(docker inspect --format='{{.State.Pid}}' api) -g -- sleep 30
sudo perf script > out.perf
```

**Java con JFR (JDK Flight Recorder):**

```bash
docker exec api jcmd 1 JFR.start duration=60s filename=/tmp/recording.jfr
docker cp api:/tmp/recording.jfr .
# Abrir recording.jfr en JDK Mission Control
```

**Go con pprof:**

```bash
# Si tu app Go expone pprof en :6060
docker exec api curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof -http=:8081 cpu.prof
```

### 11.9.5 Rendimiento contenedorizado vs nativo

**El overhead de los contenedores es negligible para CPU y memoria** (típicamente < 2%).
Donde sí puede haber diferencia:

- **I/O de disco**: el storage driver (overlay2, devicemapper) puede añadir overhead,
  especialmente en escrituras intensivas. Para bases de datos, considera volúmenes
  con driver de alto rendimiento (local, NVMe).
- **Red**: bridge NAT añade una pequeña latencia (< 1ms). Para rendimiento máximo sin
  overhead, usa `--network host`.
- **Syscalls**: seccomp y AppArmor pueden añadir overhead en workloads que hacen
  muchas syscalls. Si es necesario, ajusta los perfiles.

---

## 11.10 Laboratorio: Stack Completo de Monitoreo

### 11.10.1 Objetivo

Desplegar un stack que incluya:

1. **Aplicación demo**: web app con endpoints `/health`, `/api`, `/metrics` — emite logs
   estructurados y métricas Prometheus.
2. **Prometheus**: recolecta métricas de la app, cAdvisor, Node Exporter.
3. **Grafana**: dashboards con paneles de CPU, memoria, red, requests/segundo, errores.
4. **cAdvisor**: métricas de contenedores para Prometheus.
5. **Node Exporter**: métricas del host.
6. **AlertManager**: alertas (CPU > 80% por 5 minutos, memoria > 90%).
7. **Simulación de carga** con Apache Bench (ab) para ver métricas en tiempo real.

### 11.10.2 Demo App: aplicación Node.js con métricas

**Archivo `demo-app/package.json`:**

```json
{
  "name": "demo-monitoring-app",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pino": "^8.17.0",
    "pino-pretty": "^10.3.0",
    "prom-client": "^15.1.0"
  }
}
```

**Archivo `demo-app/server.js`:**

```javascript
const express = require('express');
const pino = require('pino');
const prometheus = require('prom-client');

const app = express();
const PORT = process.env.PORT || 4000;

// ====== Logger estructurado con Pino ======
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level(label) {
      return { level: label };
    }
  },
  timestamp: pino.stdTimeFunctions.isoTime
});

// ====== Métricas Prometheus ======
const collectDefaultMetrics = prometheus.collectDefaultMetrics;
collectDefaultMetrics({ prefix: 'app_' });

// Contador de requests HTTP
const httpRequestsTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'endpoint', 'status']
});

// Histograma de duración de requests
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'endpoint', 'status'],
  buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
});

// Gauge de requests en vuelo
const httpRequestsInFlight = new prometheus.Gauge({
  name: 'http_requests_in_flight',
  help: 'HTTP requests currently being processed'
});

// ====== Middleware de logging y métricas ======
app.use((req, res, next) => {
  const start = Date.now();
  httpRequestsInFlight.inc();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestsInFlight.dec();

    const labels = {
      method: req.method,
      endpoint: req.route ? req.route.path : req.path,
      status: res.statusCode.toString()
    };

    httpRequestsTotal.inc(labels);
    httpRequestDuration.observe(labels, duration);

    logger.info({
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration_ms: Date.now() - start,
      ip: req.ip,
      userAgent: req.get('user-agent')
    }, `${req.method} ${req.path} ${res.statusCode}`);
  });

  next();
});

// ====== Endpoints ======

// Health check
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
    memory: process.memoryUsage()
  });
});

// Métricas Prometheus
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await prometheus.register.metrics());
});

// API: simular trabajo
app.get('/api', (req, res) => {
  // Simular latencia variable
  const delay = Math.random() * 200;
  const start = Date.now();
  while (Date.now() - start < delay) {
    // burn CPU
  }

  res.json({
    message: 'Hello from Docker monitored API',
    delay_ms: Math.round(delay),
    timestamp: new Date().toISOString()
  });
});

// API: simular carga pesada
app.get('/api/heavy', (req, res) => {
  logger.info({ endpoint: '/api/heavy' }, 'Heavy computation started');

  // Simular CPU intensiva
  const start = Date.now();
  let result = 0;
  for (let i = 0; i < 5_000_000; i++) {
    result += Math.sqrt(i);
  }

  logger.info({
    endpoint: '/api/heavy',
    duration_ms: Date.now() - start,
    result: result.toFixed(2)
  }, 'Heavy computation completed');

  res.json({
    message: 'Heavy computation done',
    duration_ms: Date.now() - start,
    result: result.toFixed(2)
  });
});

// API: simular error
app.get('/api/error', (req, res) => {
  logger.error({
    endpoint: '/api/error',
    message: 'Simulated error for monitoring demo'
  }, 'Error endpoint triggered');

  res.status(500).json({
    error: 'Internal Server Error',
    message: 'This is a simulated error for monitoring demo',
    timestamp: new Date().toISOString()
  });
});

// API: simular 404
app.get('/api/notfound', (req, res) => {
  res.status(404).json({
    error: 'Not Found',
    message: 'Resource not found (simulated)'
  });
});

// ====== Iniciar servidor ======
app.listen(PORT, '0.0.0.0', () => {
  logger.info({
    port: PORT,
    node_version: process.version,
    env: process.env.NODE_ENV
  }, `Demo monitoring API started on port ${PORT}`);
  console.log(JSON.stringify({
    level: 'info',
    message: 'Server ready',
    port: PORT,
    pid: process.pid
  }));
});
```

**Archivo `demo-app/Dockerfile`:**

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --production && npm cache clean --force

COPY server.js .

RUN chown -R node:node /app
USER node

EXPOSE 4000

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -q --spider http://localhost:4000/health || exit 1

CMD ["node", "server.js"]
```

### 11.10.3 Docker Compose del laboratorio completo

**Archivo `docker-compose.lab.yml`:**

```yaml
version: "3.8"

services:
  # ========== DEMO APP ==========
  demo-app:
    build:
      context: ./demo-app
      dockerfile: Dockerfile
    container_name: demo-app
    ports:
      - "4000:4000"
    environment:
      - PORT=4000
      - NODE_ENV=production
      - LOG_LEVEL=info
    networks:
      - monitoring
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:4000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 10s

  # ========== PROMETHEUS ==========
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus_data:/prometheus
    networks:
      - monitoring
    restart: unless-stopped

  # ========== GRAFANA ==========
  grafana:
    image: grafana/grafana:10.2.2
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/datasources/prometheus.yml:/etc/grafana/provisioning/datasources/prometheus.yml:ro
      - ./grafana/dashboards/docker-monitoring.json:/etc/grafana/provisioning/dashboards/docker-monitoring.json:ro
      - ./grafana/dashboards/default.yml:/etc/grafana/provisioning/dashboards/default.yml:ro
    networks:
      - monitoring
    restart: unless-stopped

  # ========== CADVISOR ==========
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring
    restart: unless-stopped
    command:
      - '--housekeeping_interval=10s'
      - '--docker_only=true'
      - '--disable_metrics=percpu,sched,tcp,udp,disk,diskIO,accelerator,hugetlb,referenced_memory,cpu_topology,resctrl,process_metrics'

  # ========== NODE EXPORTER ==========
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    pid: host
    network_mode: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  # ========== ALERTMANAGER ==========
  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    networks:
      - monitoring
    restart: unless-stopped

networks:
  monitoring:
    driver: bridge
    name: monitoring

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:
```

### 11.10.4 Instrucciones del laboratorio

**Paso 0: Crear estructura de directorios y archivos**

```bash
# Crear todos los directorios necesarios
mkdir -p demo-app prometheus grafana/datasources grafana/dashboards alertmanager

# Crear los archivos según las secciones anteriores:
# - demo-app/package.json
# - demo-app/server.js
# - demo-app/Dockerfile
# - prometheus/prometheus.yml
# - prometheus/alerts.yml
# - grafana/datasources/prometheus.yml
# - grafana/dashboards/default.yml
# - grafana/dashboards/docker-monitoring.json
# - alertmanager/alertmanager.yml
# - docker-compose.lab.yml
```

**Archivo `grafana/dashboards/default.yml`** (provisioning de dashboards):

```yaml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
      foldersFromFilesStructure: true
```

**Paso 1: Levantar el stack**

```bash
# Construir la demo app
docker compose -f docker-compose.lab.yml build

# Levantar todos los servicios
docker compose -f docker-compose.lab.yml up -d

# Verificar que todo está corriendo
docker compose -f docker-compose.lab.yml ps

# Output esperado: todos los servicios con State "Up" y healthy
```

**Paso 2: Verificar que los endpoints de la demo app funcionan**

```bash
# Health
curl http://localhost:4000/health | jq .

# API normal
curl http://localhost:4000/api | jq .

# API pesada (CPU)
curl http://localhost:4000/api/heavy | jq .

# Simular error
curl http://localhost:4000/api/error | jq .

# Métricas Prometheus
curl http://localhost:4000/metrics | head -20
```

**Paso 3: Verificar Prometheus**

Abre http://localhost:9090 en tu navegador.

- **Status > Targets**: confirmar que cAdvisor, node-exporter y demo-app aparecen como "UP".
- **Graph**: ejecutar consultas de prueba:
  - `container_memory_usage_bytes{name="demo-app"}`
  - `rate(http_requests_total[1m])`
  - `container_cpu_usage_seconds_total{name="demo-app"}`

**Paso 4: Verificar Grafana**

Abre http://localhost:3000 (admin/admin) en tu navegador.

- **Home > Dashboards > Docker Containers Monitoring**: deberías ver los paneles.
- Verificar que las "stat cards" de CPU, RAM y contenedores activos muestren datos.

**Paso 5: Simular carga con Apache Bench**

```bash
# Instalar Apache Bench si no está disponible
# macOS: viene incluido
# Linux: sudo apt-get install apache2-utils

# Simular carga normal (1000 requests, 10 concurrentes)
ab -n 1000 -c 10 http://localhost:4000/api

# Simular carga pesada (500 requests a /api/heavy, 5 concurrentes)
ab -n 500 -c 5 http://localhost:4000/api/heavy

# Generar errores
ab -n 200 -c 5 http://localhost:4000/api/error

# Generar 404s
ab -n 100 -c 5 http://localhost:4000/api/notfound

# Mix completo de tráfico
for i in {1..1000}; do
  curl -s http://localhost:4000/api > /dev/null &
  curl -s http://localhost:4000/api/heavy > /dev/null &
  curl -s http://localhost:4000/api/error > /dev/null &
  curl -s http://localhost:4000/api/notfound > /dev/null &
  sleep 0.1
done
wait
```

**Paso 6: Ver métricas en tiempo real**

Mientras ejecutas las pruebas de carga, observa:

1. **Grafana Dashboard**: los paneles de CPU, requests/segundo y errores deben moverse.
2. **`docker stats`**: en otra terminal, observa el uso de recursos de la demo-app.

```bash
watch -n 1 'docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"'
```

3. **Prometheus > Graph**: ejecuta `rate(http_requests_total[1m])` para ver la tasa de
   requests en tiempo real.

**Paso 7: Disparar una alerta**

Para probar AlertManager, fuerza el uso de CPU por encima del 80%:

```bash
# Ejecutar múltiples requests a /api/heavy en paralelo por varios minutos
for i in {1..10}; do
  while true; do curl -s http://localhost:4000/api/heavy > /dev/null; done &
done

# Dejar correr por 5+ minutos. Verificar en:
# - Prometheus > Alerts: HighCPUUsage debe aparecer en estado "FIRING"
# - AlertManager UI: http://localhost:9093/#/alerts

# Para detener la carga:
kill %{1..10} 2>/dev/null
pkill -f "curl.*heavy"
```

**Paso 8: Simular fallo del contenedor y recuperación**

```bash
# Ver el health check en acción
docker inspect demo-app --format='{{json .State.Health}}' | jq .

# Matar el proceso dentro del contenedor (simula crash)
docker exec demo-app kill 1

# Observar que Docker intenta reiniciar (restart: unless-stopped)
docker ps -a --filter name=demo-app
docker logs demo-app --tail 10

# Verificar que se recupera
sleep 10
curl http://localhost:4000/health
```

**Paso 9: Explorar métricas históricas**

En Grafana, selecciona un rango de tiempo que incluya tus pruebas de carga (últimos
15-30 minutos). Deberías ver:

- **Pico en CPU** durante `/api/heavy`.
- **Pico en requests/segundo** durante `ab`.
- **Aumento en errores 5xx** durante las pruebas de error.
- **Latencia p99 elevada** durante carga concurrente.

**Paso 10: Limpiar**

```bash
# Detener y eliminar el stack
docker compose -f docker-compose.lab.yml down -v

# Limpiar imágenes construidas
docker rmi demo-monitoring-app demo-app 2>/dev/null

# Limpiar datos persistentes
docker volume rm $(docker volume ls -q | grep monitoring) 2>/dev/null
```

### 11.10.5 Checklist de verificación del laboratorio

| Item | Comando de verificación | Esperado |
|---|---|---|
| Demo app responde | `curl localhost:4000/health` | `{"status":"healthy"}` |
| Métricas expuestas | `curl localhost:4000/metrics \| grep http_requests_total` | Métricas Prometheus |
| Prometheus UP | http://localhost:9090/targets | Todos "UP" en verde |
| Grafana UP | http://localhost:3000 | Login exitoso |
| Dashboard carga datos | http://localhost:3000 > Dashboard | Paneles con datos |
| cAdvisor expone métricas | `curl localhost:8080/metrics \| grep container_memory_usage_bytes` | Métricas |
| Node Exporter expone métricas | `curl localhost:9100/metrics \| grep node_cpu_seconds_total` | Métricas |
| AlertManager UP | http://localhost:9093 | UI de AlertManager |
| Logs estructurados | `docker logs demo-app --tail 5` | JSON con campos level, message |
| Carga simulada genera métricas | Ejecutar `ab -n 1000 -c 10 localhost:4000/api` | Incremento en panels de Grafana |

---

## Resumen del Capítulo

- **Logging**: Docker captura stdout/stderr. Usa `json-file` con rotación para
  desarrollo/single-host. Para multi-host, ELK (Filebeat + Logstash + Elasticsearch +
  Kibana). Emite logs en **JSON estructurado** (Pino, structlog, zerolog, logstash-logback-encoder).
- **Métricas**: `docker stats` para monitoreo reactivo. Prometheus + cAdvisor + Node
  Exporter + Grafana para monitoreo proactivo con dashboards y series temporales.
- **Health checks**: `HEALTHCHECK` en Dockerfile. En Kubernetes: liveness, readiness y
  startup probes. Cada uno con propósito y configuración diferente.
- **AlertManager**: define reglas en Prometheus (PromQL) y rutas de notificación en
  AlertManager. Agrupa, deduplica y envía alertas por email, Slack, PagerDuty.
- **Troubleshooting**: flujo sistemático: `docker logs` → `docker inspect` (ExitCode,
  OOMKilled, Health) → shell en contenedor → `nsenter` → `netshoot`. Aprende los
  códigos de salida (137=SIGKILL, 139=SIGSEGV, 143=SIGTERM).
- **Herramientas de debug**: `nicolaka/netshoot` para red, `nsenter` para namespaces,
  `strace` para syscalls, `perf` para profiling.
- **Docker Events**: stream en tiempo real de todo lo que ocurre en el daemon. Úsalo
  para auditoría, alertas y automatización.
- **Benchmark**: `sysbench` para CPU, memoria e I/O. El overhead de contenedores es
  típicamente < 2% para CPU y memoria. Considera `--network host` para máximo
  rendimiento de red.

> **Principio fundamental**: No esperes a que algo falle para saber qué pasa. La
> observabilidad —logs, métricas y trazas— debe ser parte del despliegue, no un
> afterthought. Cuando un contenedor muere a las 3 AM, tu yo del futuro te lo
> agradecerá.

---

← [Capítulo anterior](capitulo-10-seguridad.md) | [Inicio](../README.md) | [Capítulo siguiente →](capitulo-12-produccion.md)
