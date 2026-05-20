# Capitulo 16: CI/CD con Git

La integracion continua (CI) y la entrega/despliegue continuo (CD) son practicas fundamentales del desarrollo de software moderno. Git, como sistema de control de versiones distribuido, es el eje central que desencadena y coordina estos pipelines automatizados. Este capitulo explora como integrar Git con las principales plataformas de CI/CD.

---

## 16.1 CI/CD: Definicion y Beneficios

### Definicion

| Termino | Definicion |
|---------|------------|
| **Integracion Continua (CI)** | Practica de fusionar cambios de codigo frecuentemente (varias veces al dia) en una rama compartida, activando builds y pruebas automatizadas. |
| **Entrega Continua (CD)** | Extension de CI donde cada cambio que pasa las pruebas se empaqueta y esta listo para desplegarse manualmente. |
| **Despliegue Continuo (CD)** | Cada cambio que pasa las pruebas se despliega automaticamente a produccion sin intervencion manual. |

### Ciclo CI/CD Tipico

```
[Git Push] --> [Build] --> [Unit Tests] --> [Integration Tests]
    --> [Security Scan] --> [Package] --> [Deploy Staging]
    --> [Smoke Tests] --> [Deploy Production]
```

### Beneficios

- **Deteccion temprana de errores**: los tests se ejecutan en cada cambio.
- **Feedback inmediato**: el desarrollador sabe en minutos si su cambio rompe algo.
- **Reduccion de conflictos de merge**: integracion frecuente evita divergencias grandes.
- **Automatizacion de tareas repetitivas**: build, test, deploy, linting.
- **Trazabilidad**: cada artefacto desplegado esta vinculado a un commit/tag de Git.
- **Velocidad de entrega**: de semanas a horas o minutos.

> **Tip**: Adopta CI/CD incrementalmente. Comienza con CI basico (build + tests), luego agrega CD (deploy staging manual), y finalmente despliegue continuo cuando tengas confianza en la suite de pruebas.

---

## 16.2 Pipelines Desencadenados por Eventos de Git

Los pipelines de CI/CD se disparan automaticamente por eventos del repositorio Git.

### Eventos Comunes

| Evento Git | Gatillo Tipico | Proposito |
|------------|----------------|-----------|
| `push` a rama | Cada commit/rama | CI: build + test |
| `pull_request` | PR abierto/actualizado | Validar antes del merge |
| `tag` (ej. v1.0.0) | Creacion de tag | CD: release + deploy |
| `schedule` (cron) | Tiempo programado | Tests nocturnos, seguridad |
| `release` | Release publicado | Deploy a produccion |
| `workflow_dispatch` | Manual | Ejecucion bajo demanda |

### Flujo de Eventos

```
Desarrollador hace git push --> Webhook notifica a CI/CD
    --> Pipeline lee configuracion (YAML)
    --> Ejecuta jobs definidos
    --> Reporta resultado (commit status)
```

> **Advertencia crítica:** `pull_request_target` es la **causa #1 de pwn requests** en GitHub Actions. Este evento ejecuta el workflow en el contexto del repositorio base (con acceso a todos los secrets), incluso en PRs de forks maliciosos. Evítalo a menos que sea absolutamente necesario, y si lo usas, nunca hagas checkout del código del PR sin aislarlo. Prefiere `pull_request` para la mayoría de los casos.

---

## 16.3 GitHub Actions

GitHub Actions es el sistema de CI/CD nativo de GitHub, integrado profundamente con el repositorio y sus eventos.

### 16.3.1 Workflows, Jobs y Steps

Un **workflow** es un proceso automatizado definido en `.github/workflows/archivo.yml`. Un workflow contiene uno o mas **jobs** que ejecutan **steps**.

