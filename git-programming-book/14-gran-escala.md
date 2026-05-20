# Capítulo 14: Git a Gran Escala

Cuando un repositorio crece más allá de unos cientos de megabytes o cientos de miles de archivos, Git comienza a mostrar sus límites. Las operaciones cotidianas se vuelven lentas, los clones consumen minutos (u horas) y el espacio en disco se dispara. Este capítulo explora las estrategias, herramientas y configuraciones para que Git funcione eficientemente a gran escala.

## 14.1 Desafíos de los Repositorios Grandes

Antes de abordar soluciones, entendamos los problemas:

| Desafío                     | Síntoma Típico                                    | Causa Raíz                            |
| --------------------------- | ------------------------------------------------- | ------------------------------------- |
| Tamaño del repositorio      | Clone de 30+ minutos, `.git` de varios GB         | Archivos binarios grandes en historial|
| Velocidad de operaciones    | `git status` tarda 5+ segundos, `git log` lento   | Demasiados archivos en el working tree|
| CI/CD lento                 | Pipeline tarda 45+ minutos                         | Clone completo cada build             |
| Espacio en disco            | 50 GB por clon en cada máquina del equipo         | Historial completo de archivos binarios|
| Anchos de banda             | Saturación de red en clones masivos               | Transferencia de blobs innecesarios   |

```
Evolución típica de un repositorio problemático:
                                                            ┌─────────────┐
                                                            │ 50 GB+      │
                                                            │ CI lento    │
                                                            │ Equipo      │
                                                            │ frustrado   │
                                                            └─────────────┘
                                          ┌───────────┐           ▲
                                          │ 10 GB     │           │
                                          │ Clones    │           │
                                          │ lentos    │           │
                                          └───────────┘           │
                    ┌───────────┐              ▲                  │
                    │ 1 GB      │              │                  │
                    │ Empiezan  │              │                  │
                    │ molestias │              │                  │
                    └───────────┘              │                  │
  ┌───────────┐          ▲                    │                  │
  │ 100 MB    │          │                    │                  │
  │ Todo bien │──────────┼────────────────────┼──────────────────┘
  └───────────┘          │                    │
                    Mes 1              Mes 12              Mes 24
```

## 14.2 Git LFS (Large File Storage)

Git LFS resuelve el problema de archivos binarios grandes almacenándolos fuera del repositorio principal. En lugar de guardar el archivo completo, Git guarda un puntero (archivo de texto de ~130 bytes) que referencia al objeto real, el cual se almacena en un servidor LFS.

### 14.2.1 Concepto y Arquitectura

```
Sin Git LFS:
  .git/objects/  ────  archivo.psd  (200 MB de blob)

Con Git LFS:
  .git/objects/  ────  archivo.psd  (130 bytes, puntero texto)
  Servidor LFS   ────  archivo.psd  (200 MB, objeto real)
  Caché local    ────  .git/lfs/    (caché de objetos LFS)
```

El puntero LFS tiene este aspecto:

```
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daa...
size 209715200
```

### 14.2.2 Instalación y Configuración

```bash
# Instalar Git LFS (requiere Git ≥ 1.8.5)
# macOS
brew install git-lfs

# Ubuntu/Debian
sudo apt install git-lfs

# Windows (con Chocolatey)
choco install git-lfs

# Inicializar LFS en el repositorio
cd mi-repo
git lfs install
# Updated Git hooks.
# Git LFS initialized.

# Verificar instalación
git lfs version
# git-lfs/3.4.0
```

### 14.2.3 Rastreo de Patrones de Archivos

```bash
# Rastrear archivos por extensión
git lfs track "*.psd"
git lfs track "*.mp4"
git lfs track "*.zip"
git lfs track "*.tar.gz"

# Rastrear archivos en un directorio específico
git lfs track "assets/images/**"

# Rastrear un archivo concreto
git lfs track "database/dump.sql"

# Ver qué patrones están siendo rastreados
git lfs track
# Listing tracked patterns
#     *.psd (.gitattributes)
#     *.mp4 (.gitattributes)

# Dejar de rastrear un patrón
git lfs untrack "*.psd"
```

