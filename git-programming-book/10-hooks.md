# Capítulo 10: Git Hooks

Los Git Hooks son uno de los mecanismos de automatización más poderosos y a la vez más infrautilizados de Git. Permiten ejecutar scripts personalizados en respuesta a eventos del ciclo de vida de Git: antes de un commit, después de un merge, al recibir un push en el servidor, y muchos más. Este capítulo explora todos los hooks disponibles, patrones prácticos de implementación y las herramientas modernas que facilitan su gestión en equipos de desarrollo.

---

## 10.1 ¿Qué Son los Git Hooks?

Los hooks son scripts ejecutables que Git invoca automáticamente cuando ocurren determinados eventos. Residen en el directorio `.git/hooks/` de cada repositorio y pueden estar escritos en cualquier lenguaje que el sistema pueda ejecutar (bash, Python, Ruby, Node.js, Perl, etc.).

```
CICLO DE VIDA DE UN COMMIT CON HOOKS DEL LADO CLIENTE
═══════════════════════════════════════════════════════════════

git commit -m "mensaje"
    │
    ▼
┌─────────────┐
│  pre-commit  │  ← ¿Pasa linting? ¿Pasan los tests?
└──────┬──────┘
       │ (éxito)
       ▼
┌────────────────────┐
│ prepare-commit-msg  │  ← Modificar mensaje por defecto
└────────┬───────────┘
         │
         ▼
┌──────────────┐
│  commit-msg   │  ← ¿El mensaje sigue el formato?
└──────┬───────┘
       │ (éxito)
       ▼
┌──────────────┐
│ post-commit   │  ← Notificar, registrar, loguear
└──────────────┘
```

> **Concepto clave:** Si un hook del lado cliente retorna un código de salida distinto de cero, Git aborta la operación. Esto los convierte en una herramienta de validación y control de calidad en el punto más temprano posible del flujo de desarrollo.

---

## 10.2 Ubicación y Descubrimiento

### Directorio de Hooks

```bash
# Ver los hooks disponibles en un repositorio
ls -la .git/hooks/

# Salida típica:
# applypatch-msg.sample
# commit-msg.sample
# fsmonitor-watchman.sample
# post-update.sample
# pre-applypatch.sample
# pre-commit.sample
# pre-merge-commit.sample
# prepare-commit-msg.sample
# pre-push.sample
# pre-rebase.sample
# pre-receive.sample
# push-to-checkout.sample
# update.sample
```

Los archivos `.sample` son ejemplos proporcionados por Git. Para activar un hook, simplemente renómbralo eliminando la extensión `.sample` y hazlo ejecutable.

```bash
# Activar el hook pre-commit de ejemplo
cp .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit

# O crea tu propio hook desde cero
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
echo "Ejecutando pre-commit hook personalizado"
EOF
chmod +x .git/hooks/pre-commit
```

### Configuración Global de Hooks

A partir de Git 2.9, puedes configurar un directorio global de hooks que se aplica a todos tus repositorios:

```bash
# Crear directorio global de hooks
mkdir -p ~/.git-hooks

# Configurar Git para usar este directorio
git config --global core.hooksPath ~/.git-hooks

# Crear un hook global (afecta a TODOS los repositorios)
cat > ~/.git-hooks/pre-commit << 'EOF'
#!/bin/bash
# Hook global: verifica que no se commitean archivos con conflict markers
if git diff --cached --check | grep -E '^(<<<<<<<|=======|>>>>>>>)'; then
    echo "ERROR: Encontrados conflict markers sin resolver"
    exit 1
fi
EOF
chmod +x ~/.git-hooks/pre-commit
```

> **Advertencia:** `core.hooksPath` sobrescribe completamente los hooks locales del repositorio. Los hooks en `.git/hooks/` dejarán de ejecutarse. Si necesitas hooks globales Y locales, tendrás que implementar un mecanismo de despacho.

### Encadenar Hooks Globales y Locales

Dado que `core.hooksPath` reemplaza completamente `.git/hooks/`, para tener ambos necesitas un hook que actue como "despachador":

```bash
#!/bin/bash
# ~/.git-hooks/pre-commit (global)
# Ejecuta primero hooks globales, luego hooks locales del repo

# 1. Ejecutar logica global
echo "Ejecutando hooks globales..."

# 2. Buscar y ejecutar hook local si existe
LOCAL_HOOK="$(git rev-parse --git-dir)/hooks/pre-commit"
if [ -f "$LOCAL_HOOK" ] && [ -x "$LOCAL_HOOK" ]; then
    echo "Ejecutando hooks locales del repositorio..."
    "$LOCAL_HOOK" || exit 1
fi

exit 0
```

Este patron permite que cada repositorio tenga sus propios hooks locales ademas de los globales. Alternativamente, usa herramientas como `pre-commit framework` o `Lefthook` que gestionan esto nativamente.

---

## 10.3 Hooks del Lado Cliente

Los hooks del lado cliente se ejecutan en la máquina del desarrollador, durante operaciones locales como commit, merge o checkout.

### pre-commit

Se ejecuta antes de que se abra el editor para el mensaje del commit. Es el hook más utilizado.

