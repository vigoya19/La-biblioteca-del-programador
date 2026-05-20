# Capítulo 12: Producción — El Último Kilómetro

> *"Un sistema en producción es como un avión en vuelo. No basta con que vuele: debe poder aterrizar, soportar turbulencias, y que cada pasajero sepa que hay un plan B, C y D."*

---

Bienvenido al capítulo final. Has recorrido un largo camino —desde tu primer `docker run hello-world` hasta orquestrar clústeres completos— y ahora estás frente a la última frontera: **producción**. Este no es un capítulo cualquiera. Es el capítulo donde todo cobra sentido, donde las piezas sueltas de los capítulos anteriores se ensamblan en un todo cohesionado. Aquí aprenderás no solo a desplegar, sino a mantener sistemas vivos, que respiran tráfico real, que deben sobrevivir a las 3 AM de un domingo y a los picos de Black Friday.

Producción no es un destino; es un estado mental. No se trata de "funciona en mi máquina", sino de "funciona para miles de usuarios simultáneos, con seguridad, resiliencia y observabilidad". Este capítulo te dará las herramientas, las estrategias y —más importante— el criterio para navegar ese territorio.

Al terminarlo, no serás un novato. Serás un ingeniero de software capaz de diseñar, desplegar y mantener sistemas contenerizados en producción. Y cuando las cosas fallen —porque fallarán— sabrás exactamente qué hacer.

El viaje comienza ahora.

---

## 12.1 El Checklist de Producción: 25 Puntos Que Debes Verificar Antes de Desplegar

> *"Un checklist no es una muestra de debilidad; es la marca del profesional que sabe que la memoria es frágil y la producción, implacable."*
>
> — Inspirado en *The Checklist Manifesto* de Atul Gawande

Los cirujanos usan checklists antes de operar. Los pilotos, antes de despegar. Tú, antes de un `docker push` a producción, también deberías. Aquí tienes 25 puntos, organizados en categorías, que debes verificar sistemáticamente. No son opcionales: cada uno previene una clase específica de desastre que alguien —probablemente tú— ya sufrió.

### 12.1.1 Imágenes: La Base de Todo

**Punto 1 — Usa tags específicos, nunca `latest`**

El tag `latest` es una mentira conveniente. No te dice qué commit, qué versión, qué estado del código estás ejecutando. En producción, `latest` significa "sorpresa" —y las sorpresas en producción rara vez son buenas.

```bash
# MAL: impredecible
docker pull miapp:latest

# BIEN: inmutable y trazable
docker pull miapp:1.2.3
docker pull miapp:git-abc1234
docker pull miapp:prod-2024-03-15
```

Cada despliegue debe ser determinístico. Si algo falla, necesitas saber exactamente qué estaba corriendo. El tag `latest` se mueve; tu confianza en el código, no.

**Punto 2 — Usa imágenes slim o alpine siempre que sea posible**

Una imagen de Node.js completa pesa ~950 MB. Su equivalente `node:20-slim`: ~240 MB. La versión `alpine`: ~120 MB. La diferencia no es solo espacio en disco: es velocidad de despliegue, superficie de ataque reducida, y menos paquetes que puedan contener vulnerabilidades.

```dockerfile
# MAL: 950 MB, cientos de paquetes innecesarios
FROM node:20

# BIEN: 120 MB, solo lo esencial
FROM node:20-alpine

# MEJOR AÚN: multi-stage + distroless (ver punto 5)
```

**Regla práctica:** si no necesitas compiladores, herramientas de build, o glibc completo, alpine es tu opción por defecto. Si algo no compila en alpine (ej. paquetes con bindings nativos), prueba `slim` basado en Debian.

**Punto 3 — Escanea tus imágenes: zero CRITICALs es la meta**

Un CVE (Common Vulnerability and Exposure) CRITICAL en tu imagen de producción es una bomba de tiempo. Docker Scout, Trivy, Snyk, Grype —elige tu herramienta, pero escanea.

```bash
# Con Docker Scout (integrado en Docker Desktop)
docker scout quickview miapp:1.2.3

# Con Trivy (open source, de Aqua Security)
trivy image miapp:1.2.3

# Con Grype (de Anchore)
grype miapp:1.2.3
```

Integra el escaneo en tu CI/CD. **Política estricta:** si hay un CRITICAL, el pipeline falla. No se debate, no se discute, no hay excepción sin aprobación del CISO. Los HIGH deben revisarse caso por caso. MEDIUM y LOW, documentarse y resolverse en el próximo sprint.

```yaml
# Ejemplo en GitHub Actions
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'miapp:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'  # Falla el pipeline si encuentra CRITICAL o HIGH
```

**Punto 4 — Usa multi-stage builds**

Multi-stage no es opcional; es arquitectura de imagen. Separas la etapa de compilación (con todas sus herramientas pesadas) de la etapa de ejecución (mínima, limpia). El resultado: imágenes pequeñas, seguras, sin artefactos de build.

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server .

# Stage 2: Runtime — una imagen scratch o distroless
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/server /server
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

¿El resultado? Una imagen de ~8 MB que solo contiene tu binario compilado. Sin shell, sin package manager, sin curl, sin nada que un atacante pueda explotar. Eso es defensa en profundidad real.

**Punto 5 — El Dockerfile debe ser revisado como código de producción**

Tu Dockerfile no es un script de conveniencia; es el blueprint de tu aplicación en producción. Debe pasar code review, debe tener test, debe seguir convenciones. Algunas reglas de oro:

- **Ordena las capas por frecuencia de cambio:** dependencias primero (raramente cambian), código fuente después. Así aprovechas el cache de Docker.
- **Un solo `RUN` por capa lógica,** encadenando comandos con `&&` para minimizar capas:
  ```dockerfile
  RUN apt-get update \
      && apt-get install -y --no-install-recommends ca-certificates curl \
      && rm -rf /var/lib/apt/lists/* \
      && curl -sL https://example.com/some-script.sh | bash \
      && rm -f some-script.sh
  ```
- **`.dockerignore` siempre presente:** evita que secretos, `node_modules` locales, archivos `.env` y binarios pesados se cuelen en el contexto de build.
  ```
  .git
  .env
  node_modules
  *.log
  .vscode
  Dockerfile*
  docker-compose*.yml
  ```

### 12.1.2 Seguridad: Defensa en Profundidad

**Punto 6 — Nunca ejecutes como root**

El usuario por defecto en casi todas las imágenes es `root`. Si un atacante explota tu aplicación, hereda privilegios de root dentro del contenedor. Aunque el contenedor tiene namespaces aislados, un root en el contenedor puede ser root en el host bajo ciertas configuraciones.

```dockerfile
# Al final de tu Dockerfile
RUN addgroup --system --gid 1001 appgroup \
    && adduser --system --uid 1001 --gid 1001 appuser
USER appuser
```

En Kubernetes, refuérzalo con un SecurityContext:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  allowPrivilegeEscalation: false
```

Si por alguna razón debes usar root (ej. puerto <1024), documenta la excepción, establece una fecha de caducidad y trabaja activamente para eliminarla.

**Punto 7 — Capacidades mínimas: drop ALL, add solo las necesarias**

Linux capabilities dividen los privilegios de root en unidades más pequeñas. Por defecto, Docker otorga un subconjunto razonable, pero puedes —y debes— ser más restrictivo.

```yaml
# Kubernetes SecurityContext
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE   # Solo si necesitas puerto <1024
```

La mayoría de las aplicaciones necesitan **cero** capabilities adicionales. Si tu aplicación "necesita" `SYS_ADMIN`, `SYS_PTRACE`, o `NET_RAW`, estás haciendo algo mal o estás ejecutando software que no debería estar en un contenedor.

**Punto 8 — Read-only filesystem en producción**

Tu aplicación no debería escribir en el sistema de archivos del contenedor en producción. Si necesita escribir, que sea en un volumen montado (temporal o persistente), no en la capa del contenedor.

```yaml
# Docker / Docker Compose
read_only: true
tmpfs:
  - /tmp
  - /var/run

# Kubernetes
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: cache
    mountPath: /var/cache/app
```

Un filesystem de solo lectura frustra ataques que intentan descargar malware, modificar binarios, o crear backdoors. Es un control barato y de alto impacto.

**Punto 9 — Secrets externalizados: nunca en la imagen, nunca en el código**

Los secretos (contraseñas, API keys, tokens, certificados) tienen un solo lugar: un gestor de secretos. No en variables de entorno hardcodeadas, no en archivos `.env` commitados, no en la imagen de Docker.

```yaml
# Kubernetes: Secrets montados como archivos
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: bXlhcHA=       # base64 de "myapp"
  password: c3VwZXJzZWNyZXQ=  # base64 de "supersecret"
---
# En el Deployment
volumeMounts:
  - name: secrets
    mountPath: /etc/secrets
    readOnly: true
volumes:
  - name: secrets
    secret:
      secretName: db-credentials
```

En la nube, integra con el gestor de secretos nativo:

- **AWS:** Secrets Manager o Parameter Store
- **GCP:** Secret Manager
- **Azure:** Key Vault
- **HashiCorp Vault:** para multi-cloud u on-premise

Usa herramientas como **External Secrets Operator (ESO)** para sincronizar secretos de proveedores cloud a Kubernetes Secrets automáticamente.

### 12.1.3 Salud y Resiliencia

**Punto 10 — HEALTHCHECK configurado en cada imagen**

Sin HEALTHCHECK, Docker no sabe si tu aplicación está viva o es un zombi que acepta conexiones TCP pero no responde. Un HEALTHCHECK correctamente definido es la diferencia entre que Docker reinicie un contenedor enfermo o que lo deje morir lentamente.

```dockerfile
# HEALTHCHECK para una API web
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Para una app que no tiene curl (imagen distroless/minimal):
# Usa wget, o mejor, expón un endpoint que el orquestrador consulte
```

El endpoint `/health` debe verificar dependencias críticas:

```python
# Ejemplo en Python/FastAPI
@app.get("/health")
async def health():
    db_ok = await check_database()
    redis_ok = await check_redis()
    if db_ok and redis_ok:
        return {"status": "healthy"}
    raise HTTPException(status_code=503, detail="unhealthy")
```

En Kubernetes, usa `livenessProbe` y `readinessProbe` en lugar del HEALTHCHECK de Docker (Kubernetes tiene su propio sistema):

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

La diferencia es crucial:
- **Liveness:** ¿Debo reiniciar este pod? (la app está muerta/congelada)
- **Readiness:** ¿Debo enviarle tráfico? (la app está viva pero no lista —ej. calentando cache)

**Punto 11 — Restart policy: unless-stopped o always, nunca no**

La restart policy determina qué hace Docker cuando tu contenedor muere:

```yaml
# Docker Compose
services:
  app:
    restart: unless-stopped  # Recomendado
    # restart: always        # Alternativa (reinicia incluso tras docker stop)
    # restart: on-failure    # Solo si el exit code no es 0
    # restart: no            # NUNCA en producción
```

`unless-stopped` es generalmente la mejor opción: reinicia automáticamente si el contenedor crashea, pero respeta un `docker stop` explícito. `always` también es válido pero reiniciará el contenedor incluso si tú lo detuviste manualmente (puede ser confuso en mantenimiento).

En Kubernetes, esto es implícito: un Deployment siempre intenta mantener el número deseado de réplicas. Pero puedes refinarlo con `restartPolicy` en Pods individuales si usas Jobs o CronJobs.

**Punto 12 — Timeouts y retries configurados en todas las conexiones**

Tu aplicación no vive en un mundo ideal. Las redes fallan, las bases de datos se saturan, los servicios externos se degradan. Sin timeouts, tu aplicación se colgará indefinidamente esperando una respuesta que nunca llegará.

```python
# Python: configura timeouts en todas las conexiones HTTP
import httpx
client = httpx.AsyncClient(timeout=httpx.Timeout(10.0, connect=3.0))

# Node.js
const axios = require('axios');
const instance = axios.create({
  timeout: 5000,  # 5 segundos
});

# Go
client := &http.Client{
    Timeout: 10 * time.Second,
}
```

En bases de datos:

```
# PostgreSQL
connect_timeout=10

# Redis
timeout 5
```

Y un principio fundamental: **siempre implementa circuit breakers** cuando llames a servicios externos. Si un servicio está caído, no sigas intentando —falla rápido, da una respuesta degradada, y protege tus recursos.

```python
# Circuit breaker con tenacity (Python)
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=1, max=10))
async def call_external_service():
    ...
```

### 12.1.4 Recursos: Ponle Límites al Contenedor

**Punto 13 — Memory limit siempre configurado**

Sin límites de memoria, un contenedor con memory leak o bajo carga extrema puede consumir toda la RAM del host y provocar OOM (Out of Memory) kills en cascada. El kernel de Linux matará procesos al azar —incluidos el daemon de Docker, SSH, o tu base de datos.

```yaml
# Docker Compose
services:
  app:
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
```

```yaml
# Kubernetes
resources:
  requests:
    memory: "256Mi"    # Lo que reservas para este pod
  limits:
    memory: "512Mi"    # El máximo que puede usar
```

La diferencia entre `requests` y `limits` es crucial:
- **requests:** el scheduler de Kubernetes usa esto para decidir en qué nodo colocar el pod. Si no hay un nodo con al menos 256Mi libres, el pod no se ejecuta.
- **limits:** el runtime (Docker/containerd) impone este hard limit. Si el contenedor excede 512Mi, es OOMKilled.

**Regla empírica:** para aplicaciones web típicas, `requests = limits` en producción simplifica el comportamiento. Para servicios batch o con picos de memoria, usa `requests < limits` para permitir bursting.

**Punto 14 — CPU limit configurado**

La CPU es un recurso comprimible (a diferencia de la memoria). Si no pones límites, un contenedor puede acaparar ciclos de CPU y degradar a todos los demás.

```yaml
# Kubernetes
resources:
  requests:
    cpu: "250m"     # 0.25 cores garantizados
  limits:
    cpu: "1000m"    # Máximo 1 core
