# Capítulo 15: Flujos de Trabajo

Git es extraordinariamente flexible en cuanto a cómo los equipos organizan su trabajo. Esa flexibilidad es una virtud, pero también puede ser una maldición: sin un flujo de trabajo claro, el repositorio degenera en caos. Este capítulo presenta los principales workflows de Git, sus ventajas, desventajas y criterios para elegir el más adecuado.

## 15.1 ¿Qué es un Workflow de Git y Por Qué Importa?

Un flujo de trabajo (workflow) es un conjunto de convenciones que define:

- Qué ramas existen y con qué propósito
- Cómo se crean y fusionan ramas
- Cuándo y cómo se despliega código a producción
- Cómo se gestionan releases, hotfixes y features
- Quién revisa qué y bajo qué condiciones

```
Proyecto sin workflow definido:
  main ────┬─── feat1 ────┬─── fix-urgente ─── ???
           │               │
           ├─── experimento
           │
           ├─── feature/login (abandonada hace 6 meses)
           │
           └─── produccion (¿es la misma que release/2.1?)
```

Sin convenciones, los equipos sufren de:
- **Deuda de ramas:** Ramas abandonadas que nadie limpia.
- **Regresiones:** Código roto en producción porque el deploy fue improvisado.
- **Bloqueos:** Una rama de release congela a todo el equipo por semanas.
- **Conflictos masivos:** Ramas de larga vida que divergen irremediablemente.

## 15.2 Git Flow (Vincent Driessen, 2010)

Git Flow es el workflow más conocido y estructurado. Publicado por Vincent Driessen en 2010, define un modelo de ramificación estricto con roles específicos para cada rama.

### 15.2.1 Ramas Principales

```
main (o master)
  └── Contiene solo código en producción
  └── Cada commit en main es un release
  └── Se etiqueta con número de versión

develop
  └── Rama de integración para features
  └── Contiene el código para el próximo release
  └── De aquí nacen las ramas de feature
```

### 15.2.2 Ramas de Soporte

| Tipo de Rama   | Nomenclatura          | Nace de   | Fusiona en       | Ciclo de Vida           |
| -------------- | --------------------- | --------- | ---------------- | ----------------------- |
| Feature        | `feature/<nombre>`    | develop   | develop          | Días a semanas          |
| Release        | `release/<version>`   | develop   | main + develop   | Días (preparación)      |
| Hotfix         | `hotfix/<version>`    | main      | main + develop   | Horas (urgente)         |

### 15.2.3 Ciclo Completo con Ejemplo

```
Fase 1: Desarrollo de Features

develop ──o───o───o────────────────────o───
              │                        │
              ├── feature/login ───────┘
              │       o───o───o
              │
              └── feature/pagos ───────┐
                      o───o───o        │
                                       │
Fase 2: Preparación de Release        │
                                       │
develop ──────────────────o───────────┼───o───
                          │           │   │
                          └── release/1.0 ──┘───┐
                              o───o───o (QA)    │
                                                │
Fase 3: Release a Producción                   │
                                                │
main ─────────────────────────────o─────────────┘ (merge de release/1.0)
                                  │
                                  ● (tag v1.0)

Fase 4: Hotfix Urgente

main ───● v1.0 ──────────────────────────o───● v1.0.1
              │                           │
              └── hotfix/1.0.1 ───────────┘───┐
                      o                       │
                                              │
develop ──────────────────────────────────────┘ (merge del hotfix)
```

### 15.2.4 Ejemplo de Comandos

```bash
# Iniciar una feature
git checkout develop
git checkout -b feature/autenticacion-oauth
# ... desarrollar, commits ...
git checkout develop
git merge --no-ff feature/autenticacion-oauth
git branch -d feature/autenticacion-oauth

# Iniciar un release
git checkout develop
git checkout -b release/1.2.0
# ... pruebas, ajustes de versión ...
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release 1.2.0"
git checkout develop
git merge --no-ff release/1.2.0
git branch -d release/1.2.0

# Hotfix urgente
git checkout main
git checkout -b hotfix/1.2.1
# ... corrección ...
git checkout main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git checkout develop
git merge --no-ff hotfix/1.2.1
git branch -d hotfix/1.2.1
```

### 15.2.5 Extensión `git-flow` (Herramienta CLI)

