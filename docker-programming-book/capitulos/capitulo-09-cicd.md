# Capítulo 9: CI/CD con Docker

---

> *"El software nunca está terminado, solo está en producción."* — Anónimo

---

## 9.1 Docker en CI/CD: la revolución

La integración continua y el despliegue continuo (CI/CD) no nacieron con Docker, pero Docker las transformó de manera irreversible. Para entender la magnitud del cambio, debemos viajar brevemente al pasado.

### 9.1.1 El mundo antes de Docker

En la era pre-Docker, el pipeline de CI/CD era un artefacto frágil. Cada desarrollador trabajaba en su máquina local con una combinación específica de sistema operativo, versiones de runtime, librerías del sistema y variables de entorno. Cuando el código llegaba al servidor de CI, ocurría el temido:

> *"Funciona en mi máquina, no en CI."*

Este mantra era el síntoma de un problema más profundo: la **deriva de configuración** (configuration drift). Cada entorno —desarrollo, CI, staging, producción— era un copo de nieve único e irreproducible. Los equipos invertían horas en depurar diferencias sutiles:

- La versión de OpenSSL en la máquina del desarrollador era 1.1.1, pero en CI era 1.0.2.
- El JDK en staging estaba parcheado con un hotfix que producción no tenía.
- La variable `LANG` causaba que los tests de ordenamiento fallaran solo en CI.
- Una librería del sistema (`libxml2`, `libreoffice-headless`) ausente rompía el build.

Los equipos de DevOps mantenían grotescos scripts de provisioning: Bash scripts de 500 líneas, playbooks de Ansible con condicionales anidados, imágenes AMI de AWS "doradas" que nadie recordaba cómo se habían creado. La pesadilla operativa era la norma.

### 9.1.2 La promesa de Docker

Docker introdujo un concepto radical: **el entorno es código**. El `Dockerfile` no solo describe cómo construir la aplicación; describe el entorno completo donde esa aplicación vive. Kernel, librerías del sistema, runtime del lenguaje, dependencias, variables de entorno, usuario, permisos — todo está declarado de forma explícita y reproducible.

Con Docker:

```
Desarrollador       →    docker build    →    imagen inmutable
Servidor de CI      →    docker build    →    misma imagen inmutable
Staging             →    docker run      →    misma imagen inmutable
Producción          →    docker run      →    misma imagen inmutable
```

La **misma** imagen viaja por todos los entornos. No reconstruyes en cada etapa; construyes una vez, ejecutas en todas partes. Esto elimina de raíz la deriva de configuración.

### 9.1.3 Beneficios concretos

**Builds reproducibles**

El mismo `Dockerfile` + el mismo contexto de build = la misma imagen (bit por bit, si usas `--provenance` y builds determinísticos). Si un build falla, puedes reproducirlo localmente con `docker build` y depurarlo. Si un build pasa en CI, pasarás también en producción porque es la misma imagen.

```dockerfile
FROM node:20.11.1-alpine3.19@sha256:ee0d91e38c...
RUN apk add --no-cache python3 make g++
COPY package*.json ./
RUN npm ci --production
COPY . .
```

Cada línea es explícita. No hay "depende de lo que esté instalado en el host".

**Tests aislados**

Cada job de CI ejecuta tests en un contenedor limpio. No hay contaminación entre ejecuciones. No hay estado residual de jobs anteriores. Los tests de integración pueden levantar bases de datos, colas de mensajes y servicios mock como contenedores efímeros:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    env:
      POSTGRES_PASSWORD: test
  redis:
    image: redis:7-alpine
```

Al terminar el job, los contenedores se destruyen. El entorno queda impoluto.

**Artefactos inmutables**

Una imagen Docker es un artefacto inmutable. Una vez construida, su hash SHA256 la identifica de forma única. Nadie puede modificar una imagen después del build. Esto es fundamental para auditoría, compliance y debugging: sabes exactamente qué código y qué dependencias estaban presentes en el momento del despliegue.

```bash
docker inspect myapp:1.2.3 --format '{{.RepoDigests}}'
# [myrepo/myapp@sha256:abc123def456...]
```

Ese hash es tu garantía de que el artefacto no ha sido alterado.

### 9.1.4 El pipeline ideal con Docker

```
┌──────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐    ┌────────────┐
│  Commit  │ →  │   Lint   │ →  │   Test    │ →  │  Build   │ →  │   Push     │
│  & Push  │    │          │    │           │    │  Image   │    │  Registry  │
└──────────┘    └──────────┘    └───────────┘    └──────────┘    └────────────┘
                                                                       │
                                                                       ▼
                                                              ┌────────────┐
                                                              │  Deploy    │
                                                              │  Strategy  │
                                                              └────────────┘
```

Cada etapa produce un artefacto verificable. Si alguna falla, la tubería se detiene y notifica al equipo. Si todo pasa, la imagen viaja al registro y se despliega automáticamente.

---

## 9.2 GitHub Actions

GitHub Actions es el sistema de CI/CD nativo de GitHub. Lanzado en 2019, se ha convertido en la opción predilecta de millones de desarrolladores por su integración perfecta con el ecosistema GitHub y su marketplace de acciones reutilizables.

### 9.2.1 Conceptos fundamentales

**Workflows**

Un workflow es un pipeline automatizado definido en un archivo YAML dentro de `.github/workflows/`. Se dispara por eventos: push, pull_request, schedule (cron), workflow_dispatch (manual) y muchos más.

```
.github/
└── workflows/
    ├── ci.yml          # Integración continua
    ├── cd.yml          # Despliegue continuo
    ├── release.yml     # Release automatizado
    └── security.yml    # Escaneo de seguridad
```

Cada archivo YAML define un workflow independiente. Puedes tener workflows activos simultáneamente, cada uno respondiendo a diferentes eventos.

**Jobs**

Un workflow contiene uno o más jobs que se ejecutan en paralelo por defecto. Cada job tiene su propio runner (máquina virtual) y su propio sistema de archivos. Puedes definir dependencias entre jobs con `needs`:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    # ... (ejecuta en paralelo con test)

  test:
    runs-on: ubuntu-latest
    # ...

  build:
    needs: [lint, test]    # Espera a que lint y test terminen exitosamente
    runs-on: ubuntu-latest
    # ...
```

**Steps**

Cada job está compuesto de steps. Un step puede ser:
- Una **acción** (`uses: actions/checkout@v4`)
- Un **comando shell** (`run: npm test`)

Los steps se ejecutan secuencialmente dentro del job. Si un step falla, el job se detiene (a menos que configures `continue-on-error`).

**Actions**

Una action es una unidad reutilizable de código. El marketplace de GitHub contiene miles de acciones mantenidas por la comunidad y por GitHub mismo:

- `actions/checkout` — clona el repositorio
- `actions/setup-node` — instala Node.js
- `docker/setup-buildx-action` — configura Buildx
- `docker/login-action` — autentica en un registro
- `aws-actions/configure-aws-credentials` — configura credenciales AWS

Las acciones se referencian por `{owner}/{repo}@{ref}`:

```yaml
- uses: actions/checkout@v4
- uses: docker/setup-buildx-action@v3
```

**Runners**

Un runner es el agente que ejecuta los jobs. GitHub ofrece runners hospedados (`ubuntu-latest`, `windows-latest`, `macos-latest`) con herramientas preinstaladas. Para necesidades específicas, puedes usar **self-hosted runners**: máquinas propias registradas en GitHub, útiles cuando necesitas acceso a redes privadas, GPUs, o hardware específico.

| Runner | OS | CPU | RAM | SSD |
|--------|----|-----|-----|-----|
| ubuntu-latest | Ubuntu 22.04 | 4 vCPU | 16 GB | 150 GB |
| windows-latest | Windows Server 2022 | 4 vCPU | 16 GB | 150 GB |
| macos-latest | macOS 14 (M1) | 4 vCPU | 14 GB | 150 GB |

### 9.2.2 Flujo completo para una aplicación

Veamos el flujo completo paso a paso, desde el checkout hasta el despliegue:

#### Paso 1: Checkout del código

```yaml
- name: Checkout código
  uses: actions/checkout@v4
  with:
    fetch-depth: 0    # Clonar todo el historial (necesario para versionado automático)
```

`fetch-depth: 0` clona el historial completo de Git, necesario para herramientas como `semantic-release` que analizan commits convencionales.

#### Paso 2: Setup del lenguaje

Cada lenguaje tiene su acción oficial de setup:

**Node.js**

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
```

**Python**

```yaml
- name: Setup Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
```

**Java (Maven/Gradle)**

```yaml
- name: Setup Java
  uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'maven'        # o 'gradle'
```

**Go**

```yaml
- name: Setup Go
  uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    cache-dependency-path: "**/go.sum"
```

El parámetro `cache` ahorra minutos en cada build al preservar las dependencias entre ejecuciones.

#### Paso 3: Lint + Test

```yaml
- name: Instalar dependencias
  run: npm ci

- name: Lint
  run: npm run lint

- name: Test
  run: npm test -- --coverage
```

La diferencia entre `npm install` y `npm ci` es crucial en CI: `npm ci` es determinístico, borra `node_modules` antes de instalar, y falla si `package.json` y `package-lock.json` están desincronizados.

#### Paso 4: Build de imagen Docker

```yaml
- name: Configurar Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build imagen Docker
  uses: docker/build-push-action@v6
  with:
    context: .
    push: false              # No pusheamos todavía
    load: true               # Cargar en Docker local
    tags: myapp:test
    cache-from: type=gha     # Cache de GitHub Actions
    cache-to: type=gha,mode=max
```

La acción `docker/build-push-action` de Docker abstrae toda la complejidad de Buildx. Con `cache-from` y `cache-to` usando el cache de GitHub Actions (`type=gha`), las capas de Docker se reutilizan entre builds, reduciendo tiempos drásticamente.

#### Paso 5: Push a Registry

```yaml
- name: Login a Docker Hub
  if: github.ref == 'refs/heads/main'
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Build y Push a Docker Hub
  if: github.ref == 'refs/heads/main'
  uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: |
      myorg/myapp:latest
      myorg/myapp:${{ github.sha }}
    platforms: linux/amd64,linux/arm64
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

Etiquetamos con `latest` y con el SHA del commit (`github.sha`) para trazabilidad. La imagen es multi-arquitectura (`linux/amd64,linux/arm64`), cubriendo tanto servidores x86 tradicionales como instancias ARM (Graviton, Apple Silicon).

#### Paso 6: Deploy

```yaml
deploy:
  needs: build-and-push
  runs-on: ubuntu-latest
  environment: production

  steps:
    - name: Desplegar en Kubernetes
      uses: azure/k8s-deploy@v4
      with:
        manifests: k8s/deployment.yaml
        images: myorg/myapp:${{ github.sha }}
```

### 9.2.3 Workflow YAML completo

El siguiente workflow representa un pipeline de CI/CD completo para una aplicación Node.js. Cada step está comentado línea por línea:

```yaml
name: CI/CD Pipeline

# ─── Eventos que disparan el workflow ─────────────────────────
on:
  push:
    branches: [main, develop]
    tags: ['v*']                    # Tags de versión
  pull_request:
    branches: [main]
  workflow_dispatch:                # Ejecución manual desde UI
    inputs:
      environment:
        description: 'Entorno de despliegue'
        required: true
        type: choice
        options: [staging, production]

# ─── Variables de entorno globales ─────────────────────────────
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}    # usuario/repo

# ─── Jobs ──────────────────────────────────────────────────────
jobs:

  # ── Job 1: Lint ──────────────────────────────────────────────
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5                    # Timeout de seguridad

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      - name: Ejecutar linter
        run: npm run lint

  # ── Job 2: Tests con matriz de versiones ────────────────────
  test:
    needs: lint                           # Solo si lint pasa
    runs-on: ubuntu-latest
    timeout-minutes: 15

    strategy:
      fail-fast: false                    # No cancelar si una falla
      matrix:
        node-version: [18, 20, 22]        # Testear múltiples versiones

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      - name: Ejecutar tests
        run: npm test -- --coverage
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

      - name: Subir cobertura a Codecov
        if: matrix.node-version == '20'   # Solo una vez
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: true

  # ── Job 3: Escaneo de seguridad ─────────────────────────────
  security-scan:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Escaneo de secretos con truffleHog
        uses: trufflesecurity/trufflehog-action@v1
        with:
          path: ./
          base: ${{ github.event.before }}
          head: ${{ github.sha }}
          extra_args: --only-verified

      - name: Escaneo de dependencias con npm audit
        run: npm audit --audit-level=high

  # ── Job 4: Build y Push de imagen Docker ────────────────────
  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      packages: write       # Para push a GHCR
      id-token: write       # Para OIDC (firma con Cosign opcional)

    steps:
      - name: Checkout código
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: |
            image=moby/buildkit:latest
            network=host

      - name: Cache de capas Docker
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      - name: Login en GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Login en Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extraer metadatos Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
            docker.io/${{ secrets.DOCKERHUB_USERNAME }}/myapp
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=,format=short
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build y Push imagen multi-arquitectura
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          platforms: linux/amd64,linux/arm64
          provenance: true           # Generar SLSA provenance
          sbom: true                 # Generar SBOM (Software Bill of Materials)
          cache-from: type=gha,scope=buildkit
          cache-to: type=gha,scope=buildkit,mode=max

      - name: Firmar imagen con Cosign
        if: github.ref == 'refs/heads/main'
        run: |
          cosign sign --yes \
            --key env://COSIGN_PRIVATE_KEY \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
        env:
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}

      - name: Escanear imagen con Trivy
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: 1              # Fallar si hay vulnerabilidades críticas

      - name: Subir resultados de Trivy a GitHub
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  # ── Job 5: Deploy a Staging (automático) ────────────────────
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Configurar kubectl
        uses: azure/setup-kubectl@v4

      - name: Configurar kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > $HOME/.kube/config

      - name: Desplegar en Kubernetes (staging)
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace=staging
          kubectl rollout status deployment/myapp --namespace=staging --timeout=5m

      - name: Notificar en Slack (éxito staging)
        if: success()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {"text": "Despliegue exitoso en staging\nApp: myapp\nVersión: ${{ github.sha }}\nEntorno: staging\nAutor: ${{ github.actor }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # ── Job 6: Deploy a Producción (con aprobación manual) ─────
  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com

    steps:
      - name: Esperar aprobación manual
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: ${{ vars.PROD_APPROVERS }}
          minimum-approvals: 1
          issue-body: "Despliegue a producción de myapp:${{ github.sha }}"

      - name: Verificar firma de imagen
        run: |
          cosign verify \
            --key cosign.pub \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.build-and-push.outputs.digest }}

      - name: Configurar kubectl
        uses: azure/setup-kubectl@v4

      - name: Configurar kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_PRODUCTION }}" | base64 -d > $HOME/.kube/config

      - name: Desplegar en Kubernetes (producción) - Rolling Update
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace=production
          kubectl rollout status deployment/myapp --namespace=production --timeout=10m

      - name: Rollback automático si falla
        if: failure()
        run: |
          kubectl rollout undo deployment/myapp --namespace=production
          echo "ROLLBACK ejecutado. La versión anterior ha sido restaurada."

      - name: Notificar en Slack (éxito producción)
        if: success()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {"text": "Despliegue exitoso en producción\nApp: myapp\nVersión: ${{ github.sha }}\nDisponible en https://myapp.com"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: Notificar en Slack (fallo producción)
        if: failure()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {"text": "DESPLIEGUE FALLIDO en producción\nApp: myapp\nVersión: ${{ github.sha }}\nSe ejecutó rollback automático."}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # ── Job 7: Release automático con semantic-release ──────────
  release:
    if: github.ref == 'refs/heads/main'
    needs: deploy-production
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout código
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GH_RELEASE_TOKEN }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Instalar dependencias
        run: npm ci

      - name: Ejecutar semantic-release
        run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GH_RELEASE_TOKEN }}
```