```
Workflow (.github/workflows/ci.yml)
  |
  +-- Job: build (ubuntu-latest)
  |     +-- Step: Checkout code
  |     +-- Step: Setup Node.js
  |     +-- Step: Install dependencies
  |     +-- Step: Build project
  |
  +-- Job: test (ubuntu-latest) [needs: build]
  |     +-- Step: Checkout code
  |     +-- Step: Run unit tests
  |     +-- Step: Upload coverage
  |
  +-- Job: lint (ubuntu-latest) [parallel]
        +-- Step: Checkout code
        +-- Step: Run linter
```

### 16.3.2 Ejemplo: Test + Build + Deploy

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  release:
    types: [published]

env:
  NODE_VERSION: '20'

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run lint

  build:
    name: Build
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  deploy-staging:
    name: Deploy Staging
    if: github.ref == 'refs/heads/develop'
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - run: echo "Deploying to staging..."
      # - uses: cloud-deploy-action@v1

  deploy-production:
    name: Deploy Production
    if: github.event_name == 'release'
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - run: echo "Deploying to production..."
      # - uses: cloud-deploy-action@v1
```

### 16.3.3 Actions del Marketplace

El [GitHub Marketplace](https://github.com/marketplace?type=actions) ofrece miles de acciones reutilizables.

| Action | Uso |
|--------|-----|
| `actions/checkout` | Clonar el repositorio |
| `actions/setup-node` | Configurar Node.js |
| `actions/cache` | Cachear dependencias |
| `actions/upload-artifact` | Compartir archivos entre jobs |
| `docker/build-push-action` | Construir y publicar imagen Docker |
| `peaceiris/actions-gh-pages` | Desplegar a GitHub Pages |
| `semantic-release/semantic-release` | Versionado automatico |
| `codecov/codecov-action` | Subir cobertura a Codecov |

```yaml
# Ejemplo: usar action del marketplace para publicar en npm
- uses: JS-DevTools/npm-publish@v3
  with:
    token: ${{ secrets.NPM_TOKEN }}
```

### 16.3.4 Secrets y Environments

Los **secrets** almacenan informacion sensible encriptada (tokens, claves SSH, contraseñas). Se configuran en Settings > Secrets and variables > Actions.

```yaml
steps:
  # CORRECTO: El token se inyecta como variable de entorno
  # que la herramienta de deploy lee internamente. NUNCA hagas echo.
  - run: npx deploy-tool --production
    env:
      DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

> **Peligro:** Nunca hagas `echo $DEPLOY_TOKEN` o `printenv` en un step que use secrets. Los logs de CI son texto plano y cualquier persona con acceso de lectura al repositorio puede verlos. Si un atacante modifica el código para imprimir variables de entorno, el token queda expuesto. Usa secrets solo dentro de `env:` o `with:` y deja que las herramientas los lean internamente.

```yaml
# PELIGROSO: El token se imprime en logs
- run: echo "Deploying with token $DEPLOY_TOKEN"
  env:
    DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}

# CORRECTO: La herramienta lee el token sin exponerlo
- run: deploy-cli --production
  env:
    DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

Los **environments** permiten aprobaciones manuales, protecciones y configuraciones especificas por entorno.

```yaml
deploy-prod:
  environment:
    name: production
    url: https://miapp.com
  steps: ...
```

> **Tip**: Usa `${{ secrets.NOMBRE }}` solo dentro de `env:` o `with:`. Nunca uses `echo $SECRET` en un step que pueda exponer el valor en logs.

### 16.3.5 Matrix Builds

Ejecuta el mismo job con multiples combinaciones de parametros.

```yaml
test:
  strategy:
    matrix:
      os: [ubuntu-latest, windows-latest, macos-latest]
      node: [18, 20, 22]
      exclude:
        - os: windows-latest
          node: 18
  runs-on: ${{ matrix.os }}
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node }}
    - run: npm ci
    - run: npm test
```

---

### 16.3.6 GitHub Merge Queue

El **Merge Queue** de GitHub (disponible en planes Team y Enterprise) serializa merges a ramas protegidas, garantizando que cada PR se valida contra el estado más reciente de `main` antes de mergear:

```
PR #1  ──>  [Cola de Merge]
PR #2  ──>       │
PR #3  ──>       ▼
           ┌──────────────────┐
           │ 1. Agrupa PRs    │
           │ 2. Crea rama tmp │
           │ 3. Ejecuta CI    │
           │ 4. Si pasa: merge│
           │ 5. Si falla:     │
           │    fuera de cola │
           └──────────────────┘