```

En CPU, a diferencia de memoria, **exceder el límite no mata el proceso**: el kernel lo throttlea. Tu aplicación se vuelve más lenta pero no muere. Esto es bueno (no pierdes requests) y malo (la latencia aumenta sin que te enteres si no monitoreas).

**Mejor práctica:** en producción, `requests = limits` para CPU evita sorpresas de throttling. En staging/desarrollo, `requests < limits` para ahorrar costos.

**Punto 15 — PIDs limit configurado**

Cada proceso consume un PID. Si tu aplicación hace fork-bomb (accidental o maliciosamente), agota los PIDs del kernel y el host entero colapsa. El PID limit previene esto.

```yaml
# Docker
docker run --pids-limit 100 miapp

# Docker Compose
deploy:
  resources:
    limits:
      pids: 100

# Kubernetes
securityContext:
  # No directamente, pero puedes usar PodPidsLimit feature gate
```

Para la mayoría de aplicaciones web, 100-200 PIDs es más que suficiente. Herramientas como Nginx o Apache pueden necesitar más si usan worker processes.

### 12.1.5 Red: Aislar, Cifrar, Proteger

**Punto 16 — Redes segregadas: frontend/backend separados**

Un principio fundamental de seguridad de red: no todos los servicios necesitan hablar con todos. Separa tu infraestructura en redes lógicas:

```
Internet
   │
   ▼
[Frontend Network] — solo servicios que reciben tráfico externo
   │  (Nginx, API Gateway)
   ▼
[Backend Network] — servicios internos, sin exposición externa
   │  (microservicios, workers, tareas programadas)
   ▼
[Data Network] — bases de datos, caches, colas
   │  (PostgreSQL, Redis, RabbitMQ)
```

En Docker Compose:

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true   # Sin acceso a internet desde esta red
  data:
    driver: bridge
    internal: true

services:
  nginx:
    networks:
      - frontend
      - backend

  api:
    networks:
      - backend
      - data

  postgres:
    networks:
      - data   # Solo accesible desde la red de datos
```

En Kubernetes, usa NetworkPolicies con un CNI que las soporte (Calico, Cilium, Weave):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress: []  # Niega todo tráfico entrante por defecto
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-postgres-from-backend
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: backend
      ports:
        - protocol: TCP
          port: 5432
```

**Punto 17 — TLS en todo tráfico externo (y recomendado en tráfico interno)**

El tráfico HTTP sin cifrar es legible por cualquiera en la red. En 2024, no hay excusa para no usar TLS:

- **Tráfico externo:** obligatorio. Usa Let's Encrypt, cert-manager en Kubernetes.
- **Tráfico interno:** altamente recomendado. Service mesh (Istio/Linkerd) con mTLS automático.

```yaml
# cert-manager en Kubernetes: automatiza certificados Let's Encrypt
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mi-dominio-tls
spec:
  secretName: mi-dominio-tls-secret
  dnsNames:
    - api.midominio.com
    - www.midominio.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

Con Traefik como ingress/reverse proxy (alternativa más simple a Nginx Ingress para algunos casos):

```yaml
# Traefik configurado con Let's Encrypt automático
commands:
  - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
  - "--certificatesresolvers.letsencrypt.acme.email=admin@midominio.com"
  - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
```

**Punto 18 — Rate limiting y WAF**

Protege tus endpoints del abuso con rate limiting. Un ataque DDoS no tiene que ser sofisticado para tumbarte; a veces basta con un script de 10 líneas haciendo requests en bucle.

```yaml
# NGINX Ingress con rate limiting
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "50"
    nginx.ingress.kubernetes.io/limit-rate: "1024k"
```

Para protección más avanzada, considera un WAF (Web Application Firewall):
- **Cloud:** AWS WAF, Cloudflare WAF, GCP Cloud Armor
- **Self-hosted:** ModSecurity con NGINX, Coraza (WAF en Go)

### 12.1.6 Logging: Tu Linterna en la Oscuridad

**Punto 19 — Logs a stdout/stderr, nunca a archivos**

La regla de oro del logging en contenedores: **tu aplicación escribe logs a stdout/stderr, el runtime de contenedores los captura, y tu plataforma de logging los agrega y procesa.** No escribas logs a archivos dentro del contenedor. Si lo haces, desaparecerán cuando el contenedor muera y no podrás centralizarlos.

```python
# MAL
import logging
logging.basicConfig(filename='/var/log/app.log')

# BIEN
import logging
logging.basicConfig(stream=sys.stdout, level=logging.INFO)
```

**Punto 20 — Logs estructurados (JSON)**

En desarrollo, `ERROR - algo falló` es suficiente. En producción con cientos de servicios, necesitas poder filtrar por `service`, `trace_id`, `user_id`, `request_path`. Los logs estructurados lo hacen posible.

```python
# Python: structlog
import structlog
logger = structlog.get_logger()
logger.info("order_created", order_id="123", user_id="456", amount=99.99)
# Output: {"event": "order_created", "order_id": "123", "user_id": "456", "amount": 99.99, "timestamp": "..."}

# Node.js: pino
const pino = require('pino');
const logger = pino();
logger.info({ order_id: '123', user_id: '456' }, 'order_created');

# Go: zerolog
log.Info().Str("order_id", "123").Str("user_id", "456").Msg("order_created")
```

Elige un estándar y sé consistente. Campos que todo log debería tener:

| Campo       | Descripción                        |
|-------------|------------------------------------|
| `timestamp` | ISO 8601 con zona horaria          |
| `level`     | debug, info, warn, error, fatal    |
| `service`   | Nombre del servicio                |
| `trace_id`  | ID de tracing distribuido          |
| `user_id`   | Usuario autenticado (si aplica)    |
| `message`   | Descripción legible del evento     |

**Punto 21 — Rotación y retención configuradas**

Los logs crecen indefinidamente. Sin rotación, llenan discos y causan outages. Docker tiene rotación integrada:

```yaml
# /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

En producción, usa un driver de logging que envíe logs a un agregador central:

```bash
# Docker: enviar logs directamente a Fluentd
docker run --log-driver=fluentd --log-opt fluentd-address=localhost:24224 miapp

# O a syslog
docker run --log-driver=syslog miapp
```

En Kubernetes, el stack típico es:

```
Aplicación → stdout/stderr → containerd/Docker →
  Fluentd/Fluent Bit (DaemonSet) → Elasticsearch/Loki → Grafana/Kibana
```

### 12.1.7 Persistencia: Los Datos No Son Efímeros

**Punto 22 — Volúmenes nombrados y backups automáticos**

Los datos que importan no viven en el contenedor. Usa siempre volúmenes nombrados y ten un script de backup programado:

```yaml
# Docker Compose
services:
  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
volumes:
  postgres_data:
    name: postgres_data_prod
```

En Kubernetes, usa PersistentVolumeClaims con StorageClass que soporte snapshots:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  storageClassName: ssd-encrypted
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

**Punto 23 — Backup de base de datos automatizado y testeado**

Un backup que nunca has restaurado no es un backup; es una ilusión. Automatiza backups de BD con sidecars o CronJobs, y programa restauraciones de prueba mensuales.

```yaml
# Kubernetes CronJob: backup diario de PostgreSQL
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 2 * * *"  # Todos los días a las 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16-alpine
              command:
                - /bin/sh
                - -c
                - |
                  pg_dump -h postgres -U myapp -d myapp | gzip > /backup/myapp-$(date +%Y%m%d).sql.gz
                  aws s3 cp /backup/ s3://myapp-backups/ --recursive
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: password
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-pvc
          restartPolicy: OnFailure
```

### 12.1.8 Monitoreo: Saber Antes de Que Sea Tarde

**Punto 24 — Métricas expuestas en formato Prometheus**

Tus servicios deben exponer métricas en el endpoint `/metrics` en formato Prometheus. No necesitas implementarlo desde cero: cada lenguaje tiene librerías.

```python
# Python: prometheus_client
from prometheus_client import Counter, Histogram, generate_latest