### 9.2.4 Caché de capas Docker

Uno de los aspectos más impactantes en el tiempo de build es el caché de capas Docker. Sin caché, cada build descarga capas base, reinstala dependencias y recompila desde cero. Con un caché bien configurado, los builds incrementales pueden ser **10-50x más rápidos**.

Docker Buildx soporta varios backends de caché:

**GitHub Actions Cache (`type=gha`)**

El más simple e integrado. Las capas se almacenan en el cache de GitHub Actions:

```yaml
- uses: docker/build-push-action@v6
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Limitación**: el cache de GitHub Actions tiene un límite de 10 GB por repositorio y las entradas inactivas se eliminan después de 7 días.

**Registro de contenedores como caché (`type=registry`)**

Para cachés persistentes y compartidos entre equipos, usa un registro:

```yaml
- uses: docker/build-push-action@v6
  with:
    cache-from: type=registry,ref=ghcr.io/myorg/myapp:buildcache
    cache-to: type=registry,ref=ghcr.io/myorg/myapp:buildcache,mode=max
```

La etiqueta `buildcache` se actualiza en cada build, conteniendo las capas intermedias. Los builds posteriores descargan solo las capas que han cambiado.

**Estrategia de cacheado en el Dockerfile**

El orden de las instrucciones en el Dockerfile determina la eficiencia del caché:

```dockerfile
# BUENO: capa de dependencias separada ────────────────────────
FROM node:20-alpine
WORKDIR /app

# Copiar manifiestos primero (capa que cambia poco)
COPY package*.json ./
RUN npm ci --production

# Copiar código después (capa que cambia frecuentemente)
COPY . .

# MALO: todo en una capa ──────────────────────────────────────
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci    # El caché se invalida con cualquier cambio de código
```

**Modo `max` vs `min`**

```yaml
cache-to: type=gha,mode=max    # Cachea TODAS las capas (incluyendo RUN)
cache-to: type=gha,mode=min    # Cachea solo las capas del stage final
```

`mode=max` es ideal para CI porque cachea capas intermedias de builds multi-etapa, pero consume más espacio. `mode=min` es más ligero pero solo cachea las capas de la imagen final.

**Métrica real**

En un proyecto Node.js típico (~500 dependencias, ~10k líneas de código), los tiempos de build con y sin caché:

| Escenario | Sin caché | Con caché (gha) | Con caché (registry) |
|-----------|-----------|-----------------|-----------------------|
| Primer build | 4m 32s | 4m 38s (+overhead) | 4m 40s |
| Build subsiguiente (sin cambios) | 4m 30s | 12s | 15s |
| Build con cambio en código | 4m 30s | 45s | 48s |
| Build con cambio en dependencias | 4m 30s | 2m 10s | 2m 12s |

### 9.2.5 Multi-arch builds

Las imágenes multi-arquitectura son esenciales en 2026. Con la proliferación de instancias ARM (AWS Graviton, Apple Silicon, Raspberry Pi), una imagen `linux/amd64` ya no es suficiente.

Buildx crea imágenes multi-arquitectura mediante **QEMU emulation** o **native builders**:

```yaml
- name: Setup QEMU (para emulación ARM en runners x86)
  uses: docker/setup-qemu-action@v3
  with:
    platforms: arm64,arm

- name: Setup Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build multi-arquitectura
  uses: docker/build-push-action@v6
  with:
    platforms: linux/amd64,linux/arm64,linux/arm/v7
    push: true
    tags: myorg/myapp:latest
```

**Cómo funciona internamente**

Cuando Buildx recibe `--platform linux/amd64,linux/arm64`:

1. Crea un builder multi-nodo (puede ser un solo nodo con QEMU).
2. Para cada plataforma, ejecuta las instrucciones del Dockerfile.
3. Si la plataforma del runner difiere de la target, QEMU emula la arquitectura (por ejemplo, corre `aarch64` en un runner `x86_64`).
4. Genera un **manifest list** (también llamado **fat manifest**) que referencia los digests de cada arquitectura.
5. Al hacer `docker pull myorg/myapp:latest`, Docker elige automáticamente la arquitectura correcta según el host.

**Verificación de la imagen multi-arquitectura**

```bash
docker buildx imagetools inspect myorg/myapp:latest

# Output:
# Name:      docker.io/myorg/myapp:latest
# MediaType: application/vnd.oci.image.index.v1+json
#
# Manifests:
#   Name:        docker.io/myorg/myapp:latest@sha256:abc...
#   MediaType:   application/vnd.oci.image.manifest.v1+json
#   Platform:    linux/amd64
#
#   Name:        docker.io/myorg/myapp:latest@sha256:def...
#   MediaType:   application/vnd.oci.image.manifest.v1+json
#   Platform:    linux/arm64
```

**Builders nativos vs emulación**

| Método | Pros | Contras |
|--------|------|---------|
| QEMU | Funciona en cualquier runner | Lento (emulación pura, ~4-10x más lento que nativo) |
| Native node | Rápido (compilación nativa) | Requiere runners ARM dedicados |
| Cross-compilation | Rápido | Solo para lenguajes que lo soportan (Go, Rust) |

**Ejemplo avanzado: builder híbrido**

Si tienes runners auto-hospedados ARM:

```bash
# En el runner ARM
docker buildx create --name multi-arch \
  --node amd64-node --platform linux/amd64 \
  --node arm64-node --platform linux/arm64

docker buildx use multi-arch
docker buildx build --platform linux/amd64,linux/arm64 --push -t myorg/myapp .
```

Cada nodo construye nativamente su plataforma. La imagen resultante es idéntica pero el build es sustancialmente más rápido.

### 9.2.6 Matrix builds

La estrategia `matrix` ejecuta el mismo job con diferentes combinaciones de parámetros. Es ideal para probar compatibilidad con múltiples versiones de runtime, sistemas operativos y configuraciones:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false      # Continuar con otras combinaciones si una falla
      max-parallel: 4       # Limitar concurrencia
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest]
        db: [postgres, mysql, sqlite]
        exclude:            # Excluir combinaciones inválidas
          - node-version: 18
            db: mysql
        include:            # Incluir combinaciones extras
          - node-version: 22
            os: windows-latest
            db: sqlite

    services:
      postgres:
        if: matrix.db == 'postgres'
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: test
        ports: [5432:5432]

      mysql:
        if: matrix.db == 'mysql'
        image: mysql:8.4
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: testdb
        ports: [3306:3306]
        options: >-
          --health-cmd "mysqladmin ping -h localhost"
          --health-interval 10s

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test
        env:
          DB: ${{ matrix.db }}
```

Este ejemplo genera 9 combinaciones de test (3 versiones de Node × 3 bases de datos) más 1 combinación extra en Windows, para un total de 10 ejecuciones paralelas. Con `fail-fast: false`, incluso si una combinación falla, las demás continúan y puedes ver exactamente qué combinaciones son problemáticas.

**Filtros condicionales**

Puedes excluir combinaciones con `exclude` (no ejecuta esa combinación) o incluir combinaciones adicionales con `include`. También puedes usar condicionales en los steps para solo ejecutar ciertas acciones en ciertas combinaciones:

```yaml
- name: Pruebas de rendimiento (solo Node 22)
  if: matrix.node-version == '22' && matrix.db == 'postgres'
  run: npm run benchmark
```

### 9.2.7 Secrets

Los secrets son variables encriptadas que GitHub almacena de forma segura y expone a los workflows en tiempo de ejecución. **Nunca** se muestran en logs; si accidentalmente intentas imprimir un secret, GitHub lo redacta como `***`.

**Configuración**

Los secrets se configuran en Settings → Secrets and variables → Actions:

```
Repositorio: myorg/myapp
├── Actions secrets and variables
│   ├── Secrets
│   │   ├── DOCKERHUB_TOKEN        # Token de acceso de Docker Hub
│   │   ├── AWS_ACCESS_KEY_ID      # AWS IAM
│   │   ├── AWS_SECRET_ACCESS_KEY  # AWS IAM
│   │   ├── KUBE_CONFIG_STAGING   # kubeconfig base64
│   │   ├── KUBE_CONFIG_PRODUCTION # kubeconfig base64
│   │   ├── SLACK_WEBHOOK          # URL de webhook de Slack
│   │   ├── COSIGN_PRIVATE_KEY     # Clave privada de Cosign
│   │   ├── COSIGN_PASSWORD        # Password de la clave Cosign
│   │   └── GH_RELEASE_TOKEN      # PAT para semantic-release
│   └── Variables
│       ├── PROD_APPROVERS         # user1,user2
│       └── AWS_REGION             # us-east-1
```

Diferencia entre Secrets y Variables:
- **Secrets**: encriptados, ocultos en logs, acceso restringido por Environments.
- **Variables**: no encriptadas, visibles en logs, útiles para configuración no sensible.

**Uso en workflows**

```yaml
- name: Login a Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Configurar AWS
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ vars.AWS_REGION }}
```

**Secrets en entornos**

Los secrets pueden ser específicos de un entorno. Por ejemplo, `KUBE_CONFIG` puede tener valores diferentes para `staging` y `production`:

```
Environment: staging
  ├── KUBE_CONFIG = (config del cluster de staging)
  └── SLACK_WEBHOOK = (canal #deploy-staging)

Environment: production
  ├── KUBE_CONFIG = (config del cluster de producción)
  └── SLACK_WEBHOOK = (canal #deploy-prod)
```

**Buenas prácticas**

1. **Nunca hardcodear secretos.** Ni en el código fuente, ni en el workflow YAML, ni en variables de entorno no encriptadas.

2. **Usar el principio de mínimo privilegio.** Crea tokens con los permisos mínimos necesarios. Por ejemplo, el token de Docker Hub solo necesita permisos de lectura/escritura en el repositorio específico, no acceso administrativo.

3. **Rotar secretos periódicamente.** Especialmente tokens de acceso personal (PAT) y claves de API.

4. **Usar OIDC (OpenID Connect) en lugar de secretos de larga duración.** Para AWS, Azure y GCP, GitHub Actions puede autenticarse mediante OIDC, eliminando la necesidad de almacenar credenciales:

```yaml
- name: Configurar AWS via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
    aws-region: us-east-1
    # Sin access key ni secret key — OIDC genera credenciales temporales
```

5. **No uses secrets en steps condicionales con `if:`**. Los secrets no están disponibles en contextos condicionales; usa `vars` en su lugar.

### 9.2.8 Environments

Los entornos (Environments) en GitHub Actions permiten segmentar despliegues con reglas de protección específicas:

**Creación de un entorno**

En Settings → Environments:

```
Environment: production
├── Protection rules
│   ├── Required reviewers: [alice, bob] (mínimo 1 aprobación)
│   ├── Wait timer: 0 minutes (espera antes de ejecutar)
│   ├── Deployment branches: main (solo desde esta rama)
│   └── Custom deployment protection rules: (apps de terceros)
├── Environment secrets
│   └── KUBE_CONFIG = ...
└── Environment variables
    └── CLUSTER_URL = https://prod-cluster.example.com
```

**Uso en workflow**

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com    # URL del despliegue, visible en UI
    steps:
      - name: Desplegar
        run: kubectl apply -f k8s/production.yaml
        env:
          KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```

Cuando el workflow llega al job `deploy-prod`, GitHub:
1. Verifica que la rama sea `main` (según la regla de deployment branches).
2. Verifica que el timer haya transcurrido.
3. Notifica a los reviewers (alice, bob).
4. Al menos uno debe aprobar el despliegue desde la UI de GitHub.
5. Una vez aprobado, los secrets del entorno se desencriptan y el job se ejecuta.

**Protección por ramas**

Puedes restringir qué ramas pueden desplegar a cada entorno:

```yaml
on:
  push:
    branches: [main, develop]

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'    # staging solo desde develop
    environment: staging

  deploy-production:
    if: github.ref == 'refs/heads/main'       # producción solo desde main
    environment: production