```

```yaml
# Requiere protección de rama con "Require merge queue"
# Los PRs se encolan automáticamente al aprobarse
# El CI se ejecuta en el commit mergeado (no en la rama del PR)

# Configuración en el workflow:
on:
  merge_group:
    types: [checks_requested]
```

**Beneficio:** Elimina el problema de "mi PR pasó CI pero main avanzó y ahora necesito re-mergear". La cola garantiza que cada merge se valida contra el HEAD real de main.

### GitLab Merge Trains

GitLab ofrece funcionalidad equivalente con **Merge Trains** (Premium):

```yaml
# .gitlab-ci.yml
merge_trains:
  stage: merge
  script: echo "en cola"
  only:
    - merge_requests
```

---

## 16.4 GitLab CI/CD

GitLab ofrece un sistema de CI/CD profundamente integrado, definido en el archivo `.gitlab-ci.yml`.

### 16.4.1 Estructura Basica

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  NODE_VERSION: "20"

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/

before_script:
  - npm ci

build-job:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

unit-test-job:
  stage: test
  script:
    - npm test
  coverage: '/Statements\s+:\s(\d+\.\d+)%/'

deploy-job:
  stage: deploy
  script:
    - echo "Deploying..."
  environment:
    name: production
    url: https://miapp.com
  only:
    - main
```

### 16.4.2 Pipeline Multi-Etapa

```yaml
stages:
  - lint
  - test
  - build
  - security
  - deploy-staging
  - deploy-prod

lint:
  stage: lint
  script: npm run lint

unit-tests:
  stage: test
  script: npm test
  parallel: 3

integration-tests:
  stage: test
  script: npm run test:integration

build:
  stage: build
  needs: [lint, unit-tests, integration-tests]
  script: npm run build
  artifacts:
    paths: [dist/]

sast:
  stage: security
  script:
    - echo "Running SAST scan"

deploy-staging:
  stage: deploy-staging
  script: ./deploy.sh staging
  environment: staging
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy-prod:
  stage: deploy-prod
  script: ./deploy.sh production
  when: manual
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_TAG
```

### 16.4.3 GitLab Runner

Los runners ejecutan los jobs. Pueden ser:
- **Shared runners**: proporcionados por GitLab (SaaS).
- **Specific runners**: auto-gestionados en tu infraestructura.
- **Group runners**: compartidos entre proyectos del grupo.

```bash
# Registrar un runner local
gitlab-runner register \
  --url https://gitlab.com \
  --registration-token PROYECTO_TOKEN \
  --executor docker \
  --docker-image alpine:latest \
  --description "Mi Runner Local"
```

---

## 16.5 Bitbucket Pipelines

Bitbucket Pipelines usa `bitbucket-pipelines.yml` en la raiz del repositorio.

```yaml
# bitbucket-pipelines.yml
image: node:20

pipelines:
  default:
    - step:
        name: Test and Build
        caches:
          - node
        script:
          - npm ci
          - npm test
          - npm run build
        artifacts:
          - dist/**

  branches:
    main:
      - step:
          name: Deploy to Production
          deployment: production
          script:
            - pipe: atlassian/aws-s3-deploy:1.0.0
              variables:
                AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID
                AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY
                AWS_DEFAULT_REGION: $AWS_DEFAULT_REGION
                S3_BUCKET: 'my-bucket'
                LOCAL_PATH: 'dist'

  pull-requests:
    '**':
      - step:
          name: PR Checks
          script:
            - npm ci
            - npm test
            - npm run lint
```

---

## 16.6 CircleCI

CircleCI es una de las plataformas de CI/CD en la nube más veteranas, con un modelo de configuración declarativa en `.circleci/config.yml`.

### 16.6.1 Estructura Básica