> **Nota importante sobre mantenimiento:** La herramienta CLI `git-flow` (de nvie) está **esencialmente abandonada**. Su último release estable data de ~2013 y no ha recibido actualizaciones significativas desde entonces. Aunque funcional para flujos básicos, carece de soporte para features modernas de Git. Para equipos que aún eligen Git Flow, se recomienda implementar los comandos manualmente (como se muestra en 15.2.4) o usar wrappers mantenidos como `git-flow-avh` (AVH Edition), que sigue recibiendo actualizaciones.

```bash
# Instalación (si decides usarla de todas formas)
brew install git-flow         # macOS
sudo apt install git-flow     # Ubuntu/Debian

# Inicializar Git Flow en un repositorio
git flow init
# Responde: nombre de main, develop, prefijos de feature/release/hotfix

# Iniciar una feature
git flow feature start integracion-pasarela-pago

# Finalizar una feature (merge a develop y borra la rama)
git flow feature finish integracion-pasarela-pago

# Iniciar release
git flow release start 2.0.0

# Finalizar release (merge a main + develop, tag, borra rama)
git flow release finish 2.0.0

# Hotfix
git flow hotfix start 2.0.1
git flow hotfix finish 2.0.1
```

### 15.2.6 Pros y Contras

| Pros                                          | Contras                                       |
| --------------------------------------------- | --------------------------------------------- |
| Estructura clara y predecible                 | Complejidad innecesaria para equipos pequeños |
| Ideal para software con versiones (SaaS, App) | `develop` puede divergir mucho de `main`      |
| Facilita releases planificadas                | Retrasa la entrega continua                   |
| Separación limpia de concerns                 | Sobrecarga de ramas y merges                  |
| Muy documentado, gran comunidad               | Poco adecuado para web apps con deploy continuo|

> **Recomendación:** Usa Git Flow si tu producto tiene versiones planificadas (v1.0, v1.1, v2.0), requieres QA formal antes de producción, y despliegas en ciclos semanales o quincenales.

## 15.3 GitHub Flow (Scott Chacon, 2011)

GitHub Flow simplifica radicalmente el modelo. Scott Chacon lo publicó en 2011 argumentando que Git Flow era excesivamente complejo para la mayoría de equipos.

### 15.3.1 Principios

1. **Una sola rama principal:** `main` siempre está lista para producción.
2. **Ramas de feature desde `main`:** Se crean para cada cambio, con nombres descriptivos.
3. **Pull Requests para todo:** Incluso para cambios de una línea.
4. **Deploy inmediato desde `main`:** Cada merge a `main` va a producción (CI/CD).
5. **Merge solo después de revisión y CI verde.**

### 15.3.2 Ciclo de Trabajo

```
main ────o────────o───────────o────────o───────
         │        │           │        │
         │   feature/login    │   feature/api
         │   o──o──o──o──┘    │   o──o──┘
         │                    │
         │   Cada merge a main = potencial deploy a producción
         │
         ▼
      PRODUCCIÓN
```

### 15.3.3 Ejemplo

```bash
# Crear rama desde main
git checkout main
git pull origin main
git checkout -b feature/mejora-rendimiento

# Desarrollar la feature
echo "Nueva caché" >> cache.py
git add cache.py
git commit -m "Añadir caché Redis para consultas frecuentes"

# Empujar y crear Pull Request
git push -u origin feature/mejora-rendimiento
# Abrir PR en GitHub

# Después de revisión y CI verde, mergear
# (Desde la UI de GitHub o el CLI)

# Eliminar rama remota y local
git push origin --delete feature/mejora-rendimiento
git branch -d feature/mejora-rendimiento
```

### 15.3.4 Pros y Contras

| Pros                                          | Contras                                       |
| --------------------------------------------- | --------------------------------------------- |
| Simplicidad máxima                            | No soporta versiones simultáneas              |
| Entrega continua natural                      | Difícil para productos con múltiples entornos |
| Ideal para equipos pequeños y medianos        | Hotfix urgente = revert en main               |
| Sin sobrecarga de ramas                       | Sin distinción entre staging y producción     |
| Integración perfecta con CI/CD                | Requiere disciplina en tests automatizados    |

> **Recomendación:** Usa GitHub Flow si despliegas continuamente (varias veces al día), tu producto es una web app o API que siempre está en una sola versión, y tienes CI/CD robusto con tests automatizados.

### 15.3.5 GitHub Flow + Releases: Gestión de Lanzamientos sin Release Branches

Cuando el producto necesita releases versionadas sin ramas dedicadas:

```bash
# Estrategia: tags en main + GitHub Releases
# 1. El código siempre se mergea a main con PR normal
# 2. Cuando quieres hacer un release, creas un tag en el commit deseado
git tag -a v1.2.0 -m "Release 1.2.0: nuevo motor de búsqueda"
git push origin v1.2.0

# 3. El tag dispara CI/CD que:
#    - Genera changelog automático
#    - Construye artefactos (binarios, docker, npm)
#    - Crea GitHub Release con release notes

# 4. Si necesitas hotfix en una versión antigua:
git checkout v1.1.0
git checkout -b hotfix/1.1.1
# ... fix ...
git tag -a v1.1.1 -m "Hotfix 1.1.1"
git push origin v1.1.1
# La rama hotfix se elimina tras el tag, no se mantiene
```

Este enfoque evita ramas de release longevas: cada release es un tag. Los hotfixes se hacen desde el tag correspondiente y se mergean a main también.

## 15.4 GitLab Flow

GitLab Flow extiende GitHub Flow con ramas de entorno para resolver el problema de múltiples ambientes (staging, pre-production, production).

### 15.4.1 Ramas de Entorno

```
main ──o────o────o────o────o────o────o────
       │    │    │    │    │    │    │
       │    │    │    │    │    │    └────> merge a pre-production
       │    │    │    │    │    │
       │    │    │    │    └──────────────> merge a pre-production
       │    │    │    │
       └────┼────┼────┼───────────────────> merge a production (release)
            │    │    │
            │    │    └────────────────────> merge a production
            │    │
staging ────┼────┼─────────────────────────> staging se actualiza continuamente
            │    │
pre-production ──┼─────────────────────────> pre-production para pruebas finales
                 │
production ────────────────────────────────> solo se actualiza con releases aprobados
```

### 15.4.2 Feature Branches + Merge Requests

Igual que GitHub Flow pero con Merge Requests (MR) en lugar de Pull Requests:

```bash
git checkout -b feature/nueva-api main
# ... desarrollo ...
git push -u origin feature/nueva-api
# Crear Merge Request en GitLab
# Esperar CI + revisión + aprobación
# Merge a main
```

### 15.4.3 Release Branches con GitLab Flow

Cuando necesitas mantener múltiples versiones en producción:

```bash
# Crear rama de release
git checkout -b release-2.x main

# El release-2.x recibe backports de fixes críticos
git checkout release-2.x
git cherry-pick <commit-del-fix>  # Fix desde main

# main sigue avanzando con nuevas features
```

### 15.4.4 Pros y Contras

| Pros                                            | Contras                                  |
| ----------------------------------------------- | ---------------------------------------- |
| Soporte para múltiples entornos                 | Más complejo que GitHub Flow             |
| Flexible: desde simple hasta complejo           | Release branches pueden acumular deuda   |
| Buena integración con GitLab CI/CD              | Requiere disciplina en promoción de código|

## 15.5 Trunk-Based Development (TBD)

Trunk-Based Development es el workflow utilizado por Google, Facebook y la mayoría de las empresas de élite según el informe State of DevOps de DORA.

### 15.5.1 Principios Fundamentales

1. **Una sola rama principal:** `main` (trunk) es la única fuente de verdad.
2. **Ramas de vida extremadamente corta:** Menos de 24 horas. Idealmente, menos de 2 horas.
3. **Integración continua a trunk:** Múltiples merges al día por desarrollador.
4. **Feature flags para código incompleto:** En lugar de ramas de feature largas.
5. **Branch by Abstraction:** Refactorizaciones grandes sin romper trunk.

### 15.5.2 Feature Flags

```python
# En lugar de mantener código en una rama durante semanas,
# se mergea a main detrás de un feature flag:

if feature_flag_enabled("nuevo_motor_busqueda"):
    resultado = motor_v2.buscar(query)
else:
    resultado = motor_v1.buscar(query)

# El flag se activa gradualmente en producción
# Cuando la feature está validada:
#   1. Se elimina el flag
#   2. Se elimina el código viejo
```

### 15.5.3 Branch by Abstraction

Para refactorizaciones grandes que no pueden completarse en horas:

```
Paso 1: Introducir abstracción (merge a main con el código viejo intacto)
        └── Interfaz IPaymentProcessor

Paso 2: Implementar nueva versión detrás de la abstracción
        └── NuevoPaymentProcessor implementa IPaymentProcessor

Paso 3: Migrar consumidores uno a uno (cada cambio va a main)
        └── Servicio A migra, Servicio B migra...

Paso 4: Eliminar código viejo (cuando nadie lo usa)
        └── Eliminar ViejoPaymentProcessor
```