REQUEST_COUNT = Counter('http_requests_total', 'Total requests', ['method', 'endpoint', 'status'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'Request latency', ['method', 'endpoint'])

@app.get("/metrics")
async def metrics():
    return Response(content=generate_latest(), media_type="text/plain")
```

Métricas que TODO servicio debería exponer:

| Métrica                          | Tipo       | Descripción                              |
|----------------------------------|------------|------------------------------------------|
| `http_requests_total`            | Counter    | Requests por método, endpoint, status    |
| `http_request_duration_seconds`  | Histogram  | Latencia de requests                     |
| `http_requests_in_flight`        | Gauge      | Requests concurrentes actuales           |
| `application_errors_total`       | Counter    | Errores de aplicación                    |
| `db_connections_active`          | Gauge      | Conexiones activas a la base de datos    |
| `cache_hit_ratio`                | Gauge      | Ratio de hits de cache                   |

**Punto 25 — Alertas configuradas y probadas**

Monitorear sin alertas es como un detector de humo sin batería. Las alertas deben ser:

- **Accionables:** cada alerta debe tener un runbook asociado.
- **No ruidosas:** si una alerta se dispara 50 veces al día sin consecuencias, la ignorarás.
- **Escalables:** después de N minutos sin acknowledge, escala a otra persona/equipo.

```yaml
# Prometheus: regla de alerta
groups:
  - name: production-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Tasa de errores elevada en {{ $labels.service }}"
          description: "El servicio {{ $labels.service }} tiene {{ $value }} errores/segundo en los últimos 5 minutos."
          runbook_url: "https://wiki.empresa.com/runbooks/high-error-rate"
```

Integra Alertmanager con tu sistema de notificaciones: PagerDuty, OpsGenie, Slack, email. El canal debe ser proporcional a la severidad: una alerta `critical` a las 3 AM debe ir por PagerDuty (llamada telefónica), no por Slack.

---

```text
╔══════════════════════════════════════════════════════════════╗
║       CHECKLIST DE PRODUCCIÓN — HOJA DE VERIFICACIÓN         ║
╠══════════════════════════════════════════════════════════════╣
║  [ ] Tags específicos (no latest)                           ║
║  [ ] Imágenes slim/alpine                                   ║
║  [ ] Imagen escaneada — 0 CRITICALs                         ║
║  [ ] Multi-stage build                                      ║
║  [ ] Dockerfile revisado y optimizado                       ║
║  [ ] USER no root                                           ║
║  [ ] Capabilities: drop ALL                                 ║
║  [ ] Read-only root filesystem                              ║
║  [ ] Secrets externalizados                                 ║
║  [ ] HEALTHCHECK + liveness/readiness probes                ║
║  [ ] Restart policy: unless-stopped/always                  ║
║  [ ] Timeouts + retries + circuit breakers                  ║
║  [ ] Memory limits                                          ║
║  [ ] CPU limits                                             ║
║  [ ] PIDs limit                                             ║
║  [ ] Redes segregadas frontend/backend/data                 ║
║  [ ] TLS everywhere                                         ║
║  [ ] Rate limiting configurado                              ║
║  [ ] Logging a stdout/stderr                                ║
║  [ ] Logs estructurados JSON                                ║
║  [ ] Rotación de logs configurada                           ║
║  [ ] Volúmenes nombrados con backup automático              ║
║  [ ] Backup de BD automatizado + restore testeado           ║
║  [ ] Métricas Prometheus expuestas                          ║
║  [ ] Alertas configuradas + probadas                        ║
╚══════════════════════════════════════════════════════════════╝
```



---

## 12.2 Estrategias de Despliegue en Producción

> *"La forma en que despliegas define la forma en que duermes."*

Desplegar en producción no es un acto único; es una estrategia. La diferencia entre un deploy que pasa desapercibido y uno que hace caer el sistema está en cómo introduces el cambio.

### 12.2.1 Blue/Green Deployment: El Interruptor de Luz

**Concepto**

Blue/Green es la estrategia más simple y robusta. Funciona así:

1. Dos entornos completos: **Blue** (actual, producción) y **Green** (nuevo, idle).
2. Despliegas la nueva versión en Green y la validas completamente.
3. Cambias el tráfico de Blue a Green instantáneamente.
4. Si algo falla, reviertes instantáneamente re-apuntando a Blue.

```
ANTES DEL CUTOVER              DESPUÉS DEL CUTOVER

  Usuarios                        Usuarios
     │                                │
     ▼                                ▼
┌─────────┐                    ┌─────────┐
│ Service │                    │ Service │
│ selector│                    │ selector│
│  color: │                    │  color: │
│  blue   │                    │  green  │
└────┬────┘                    └────┬────┘
     │                                │
     ▼                                ▼
┌─────────────┐                ┌─────────────┐
│ Deployment  │                │ Deployment  │
│ Blue v1.0.0 │  (idle)        │ Green v2.0.0│
└─────────────┘                └─────────────┘
```

**Ejemplo completo en Kubernetes**

```yaml
# ============================================
# Paso 1: Deploy Blue (versión actual)
# ============================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  labels:
    app: myapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: app
          image: myregistry.azurecr.io/myapp:1.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
---
# ============================================
# Paso 2: Service apuntando a Blue
# ============================================
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  type: ClusterIP
  selector:
    app: myapp
    version: blue    # Este es el selector que cambiaremos
  ports:
    - port: 80
      targetPort: 8080
---
# ============================================
# Paso 3: Deploy Green (nueva versión)
# ============================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  labels:
    app: myapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: app
          image: myregistry.azurecr.io/myapp:2.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

**El Cutover: El Momento de la Verdad**

Cuando Green está listo (todos los pods en Ready), cambias el Service:

```bash
# Verifica que Green está healthy
kubectl get pods -l version=green
kubectl logs -l version=green --tail=5

# Smoke test contra Green
kubectl run smoke-test --rm -i --image=curlimages/curl -- curl -s http://app-green:8080/health

# CUTOVER: cambia el selector del Service
kubectl patch service app-service -p '{"spec":{"selector":{"app":"myapp","version":"green"}}}'

# Observa el tráfico
kubectl logs -f -l version=green
```

**Rollback instantáneo:**

```bash
kubectl patch service app-service -p '{"spec":{"selector":{"app":"myapp","version":"blue"}}}'
# Tiempo total: < 1 segundo
```

**Ventajas y Desventajas**

| Ventajas                                  | Desventajas                                   |
|-------------------------------------------|-----------------------------------------------|
| Rollback instantáneo (< 1s)               | Requiere 2x recursos                          |
| Validación completa antes de cutover      | Migraciones de BD deben ser compatibles       |
| Zero downtime en cutover y rollback       | No permite testing gradual con tráfico real   |
| Simple de entender y operar               |                                                |

### 12.2.2 Canary Release: El Arte de la Gradualidad

**Concepto**

Despliegas la nueva versión junto a la actual, pero solo le envías un porcentaje del tráfico. Observas métricas. Si funciona, aumentas gradualmente: 5% → 20% → 50% → 100%.

```
Tráfico: 100%
    │
    ├── 90% ──► Deployment v1.0.0 (stable) ──► Pods v1.0.0
    │
    └── 10% ──► Deployment v1.1.0 (canary) ──► Pods v1.1.0
```

**Ejemplo en Kubernetes con NGINX Ingress**

```yaml
# Deployment Stable
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
  labels:
    app: myapp
    track: stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
        - name: app
          image: myregistry.azurecr.io/myapp:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: VERSION
              value: "1.0.0"
---
# Deployment Canary
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
  labels:
    app: myapp
    track: canary
spec:
  replicas: 1   # 1 de 10 = 10%
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
        - name: app
          image: myregistry.azurecr.io/myapp:1.1.0
          ports:
            - containerPort: 8080
          env:
            - name: VERSION
              value: "1.1.0"
---
# Service compartido
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
---
# Ingress: Canary (10% del tráfico)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-canary
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
    - host: api.midominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
```

**Progresión de Canary**

```bash
# Fase 1: 5%
kubectl scale deployment app-stable --replicas=19
kubectl scale deployment app-canary --replicas=1
kubectl patch ingress app-ingress-canary -p '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"5"}}}'

# Monitorear 24 horas: tasa de errores, latencia P95, uso CPU/memoria, logs

# Fase 2: 20%
kubectl scale deployment app-stable --replicas=8
kubectl scale deployment app-canary --replicas=2
kubectl patch ingress app-ingress-canary -p '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"20"}}}'

# Fase 3: 50%
kubectl scale deployment app-stable --replicas=5
kubectl scale deployment app-canary --replicas=5
kubectl patch ingress app-ingress-canary -p '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"50"}}}'

# Fase 4: 100% — promoción completa
kubectl delete deployment app-stable
kubectl patch deployment app-canary -p '{"metadata":{"labels":{"track":"stable"}}}'
kubectl patch ingress app-ingress-canary -p '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"100"}}}'
```

**Análisis de métricas durante Canary**

Compara métricas cuantitativamente con Prometheus:

```promql
# Compara tasa de errores stable vs canary
(
  rate(http_requests_total{track="canary",status=~"5.."}[5m])
  /
  rate(http_requests_total{track="canary"}[5m])
)
/
(
  rate(http_requests_total{track="stable",status=~"5.."}[5m])
  /
  rate(http_requests_total{track="stable"}[5m])
)
# Si el ratio > 2 → abortar canary
```

```python
def analyze_canary(prometheus_url, stable_track, canary_track):
    query = (
        "(sum(rate(http_requests_total{track=\"" + canary_track + "\",status=~\"5..\"}[5m]))"
        "/ sum(rate(http_requests_total{track=\"" + canary_track + "\"}[5m]))"
        ") / ("
        "sum(rate(http_requests_total{track=\"" + stable_track + "\",status=~\"5..\"}[5m]))"
        "/ sum(rate(http_requests_total{track=\"" + stable_track + "\"}[5m]))"
        ")"
    )
    response = requests.get(f"{prometheus_url}/api/v1/query", params={"query": query})
    result = response.json()
    if result["data"]["result"]:
        error_ratio = float(result["data"]["result"][0]["value"][1])
        if error_ratio > 2.0:
            print(f"ABORTAR: Canary error ratio = {error_ratio}")
            return False
        print(f"Canary OK: error ratio = {error_ratio}")
        return True
    return True
```

**Ventajas y Desventajas**

| Ventajas                                      | Desventajas                                      |
|-----------------------------------------------|--------------------------------------------------|
| Testing con tráfico real, usuarios reales     | Más complejo que Blue/Green                      |
| Impacto limitado en caso de error             | Requiere sistema de métricas robusto             |
| Mayor confianza en la release                 | Requiere disciplina en cada escalón              |

### 12.2.3 A/B Testing: Diferente a Canary

Aunque Canary y A/B Testing usan mecanismos similares, sus objetivos son radicalmente distintos:

|                  | Canary Release                      | A/B Testing                          |
|------------------|-------------------------------------|--------------------------------------|
| **Objetivo**     | Validar estabilidad de nueva versión| Comparar efectividad de variantes    |
| **Criterio**     | Métricas técnicas (errores, latencia)| Métricas de negocio (conversión, engagement) |
| **Duración**     | Horas a días                        | Días a semanas                       |
| **Ruteo**        | Por peso (weight)                   | Por header, cookie, o user ID        |
| **Usuarios**     | Aleatorios                          | Segmentados (beta testers, región)   |

**A/B Testing basado en cookie:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-beta
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-cookie: "beta_tester"
spec:
  rules:
    - host: api.midominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service-beta
                port:
                  number: 80
```

**A/B Testing basado en header HTTP:**

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "x-feature-flag"
    nginx.ingress.kubernetes.io/canary-by-header-value: "new-ui"
```

**Caso de uso: probar nueva UI**

```
Usuarios beta:
  Cookie beta_tester=always → NGINX Ingress → Deploy v2 (nueva UI)

Usuarios normales:
  Sin cookie → NGINX Ingress → Deploy v1 (UI actual)
```

Para granularidad más fina, usa feature flags:

```javascript
const ldClient = LDClient.initialize('sdk-key-xxx', user);
const showNewCheckout = await ldClient.variation('new-checkout-flow', user, false);
if (showNewCheckout) {
    renderNewCheckout();
} else {
    renderOldCheckout();
}
```

### 12.2.4 Rolling Update: El Caballo de Batalla

Rolling update reemplaza los pods gradualmente, manteniendo el servicio disponible.

```
Antes:   [Pod v1] [Pod v1] [Pod v1] → 3/3 running
Step 1:  [Pod v2] [Pod v1] [Pod v1] → 3/3 running
Step 2:  [Pod v2] [Pod v2] [Pod v1] → 3/3 running
Step 3:  [Pod v2] [Pod v2] [Pod v2] → 3/3 running
```

**Kubernetes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myregistry.azurecr.io/myapp:1.1.0
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

| Parámetro         | Significado                             | Recomendación     |
|-------------------|-----------------------------------------|-------------------|
| `maxSurge`        | Pods extra durante el update            | 1 (o 25%)         |
| `maxUnavailable`  | Pods no disponibles durante el update   | 0 (zero-downtime) |

**Docker Swarm:**

```bash
docker service create   --name app --replicas 6   --update-parallelism 2   --update-delay 10s   --update-order start-first   --update-failure-action rollback   myregistry.azurecr.io/myapp:1.1.0
```

**Rollback en Kubernetes:**

```bash
kubectl rollout history deployment/app
kubectl rollout undo deployment/app
kubectl rollout undo deployment/app --to-revision=3
kubectl rollout status deployment/app
```

### 12.2.5 Comparativa: Árbol de Decisión

```
¿Tu nueva versión es backward-compatible?
│
├── NO → Blue/Green Deployment
│        Rollback instantáneo, validación completa pre-cambiar tráfico
│
└── SÍ → ¿Necesitas validar con tráfico real gradual?
         │
         ├── SÍ → Canary Release
         │        5% → 20% → 50% → 100%, monitoreo continuo
         │
         └── NO → ¿Quieres probar variantes con usuarios?
                  │
                  ├── SÍ → A/B Testing
                  │        Segmentación por cookie/header, métricas de negocio
                  │
                  └── NO → Rolling Update
                           Simple, efectivo, maxUnavailable: 0
```


---

## 12.3 Alta Disponibilidad (HA): Que Nada Te Despierte a las 3 AM

> *"Disponibilidad no es que todo funcione bien. Es que cuando algo falla —y fallará— nadie se dé cuenta."*

Alta disponibilidad significa que tu sistema sigue funcionando incluso cuando componentes individuales fallan. No es un producto que compras, sino una propiedad que diseñas en cada capa de tu arquitectura.

### 12.3.1 Réplicas Múltiples: El Principio de Redundancia

Nada puede ser un punto único de fallo (Single Point of Failure, SPOF).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3   # Si un pod muere, los otros 2 sirven
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myapp:1.0.0
```

| Réplicas | Tolerancia a fallos | Costo | Cuándo usarlo            |
|----------|---------------------|-------|--------------------------|
| 1        | Ninguna             | $     | Desarrollo               |
| 2        | 1 fallo             | $$    | Staging, apps no críticas|
| 3        | 1 fallo (quorum)    | $$$   | Producción (mínimo)      |
| 5+       | 2+ fallos           | $$$$  | Críticas, alta demanda   |

### 12.3.2 Pod Anti-Affinity: No Todos los Huevos en la Misma Canasta

Anti-affinity asegura que tus réplicas se distribuyan en nodos diferentes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: api
              topologyKey: kubernetes.io/hostname
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: api
                topologyKey: topology.kubernetes.io/zone
      containers:
        - name: api
          image: myapp:1.0.0
```

| Tipo                                               | Comportamiento                                       |
|----------------------------------------------------|------------------------------------------------------|
| `requiredDuringSchedulingIgnoredDuringExecution`   | Obligatorio. Si no se cumple, el pod no se ejecuta   |
| `preferredDuringSchedulingIgnoredDuringExecution`  | Preferencia. El scheduler lo intenta (soft)          |

Topology keys:

| Key                                     | Significado                       |
|-----------------------------------------|-----------------------------------|
| `kubernetes.io/hostname`                | Distribuir en nodos diferentes    |
| `topology.kubernetes.io/zone`           | Distribuir en zonas de AZ         |
| `topology.kubernetes.io/region`         | Distribuir en regiones            |

### 12.3.3 Topología Multi-AZ y Multi-Region

La disponibilidad se diseña en capas geográficas:

```
Region: us-east-1
├── AZ: us-east-1a
│   ├── Node 1 → Pod A (réplica 1)
│   └── Node 2 → Pod B
├── AZ: us-east-1b
│   ├── Node 3 → Pod A (réplica 2)
│   └── Node 4 → Pod B
└── AZ: us-east-1c
    ├── Node 5 → Pod A (réplica 3)
    └── Node 6 → PostgreSQL réplica

Region: eu-west-1 (DR)
├── AZ: eu-west-1a → Cluster standby
└── AZ: eu-west-1b → Réplica de base de datos
```

### 12.3.4 Load Balancing: Externo e Interno

```
Internet
   │
   ▼
┌──────────────────────┐
│ Cloud Load Balancer  │  ← AWS ALB/NLB, GCP LB, Azure LB
│ (capa 7: TLS, ruteo) │
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐
│ Kubernetes Ingress   │  ← NGINX, Traefik, Istio Gateway
│ Controller           │
└─────────┬────────────┘
          │
          ▼
   ┌─────────────┐
   │ Service      │  ← ClusterIP (balanceo interno capa 4)
   │ (kube-proxy) │
   └──────┬───────┘
          │
          ▼
   ┌─────────────┐
   │ Pods         │  ← Endpoints reales
   └──────────────┘
```

**Service Mesh: balanceo capa 7 interno con Istio:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-vs
spec:
  hosts:
    - api
  http:
    - route:
        - destination:
            host: api
            subset: v1
          weight: 90
        - destination:
            host: api
            subset: v2
          weight: 10
      retries:
        attempts: 3
        perTryTimeout: 2s
      timeout: 10s
```

### 12.3.5 Failover Automático: El Sistema que se Cura Solo

Kubernetes implementa failover en varios niveles:

**Nivel 1: Self-healing de pods** — Kubernetes mantiene el número deseado de réplicas automáticamente.

**Nivel 2: Liveness probes:**

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5
```

**Nivel 3: Readiness probes:**

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
```

Cuando una readiness probe falla, Kubernetes retira el pod del Service (deja de enviarle tráfico) pero NO lo reinicia.

**Nivel 4: Node failure** — El scheduler reubica pods en nodos disponibles.

**Nivel 5: Cluster failure (DR)** — Entra en juego la estrategia de Disaster Recovery (sección 12.6).

### 12.3.6 Patrones de HA para Bases de Datos

**PostgreSQL con CloudNativePG:**

```
┌──────────────────────────┐
│ PostgreSQL Primary       │  ← Lee/Escribe
│ (StatefulSet, pod 0)     │
└──────────┬───────────────┘
           │ replicación streaming
           ▼
┌──────────────────────────┐
│ PostgreSQL Replica 1     │  ← Solo lectura
│ (StatefulSet, pod 1)     │
└──────────────────────────┘
           │
           ▼
┌──────────────────────────┐
│ PostgreSQL Replica 2     │  ← Solo lectura + failover target
│ (StatefulSet, pod 2)     │
└──────────────────────────┘
```

**Redis con StatefulSet y Sentinel:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

---

## 12.4 Auto-Escalado: Elasticidad que No Duerme

> *"La diferencia entre una aplicación que escala y una que se cae es que la primera sabe cuándo pedir ayuda."*

### 12.4.1 Horizontal Pod Autoscaler (HPA)

El HPA aumenta o disminuye réplicas basado en CPU, memoria, o métricas personalizadas.

```bash
kubectl autoscale deployment api --cpu-percent=70 --min=2 --max=10
```

Equivalente en YAML:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
      selectPolicy: Max
```

**Fórmula del HPA (simplificada):**

```
desiredReplicas = ceil(currentReplicas * (currentMetricValue / targetMetricValue))
```

Ejemplo: 2 réplicas, CPU actual = 90%, target = 70%.
`desiredReplicas = ceil(2 * (90/70)) = ceil(2.57) = 3`

**Métricas personalizadas con Prometheus Adapter:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

### 12.4.2 Cluster Autoscaler

El HPA escala pods; el Cluster Autoscaler escala nodos cuando los pods no caben.

**Flujo completo de auto-escalado:**

```
Aumenta el tráfico
    │
    ▼
Métricas superan umbral (HPA)
    │
    ▼
HPA escala Deployment → nuevos pods en Pending
    │
    ▼
Cluster Autoscaler detecta pods pending → añade nodo(s)
    │
    ▼
Nuevos pods se programan en el nuevo nodo
    │
    ▼
Tráfico se distribuye entre todos los pods
    │
    ▼ (cuando el tráfico baja)
HPA reduce réplicas → Cluster Autoscaler drena y elimina nodos vacíos
```

**Pod Disruption Budget (PDB):**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
```

### 12.4.3 KEDA: Event-Driven Autoscaling

KEDA escala pods basado en eventos externos (colas de mensajes, streams).

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaler
spec:
  scaleTargetRef:
    name: worker-deployment
  minReplicaCount: 0    # Escalar a CERO cuando no hay trabajo
  maxReplicaCount: 20
  triggers:
    - type: rabbitmq
      metadata:
        host: amqp://rabbitmq.default.svc.cluster.local
        queueName: orders
        queueLength: "10"   # 1 pod por cada 10 mensajes en cola
```

**Triggers soportados por KEDA:**

| Trigger       | Caso de uso                         |
|---------------|-------------------------------------|
| Kafka         | Consumidores basados en lag         |
| RabbitMQ      | Workers de colas de mensajes        |
| AWS SQS       | Procesamiento de mensajes SQS       |
| Prometheus    | Cualquier métrica de Prometheus     |
| Cron          | Escalar en horarios específicos     |
| Redis Lists   | Procesamiento de listas Redis       |

**KEDA con métricas de Prometheus:**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-prometheus-scaler
spec:
  scaleTargetRef:
    name: api
  minReplicaCount: 2
  maxReplicaCount: 50
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-server.monitoring.svc:9090
        metricName: http_requests_in_flight
        query: sum(rate(http_requests_total{app="api"}[2m]))
        threshold: "100"
```

### 12.4.4 Vertical Pod Autoscaler (VPA)

Mientras HPA escala horizontalmente, VPA escala verticalmente (más CPU/memoria al mismo pod).

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
        controlledResources: ["cpu", "memory"]
```

```
┌──────────────────────────────────────────────────────────────┐
│                   CAPAS DE AUTO-ESCALADO                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   HPA (Horizontal Pod Autoscaler)                            │
│   "Más pods cuando sube CPU"                                 │
│                                                              │
│   Cluster Autoscaler                                         │
│   "Más nodos cuando no caben más pods"                        │
│                                                              │
│   KEDA (Event-Driven Autoscaler)                             │
│   "Más pods cuando hay mensajes en la cola"                  │
│                                                              │
│   VPA (Vertical Pod Autoscaler)                              │
│   "Pods más grandes cuando necesitan más recursos"            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 12.5 Configuración Externalizada: El Manifiesto 12-Factor

> *"La configuración que varía entre entornos debe estar estrictamente separada del código."*
>
> — 12-Factor App, Factor III

El manifiesto 12-Factor App es el estándar para aplicaciones cloud-native. La configuración debe vivir fuera del contenedor, en el entorno de ejecución.

### 12.5.1 Environment Variables: La Puerta de Entrada

```python
# MAL: hardcodear en el código
DATABASE_URL = "postgresql://user:pass@localhost:5432/mydb"

# BIEN: leer de variable de entorno
DATABASE_URL = os.environ["DATABASE_URL"]
```

Pero no basta con usar variables de entorno. Hay que externalizarlas del contenedor.

### 12.5.2 ConfigMaps: Configuración No Sensible

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  CACHE_TTL: "3600"
  config.json: |
    {
      "features": {
        "new_checkout": false,
        "beta_ui": false
      },
      "rate_limits": {
        "api": 100,
        "upload": 10
      }
    }
```

Montar ConfigMap como archivo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: api
          image: myapp:1.0.0
          envFrom:
            - configMapRef:
                name: app-config
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
              readOnly: true
      volumes:
        - name: config-volume
          configMap:
            name: app-config
            items:
              - key: config.json
                path: config.json
```

### 12.5.3 Secrets: Configuración Sensible

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: "m1C0ntr4s3n4Sup3rS3gur4!"
  REDIS_PASSWORD: "0tr4C0ntr4s3N4"
  JWT_SECRET: "clave-jwt-de-256-bits-minimo"
```

Montar Secrets como archivos (recomendado para producción):

```yaml
containers:
  - name: api
    volumeMounts:
      - name: secrets-volume
        mountPath: /etc/secrets
        readOnly: true
volumes:
  - name: secrets-volume
    secret:
      secretName: app-secrets
```

En tu código:

```python
def get_secret(name):
    try:
        with open(f"/etc/secrets/{name}") as f:
            return f.read().strip()
    except FileNotFoundError:
        return os.environ.get(name)

DATABASE_PASSWORD = get_secret("DATABASE_PASSWORD")
```

### 12.5.4 Gestión Avanzada de Secretos

**Sealed Secrets (Bitnami):** Secrets encriptados para Git:

```bash
kubectl create secret generic db-credentials   --from-literal=password=supersecret   --dry-run=client -o yaml |   kubeseal --format=yaml > db-credentials-sealed.yaml
# db-credentials-sealed.yaml es seguro para comitear
```

**External Secrets Operator:** sincroniza secretos desde AWS/GCP/Azure a Kubernetes:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-external-secrets
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: aws-secretsmanager-backend
    kind: ClusterSecretStore
  target:
    name: app-secrets
  data:
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: prod/myapp/database
        property: password
```

### 12.5.5 Hot Reload de Configuración

**Opción 1: Reloader (Stakater)**

```yaml
# Anotar el Deployment para auto-restart cuando el ConfigMap cambie
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```

**Opción 2: Aplicación que recarga automáticamente**

```python
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class ConfigReloadHandler(FileSystemEventHandler):
    def on_modified(self, event):
        if event.src_path == "/app/config/config.json":
            with open(event.src_path) as f:
                app.state.config = json.load(f)
            logger.info("Configuración recargada")
```

### 12.5.6 Configuración por Entorno: El Patrón de Directorios

```
config/
├── base.yaml           # Común a todos los entornos
├── development.yaml    # Sobrescritura para desarrollo
├── staging.yaml        # Sobrescritura para staging
└── production.yaml     # Sobrescritura para producción
```

```yaml
# base.yaml
app:
  name: myapp
  port: 8080
  log_level: info
database:
  pool_size: 10
  timeout: 30

# production.yaml (sobrescribe base.yaml)
app:
  log_level: warn
database:
  pool_size: 50
  timeout: 10
  ssl: true
```

Fusión de archivos en entrypoint:

```bash
#!/bin/sh
ENV=${APP_ENV:-development}
CONFIG_DIR=/app/config

yq eval-all 'select(fileIndex == 0) * select(fileIndex == 1)'   ${CONFIG_DIR}/base.yaml ${CONFIG_DIR}/${ENV}.yaml > /app/config/merged.yaml

exec /app/server --config /app/config/merged.yaml
```


---

## 12.6 Backup y Disaster Recovery: Cuando el Plan A Falla

> *"Hay dos tipos de empresas: las que han tenido un incidente de pérdida de datos y las que lo tendrán."*

### 12.6.1 La Estrategia 3-2-1

```
3 COPIAS totales de los datos
2 MEDIOS diferentes de almacenamiento
1 COPIA fuera del sitio (off-site)
```

Ejemplo práctico:

1. **Copia 1:** Volumen de producción (EBS, PersistentVolume).
2. **Copia 2:** Snapshot diario en la misma región (S3).
3. **Copia 3:** Réplica en otra región geográfica.

### 12.6.2 Backup de Volúmenes en Kubernetes

**Volume Snapshots (CSI):**

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-data-snapshot-20240315
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: postgres-data
```

**CronJob de snapshots:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: volume-snapshotter
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-sa
          containers:
            - name: snapshotter
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  TIMESTAMP=$(date +%Y%m%d-%H%M%S)
                  cat <<EOF | kubectl apply -f -
                  apiVersion: snapshot.storage.k8s.io/v1
                  kind: VolumeSnapshot
                  metadata:
                    name: postgres-snap-${TIMESTAMP}
                  spec:
                    volumeSnapshotClassName: ebs-snapshot-class
                    source:
                      persistentVolumeClaimName: postgres-data
                  EOF
                  echo "Snapshot postgres-snap-${TIMESTAMP} creado"
          restartPolicy: OnFailure
```

### 12.6.3 Backup de Bases de Datos

**PostgreSQL: Backup lógico con pg_dump:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16-alpine
              env:
                - name: PGHOST
                  value: postgres.default.svc.cluster.local
                - name: PGUSER
                  value: myapp
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: password
              command:
                - /bin/sh
                - -c
                - |
                  set -e
                  DATE=$(date +%Y%m%d-%H%M%S)
                  FILE="backup-${DATE}.sql.gz"
                  echo "Iniciando backup: ${FILE}"
                  pg_dump -d myapp --no-owner --no-acl | gzip > /backups/${FILE}
                  echo "Subiendo a S3..."
                  aws s3 cp /backups/${FILE} s3://myapp-backups/prod/daily/${FILE} --sse AES256
                  echo "Limpiando backups locales > 7 días..."
                  find /backups -name "*.sql.gz" -mtime +7 -delete
                  echo "Backup completado: ${FILE}"
              volumeMounts:
                - name: backup-storage
                  mountPath: /backups
          volumes:
            - name: backup-storage
              emptyDir: {}
          restartPolicy: OnFailure
```

**Política de retención:**

```
- Backups diarios: retener 30 días
- Backups semanales: retener 12 semanas
- Backups mensuales: retener 12 meses
- Backups anuales: retener 7 años
```

S3 Lifecycle Policy para automatizar retención:

```json
{
  "Rules": [
    {
      "Id": "delete-old-daily",
      "Status": "Enabled",
      "Filter": { "Prefix": "prod/daily/" },
      "Expiration": { "Days": 30 }
    },
    {
      "Id": "transition-to-glacier",
      "Status": "Enabled",
      "Filter": { "Prefix": "prod/monthly/" },
      "Transitions": [{ "Days": 30, "StorageClass": "GLACIER" }]
    }
  ]
}
```

### 12.6.4 Simulacro de Restauración

> *"Un plan de DR que nunca has probado no es un plan. Es una fantasía."*

**Script de restore de PostgreSQL:**

```bash
#!/bin/bash
# restore-database.sh — Probado y documentado
set -e

BACKUP_FILE=${1:?"Uso: $0 <backup-file.sql.gz>"}
TARGET_HOST=${PGHOST:-localhost}
TARGET_DB=${PGDATABASE:-myapp_restore_test}

echo "=== SIMULACRO DE RESTAURACIÓN ==="
echo "Fecha: $(date)"

# 1. Descargar backup de S3
echo "[1/5] Descargando backup..."
aws s3 cp s3://myapp-backups/${BACKUP_FILE} /tmp/${BACKUP_FILE}

# 2. Verificar integridad
echo "[2/5] Verificando integridad..."
gunzip -t /tmp/${BACKUP_FILE}
SIZE=$(stat -f%z /tmp/${BACKUP_FILE} 2>/dev/null || stat -c%s /tmp/${BACKUP_FILE})
echo "  Tamaño: ${SIZE} bytes — OK"

# 3. Crear base de datos de prueba
echo "[3/5] Creando base de datos de prueba..."
dropdb --if-exists -h ${TARGET_HOST} ${TARGET_DB} 2>/dev/null || true
createdb -h ${TARGET_HOST} ${TARGET_DB}

# 4. Restaurar
echo "[4/5] Restaurando..."
gunzip -c /tmp/${BACKUP_FILE} | psql -h ${TARGET_HOST} -d ${TARGET_DB} -v ON_ERROR_STOP=1

# 5. Verificar
echo "[5/5] Verificando..."
TABLE_COUNT=$(psql -h ${TARGET_HOST} -d ${TARGET_DB} -t -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';")
echo "  Tablas restauradas: ${TABLE_COUNT}"

echo "=== RESTAURACIÓN COMPLETADA (${SECONDS}s) ==="
rm /tmp/${BACKUP_FILE}
```

**Programa de simulacros:**

| Frecuencia     | Actividad                                             |
|----------------|-------------------------------------------------------|
| **Mensual**    | Restaurar último backup en entorno de prueba          |
| **Trimestral** | Restaurar en región/AZ diferente (DR test real)       |
| **Anual**      | Simulacro completo: failover a región standby         |

### 12.6.5 RPO y RTO: Las Dos Métricas del DR

```
RPO (Recovery Point Objective):
  ¿Cuántos datos PUEDO perder?
  |--- RPO ---| (datos perdidos) |--- backup ---|
  Último backup     Incidente      Ahora
  Ejemplo: RPO = 1 hora → backups cada hora

RTO (Recovery Time Objective):
  ¿Cuánto tiempo PUEDO estar caído?
  |Incidente|--- RTO ---|Recuperado|
  Ejemplo: RTO = 15 minutos → automatización crítica
```

Matriz de RPO/RTO por criticidad:

| Sistema        | RPO      | RTO      | Estrategia                              |
|----------------|----------|----------|-----------------------------------------|
| Transacciones  | 0        | < 5 min  | Replicación sincrónica + failover auto  |
| Base de datos  | < 1 hora | < 30 min | Backups horarios + restore automatizado |
| Archivos       | < 24h    | < 4h     | Snapshots diarios + restore manual      |
| Logs/Analytics | < 7 días | < 24h    | Replicación asíncrona                   |

---

## 12.7 Estrategia de Versionado de Imágenes

> *"Nombrar cosas es uno de los dos problemas difíciles en computación."*
> — Dicho popular

### 12.7.1 Versionado Semántico (SemVer)

```
app:1.2.3
│   │ │
│   │ └── PATCH: bug fixes, seguridad, cambios backward-compatibles
│   └──── MINOR: nueva funcionalidad, backward-compatible
└──────── MAJOR: cambios incompatibles
```

```bash
docker build -t myapp:1.2.3 .
docker tag myapp:1.2.3 myapp:1.2
docker tag myapp:1.2.3 myapp:1
docker tag myapp:1.2.3 myapp:latest  # Solo en desarrollo
```

### 12.7.2 Tags por Entorno

```bash
docker build -t myapp:dev-abc123 .
docker build -t myapp:staging-1.2.3 .
docker build -t myapp:prod-1.2.3 .
```

| Tag                | Entorno  | Contenido                          |
|--------------------|----------|------------------------------------|
| `app:dev-abc123`   | Dev      | Rama feat, commit abc123           |
| `app:staging-1.2.3`| Staging  | Rama main, tag 1.2.3               |
| `app:prod-1.2.3`   | Prod     | Misma imagen que staging           |

La misma imagen viaja por los entornos. Lo que pruebas en staging es EXACTAMENTE lo que va a producción.

### 12.7.3 Git Commit SHA

```bash
GIT_SHA=$(git rev-parse --short HEAD)
docker build -t myapp:git-${GIT_SHA} .
```

### 12.7.4 Estrategia Combinada (Recomendada)

```bash
VERSION=1.2.3
GIT_SHA=$(git rev-parse --short HEAD)
BUILD_DATE=$(date -u +%Y%m%d-%H%M%S)

docker build   --label "org.opencontainers.image.version=${VERSION}"   --label "org.opencontainers.image.revision=${GIT_SHA}"   --label "org.opencontainers.image.created=${BUILD_DATE}"   -t myapp:${VERSION}   -t myapp:${VERSION}-${GIT_SHA}   -t myapp:prod-${VERSION}   .
```

Labels OCI estándar:

| Label                                        | Propósito              |
|----------------------------------------------|------------------------|
| `org.opencontainers.image.version`           | Versión semántica      |
| `org.opencontainers.image.revision`          | Git commit SHA         |
| `org.opencontainers.image.created`           | Fecha de build         |
| `org.opencontainers.image.source`            | URL del repositorio    |

### 12.7.5 Inmutabilidad de Tags

> **Regla de oro:** una vez que un tag está en producción, nunca se reasigna. Nunca.

Si encuentras un bug en `app:1.2.3`, crea `app:1.2.4`. La inmutabilidad hace posible auditoría, rollback y debugging.

---

## 12.8 Actualización de Imágenes Base: La Cadena de Suministro

> *"Tu imagen es tan segura como su imagen base más vulnerable."*

### 12.8.1 El Problema de las Imágenes Base

Tu Dockerfile comienza con `FROM node:20-alpine`. Esa imagen base contiene cientos de paquetes. Cada mes se descubren nuevas CVEs.

```
Tu imagen (app:1.2.3)
  └── FROM node:20-alpine (marzo 2024)
        └── libssl3: CVE-2024-XXXX
        └── libcrypto3: CVE-2024-YYYY
```

### 12.8.2 Herramientas de Monitoreo de CVEs

**Docker Scout:**

```bash
docker scout quickview myapp:1.2.3
docker scout recommendations myapp:1.2.3
docker scout cves myapp:1.2.3 --exit-code --only-severity critical,high
```

**Trivy (open source):**

```bash
trivy image myapp:1.2.3
trivy image --severity CRITICAL,HIGH --format json -o trivy-report.json myapp:1.2.3
trivy config ./deploy/   # Escanear manifiestos Kubernetes
```

**Snyk:**

```bash
snyk container test myapp:1.2.3
snyk container monitor myapp:1.2.3
```

### 12.8.3 Automatización con Renovate / Dependabot

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "docker": {
    "fileMatch": ["Dockerfile", "Dockerfile.*"],
    "pinDigests": true
  },
  "packageRules": [{
    "description": "Auto-merge patch updates for docker images",
    "matchDatasources": ["docker"],
    "matchUpdateTypes": ["patch", "digest"],
    "automerge": true
  }]
}
```

### 12.8.4 Pipeline de Rebuild Automático

```yaml
# .github/workflows/base-image-update.yml
name: Rebuild on base image update
on:
  schedule:
    - cron: '0 2 * * 1'  # Lunes 2 AM UTC