```yaml
# .circleci/config.yml
version: 2.1

orbs:
  node: circleci/node@5.0.3

jobs:
  test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - node/install-packages:
          pkg-manager: npm
      - run:
          name: Run unit tests
          command: npm test
      - run:
          name: Run linter
          command: npm run lint

  build:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - node/install-packages:
          pkg-manager: npm
      - run:
          name: Build project
          command: npm run build
      - persist_to_workspace:
          root: .
          paths:
            - dist/

  deploy:
    docker:
      - image: cimg/base:stable
    steps:
      - attach_workspace:
          at: .
      - run:
          name: Deploy to production
          command: ./deploy.sh

workflows:
  test-build-deploy:
    jobs:
      - test
      - build:
          requires:
            - test
      - deploy:
          requires:
            - build
          filters:
            branches:
              only: main
```

### 16.6.2 Orbs (Acciones Reutilizables)

Los **orbs** son el equivalente de CircleCI a las Actions de GitHub Marketplace:

```yaml
orbs:
  aws-s3: circleci/aws-s3@3.1.1
  slack: circleci/slack@4.10

jobs:
  notify:
    docker:
      - image: cimg/base:stable
    steps:
      - slack/notify:
          event: fail
          template: basic_fail_1
```

### 16.6.3 Paralelismo y Test Splitting

```yaml
jobs:
  test:
    parallelism: 4  # Divide tests en 4 workers
    steps:
      - checkout
      - run:
          command: |
            TEST_FILES=$(circleci tests glob "test/**/*.test.js" | circleci tests split --split-by=timings)
            jest $TEST_FILES
```

---

## 16.7 Jenkins + Git

Jenkins es un servidor de automatizacion auto-gestionado con gran flexibilidad y ecosistema de plugins.

### 16.6.1 Webhooks y Polling

| Metodo | Descripcion | Recomendado |
|--------|-------------|-------------|
| **Webhook** | GitHub/GitLab notifica a Jenkins en cada push | Si, respuesta inmediata |
| **Polling (SCM)** | Jenkins consulta periodicamente el repo (`* * * * *`) | No, consume recursos, latencia |

```bash
# Configurar webhook en GitHub:
# Settings > Webhooks > Add webhook
# Payload URL: https://jenkins.midominio.com/github-webhook/
# Content type: application/json
# Events: Just the push event
```

### 16.6.2 Jenkinsfile (Declarativo)

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        NODE_VERSION = '20'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm test'
                    }
                }
                stage('Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
            }
        }

        stage('Build') {
            when {
                branch 'main'
            }
            steps {
                sh 'npm run build'
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
            }
        }

        stage('Deploy') {
            when {
                tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
            }
            steps {
                sh './deploy.sh production'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            emailext(
                subject: "Pipeline exitoso: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "El build paso correctamente.",
                to: 'equipo@midominio.com'
            )
        }
        failure {
            emailext(
                subject: "Pipeline fallido: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Revisar logs: ${env.BUILD_URL}",
                to: 'equipo@midominio.com'
            )
        }
    }
}
```

---

## 16.8 Semantic Release

Semantic Release automatiza el versionado y publicacion basandose en los mensajes de commit convencionales.

### 16.7.1 Versionado Automatico Basado en Conventional Commits

```
feat: agregar login     --> Version MINOR incrementa  (1.0.0 -> 1.1.0)
fix: corregir overflow  --> Version PATCH incrementa  (1.1.0 -> 1.1.1)

feat!: nueva API
BREAKING CHANGE: cambia contrato
                        --> Version MAJOR incrementa  (1.1.1 -> 2.0.0)
```

### 16.7.2 Configuracion con GitHub Actions

```bash
npm install --save-dev semantic-release @semantic-release/git
```

```json
// .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/git",
    "@semantic-release/github"
  ]
}
```

```yaml
# .github/workflows/release.yml
name: Semantic Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 16.7.3 Generacion de Changelog

Semantic Release genera automaticamente `CHANGELOG.md` con el formato "Keep a Changelog":