### 14.2.4 `.gitattributes` y LFS

`git lfs track` escribe en `.gitattributes`. Debes hacer commit de este archivo para que todos los colaboradores usen LFS:

```
# .gitattributes generado
*.psd filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
assets/images/** filter=lfs diff=lfs merge=lfs -text
```

### 14.2.5 Comandos Esenciales de LFS

```bash
# Ver qué archivos están gestionados por LFS
git lfs ls-files
# 3a2b1c4d5e * assets/banner.psd
# 6f7e8d9c0b * assets/hero.mp4

# Obtener los objetos LFS (los archivos reales)
git lfs fetch

# Descargar solo LFS de la rama actual
git lfs pull

# Diferencia: git lfs fetch vs git lfs pull
#   git lfs fetch     — Descarga todos los objetos LFS referenciados
#                         desde el remoto, pero NO actualiza el working tree
#   git lfs pull      — Equivale a git lfs fetch + checkout de archivos LFS
#                         actuales; actualiza el working tree inmediatamente
#   git lfs fetch --all — Descarga TODOS los objetos LFS de todas las ramas

# Migrar archivos existentes del historial a LFS
git lfs migrate import --include="*.psd,*.mp4" --everything

# Migrar excluyendo ciertas ramas
git lfs migrate import --include="*.iso" --everything --exclude-ref="refs/heads/legacy"

# Ver información de un archivo LFS
git lfs ls-files -l assets/banner.psd
# 3a2b1c4d5e * assets/banner.psd (2.4 MB)

# Hacer checkout selectivo de archivos LFS
git lfs checkout assets/banner.psd

# Gestionar caché local de LFS
git lfs prune --dry-run      # Ver qué se eliminaría
git lfs prune --verbose       # Eliminar objetos LFS locales no referenciados
# La caché LFS local puede crecer mucho; prune la limpia periódicamente
```

### 14.2.6 Bloqueo de Archivos LFS (File Locking)

Para evitar conflictos al editar archivos binarios (que no se pueden mergear), LFS soporta bloqueo de archivos:

```bash
# Bloquear un archivo para edición exclusiva
git lfs lock assets/banner.psd
# Locked assets/banner.psd

# Ver todos los archivos bloqueados
git lfs locks
# assets/banner.psd  Alice  ID:abc123  2026-05-19

# Desbloquear un archivo
git lfs unlock assets/banner.psd

# Forzar desbloqueo (si otro usuario lo bloqueó)
git lfs unlock --force assets/banner.psd

# Ver archivos bloqueables (según .gitattributes)
git lfs locks --verify
```

Para habilitar el bloqueo, añade el atributo `lockable` en `.gitattributes`:

```
# .gitattributes
*.psd filter=lfs diff=lfs merge=lfs -text lockable
*.blend filter=lfs diff=lfs merge=lfs -text lockable
*.unity filter=lfs diff=lfs merge=lfs -text lockable
```

> **Tip:** El bloqueo LFS es esencial para equipos de diseño y game dev donde varios colaboradores editan archivos binarios no mergeables.

### 14.2.7 Estrategia de Migración a LFS sin Romper CI/CD

Migrar un repositorio existente a LFS requiere planificación para no interrumpir los pipelines:

```bash
# Fase 1: Preparación local (en rama dedicada)
git checkout -b lfs-migration
git lfs install
git lfs track "*.psd" "*.mp4" "*.zip"
git add .gitattributes
git commit -m "chore: configurar Git LFS"

# Fase 2: Migrar historial existente
git lfs migrate import --include="*.psd,*.mp4,*.zip" --everything
# Esto reescribe TODO el historial: los blobs grandes se convierten en punteros

# Fase 3: Verificar antes del push
git lfs ls-files
du -sh .git  # Debe ser mucho menor que antes

# Fase 4: Coordinar con el equipo (anunciar ventana de mantenimiento)
# - Todos deben pushear su trabajo
# - Todos deben re-clonar después del force-push

# Fase 5: Push forzado y notificación
git push --force-with-lease origin lfs-migration:main

# Fase 6: Actualizar CI/CD (verificar que los runners tengan git-lfs instalado)
# En GitHub Actions:
#   - uses: actions/checkout@v4
#     with:
#       lfs: true
#
# En GitLab CI:
#   before_script:
#     - apt-get update && apt-get install -y git-lfs
#     - git lfs install
```