### 15.5.4 Ciclo de Trabajo Diario

```bash
# Mañana: sincronizar con trunk
git checkout main
git pull --rebase origin main

# Crear rama de corta vida
git checkout -b fix/typo-readme

# Hacer el cambio y commit
echo "Corrección" >> README.md
git add README.md
git commit -m "Fix typo in README installation section"

# Empujar al instante
git push -u origin fix/typo-readme

# Abrir PR y mergear en < 1 hora

# Volver a main y borrar rama
git checkout main
git pull --rebase origin main
git branch -d fix/typo-readme

# Tarde: repetir el ciclo con la siguiente tarea
```

### 15.5.5 Pros y Contras

| Pros                                               | Contras                                       |
| -------------------------------------------------- | --------------------------------------------- |
| Máxima velocidad de entrega                        | Requiere mucha disciplina en tests            |
| Conflictos mínimos (ramas de horas, no semanas)    | Feature flags añaden complejidad temporal     |
| Code review más rápido (commits pequeños)          | Branch by Abstraction requiere diseño previo  |
| Alineado con élite DevOps (DORA)                   | Curva de aprendizaje para equipos novatos     |
| Sin infierno de merges de release                  | Inversión inicial en CI/CD y testing          |

> **Recomendación:** Trunk-Based Development es el estándar de la industria para equipos de alto rendimiento. Si tu equipo puede hacer deploys diarios y tiene cultura de testing, TBD te llevará a otro nivel de productividad.

## 15.6 Ship/Show/Ask (GitHub Internal Workflow)

GitHub desarrolló internamente el modelo **Ship/Show/Ask** para escalar la toma de decisiones en PRs. Clasifica cada cambio en una de tres categorías:

| Categoría | Significado | Merge sin revisión |
|-----------|-------------|---------------------|
| **Ship** | Cambios triviales, fixes obvios, typos, docs | Sí, inmediato |
| **Show** | Cambios que otros deben ver pero no necesitan aprobación | Sí, después de CI verde |
| **Ask** | Cambios que requieren revisión y aprobación explícita | No, requiere review |

```bash
# Convención en el prefijo del título del PR:
# [Ship] fix: corregir typo en README
# [Show] refactor(auth): simplificar lógica de tokens
# [Ask] feat(api): nuevo endpoint de pagos

# Los CODEOWNERS se configuran para que [Ask] requiera aprobación obligatoria
```

Este modelo reduce la fricción de revisión: no todo cambio necesita el mismo nivel de escrutinio. Especialmente útil en equipos de 10+ personas donde los PRs pequeños pueden auto-mergearse.

## 15.7 Stacked PRs / Stacked Diffs

El patrón de **stacked PRs** (popularizado por Uber, Meta y Phabricator) permite trabajar en múltiples cambios dependientes sin ramas de feature gigantes:

```
main
  └── feature/parte-1 (PR #1): Refactorizar interfaz del parser
        └── feature/parte-2 (PR #2): Implementar nuevo parser
              └── feature/parte-3 (PR #3): Migrar consumidores
```

Cada PR se abre contra el anterior (no contra main), creando una pila. Las ventajas:

- **Revisiones más rápidas:** Cada PR es pequeño y enfocado.
- **Sin ramas longevas:** Cada parte se mergea cuando está lista.
- **Paralelismo real:** Varios desarrolladores pueden trabajar en distintas partes de la pila.

```bash
# Implementación con --update-refs (Git 2.38+)
# Durante un rebase interactivo, --update-refs mantiene las ramas
# apuntando a los commits correctos tras reordenar/squash

git rebase -i --update-refs HEAD~5
# Si reorganizas commits en la pila, Git actualiza automáticamente
# las referencias de las ramas que apuntaban a esos commits

# Herramientas que facilitan stacked PRs:
# - ghstack (Python): https://github.com/ezyang/ghstack
# - graphite.dev: Plataforma web + CLI para stacked diffs
# - spr (Go): https://github.com/ejoffe/spr
```

## 15.8 Conventional Commits + Semantic Release

Conventional Commits es una convención de mensajes de commit que, combinada con Semantic Release, automatiza versionado y changelogs.

### 15.6.1 Conventional Commits

```
<tipo>[ámbito opcional]: <descripción>

[CUERPO opcional]

[PIE(S) opcional(es)]
```