```

**Environments vs Environment Secrets**

Los secrets configurados a nivel de repositorio están disponibles en todos los jobs. Los secrets de entorno solo están disponibles en jobs que referencian ese entorno específico. Esto previene que un job de staging use accidentalmente credenciales de producción.

### 9.2.9 Ejemplo completo: Node.js → Test → Build → Push → Deploy a Kubernetes

Este ejemplo representa un flujo de producción completo para una API REST en Node.js con Express, desplegada en un clúster de Kubernetes:

**Estructura del proyecto**

```
myapp/
├── .github/
│   └── workflows/
│       └── ci-cd.yml          # Workflow principal
├── src/
│   ├── app.js
│   ├── routes/
│   └── services/
├── tests/
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── overlays/
│   │   ├── staging/
│   │   │   ├── kustomization.yaml
│   │   │   └── patch-env.yaml
│   │   └── production/
│   │       ├── kustomization.yaml
│   │       └── patch-env.yaml
├── Dockerfile
├── docker-compose.yml
├── package.json
└── .dockerignore
```

**Dockerfile optimizado**

```dockerfile
# ─── Stage 1: Builder ────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# ─── Stage 2: Production ─────────────────────────────────────
FROM node:20-alpine AS production
WORKDIR /app

RUN apk add --no-cache tini curl
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY package*.json ./
RUN npm ci --production && npm cache clean --force

COPY --from=builder /app/dist ./dist

USER appuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/app.js"]
```

**Deployment de Kubernetes**

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0    # Zero-downtime deployment
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:latest
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

**Workflow CI/CD completo**

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline - Node.js a Kubernetes

on:
  push:
    branches: [main, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.gitignore'
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Entorno de despliegue'
        type: choice
        options: [staging, production]
        required: true

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  K8S_NAMESPACE_STAGING: staging
  K8S_NAMESPACE_PRODUCTION: production

jobs:
  # ── Lint y Análisis Estático ─────────────────────────────────
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: ESLint
        run: npm run lint

      - name: TypeScript check
        run: npm run typecheck

      - name: Prettier check
        run: npx prettier --check 'src/**/*.ts'

  # ── Tests ────────────────────────────────────────────────────
  test:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 15

    strategy:
      matrix:
        node-version: [18, 20, 22]

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: app_test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci

      - name: Unit tests
        run: npm test

      - name: Integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/app_test
          REDIS_URL: redis://localhost:6379

      - name: Coverage report
        if: matrix.node-version == '20'
        run: npm run test:coverage

  # ── Escaneo de Seguridad ────────────────────────────────────
  security:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4

      - name: Detectar secretos en código
        uses: trufflesecurity/trufflehog-action@v1
        with:
          path: ./
          base: ${{ github.event.before }}
          head: ${{ github.sha }}
          extra_args: --only-verified

      - name: npm audit
        run: |
          npm audit --audit-level=high || \
          echo "::warning::Vulnerabilidades detectadas en dependencias"

      - name: Trivy scan del Dockerfile
        uses: aquasecurity/trivy-action@0.24.0
        with:
          scan-type: 'config'
          scan-ref: './Dockerfile'
          severity: 'CRITICAL,HIGH'

  # ── Build y Push ────────────────────────────────────────────
  build-and-push:
    needs: [test, security]
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      packages: write
      id-token: write

    outputs:
      digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: docker/setup-qemu-action@v3
        with:
          platforms: arm64

      - uses: docker/setup-buildx-action@v3
        with:
          driver-opts: network=host

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,format=short
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
            type=raw,value=staging,enable=${{ github.ref == 'refs/heads/develop' }}

      - id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          platforms: linux/amd64,linux/arm64
          provenance: true
          sbom: true
          cache-from: type=gha,scope=node-app
          cache-to: type=gha,scope=node-app,mode=max

      - name: Firmar imagen con Cosign
        if: github.ref == 'refs/heads/main'
        run: |
          cosign sign --yes \
            --key env://COSIGN_PRIVATE_KEY \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
        env:
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}

  # ── Deploy a Staging ────────────────────────────────────────
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.example.com

    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-kubectl@v4

      - name: Configurar kubeconfig (staging)
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      - name: Validar conexión al clúster
        run: kubectl cluster-info --namespace=${{ env.K8S_NAMESPACE_STAGING }}

      - name: Actualizar imagen en deployment
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace=${{ env.K8S_NAMESPACE_STAGING }}

      - name: Esperar rollout exitoso
        run: |
          kubectl rollout status deployment/myapp \
            --namespace=${{ env.K8S_NAMESPACE_STAGING }} \
            --timeout=5m

      - name: Smoke test
        run: |
          kubectl port-forward svc/myapp 8080:80 \
            --namespace=${{ env.K8S_NAMESPACE_STAGING }} &
          sleep 5
          curl -f http://localhost:8080/health
          kill %1

      - name: Notificar despliegue exitoso
        if: success()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {"text": "Despliegue en staging exitoso\nApp: myapp\nCommit: ${{ github.sha }}\nURL: https://staging.myapp.example.com\nAutor: ${{ github.actor }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_STAGING }}

  # ── Deploy a Producción ─────────────────────────────────────
  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Verificar firma de imagen
        run: |
          cosign verify \
            --key cosign.pub \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.build-and-push.outputs.digest }}

      - uses: azure/setup-kubectl@v4

      - name: Configurar kubeconfig (producción)
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_PRODUCTION }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      - name: Validar conexión al clúster
        run: kubectl cluster-info --namespace=${{ env.K8S_NAMESPACE_PRODUCTION }}

      - name: Actualizar imagen (Rolling Update)
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --namespace=${{ env.K8S_NAMESPACE_PRODUCTION }} \
            --record

      - name: Monitorear despliegue
        run: |
          kubectl rollout status deployment/myapp \
            --namespace=${{ env.K8S_NAMESPACE_PRODUCTION }} \
            --timeout=10m

      - name: Rollback automático si falla
        if: failure()
        run: |
          echo "::error::Despliegue fallido, ejecutando rollback..."
          kubectl rollout undo deployment/myapp \
            --namespace=${{ env.K8S_NAMESPACE_PRODUCTION }}
          kubectl rollout status deployment/myapp \
            --namespace=${{ env.K8S_NAMESPACE_PRODUCTION }} \
            --timeout=5m

      - name: Notificar despliegue a producción
        if: success()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {"text": "Despliegue en produccion completado\nApp: myapp\nVersion: ${{ github.sha }}\nURL: https://myapp.example.com\nAutor: ${{ github.actor }}\nDashboard: https://grafana.example.com/d/myapp"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_PRODUCTION }}

  # ── Release Automático ──────────────────────────────────────
  release:
    if: github.ref == 'refs/heads/main'
    needs: deploy-production
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GH_RELEASE_TOKEN }}

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci

      - name: semantic-release
        run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GH_RELEASE_TOKEN }}

  # ── Notificación en Discord ─────────────────────────────────
  notify:
    needs: [deploy-staging, deploy-production, release]
    if: always() && (needs.deploy-production.result == 'success' || needs.deploy-production.result == 'failure' || needs.deploy-staging.result == 'success')
    runs-on: ubuntu-latest

    steps:
      - name: Notificar en Discord
        uses: tsickert/discord-webhook-action@v5
        with:
          webhook-url: ${{ secrets.DISCORD_WEBHOOK }}
          title: "CI/CD Pipeline - ${{ job.status }}"
          description: |
            **Repositorio:** ${{ github.repository }}
            **Rama:** ${{ github.ref_name }}
            **Commit:** ${{ github.sha }}
            **Actor:** ${{ github.actor }}
            **Workflow:** ${{ github.workflow }}
            **URL:** ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          color: ${{ job.status == 'success' && 65280 || 16711680 }}
```

### 9.2.10 Ejemplo completo: Java Spring Boot → Maven → Docker Build → Push ECR → Deploy ECS

El ecosistema Java tiene necesidades particulares en CI/CD: compilación Maven/Gradle, optimización de capas para dependencias que cambian poco, y despliegue en servicios gestionados de AWS como ECS (Elastic Container Service):

**Estructura del proyecto**

```
spring-app/
├── .github/workflows/
│   └── ci-cd-java.yml
├── src/
│   ├── main/
│   │   ├── java/com/springapp/
│   │   └── resources/
│   └── test/
├── pom.xml
├── Dockerfile
└── .dockerignore
```

**Dockerfile optimizado para Spring Boot**

```dockerfile
# ─── Stage 1: Build con Maven ────────────────────────────────
FROM maven:3.9-eclipse-temurin-21-alpine AS builder
WORKDIR /app

# Descargar dependencias primero (capa cacheable)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Compilar el proyecto
COPY src ./src
RUN mvn package -DskipTests -B

# Extraer layers para optimización de cache
RUN java -Djarmode=layertools -jar target/*.jar extract

# ─── Stage 2: Runtime ────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

RUN apk add --no-cache curl
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copiar layers de forma ordenada (cambios infrecuentes primero)
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

USER appuser
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**Workflow CI/CD para Java + AWS ECS**

```yaml
# .github/workflows/ci-cd-java.yml
name: CI/CD - Java Spring Boot a AWS ECS

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: spring-app
  ECS_CLUSTER: spring-app-cluster
  ECS_SERVICE: spring-app-service
  ECS_TASK_DEFINITION: spring-app-task