**Claves para no romper CI/CD:**
1. Asegúrate de que todos los runners/servidores de CI tengan `git-lfs` instalado.
2. Configura `lfs: true` en el step de checkout (GitHub Actions) o `GIT_LFS_SKIP_SMUDGE=1` para saltar LFS en builds que no necesitan los binarios.
3. Migra primero en un fork/prueba para validar que los pipelines funcionan.
4. Programa la migración en una ventana de bajo tráfico.

### 14.2.8 Proveedores de LFS

| Proveedor         | Capacidad Gratuita         | Límites                         | Notas                        |
| ----------------- | -------------------------- | ------------------------------- | ---------------------------- |
| GitHub LFS        | 1 GB almacenamiento, 1 GB/mes ancho de banda | Paquetes de datos adicionales   | Integración nativa           |
| GitLab LFS        | 10 GB por repositorio      | Límites por plan                | GitLab.com y self-hosted     |
| Bitbucket LFS     | 1 GB (plan gratuito)       | Escala con el plan              | Activación en settings       |
| Azure DevOps      | Ilimitado (tarifa por uso) | Facturación por consumo         | Integración Azure Pipelines  |
| Self-hosted       | Depende del servidor       | Sin límites externos            | Implementación propia o S3   |

### 14.2.9 Self-Hosted LFS con Servidor Personalizado

```bash
# Configurar URL de LFS personalizada
git config lfs.url https://lfs.mi-empresa.com/endpoint

# Implementación con servidor de referencia
# https://github.com/git-lfs/lfs-test-server
```

## 14.3 Clon Superficial (Shallow Clone)

Un clon superficial descarga solo los commits más recientes, omitiendo el historial completo:

### 14.3.1 `git clone --depth`

```bash
# Clonar solo el último commit (historial de profundidad 1)
git clone --depth 1 https://github.com/grande/repo.git

# Tamaño típico:
#   Clone completo:  2.1 GB
#   Clone --depth 1: 180 MB (reducción del 91%)

# Clonar los últimos 50 commits
git clone --depth 50 https://github.com/grande/repo.git
```

### 14.3.2 Convertir un Clon Superficial en Completo

```bash
# Obtener más historial
git fetch --deepen=100   # 100 commits más

# Obtener el historial completo
git fetch --unshallow

# Verificar la profundidad actual
git rev-list --count HEAD
```

### 14.3.3 `--shallow-since` y `--shallow-exclude`

```bash
# Clonar commits desde una fecha específica
git clone --shallow-since="2024-01-01" https://github.com/repo.git

# Clonar excluyendo commits hasta una referencia
git clone --shallow-exclude=v1.0 https://github.com/repo.git
```

### 14.3.4 Limitaciones de Clones Superficiales

- No se puede hacer `git push` desde un clon superficial (necesita historial completo para la negociación).
- Bisect y blame no funcionan sobre commits no clonados.
- Algunas operaciones de merge pueden fallar si el ancestro común no está disponible.

## 14.4 Partial Clone (Clon Parcial)

Evoluciona el concepto de clon superficial: en lugar de limitar commits, omite ciertos tipos de objetos (blobs, trees) bajo demanda.

### 14.4.1 `--filter=blob:none`

Omite todos los blobs (contenido de archivos) al clonar. Los descarga solo cuando son necesarios:

```bash
# Clonar sin blobs (solo estructura de commits y trees)
git clone --filter=blob:none https://github.com/gigante/monorepo.git

# En el working tree, los archivos aparecen pero su contenido
# se descarga bajo demanda al hacer checkout o diff
```

### 14.4.2 `--filter=tree:0`

Ni siquiera descarga los trees; solo los commits:

```bash
git clone --filter=tree:0 https://github.com/gigante/repo.git
```

> **Advertencia:** Un clon con `--filter=tree:0` **no produce un working tree utilizable** por sí solo. Sin trees no hay estructura de directorios. Este filtro solo es útil en combinación con `--sparse` (sección 14.5.2) o para servidores de CI que solo necesitan metadatos de commits. Para la mayoría de casos, prefiere `--filter=blob:none`.