```bash
#!/bin/bash
# .git/hooks/pre-commit
# Validación: linting, formato, tests unitarios rápidos

set -e

echo "🔍 Ejecutando validaciones pre-commit..."

# Verificar archivos staged
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM)

if [ -z "$STAGED_FILES" ]; then
    echo "No hay archivos para validar."
    exit 0
fi

# 1. Verificar que no hay trailing whitespace
echo "▸ Verificando trailing whitespace..."
git diff --cached --check || {
    echo "ERROR: Encontrado trailing whitespace. Corrígelo antes de commitear."
    exit 1
}

# 2. Ejecutar linter solo en archivos modificados (ejemplo con shellcheck)
if echo "$STAGED_FILES" | grep -q '\.sh$'; then
    echo "▸ Ejecutando shellcheck..."
    echo "$STAGED_FILES" | grep '\.sh$' | xargs -r shellcheck || exit 1
fi

# 3. Ejecutar tests rápidos
echo "▸ Ejecutando tests unitarios..."
make test-unit || {
    echo "ERROR: Tests unitarios fallaron."
    exit 1
}

echo "✅ Todas las validaciones pasaron."
exit 0
```

### prepare-commit-msg

Se ejecuta después de `pre-commit`, antes de que se abra el editor. Permite modificar o enriquecer el mensaje de commit por defecto.

```bash
#!/bin/bash
# .git/hooks/prepare-commit-msg
# Añade el nombre de la rama como prefijo automáticamente

COMMIT_MSG_FILE=$1
COMMIT_SOURCE=$2
SHA1=$3

# Solo modificar si no es un commit de merge, squash o amend
if [ "$COMMIT_SOURCE" = "message" ] || [ "$COMMIT_SOURCE" = "template" ]; then
    BRANCH_NAME=$(git symbolic-ref --short HEAD 2>/dev/null)
    if [ -n "$BRANCH_NAME" ]; then
        # Extraer Issue Key del nombre de la rama (ej. feature/PROJ-123-login)
        ISSUE_KEY=$(echo "$BRANCH_NAME" | grep -oE '[A-Z]+-[0-9]+' | head -1)
        PREFIX=""
        if [ -n "$ISSUE_KEY" ]; then
            PREFIX="[$ISSUE_KEY] "
        fi

        # Prefijar el mensaje
        sed -i.bak "1s/^/$PREFIX/" "$COMMIT_MSG_FILE"
        rm -f "$COMMIT_MSG_FILE.bak"  # Limpiar backup de sed en macOS
    fi
fi
```

### commit-msg

Recibe la ruta a un archivo temporal con el mensaje de commit. Se ejecuta después de que el usuario guarda y cierra el editor.

```bash
#!/bin/bash
# .git/hooks/commit-msg
# Valida que el mensaje siga Conventional Commits

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Ignorar commits de merge
if echo "$COMMIT_MSG" | grep -qE '^Merge (branch|pull request)'; then
    exit 0
fi

# Patrón Conventional Commits: type(scope): description
PATTERN='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-zA-Z0-9_-]+\))?: .{1,72}$'

FIRST_LINE=$(echo "$COMMIT_MSG" | head -1)

if ! echo "$FIRST_LINE" | grep -qE "$PATTERN"; then
    echo "ERROR: El mensaje de commit no sigue Conventional Commits."
    echo ""
    echo "Formato esperado: <type>(<scope>): <description>"
    echo ""
    echo "Tipos válidos: feat, fix, docs, style, refactor, perf, test,"
    echo "               build, ci, chore, revert"
    echo ""
    echo "Ejemplo: feat(auth): añadir autenticación OAuth 2.0"
    exit 1
fi

# Verificar que no exceda 72 caracteres en la primera línea
LENGTH=$(echo "$FIRST_LINE" | wc -c)
if [ "$LENGTH" -gt 72 ]; then
    echo "ADVERTENCIA: La primera línea del mensaje excede 72 caracteres ($LENGTH)."
    echo "Considera acortarla para mejor visualización en logs."
fi

exit 0
```

### post-commit

Se ejecuta después de que el commit se completa. No puede abortar la operación.

```bash
#!/bin/bash
# .git/hooks/post-commit
# Registra estadísticas del commit para análisis

COMMIT_HASH=$(git rev-parse HEAD)
COMMIT_DATE=$(git log -1 --format=%ci)
FILES_CHANGED=$(git diff-tree --no-commit-id --name-only -r HEAD | wc -l)
INSERTIONS=$(git diff-tree --no-commit-id --shortstat -r HEAD | grep -oP '\d+(?= insertion)' || echo 0)

echo "📊 Commit: $COMMIT_HASH"
echo "   Fecha: $COMMIT_DATE"
echo "   Archivos modificados: $FILES_CHANGED"
echo "   Líneas insertadas: $INSERTIONS"

# Registrar en archivo de estadísticas
echo "$COMMIT_DATE | $COMMIT_HASH | $FILES_CHANGED | $INSERTIONS" >> .git/commit-stats.log
```

### pre-rebase

Se ejecuta antes de un rebase. El hook puede abortar el rebase.

```bash
#!/bin/bash
# .git/hooks/pre-rebase
# Previene rebase de main/develop para proteger ramas principales

UPSTREAM=$1
BRANCH=$2

PROTECTED_BRANCHES="main master develop"

for PROTECTED in $PROTECTED_BRANCHES; do
    if [ "$BRANCH" = "$PROTECTED" ]; then
        echo "ERROR: No se permite hacer rebase sobre la rama '$BRANCH'."
        echo "Utiliza merge en su lugar o consulta con el equipo."
        exit 1
    fi
done

exit 0
```