```markdown
## [1.2.0] - 2026-05-19

### Features
- Agregar soporte multi-idioma (abc1234)

### Bug Fixes
- Corregir overflow en calculadora (def5678)
```

### 16.7.4 Publicacion de Release

Al ejecutarse, semantic release:
1. Analiza los commits desde el ultimo release.
2. Determina la nueva version (MAJOR, MINOR, PATCH).
3. Genera las release notes.
4. Actualiza `CHANGELOG.md`.
5. Publica la release en GitHub con el tag correspondiente.
6. Publica el paquete en npm (si esta configurado).

---

## 16.9 Automatizacion de Versionado con Tags

```bash
# Versionado manual con tags anotados
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# Obtener la version actual con git describe
git describe --tags --abbrev=0
# Salida: v1.0.0

# Incrementar version automaticamente (script)
npm version patch   # 1.0.0 -> 1.0.1
npm version minor   # 1.0.1 -> 1.1.0
npm version major   # 1.1.0 -> 2.0.0
```

```bash
#!/bin/bash
# Script: auto-tag.sh - Versionado automatico
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
echo "Ultimo tag: $LAST_TAG"

# Leer tipo de cambio desde mensaje de commit
COMMIT_MSG=$(git log -1 --pretty=%B)
if echo "$COMMIT_MSG" | grep -qE '^BREAKING CHANGE|!:' ; then
    npx semver -i major "$LAST_TAG"
elif echo "$COMMIT_MSG" | grep -qE '^feat' ; then
    npx semver -i minor "$LAST_TAG"
elif echo "$COMMIT_MSG" | grep -qE '^fix' ; then
    npx semver -i patch "$LAST_TAG"
fi
```

---

## 16.10 Estrategias de Deploy

### Comparativa de Estrategias

| Estrategia | Descripcion | Downtime | Rollback | Complejidad |
|------------|-------------|----------|----------|-------------|
| **Blue-Green** | Dos entornos identicos, switch instantaneo | Cero | Instantaneo | Media |
| **Canary** | Trafico incremental a nueva version | Cero | Rapido | Alta |
| **Rolling** | Reemplazo progresivo de instancias | Cero | Manual | Baja |
| **Recreate** | Apagar todo, desplegar nuevo | Alto | Lento | Muy baja |

### Blue-Green

```
[Load Balancer]
       |
   +---|---+
   v       v
[BLUE]  [GREEN]
(prod)  (nueva)

# Switch: cambiar direccion del LB a GREEN
```

```yaml
# Implementacion Blue-Green con Docker
deploy-blue-green:
  script:
    - docker compose up -d app-green
    - docker compose exec app-green ./health-check.sh
    - |
      if ./health-check.sh; then
        # Switch trafico
        docker compose exec nginx nginx -s reload -c /etc/nginx/green.conf
        # Opcional: mantener blue para rollback rapido
        echo "Desplegado en GREEN. Blue disponible para rollback."
      else
        docker compose stop app-green
        echo "Health check fallo. Rollback inmediato."
      fi
```

### Canary

```
Trafico: 5% -> 25% -> 50% -> 100%  (monitoreando metricas)
         v1.0                           v2.0
```

### Rolling

```
[Instancia 1: v2.0] ... [Instancia 2: v1.0] ... [Instancia 3: v1.0]
[Instancia 1: v2.0] ... [Instancia 2: v2.0] ... [Instancia 3: v1.0]
[Instancia 1: v2.0] ... [Instancia 2: v2.0] ... [Instancia 3: v2.0]
```

---

## 16.11 Integracion con Docker y Kubernetes

### Docker Build en CI/CD

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    tags: ['v*']

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: mi-usuario/mi-app
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,format=short
            type=raw,value=latest

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Deploy a Kubernetes