### 14.4.3 `--filter=blob:limit=N`

Omite blobs individuales que superen un tamaño específico:

```bash
# Clonar omitiendo archivos mayores a 10 MB
git clone --filter=blob:limit=10m https://github.com/gigante/repo.git

# Los archivos >10MB se descargan bajo demanda (al hacer checkout/diff)
# Ideal cuando sabes que hay binarios grandes pero quieres todo lo demás
```

### 14.4.4 Fetch con Filtros

```bash
# Configurar filtro en un repositorio existente
git config remote.origin.partialclonefilter blob:none

# Fetch bajo demanda ocurre automáticamente
git checkout feature-branch  # Descarga blobs necesarios para esta rama

# Fetch explícito con filtro
git fetch --filter=blob:limit=10m origin
# Omite blobs mayores a 10 MB
```

### Tabla Comparativa de Estrategias de Clonado

| Estrategia                | Comando                                  | Objetos Descargados       | Tamaño Relativo |
| ------------------------- | ---------------------------------------- | ------------------------- | --------------- |
| Clone completo            | `git clone <url>`                        | Todos                     | 100%            |
| Shallow (`--depth 1`)     | `git clone --depth 1 <url>`              | Commits recientes + blobs | ~10-30%         |
| Partial (`blob:none`)     | `git clone --filter=blob:none <url>`     | Commits + trees           | ~5-15%          |
| Partial (`tree:0`)        | `git clone --filter=tree:0 <url>`        | Solo commits              | ~1-5%           |

## 14.5 Sparse Checkout

Mientras que `partial clone` reduce qué se descarga del servidor, `sparse checkout` reduce qué se expande en el working tree.

### 14.5.1 Configuración Básica

```bash
# Habilitar sparse checkout (modo cone)
git sparse-checkout init --cone

# Definir qué directorios incluir en el working tree
git sparse-checkout set src/backend docs/

# Ver el patrón actual
git sparse-checkout list
# src/backend
# docs/

# Añadir más directorios
git sparse-checkout add src/frontend tests/

# Volver al checkout completo
git sparse-checkout disable
```

### 14.5.2 Combinación con Partial Clone

La combinación más potente para repositorios enormes:

```bash
# 1. Clon parcial (sin blobs)
git clone --filter=blob:none --sparse https://github.com/gigante/monorepo.git
cd monorepo

# 2. Sparse checkout: solo los directorios que necesitas
git sparse-checkout init --cone
git sparse-checkout set services/auth services/api

# Ahora solo tienes en tu working tree los directorios de auth y api,
# y los blobs solo se descargan cuando son necesarios
```

### 14.5.3 Sparse Index (Experimental)

Git 2.34+ incluye un índice disperso experimental que acelera `git status` y `git add`:

```bash
git config core.sparseCheckoutCone true
git sparse-checkout init --cone
git config index.sparse true  # Activar sparse index
```

## 14.6 Monorepos

Un monorepo es un repositorio único que contiene múltiples proyectos, bibliotecas y servicios. Empresas como Google, Meta, Microsoft y Uber operan monorepos masivos.

### 14.6.1 Ventajas

- **Código compartido inmediato:** Una modificación en una biblioteca se refleja instantáneamente en todos los consumidores.
- **Refactorización atómica:** Un solo commit puede tocar el backend, frontend y bibliotecas compartidas.
- **Estándares uniformes:** Linting, testing y CI se aplican consistentemente.
- **Visibilidad total:** Cualquier ingeniero puede explorar todo el código de la organización.

### 14.6.2 Desventajas

- **Escala:** Los clones pueden ser enormes.
- **CI/CD complejo:** Determinar qué construir y testear en cada commit requiere inteligencia.
- **Acoplamiento:** Proyectos que deberían ser independientes terminan compartiendo dependencias no deseadas.
- **Propiedad difusa:** Sin CODEOWNERS, es fácil meter cambios donde no se debe.

### 14.6.3 Herramientas para Monorepos