### pre-merge-commit

Se ejecuta despues de que un merge automatico tiene exito (sin conflictos), justo antes de que se cree el commit de merge. Permite inspeccionar o abortar el resultado del merge automatico.

```bash
#!/bin/bash
# .git/hooks/pre-merge-commit
# Verifica que el merge automatico no introdujo conflict markers

if git diff --cached --check | grep -qE '^(<<<<<<<|=======|>>>>>>>)'; then
    echo "ERROR: Conflict markers encontrados en el merge automatico."
    exit 1
fi

# Ejecutar tests rapidos post-merge
make test-unit || {
    echo "ERROR: Tests fallaron despues del merge automatico."
    exit 1
}

exit 0
```

### post-index-change (Git 2.36+)

Se ejecuta cada vez que el index (staging area) cambia: `git add`, `git reset`, etc. Util para actualizar herramientas que dependen del estado del index.

```bash
#!/bin/bash
# .git/hooks/post-index-change

OLD_HASH=$1
NEW_HASH=$2
WAS_AMEND=$3  # 1 si fue amend, 0 si no

echo "Index actualizado. Refrescando cache de linter..."
touch .eslintcache 2>/dev/null
```

### Hooks para `git am` (Apply Mailbox)

Estos hooks se ejecutan durante el flujo de `git am` (aplicar parches desde formato mailbox):

```bash
#!/bin/bash
# .git/hooks/applypatch-msg
# Recibe el archivo con el mensaje de commit antes de aplicarlo
COMMIT_MSG_FILE=$1
echo "Validando mensaje del parche..."
# Ej: verificar que tenga Signed-off-by
grep -q '^Signed-off-by:' "$COMMIT_MSG_FILE" || {
    echo "ERROR: El parche no tiene Signed-off-by."
    exit 1
}
```

```bash
#!/bin/bash
# .git/hooks/pre-applypatch
# Se ejecuta despues del parche, antes del commit
# Permite inspeccionar el working tree resultante
echo "Parche aplicado. Verificando estado..."
make test-unit || exit 1
```

```bash
#!/bin/bash
# .git/hooks/post-applypatch
# Se ejecuta despues de que el commit del parche se completo
COMMIT_HASH=$(git rev-parse HEAD)
echo "Parche aplicado como commit $COMMIT_HASH"
# Notificar, loguear, etc.
```

### post-checkout y post-merge

Se ejecutan después de `git checkout` y `git merge`, respectivamente. Útiles para tareas como instalar dependencias o migrar bases de datos cuando cambia la rama.

```bash
#!/bin/bash
# .git/hooks/post-checkout
# Instala dependencias automáticamente si package.json cambió

PREVIOUS_HEAD=$1
NEW_HEAD=$2
IS_BRANCH_CHECKOUT=$3

if [ "$IS_BRANCH_CHECKOUT" = "1" ]; then
    # Solo en cambio de rama (no en checkout de archivos individuales)
    if git diff --name-only "$PREVIOUS_HEAD" "$NEW_HEAD" | grep -q 'package.json'; then
        echo "📦 package.json cambió. Instalando dependencias..."
        npm install --silent
    fi
fi
```

### pre-push

Se ejecuta antes de que `git push` envíe datos al remoto. Ideal para ejecutar la suite completa de tests.

```bash
#!/bin/bash
# .git/hooks/pre-push
# Ejecuta la suite completa de tests antes de hacer push

REMOTE=$1
URL=$2

# Proteger contra push --force a main/master
PROTECTED_REFS="refs/heads/main refs/heads/master"

while read LOCAL_REF LOCAL_SHA REMOTE_REF REMOTE_SHA; do
    if echo "$PROTECTED_REFS" | grep -q "$REMOTE_REF"; then
        # Verificar si es force push (remote SHA no es ancestro de local SHA)
        if [ "$REMOTE_SHA" != "0000000000000000000000000000000000000000" ]; then
            if ! git merge-base --is-ancestor "$REMOTE_SHA" "$LOCAL_SHA"; then
                echo "ERROR: Push --force a $REMOTE_REF no está permitido."
                exit 1
            fi
        fi
    fi
done

echo "🧪 Ejecutando suite de tests completa antes del push..."
make test || {
    echo "ERROR: Los tests fallaron. Corrige los errores antes de hacer push."
    exit 1
}

exit 0
```

---

## 10.4 Hooks del Lado Servidor

Estos hooks se ejecutan en el repositorio remoto cuando recibe un push. Son la primera línea de defensa para mantener la integridad del repositorio central.

### pre-receive

Se ejecuta una vez por cada `git push`. Recibe una lista de todas las referencias que se están actualizando.