```yaml
# .github/workflows/deploy-k8s.yml
name: Deploy to Kubernetes

on:
  workflow_run:
    workflows: ["Docker Build and Push"]
    types: [completed]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set image tag
        run: |
          TAG=${GITHUB_SHA::7}
          sed -i "s|IMAGE_TAG|$TAG|g" k8s/deployment.yaml

      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v4
        with:
          manifests: k8s/deployment.yaml
          namespace: production
          strategy: rolling
```

```yaml
# k8s/deployment.yaml (fragmento)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      containers:
        - name: app
          image: mi-usuario/mi-app:IMAGE_TAG
```

### GitOps con ArgoCD

GitOps usa un repositorio Git como fuente unica de verdad para la configuracion de infraestructura.

```
[Repositorio App] --CI--> [Imagen Docker] --Push--> [Registry]
                                                          |
[Repositorio Config] <--Actualizar tag de imagen--       |
         |
         v
[ArgoCD] --Sync automatico--> [Cluster Kubernetes]
```

---

## 16.12 Buenas Practicas de CI/CD con Git

### Pipelines

1. **Pipeline como codigo**: almacena la configuracion de CI/CD en `.github/workflows/`, `.gitlab-ci.yml`, etc., versionada junto al codigo.
2. **Jobs idempotentes**: ejecutar el mismo job dos veces debe producir el mismo resultado.
3. **Builds rapidos (Fast CI)**:
   - Cachea dependencias entre ejecuciones.
   - Paraleliza jobs independientes.
   - Usa matrix builds para variantes.
   - No ejecutes todo en cada push; usa filtros de paths.

```yaml
# Filtro de paths: solo ejecutar si cambia el codigo fuente
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
```

4. **Secrets seguros**: nunca hardcodees tokens o contraseñas en workflows.
5. **Revisar pipelines externos**: en PRs de forks, usar `pull_request_target` con `actions/checkout` seguro.
6. **Notificaciones**: alertar al equipo en caso de fallo via Slack, email, MS Teams.

### Estrategia de Ramas y CI/CD

| Estrategia de Rama | Pipeline | Entorno |
|--------------------|----------|---------|
| `main` / `master` | CI completa + deploy produccion (si CD) | Produccion |
| `develop` | CI completa + deploy staging | Staging |
| `feature/*` | CI basica (build + test) | - |
| `release/*` | CI + E2E tests | Pre-produccion |
| `hotfix/*` | CI + tests | Produccion (via PR a main) |

### Checklist de CI/CD Saludable

- [ ] Todo push a `main` pasa la suite de tests completa.
- [ ] Los PRs requieren que CI pase antes de hacer merge.
- [ ] Hay ambientes de staging y produccion separados.
- [ ] Los deploys son reproducibles (infraestructura como codigo).
- [ ] Existen mecanismos de rollback automaticos o rapidos.
- [ ] Los secrets roten periodicamente.
- [ ] Los logs de CI/CD se retienen por al menos 30 dias.
- [ ] Hay tests de humo (smoke tests) post-deploy.
- [ ] El equipo recibe notificaciones de fallos.

> **Tip**: Empieza con lo minimo viable: un pipeline de CI que ejecute `test` en cada PR. Agrega complejidad (deploy, security scans, performance tests) gradualmente segun las necesidades del equipo.

---

## 16.13 Dependabot y Renovate: Automatización de Dependencias

Mantener dependencias actualizadas es crítico para la seguridad. Dependabot (GitHub nativo) y Renovate (multi-plataforma) automatizan esta tarea.

### Dependabot (GitHub)

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
      - "security"
    reviewers:
      - "@equipo/backend"

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "daily"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

```yaml
# Integración con CI: el workflow se dispara en PRs de Dependabot
on:
  pull_request:
    branches: [main]
  # Dependabot crea PRs con actualizaciones; el CI los valida automáticamente
```

### Renovate (Alternativa Multi-Plataforma)

Renovate soporta GitHub, GitLab, Bitbucket y más. Ofrece más control que Dependabot:

```json
// renovate.json
{
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true
    },
    {
      "matchUpdateTypes": ["major"],
      "labels": ["breaking-change"],
      "reviewers": ["team:architects"]
    }
  ]
}
```