jobs:
  # ── Maven Build y Test ───────────────────────────────────────
  build-and-test:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    strategy:
      matrix:
        java-version: ['21']

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: spring
          POSTGRES_PASSWORD: spring
          POSTGRES_DB: springdb
        ports: ['5432:5432']

      localstack:
        image: localstack/localstack:3
        env:
          SERVICES: sqs,sns,s3
          DEBUG: 0
        ports: ['4566:4566']

    steps:
      - uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Cache Maven local
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-maven-

      - name: Compilar (sin tests)
        run: mvn compile -B -q

      - name: Checkstyle + SpotBugs
        run: mvn checkstyle:check spotbugs:check -B

      - name: Tests unitarios
        run: mvn test -B

      - name: Tests de integración
        run: mvn verify -B -P integration
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/springdb
          SPRING_DATASOURCE_USERNAME: spring
          SPRING_DATASOURCE_PASSWORD: spring
          AWS_ENDPOINT_URL: http://localhost:4566

      - name: Publicar resultados de tests
        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: Maven Tests
          path: target/surefire-reports/*.xml
          reporter: java-junit

  # ── Docker Build y Push a ECR ──────────────────────────────
  docker-build-and-push:
    needs: build-and-test
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      id-token: write    # OIDC para AWS

    steps:
      - uses: actions/checkout@v4

      - name: Configurar AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-ecr
          aws-region: ${{ env.AWS_REGION }}

      - name: Login a Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
        with:
          registry-type: private

      - uses: docker/setup-buildx-action@v3

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=sha,prefix=,format=short
            type=ref,event=branch
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          platforms: linux/amd64,linux/arm64
          provenance: true
          sbom: true
          cache-from: type=gha,scope=spring-app
          cache-to: type=gha,scope=spring-app,mode=max

  # ── Escaneo de Vulnerabilidades (ECR Scan) ──────────────────
  ecr-scan:
    needs: docker-build-and-push
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Configurar AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-ecr
          aws-region: ${{ env.AWS_REGION }}

      - name: Iniciar escaneo ECR
        run: |
          aws ecr start-image-scan \
            --repository-name ${{ env.ECR_REPOSITORY }} \
            --image-id imageTag=${{ github.sha }}

      - name: Esperar y verificar resultados
        run: |
          for i in $(seq 1 30); do
            RESULT=$(aws ecr describe-image-scan-findings \
              --repository-name ${{ env.ECR_REPOSITORY }} \
              --image-id imageTag=${{ github.sha }} \
              --query 'imageScanStatus.status' --output text)

            if [ "$RESULT" = "COMPLETE" ]; then
              echo "Escaneo completado"
              break
            fi

            echo "Esperando escaneo... intento $i/30"
            sleep 10
          done

          CRITICAL=$(aws ecr describe-image-scan-findings \
            --repository-name ${{ env.ECR_REPOSITORY }} \
            --image-id imageTag=${{ github.sha }} \
            --query 'imageScanFindings.findingSeverityCounts.CRITICAL' \
            --output text)

          if [ "$CRITICAL" != "None" ] && [ "$CRITICAL" -gt 0 ]; then
            echo "::error::Se encontraron $CRITICAL vulnerabilidades CRITICAS"
            exit 1
          fi

          echo "No se encontraron vulnerabilidades criticas"

  # ── Deploy a ECS ────────────────────────────────────────────
  deploy-ecs:
    needs: ecr-scan
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.springapp.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Configurar AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Login a ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Descargar task definition actual
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_TASK_DEFINITION }} \
            --query taskDefinition > task-definition.json

      - name: Actualizar imagen en task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: spring-app
          image: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

      - name: Desplegar en ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true

      - name: Rollback si falla
        if: failure()
        run: |
          aws ecs update-service \
            --cluster ${{ env.ECS_CLUSTER }} \
            --service ${{ env.ECS_SERVICE }} \
            --force-new-deployment

      - name: Notificar en Slack
        if: always()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {"text": "${{ job.status == 'success' && 'Spring Boot desplegado exitosamente en ECS' || 'Fallo el despliegue de Spring Boot' }}\nCluster: ${{ env.ECS_CLUSTER }}\nServicio: ${{ env.ECS_SERVICE }}\nCommit: ${{ github.sha }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 9.3 GitLab CI

GitLab CI es el sistema de CI/CD integrado en GitLab. A diferencia de GitHub Actions (que requiere archivos separados por workflow), GitLab CI se define en un único archivo `.gitlab-ci.yml` en la raíz del repositorio. Su integración con GitLab Registry, Auto DevOps y Kubernetes lo convierten en una opción muy potente para equipos que ya usan GitLab.

### 9.3.1 Conceptos fundamentales

**Pipeline**

Un pipeline es el conjunto de jobs organizados en stages. El pipeline se dispara automáticamente en cada push, merge request, o evento de tag:

```yaml
stages:
  - build
  - test
  - security
  - package
  - deploy
```

Los stages se ejecutan secuencialmente. Los jobs dentro de un mismo stage pueden ejecutarse en paralelo (si hay runners disponibles).

**Jobs**

Un job define qué hacer, dónde ejecutarse y qué artefactos producir:

```yaml
lint:
  stage: build
  image: node:20-alpine
  script:
    - npm ci
    - npm run lint
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
```

**Artifacts**

Los artifacts son archivos que un job produce y que jobs posteriores pueden consumir:

```yaml
build:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

test:
  stage: test
  needs: [build]        # Depende del job build
  script:
    - ls dist/          # Los artifacts de build están disponibles
    - npm test
```

**Cache**

A diferencia de los artifacts (que son resultados del job), el cache almacena dependencias para acelerar builds futuros:

```yaml
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/
```

La `key` define cómo se identifican las entradas de caché. Usar `CI_COMMIT_REF_SLUG` (nombre de rama) significa que cada rama tiene su propia caché. Para compartir caché entre todas las ramas:

```yaml
cache:
  key:
    files:
      - package-lock.json
  paths:
    - node_modules/
```

Este enfoque usa el hash del `package-lock.json` como clave, por lo que la caché se reutiliza mientras las dependencias no cambien.

### 9.3.2 Docker-in-Docker (DinD)

GitLab CI ejecuta jobs dentro de contenedores Docker. Para construir imágenes Docker dentro de un job, tienes tres opciones principales:

**Opción 1: Docker-in-Docker (DinD)**

El job ejecuta un contenedor Docker que a su vez ejecuta un daemon Docker interno:

```yaml
variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_TLS_CERTDIR: ""

docker-build:
  stage: build
  image: docker:26-dind-rootless
  services:
    - docker:26-dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

**Cómo funciona:**

1. GitLab Runner inicia el job en un contenedor `docker:26-dind-rootless`.
2. El service `docker:26-dind` inicia un daemon Docker en un contenedor separado.
3. La variable `DOCKER_HOST: tcp://docker:2375` conecta el cliente Docker del job al daemon del service.
4. El job puede ejecutar `docker build`, `docker push`, etc., contra ese daemon.

**Ventajas:**
- Aislamiento: cada job tiene su propio daemon Docker.
- Compatibilidad: funciona con cualquier imagen Docker.

**Desventajas:**
- Sobrecarga: dos contenedores en lugar de uno.
- Almacenamiento: el daemon interno no comparte caché con otros jobs.
- Seguridad: ejecutar Docker dentro de Docker requiere `--privileged` en el runner si no usas `rootless`.

**Opción 2: Docker Socket Binding**

Montar el socket Docker del host en el contenedor del job:

```yaml
docker-build-socket:
  stage: build
  image: docker:26
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  tags:
    - docker        # Runner con acceso al socket
```

El runner debe estar configurado con:

```toml
# /etc/gitlab-runner/config.toml
[[runners]]
  executor = "docker"
  [runners.docker]
    image = "docker:26"
    privileged = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock"]
```

**Ventajas:**
- Sin overhead de DinD: usa el daemon del host directamente.
- Velocidad: la caché de imágenes del host se comparte entre jobs.

**Desventajas y riesgos de seguridad:**
- **Escalada de privilegios.** Al compartir `/var/run/docker.sock`, un contenedor malicioso puede lanzar otro contenedor con `--privileged` y escapar al host.
- **Contaminación.** Los jobs comparten el mismo daemon Docker. Un job puede ver y manipular contenedores e imágenes de otros jobs.
- **No aislado.** No hay separación real de entornos entre jobs.

> **Regla general:** Si estás en un entorno multi-tenant (runners compartidos), **nunca** uses socket binding. Usa DinD o Kaniko.

**Opción 3: Kaniko (recomendada)**

Kaniko construye imágenes Docker sin necesidad de un daemon Docker. Es la opción más segura para entornos compartidos.

### 9.3.3 Kaniko: builds sin privilegios

Kaniko es una herramienta de Google que construye imágenes de contenedor en espacio de usuario, sin acceso a un daemon Docker. Esto significa que puede ejecutarse en entornos restringidos como Kubernetes o GitLab CI sin `privileged: true`:

```yaml
kaniko-build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.23.1-debug
    entrypoint: [""]
  script:
    - |
      /kaniko/executor \
        --context $CI_PROJECT_DIR \
        --dockerfile $CI_PROJECT_DIR/Dockerfile \
        --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA \
        --destination $CI_REGISTRY_IMAGE:latest \
        --cache=true \
        --cache-ttl=24h \
        --snapshot-mode=redo
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

**Cómo funciona Kaniko:**

1. Lee el `Dockerfile` línea por línea.
2. Para cada instrucción, ejecuta la acción correspondiente en el sistema de archivos del contenedor de Kaniko (sin daemon).
3. Toma snapshots del sistema de archivos después de cada instrucción.
4. Construye las capas de la imagen a partir de los snapshots.
5. Empuja la imagen directamente al registro.

**Ventajas frente a DinD y socket binding:**

| Característica | DinD | Socket Binding | Kaniko |
|----------------|------|----------------|--------|
| Necesita daemon Docker | Sí | Sí | No |
| Necesita `privileged` | Sí | No (pero peligroso) | No |
| Aislamiento | Alto | Bajo | Medio |
| Velocidad | Media | Alta | Media |
| Cache | No persistente | Nativo | Remoto (registro) |
| Seguridad | Media | Baja | Alta |

**Configuración de caché en Kaniko**

Kaniko soporta caché remota, almacenando las capas en el registro de contenedores:

```bash
/kaniko/executor \
  --cache=true \
  --cache-repo=$CI_REGISTRY_IMAGE/cache \
  --cache-ttl=72h \
  --destination=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

También soporta caché basada en contenido, usando layers de imágenes ya existentes si el contenido es idéntico:

```bash
/kaniko/executor \
  --cache=true \
  --cache-copy-layers=true \
  --destination=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

### 9.3.4 GitLab Container Registry

GitLab incluye un registro de contenedores integrado para cada proyecto. Las imágenes se almacenan en `registry.gitlab.com/{namespace}/{project}` y las credenciales se configuran automáticamente mediante variables predefinidas:

| Variable | Descripción |
|----------|-------------|
| `CI_REGISTRY` | URL del registro (`registry.gitlab.com`) |
| `CI_REGISTRY_IMAGE` | Ruta completa de la imagen (`registry.gitlab.com/namespace/project`) |
| `CI_REGISTRY_USER` | Usuario para autenticación |
| `CI_REGISTRY_PASSWORD` | Token temporal para autenticación |

```yaml
docker-build:
  stage: build
  image: docker:26
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

### 9.3.5 Auto DevOps

Auto DevOps es la funcionalidad de GitLab que genera automáticamente un pipeline CI/CD completo basado en el lenguaje y framework detectados en tu repositorio. **No necesitas escribir `.gitlab-ci.yml`**.

**Activación**

Simplemente agrega un template:

```yaml
# .gitlab-ci.yml
include:
  - template: Auto-DevOps.gitlab-ci.yml

variables:
  AUTO_DEVOPS_PLATFORM_TARGET: ECS
  AUTO_DEVOPS_DEPLOY_DEBUG: "true"
```

**Pipeline generado automáticamente**

Auto DevOps genera los siguientes stages:

```
build → test → code_quality → license_management →
container_scanning → dast → review → staging →
canary → production → incremental_rollout → performance
```

**Build automático**

Auto DevOps usa **Herokuish** (para aplicaciones que usan buildpacks) o un `Dockerfile` existente. Si no hay `Dockerfile`, Herokuish detecta el lenguaje e instala dependencias:

```
Detección de lenguaje:
  - package.json → Node.js
  - requirements.txt → Python
  - pom.xml → Java
  - Gemfile → Ruby
  - go.mod → Go
```

**Personalización con variables**

Puedes personalizar el pipeline con variables:

```yaml
variables:
  # Desactivar stages que no necesites
  TEST_DISABLED: "true"
  CODE_QUALITY_DISABLED: "true"
  LICENSE_MANAGEMENT_DISABLED: "true"
  DAST_DISABLED: "true"
  PERFORMANCE_DISABLED: "true"

  # Configurar despliegue
  KUBE_NAMESPACE: myapp
  HELM_UPGRADE_VALUES_FILE: .gitlab/helm-values.yaml

  # Configurar build
  AUTO_BUILD_IMAGE_VERSION: "1.0.0"
  BUILD_IMAGE_EXTRA_ARGS: "--build-arg NODE_ENV=production"
```

**Limitaciones de Auto DevOps**

- Curva de personalización alta cuando necesitas algo muy específico.
- El pipeline es genérico; puede no optimizar para tu stack particular.
- No es ideal si ya tienes un pipeline optimizado con Kaniko, Buildx, etc.

### 9.3.6 Ejemplo completo: Python → Test → Build → Push GitLab Registry → Deploy

Pipeline completo para una aplicación Python (FastAPI) con tests, build de imagen vía Kaniko, push a GitLab Container Registry y deploy a Kubernetes:

```yaml
# .gitlab-ci.yml
# Pipeline CI/CD completo para app Python + Kubernetes

stages:
  - setup
  - lint
  - test
  - security
  - build
  - deploy-staging
  - deploy-production
  - release

# ─── Variables globales ────────────────────────────────────────
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  LATEST_TAG: $CI_REGISTRY_IMAGE:latest
  KUBE_NAMESPACE_STAGING: staging
  KUBE_NAMESPACE_PROD: production

# ─── Cache de dependencias Python ──────────────────────────────
.python-cache: &python-cache
  cache:
    key:
      files:
        - requirements.txt
        - requirements-dev.txt
    paths:
      - .cache/pip
      - .venv/

# ─── Template para jobs Python ─────────────────────────────────
.python-job: &python-job
  image: python:3.12-slim
  before_script:
    - pip install --upgrade pip
    - pip install -r requirements-dev.txt

# ─── Stage 1: Setup ────────────────────────────────────────────
setup:
  stage: setup
  <<: *python-job
  <<: *python-cache
  script:
    - pip install -r requirements.txt
    - python -c "import sys; print(f'Python {sys.version}')"
  artifacts:
    paths:
      - .venv/
    expire_in: 1 hour

# ─── Stage 2: Lint ─────────────────────────────────────────────
lint:
  stage: lint
  <<: *python-job
  <<: *python-cache
  needs: [setup]
  script:
    - black --check src/ tests/
    - isort --check-only src/ tests/
    - ruff check src/ tests/
    - mypy src/

# ─── Stage 3: Test ─────────────────────────────────────────────
unit-tests:
  stage: test
  <<: *python-job
  <<: *python-cache
  needs: [lint]
  services:
    - postgres:16-alpine
    - redis:7-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://test:test@postgres:5432/testdb
    REDIS_URL: redis://redis:6379/0
  script:
    - pytest tests/unit/ -v --cov=src --cov-report=term-missing --cov-report=xml
  coverage: '/TOTAL.+ ([0-9]{1,3}%)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - coverage.xml
    expire_in: 7 days

integration-tests:
  stage: test
  <<: *python-job
  <<: *python-cache
  needs: [lint]
  services:
    - postgres:16-alpine
    - redis:7-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://test:test@postgres:5432/testdb
    REDIS_URL: redis://redis:6379/0
  script:
    - pytest tests/integration/ -v --cov=src --cov-append

# ─── Stage 4: Security ─────────────────────────────────────────
secret-detection:
  stage: security
  image: python:3.12-slim
  script:
    - pip install detect-secrets
    - detect-secrets scan --all-files > secrets-report.json
  artifacts:
    paths:
      - secrets-report.json
    expire_in: 30 days
  allow_failure: true

dependency-scan:
  stage: security
  image: python:3.12-slim
  <<: *python-cache
  script:
    - pip install safety
    - safety check --full-report
  allow_failure: true

# ─── Stage 5: Build con Kaniko ─────────────────────────────────
build-image:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.23.1-debug
    entrypoint: [""]
  needs: [unit-tests, integration-tests, dependency-scan]
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "develop"
    - if: $CI_COMMIT_TAG
  script:
    - |
      /kaniko/executor \
        --context $CI_PROJECT_DIR \
        --dockerfile $CI_PROJECT_DIR/Dockerfile \
        --destination $IMAGE_TAG \
        --destination $LATEST_TAG \
        --cache=true \
        --cache-repo=$CI_REGISTRY_IMAGE/cache \
        --cache-ttl=24h \
        --snapshot-mode=redo \
        --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
        --build-arg VCS_REF=$CI_COMMIT_SHA \
        --label "org.opencontainers.image.created=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
        --label "org.opencontainers.image.revision=$CI_COMMIT_SHA"

container-scanning:
  stage: build
  needs: [build-image]
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  rules:
    - if: $CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"
  script:
    - trivy image --severity CRITICAL,HIGH --no-progress --exit-code 1 $IMAGE_TAG
    - trivy image --severity CRITICAL,HIGH --no-progress --exit-code 1 $LATEST_TAG
  allow_failure: true

# ─── Stage 6: Deploy a Staging ─────────────────────────────────
deploy-staging:
  stage: deploy-staging
  image:
    name: bitnami/kubectl:latest
    entrypoint: [""]
  needs: [build-image]
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
  environment:
    name: staging
    url: https://staging.myapp.example.com
  variables:
    KUBE_NAMESPACE: $KUBE_NAMESPACE_STAGING
  before_script:
    - kubectl config set-cluster staging-cluster --server=$KUBE_STAGING_SERVER --insecure-skip-tls-verify=true
    - kubectl config set-credentials staging-user --token=$KUBE_STAGING_TOKEN
    - kubectl config set-context staging --cluster=staging-cluster --user=staging-user --namespace=$KUBE_NAMESPACE
    - kubectl config use-context staging
  script:
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG --namespace=$KUBE_NAMESPACE
    - kubectl rollout status deployment/myapp --namespace=$KUBE_NAMESPACE --timeout=5m
    - kubectl get pods --namespace=$KUBE_NAMESPACE

  after_script:
    - |
      if [ $CI_JOB_STATUS == "success" ]; then
        curl -X POST -H 'Content-type: application/json' \
          --data "{\"text\":\"Staging desplegado exitosamente: $IMAGE_TAG\"}" \
          $SLACK_WEBHOOK
      else
        curl -X POST -H 'Content-type: application/json' \
          --data "{\"text\":\"Fallo despliegue staging: $IMAGE_TAG\"}" \
          $SLACK_WEBHOOK
      fi

# ─── Stage 7: Deploy a Producción ──────────────────────────────
deploy-production:
  stage: deploy-production
  image:
    name: bitnami/kubectl:latest
    entrypoint: [""]
  needs: [build-image, container-scanning]
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  environment:
    name: production
    url: https://myapp.example.com
  variables:
    KUBE_NAMESPACE: $KUBE_NAMESPACE_PROD
  when: manual    # Despliegue manual — requiere botón en UI
  before_script:
    - kubectl config set-cluster prod-cluster --server=$KUBE_PROD_SERVER --insecure-skip-tls-verify=true
    - kubectl config set-credentials prod-user --token=$KUBE_PROD_TOKEN
    - kubectl config set-context production --cluster=prod-cluster --user=prod-user --namespace=$KUBE_NAMESPACE
    - kubectl config use-context production
  script:
    - |
      kubectl set image deployment/myapp myapp=$IMAGE_TAG \
        --namespace=$KUBE_NAMESPACE --record
    - |
      if kubectl rollout status deployment/myapp \
        --namespace=$KUBE_NAMESPACE --timeout=10m; then
        echo "Despliegue exitoso en produccion"
      else
        echo "Rollback automatico..."
        kubectl rollout undo deployment/myapp --namespace=$KUBE_NAMESPACE
        exit 1
      fi
    - kubectl get pods --namespace=$KUBE_NAMESPACE

  after_script:
    - |
      if [ $CI_JOB_STATUS == "success" ]; then
        curl -X POST -H 'Content-type: application/json' \
          --data "{\"text\":\"PRODUCCION desplegada: $IMAGE_TAG\nURL: https://myapp.example.com\"}" \
          $SLACK_WEBHOOK
      else
        curl -X POST -H 'Content-type: application/json' \
          --data "{\"text\":\"DESPLIEGUE FALLIDO en PRODUCCION. Rollback ejecutado. Imagen: $IMAGE_TAG\"}" \
          $SLACK_WEBHOOK
      fi

# ─── Stage 8: Release ──────────────────────────────────────────
release:
  stage: release
  image: python:3.12-slim
  needs: [deploy-production]
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  before_script:
    - pip install python-semantic-release
  script:
    - semantic-release publish
  environment:
    name: production
```

---

## 9.4 Jenkins

Jenkins es el abuelo venerable del ecosistema CI/CD. Con más de 15 años de historia y un ecosistema de más de 1800 plugins, sigue siendo una opción sólida — especialmente en organizaciones con infraestructura on-premise, requisitos de compliance estrictos o pipelines altamente personalizados.

### 9.4.1 Declarative Pipeline (Jenkinsfile)

Jenkins moderno usa **Declarative Pipeline**, una sintaxis estructurada y legible definida en un `Jenkinsfile` versionado junto al código:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing...'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying...'
            }
        }
    }
}
```

Un `Jenkinsfile` es un script Groovy con una estructura bien definida:

```
pipeline {
    agent {}          // Dónde ejecutar
    environment {}    // Variables de entorno
    options {}        // Opciones del pipeline
    parameters {}     // Parámetros de entrada
    triggers {}       // Disparadores automáticos
    stages {          // Etapas del pipeline
        stage('Name') {
            agent {}      // (opcional) Override de agent
            when {}       // Condicionales
            steps {}      // Acciones a ejecutar
            post {}       // Acciones post-ejecución
        }
    }
    post {}           // Acciones post-pipeline
}
```

### 9.4.2 Agentes Docker

Jenkins puede ejecutar stages completos dentro de contenedores Docker. Esto elimina la necesidad de instalar herramientas en el host de Jenkins:

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /tmp/.npm:/root/.npm:rw'
        }
    }

    stages {
        stage('Install & Build') {
            steps {
                sh 'npm ci'
                sh 'npm run build'
            }
        }
    }
}
```

**Agentes específicos por stage**

Diferentes stages pueden usar diferentes imágenes:

```groovy
pipeline {
    agent none    // Sin agente global — cada stage define el suyo

    stages {
        stage('Lint & Test') {
            agent {
                docker {
                    image 'node:20-alpine'
                    reuseNode true
                }
            }
            steps {
                sh 'npm ci && npm run lint && npm test'
            }
        }

        stage('Build Docker Image') {
            agent {
                docker {
                    image 'docker:26-dind'
                    args '--privileged'
                }
            }
            steps {
                sh '''
                    docker build -t myapp:${BUILD_NUMBER} .
                    docker tag myapp:${BUILD_NUMBER} myreg/myapp:${BUILD_NUMBER}
                '''
            }
        }

        stage('Security Scan') {
            agent {
                docker {
                    image 'aquasec/trivy:latest'
                    args '--entrypoint=""'
                }
            }
            steps {
                sh 'trivy image --severity CRITICAL,HIGH myreg/myapp:${BUILD_NUMBER}'
            }
        }
    }
}
```

### 9.4.3 Credentials

Jenkins maneja credenciales mediante el **Credentials Plugin**. Los secretos se configuran en la UI de Jenkins y se referencian por ID en el pipeline:

```groovy
pipeline {
    environment {
        DOCKER_REGISTRY = credentials('docker-registry-credentials')
        KUBE_CONFIG      = credentials('kubeconfig-secret')
        AWS_CREDENTIALS  = credentials('aws-credentials')
    }

    stages {
        stage('Docker Login') {
            steps {
                sh '''
                    echo "${DOCKER_REGISTRY_PSW}" | \
                    docker login registry.example.com \
                      -u ${DOCKER_REGISTRY_USR} \
                      --password-stdin
                '''
                // Jenkins automáticamente crea variables:
                //   DOCKER_REGISTRY_USR (username)
                //   DOCKER_REGISTRY_PSW (password)
            }
        }

        stage('Deploy to K8s') {
            steps {
                sh '''
                    mkdir -p ~/.kube
                    echo "${KUBE_CONFIG}" > ~/.kube/config
                    kubectl set image deployment/myapp \
                      myapp=registry.example.com/myapp:${BUILD_NUMBER}
                '''
            }
        }
    }
}
```

**Tipos de credenciales soportados:**

| Tipo | Uso típico |
|------|------------|
| Username/Password | Docker Hub, Git, APIs |
| SSH Key | Git SSH, acceso a servidores |
| Secret File | kubeconfig, certificados |
| Secret Text | Tokens, API keys |
| Certificate | Certificados TLS |

### 9.4.4 Jenkinsfile completo: Build + Test + Push + Deploy

El siguiente `Jenkinsfile` representa un pipeline de producción completo para una aplicación Dockerizada:

```groovy
// ─── Jenkinsfile ──────────────────────────────────────────────
pipeline {
    // ── Agente global ─────────────────────────────────────────
    agent none

    // ── Opciones del pipeline ─────────────────────────────────
    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        timeout(time: 60, unit: 'MINUTES')
        ansiColor('xterm')
        timestamps()
    }

    // ── Parámetros de entrada ──────────────────────────────────
    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['staging', 'production'],
            description: 'Entorno de despliegue'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Omitir tests (solo emergencias)'
        )
        string(
            name: 'IMAGE_TAG',
            defaultValue: '',
            description: 'Override de tag de imagen (vacío = BUILD_NUMBER)'
        )
    }

    // ── Variables de entorno ──────────────────────────────────
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE_NAME = "${REGISTRY}/myorg/myapp"
        DOCKERHUB_CREDS = credentials('dockerhub-credentials')
        KUBE_CONFIG = credentials('kubeconfig')
        SONAR_TOKEN = credentials('sonar-token')
        NPM_TOKEN = credentials('npm-token')
    }

    // ── Stages ─────────────────────────────────────────────────
    stages {

        // ─ Stage 1: Checkout y Preparación ────────────────────
        stage('Checkout') {
            agent { label 'docker' }
            steps {
                checkout scm

                script {
                    if (params.IMAGE_TAG) {
                        env.DEPLOY_TAG = params.IMAGE_TAG
                    } else {
                        env.DEPLOY_TAG = "${BUILD_NUMBER}"
                    }

                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    env.GIT_BRANCH = sh(
                        script: 'git rev-parse --abbrev-ref HEAD',
                        returnStdout: true
                    ).trim()
                }

                echo """
                ================================================
                PIPELINE INICIADO
                Build:       ${BUILD_NUMBER}
                Tag:         ${DEPLOY_TAG}
                Branch:      ${GIT_BRANCH}
                Commit:      ${GIT_COMMIT_SHORT}
                Entorno:     ${DEPLOY_ENV}
                ================================================
                """.stripIndent()
            }
        }

        // ─ Stage 2: Lint y Análisis de Código ────────────────
        stage('Lint & Static Analysis') {
            agent {
                docker {
                    image 'node:20-alpine'
                    reuseNode true
                    args '-v /tmp/.npm:/root/.npm:rw'
                }
            }
            steps {
                sh 'npm ci'
                sh 'npm run lint'
                sh 'npm run typecheck'
                sh 'npx prettier --check "src/**/*.ts"'
            }
        }

        // ─ Stage 3: Tests Unitarios ───────────────────────────
        stage('Unit Tests') {
            when {
                expression { !params.SKIP_TESTS }
            }
            agent {
                docker {
                    image 'node:20-alpine'
                    reuseNode true
                    args '-v /tmp/.npm:/root/.npm:rw'
                }
            }
            steps {
                sh 'npm test -- --coverage'

                junit 'reports/junit/*.xml'

                publishHTML(target: [
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'coverage/lcov-report',
                    reportFiles: 'index.html',
                    reportName: 'Coverage Report'
                ])
            }

            post {
                success {
                    echo 'Tests unitarios pasaron'
                }
                failure {
                    error('Tests unitarios fallaron')
                }
            }
        }

        // ─ Stage 4: Tests de Integración ──────────────────────
        stage('Integration Tests') {
            when {
                expression { !params.SKIP_TESTS }
            }
            agent {
                docker {
                    image 'node:20-alpine'
                    reuseNode true
                    args '-v /tmp/.npm:/root/.npm:rw --network host'
                }
            }
            steps {
                script {
                    docker.image('postgres:16-alpine')
                        .withRun('-e POSTGRES_PASSWORD=test -e POSTGRES_DB=testdb -p 5432:5432') { pg ->
                            docker.image('redis:7-alpine')
                                .withRun('-p 6379:6379') { redis ->
                                    sleep 10
                                    sh '''
                                        DATABASE_URL=postgres://postgres:test@localhost:5432/testdb
                                        REDIS_URL=redis://localhost:6379
                                        npm run test:integration
                                    '''
                                }
                        }
                }
            }
        }

        // ─ Stage 5: Build de Imagen Docker ───────────────────
        stage('Docker Build') {
            agent { label 'docker' }
            steps {
                sh """
                    docker build \
                      --build-arg NODE_ENV=production \
                      --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                      --build-arg VCS_REF=${GIT_COMMIT_SHORT} \
                      -t ${IMAGE_NAME}:${DEPLOY_TAG} \
                      -t ${IMAGE_NAME}:latest \
                      .
                """
            }
        }

        // ─ Stage 6: Escaneo de Vulnerabilidades ──────────────
        stage('Security Scan') {
            agent {
                docker {
                    image 'aquasec/trivy:latest'
                    args '--entrypoint=""'
                }
            }
            steps {
                sh """
                    trivy image \
                      --severity CRITICAL,HIGH \
                      --exit-code 1 \
                      --no-progress \
                      ${IMAGE_NAME}:${DEPLOY_TAG}
                """
            }
        }

        // ─ Stage 7: Push a Registry ──────────────────────────
        stage('Docker Push') {
            agent { label 'docker' }
            steps {
                sh """
                    echo "\${DOCKERHUB_CREDS_PSW}" | \
                    docker login ${REGISTRY} \
                      -u \${DOCKERHUB_CREDS_USR} \
                      --password-stdin
                """
                sh "docker push ${IMAGE_NAME}:${DEPLOY_TAG}"
                sh "docker push ${IMAGE_NAME}:latest"
            }

            post {
                success {
                    echo "Imagen publicada: ${IMAGE_NAME}:${DEPLOY_TAG}"
                }
            }
        }

        // ─ Stage 8: Despliegue ────────────────────────────────
        stage('Deploy') {
            agent { label 'docker' }
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    if (params.DEPLOY_ENV == 'production') {
                        input(
                            message: "Confirmar despliegue a PRODUCCION?",
                            ok: 'Desplegar',
                            submitter: 'admin,devops-lead'
                        )
                    }
                }

                sh """
                    mkdir -p ~/.kube
                    echo "\${KUBE_CONFIG}" > ~/.kube/config
                    chmod 600 ~/.kube/config
                """

                script {
                    def namespace = params.DEPLOY_ENV == 'production'
                        ? 'production'
                        : 'staging'

                    sh """
                        kubectl set image deployment/myapp \
                          myapp=${IMAGE_NAME}:${DEPLOY_TAG} \
                          --namespace=${namespace} --record
                    """

                    def rolloutStatus = sh(
                        script: """
                            kubectl rollout status deployment/myapp \
                              --namespace=${namespace} --timeout=10m
                        """,
                        returnStatus: true
                    )

                    if (rolloutStatus != 0) {
                        error("Rollout fallido en ${namespace}")
                    }
                }
            }

            post {
                failure {
                    script {
                        def namespace = params.DEPLOY_ENV == 'production'
                            ? 'production'
                            : 'staging'

                        sh """
                            kubectl rollout undo deployment/myapp \
                              --namespace=${namespace}
                            echo "## ROLLBACK ejecutado en ${namespace}" \
                              >> ${BUILD_NUMBER}-rollback.log
                        """

                        // Notificar rollback
                        slackSend(
                            color: 'danger',
                            message: "DESPLIEGUE FALLIDO en ${params.DEPLOY_ENV}\n" +
                                     "Build: ${BUILD_NUMBER}\nRollback ejecutado",
                            channel: '#deploy-alerts'
                        )
                    }
                }
                success {
                    slackSend(
                        color: 'good',
                        message: "Despliegue exitoso en ${params.DEPLOY_ENV}\n" +
                                 "Build: ${BUILD_NUMBER}\n" +
                                 "Imagen: ${IMAGE_NAME}:${DEPLOY_TAG}",
                        channel: '#deploy-alerts'
                    )
                }
            }
        }
    }

    // ── Post-acciones del pipeline ────────────────────────────
    post {
        always {
            cleanWs(
                deleteDirs: true,
                patterns: [[pattern: 'node_modules/', type: 'INCLUDE']]
            )

            echo "Pipeline ${BUILD_NUMBER} finalizado con estado: ${currentBuild.result}"
        }

        success {
            build(
                job: 'post-deploy-smoke-tests',
                parameters: [
                    string(name: 'IMAGE_TAG', value: "${DEPLOY_TAG}"),
                    string(name: 'ENV', value: "${DEPLOY_ENV}")
                ],
                wait: false
            )
        }

        aborted {
            echo 'Pipeline abortado manualmente'
        }
    }
}
```

---

## 9.5 Estrategias de despliegue automatizado

El despliegue no es simplemente copiar una nueva imagen a producción. La forma en que introduces la nueva versión determina el riesgo, el downtime y la capacidad de recuperación. Veamos las estrategias principales.

### 9.5.1 Rolling Update

**Rolling Update** reemplaza instancias gradualmente, manteniendo el servicio disponible durante todo el proceso.

```
Antes del despliegue:
  [Pod v1] [Pod v1] [Pod v1]  → todas las réplicas en v1