```bash
#!/bin/bash
# server/.git/hooks/pre-receive
# Validaciones antes de aceptar un push en el servidor

set -e

# Políticas de protección
PROTECTED_REFS="refs/heads/main refs/heads/master"
MAX_COMMIT_SIZE_KB=500

while read OLD_REV NEW_REV REF_NAME; do
    # 1. Prevenir eliminación de ramas protegidas
    if echo "$PROTECTED_REFS" | grep -q "$REF_NAME"; then
        if [ "$NEW_REV" = "0000000000000000000000000000000000000000" ]; then
            echo "ERROR: No se permite eliminar la rama $REF_NAME"
            exit 1
        fi
    fi

    # 2. Verificar que todos los commits tienen autor válido
    for COMMIT in $(git rev-list "$OLD_REV..$NEW_REV"); do
        AUTHOR_EMAIL=$(git log -1 --format=%ae "$COMMIT")
        if ! echo "$AUTHOR_EMAIL" | grep -q '@empresa\.com$'; then
            echo "ERROR: Commit $COMMIT tiene email de autor no corporativo: $AUTHOR_EMAIL"
            exit 1
        fi
    done

    # 3. Verificar tamaño máximo de commit
    for COMMIT in $(git rev-list "$OLD_REV..$NEW_REV"); do
        COMMIT_SIZE=$(git cat-file -s "$COMMIT")
        if [ "$COMMIT_SIZE" -gt $((MAX_COMMIT_SIZE_KB * 1024)) ]; then
            echo "ERROR: Commit $COMMIT excede el tamaño máximo de ${MAX_COMMIT_SIZE_KB}KB"
            exit 1
        fi
    done
done

exit 0
```

### update

Similar a `pre-receive`, pero se ejecuta **por cada referencia** individualmente. Recibe la rama, el SHA antiguo y el nuevo.

### reference-transaction (Git 2.28+)

Reemplazo moderno de `update` para interaccion programatica con cambios de referencia. Se ejecuta para cada transaccion planeada, ejecutada y abortada de referencias. Recibe el estado de la transaccion (prepared, committed, aborted):

```bash
#!/bin/bash
# server/.git/hooks/reference-transaction
# Para integracion con sistemas externos de auditoria

while read OLD_REV NEW_REV REF_NAME; do
    STATE=$1  # prepared, committed, aborted
    echo "Transaccion $STATE: $REF_NAME $OLD_REV -> $NEW_REV"

    if [ "$STATE" = "committed" ] && echo "$REF_NAME" | grep -q '^refs/heads/'; then
        curl -s -X POST -H 'Content-type: application/json' \
            --data "{\"ref\":\"$REF_NAME\",\"old\":\"$OLD_REV\",\"new\":\"$NEW_REV\"}" \
            "$AUDIT_WEBHOOK_URL" &
    fi
done

exit 0
```

A diferencia de `update`, `reference-transaction` cubre TODOS los cambios de referencia, no solo pushes, y notifica en todas las fases de la transaccion. Es la opcion preferida para integracion con sistemas externos desde Git 2.28+.

### Hook `update` (legado)

```bash
#!/bin/bash
# server/.git/hooks/update
# Se ejecuta una vez por cada rama en el push

REF_NAME=$1
OLD_REV=$2
NEW_REV=$3

echo "Procesando actualización: $REF_NAME"

# Proteger ramas con prefijo release/
if echo "$REF_NAME" | grep -q '^refs/heads/release/'; then
    # Solo el release manager puede pushear a ramas de release
    USER=$(whoami)
    if [ "$USER" != "release-bot" ]; then
        echo "ERROR: Solo el release manager puede modificar ramas release/"
        exit 1
    fi
fi

# Verificar que no se introducen archivos grandes (>5MB)
for COMMIT in $(git rev-list "$OLD_REV..$NEW_REV"); do
    git diff-tree --no-commit-id -r "$COMMIT" | while read MODE TYPE HASH FILE; do
        SIZE=$(git cat-file -s "$HASH")
        if [ "$SIZE" -gt 5242880 ]; then
            echo "ERROR: Archivo $FILE excede 5MB en commit $COMMIT"
            exit 1
        fi
    done
done

exit 0
```

### post-receive

Se ejecuta después de que el push se ha completado. No puede rechazar el push, pero es ideal para notificaciones, despliegues automáticos e integración continua.

```bash
#!/bin/bash
# server/.git/hooks/post-receive
# Notificar y desplegar después de recibir un push

while read OLD_REV NEW_REV REF_NAME; do
    BRANCH=$(basename "$REF_NAME")

    echo "📥 Push recibido en $BRANCH: $OLD_REV → $NEW_REV"

    # Notificar por webhook (Slack, Discord, Teams)
    COMMITS=$(git log --oneline "$OLD_REV..$NEW_REV" | head -5)
    curl -s -X POST -H 'Content-type: application/json' \
        --data "{\"text\":\"Push a \`$BRANCH\`: $COMMITS\"}" \
        "$SLACK_WEBHOOK_URL" &

    # Desplegar automáticamente si es main/master
    if [ "$BRANCH" = "main" ] || [ "$BRANCH" = "master" ]; then
        echo "🚀 Iniciando despliegue automático..."
        export GIT_WORK_TREE=/var/www/produccion
        git checkout -f "$BRANCH"
        cd /var/www/produccion
        make deploy
        echo "✅ Despliegue completado."
    fi

    # Registrar en archivo de auditoría
    echo "$(date -Iseconds) | $BRANCH | $USER | $OLD_REV → $NEW_REV" >> /var/log/git-pushes.log
done

exit 0
```

---

## 10.5 Variables de Entorno para Hooks

Git expone variables de entorno a los scripts de hooks que permiten acceder a informacion del repositorio sin necesidad de invocar comandos Git adicionales:

| Variable | Descripcion |
|----------|-------------|
| `GIT_DIR` | Ruta al directorio `.git` del repositorio |
| `GIT_WORK_TREE` | Ruta al working tree (directorio raiz del proyecto) |
| `GIT_INDEX_FILE` | Ruta al archivo index (staging area) |
| `GIT_EDITOR` | Editor configurado para mensajes de commit |
| `GIT_AUTHOR_NAME` | Nombre del autor del commit |
| `GIT_AUTHOR_EMAIL` | Email del autor del commit |
| `GIT_AUTHOR_DATE` | Fecha del autor |
| `GIT_PREFIX` | Prefijo de ruta desde la raiz del repo (subdirectorios) |

```bash
#!/bin/bash
# Ejemplo: usar variables de entorno en un hook
echo "Repo: $GIT_DIR"
echo "Worktree: $GIT_WORK_TREE"
echo "Autor: $GIT_AUTHOR_NAME <$GIT_AUTHOR_EMAIL>"
```

## 10.6 Ejecutar Hooks Manualmente: `git hook run` (Git 2.36+)

Git 2.36+ permite invocar hooks manualmente, util para depuracion, pruebas y CI:

```bash
# Ejecutar un hook especifico manualmente
git hook run pre-commit

# Pasando argumentos (como lo haria Git internamente)
git hook run commit-msg -- .git/COMMIT_EDITMSG

git hook run post-commit

# En CI: verificar que los hooks del proyecto pasan
git hook run pre-commit || echo "Hooks fallaron"
```

> **Nota:** `git hook run` respeta `core.hooksPath`. Si configuraste un directorio global de hooks, `git hook run` ejecutara los hooks desde alli, no desde `.git/hooks/`.

---

## 10.7 Ejemplos Prácticos Completos

### Ejemplo 1: pre-commit con Linter (ESLint/Flake8)

**Versión JavaScript (ESLint):**
```bash
#!/bin/bash
# .git/hooks/pre-commit - Linter JavaScript/TypeScript

echo "🔍 Ejecutando ESLint en archivos staged..."

STAGED_JS=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(js|jsx|ts|tsx)$')

if [ -z "$STAGED_JS" ]; then
    echo "No hay archivos JavaScript/TypeScript para revisar."
    exit 0
fi

# Ejecutar ESLint solo en archivos staged
echo "$STAGED_JS" | xargs npx eslint --quiet

if [ $? -ne 0 ]; then
    echo ""
    echo "⚠️  ESLint encontró errores. Corrígelos antes de commitear."
    echo "   Usa 'npx eslint --fix <archivo>' para auto-corregir algunos problemas."
    exit 1
fi

echo "✅ ESLint: sin errores."
exit 0
```

**Versión Python (Flake8):**
```bash
#!/bin/bash
# .git/hooks/pre-commit - Linter Python (Flake8)

echo "🐍 Ejecutando Flake8 en archivos Python staged..."

STAGED_PY=$(git diff --cached --name-only --diff-filter=ACM | grep '\.py$')

if [ -z "$STAGED_PY" ]; then
    echo "No hay archivos Python para revisar."
    exit 0
fi

echo "$STAGED_PY" | xargs flake8 --max-line-length=100

if [ $? -ne 0 ]; then
    echo ""
    echo "⚠️  Flake8 encontró errores de estilo."
    exit 1
fi

# Verificar imports ordenados (isort)
echo "$STAGED_PY" | xargs isort --check-only --diff

echo "✅ Flake8 e isort: sin errores."
exit 0
```

### Ejemplo 2: commit-msg Validando Conventional Commits con Múltiples Reglas

```bash
#!/bin/bash
# .git/hooks/commit-msg - Validación avanzada de mensajes

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Regla 1: Primera línea no vacía
FIRST_LINE=$(echo "$COMMIT_MSG" | head -1)
if [ -z "$FIRST_LINE" ]; then
    echo "ERROR: El mensaje de commit no puede estar vacío."
    exit 1
fi

# Regla 2: Separar título del cuerpo con línea en blanco
LINE_COUNT=$(echo "$COMMIT_MSG" | head -2 | wc -l)
if [ "$LINE_COUNT" -ge 2 ]; then
    SECOND_LINE=$(echo "$COMMIT_MSG" | sed -n '2p')
    if [ -n "$SECOND_LINE" ]; then
        echo "ERROR: Separa el título del cuerpo con una línea en blanco."
        exit 1
    fi
fi

# Regla 3: No empezar con mayúscula (convención de equipo)
if echo "$FIRST_LINE" | grep -q '^[A-Z]'; then
    echo "ERROR: El mensaje debe empezar con minúscula según la convención del equipo."
    exit 1
fi

# Regla 4: No terminar con punto
if echo "$FIRST_LINE" | grep -q '\.$'; then
    echo "ERROR: La primera línea no debe terminar con punto."
    exit 1
fi

# Regla 5: Usar modo imperativo (heurística simple)
FORBIDDEN_STARTS="^(Added|Fixed|Removed|Updated|Changed|Modified|Created|Deleted|Refactored)"
if echo "$FIRST_LINE" | grep -qE "$FORBIDDEN_STARTS"; then
    echo "ADVERTENCIA: El mensaje no parece usar modo imperativo."
    echo "  Usa 'add' en lugar de 'Added', 'fix' en lugar de 'Fixed', etc."
fi

echo "✅ Mensaje de commit validado correctamente."
exit 0
```