## 16.14 Security Scanning en CI/CD

Integra escaneos de seguridad en el pipeline para detectar vulnerabilidades antes del deploy:

```yaml
# .github/workflows/security.yml
name: Security Scanning
on: [push, pull_request]

jobs:
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: CodeQL Analysis (SAST)
        uses: github/codeql-action/init@v3
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

  secret-scanning:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Gitleaks scan
        uses: gitleaks/gitleaks-action@v2

  container-scanning:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t app:${{ github.sha }} .
      - name: Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: app:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run npm audit
        run: npm audit --audit-level=high
        continue-on-error: true
```

| Escaneo | Herramienta | Qué detecta |
|---------|------------|-------------|
| **SAST** | CodeQL, SonarQube, Semgrep | Vulnerabilidades en código propio |
| **Secretos** | Gitleaks, TruffleHog, GitHub Secret Scanning | API keys, tokens, contraseñas |
| **Contenedores** | Trivy, Snyk, Aqua | Vulnerabilidades en imágenes Docker |
| **Dependencias** | npm audit, Dependabot, Snyk, OWASP Dependency Check | CVEs en librerías de terceros |
| **Infraestructura** | tfsec, Checkov, Terrascan | IaC mal configurada (Terraform, CloudFormation) |

## 16.15 Estrategias de Deploy Avanzadas (YAML Concreto)

### Canary con Istio

```yaml
# k8s/canary.yaml: VirtualService con traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: mi-app
spec:
  hosts:
    - mi-app.example.com
  http:
    - match:
        - headers:
            canary:
              exact: "true"
      route:
        - destination:
            host: mi-app
            subset: v2
    - route:
        - destination:
            host: mi-app
            subset: v1
          weight: 95
        - destination:
            host: mi-app
            subset: v2
          weight: 5
```

```bash
# Progresión de tráfico en CI/CD:
# 1. Deploy v2 con 5% de tráfico
# 2. Monitorear métricas por 10 min
# 3. Si todo OK: aumentar a 25%, luego 50%, luego 100%
# 4. Si hay errores: revertir a v1
kubectl apply -f k8s/canary-5.yaml && sleep 600
kubectl apply -f k8s/canary-50.yaml && sleep 600
kubectl apply -f k8s/canary-100.yaml
```

### Blue-Green con AWS ALB

```yaml
# CI/CD step para blue-green con AWS ALB
deploy-blue-green:
  script:
    - TARGET_GROUP_BLUE="arn:aws:elasticloadbalancing:...:targetgroup/app-blue/xxx"
    - TARGET_GROUP_GREEN="arn:aws:elasticloadbalancing:...:targetgroup/app-green/yyy"
    - |
      # 1. Determinar qué color está activo
      ACTIVE_COLOR=$(aws elbv2 describe-listeners \
        --listener-arn $LISTENER_ARN \
        --query "Listeners[0].DefaultActions[0].TargetGroupArn" \
        --output text)

      if [[ "$ACTIVE_COLOR" == "$TARGET_GROUP_BLUE" ]]; then
        DEPLOY_COLOR="green"
        DEPLOY_TG="$TARGET_GROUP_GREEN"
      else
        DEPLOY_COLOR="blue"
        DEPLOY_TG="$TARGET_GROUP_BLUE"
      fi

      # 2. Desplegar nueva versión en el color inactivo
      aws ecs update-service --cluster prod --service app-$DEPLOY_COLOR --force-new-deployment
      aws ecs wait services-stable --cluster prod --services app-$DEPLOY_COLOR

      # 3. Cambiar tráfico al nuevo color (rollback instantáneo: revertir este cambio)
      aws elbv2 modify-listener \
        --listener-arn $LISTENER_ARN \
        --default-actions Type=forward,TargetGroupArn=$DEPLOY_TG

      echo "Deploy completado en $DEPLOY_COLOR"
```

## 16.16 Optimización de Costos en CI/CD

### GitHub Actions Pricing