Durante el despliegue (3 réplicas, maxSurge=1, maxUnavailable=0):
  [Pod v1] [Pod v1] [Pod v1] [Pod v2]  → se crea v2 extra
  [Pod v1] [Pod v1] [Pod v2]  → se elimina un v1
  [Pod v1] [Pod v2] [Pod v2]  → se crea otro v2
  [Pod v2] [Pod v2] [Pod v2]  → despliegue completo
```

**Kubernetes Rolling Update**

Es el comportamiento por defecto de un Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # Hasta 2 pods extra durante el despliegue
      maxUnavailable: 0    # Ningún pod no disponible (zero-downtime)
  template:
    spec:
      containers:
        - name: myapp
          image: registry.example.com/myapp:v2.0.0
          readinessProbe:  # CRÍTICO: Kubernetes necesita saber cuándo el nuevo pod está listo
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

**Parámetros clave:**

| Parámetro | Valor | Significado |
|-----------|-------|-------------|
| `maxSurge: 1` | "cuántos pods extra puedo crear" | Rolling update más rápido |
| `maxSurge: 25%` | Valor por defecto | Balance entre velocidad y recursos |
| `maxUnavailable: 0` | "cuántos pods pueden estar caídos" | Zero-downtime (requiere readinessProbes) |
| `maxUnavailable: 1` | Default | Se tolera 1 pod caído |

**El rol crítico de las probes en Rolling Update**

Sin `readinessProbe`, Kubernetes asume que el contenedor está listo apenas arranca. Si tu aplicación tarda 10 segundos en inicializar, Kubernetes podría enviar tráfico a un pod no preparado, causando errores 502/503.

```yaml
readinessProbe:
  httpGet:
    path: /ready            # Endpoint que solo retorna 200 cuando la app está lista
    port: 8080
  initialDelaySeconds: 10   # Esperar 10s antes del primer chequeo
  periodSeconds: 5          # Chequear cada 5s
  failureThreshold: 3       # Marcar como "not ready" después de 3 fallos
  successThreshold: 1       # Marcar como "ready" después de 1 éxito