| Herramienta     | Enfoque                  | Ecosistema Principal           |
| --------------- | ------------------------ | ------------------------------ |
| **Bazel**       | Build system (Google)    | Multi-lenguaje, build caching  |
| **Lerna**       | Gestión de paquetes      | JavaScript/TypeScript, npm     |
| **Nx**          | Build system + CLI       | JavaScript/TypeScript, testing |
| **Turborepo**   | Build caching            | JavaScript/TypeScript          |
| **Pants**       | Build system             | Python, JVM, Go                |
| **Rush**        | Gestión de monorepo      | JavaScript/TypeScript          |

### 14.6.4 Estrategia de Branching en Monorepos

```
En un monorepo con Trunk-Based Development:

main
  ├── commit: "feat(auth): añadir OAuth2"
  │     cambia: services/auth/**, libs/oauth/**
  ├── commit: "fix(payments): corregir cálculo de IVA"
  │     cambia: services/payments/**, libs/tax/**
  └── commit: "chore(deps): actualizar React en dashboard"
        cambia: apps/dashboard/**

Cada commit afecta solo a una parte del monorepo.
CI detecta qué cambió y ejecuta solo lo necesario.
```

### 14.6.5 CI/CD en Monorepos

```yaml
# Ejemplo conceptual: GitHub Actions con detección de cambios
name: CI
on: [push]
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      auth: ${{ steps.filter.outputs.auth }}
      payments: ${{ steps.filter.outputs.payments }}
    steps:
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            auth: 'services/auth/**'
            payments: 'services/payments/**'

  test-auth:
    needs: changes
    if: ${{ needs.changes.outputs.auth == 'true' }}
    steps:
      - run: cd services/auth && npm test

  test-payments:
    needs: changes
    if: ${{ needs.changes.outputs.payments == 'true' }}
    steps:
      - run: cd services/payments && npm test
```

## 14.7 Gestión de Repositorios Grandes en la Empresa

### 14.7.1 Git en NFS/SMB: NO Recomendado

```bash
# Esto es una MALA IDEA:
git init /mnt/nfs/shared/repo.git
```

Git depende de operaciones de sistema de archivos rápidas y atómicas. NFS y SMB introducen latencia, problemas de bloqueo y corrupción potencial.

> **Mejor práctica:** Usa siempre almacenamiento local (SSD) y un servidor Git dedicado.

### 14.7.2 Opciones de Servidor Git: Self-Hosted vs SaaS

| Solución           | Tipo          | Mejor para                              |
|--------------------|---------------|-----------------------------------------|
| **GitHub Enterprise** | SaaS/On-prem  | Ecosistema GitHub, Actions, comunidad   |
| **GitLab**         | SaaS/On-prem  | DevOps integrado, CI/CD nativo          |
| **Bitbucket**      | SaaS/On-prem  | Integración Atlassian, Jira             |
| **Gitea**          | Self-hosted   | Alternativa ligera, open source, Raspberry Pi, equipos pequeños |
| **Gitolite**       | Self-hosted   | Control de acceso fino por rama, minimalista, solo SSH |
| **Gerrit**         | Self-hosted   | Code review obligatorio, Google-style   |

**Gitea** (gitea.io): Un fork de Gogs, escrito en Go, consume pocos recursos (~64 MB RAM). Ideal para self-hosted en VPS pequeño o intranet. Soporta LFS, CI vía Gitea Actions (compatible con GitHub Actions), y webhooks.

**Gitolite** (gitolite.com): No tiene interfaz web. Configuración de permisos por rama mediante archivos de texto en un repositorio Git (`gitolite-admin`). Extremadamente ligero, usado por el kernel de Linux y proyectos que solo necesitan SSH + control de acceso granular.

### 14.7.3 Replicación Geográfica

Equipos distribuidos globalmente sufren latencia de red:

| Solución                        | Descripción                                    |
| ------------------------------- | ---------------------------------------------- |
| **Mirrors geográficos**         | Réplicas de solo lectura en cada región        |
| **GitLab Geo**                  | Replicación activa/pasiva entre regiones       |
| **GitHub Enterprise replicas**  | Réplicas de lectura cerca de los equipos       |
| **Gerrit replication**          | Replicación de servidores Gerrit               |
| **CDN para LFS**                | Distribuir objetos LFS vía CDN (CloudFront, etc.)|