| Plan | Minutos gratis/mes | Costo extra |
|------|-------------------|-------------|
| Free | 2,000 min | N/A |
| Team | 3,000 min | $0.008/min (Linux) |
| Enterprise | 50,000 min | Negociable |

**Los runners de macOS son 10x más caros y los de Windows 2x más caros que Linux.**

### Estrategias de Reducción de Costos

```yaml
# 1. Cache agresivo de dependencias
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: ${{ runner.os }}-node-

# 2. Path filtering: solo ejecutar CI si cambia código fuente
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'

# 3. Usar Linux (más barato) para la mayoría de jobs
runs-on: ubuntu-latest

# 4. Limitar tiempo máximo de ejecución
timeout-minutes: 30

# 5. Auto-cancelar jobs redundantes
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# 6. Self-hosted runners para cargas pesadas
# runs-on: self-hosted  # Tu propia VM, sin costo de minutos
```

## 16.17 GitOps con ArgoCD / Flux (Ejemplo Concreto)

GitOps extiende CI/CD haciendo que la configuración de infraestructura viva en Git y se sincronice automáticamente:

```
[Dev hace push] → [GitHub Actions CI: build + push imagen] → [Actualizar repo de config]
                                                                    │
                                                                    ▼
                                                           [ArgoCD detecta cambio]
                                                                    │
                                                                    ▼
                                                           [Sync → Cluster K8s]
```

### ArgoCD Application

```yaml
# argocd/app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mi-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/empresa/config-repo.git
    targetRevision: main
    path: overlays/production
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

### Flux Kustomization

```yaml
# flux/kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: mi-app
  namespace: flux-system
spec:
  interval: 1m
  path: ./overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: config-repo
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: mi-app
      namespace: production
```

---

La integracion y entrega continua transforman Git de un simple VCS al nucleo de la automatizacion de software. Los pipelines desencadenados por eventos de Git (push, PR, tag) ejecutan builds, tests y deploys automaticamente. GitHub Actions, GitLab CI/CD, CircleCI, Bitbucket Pipelines y Jenkins ofrecen diferentes enfoques para definir pipelines como codigo. Merge Queue (GitHub) y Merge Trains (GitLab) serializan merges evitando conflictos en main. Semantic Release automatiza el versionado basado en conventional commits. Dependabot y Renovate mantienen dependencias actualizadas automaticamente. El security scanning (SAST, secretos, contenedores) debe integrarse en el pipeline. Las estrategias de deploy como blue-green (ALB) y canary (Istio) reducen el riesgo con YAML concreto. GitOps con ArgoCD/Flux sincroniza infraestructura desde Git. La optimizacion de costos (cache, path filtering, Linux runners) es critica en equipos grandes. La clave es mantener los pipelines simples, rapidos y seguros.

---

## Ejercicios Propuestos

1. **Pipeline basico en GitHub Actions**: Crea un workflow que se ejecute en cada push a `main` y ejecute `npm ci`, `npm test`, y `npm run build`. Agrega un badge de estado en el README.

2. **Pipeline multi-entorno**: Extiende el pipeline anterior para que despliegue automaticamente a staging cuando se hace push a `develop`, y requiera aprobacion manual para desplegar a produccion desde un tag `v*`.

3. **Matrix build**: Configura un workflow que ejecute tests en Node.js 18, 20 y 22, en sistemas operativos ubuntu y macos. Usa `exclude` para evitar combinaciones no deseadas.

4. **Semantic Release**: Configura semantic-release en un repositorio de prueba con conventional commits. Realiza commits de tipo `feat`, `fix` y `BREAKING CHANGE` y verifica que las versiones y el changelog se generen correctamente.

5. **Docker + CI/CD**: Crea un workflow que construya una imagen Docker en cada tag semantico (`v*`), la publique en Docker Hub o GHCR, y despliegue un deployment de Kubernetes usando una estrategia rolling update.

---

---

← [Capítulo anterior](15-workflows.md) | [Inicio](README.md) | [Capítulo siguiente →](17-avanzado.md)