### Ejemplo 3: post-receive para Despliegue Continuo

```bash
#!/bin/bash
# server/.git/hooks/post-receive
# Sistema completo de despliegue continuo

set -e

DEPLOY_DIR="/var/www/app"
BACKUP_DIR="/var/backups/app"
LOG_FILE="/var/log/deployments.log"
MAX_BACKUPS=5

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

while read OLD_REV NEW_REV REF_NAME; do
    BRANCH=$(basename "$REF_NAME")

    log "Push recibido: $BRANCH ($OLD_REV → $NEW_REV)"

    if [ "$BRANCH" != "main" ]; then
        log "Rama '$BRANCH' ignorada para despliegue."
        continue
    fi

    # Crear backup de la versión actual
    TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
    BACKUP_PATH="$BACKUP_DIR/$TIMESTAMP"

    if [ -d "$DEPLOY_DIR" ]; then
        log "Creando backup en $BACKUP_PATH..."
        mkdir -p "$BACKUP_PATH"
        rsync -a "$DEPLOY_DIR/" "$BACKUP_PATH/"
    fi

    # Desplegar nueva versión
    log "Desplegando commit $NEW_REV..."
    GIT_WORK_TREE="$DEPLOY_DIR" git checkout -f "$NEW_REV"

    # Post-despliegue: instalar dependencias, migrar, reiniciar
    cd "$DEPLOY_DIR"
    log "Instalando dependencias..."
    npm ci --production --silent

    if [ -f "migrations/run.sh" ]; then
        log "Ejecutando migraciones..."
        bash migrations/run.sh
    fi

    log "Reiniciando aplicación..."
    pm2 reload app --update-env || systemctl restart app

    # Limpiar backups antiguos
    BACKUP_COUNT=$(ls -1d "$BACKUP_DIR"/*/ 2>/dev/null | wc -l)
    if [ "$BACKUP_COUNT" -gt "$MAX_BACKUPS" ]; then
        log "Limpiando backups antiguos..."
        ls -1dt "$BACKUP_DIR"/*/ | tail -n +$((MAX_BACKUPS + 1)) | xargs rm -rf
    fi

    log "✅ Despliegue completado exitosamente."
done

exit 0
```

---

## 10.8 Lenguajes para Hooks

### Bash

Ventajas: nativo, sin dependencias, ejecución rápida. Ideal para hooks simples de validación.

### Python

Útil cuando necesitas lógica más compleja o acceso a APIs.

```python
#!/usr/bin/env python3
# .git/hooks/commit-msg
"""Valida mensajes de commit contra un patrón configurable."""

import sys
import re
import os

COMMIT_MSG_FILE = sys.argv[1]

with open(COMMIT_MSG_FILE, 'r') as f:
    commit_msg = f.read().strip()

first_line = commit_msg.split('\n')[0] if commit_msg else ''

# Cargar patrón desde .git/config o usar default
pattern_str = os.popen(
    'git config hooks.commit-msg.pattern'
).read().strip() or r'^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?: .{1,72}'

if not re.match(pattern_str, first_line):
    print("ERROR: El mensaje de commit no cumple con el formato requerido.")
    print(f"Patrón: {pattern_str}")
    sys.exit(1)

sys.exit(0)
```

### Node.js

Popular en equipos full-stack JavaScript. Usa el stack nativo del proyecto.

```javascript
#!/usr/bin/env node
// .git/hooks/pre-push
const { execSync } = require('child_process');

try {
    console.log('🧪 Ejecutando tests antes del push...');
    execSync('npm test', { stdio: 'inherit', cwd: process.cwd() });
    console.log('✅ Tests aprobados. Push autorizado.');
    process.exit(0);
} catch (error) {
    console.error('❌ Tests fallaron. Push rechazado.');
    process.exit(1);
}
```

---

## 10.9 El Problema de Compartir Hooks

> **Problema fundamental:** El directorio `.git/hooks/` **no se versiona** porque está dentro de `.git`, que es el repositorio local. No puedes hacer `git add .git/hooks/` ni `git commit` de esos archivos. Cada miembro del equipo debe configurar sus hooks manualmente.

### Solución 1: Directorio de Hooks + Script de Instalación

```bash
# Estructura del proyecto
my-project/
├── .githooks/              # Hooks versionados en el repo
│   ├── pre-commit
│   ├── commit-msg
│   ├── pre-push
│   └── post-checkout
├── scripts/
│   └── install-hooks.sh    # Script de instalación
├── src/
└── .gitignore
```

```bash
#!/bin/bash
# scripts/install-hooks.sh
# Instala los hooks del proyecto en .git/hooks/

HOOKS_SOURCE=".githooks"
HOOKS_TARGET=".git/hooks"

echo "Instalando Git hooks..."

for hook in "$HOOKS_SOURCE"/*; do
    hook_name=$(basename "$hook")
    target="$HOOKS_TARGET/$hook_name"

    # Crear symlink (preferible) o copiar
    ln -sf "../../$hook" "$target"
    chmod +x "$target"
    echo "  ✓ $hook_name"
done

echo "✅ Hooks instalados correctamente."
```