```

**Rolling Update en Docker Swarm**

```yaml
services:
  myapp:
    image: registry.example.com/myapp:v2.0.0
    deploy:
      replicas: 5
      update_config:
        parallelism: 2     # Actualizar 2 réplicas a la vez
        delay: 10s         # Esperar 10s entre lotes
        failure_action: rollback
        monitor: 30s       # Monitorear 30s antes de marcar como exitoso
        max_failure_ratio: 0.3  # Abortar si más del 30% falla
        order: start-first # Iniciar nuevos antes de detener viejos
```

**Rolling Update con AWS ECS**

```json
{
  "deploymentConfiguration": {
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    },
    "maximumPercent": 200,
    "minimumHealthyPercent": 100
  }
}
```

ECS automáticamente registra nuevas tasks en el Target Group del ALB, hace draining de las tasks viejas y las detiene. Con `deploymentCircuitBreaker`, si el despliegue falla consistentemente, ECS hace rollback automático.

### 9.5.2 Blue/Green Deployment

**Blue/Green** mantiene dos entornos idénticos: **Blue** (actual, en producción) y **Green** (nueva versión). El tráfico se redirige instantáneamente de Blue a Green una vez que Green está validado.

```
ANTES:
  Trafico → [Blue: v1.0]        [Green: (vacío)]

DESPLIEGUE:
  Trafico → [Blue: v1.0]        [Green: v2.0]  ← se despliega v2 sin tráfico

CUTOVER:
  Trafico → [Blue: (idle)]      [Green: v2.0]  ← tráfico redirigido

ROLLBACK:
  Trafico → [Blue: v1.0]        [Green: v2.0]  ← redirigir de vuelta (instantáneo)
```

**Ventajas:**
- Rollback instantáneo (menos de 1 segundo).
- Validación completa de Green antes del cutover.
- Sin problemas de compatibilidad entre versiones (no coexisten sirviendo tráfico).

**Desventajas:**
- Requiere el doble de recursos durante el despliegue.
- Más complejidad operativa.

**Blue/Green con Kubernetes Services**

La técnica más simple usa selectores de Service:

```yaml
# ── Blue Deployment ──────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
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
        - name: myapp
          image: registry.example.com/myapp:v1.0.0
---
# ── Green Deployment (se crea vía CI/CD) ─────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
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
        - name: myapp
          image: registry.example.com/myapp:v2.0.0
---
# ── Service — el selector apunta a BLUE por ahora ────────────
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue      # ← Cambiar a "green" para redirigir tráfico
  ports:
    - port: 80
      targetPort: 8080
```

**Procedimiento de cutover:**

```bash
# 1. Desplegar Green
kubectl apply -f myapp-green.yaml

# 2. Validar Green (smoke tests, health checks)
kubectl port-forward deployment/myapp-green 8081:8080
curl -f http://localhost:8081/health

# 3. Cutover: cambiar el selector del Service
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# 4. Monitorear
kubectl get pods -l app=myapp

# 5. Si rollback necesario:
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'

# 6. Limpiar (días después, cuando ya no se necesite rollback)
kubectl delete deployment myapp-blue
```

**Blue/Green con AWS CodeDeploy**

AWS CodeDeploy nativamente soporta Blue/Green para ECS y EC2:

```yaml
# appspec.yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION_ARN>
        LoadBalancerInfo:
          ContainerName: myapp
          ContainerPort: 8080
Hooks:
  - BeforeAllowTraffic: "BeforeAllowTrafficHook"
  - AfterAllowTraffic: "AfterAllowTrafficHook"
```

CodeDeploy:
1. Crea el nuevo Task Set (Green) en el servicio ECS.
2. Ejecuta `BeforeAllowTraffic` hook.
3. Redirige tráfico del ALB del Task Set Blue al Green.
4. Ejecuta `AfterAllowTraffic` hook.
5. Si hay problemas, redirige automáticamente de vuelta.

### 9.5.3 Canary Release

**Canary Release** envía un porcentaje pequeño del tráfico a la nueva versión, monitorea métricas (errores, latencia, tasa de éxito) y aumenta gradualmente:

```
T=0min:   v1: 100%  |  v2: 0%
T=5min:   v1: 95%   |  v2: 5%     ← liberar al 5% del tráfico
T=15min:  v1: 75%   |  v2: 25%    ← si métricas OK, subir a 25%
T=30min:  v1: 50%   |  v2: 50%    ← OK, subir a 50%
T=60min:  v1: 0%    |  v2: 100%   ← despliegue completo
```

**Diferencia con Blue/Green:** Canary mezcla tráfico entre versiones simultáneamente y avanza gradualmente. Blue/Green redirige todo el tráfico de golpe después de validar Green.

**Canary con Istio**

Istio ofrece control de tráfico fino a nivel de Service Mesh:

```yaml
# ── VirtualService con división de tráfico ────────────────────
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp.example.com
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            host: myapp
            subset: v1
          weight: 90        # 90% a v1
        - destination:
            host: myapp
            subset: v2
          weight: 10        # 10% a v2 (canary)
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
    - name: v1
      labels:
        version: "1.0"
    - name: v2
      labels:
        version: "2.0"
```

**Flagger (Canary automatizado)**

Flagger es un operador de Kubernetes que automatiza canary releases con Prometheus/Grafana:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 8080
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10           # Aumentar 10% en cada paso
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99            # Debe mantener >99% de éxito
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500           # P99 debe ser <500ms
    webhooks:
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 http://myapp-canary.production:8080/"
```

Flagger:

1. Detecta cambios en la imagen del Deployment.
2. Crea un nuevo deployment canary (`myapp-primary` y `myapp`).
3. Libera tráfico gradualmente (10%, 20%, 30%...).
4. En cada paso, analiza métricas de Prometheus.
5. Si las métricas degradan, hace rollback automático.
6. Si todo bien, promueve la nueva versión a primary.

**Canary con NGINX Ingress Controller**

Si no usas Istio, NGINX Ingress soporta canary mediante annotations:

```yaml
# ── Ingress principal (v1, estable) ───────────────────────────
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-stable
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-stable
                port:
                  number: 80
---
# ── Ingress canary (v2, experimental) ─────────────────────────
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"     # 10% tráfico a canary
    # nginx.ingress.kubernetes.io/canary-by-header: "x-canary"  # Por header
    # nginx.ingress.kubernetes.io/canary-by-cookie: "canary"   # Por cookie
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-canary
                port:
                  number: 80
```

Para cambiar el peso del canary en CI/CD, simplemente actualiza la annotation:

```bash
kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight="25" --overwrite
```

### 9.5.4 A/B Testing

**A/B Testing** se confunde frecuentemente con canary, pero tienen propósitos distintos:

| | Canary Release | A/B Testing |
|---|---|---|
| **Objetivo** | Validar estabilidad operativa | Validar hipótesis de negocio |
| **Métrica** | Errores, latencia, uso de CPU | Conversiones, clicks, revenue |
| **Routing** | Por peso (porcentaje) | Por header/cookie/usuario |
| **Duración** | Minutos a horas | Días a semanas |
| **Riesgo** | Disponibilidad del servicio | Métricas de negocio |

**A/B Testing con Istio**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp.example.com
  http:
    # Usuarios en bucket "experiment-A" → versión B
    - match:
        - headers:
            x-ab-group:
              exact: "experiment-A"
      route:
        - destination:
            host: myapp
            subset: v2      # Versión B (nueva feature)

    # Usuarios en bucket "control" → versión A
    - match:
        - headers:
            x-ab-group:
              exact: "control"
      route:
        - destination:
            host: myapp
            subset: v1      # Versión A (actual)

    # Resto de usuarios → versión A
    - route:
        - destination:
            host: myapp
            subset: v1
```

El header `x-ab-group` es seteado por un proxy reverso, una API gateway, o directamente por el frontend basado en el ID de usuario (hash consistente):

```python
# Lógica de asignación de buckets
import hashlib

def get_ab_group(user_id: str) -> str:
    h = hashlib.md5(user_id.encode()).hexdigest()
    return "experiment-A" if int(h, 16) % 100 < 20 else "control"
    # 20% de usuarios en el experimento
```

### 9.5.5 GitOps: ArgoCD y Flux

**GitOps** es un paradigma donde el estado deseado de la infraestructura se declara en Git, y un operador en el clúster de Kubernetes lo sincroniza automáticamente.

**Principio fundamental: Pull, no Push**

```
Push-based (CI/CD tradicional):
  CI → kubectl apply -f deployment.yaml  ← CI empuja cambios al clúster