jobs:
  check-and-rebuild:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Pull latest base images
        run: |
          docker pull node:20-alpine
          docker pull alpine:3.20
      - name: Build
        run: docker build -t myapp:ci-${GITHUB_SHA} .
      - name: Scan
        run: docker scout cves myapp:ci-${GITHUB_SHA} --exit-code --only-severity critical,high
      - name: Test
        run: docker compose -f docker-compose.test.yml up --exit-code-from test
      - name: Push
        if: success()
        run: |
          docker tag myapp:ci-${GITHUB_SHA} myapp:prod-$(date +%Y%m%d)
          docker push myapp:prod-$(date +%Y%m%d)
```

### 12.8.5 Probar en Staging Antes de Producción

```
Detectar nueva versión de imagen base (Renovate)
    │
    ▼
PR automático → CI build & test → merge a main
    │
    ▼
Build en CI/CD → push a registry con tag staging
    │
    ▼
Desplegar en staging → smoke tests → monitoreo 24-48h
    │
    ▼
Desplegar en producción (rolling o canary)
```

---

## 12.9 Infraestructura como Código (IaC): El Último Eslabón

> *"Trata tu infraestructura como tratas tu código: versionada, testeada, revisada."*

### 12.9.1 Terraform + Kubernetes Provider

Terraform puede gestionar toda tu infraestructura, incluidos los recursos dentro de Kubernetes:

```hcl
# main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
    kubernetes = { source = "hashicorp/kubernetes", version = "~> 2.20" }
  }
}