Tipos estándar:
- `feat:` nueva funcionalidad (MINOR version bump)
- `fix:` corrección de bug (PATCH version bump)
- `BREAKING CHANGE:` cambio incompatible (MAJOR version bump)
- `docs:` documentación
- `style:` formato, punto y coma, etc.
- `refactor:` refactorización sin cambio funcional
- `perf:` mejora de rendimiento
- `test:` añadir o corregir tests
- `chore:` tareas de mantenimiento, dependencias
- `ci:` cambios en CI/CD

### 15.6.2 Ejemplos

```bash
git commit -m "feat(auth): añadir inicio de sesión con Google OAuth"

git commit -m "fix(payments): corregir redondeo de decimales en cálculo de IVA"

git commit -m "feat(api)!: eliminar endpoint v1 deprecado

BREAKING CHANGE: El endpoint /api/v1/users ha sido eliminado.
Migrar a /api/v2/users."

git commit -m "docs(readme): actualizar instrucciones de instalación para macOS"
```

### 15.6.3 Semantic Release Automatizado

Con herramientas como `semantic-release` (Node.js) o `semantic-release` para otros lenguajes:

```
Flujo automatizado:
  1. Developer hace commit siguiendo Conventional Commits
  2. PR se mergea a main
  3. CI analiza los commits desde el último release
  4. Determina la nueva versión (MAJOR, MINOR, PATCH)
  5. Genera CHANGELOG automáticamente
  6. Crea tag de versión
  7. Publica release en GitHub/GitLab
  8. Publica paquete (npm, PyPI, Docker, etc.)
```

### 15.6.4 Configuración en CI (GitHub Actions)

```yaml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 15.9 CODEOWNERS: Dueños de Código por Directorio

El archivo `CODEOWNERS` en `.github/` (GitHub) o `.gitlab/` (GitLab) define quién debe revisar cambios en cada parte del código.

### 15.9.1 Sintaxis (GitHub)

```
# .github/CODEOWNERS

# Dueños globales (revisan todo)
*                         @equipo/arquitectos

# Dueños por directorio
/services/auth/           @equipo/autenticacion
/services/payments/       @equipo/pagos
/frontend/                @equipo/frontend