### 14.7.4 Backup y Disaster Recovery

```bash
# Backup completo (todos los objetos, refs, hooks)
git clone --mirror https://github.com/empresa/repo.git repo-backup.git

# Backup incremental
cd repo-backup.git
git fetch --all

# Restauración de un mirror
git clone repo-backup.git repo-restaurado
cd repo-restaurado
git remote add origin https://github.com/empresa/repo.git
git push --all origin
git push --tags origin
```

Estrategia recomendada de backup:

```
Backup diario:
  git clone --mirror → snapshot en S3/Backblaze/GCS

Backup semanal:
  Snapshot completo del servidor Git + LFS

Backup mensual:
  Verificación de integridad del backup más reciente
  (git fsck en el mirror restaurado)
```

### 14.7.5 Políticas de Retención

```bash
# Políticas de retención con GC agresivo
git config gc.auto 256          # GC más frecuente
git config gc.autoPackLimit 32  # Menos packs
git config gc.reflogExpire "30 days"
git config gc.reflogExpireUnreachable "7 days"
git config gc.rerereResolved "60 days"
git config gc.rerereUnresolved "15 days"

# Limpiar objetos innecesarios
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

### 14.7.6 Estrategia de Almacenamiento por Tiempo

```
Inmediato (~0-6 meses):
  └── Repositorio principal + LFS en caliente (SSD/NVMe)

Mediano plazo (~6-12 meses):
  └── Objetos LFS movidos a almacenamiento en frío (HDD)

Largo plazo (+12 meses):
  └── Snapshots comprimidos en almacenamiento de archivo (Glacier, Archive)

Análisis:
  git lfs ls-files -l | sort -rn -k2 | head -20
  # Identificar los archivos LFS más grandes para decidir qué mover a frío
```

---

## 14.8 `git maintenance` (Git 2.31+): Mantenimiento Programado

A partir de Git 2.31, `git maintenance` automatiza tareas de housekeeping que antes requerían scripts manuales:

```bash
# Registrar el repositorio para mantenimiento automático
git maintenance start
# Git configura tareas en cron (Linux/macOS) o Task Scheduler (Windows)

# Ver tareas registradas
git maintenance run --list

# Ejecutar todas las tareas ahora
git maintenance run

# Tareas que se ejecutan automáticamente:
# - gc (recolección de basura incremental)
# - commit-graph (acelera git log y git merge-base)
# - loose-objects (empaqueta objetos sueltos)
# - incremental-repack (reempaqueta incrementalmente)
# - prefetch (fetch en segundo plano, mantiene el repo actualizado)

# Desregistrar
git maintenance stop