```makefile
# Makefile
.PHONY: setup-hooks

setup-hooks:
	bash scripts/install-hooks.sh

# Incluir en la configuración inicial
setup: setup-hooks
	npm install
```

### Solución 2: HooksPath de Git

```bash
# En el repositorio, crea directorio de hooks
mkdir -p .githooks

# Cada miembro del equipo configura el hooksPath
git config core.hooksPath .githooks
```

El problema es que `core.hooksPath` es una configuración local (no se versiona). Se puede automatizar en el script de setup o en el Makefile.

### Solución 3: Makefile / Script de Bootstrap

```bash
# Makefile (parcial)
.PHONY: bootstrap bootstrap-hooks

bootstrap: bootstrap-hooks install deps

bootstrap-hooks:
	git config core.hooksPath .githooks
	@echo "Hooks configurados en .githooks/"

install:
	npm ci

deps:
	npm install
```

```bash
#!/bin/bash
# bin/bootstrap.sh - Script de inicialización del proyecto
# Ejecutar después de clonar: bash bin/bootstrap.sh

set -e

echo "🚀 Inicializando proyecto..."

# 1. Configurar hooks
echo "▸ Configurando Git hooks..."
git config core.hooksPath .githooks
ln -sf ../../.githooks .git/hooks 2>/dev/null || true
echo "  ✓ Hooks configurados."

# 2. Instalar dependencias
echo "▸ Instalando dependencias..."
npm ci
echo "  ✓ Dependencias instaladas."

# 3. Configurar linters
echo "▸ Configurando herramientas de calidad..."
npx husky install 2>/dev/null || true
echo "  ✓ Herramientas configuradas."

echo ""
echo "✅ Proyecto inicializado. ¡Listo para desarrollar!"
```

---

## 10.10 Alternativas Modernas para Git Hooks

### Husky (JavaScript/Node.js)

La herramienta más popular en el ecosistema Node.js para gestionar hooks.

```bash
# Instalación
npm install --save-dev husky
npx husky init

# Estructura generada:
# .husky/
# ├── pre-commit
# ├── commit-msg
# └── _
#     └── husky.sh
```

```bash
# .husky/pre-commit
npm run lint-staged
```

```bash
# .husky/commit-msg
npx --no -- commitlint --edit $1
```

**Ventajas de Husky:**
- Instalación automática con `npm install` (postinstall script)
- Gestión declarativa de hooks
- Corre en CI sin configuración adicional (porque no depende de `.git/hooks/`)

### Lefthook (Multi-lenguaje, Rápido)

Alternativa escrita en Go, ideal para equipos políglotas. Más rápido que Husky porque no necesita Node.js.

```bash
# Instalación
# macOS
brew install lefthook
# Linux
curl -1sLf 'https://dl.cloudsmith.io/public/evilmartians/lefthook/setup.deb.sh' | bash
sudo apt install lefthook

# Inicializar en proyecto
lefthook install
```

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    eslint:
      glob: "*.{js,ts,jsx,tsx}"
      run: npx eslint {staged_files} --fix
    prettier:
      glob: "*.{js,ts,jsx,tsx,json,md,yaml}"
      run: npx prettier --check {staged_files}
    rubocop:
      glob: "*.rb"
      run: bundle exec rubocop --force-exclusion {staged_files}

commit-msg:
  commands:
    commitlint:
      run: npx commitlint --edit {1}

pre-push:
  commands:
    tests:
      run: npm test
    security:
      run: npm audit --audit-level=high
```

### pre-commit Framework (Python)

Originalmente creado por el equipo de Yelp, es ahora el estándar de facto para equipos Python pero soporta múltiples lenguajes.

```bash
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: detect-private-key

  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks:
      - id: black
        language_version: python3.12

  - repo: https://github.com/PyCQA/flake8
    rev: 7.1.0
    hooks:
      - id: flake8
        args: ['--max-line-length=100']

  - repo: https://github.com/PyCQA/isort
    rev: 5.13.2
    hooks:
      - id: isort

  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.29.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

```bash
# Instalar los hooks
pre-commit install           # Instala pre-commit
pre-commit install --hook-type commit-msg   # Instala commit-msg
pre-commit install --hook-type pre-push     # Instala pre-push

# Ejecutar todos los hooks manualmente
pre-commit run --all-files

# Probar hooks en CI
pre-commit run --all-files --show-diff-on-failure
```

---

## 10.11 Comparativa de Herramientas

| Herramienta | Lenguaje | Velocidad | Curva | Instalación | Ideal para |
|-------------|----------|-----------|-------|-------------|------------|
| **Hooks nativos** | Cualquiera | Alta | Media | Manual | Proyectos pequeños, servidores |
| **Husky** | Node.js | Media | Baja | Automática (npm) | Proyectos JavaScript/TypeScript |
| **Lefthook** | Go | Alta | Baja | Binario / brew | Equipos políglotas |
| **pre-commit** | Python | Media | Baja | `pip install` | Proyectos Python, entornos CI |

---

## 10.12 Cuándo Usar Hooks vs CI/CD

Una pregunta frecuente es si la validación debe ir en hooks locales o en el pipeline de CI/CD.

```
RECOMENDACIÓN: USAR AMBOS (DEFENSA EN CAPAS)

Capa 1 - Hooks locales     →   Validación rápida, feedback inmediato
Capa 2 - Pre-push hook     →   Suite de tests, seguridad
Capa 3 - CI/CD pipeline    →   Validación completa, builds, integración
Capa 4 - Server hooks      →   Protección del repositorio central
```