Pull-based (GitOps):
  CI → git push (cambiar YAML en Git)    ← CI solo toca Git
  Operador → observa Git → aplica cambios al clúster ← el clúster jala cambios
```

**Ventajas de GitOps:**

1. **Auditabilidad.** Cada cambio en producción tiene un commit asociado, con autor, timestamp y mensaje. Rollback es simplemente `git revert`.
2. **Seguridad.** El clúster no expone credenciales. Solo el operador (dentro del clúster) necesita acceso a Git y al clúster, no los pipelines de CI.
3. **Convergencia automática.** Si alguien modifica manualmente un recurso en el clúster (kubectl edit), el operador lo revierte al estado declarado en Git.
4. **Disaster Recovery.** Recrear un clúster es simplemente apuntar ArgoCD al mismo repositorio Git.

**ArgoCD**

ArgoCD es el operador GitOps más popular para Kubernetes:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-deploy     # Repo de config
    targetRevision: main
    path: overlays/production                          # Kustomize overlay
  destination:
    server: https://kubernetes.default.svc             # Mismo clúster
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Eliminar recursos que se quitan de Git
      selfHeal: true     # Revertir cambios manuales en el clúster
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Flujo GitOps con ArgoCD:**

```
1. Desarrollador hace merge a main.
2. CI pipeline:
   a. Test + Build + Push imagen a registry.
   b. Actualizar archivo YAML en repo de deploy (kustomize edit set image).
   c. git commit + git push al repo de deploy.
3. ArgoCD (dentro del clúster):
   a. Detecta cambio en Git (polling cada 3 min o webhook).
   b. Compara estado deseado (Git) vs estado actual (clúster) = diff.
   c. Aplica los cambios necesarios (kubectl apply).
   d. Monitorea health de los nuevos recursos.
   e. Si sync exitoso, marca Application como "Synced + Healthy".
```

**Actualización automática de la imagen en GitOps**

En el pipeline de CI, después de construir y pushear la imagen, actualizas el repositorio de deploy:

```yaml
# En GitHub Actions, después de build-and-push
update-gitops:
  needs: build-and-push
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with:
        repository: myorg/myapp-deploy
        token: ${{ secrets.GITOPS_PAT }}

    - name: Setup Kustomize
      uses: imranismail/setup-kustomize@v2

    - name: Actualizar imagen
      run: |
        cd overlays/production
        kustomize edit set image \
          myapp=ghcr.io/myorg/myapp:${{ github.sha }}

    - name: Commit y Push
      run: |
        git config user.name "CI Bot"
        git config user.email "ci@example.com"
        git add .
        git commit -m "chore(deploy): bump myapp to ${{ github.sha }}"
        git push origin main
```

**Flux CD**

Flux es la alternativa CNCF a ArgoCD con un enfoque ligeramente diferente:

```yaml
# Flux: reconciler que observa un repositorio Git
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myapp-deploy
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  path: ./overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: production
```

**Flux Image Automation**

Flux puede automáticamente actualizar imágenes cuando detecta nuevas versiones en el registro:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: ghcr.io/myorg/myapp
  interval: 1m
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: ">=1.0.0"
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: myapp
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@example.com
        name: Flux CD
      messageTemplate: "chore: update myapp to {{ .Image }}"
    push:
      branch: main
  update:
    path: ./overlays/production
    strategy: Setters
```

Con esta configuración, Flux:
1. Monitorea el registry cada 1 minuto.
2. Si encuentra una nueva imagen que satisface la política semver (>=1.0.0).
3. Automáticamente actualiza el archivo YAML en Git y hace push.
4. Luego, el Kustomization reconciler aplica los cambios al clúster.

Esto convierte el proceso en **completamente automatizado**: el desarrollador solo hace merge a main, y Flux se encarga de todo lo demás.

---

## 9.6 Pipeline de seguridad en CI/CD

La seguridad no puede ser una ocurrencia tardía. Debe estar integrada en cada etapa del pipeline — un concepto conocido como **DevSecOps** o "shift-left security".

### 9.6.1 Escaneo de vulnerabilidades con Trivy

Trivy es un escáner de vulnerabilidades open-source de Aqua Security. Es rápido, fácil de integrar y soporta múltiples artefactos (imágenes Docker, sistemas de archivos, repositorios Git, configuraciones de Kubernetes):

**Integración en GitHub Actions**

```yaml
- name: Trivy - Escanear imagen Docker
  uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: ghcr.io/myorg/myapp:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: 1                    # Fallar el job si hay CRITICAL o HIGH
    ignore-unfixed: true            # Solo vulnerabilidades con fix disponible

- name: Subir resultados a GitHub Security
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

**En GitLab CI**

```yaml
trivy-scan:
  stage: security
  image: aquasec/trivy:latest
  variables:
    TRIVY_SEVERITY: CRITICAL,HIGH
    TRIVY_EXIT_CODE: 1
    TRIVY_IGNORE_UNFIXED: "true"
  script:
    - trivy image --no-progress $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: false
```

**Política de bloqueo de despliegue**

Define reglas claras sobre qué vulnerabilidades bloquean el despliegue:

```yaml
# .trivy.yaml (configuración de Trivy)
severity:
  - CRITICAL
  - HIGH

vulnerability:
  type:
    - os
    - library

ignore:
  # Ignorar vulnerabilidades con ID específico (temporal, mientras se prepara fix)
  - CVE-2024-XXXX
```

```yaml
# GitHub Actions: Política de bloqueo
- name: Trivy Scan (CRITICAL)
  id: trivy
  uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    severity: 'CRITICAL'
    exit-code: 1        # CRITICAL = bloqueo total

- name: Trivy Scan (HIGH - warning)
  if: steps.trivy.outcome == 'success'
  uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    severity: 'HIGH'
    exit-code: 0        # HIGH = warning pero no bloquea
```

**Por qué dos pasos separados:** Las vulnerabilidades CRITICAL (CVSS 9.0+) son potencialmente explotables remotamente y deben bloquear el despliegue. Las HIGH (CVSS 7.0-8.9) son serias pero pueden requerir condiciones específicas; generan alerta pero no detienen la release.

### 9.6.2 Escaneo de secretos

Credenciales en el código fuente es una de las vulnerabilidades más comunes y peligrosas. El escaneo automático en CI previene que secretos lleguen al repositorio:

**TruffleHog**

```yaml
# GitHub Actions
- name: TruffleHog - Escanear secretos
  uses: trufflesecurity/trufflehog-action@v1
  with:
    path: ./
    base: ${{ github.event.before }}
    head: ${{ github.sha }}
    extra_args: --only-verified
```

TruffleHog no solo busca patrones regex (como `AKIA[0-9A-Z]{16}` para AWS keys), sino que **verifica** si las credenciales encontradas son válidas realizando llamadas API. Un secreto detectado pero no verificado es un warning; un secreto verificado debe ser un bloqueo total.

**git-secrets (pre-commit + CI)**

```yaml
# GitHub Actions
- name: git-secrets scan
  run: |
    git clone https://github.com/awslabs/git-secrets.git
    cd git-secrets && make install
    git-secrets --register-aws
    git-secrets --scan-history
```

**detect-secrets (Yelp)**

```yaml
# GitLab CI
secret-detection:
  stage: security
  image: python:3.12-slim
  before_script:
    - pip install detect-secrets
  script:
    - detect-secrets scan --all-files > secrets-report.json
    - |
      if detect-secrets audit secrets-report.json | grep -q "True"; then
        echo "::error::Secretos detectados y verificados en el codigo"
        exit 1
      fi
  artifacts:
    paths:
      - secrets-report.json
    expire_in: 30 days
```

**Buenas prácticas de escaneo de secretos:**

1. Ejecutar secreto-scan **antes** de cualquier otro step. No queremos hacer build ni test si hay credenciales expuestas.
2. Usar `--only-verified` para reducir falsos positivos.
3. Si se detecta un secreto verificado, **invalidar inmediatamente** la credencial comprometida (rotar keys).
4. Configurar pre-commit hooks además del escaneo en CI. El CI es la red de seguridad; el pre-commit es la prevención.

### 9.6.3 Firma de imágenes con Cosign

Cosign firma imágenes de contenedor para garantizar su integridad y procedencia:

```bash
# Generar par de llaves
cosign generate-key-pair

# Firmar una imagen
cosign sign --key cosign.key myorg/myapp:v1.2.3

# Verificar firma
cosign verify --key cosign.pub myorg/myapp:v1.2.3
```

**Cosign en GitHub Actions**

```yaml
- name: Firmar imagen con Cosign
  run: |
    cosign sign --yes \
      --key env://COSIGN_PRIVATE_KEY \
      ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```

**Verificación en el clúster de Kubernetes**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    cosign.sigstore.dev/message: "verificado"
spec:
  containers:
    - name: myapp
      image: ghcr.io/myorg/myapp:v1.2.3
```

**Integración con Kyverno (admission control)**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: cosign-verify
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
              - keyless:
                  subject: "https://github.com/myorg/myapp/.github/workflows/ci-cd.yml@refs/heads/main"
                  issuer: "https://token.actions.githubusercontent.com"
                  rekor:
                    url: https://rekor.sigstore.dev
```

### 9.6.4 Políticas con OPA/Kyverno

**OPA (Open Policy Agent)** y **Kyverno** son admission controllers de Kubernetes que validan, mutan o rechazan recursos basándose en políticas:

**Ejemplo Kyverno: prohibir imágenes sin tag inmutable**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-registries
      match:
        any:
        - resources:
            kinds:
              - Pod
      validate:
        message: "Solo se permiten imagenes con digest SHA256 (no :latest)"
        pattern:
          spec:
            containers:
              - image: "*@sha256:*"
```

**Ejemplo OPA/Gatekeeper: escanear imágenes antes de admitirlas**

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireImageVulnerabilityScan
metadata:
  name: require-trivy-scan
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "production"
      - "staging"
  parameters:
    maxCriticalSeverity: 0
    maxHighSeverity: 5
    scanResultFreshness: 24h
```

Con estas políticas, Kubernetes rechaza automáticamente pods en `production` que usen imágenes con vulnerabilidades CRITICAL detectadas en las últimas 24 horas.

**Kyverno: exigir SBOM**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-sbom
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-sbom-attestation
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
        - imageReferences:
            - "*"
          required: true
          attestations:
            - predicateType: https://spdx.dev/Document
              conditions:
                - all:
                  - key: "{{ name }}"
                    operator: NotEquals
                    value: ""
```

---

## 9.7 Laboratorio: Pipeline Completo GitHub Actions para Aplicación Real

En este laboratorio construiremos un pipeline CI/CD completo para una aplicación Node.js con Fastify, PostgreSQL y Redis. El pipeline incluye: lint, tests, build de imagen, push a GHCR, despliegue en Kubernetes (o VPS con docker compose), etiquetado automático de releases con semantic-release, y notificaciones en Slack/Discord.

### Objetivos

Al finalizar este laboratorio tendrás:

1. Un repositorio GitHub con Actions que ejecuta lint + test en cada push.
2. Build de imagen Docker multi-arquitectura en cada push a `main`.
3. Push automático a GitHub Container Registry.
4. Despliegue automático a staging y despliegue manual a producción.
5. Etiquetado automático de versiones con semantic-release.
6. Notificaciones de despliegue en Slack y Discord.

### Requisitos previos

- Cuenta en GitHub.
- Cuenta en Docker Hub (opcional, usaremos GHCR principalmente).
- Clúster de Kubernetes (Minikube, Kind, K3s, EKS, GKE, AKS) o VPS con Docker Compose.
- Token de acceso personal en GitHub con permisos `repo` y `packages:write`.

### Paso 1: Estructura del proyecto

```
lab-cicd/
├── .github/
│   ├── workflows/
│   │   └── ci-cd.yml
│   └── dependabot.yml
├── src/
│   ├── index.js
│   ├── routes/
│   │   └── items.js
│   └── config.js
├── tests/
│   ├── unit/
│   │   └── items.test.js
│   └── integration/
│       └── api.test.js
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── overlays/
│       ├── staging/
│       │   └── kustomization.yaml
│       └── production/
│           └── kustomization.yaml
├── scripts/
│   └── deploy-vps.sh
├── Dockerfile
├── docker-compose.yml
├── package.json
├── .dockerignore
├── .gitignore
└── .releaserc.json
```

### Paso 2: Configurar secrets en GitHub

```
Settings → Secrets and variables → Actions → Secrets

Nombre                          Valor
─────────────────────────────────────────────────────────
GHCR_TOKEN                      ghp_xxxxxxxxxxxx (PAT con packages:write)
SLACK_WEBHOOK                   https://hooks.slack.com/services/...
DISCORD_WEBHOOK                 https://discord.com/api/webhooks/...
KUBE_CONFIG_STAGING             (base64 del kubeconfig de staging)
KUBE_CONFIG_PRODUCTION          (base64 del kubeconfig de producción)
VPS_HOST                        123.45.67.89
VPS_USER                        deploy
VPS_SSH_KEY                     (clave privada SSH en base64)
SEMANTIC_RELEASE_TOKEN          ghp_xxxxxxxxxxxx (PAT con repo scope)

Settings → Secrets and variables → Actions → Variables

Nombre                          Valor
─────────────────────────────────────────────────────────
PROD_APPROVERS                  alice,bob
```

### Paso 3: Configurar semantic-release

```json
// .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    [
      "@semantic-release/changelog",
      {
        "changelogFile": "CHANGELOG.md"
      }
    ],
    [
      "@semantic-release/npm",
      {
        "npmPublish": false
      }
    ],
    [
      "@semantic-release/git",
      {
        "assets": ["package.json", "CHANGELOG.md"],
        "message": "chore(release): ${nextRelease.version}\n\n${nextRelease.notes}"
      }
    ],
    "@semantic-release/github"
  ]
}
```