# Configurar frecuencia de tareas
git config maintenance.gc.enabled true
git config maintenance.commit-graph.schedule hourly
```

**Beneficio:** En repositorios grandes, `git maintenance` previene la degradación progresiva del rendimiento sin intervención manual.

### Scalar (Microsoft)

[Scalar](https://github.com/microsoft/scalar) es una herramienta de Microsoft diseñada para repositorios gigantes (Windows, Office). Extiende `git maintenance` con optimizaciones agresivas:

- **Watchman integration:** Acelera `git status` usando el sistema de archivos.
- **Background maintenance:** Fetch y GC en segundo plano.
- **Sparse-checkout por defecto:** Solo descarga lo necesario.

```bash
# Clonar con Scalar
scalar clone https://github.com/gigante/repo.git
scalar register  # Activar mantenimiento en segundo plano
```

> **Nota:** Scalar es la base de VFS for Git (antes GVFS). Para la mayoría de equipos, `git maintenance` + sparse checkout son suficientes.

---

## Resumen del Capítulo 14

Git a gran escala requiere un conjunto de herramientas y estrategias que van más allá del uso básico:

1. **Git LFS** resuelve el problema de archivos binarios grandes, almacenándolos fuera del repositorio principal y descargándolos bajo demanda. Incluye bloqueo de archivos para prevenir conflictos en binarios no mergeables.
2. **Shallow Clone** (`--depth`) omite historial, ideal para CI/CD donde solo se necesita el último commit.
3. **Partial Clone** (`--filter`) omite blobs/trees al clonar, descargándolos bajo demanda, combinando lo mejor de shallow y completo. Usa `--filter=blob:limit=N` para omitir archivos grandes selectivamente.
4. **Sparse Checkout** reduce el working tree a solo los directorios necesarios, esencial en monorepos gigantes.
5. **Monorepos** ofrecen ventajas de cohesión pero requieren herramientas especializadas (Bazel, Nx, Turborepo) y estrategias de CI incremental.
6. **Infraestructura empresarial:** Nunca NFS/SMB, siempre SSD local + servidor dedicado + replicación geográfica + backups regulares. Alternativas self-hosted como Gitea y Gitolite cubren casos simples sin costo.
7. **Las políticas de retención y GC** mantienen el repositorio saludable a largo plazo. `git maintenance` (2.31+) automatiza estas tareas.
8. **Scalar** de Microsoft optimiza repositorios de escala extrema (>100 GB) con mantenimiento en segundo plano y sparse-checkout.

## Ejercicios Propuestos

### Ejercicio 1: Git LFS desde Cero

1. Crea un repositorio nuevo y configura Git LFS para rastrear archivos `*.png` y `*.mp4`.
2. Genera 3 archivos de imagen de prueba (puedes usar `dd` o descargar imágenes pequeñas) y haz commit.
3. Verifica con `git lfs ls-files` que los archivos están siendo gestionados por LFS.
4. Inspecciona el contenido de un puntero LFS en `.git/objects/` y compáralo con el archivo original.
5. Simula un `git lfs fetch` y verifica el contenido del directorio `.git/lfs/`.

### Ejercicio 2: Clon Superficial y Parcial

1. Toma un repositorio con al menos 100 commits (o crea uno con un script) y mide el tamaño de un clon completo.
2. Realiza un clon superficial con `--depth 1` y mide el tamaño. Calcula el porcentaje de reducción.
3. Realiza un clon parcial con `--filter=blob:none` y mide el tamaño.
4. Intenta hacer `git log` en cada tipo de clon y documenta qué información falta en cada uno.
5. Convierte el clon superficial a completo usando `git fetch --unshallow`.

### Ejercicio 3: Sparse Checkout en Monorepo Simulado

1. Crea una estructura de monorepo con al menos 5 servicios en directorios separados:
   ```
   services/auth/ services/api/ services/db/ services/cache/ services/webhook/
   ```
2. Realiza commits que modifiquen cada servicio de forma independiente.
3. Configura sparse checkout para ver solo `services/auth` y `services/api` en tu working tree.
4. Verifica con `ls` que los demás directorios no están presentes.
5. Añade `services/db` al sparse checkout y verifica que ahora aparece.
6. Vuelve al checkout completo y verifica que todos los directorios reaparecen.

### Ejercicio 4: Combinación Partial Clone + Sparse Checkout

1. Crea un repositorio que simule un monorepo de 200+ MB (usa `dd` para generar archivos grandes en varios directorios).
2. Combina partial clone con sparse checkout para clonar solo un subconjunto del repositorio:
   ```bash
   git clone --filter=blob:none --sparse <url>
   ```
3. Configura sparse checkout para ver solo el directorio que te interesa.
4. Mide el tiempo de clonado, el espacio ocupado por `.git` y el tamaño del working tree.
5. Compara estas métricas con un clon completo del mismo repositorio.

### Ejercicio 5: Análisis de Viabilidad de Monorepo

1. Investiga un proyecto real de tu entorno laboral o personal que esté dividido en múltiples repositorios.
2. Analiza qué beneficios y desventajas tendría migrarlo a un monorepo.
3. Especifica qué herramientas de monorepo serían adecuadas (Bazel, Nx, Turborepo, etc.) según el stack tecnológico.
4. Diseña una estrategia de branching y CI/CD para ese monorepo hipotético.
5. Documenta los riesgos de la migración y cómo los mitigarías.

---

← [Capítulo anterior](13-reescritura-historia.md) | [Inicio](README.md) | [Capítulo siguiente →](15-workflows.md)