# Dueños por tipo de archivo
*.tf                      @equipo/infraestructura
*.sql                     @equipo/dba
docs/*.md                 @equipo/documentacion

# Múltiples dueños (todos deben aprobar)
/database/migrations/     @equipo/backend @equipo/dba

# Dueño individual
/config/deploy/           @tech-lead
```

### 15.9.2 Sintaxis (GitLab)

En GitLab, el archivo se ubica en `.gitlab/CODEOWNERS` (GitLab 15.9+). La sintaxis es similar pero con algunas diferencias:

```
# .gitlab/CODEOWNERS
# Dueños globales
*                         @arquitectos

# Directorios
/services/auth/           @equipo-auth
/services/payments/       @equipo-pagos @equipo-backend

# GitLab soporta patrones más granulares:
[Documentation] docs/*.md @equipo-docs
[Infrastructure] *.tf     @equipo-devops
```

### 15.9.3 Precedencia de Reglas y Wildcards

Las reglas de CODEOWNERS siguen estas reglas de precedencia:

1. **Regla más específica gana:** `/services/auth/login.js` toma `.js` sobre `/services/auth/` sobre `*`.
2. **Último match gana:** Si dos reglas empatan en especificidad, la última definida es la que vale (GitHub) o la primera (GitLab — verificar documentación actualizada).
3. **Wildcards:**
   - `*` = cualquier archivo en ese nivel (no recursivo)
   - `**` = recursivo en subdirectorios
   - `?` = un carácter cualquiera
   - `[abc]` = rango de caracteres

### 15.9.4 Peligro Común: Demasiados Pocos Owners

**Antipatrón:** Asignar un solo equipo/dueño para todo el repositorio:

```
* @tech-lead  # Tech Lead se convierte en cuello de botella de todos los PRs
```

**Solución:** Distribuir ownership granularmente y mantener backups (al menos 2 personas por path crítico). Si un owner se va de vacaciones, los PRs se bloquean.

```
/services/auth/  @equipo-auth @equipo-backend  # Al menos 2 equipos
```

## 15.10 Protección de Ramas

Configurar reglas de protección protege las ramas críticas (especialmente `main`) de cambios accidentales o no revisados.

### 15.10.1 Configuraciones de Protección (GitHub/GitLab/Bitbucket)

| Regla                     | GitHub                                      | GitLab                                 | Bitbucket                            |
| ------------------------- | ------------------------------------------- | -------------------------------------- | ------------------------------------ |
| **Require PR/MR**         | Branch protection → Require a PR            | Protected branch → Allowed to merge    | Branch restrictions → Prevent pushes |
| **Require approvals**     | Require N approvals                         | Code owner approval (Free)             | Default reviewers + Prevent merges  |
| **Require status checks** | Require status checks to pass               | Pipelines must succeed                 | Minimum successful builds            |
| **Require linear history**| Require linear history (squash/rebase only) | Merge method: FF-only or Squash        | Merge checks → Rebase/Squash only    |
| **Require conversation resolution** | Require conversations resolved | MR approvals → All threads resolved    | Not available                        |
| **Require up-to-date**    | Require branches up to date                 | Pipelines must succeed                 | Not available explicitly             |
| **Restrict push**         | Restrict who can push                       | Allowed to push (No one by default)    | Branch permissions → Write access    |
| **Require signed commits**| Require signed commits (Enterprise only)    | Push rules → Reject unsigned (Premium) | Not available                        |

### 15.10.2 Ejemplos por Plataforma

**GitHub:**
```yaml
# Configuración conceptual de protección de rama
# (Se configura en la UI de GitHub, no en archivo)

main:
  require_pull_request: true
  required_approving_review_count: 2
  require_code_owner_reviews: true
  required_status_checks:
    - "lint"
    - "test (ubuntu)"
    - "test (macos)"
    - "build"
  require_linear_history: true
  allow_force_pushes: false
  allow_deletions: false
  restrictions:
    users: []
    teams: ["admins"]
```

**GitLab:**
```
# Settings > Repository > Protected Branches
main:
  Allowed to merge: Maintainers + Developers
  Allowed to push: No one
  Code owner approval: Required (2 approvals)
  Push rules (Premium): Reject unsigned commits, Check author email
```

**Bitbucket:**
```
# Repository Settings > Branch Permissions
Branch pattern: main
  Prevent deletion: Yes
  Prevent rewriting history: Yes
  Default reviewers: @team-backend (2 reviewers required)
  Minimum successful builds: 2
```

## 15.11 Elegir el Workflow Adecuado

No existe un workflow universalmente correcto. La elección depende de:

### Matriz de Decisión

| Factor                         | Git Flow        | GitHub Flow     | GitLab Flow     | Trunk-Based Dev |
| ------------------------------ | --------------- | --------------- | --------------- | --------------- |
| Tamaño del equipo              | 5-50+           | 2-20            | 5-50+           | 2-100+          |
| Frecuencia de deploy           | Semanal/Quincenal | Continuo (diario)| Continuo/Semanal| Continuo (horas)|
| Múltiples versiones soportadas | Sí (ideal)      | Rara vez        | Sí              | Vía feature flags|
| Madurez de CI/CD               | Básico          | Intermedio      | Intermedio/Alto | Alto            |
| Testing automatizado           | Parcial         | Bueno           | Bueno           | Excelente       |
| Tipo de producto               | App, SaaS, On-prem | Web, API, SaaS | Web, SaaS       | Web, API, SaaS  |
| Cultura DevOps                 | Tradicional     | Ágil            | Ágil/DevOps     | DevOps/Élite    |

### Árbol de Decisión

```
¿Tu producto tiene múltiples versiones en producción?
  ├── Sí ──> ¿Despliegas continuamente?
  │           ├── Sí ──> Trunk-Based Dev + feature flags
  │           └── No  ──> Git Flow o GitLab Flow con release branches
  │
  └── No ──> ¿Equipo < 5 personas?
              ├── Sí ──> GitHub Flow (máxima simplicidad)
              └── No  ──> Trunk-Based Dev o GitLab Flow
```

### Reglas de Oro Independientes del Workflow

1. **Nunca empujes directamente a `main`/`master`.** Siempre vía Pull Request.
2. **Mantén las ramas con vida corta.** Menos de 2 días idealmente, menos de 1 semana máximo.
3. **Escribe mensajes de commit significativos.** Son documentación viva del proyecto.
4. **Limpia las ramas mergeadas.** Si ya fue fusionada, bórrala.
5. **Los tests pasan en local antes del push.** No externalices la verificación al CI.
6. **El código en `main` siempre debe ser desplegable.**

## 15.12 Escalar Equipos: Código Compartido, Comunicación y Sincronización

A medida que el equipo crece, el workflow debe escalar en tres dimensiones:

### 15.10.1 Código Compartido

```
Equipo de 5 personas:
  Un solo repositorio, comunicación directa

Equipo de 20 personas:
  CODEOWNERS por módulo, revisión cruzada

Equipo de 100+ personas:
  Monorepo con propiedad clara por directorio
  CODEOWNERS obligatorios
  Arquitectos revisan cambios en interfaces compartidas
  Feature flags para coordinación entre equipos
```

### 15.10.2 Comunicación

- **Daily standup de 15 minutos** enfocado en bloqueos y dependencias entre equipos.
- **RFC (Request for Comments)** para cambios arquitectónicos que afectan a múltiples equipos.
- **Canales por módulo** (Slack/Discord/Teams) donde los CODEOWNERS responden dudas.
- **Sprint planning con visibilidad cruzada** para anticipar conflictos.

### 15.10.3 Sincronización Técnica

```bash
# Sincronización frecuente con main
# Cada mañana, todos los desarrolladores:
git checkout main
git pull --rebase origin main

# Sincronización de ramas de feature
git checkout feature/mi-tarea
git rebase main  # o git merge main si el equipo prefiere merges

# Detección temprana de conflictos
# Usar git fetch + git diff para anticipar problemas:
git fetch origin
git diff HEAD origin/main -- <mis-archivos>
```

### 15.10.4 Protocolo de Escalamiento para Conflictos entre Equipos

```
Nivel 1: Desarrolladores de cada equipo resuelven el conflicto en el PR
         └── Comunicación directa en el PR o chat

Nivel 2: Si no hay acuerdo en 24h, los Tech Leads de ambos equipos deciden
         └── Criterio: menor riesgo para producción

Nivel 3: Si el conflicto implica decisiones arquitectónicas, RFC formal
         └── Documento con opciones, pros/cons, decisión y justificación

Nivel 4: Staff Engineer / Arquitecto Principal toma la decisión final
         └── Basada en RFC y principios de arquitectura del proyecto
```

---

## 15.13 DORA Metrics para Equipos Git

Las métricas DORA (DevOps Research and Assessment) miden el rendimiento de los equipos de software. Integrarlas con Git permite evaluar la efectividad de tu workflow:

| Métrica DORA | Qué mide | Cómo medirla desde Git | Élite | Medio | Bajo |
|-------------|----------|----------------------|-------|-------|------|
| **Deployment Frequency** | Frecuencia de deploys a producción | Tags/Releases por semana | On-demand (diario) | Semanal | Mensual |
| **Lead Time for Changes** | Tiempo desde commit hasta deploy | `git log v1.1.0...v1.2.0 --format=%aI` (diferencia commit→tag) | < 1 hora | 1 día-1 semana | > 6 meses |
| **Change Failure Rate** | % de deploys que requieren hotfix/revert | Commits de revert o hotfix / total deploys | < 15% | 15-30% | > 45% |
| **Time to Restore Service** | Tiempo en recuperar tras incidente | Tiempo entre commit de hotfix y tag de release que lo contiene | < 1 hora | < 1 día | > 6 meses |

```bash
# Script para calcular Lead Time desde Git:
LAST_TAG=$(git describe --tags --abbrev=0)
FIRST_COMMIT=$(git rev-list --max-parents=0 HEAD)

# Tiempo total del ciclo
TOTAL_DAYS=$(( ($(date -d "$(git log -1 --format=%aI $LAST_TAG)" +%s) -
                 $(date -d "$(git log -1 --format=%aI $FIRST_COMMIT)" +%s)) / 86400 ))
echo "Lead Time: $TOTAL_DAYS días desde primer commit hasta $LAST_TAG"

# Deployment Frequency:
RELEASES_LAST_MONTH=$(git tag --sort=-creatordate --list 'v*' --format='%(creatordate:short)' | \
  awk -v cutoff="$(date -d '30 days ago' +%Y-%m-%d)" '$1 >= cutoff' | wc -l)
echo "Deploys (últimos 30 días): $RELEASES_LAST_MONTH"
```

> **Objetivo:** Si adoptas Trunk-Based Development + CI/CD robusto, apunta a métricas de nivel Élite: deploys diarios, lead time < 1 hora, y failure rate < 15%.

---

## Resumen del Capítulo 15

Los flujos de trabajo en Git definen cómo los equipos colaboran y entregan software. Los puntos clave son:

1. **Git Flow** es ideal para productos con versiones y releases planificadas. El CLI `git-flow` original está abandonado (~2013); usa comandos manuales o `git-flow-avh`.
2. **GitHub Flow** simplifica al máximo: `main` siempre lista para producción, ramas de feature cortas, PRs para todo. Las releases se gestionan con tags, no con ramas dedicadas.
3. **GitLab Flow** extiende GitHub Flow con ramas de entorno y release branches, ofreciendo un punto intermedio flexible.
4. **Ship/Show/Ask** (GitHub internal) clasifica PRs para reducir fricción de revisión: no todo cambio necesita aprobación.
5. **Stacked PRs / Stacked Diffs** (Uber/Meta) dividen features grandes en PRs apilables para revisiones rápidas y paralelismo.
6. **Trunk-Based Development** es el estándar de equipos de élite: ramas de horas, feature flags, branch by abstraction. Velocidad máxima con disciplina.
7. **Conventional Commits + Semantic Release** automatizan versionado y changelogs, eliminando trabajo manual y errores humanos.
8. **CODEOWNERS** y **protección de ramas** (GitHub, GitLab, Bitbucket) son mecanismos de gobernanza que escalan con el equipo.
9. **DORA Metrics** (Deployment Frequency, Lead Time, CFR, MTTR) miden la efectividad del workflow y guían la mejora continua.

## Ejercicios Propuestos

### Ejercicio 1: Implementación de Git Flow

1. En un repositorio nuevo, configura Git Flow usando `git flow init` con los valores predeterminados.
2. Simula el ciclo completo:
   - Crea una feature `login` con `git flow feature start login`
   - Haz 3 commits en la feature
   - Finaliza la feature con `git flow feature finish login`
   - Crea un release `1.0.0` con `git flow release start 1.0.0`
   - Finaliza el release y verifica que main recibe el tag `v1.0.0`
3. Dibuja el grafo de commits resultante (`git log --oneline --graph --all`) y compáralo con el diagrama ASCII de la sección 15.2.3.

### Ejercicio 2: Simulación de Trunk-Based Development

1. Crea un repositorio con un servidor web simple (puede ser un `index.html` o un script en Node.js/Python).
2. Simula un día de trabajo con TBD: crea 4 ramas de feature de corta vida, haz un commit en cada una, mergea vía PR simulado (no hace falta GitHub real).
3. Establece la regla: ninguna rama debe existir por más de 2 commits de actividad en main.
4. Al final del día, verifica que main tiene un historial lineal (o casi lineal) y que todas las ramas han sido eliminadas.

### Ejercicio 3: Conventional Commits

1. Toma los últimos 20 commits de un proyecto personal o crea un repositorio de práctica.
2. Reescribe los mensajes de commit para que sigan el estándar Conventional Commits (`feat:`, `fix:`, `refactor:`, etc.).
3. Usa una herramienta como `commitlint` para validar que los mensajes cumplen el estándar.
4. Simula un release con `standard-version` (Node.js) o una herramienta equivalente para tu lenguaje.
5. Analiza el CHANGELOG generado y explica cómo se determinó la nueva versión.

### Ejercicio 4: Configuración de CODEOWNERS y Protección de Ramas

1. Investiga la documentación de CODEOWNERS para GitHub o GitLab.
2. Para un proyecto hipotético con estructura:
   ```
   /api/
   /frontend/
   /infra/terraform/
   /docs/
   ```
3. Escribe un archivo `CODEOWNERS` que asigne:
   - `@equipo/backend` para `/api/`
   - `@equipo/frontend` para `/frontend/`
   - `@equipo/devops` para `/infra/terraform/`
   - `@equipo/backend` y `@equipo/frontend` juntos para cambios en interfaces compartidas
4. Define las reglas de protección de rama que aplicarías para `main` y justifica cada una.

### Ejercicio 5: Elección y Justificación de Workflow

1. Describe tres escenarios hipotéticos de equipos diferentes (usa la tabla de la sección 15.9 como guía):
   - Startup de 3 personas haciendo una app web.
   - Empresa mediana (20 devs) con producto SaaS y releases mensuales.
   - Gran corporación (100+ devs) con monorepo y deploys continuos.
2. Para cada escenario, elige el workflow más apropiado y escribe una justificación de al menos 5 líneas.
3. Identifica qué métricas usarías para evaluar si el workflow está funcionando (e.g., tiempo desde commit hasta deploy, frecuencia de conflictos, tiempo de revisión de PRs).
4. Para el escenario de la gran corporación, propón una estrategia de escalamiento de equipos usando CODEOWNERS, feature flags y protocolos de comunicación.

---

← [Capítulo anterior](14-gran-escala.md) | [Inicio](README.md) | [Capítulo siguiente →](16-ci-cd.md)