# Provisionar cluster EKS
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"
  cluster_name    = "myapp-prod"
  cluster_version = "1.28"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  node_groups = {
    main = {
      desired_capacity = 3
      max_capacity     = 10
      min_capacity     = 2
      instance_types = ["t3.medium"]
    }
    spot = {
      desired_capacity = 2
      max_capacity     = 20
      min_capacity     = 0
      instance_types = ["t3.medium"]
      capacity_type  = "SPOT"
    }
  }
}

# Desplegar aplicación via Terraform en Kubernetes
resource "kubernetes_deployment" "api" {
  metadata {
    name      = "api"
    namespace = kubernetes_namespace.app.metadata[0].name
  }
  spec {
    replicas = 3
    selector {
      match_labels = { app = "api" }
    }
    template {
      metadata {
        labels = { app = "api" }
      }
      spec {
        container {
          name  = "api"
          image = "myregistry.azurecr.io/myapp:${var.app_version}"
          port { container_port = 8080 }
          resources {
            limits   = { cpu = "500m", memory = "512Mi" }
            requests = { cpu = "250m", memory = "256Mi" }
          }
        }
      }
    }
  }
}
```

**Terraform State remoto (IMPRESCINDIBLE):**

```hcl
terraform {
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### 12.9.2 Helm: El Package Manager de Kubernetes

Helm empaqueta tu aplicación completa en un chart versionado:

```bash
helm create myapp-chart
```

Estructura resultante:

```
myapp-chart/
├── Chart.yaml          # Metadata (nombre, versión)
├── values.yaml         # Valores por defecto
├── values/
│   ├── dev.yaml        # Override para dev
│   ├── staging.yaml    # Override para staging
│   └── prod.yaml       # Override para prod
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl
└── .helmignore
```

**values.yaml:**

```yaml
replicaCount: 2
image:
  repository: myregistry.azurecr.io/myapp
  tag: "latest"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
  targetPort: 8080
resources:
  limits: { cpu: 500m, memory: 512Mi }
  requests: { cpu: 250m, memory: 256Mi }
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
config:
  logLevel: info
```

**values/prod.yaml:**

```yaml
replicaCount: 6
image:
  tag: "prod-1.2.3"
  pullPolicy: Always
ingress:
  enabled: true
  hosts:
    - host: api.midominio.com
  tls:
    - hosts: [api.midominio.com]
      secretName: api-tls-secret
resources:
  limits: { cpu: 2000m, memory: 2Gi }
  requests: { cpu: 1000m, memory: 1Gi }
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
config:
  logLevel: warn
```

Instalar/actualizar:

```bash
helm upgrade --install myapp ./myapp-chart   --namespace production --create-namespace   --values values/prod.yaml

helm rollback myapp --namespace production
helm list --namespace production
```

### 12.9.3 GitOps con ArgoCD: La Verdad en Git

GitOps lleva IaC a su conclusión lógica: Git es la fuente única de verdad. ArgoCD sincroniza continuamente el cluster con Git.

```
┌─────────────────────────────────────────────────────────┐
│                    FLUJO GITOPS                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Developer push a main                                │
│          │                                              │
│          ▼                                              │
│   CI: build → test → push image → update Git            │
│          │                                              │
│          ▼                                              │
│   ArgoCD detecta cambio en Git                          │
│          │                                              │
│          ▼                                              │
│   ArgoCD sincroniza cluster con Git                     │
│          │                                              │
│          ▼                                              │
│   Cluster converge a desired state                      │
│                                                         │
│   Si alguien hace kubectl edit, ArgoCD lo revierte      │
│   en 3 minutos (self-heal).                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**ArgoCD Application:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mi-empresa/myapp-deploy
    targetRevision: main
    path: overlays/prod
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Estructura de repositorio GitOps:**

```
myapp-deploy/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   ├── staging/
│   └── prod/
│       ├── kustomization.yaml
│       └── patches/
│           ├── replicas-patch.yaml
│           └── resources-patch.yaml
└── charts/
    └── myapp/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```


---

## 12.10 Casos de Estudio Reales: De la Teoría a la Trinchera

> *"La teoría te enseña qué hacer. Los casos reales te enseñan qué NO hacer —que es mucho más valioso."*

### 12.10.1 Caso 1: Startup SaaS — Del Monolito Django a Microservicios

**Contexto**

*PhotoReview*, startup SaaS de edición fotográfica colaborativa. Dos años de desarrollo, 5,000 usuarios activos, 1 desarrollador backend (el CTO).

**Estado Inicial: El Monolito**

```
DigitalOcean Droplet ($40/mes)
└── Django 2.2 monolítico
    ├── Procesamiento de imágenes (Pillow)
    ├── API REST
    ├── Autenticación
    ├── Notificaciones (email)
    └── Tareas programadas (cron)
```

Problemas:
- Deploy manual: SSH + git pull + `systemctl restart gunicorn` + rezar.
- Si Pillow crasheaba procesando una imagen RAW grande, caía toda la API.
- El cron job nocturno consumía toda la RAM y mataba Gunicorn.
- "Funciona en máquina de desarrollo" era una frase diaria.

**Fase 1: Containerizar (3 semanas)**

```dockerfile
# Dockerfile inicial
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["gunicorn", "photoreview.wsgi:application", "--bind", "0.0.0.0:8000"]
```

```yaml
# docker-compose.yml — Desarrollo local
services:
  db:
    image: postgres:13
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: photoreview
      POSTGRES_PASSWORD: devpassword
  redis:
    image: redis:6-alpine
  app:
    build: .
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    depends_on: [db, redis]
    environment:
      DATABASE_URL: postgresql://postgres:devpassword@db:5432/photoreview
  worker:
    build: .
    command: celery -A photoreview worker -l info
    depends_on: [redis]
volumes:
  pgdata:
```

Resultado: todos los desarrolladores trabajan en entornos idénticos. Adiós "funciona en mi máquina".

**Fase 2: Separar responsabilidades (3 semanas)**

El monolito se partió por dominio:

```
Servicios de PhotoReview:
├── api/              # API REST (Django REST Framework)
├── processor/        # Procesamiento de imágenes (Flask + Pillow)
├── auth/             # Autenticación (Django + JWT)
├── notifications/    # Notificaciones (Python + Celery)
├── scheduler/        # Tareas programadas (Celery Beat)
├── db/               # PostgreSQL 13
├── cache/            # Redis
└── queue/            # RabbitMQ
```

```yaml
# docker-compose.prod.yml — Producción en Swarm
services:
  api:
    image: ${REGISTRY}/api:${VERSION}
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 512M
  processor:
    image: ${REGISTRY}/processor:${VERSION}
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 2G    # Procesamiento requiere más RAM
  auth:
    image: ${REGISTRY}/auth:${VERSION}
    deploy:
      replicas: 2
    secrets:
      - jwt_secret
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    configs:
      - source: nginx_conf
        target: /etc/nginx/nginx.conf
```

**Fase 3: Migración a Kubernetes (4 semanas)**

Con 50,000 usuarios y rondas de inversión, Swarm se quedó corto. Necesitaban auto-escalado, blue/green deploys, y monitoreo avanzado.

Arquitectura final en Kubernetes:

```
┌─────────────────────────────────────────────────────────┐
│                  PhotoReview Kubernetes                 │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Ingress (NGINX + cert-manager)                         │
│      │                                                  │
│      ▼                                                  │
│  api (3 réplicas, HPA: 2-10, target CPU 70%)           │
│      │                                                  │
│      ├──► processor (HPA + KEDA por cola RabbitMQ)      │
│      │    └── Storage: S3 para imágenes procesadas      │
│      │                                                  │
│      ├──► auth (2 réplicas, cache Redis interno)        │
│      │                                                  │
│      └──► notifications (Celery workers, 2-5 réplicas)  │
│                                                         │
│  Bases de datos:                                        │
│  ├── PostgreSQL (CloudNativePG, 1 primary + 2 replicas) │
│  └── Redis Cluster (6 nodos, 3 masters + 3 replicas)   │
│                                                         │
│  Monitoreo: Prometheus + Grafana + Loki                 │
│  CI/CD: GitHub Actions + ArgoCD (GitOps)                │
│  Secretos: External Secrets Operator + AWS Secrets Mgr  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Lecciones Aprendidas (PhotoReview):**

1. **No empieces con Kubernetes.** Perdieron 2 semanas intentando configurar K8s desde el día 1. Docker Compose → Swarm → K8s es un camino más seguro.
2. **Los volúmenes de datos son el mayor desafío.** Migrar bases de datos entre entornos fue lo más doloroso. Siempre ten un backup antes de migrar.
3. **Monitoreo desde el día 1.** Tuvieron un incidente de 4 horas que no detectaron hasta que los usuarios se quejaron en Twitter. Prometheus + Grafana se volvió prioridad #1.
4. **La cultura DevOps > herramientas.** El CTO pasó de "el que hace los deploys" a "el que habilita al equipo para hacer deploys". La transición cultural llevó más tiempo que la técnica.
5. **Start simple.** No implementaron todos los microservicios de golpe. Separaron primero el procesamiento de imágenes (el mayor cuello de botella) y mantuvieron el resto como monolito hasta que fue necesario.

### 12.10.2 Caso 2: Empresa Tradicional Migrando a Contenedores

**Contexto**

*SegurosConfianza*, aseguradora con 30 años de historia. Stack: Java 8, WebLogic, Oracle DB, despliegues manuales con bash. Equipo de 40 desarrolladores.

**Estado Inicial: El Mundo Pre-Docker**

```
Proceso de deploy (2 semanas):
  Lunes:     Reunión de planificación del deploy.
  Martes:    Preparar script (copia de scripts anteriores).
  Miércoles: Deploy a pre-producción (entorno compartido).
  Jueves:    Test manual. Encontrar 487 diferencias con producción.
  Viernes:   Arreglar diferencias.
  --- Fin de semana (deploy NO en viernes) ---
  Lunes:     Re-test en pre-producción.
  Martes:    Ventana de deploy: 2 AM - 6 AM.
             (Si algo falla, rollback = restaurar snapshot de VM).
```

**Fase 1: Containerizar la Aplicación Existente (3 meses)**

Objetivo: encapsular lo existente, no reescribir.

```dockerfile
# Primera iteración: Java 8 + WebLogic en contenedor
FROM container-registry.oracle.com/middleware/weblogic:14.1.1.0-dev
COPY config/domain.properties /u01/oracle/properties/
COPY target/app.war /u01/oracle/applications/
USER oracle
CMD ["/u01/oracle/user_projects/domains/base_domain/startWebLogic.sh"]
```

**Fase 2: CI/CD Pipeline Inicial (2 meses)**

```groovy
// Jenkinsfile — Primer pipeline
pipeline {
    agent any
    environment {
        REGISTRY = 'harbor.segurosconfianza.com'
        IMAGE = "${REGISTRY}/app"
    }
    stages {
        stage('Checkout') { steps { checkout scm } }
        stage('Build') {
            steps {
                sh 'mvn clean package'
                sh "docker build -t ${IMAGE}:${BUILD_NUMBER} ."
            }
        }
        stage('Test') {
            steps { sh 'docker compose -f docker-compose.test.yml up --exit-code-from test' }
        }
        stage('Push') { steps { sh "docker push ${IMAGE}:${BUILD_NUMBER}" } }
        stage('Deploy to Pre') {
            when { branch 'develop' }
            steps { sh "ssh deploy@pre 'docker pull ${IMAGE}:${BUILD_NUMBER} && docker compose up -d'" }
        }
        stage('Deploy to Prod') {
            when { branch 'main' }
            steps {
                input message: 'Desplegar a producción?', ok: 'Sí'
                sh "ssh deploy@prod 'docker pull ${IMAGE}:${BUILD_NUMBER} && docker compose up -d'"
            }
        }
    }
}
```

**Fase 3: Kubernetes + GitOps (6 meses)**

Estrategia: modernizar sin interrumpir el negocio — **Strangler Fig Pattern.**

```
Strangler Fig Pattern:

Antes:
  [ Monolito VM: 100% funcionalidad ]

Fase 1 (mes 1-3):
  [ Monolito VM: 95% ]
  [ K8s: Portal clientes 5% ]

Fase 2 (mes 4-6):
  [ Monolito VM: 70% ]
  [ K8s: Portal, cotizaciones 30% ]

Fase 3 (mes 7-9):
  [ Monolito VM: 40% ]
  [ K8s: Portal, cotizaciones, siniestros 60% ]

Fase Final (mes 10-12):
  [ Monolito VM: 0% → APAGADO ]
  [ K8s: 100% funcionalidad ]
```

**Desafíos y Soluciones:**

| Desafío                                 | Solución                                           |
|-----------------------------------------|----------------------------------------------------|
| Equipo sin experiencia en contenedores  | Capacitación: 2h/semana durante 3 meses            |
| Resistencia cultural                    | Demostrar con hechos: deploy de 2 semanas → 2 horas|
| Oracle DB en contenedor                 | Se mantuvo en VM dedicada (decisión pragmática)    |
| Cumplimiento regulatorio (seguros)      | Auditoría de seguridad + escaneo CVE continuo      |
| Falta de monitoreo                      | Prometheus + Grafana implementados progresivamente |

**Lecciones Aprendidas (SegurosConfianza):**

1. **No subestimes el cambio cultural.** La parte técnica fue 30% del esfuerzo. El cambio de mentalidad ("eres responsable de tu código en producción") fue el otro 70%.
2. **No contenerices las bases de datos al principio.** Para una empresa con DBAs dedicados, mover Oracle a K8s era demasiado riesgo. Se hizo en fase posterior con operadores especializados.
3. **Celebra las pequeñas victorias.** Primer deploy con Docker (de 2 semanas a 4 horas): pizza. Primer deploy CI/CD (de 4 horas a 15 minutos): cerveza. Mantén el momentum.
4. **Documenta TODO.** Cuando el equipo creció de 2 a 30 personas, el conocimiento mental tuvo que transferirse. Wikis, runbooks, y pairing fueron cruciales.
5. **La observabilidad llegó tarde y dolió.** Durante la migración, problemas de diagnóstico por falta de logs centralizados. Si pudieran volver atrás, implementarían el stack de observabilidad ANTES de migrar.

### 12.10.3 Caso 3: E-commerce Preparando Black Friday

**Contexto**

*ModaExpress*, e-commerce de moda con 2 millones de visitas/día normales. Black Friday: 20 millones de visitas/día (10x). El año anterior, el sitio estuvo caído 45 minutos. Pérdida estimada: $280,000.

**Objetivo 2024:** Zero downtime en Black Friday.

**Estrategia de Preparación**

**1. Pruebas de Carga con k6 (3 meses antes)**

```javascript
// load-test.js — Simulación de tráfico de Black Friday
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '5m', target: 100 },     // Calentamiento
    { duration: '10m', target: 1000 },   // Rampa
    { duration: '30m', target: 20000 },  // Pico: 20K usuarios simultáneos
    { duration: '10m', target: 0 },      // Enfriamiento
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],    // 95% requests < 500ms
    errors: ['rate<0.01'],               // Tasa error < 1%
  },
};

export default function () {
  const responses = http.batch([
    ['GET', 'https://www.modaexpress.com/'],
    ['GET', 'https://www.modaexpress.com/api/products?category=moda'],
    ['POST', 'https://www.modaexpress.com/api/cart', JSON.stringify({ productId: 123 })],
  ]);
  const res = http.get('https://www.modaexpress.com/api/checkout');
  check(res, { 'status 200': (r) => r.status === 200 }) || errorRate.add(1);
  sleep(1);
}
```

Resultado inicial: el sistema colapsaba a 5,000 usuarios concurrentes.

**2. Optimizaciones Implementadas (2 meses antes)**

Basado en resultados de k6:

- **Caché agresiva con Redis (5 niveles):**

```python
@cache_decorator(ttl=300)
def get_product(product_id):
    # 1. Redis (cache de app)
    # 2. CDN (contenido estático)
    # 3. Vista materializada (BD optimizada)
    # 4. Query normal (último recurso)
    return query_database(product_id)
```

- **CDN para assets estáticos:** CloudFront distribuye imágenes, CSS, JS desde S3. Reducción de carga al origin: 85%.

- **Separación tráfico lecturas/escrituras:**

```yaml
api-read:
  replicas: 10
  hpa: { min: 5, max: 30, cpuTarget: 60 }
api-write:
  replicas: 3
  hpa: { min: 2, max: 10, cpuTarget: 70 }
```

**3. Auto-escalado Configurado (1 mes antes)**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 5
  maxReplicas: 50    # Capacidad máxima para Black Friday
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120
```

**4. Plan de Rollback (documentado y testeado)**

```markdown
# Runbook: Rollback durante Black Friday

## Antes del evento
- [ ] Blue/Green configurado con Deployments separados
- [ ] Script de rollback probado en staging
- [ ] War room abierta 30 min antes del inicio

## Si algo falla
1. Nivel 1: Degradación parcial (latencia > 1s)
   - Escalar manualmente servicios afectados
   - Si no mejora → nivel 2

2. Nivel 2: Errores > 2%
   - Activar feature flag: desactivar recomendaciones personalizadas
   - Si no mejora → nivel 3

3. Nivel 3: Rollback completo
   kubectl patch service main -p '{"spec":{"selector":{"track":"blue"}}}'
   Rollback en < 1 segundo
```

**5. War Room de Black Friday**

```
Viernes 00:00 UTC: Inicio de Black Friday

Dashboard en tiempo real (Grafana):
┌────────────────────────────────────────────────────────┐
│  BLACK FRIDAY 2024 — COMMAND CENTER                    │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Usuarios activos:   ████████████░░  18,234 / 20K     │
│  Requests/segundo:   ██████████████  12,456           │
│  Latencia P95:       ██████░░░░░░░░  312ms (OK < 500) │
│  Tasa de error:      ██░░░░░░░░░░░░  0.3% (OK < 1%)   │
│                                                        │
│  Pods activos:                                        │
│    api-read:   28/50  ██████████████░░░░░░            │
│    api-write:   6/10  ████████████████░░░░            │
│    checkout:   12/20  ████████████████████░░          │
│    search:      8/15  ████████████████░░░░░░          │
│                                                        │
│  Alertas:     0 critical | 1 warning | 12 ok           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Resultados Finales:**

| Métrica                | Año Anterior | Black Friday 2024 |
|------------------------|--------------|-------------------|
| Uptime                 | 99.1%        | 100%              |
| Downtime               | 45 min       | 0 min             |
| Tiempo respuesta P95   | 3.2s         | 312ms             |
| Pico usuarios          | 18,000       | 22,400            |
| Pedidos procesados     | 45,000       | 67,500 (+50%)     |
| Costo infra (día BF)   | $1,200       | $2,800            |
| Pérdida por downtime   | $280,000     | $0                |

**Lecciones Aprendidas (ModaExpress):**

1. **Pruebas de carga son no-negociables.** k6 reveló cuellos de botella que nunca habrían aparecido en pruebas unitarias. El sistema habría colapsado sin ellas.
2. **Cache salva vidas.** La estrategia de 5 niveles de caché absorbió el 92% de requests antes de llegar a la base de datos.
3. **El plan de rollback debe ser instintivo.** Durante los simulacros, midieron el tiempo de rollback: 47 segundos desde la decisión hasta la ejecución exitosa. En producción, debe ser < 60 segundos.
4. **Separar lecturas de escrituras es oro.** Permitió escalar independientemente cada capa. Las lecturas (90% del tráfico) escalaron agresivamente; las escrituras (10%) se mantuvieron estables.
5. **El costo del downtime es mucho mayor que el costo de la infraestructura extra.** Gastaron $1,600 adicionales en infraestructura para Black Friday. Evitaron $280,000 en pérdidas. ROI: 175x.

---

## 12.11 El Camino del DevOps / Platform Engineer: ¿Qué Sigue Después de Docker?

> *"Dominar Docker no es el final. Es la llave que abre la puerta. Lo que hay detrás es infinito."*

Has llegado al final de este libro, pero tu viaje apenas comienza. Docker es el fundamento, la herramienta que te dio superpoderes como desarrollador. Ahora, como ingeniero capaz de diseñar sistemas para producción, el horizonte se expande. Esto es lo que te espera.

### 12.11.1 Kubernetes Avanzado

Dominar los Deployment/Service/Ingress es solo el principio. El ecosistema Kubernetes es vasto:

- **Service Mesh (Istio, Linkerd, Cilium):** mTLS automático, traffic splitting, circuit breaking, fault injection, observabilidad sin modificar código.
- **Operators:** programas que encapsulan el conocimiento operativo humano. Un operator de PostgreSQL sabe hacer failover, backup, scaling. Patrón fundamental para stateful workloads.
- **Custom Resource Definitions (CRDs):** extiende la API de Kubernetes con tus propios recursos. Ejemplo: en lugar de YAMLs de Deployment+Service+Ingress+HPA, defines un CRD `WebApp` que genera todo automáticamente.
- **Policy as Code:** OPA/Gatekeeper, Kyverno. Validan y mutan recursos automáticamente. "Nunca ejecutar como root" no es un post-it, es una regla que el cluster rechaza.

### 12.11.2 Observabilidad: Más Allá de los Logs

La observabilidad moderna va más allá de "ver logs cuando algo falla":

- **Tracing distribuido (OpenTelemetry + Jaeger/Tempo):** sigue una request a través de 15 microservicios. Identifica exactamente dónde está la latencia. Fundamental en arquitecturas distribuidas.
- **SLOs y SLIs:** Service Level Objectives y Service Level Indicators. Define qué significa "disponible" cuantitativamente. "99.95% de requests responden en < 300ms en ventanas de 30 días." Las alertas deben basarse en SLOs, no en métricas crudas.
- **Error Budgets:** si tu SLO es 99.9%, tienes un "presupuesto" de 43 minutos de downtime al mes. Gastar presupuesto rápido → congelar features y estabilizar. Gastar poco → iterar más rápido.
- **Continuous Profiling (Pyroscope, Parca):** perfilado continuo de CPU/memoria en producción. Identifica memory leaks y hot paths sin esperar a que fallen.
- **eBPF:** observabilidad a nivel de kernel sin agentes. Cilium, Pixie, Hubble te muestran exactamente qué packet viaja de qué pod a qué pod.

### 12.11.3 Infraestructura como Código y Platform Engineering

- **Terraform / OpenTofu:** IaC para todo. Clusters, redes, DNS, load balancers, bases de datos gestionadas, IAM, monitoring. Todo versionado en Git.
- **Pulumi:** IaC en lenguajes de propósito general (Python, Go, TypeScript). Más expresivo que HCL, loops y condiciones sin limitaciones.
- **Crossplane:** IaC pero dentro de Kubernetes. Define infraestructura cloud como CRDs. Una `RDSInstance` es un recurso de Kubernetes como cualquier Deployment.
- **Platform Engineering:** el siguiente nivel de DevOps. En lugar de cada equipo gestionando su infraestructura, creas una Internal Developer Platform (IDP). Backstage, Port, Humanitec. Dashboards self-service: "Crear un nuevo microservicio" → se genera repo, pipeline, namespace, deployment, monitoring automáticamente.
- **Backstage (Spotify):** catálogo de servicios, documentation-as-code, scaffolding de proyectos. El estándar de facto para IDPs.

### 12.11.4 CI/CD Avanzado

- **Progressive Delivery (Argo Rollouts):** Canary, Blue/Green, y análisis automatizado integrado. Argo Rollouts consulta a Prometheus durante un canary: si las métricas se degradan, aborta automáticamente.
- **Feature Flags (LaunchDarkly, Unleash, Flagsmith):** separa deploy (código en producción) de release (código activado para usuarios). Hacer deploy el viernes a las 5 PM no da miedo si la feature está desactivada.
- **DORA Metrics:** las cuatro métricas que miden la madurez de tu entrega de software: Deployment Frequency, Lead Time for Changes, Change Failure Rate, Time to Restore Service. Mide tu progreso objetivamente.
- **Supply Chain Security (SLSA, Sigstore, Cosign):** firma de imágenes con identidad verificable. Saber exactamente quién construyó qué imagen y con qué código. SBOM (Software Bill of Materials) para cada artefacto.

### 12.11.5 Seguridad Avanzada

- **mTLS everywhere (Istio/Linkerd):** toda comunicación entre servicios cifrada y autenticada. Zero Trust: ni siquiera confías en el tráfico interno de tu propio cluster.
- **OPA/Gatekeeper:** reglas de seguridad como código. "Ningún pod puede usar hostNetwork", "todos los pods deben tener resource limits", "nadie puede montar hostPath". La seguridad no es aspirativa, es enforceable.
- **Falco:** detección de amenazas en runtime. Monitoriza syscalls del kernel y detecta comportamientos anómalos: un shell generado dentro de un contenedor, escritura en binarios del sistema, conexiones de red sospechosas.
- **Kyverno:** alternativa a OPA con sintaxis más simple, nativa de Kubernetes. Genera, muta y valida políticas. "Todo PVC debe tener encrypt: true en sus annotations."
- **Image signing (Cosign + Sigstore):** firma criptográfica de imágenes. Tu cluster solo ejecuta imágenes firmadas por tu CI y verificadas por tu policy controller.

### 12.11.6 Comunidad y Aprendizaje Continuo

El ecosistema cloud-native es uno de los más vibrantes en la historia del software:

- **KubeCon / CloudNativeCon:** la conferencia de la CNCF. 10,000+ asistentes, cientos de charlas. El lugar donde se anuncian los proyectos que usarás en 2 años.
- **CNCF Landscape:** el mapa de todas las herramientas cloud-native. Intimidante pero educativo. Navegar el landscape es una habilidad en sí misma.
- **DevOpsDays:** conferencias locales en cientos de ciudades. Más íntimas, más networking, más "esto falló en mi empresa y así lo arreglamos".
- **Libros que debes leer después de este:**
  - *"Site Reliability Engineering"* (Google) — la biblia de la operación de sistemas a escala.
  - *"The DevOps Handbook"* — cómo implementar los principios DevOps en tu organización.
  - *"Designing Data-Intensive Applications"* (Kleppmann) — fundamentos de sistemas distribuidos que todo ingeniero debe conocer.
  - *"Cloud Native Infrastructure"* — patrones para infraestructura en Kubernetes.
  - *"Kubernetes: Up and Running"* — la guía definitiva de K8s.
- **Keep the beginner's mind:** el ecosistema evoluciona tan rápido que siempre serás principiante en algo. Eso no es frustración, es oportunidad.

### 12.11.7 La Mentalidad: Automatizar, Medir, Iterar

Más allá de las herramientas, lo que distingue al experto es la mentalidad:

```
┌─────────────────────────────────────────────┐
│          LA MENTALIDAD DEL EXPERTO          │
├─────────────────────────────────────────────┤
│                                             │
│   AUTOMATIZAR                               │
│   Si lo haces dos veces, automatízalo.      │
│   Si es manual, es frágil.                  │
│   Si es frágil, fallará cuando más lo       │
│   necesites.                                │
│                                             │
│   MEDIR                                     │
│   Sin métricas, eres ciego.                 │
│   Sin SLOs, no sabes si estás mejorando.   │
│   Sin alertas, te enteras por los usuarios. │
│                                             │
│   ITERAR                                    │
│   Despliega pequeño, despliega frecuente.   │
│   Fallar rápido es mejor que fallar grande. │
│   Cada incidente es una oportunidad de      │
│   aprendizaje (post-mortem blameless).      │
│                                             │
│   COMPARTIR                                 │
│   El conocimiento que no compartes muere.   │
│   Runbooks en wiki, no en tu cabeza.        │
│   Automatiza lo que sabes, documenta lo     │
│   que automatizas.                          │
│                                             │
└─────────────────────────────────────────────┘
```

---

## Epílogo: El Círculo se Cierra

Hace once capítulos, ejecutaste tu primer `docker run hello-world`. Ese comando descargó una imagen, creó un contenedor, ejecutó un proceso, y se detuvo. Fueron 3 segundos que cambiaron tu forma de entender el software.

Hoy, once capítulos después, has aprendido a:

- Construir imágenes eficientes con multi-stage builds y Dockerfiles optimizados.
- Orquestrar servicios con Docker Compose en desarrollo y Swarm en producción.
- Navegar las profundidades de Kubernetes: Pods, Deployments, Services, Ingresses, StatefulSets, HPA.
- Diseñar redes segregadas con NetworkPolicies y service meshes.
- Externalizar configuración con ConfigMaps, Secrets, y External Secrets Operator.
- Implementar CI/CD con GitHub Actions, Jenkins, y GitOps con ArgoCD.
- Monitorear con Prometheus, Grafana, Loki, y alertas accionables.
- Asegurar tus contenedores con usuarios no-root, capabilities restringidas, filesystems read-only, y escaneo de CVEs.
- Escalar horizontalmente con HPA, verticalmente con VPA, y por eventos con KEDA.
- Recuperarte de desastres con estrategias 3-2-1, backups automatizados, y simulacros regulares.
- Desplegar en producción con Blue/Green, Canary, A/B Testing, y Rolling Updates.

Y lo más importante: **has desarrollado el criterio** para saber cuándo usar cada herramienta, cada patrón, cada estrategia.

El viaje de novato a experto no termina aquí. Esto es solo el comienzo de una carrera en la que siempre habrá algo nuevo que aprender, algo que automatizar, algo que mejorar. La tecnología cloud-native evoluciona a una velocidad vertiginosa, pero los principios que has aprendido —automatización, observabilidad, resiliencia, seguridad, mejora continua— son atemporales.

> *"Un experto no es quien tiene todas las respuestas. Es quien sabe qué preguntas hacer, y dónde buscar cuando no las encuentra."*

Recuerda tu primer `docker run hello-world`. Recuerda la primera vez que algo funcionó en un contenedor y pensaste "esto es magia". Recuerda la primera vez que algo falló en producción y lo arreglaste con confianza porque tenías un checklist, métricas, y un plan de rollback.

De novato a experto. El camino ha sido largo, pero el destino es solo el principio de algo más grande.

Ahora ve y construye algo extraordinario.

---

```
╔══════════════════════════════════════════════════════════════════╗
║                                                                  ║
║   "Un sistema de software es como un jardín.                     ║
║    No lo 'terminas'. Lo cultivas, lo podas, lo riegas,           ║
║    y te adaptas a las estaciones.                                ║
║                                                                  ║
║    Los contenedores te dan la tierra fértil.                     ║
║    La orquestración te da las herramientas de jardinería.        ║
║    La observabilidad te dice qué necesita cada planta.           ║
║    La automatización riega mientras duermes.                     ║
║                                                                  ║
║    Pero el jardinero eres tú."                                   ║
║                                                                  ║
║         — Fin del Libro —                                        ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

**Apéndice A: Checklist de Producción (Versión Imprimible)**

Si imprimes una sola página de este libro, que sea esta:

```
┌─────────────────────────────────────────────────────────────┐
│  CHECKLIST DE PRODUCCIÓN — FINAL                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  IMÁGENES                                                   │
│  [ ] Tags específicos (no latest)                           │
│  [ ] Imágenes slim/alpine/distroless                        │
│  [ ] Imagen escaneada (Trivy/Docker Scout) — 0 CRITICALs   │
│  [ ] Multi-stage build optimizado                           │
│  [ ] .dockerignore presente y completo                      │
│                                                             │
│  SEGURIDAD                                                  │
│  [ ] USER no root (UID > 1000)                              │
│  [ ] Capabilities: drop ALL, add solo necesarias            │
│  [ ] Read-only root filesystem                              │
│  [ ] Secrets externalizados (nunca en imagen)               │
│  [ ] TLS en tráfico externo (cert-manager/Let's Encrypt)    │
│                                                             │
│  SALUD Y RESILIENCIA                                        │
│  [ ] HEALTHCHECK o livenessProbe configurado                │
│  [ ] readinessProbe configurado                             │
│  [ ] Restart policy: unless-stopped/always                  │
│  [ ] Timeouts, retries y circuit breakers                   │
│                                                             │
│  RECURSOS                                                   │
│  [ ] Memory limits y requests configurados                  │
│  [ ] CPU limits y requests configurados                     │
│  [ ] PIDs limit configurado (si procede)                    │
│                                                             │
│  RED                                                        │
│  [ ] Redes segregadas (frontend/backend/data)               │
│  [ ] NetworkPolicies restrictivas                           │
│  [ ] Rate limiting configurado                              │
│                                                             │
│  LOGGING                                                    │
│  [ ] Logs a stdout/stderr (nunca archivos)                  │
│  [ ] Logs estructurados (JSON)                              │
│  [ ] Rotación y retención configuradas                      │
│                                                             │
│  PERSISTENCIA                                               │
│  [ ] Volúmenes nombrados/PVCs con backup automático         │
│  [ ] Backup de BD automatizado + restore testeado           │
│                                                             │
│  MONITOREO                                                  │
│  [ ] Métricas Prometheus expuestas en /metrics              │
│  [ ] Alertas configuradas, probadas, con runbooks           │
│                                                             │
│  DESPLIEGUE                                                 │
│  [ ] Estrategia definida (Blue/Green/Canary/Rolling)        │
│  [ ] Plan de rollback documentado y probado                 │
│  [ ] PodDisruptionBudget configurado                        │
│  [ ] HPA o KEDA configurado según necesidad                 │
│                                                             │
│  DR                                                         │
│  [ ] Estrategia de backup 3-2-1 implementada                │
│  [ ] RPO y RTO definidos y medidos                          │
│  [ ] Simulacro de restauración programado                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

**Apéndice B: Índice de Comandos Esenciales para Producción**

```bash
# ===== IMÁGENES =====
docker scout quickview <image>                  # Escanear CVEs
trivy image <image>                             # Escaneo detallado
docker build --label "org.opencontainers..." .  # Build con metadatos OCI
docker image prune -a --filter "until=168h"     # Limpiar imágenes > 7 días

# ===== KUBERNETES OPERACIONES =====
kubectl rollout history deployment/<name>       # Historial de deploys
kubectl rollout undo deployment/<name>          # Rollback
kubectl rollout status deployment/<name>        # Ver progreso
kubectl scale deployment/<name> --replicas=N    # Escalar manualmente
kubectl top pods --sort-by=cpu                  # Ver uso de recursos
kubectl top nodes                               # Ver uso de nodos
kubectl describe pod <name>                     # Debugging detallado
kubectl logs -f -l app=<name>                   # Logs en tiempo real

# ===== HELM =====
helm upgrade --install <name> <chart> -f values/prod.yaml
helm rollback <name>
helm list --namespace production
helm history <name>

# ===== RED =====
kubectl get networkpolicies                    # Ver políticas de red
kubectl describe networkpolicy <name>
kubectl get ingress -A                         # Todos los Ingresses

# ===== SECRETOS =====
kubeseal --format=yaml < secret.yaml > sealed.yaml
kubectl get externalsecrets -A

# ===== MONITOREO =====
# Prometheus queries comunes:
#   rate(http_requests_total[5m])
#   histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
#   container_memory_usage_bytes{namespace="production"}
#   kube_pod_container_resource_requests{resource="cpu"}

# ===== DEBUGGING =====
kubectl exec -it <pod> -- /bin/sh               # Shell en contenedor
kubectl port-forward <pod> 8080:8080            # Acceso local a pod
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl auth can-i create deployments           # Verificar permisos RBAC

# ===== DR Y BACKUP =====
pg_dump -h postgres -U myapp -d myapp > backup.sql
psql -h postgres -U myapp -d myapp < restore.sql
kubectl get volumesnapshots
aws s3 cp /backup/ s3://myapp-backups/ --recursive --sse AES256
```

---

> *"Los tres grandes de la producción: despliega pequeño, monitorea todo, ten siempre un plan B."*
>
> — El autor, después de innumerables incidentes a las 3 AM.

**— FIN DEL CAPÍTULO 12 —**
**— FIN DEL LIBRO —**


---

## 12.12 Extras: Profundizando en Temas Clave

### 12.12.1 La Anatomía de un Incidente en Producción

Todo ingeniero que ha trabajado en producción tiene historias de incidentes. Aquí hay un patrón común y cómo enfrentarlo.

**Cronología típica de un incidente:**

```
T+0min   — Algo se rompe. Los usuarios aún no lo notan.
T+2min   — La primera alerta de Prometheus se dispara.
           Slack de on-call recibe notificación.
T+5min   — El ingeniero on-call abre su laptop.
           Revisa el dashboard de Grafana.
T+8min   — Diagnóstico inicial: "alta latencia en el servicio de pagos".
           Mira los logs en Loki: filtra por trace_id, busca ERROR.
T+12min  — Causa raíz identificada: "la base de datos de réplica
           está caída, todas las lecturas van a la primaria".
T+15min  — Acción: promover réplica de respaldo. O escalar la
           primaria verticalmente mientras se recupera la réplica.
T+20min  — Servicio restaurado. Latencia normalizada.
T+30min  — War room virtual: ¿qué pasó exactamente? ¿cómo prevenirlo?
T+2días  — Post-mortem blameless publicado. Action items asignados.
```

**Lo que distingue un buen incident response:**

| Mal                                          | Bien                                              |
|----------------------------------------------|---------------------------------------------------|
| Pánico, culpa, "¿quién rompió esto?"         | Calma, foco en restaurar servicio primero         |
| Una persona tratando de arreglarlo todo sola | Guerra de salón virtual: todos colaborando        |
| "Reiniciamos y vemos qué pasa"               | Hipótesis → validación → acción medida            |
| Sin registro de lo ocurrido                  | Timeline documentado en canal compartido          |
| Mismo incidente se repite                    | Post-mortem con action items que realmente se hacen|

**Ejemplo de post-mortem (template):**

```markdown
# Post-Mortem: Incidente #42 — Alta latencia en checkout

**Fecha:** 2024-03-15
**Duración:** 18 minutos (14:03 - 14:21 UTC)
**Impacto:** 3% de usuarios experimentaron errores 500 en checkout
**Severidad:** P2

## Timeline
- 14:03 — Alerta: latencia P95 > 2s en api-checkout
- 14:05 — Ingeniero on-call (@maria) comienza investigación
- 14:08 — Identificado: réplica de BD en AZ us-east-1c no responde
- 14:12 — Decisión: redirigir tráfico de lectura a primaria
- 14:15 — Rollout del change (kubectl edit + rolling restart)
- 14:20 — Latencia normalizada: P95 = 180ms
- 14:21 — Incidente cerrado

## Causa Raíz
El nodo de Kubernetes que alojaba la réplica de PostgreSQL en us-east-1c
experimentó un kernel panic debido a un bug en el driver del disco.
El failover automático no se activó porque el health check del operator
tenía un timeout demasiado largo (60s).

## ¿Por qué no lo detectamos antes?
- El health check del operador estaba configurado demasiado permisivo.
- No teníamos alertas específicas para "réplica de BD no saludable".

## Action Items
- [ ] Reducir timeout del health check a 15s (@david, PR #1234)
- [ ] Crear alerta: PostgreSQL replica lag > 30s (@ana, PR #1235)
- [ ] Simulacro mensual de failover de réplica (@equipo, calendario)
- [ ] Investigar bug del driver de disco (@ops, ticket INFRA-567)
```

### 12.12.2 Kubernetes RBAC para Producción

El acceso a tu cluster de producción debe seguir el principio de mínimo privilegio.

```yaml
# developer-role.yaml — Acceso limitado a namespace de desarrollo
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
  - apiGroups: ["", "apps", "batch", "extensions"]
    resources: ["pods", "pods/log", "deployments", "services", "configmaps", "jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/exec", "pods/portforward"]
    verbs: ["create"]

---
# production-readonly.yaml — Solo lectura en producción
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: readonly
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]

---
# ClusterRole para platform engineers (acceso completo)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-engineer
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["nodes", "namespaces", "persistentvolumes"]
    verbs: ["get", "list", "watch"]

---
# Binding: asignar developer role a un grupo
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-binding
  namespace: development
subjects:
  - kind: Group
    name: "developers"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### 12.12.3 Debugging en Producción: Técnicas Avanzadas

Cuando algo falla y no entiendes por qué, estas técnicas te salvarán.

**1. kubectl debug — Contenedor efímero para diagnóstico:**

```bash
# Crear un contenedor de debug en un pod existente
kubectl debug -it api-7d4f8b9c-abcde --image=nicolaka/netshoot --target=api

# Ahora tienes tcpdump, strace, nslookup, curl, etc. en el namespace del pod
```

**2. Inspeccionar tráfico entre servicios:**

```bash
# Usando kubeshark (antes Mizu) para ver tráfico HTTP en tiempo real
kubeshark tap

# O con tcpdump en un contenedor de debug
kubectl run tcpdump --rm -it --image=nicolaka/netshoot -- tcpdump -i any host postgres
```

**3. perfilado de CPU en caliente:**

```bash
# Para aplicaciones Go: pprof endpoint
kubectl port-forward api-7d4f8b9c-abcde 6060:6060
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Para Node.js:
kubectl exec -it api-7d4f8b9c-abcde -- node --inspect-brk server.js
kubectl port-forward api-7d4f8b9c-abcde 9229:9229
# Abrir chrome://inspect
```

**4. Análisis de tráfico DNS:**

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl run dns-debug --rm -it --image=busybox:1.36 -- nslookup postgres.default.svc.cluster.local
```

### 12.12.4 Optimización de Costos en la Nube con Docker/Kubernetes

La elasticidad de Kubernetes puede ser un arma de doble filo financiero. Estrategias para optimizar:

**1. Spot instances para workloads tolerantes a interrupción:**

```yaml
# Usando Karpenter (alternativa moderna al Cluster Autoscaler)
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-pool
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
      nodeClassRef:
        name: default
  limits:
    cpu: "100"
  disruption:
    consolidationPolicy: WhenUnderutilized
```

**2. Vertical Rightsizing:**

```bash
# kubecost puede recomendarte ajustes de recursos basados en uso real
kubectl cost --serviceaccount kubecost

# VPA en modo "Off" o "Initial" para obtener recomendaciones sin riesgo
# Revisa las recomendaciones con:
kubectl describe vpa api-vpa
```

**3. Escalar a cero en entornos no productivos:**

```yaml
# KEDA: escalar a 0 fuera de horario laboral
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: staging-api-scaler
spec:
  scaleTargetRef:
    name: staging-api
  minReplicaCount: 0
  maxReplicaCount: 3
  triggers:
    - type: cron
      metadata:
        timezone: America/New_York
        start: 0 8 * * 1-5   # 8 AM L-V
        end: 0 19 * * 1-5    # 7 PM L-V
        desiredReplicas: "2"
```

**4. Políticas de Namespace con quotas:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: development
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    persistentvolumeclaims: "10"
    pods: "50"
```

### 12.12.5 Configurando un Pipeline CI/CD Completo con GitHub Actions

Un pipeline de producción completo para Docker multientorno:

```yaml
# .github/workflows/production-pipeline.yml
name: Production Pipeline

on:
  push:
    branches: [main]
    tags: ['v*.*.*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Job 1: Build y Test
  build-and-test:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Generate image metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=tag,pattern={{version}}
            type=sha,prefix=git-,format=short
            type=raw,value=prod-${{ github.run_number }}

      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run unit tests
        run: docker run --rm ${{ steps.meta.outputs.tags }} pytest

      - name: Run Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.meta.outputs.tags }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  # Job 2: Deploy a Staging
  deploy-staging:
    needs: build-and-test
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Checkout IaC repo
        uses: actions/checkout@v4
        with:
          repository: mi-empresa/myapp-deploy
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Update image tag in GitOps repo
        run: |
          cd overlays/staging
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-and-test.outputs.image-tag }}

      - name: Commit and push
        run: |
          git config user.name "CI Bot"
          git config user.email "ci@empresa.com"
          git commit -am "Staging: deploy ${{ needs.build-and-test.outputs.image-tag }}"
          git push

      - name: Wait for deployment health
        run: |
          echo "Esperando 120s para que ArgoCD sincronice..."
          sleep 120
          # Aquí iría kubectl wait o curl al health endpoint de staging

  # Job 3: Smoke Tests en Staging
  smoke-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Run smoke tests
        run: |
          curl -f https://staging.midominio.com/health || exit 1
          curl -f https://staging.midominio.com/api/version || exit 1

  # Job 4: Deploy a Production (con aprobación manual)
  deploy-production:
    needs: smoke-tests
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Checkout IaC repo
        uses: actions/checkout@v4
        with:
          repository: mi-empresa/myapp-deploy
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Update image tag in GitOps repo (production)
        run: |
          cd overlays/prod
          kustomize edit set image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-and-test.outputs.image-tag }}

      - name: Commit and push to production
        run: |
          git config user.name "CI Bot"
          git config user.email "ci@empresa.com"
          git commit -am "Prod: deploy ${{ needs.build-and-test.outputs.image-tag }}"
          git push

  # Job 5: Post-deploy verification
  post-deploy-verify:
    needs: deploy-production
    runs-on: ubuntu-latest
    steps:
      - name: Verify production health
        run: |
          for i in {1..10}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://api.midominio.com/health)
            if [ "$STATUS" = "200" ]; then
              echo "Production healthy!"
              exit 0
            fi
            echo "Intento $i: status $STATUS, reintentando..."
            sleep 10
          done
          echo "Production health check failed after 10 attempts"
          exit 1

      - name: Notify on success
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }}             -H "Content-Type: application/json"             -d '{"text":":white_check_mark: Deploy exitoso en produccion! Version: ${{ needs.build-and-test.outputs.image-tag }}"}'

      - name: Notify on failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }}             -H "Content-Type: application/json"             -d '{"text":":x: Deploy a produccion FALLIDO! @channel por favor revisar."}'