### Paso 4: Workflow principal

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline - Lab

on:
  push:
    branches: [main, develop]
    paths-ignore:
      - '**.md'
      - '.github/dependabot.yml'
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      deploy_env:
        description: 'Seleccionar entorno de despliegue'
        required: true
        type: choice
        options:
          - staging
          - production
      skip_deploy:
        description: 'Solo build, sin deploy'
        type: boolean
        default: false
  schedule:
    - cron: '0 8 * * 1'   # Cada lunes a las 8:00 UTC (escaneo de seguridad semanal)

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  NODE_VERSION: '20'

jobs:
  # ═════════════════════════════════════════════════════════════
  # JOB 1: LINT & STATIC ANALYSIS
  # ═════════════════════════════════════════════════════════════
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      - name: ESLint
        run: npx eslint 'src/**/*.js' 'tests/**/*.js'

      - name: Prettier
        run: npx prettier --check 'src/**/*.js' 'tests/**/*.js'

      - name: Secret scan (truffleHog)
        uses: trufflesecurity/trufflehog-action@v1
        with:
          path: ./
          base: ${{ github.event.before }}
          head: ${{ github.sha }}
          extra_args: --only-verified

  # ═════════════════════════════════════════════════════════════
  # JOB 2: TESTS (matrix builds)
  # ═════════════════════════════════════════════════════════════
  test:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 15

    strategy:
      fail-fast: false
      matrix:
        node-version: [18, 20, 22]

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: lab
          POSTGRES_PASSWORD: lab
          POSTGRES_DB: labdb
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      - name: Unit tests
        run: npm run test:unit -- --coverage

      - name: Integration tests
        run: npm run test:integration
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://lab:lab@localhost:5432/labdb
          REDIS_URL: redis://localhost:6379/0

      - name: Subir coverage (solo Node 20)
        if: matrix.node-version == '20'
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: false

  # ═════════════════════════════════════════════════════════════
  # JOB 3: DOCKER BUILD & PUSH (condicional: solo en main/develop)
  # ═════════════════════════════════════════════════════════════
  build-and-push:
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    needs: test
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      packages: write

    outputs:
      digest: ${{ steps.build.outputs.digest }}
      tags: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup QEMU (emulación ARM para runners x86)
        uses: docker/setup-qemu-action@v3
        with:
          platforms: arm64,arm

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: network=host

      - name: Login a GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Login a Docker Hub (si configurado)
        if: env.DOCKERHUB_USERNAME != ''
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
        env:
          DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}

      - name: Extraer metadatos Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,format=short
            type=sha,format=long
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
            type=raw,value=staging,enable=${{ github.ref == 'refs/heads/develop' }}

      - name: Build y Push imagen multi-arquitectura
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          platforms: linux/amd64,linux/arm64
          provenance: true
          sbom: true
          build-args: |
            NODE_ENV=production
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}
          cache-from: type=gha,scope=${{ github.ref_name }}
          cache-to: type=gha,scope=${{ github.ref_name }},mode=max

      - name: Escanear imagen con Trivy
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          ignore-unfixed: true
          exit-code: 1

      - name: Subir resultados de Trivy a GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  # ═════════════════════════════════════════════════════════════
  # JOB 4: DEPLOY A STAGING (automático desde develop)
  # ═════════════════════════════════════════════════════════════
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.lab.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Desplegar en Kubernetes (staging)
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

          IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}"

          cd k8s/overlays/staging
          kustomize edit set image app-image=$IMAGE

          kustomize build . | kubectl apply -f - --namespace=staging
          kubectl rollout status deployment/lab-app --namespace=staging --timeout=5m

      - name: Smoke test en staging
        run: |
          kubectl wait --for=condition=ready pod \
            -l app=lab-app --namespace=staging --timeout=60s
          echo "Staging health check OK"

      - name: Notificar en Slack (staging)
        if: always()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && ':white_check_mark: Staging desplegado' || ':x: Fallo en staging' }}\nRepositorio: ${{ github.repository }}\nCommit: ${{ github.sha }}\nBranch: ${{ github.ref_name }}\nActor: ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # ═════════════════════════════════════════════════════════════
  # JOB 5: DEPLOY A PRODUCCIÓN (manual, requiere aprobación)
  # ═════════════════════════════════════════════════════════════
  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://lab.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Desplegar en Kubernetes (producción)
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_PRODUCTION }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

          IMAGE="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}"

          cd k8s/overlays/production
          kustomize edit set image app-image=$IMAGE

          kustomize build . | kubectl apply -f - --namespace=production
          kubectl rollout status deployment/lab-app --namespace=production --timeout=10m

      - name: Rollback automático si falla
        if: failure()
        run: |
          echo "Ejecutando rollback en produccion..."
          kubectl rollout undo deployment/lab-app --namespace=production

      - name: Smoke test en producción
        if: success()
        run: |
          kubectl wait --for=condition=ready pod \
            -l app=lab-app --namespace=production --timeout=60s
          echo "Produccion health check OK"

      - name: Notificar en Slack (producción)
        if: always()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && ':rocket: PRODUCCION desplegada exitosamente' || ':x: DESPLIEGUE FALLIDO en PRODUCCION - ROLLBACK EJECUTADO' }}\nRepositorio: ${{ github.repository }}\nCommit: ${{ github.sha }}\nTag: ${{ github.sha }}\nActor: ${{ github.actor }}\nURL: https://lab.example.com"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # ═════════════════════════════════════════════════════════════
  # JOB 6: DESPLIEGUE ALTERNATIVO - VPS con Docker Compose
  # ═════════════════════════════════════════════════════════════
  deploy-vps:
    if: github.ref == 'refs/heads/main' && github.event.inputs.deploy_env == 'production'
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:
      name: production

    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH a VPS
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/lab-app

            # Login a GHCR
            echo "${{ secrets.GITHUB_TOKEN }}" | \
              docker login ghcr.io -u ${{ github.actor }} --password-stdin

            # Descargar nueva imagen
            IMAGE="ghcr.io/${{ github.repository }}:${{ github.sha }}"
            docker pull $IMAGE

            # Etiquetar como latest
            docker tag $IMAGE ghcr.io/${{ github.repository }}:latest

            # Actualizar docker compose
            echo "IMAGE_TAG=sha-${{ github.sha }}" > .env

            # Pull + Up (solo recrea servicios que cambiaron)
            docker compose pull
            docker compose up -d --remove-orphans

            # Esperar health check
            sleep 5
            docker compose ps

            # Limpiar imágenes viejas
            docker image prune -af --filter "until=24h"

      - name: Notificar resultado VPS
        if: always()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && ':computer: VPS actualizado exitosamente' || ':x: Fallo en VPS' }}\nHost: ${{ secrets.VPS_HOST }}\nCommit: ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # ═════════════════════════════════════════════════════════════
  # JOB 7: SEMANTIC RELEASE (etiquetado automático)
  # ═════════════════════════════════════════════════════════════
  semantic-release:
    if: github.ref == 'refs/heads/main'
    needs: deploy-production
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.SEMANTIC_RELEASE_TOKEN }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Instalar dependencias
        run: npm ci

      - name: Ejecutar semantic-release
        run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.SEMANTIC_RELEASE_TOKEN }}
          GIT_AUTHOR_NAME: ${{ github.actor }}
          GIT_AUTHOR_EMAIL: ${{ github.actor }}@users.noreply.github.com

  # ═════════════════════════════════════════════════════════════
  # JOB 8: NOTIFICACIÓN EN DISCORD
  # ═════════════════════════════════════════════════════════════
  notify-discord:
    needs: [deploy-staging, deploy-production, deploy-vps, semantic-release]
    if: always()
    runs-on: ubuntu-latest

    steps:
      - name: Preparar mensaje
        id: status
        run: |
          echo "PROD=${{ needs.deploy-production.result }}" >> $GITHUB_OUTPUT
          echo "STAGING=${{ needs.deploy-staging.result }}" >> $GITHUB_OUTPUT
          echo "VPS=${{ needs.deploy-vps.result }}" >> $GITHUB_OUTPUT
          echo "RELEASE=${{ needs.semantic-release.result }}" >> $GITHUB_OUTPUT

      - name: Notificar en Discord
        uses: tsickert/discord-webhook-action@v5
        with:
          webhook-url: ${{ secrets.DISCORD_WEBHOOK }}
          title: "CI/CD Pipeline - ${{ github.repository }}"
          description: |
            **Branch:** ${{ github.ref_name }}
            **Commit:** ${{ github.sha }}
            **Commit Message:** ${{ github.event.head_commit.message }}
            **Actor:** ${{ github.actor }}
            **Run URL:** ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

            **Resultados:**
            - Staging: ${{ needs.deploy-staging.result }}
            - Produccion K8s: ${{ needs.deploy-production.result }}
            - Produccion VPS: ${{ needs.deploy-vps.result }}
            - Release: ${{ needs.semantic-release.result }}
          color: ${{ needs.deploy-production.result == 'success' && 65280 || 16711680 }}

  # ═════════════════════════════════════════════════════════════
  # JOB 9: ESCANEO DE SEGURIDAD SEMANAL PROGRAMADO
  # ═════════════════════════════════════════════════════════════
  weekly-security-scan:
    if: github.event_name == 'schedule' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4

      - name: Descargar imagen latest
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
            docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

      - name: Escaneo completo Trivy
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          format: 'sarif'
          output: 'trivy-weekly.sarif'
          severity: 'CRITICAL,HIGH,MEDIUM'
          ignore-unfixed: true
          exit-code: 0     # No bloquear en escaneo semanal, solo reportar

      - name: Subir resultados
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-weekly.sarif'

      - name: Notificar escaneo completado
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {"text": ":mag: Escaneo de seguridad semanal completado\nRepositorio: ${{ github.repository }}\nRevisar Security tab para resultados"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Paso 5: Configurar dependabot para actualizaciones automáticas

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    versioning-strategy: increase
    labels:
      - "dependencies"
      - "automated"

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    labels:
      - "dependencies"
      - "docker"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    labels:
      - "dependencies"
      - "ci-cd"
```

### Paso 6: Verificación del pipeline

**Flujo esperado:**

```
1. Push a develop
   → lint ✓
   → test ✓
   → build-and-push ✓ (tag: staging)
   → deploy-staging ✓ (automático)

2. Merge develop → main
   → lint ✓
   → test ✓
   → build-and-push ✓ (tag: latest)
   → deploy-production ✓ (espera aprobación manual)
   → semantic-release ✓ (crea tag v1.2.3, release en GitHub)
   → notify-discord ✓

3. Crisis: falla el deploy a producción
   → kubectl rollout undo ✓ (rollback automático)
   → notificación en Slack/Discord ✓
   → equipo investiga con los logs del workflow

4. Lunes 8:00 AM
   → weekly-security-scan ✓
   → dependabot abre PRs si hay actualizaciones pendientes
```

### Paso 7: Personalización y extensiones

**Agregar escaneo de licencias:**

```yaml
- name: License check
  run: |
    npx license-checker --production --onlyAllow "MIT;Apache-2.0;ISC;BSD-2-Clause;BSD-3-Clause"
```

**Agregar pruebas de carga:**

```yaml
load-test:
  needs: deploy-staging
  runs-on: ubuntu-latest
  steps:
    - name: k6 load test
      uses: grafana/k6-action@v0.3.1
      with:
        filename: tests/load/staging-test.js
        flags: --vus 100 --duration 60s
```

**Agregar aprobación con comentarios:**

```yaml
- name: Solicitar aprobación con información detallada
  uses: trstringer/manual-approval@v1
  with:
    secret: ${{ github.TOKEN }}
    approvers: ${{ vars.PROD_APPROVERS }}
    minimum-approvals: 1
    issue-title: "Despliegue a PRODUCCION - ${{ github.sha }}"
    issue-body: |
      **Commit:** `${{ github.sha }}`
      **Mensaje:** ${{ github.event.head_commit.message }}
      **Autor:** ${{ github.actor }}
      **Scaneo Trivy:** ${{ github.server_url }}/${{ github.repository }}/security/code-scanning
      **Diff:** ${{ github.event.compare }}
```

---

## Resumen del capítulo

En este capítulo hemos recorrido el espectro completo de CI/CD con Docker:

1. **Fundamentos**: por qué Docker revolucionó CI/CD eliminando la deriva de configuración y garantizando artefactos inmutables.

2. **GitHub Actions**: el sistema más popular, con workflows YAML, matrix builds, secrets, environments y ejemplos completos para Node.js y Java Spring Boot.

3. **GitLab CI**: pipeline integrado con DinD, socket binding, Kaniko, Auto DevOps y GitLab Container Registry.

4. **Jenkins**: el veterano, con Declarative Pipeline, agentes Docker y el ecosistema de plugins.

5. **Estrategias de despliegue**: Rolling Update, Blue/Green, Canary, A/B Testing y GitOps con ArgoCD y Flux.

6. **Seguridad en CI/CD**: escaneo de vulnerabilidades (Trivy), escaneo de secretos (TruffleHog), firma de imágenes (Cosign) y políticas de admission control (OPA/Kyverno).

7. **Laboratorio práctico**: pipeline completo con lint, test, build multi-arquitectura, push a GHCR, despliegue en Kubernetes y VPS, semantic-release y notificaciones.

El pipeline CI/CD es el latido del desarrollo moderno. Con Docker como base y las herramientas y estrategias presentadas en este capítulo, estás equipado para construir, probar, asegurar y desplegar aplicaciones con la confianza de que el entorno de CI es idéntico al de producción, que cada artefacto es verificable y que cada despliegue es controlado y reversible.

> *"En CI/CD, el objetivo no es velocidad. El objetivo es confianza. La velocidad es el efecto secundario de la confianza."* — Charity Majors