| Validación | Hook Local | CI/CD |
|------------|------------|-------|
| Linting | ✓ (rápido, feedback inmediato) | ✓ (respaldo) |
| Tests unitarios | ✓ (si son rápidos) | ✓ (completo) |
| Tests de integración | ✗ (demasiado lentos) | ✓ |
| Build | ✗ | ✓ |
| Análisis de seguridad | △ (básico) | ✓ (Snyk, SonarQube) |
| Firmas GPG | ✓ (pre-commit) | ✓ (verificación) |

> **Tip:** Nunca confíes exclusivamente en hooks del lado cliente. Un desarrollador puede usar `--no-verify` para saltarlos. Las validaciones críticas deben estar en el servidor (`pre-receive`) y en CI/CD.

### Configurar Hooks como Respaldo en CI

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run pre-commit hooks
        uses: pre-commit/action@v3.0.1

      - name: Commit message validation
        run: |
          git log --format=%B -n 1 ${{ github.sha }} | npx commitlint

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

---

## 10.13 Buenas Prácticas

1. **Mantén los hooks rápidos.** Los hooks de pre-commit deben ejecutarse en segundos, no en minutos. Si un hook es lento, muévelo a pre-push o CI.

2. **Versiona la configuración.** Aunque `.git/hooks/` no se versiona, versiona el directorio `.githooks/` y un script de instalación.

3. **Proporciona mensajes de error útiles.** Un hook que falla sin explicación frustra al desarrollador. Indica qué falló y cómo solucionarlo.

4. **Permite bypass controlado.** `git commit --no-verify` es una válvula de escape necesaria. No intentes bloquearla, pero registra cuándo se usa (ej. en CI verifica que los hooks se ejecutaron).

5. **Usa herramientas modernas.** Husky o Lefthook reducen la fricción de instalación y mantenimiento. pre-commit framework es el estándar para equipos Python.

6. **No dupliques validación pesada.** Si el CI ya ejecuta la suite completa de tests, el hook pre-push puede limitarse a tests unitarios rápidos o un subset crítico.

7. **Estandariza los mensajes de error.** Usa un formato consistente (colores, emojis o códigos de error) para que los desarrolladores reconozcan rápidamente los fallos de hooks.

---

## Resumen del Capítulo 10

- Los **Git Hooks** son scripts que Git ejecuta automáticamente en respuesta a eventos. Se clasifican en **hooks del lado cliente** (pre-commit, commit-msg, post-commit, pre-push, pre-rebase, post-checkout, post-merge) y **hooks del lado servidor** (pre-receive, update, post-receive).
- Los hooks se ubican en `.git/hooks/` y se activan eliminando la extensión `.sample` y haciéndolos ejecutables. Git 2.9+ soporta un directorio global de hooks con `core.hooksPath`.
- Los hooks del lado cliente pueden abortar la operación retornando un código distinto de cero. Los más usados son `pre-commit` (linting, formato) y `commit-msg` (validación de mensajes).
- Los hooks del lado servidor protegen el repositorio central validando commits, previniendo force-push y ejecutando despliegues automáticos.
- El directorio `.git/hooks/` **no se versiona**, lo que crea un problema de distribución. Las soluciones incluyen directorios `.githooks/` versionados con scripts de instalación, o herramientas modernas como **Husky**, **Lefthook** y **pre-commit framework**.
- La estrategia recomendada es defensa en capas: hooks locales (rápidos) + CI/CD (completo) + server hooks (protección final).

---

## Ejercicios Propuestos

1. **Crear un hook pre-commit personalizado:** Escribe un hook en bash que verifique que ningún archivo staged contenga `console.log` (JavaScript), `print(` (Python) o `var_dump(` (PHP). Si encuentra alguno, debe abortar el commit con un mensaje explicativo y la lista de archivos ofensores.

2. **Implementar Conventional Commits obligatorio:** Escribe un hook `commit-msg` en el lenguaje que prefieras (bash, Python o Node.js) que rechace cualquier commit cuyo mensaje no siga la especificación Conventional Commits. Incluye soporte para scope opcional y breaking changes (`!`).

3. **Simular un hook de servidor post-receive:** Configura un repositorio bare local (`git init --bare`) y escribe un hook `post-receive` que, cada vez que recibe un push a `main`, copie los archivos a un directorio de "despliegue" simulado y genere un archivo de log con los detalles del push.

4. **Instalar y configurar pre-commit framework:** En un proyecto existente (preferiblemente Python), instala `pre-commit`, crea un archivo `.pre-commit-config.yaml` con al menos 3 hooks (trailing-whitespace, black/flake8, y uno de tu elección), ejecuta `pre-commit run --all-files` y corrige los problemas encontrados.

5. **Migrar de hooks nativos a Lefthook:** Copia los hooks de ejemplo del capítulo (pre-commit con linter, commit-msg con validación) a una configuración de Lefthook (`lefthook.yml`). Verifica que los hooks se ejecutan correctamente con `lefthook run pre-commit` y `lefthook run commit-msg`.

---

← [Capítulo anterior](09-submodulos.md) | [Inicio](README.md) | [Capítulo siguiente →](11-internals.md)