```

### 12.12.6 Docker en CI/CD: Optimización de Cache

El cache de Docker en CI puede marcar la diferencia entre un build de 30 segundos y uno de 10 minutos.

```yaml
# GitHub Actions: cache efectiva con buildx
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    cache-from: type=gha        # Cache de GitHub Actions
    cache-to: type=gha,mode=max  # Guardar todas las capas

# Para GitLab CI: registry cache
docker-build:
  image: docker:24-dind
  script:
    - docker buildx build
        --cache-from type=registry,ref=$CI_REGISTRY_IMAGE:buildcache
        --cache-to type=registry,ref=$CI_REGISTRY_IMAGE:buildcache,mode=max
        --push
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
```

### 12.12.7 Cuándo NO Usar Contenedores

Parte de ser experto es saber cuándo una herramienta NO es la adecuada:

- **Aplicaciones de escritorio con GUI compleja:** Docker no fue diseñado para esto.
- **Latencia de red ultra-baja (<100 microsegundos):** el overhead de bridge networking puede ser un problema. Usa host networking o no contenerices.
- **Hardware especializado sin soporte en Linux:** GPUs están soportadas (nvidia-docker), pero hardware propietario de nicho puede no tener drivers en el kernel.
- **Equipos sin cultura DevOps y sin apoyo organizacional:** forzar Kubernetes sin apoyo del management genera más problemas que soluciones.
- **Aplicaciones stateful extremadamente complejas sin operador:** migrar un Oracle RAC o un cluster Hadoop a Kubernetes sin un operador maduro es receta para el desastre.

---

## 12.13 El Futuro de los Contenedores: Lo Que Viene

### 12.13.1 WebAssembly (Wasm) en el Servidor

WebAssembly está emergiendo como complemento/alternativa a contenedores para ciertos casos de uso:

- **Arranque en microsegundos** (vs segundos de un contenedor).
- **Sandbox por defecto** a nivel de instrucción (más seguro).
- **Menor footprint** (~1MB vs ~100MB+ de una imagen contenerizada).
- **Docker + Wasm:** Docker Desktop ya soporta Wasm runtimes. `docker run --runtime=io.containerd.wasmedge.v1 mywasmapp.wasm`.

Casos de uso ideales para Wasm: plugins, funciones serverless, edge computing, filtros de red.

### 12.13.2 eBPF: El Kernel como Plataforma

eBPF está transformando observabilidad, seguridad, y networking sin necesidad de sidecars ni agentes:

- **Cilium:** reemplaza kube-proxy con eBPF para networking de alto rendimiento.
- **Hubble:** observabilidad de red con eBPF (tráfico entre pods en tiempo real).
- **Falco:** detección de amenazas a nivel de syscalls con eBPF.
- **Pixie:** debugging instantáneo de aplicaciones sin instrumentación.

### 12.13.3 IA y Operaciones (AIOps)

La inteligencia artificial está entrando en el mundo de las operaciones:

- **Detección de anomalías automática:** en lugar de thresholds estáticos, modelos ML que aprenden el comportamiento normal.
- **Root Cause Analysis asistido:** correlación automática de logs, métricas y trazas para identificar causas raíz.
- **Auto-remediation:** sistemas que no solo detectan problemas sino que aplican fixes pre-aprobados automáticamente.
- **FinOps inteligente:** optimización automática de costos basada en patrones de uso.

### 12.13.4 Plataformas Unificadas

La tendencia es hacia plataformas que integran todo el ciclo de vida:

```
De "cada equipo construye su stack" a "plataforma unificada":
  - Backstage (catálogo de servicios)
  - Crossplane (IaC composable)
  - ArgoCD (GitOps)
  - Prometheus + Grafana (observabilidad)
  - OPA/Gatekeeper (políticas)
  - Vault (secretos)
  - KEDA (auto-escalado)
  Todo integrado, self-service, gobernado.
```

---

## Epílogo Extendido: Despedida

Este libro comenzó con `docker run hello-world` y termina con sistemas complejos en producción. Pero el viaje real no está en las páginas, sino en lo que harás mañana, cuando abras tu terminal.

Recuerda:

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   El código que escribes hoy se ejecutará en            │
│   producción dentro de semanas.                         │
│                                                         │
│   Los contenedores que construyes hoy son la            │
│   infraestructura que mantendrá tu empresa en pie.      │
│                                                         │
│   Las alertas que configuras hoy te despertarán         │
│   —o no— a las 3 AM del domingo.                       │
│                                                         │
│   El checklist que sigues hoy evitará el incidente      │
│   que habría costado $100,000.                          │
│                                                         │
│   El runbook que escribes hoy será el salvavidas        │
│   del ingeniero on-call cuando tú estés de vacaciones.  │
│                                                         │
│   Cada decisión importa.                                │
│   Cada automatización cuenta.                           │
│   Cada métrica tiene una historia que contar.           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

Gracias por llegar hasta el final. Ha sido un viaje extraordinario, y me siento honrado de haber sido tu guía. Ahora el capitán eres tú.

> *"No es la herramienta lo que define al artesano, sino cómo la usa."*

Ve y construye. Los contenedores te esperan.

```bash
# Un último comando, por los viejos tiempos
docker run --rm hello-world

# Hello from Docker!
# This message shows that your installation appears to be working correctly.
#
# You are no longer a novice.
# You are an expert.
# Now go build something amazing.
```

---

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║   DOCKER: DE NOVATO A EXPERTO                                       ║
║                                                                      ║
║   Capítulo 12: Producción — El Último Kilómetro                    ║
║                                                                      ║
║   Escrito con pasión por el arte de la ingeniería de software.       ║
║   Dedicado a todos los que alguna vez ejecutaron su primer           ║
║   docker run y sintieron que el mundo se expandía.                  ║
║                                                                      ║
║   © 2024 — El viaje continúa.                                        ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

**— FIN DEL CAPÍTULO 12 —**
**— FIN DEL LIBRO —**
