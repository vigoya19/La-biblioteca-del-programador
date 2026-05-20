# Capitulo 17: Git Avanzado

Este capitulo explora comandos y tecnicas avanzadas de Git que todo desarrollador profesional debe dominar. Desde la busqueda binaria de bugs hasta la transferencia de datos sin red, estas herramientas te permitiran resolver problemas complejos con precision quirurgica.

---

## 17.1 git bisect: Busqueda Binaria de Bugs

`git bisect` utiliza busqueda binaria para encontrar el commit exacto que introdujo un bug. Dado un rango de commits entre uno "bueno" (sin bug) y uno "malo" (con bug), bisect reduce logaritmicamente el espacio de busqueda.

### 17.1.1 Uso Manual

```bash
# Iniciar sesion de bisect
git bisect start

# Marcar commit actual como malo (tiene el bug)
git bisect bad HEAD

# Marcar un commit antiguo como bueno
git bisect good v1.0.0

# Git hace checkout a un commit intermedio
# Probamos manualmente...

# Si el bug esta presente:
git bisect bad

# Si el bug NO esta presente:
git bisect good

# Repetir hasta que Git identifica el primer commit malo:
# abc1234 is the first bad commit

# Finalizar sesion
git bisect reset
```

### 17.1.2 Automatizar con git bisect run

```bash
# Crear un script que devuelva 0 si el test pasa, 1-127 si falla
cat > test_bug.sh << 'EOF'
#!/bin/bash
npm test -- --testPathPattern=bug-scenario
EOF
chmod +x test_bug.sh

# Ejecutar bisect automaticamente
git bisect start HEAD v1.0.0
git bisect run ./test_bug.sh

# Git itera automaticamente hasta encontrar el commit culpable
# Bisecting: 14 revisions left to test after this (roughly 4 steps)
```

### 17.1.3 Caso Practico: Encontrar Commit que Rompio Tests

```bash
# Escenario: la suite de tests pasaba en v2.0.0 pero falla en HEAD

# Opcion 1: Busqueda manual
git bisect start
git bisect bad HEAD
git bisect good v2.0.0
# Repetir bad/good...

# Opcion 2: Busqueda automatica
git bisect start HEAD v2.0.0
git bisect run npm test

# Opcion 3: Usando terminos personalizados
git bisect start --term-old=passed --term-new=failed
git bisect failed HEAD
git bisect passed v2.0.0
git bisect run npm test

# Al finalizar:
# abc1234 is the first failed commit
git bisect reset
git show abc1234  # Revisar que cambio introdujo el bug
```

> **Tip**: Si el test que falla tiene dependencias externas (BD, APIs), usa `git bisect run` con un script que configure el entorno necesario o mockee las dependencias. Asi garantizas que cada paso de bisect sea determinista.

### 17.1.4 Estrategia de Busqueda con Paths

```bash
# Limitar bisect a cambios en un directorio especifico
git bisect start -- src/modulo-critico/

# Solo commits que tocaron ese path seran considerados sospechosos
```

---

## 17.2 git blame: Auditoria Linea por Linea

`git blame` muestra, linea por linea, que commit modifico por ultima vez cada linea de un archivo, quien lo hizo y cuando.

### 17.2.1 Uso Basico

```bash
# Blame completo de un archivo
git blame src/app.js

# Blame de un rango de lineas especifico
git blame -L 50,100 src/app.js

# Blame desde un commit especifico hacia atras
git blame -L 50,100 abc1234 -- src/app.js

# Mostrar solo lineas entre 10 lineas antes y 5 despues del rango
git blame -L 50,100 src/app.js
```

Salida tipica:
```
abc12345 (Alice 2026-03-15 10:30:00 +0100  50) function login() {
def67890 (Bob   2026-04-01 14:22:00 +0100  51)   const user = await auth();
abc12345 (Alice 2026-03-15 10:30:00 +0100  52)   return user;
```

### 17.2.2 Opciones Avanzadas

| Opcion | Descripcion |
|--------|-------------|
| `-L <inicio>,<fin>` | Rango de lineas (ej: `-L 50,100` o `-L /funcion/)` |
| `-C` | Detectar lineas copiadas del mismo commit (entre archivos) |
| `-CC` | Detectar lineas copiadas de cualquier commit |
| `-CCC` | Busqueda exhaustiva de contenido movido/copiado (lento) |
| `-w` | Ignorar cambios de whitespace |
| `-M` | Detectar lineas movidas dentro del mismo archivo |
| `--date=relative` | Fechas relativas ("3 weeks ago") |
| `--date=short` | Fechas cortas ("2026-03-15") |
| `--date=iso` | Formato ISO 8601 |
| `-e` | Mostrar email del autor |
| `-s` | Suprimir nombre y fecha (solo commit hash) |
| `--since="2 weeks ago"` | Filtrar por tiempo |

```bash
# Trazar el origen real del codigo (detectar movido/copiado)
git blame -C -C -C -L 140,160 src/refactorizado.js

# Ignorar commit de reformateo masivo
git blame -w src/app.js

# Mostrar con emails y formato ISO
git blame -e --date=iso src/app.js
```

### 17.2.3 Integracion en IDEs

La mayoria de IDEs integran git blame como anotaciones en el margen del editor (VS Code con GitLens, IntelliJ, etc.), mostrando el autor y la fecha al posicionar el cursor sobre una linea.

> **Tip**: `git blame` es una herramienta de auditoria, no de asignacion de culpas. Usala para entender el contexto historico de un cambio (por que se hizo), no para señalar responsables.

---

## 17.3 git grep: Busqueda Rapida en el Repositorio

`git grep` busca patrones en el arbol de trabajo o en el historial del repositorio, optimizado para velocidad en repositorios grandes.

### 17.3.1 Uso Basico

```bash
# Buscar patron en archivos versionados
git grep "TODO"

# Buscar mostrando numero de linea
git grep -n "function login"

# Buscar mostrando solo nombres de archivo
git grep -l "class UserController"

# Contar ocurrencias por archivo
git grep --count "import.*from"

# Buscar en un commit/tag especifico
git grep "legacyAPI" v1.0.0

# Buscar en todos los tags
git grep "deprecated" $(git tag)
```

### 17.3.2 Opciones de Busqueda

```bash
# Buscar patron como palabra completa
git grep -w "user"

# Busqueda case-insensitive
git grep -i "error"

# Buscar en una extension especifica
git grep "fetch" -- '*.js' '*.ts'

# Buscar excluyendo directorios
git grep "debug" -- ':!dist' ':!node_modules'

# Combinar patrones con AND
git grep --all-match -e "function" -e "async"

# Combinar con OR
git grep -e "ERROR" -e "FATAL" -e "CRITICAL"

# Regex avanzado (ERE)
git grep -E "function\s+\w+\(.*\)\s*{"

# Mostrar contexto (lineas antes/despues)
git grep -A 3 -B 2 "handleError"
```

### 17.3.3 git grep vs grep Tradicional

| Caracteristica | git grep | grep tradicional |
|----------------|----------|------------------|
| Archivos versionados solamente | Si (por defecto) | No |
| Velocidad en repos grandes | Alta (indice de Git) | Variable |
| Busqueda en historial | Si (tags, commits) | No |
| Patrones en .gitignore | Respetados | No |
| Regex de Git (PCRE) | Si (`-P`) | Depende de la version |

```bash
# Comparativa de velocidad en repo con 100k archivos:
time git grep "pattern"         # ~0.5s
time grep -r "pattern" .        # ~15s
```

---

## 17.4 git reflog (Profundizacion)

El reflog registra cada cambio en la posicion de HEAD y otras referencias locales. Es la red de seguridad definitiva de Git.

### 17.4.1 Todo lo que HEAD ha Apuntado

```bash
# Ver reflog de HEAD
git reflog

# Salida tipica:
# abc1234 HEAD@{0}: commit: Add login feature
# def5678 HEAD@{1}: rebase (finish): returning to refs/heads/feature
# ghi9012 HEAD@{2}: checkout: moving from main to feature/login
# jkl3456 HEAD@{3}: merge feature/profile: Merge made by ort strategy
```

### 17.4.2 Recuperar Commits "Perdidos"

```bash
# Escenario: hiciste reset --hard y perdiste commits

# 1. Ver reflog para encontrar el commit perdido
git reflog

# 2. Recuperar el commit
git checkout HEAD@{3}        # Navegar a el
git branch recuperado HEAD@{3}  # Crear rama apuntando a el
git cherry-pick abc1234      # O aplicar el commit especifico
```

### 17.4.3 Recuperar Ramas Borradas

```bash
# Escenario: borraste una rama accidentalmente

# 1. Encontrar el ultimo commit de la rama borrada
git reflog | grep "feature/borrada"

# 2. Recrear la rama
git branch feature/borrada HEAD@{5}

# Alternativa: buscar en reflog de la rama (si existia)
git reflog show feature/borrada  # Puede no funcionar si ya se borro
```

### 17.4.4 Tiempo de Retencion

Por defecto, Git retiene entradas de reflog por **90 dias** para objetos alcanzables y **30 dias** para inalcanzables.

```bash
# Ver configuracion actual
git config gc.reflogExpire         # 90 days
git config gc.reflogExpireUnreachable  # 30 days

# Extender el tiempo de retencion
git config gc.reflogExpire 180.days
```

### 17.4.5 Gestion de Reflogs

```bash
# Ver reflog de una rama especifica
git reflog show main

# Ver reflog con timestamps
git reflog --date=iso

# Expirar entradas antiguas manualmente
git reflog expire --expire=now --all

# Eliminar entradas especificas
git reflog delete HEAD@{2}

# Limpiar completamente
git reflog expire --expire-unreachable=now --all
git gc --prune=now
```

> **Advertencia**: `git reflog expire --expire=now --all` es destructivo. Solo usalo si necesitas liberar espacio en disco y estas seguro de que no necesitas recuperar historial.

---

## 17.5 git archive: Exportar Codigo sin .git

```bash
# Exportar en formato tar
git archive --format=tar --output=release.tar HEAD

# Exportar en formato zip
git archive --format=zip --output=release.zip HEAD

# Exportar desde un tag especifico
git archive --format=zip --output=v1.0.0.zip v1.0.0

# Exportar solo un subdirectorio
git archive --format=tar HEAD:src/ | gzip > src.tar.gz

# Exportar con un prefijo de directorio
git archive --prefix=mi-proyecto/ --format=tar HEAD | gzip > mi-proyecto.tar.gz

# Usar en pipelines de CI/CD
git archive --format=tar HEAD | ssh user@server "tar -x -C /var/www"
```

---

## 17.6 git bundle: Transferir Repositorios sin Red

`git bundle` empaqueta objetos de Git en un solo archivo binario, permitiendo transferir historial entre repositorios a traves de medios offline.

### 17.6.1 Crear, Verificar y Desempaquetar

```bash
# Crear bundle de una rama completa
git bundle create repo.bundle --all

# Crear bundle de commits no enviados al remoto
git bundle create updates.bundle origin/main..HEAD

# Crear bundle de varias ramas
git bundle create feature.bundle main feature/nueva

# Crear bundle por rango de commits
git bundle create hotfix.bundle v1.0.0..hotfix/critico

# Verificar integridad del bundle
git bundle verify repo.bundle
# repo.bundle is okay
# The bundle contains 3 refs:
# abc1234 refs/heads/main
# def5678 refs/heads/feature

# Listar referencias en el bundle
git bundle list-heads repo.bundle

# Desempaquetar (clonar desde bundle)
git clone repo.bundle mi-nuevo-repo

# Agregar bundle como remoto y hacer fetch
git remote add bundle-source repo.bundle
git fetch bundle-source
git merge bundle-source/main
```

### 17.6.2 Casos de Uso

| Escenario | Comando |
|-----------|---------|
| Transferir por USB | `git bundle create mi.bundle --all` |
| Enviar por email | `git bundle create parche.bundle main..feature` |
| Backup offline | `git bundle create backup-$(date +%Y%m%d).bundle --all` |
| Cliente sin red | `git clone /ruta/bundle.repo` |

---

## 17.7 git notes: Metadatos sin Modificar SHA

`git notes` permite adjuntar metadatos a commits existentes sin modificar su hash SHA (no reescribe la historia).

### 17.7.1 Operaciones Basicas

```bash
# Agregar una nota a un commit
git notes add -m "Revisado por QA, aprobado" abc1234

# Agregar nota reutilizando mensaje de archivo
echo "Requiere migracion de BD" | git notes add --file=- abc1234

# Ver notas en el log
git log --show-notes

# Ver nota de un commit especifico
git notes show abc1234
# Revisado por QA, aprobado

# Editar nota existente
git notes edit abc1234

# Eliminar nota
git notes remove abc1234

# Listar todas las notas
git notes list
# abc1234 Revisado por QA, aprobado
# def5678 Requiere migracion de BD
```

### 17.7.2 Compartir Notas

```bash
# Las notas se almacenan en refs/notes/commits
# Se pueden push como cualquier otra referencia

git push origin refs/notes/commits
git fetch origin refs/notes/commits

# Configurar fetch automatico de notas
git config remote.origin.fetch "+refs/notes/*:refs/notes/*"
```

### 17.7.3 Namespaces de Notas

```bash
# Crear notas en namespaces separados (ej: revision vs deploy)
git notes --ref=revision add -m "Pendiente revision" abc1234
git notes --ref=deploy add -m "Desplegado en staging" abc1234

# Ver notas de un namespace especifico
git notes --ref=revision show abc1234

# Listar namespaces
ls .git/refs/notes/
# commits deploy revision
```

> **Tip**: Usa git notes en CI/CD para registrar metadata de deploy (version, entorno, timestamp) sin modificar el historial de commits.

---

## 17.8 git range-diff: Comparar Series de Commits

`git range-diff` compara dos secuencias de commits, util tras un rebase para verificar que los cambios semanticos se preservaron.

```bash
# Antes del rebase: guardar el rango original
git branch backup-feature

# Despues del rebase: comparar version original vs rebaseada
git range-diff origin/main..backup-feature origin/main..feature-rebaseada

# Forma abreviada (si la rama se actualizo con rebase)
git range-diff @{1}...
```

Salida tipica:
```
1:  a1b2c3d ! 1:  e4f5g6h Add login feature
    - Old commit message detail
    + New improved commit message detail

2:  b2c3d4e = 2:  f5g6h7i Add unit tests
    (commit identico, solo cambio el sha)

3:  c3d4e5f < -:  --- Remove unused code
    (commit eliminado en la nueva version)

-:  --- > 3:  g6h7i8j Add rate limiting
    (commit nuevo agregado)
```

---

## 17.9 Comandos Avanzados de Referencia

### git rev-parse

```bash
# Obtener SHA de referencias
git rev-parse HEAD           # abc1234...
git rev-parse main           # def5678...
git rev-parse HEAD~3         # SHA del commit 3 antes de HEAD
git rev-parse v1.0.0^{tree}  # SHA del arbol del tag

# Verificar si una referencia existe
git rev-parse --verify feature/test && echo "Existe"

# Obtener directorio .git
git rev-parse --git-dir      # /ruta/al/proyecto/.git

# Obtener raiz del working tree
git rev-parse --show-toplevel

# Obtener profundidad relativa
git rev-parse --show-cdup    # Cuantos niveles subir al toplevel (../../)

# Nombre corto de la rama actual
git rev-parse --abbrev-ref HEAD

# Verificar si estamos en medio de una operacion
git rev-parse --is-inside-work-tree
```

### git rev-list

`git rev-list` lista objetos commit en orden cronologico inverso. Es el motor tras `git log`.

```bash
# Listar commits entre dos referencias
git rev-list main ^feature  # Commits en main que no estan en feature

# Contar commits
git rev-list --count HEAD

# Commits que modificaron un archivo
git rev-list HEAD -- src/app.js

# Limitar por tiempo
git rev-list --since="2026-01-01" --until="2026-03-31" main

# Commits de un autor especifico
git rev-list --author="Alice" main

# Commits con mensaje que contiene patron
git rev-list --grep="fix:" main

# Limitar cantidad
git rev-list -n 5 HEAD

# Encontrar el ancestro comun mas cercano (merge base)
git rev-list --merges main feature | tail -1
# Equivalente a:
git merge-base main feature
```

---

## 17.10 git shortlog: Resumen de Contribuciones

```bash
# Resumen de commits por autor
git shortlog -sne

# Salida:
#   42  Alice <alice@ejemplo.com>
#   38  Bob <bob@ejemplo.com>
#   15  Carlos <carlos@ejemplo.com>

# Agrupado por autor con titulos de commits
git shortlog

# Filtrado por rango de tiempo
git shortlog --since="6 months ago"

# En un rango de versiones
git shortlog v1.0.0..v1.1.0

# Para generar release notes
git shortlog --no-merges v1.0.0..HEAD
```

---

## 17.11 git describe: Nombre Descriptivo de un Commit

`git describe` genera un identificador humano-legible basado en el tag anotado mas cercano.

```bash
# Descripcion del commit actual
git describe
# v1.2.0-14-gabc1234
#  ^tag  ^commits_despues_del_tag ^hash_abreviado

# Descripcion de un commit especifico
git describe abc1234

# Solo tags anotados (default)
git describe

# Incluir tags ligeros tambien
git describe --tags

# Siempre incluir hash abreviado
git describe --always

# Nombre del tag mas cercano sin sufijos
git describe --abbrev=0
# v1.2.0

# Excluir ciertos patrones de tag
git describe --match "v[0-9]*" --exclude "*-rc*"

# Uso en sistemas de build para generar version strings
VERSION=$(git describe --tags --always --dirty)
echo "const VERSION = '${VERSION}';" > version.js
```

> **Tip**: `git describe --tags --always --dirty` es ideal para scripts de build. `--dirty` agrega el sufijo `-dirty` si hay cambios sin commitear, alertando que el build no es de codigo limpio.

---

## 17.12 git worktree: Múltiples Ramas Simultáneas

`git worktree` permite tener **múltiples working trees** desde un mismo repositorio, cada uno en una rama diferente. Es como tener varios clones que comparten el mismo `.git`, ahorrando espacio y permitiendo trabajo paralelo real.

### 17.12.1 Operaciones Básicas

```bash
# Crear un nuevo worktree en una rama existente
git worktree add ../proyecto-hotfix hotfix/urgente

# Crear un nuevo worktree con una nueva rama
git worktree add -b feature/experimental ../proyecto-experimental main

# Listar todos los worktrees
git worktree list
# /Users/alice/proyecto          abc1234 [main]
# /Users/alice/proyecto-hotfix   def5678 [hotfix/urgente]
# /Users/alice/proyecto-exp      ghi9012 [feature/experimental]

# Eliminar un worktree (y limpiar referencias)
git worktree remove ../proyecto-hotfix

# Limpiar worktrees eliminados manualmente (sin git worktree remove)
git worktree prune
```

### 17.12.2 Casos de Uso

| Escenario | Workflow |
|-----------|----------|
| **CI/CD en paralelo** | Cada job de CI checkout en su propio worktree sin interferir |
| **Dos features simultáneas** | `feature/auth` en worktree A, `feature/api` en worktree B |
| **Hotfix sin stashear** | Trabajas en feature, surge hotfix → worktree nuevo en main sin perder tu estado |
| **Code review local** | Un worktree con la rama del PR para probar sin cerrar tu editor |
| **Build sin contaminar** | Worktree limpio para `npm run build` mientras editas en otro |

```bash
# Escenario: surge un hotfix mientras trabajas en feature
# Worktree principal: feature/login (tienes cambios sin commitear)
git worktree add -b hotfix/critico ../proyecto-hotfix main
cd ../proyecto-hotfix
# Corriges el bug, commiteas, pusheas
git push origin hotfix/critico
# Vuelves a tu worktree original: todo sigue como estaba
cd ../proyecto
git worktree remove ../proyecto-hotfix
```

> **Ventaja clave:** A diferencia de `git stash`, con worktree no pierdes el contexto de tu editor abierto, archivos sin guardar, ni el estado mental de tu tarea actual.

---

## 17.13 `git log -S` y `git log -G`: Búsqueda Pickaxe

El "pickaxe search" busca en el **contenido** de los diffs, no en los mensajes de commit. Es ideal para encontrar qué commit introdujo o eliminó una función, variable o string específico:

```bash
# -S: Busca commits donde el número de ocurrencias del string CAMBIÓ
# (apareció o desapareció)
git log -S "handlePaymentError"
# Encuentra el commit que introdujo (o eliminó) handlePaymentError

# -G: Busca commits cuyo diff CONTIENE el patrón regex
# (más flexible que -S, detecta cambios parciales)
git log -G "function\s+handlePayment"
# Encuentra commits donde se modificó código que contiene ese patrón

# Combinar con otras opciones de log
git log -S "deprecatedFunction" --oneline --all
git log -G "TODO" --since="3 months ago" --author="Alice"
git log -S "API_KEY" -- src/config/  # Solo en ese directorio
```

**Diferencia clave entre -S y -G:**
- `-S "string"`: Detecta cambios en el **número de ocurrencias** del string (introducción o eliminación).
- `-G "regex"`: Detecta **cualquier diff** que contenga el patrón (incluso si la línea simplemente se modificó sin añadir/eliminar el string).

```bash
# Caso real: encontrar quién eliminó una función de seguridad
git log -S "validateToken" --oneline --all
# abc1234 refactor(auth): extraer validación de token
# El commit abc1234 eliminó la función validateToken
git show abc1234  # Revisar qué se hizo exactamente
```

---

## 17.14 `git interpret-trailers`: Parseo de Trailers de Commit

Los "trailers" son los metadatos al final del mensaje de commit (`Signed-off-by:`, `Co-authored-by:`, `BREAKING CHANGE:`). `git interpret-trailers` los parsea y manipula:

```bash
# Ver trailers de un commit
git log -1 --format="%(trailers)" HEAD
# Co-authored-by: Bob <bob@ejemplo.com>
# Signed-off-by: Alice <alice@ejemplo.com>

# Agregar un trailer a un commit existente (sin modificar SHA si se usa git notes)
git interpret-trailers --trailer "Reviewed-by: Carlos <carlos@ejemplo.com>" <<EOF | git commit --amend -F -
$(git log -1 --format=%B HEAD)
EOF

# Extraer solo trailers en formato clave: valor
git log -1 --format="%(trailers:key=Co-authored-by,valueonly)" HEAD
# Bob <bob@ejemplo.com>

# Filtrar commits por trailer específico (ej: todos los commits firmados)
git log --grep="Signed-off-by:"
```

---

## 17.15 `git send-email` y `git format-patch`: Workflow Estilo Kernel

El kernel de Linux y proyectos como Git, U-Boot y QEMU usan patches por email en lugar de PRs. `git format-patch` genera archivos de patch; `git send-email` los envía por SMTP:

```bash
# Generar archivos .patch de los últimos 3 commits
git format-patch -3 --cover-letter
# 0000-cover-letter.patch
# 0001-feat-add-login.patch
# 0002-fix-typo.patch
# 0003-docs-update.patch

# Generar patches contra una rama upstream
git format-patch origin/main..feature --cover-letter

# Editar la cover letter para describir la serie de patches
vim 0000-cover-letter.patch

# Enviar patches por email
git send-email --to="mantainer@kernel.org" --cc="lista@kernel.org" 000*.patch
# O configurar sendemail para envío automático
git config sendemail.smtpserver smtp.gmail.com
git config sendemail.smtpuser alice@gmail.com
git config sendemail.smtpencryption tls
git config sendemail.smtpserverport 587
```

| Herramienta | Propósito |
|-------------|-----------|
| `git format-patch` | Genera archivos `.patch` con commits formateados como emails |
| `git send-email` | Envía patches por SMTP (similar a `mutt` pero integrado con Git) |
| `git am` | Aplica patches recibidos por email (opuesto a `format-patch`) |
| `git request-pull` | Genera un resumen de cambios para pedir pull (usado en Linux) |

---

## 17.16 `git range-diff` Avanzado

Ampliando la sección 17.8, `git range-diff` tiene opciones avanzadas:

```bash
# Ajustar la sensibilidad de detección de commits similares
# --creation-factor <porcentaje>: qué tan similares deben ser dos commits
# para considerarse "el mismo". Default: 60. Mayor = más estricto.
git range-diff --creation-factor=90 origin/main..old origin/main..new

# Caso: tras rebase masivo de 200 commits, verificar integridad
git range-diff --creation-factor=80 main@{1}..feature@{1} main..feature

# Integración con CI: validar que un rebase no perdió cambios
git range-diff origin/main..HEAD@{1} origin/main..HEAD
if [ $? -ne 0 ]; then
    echo "ADVERTENCIA: El rebase puede haber perdido cambios"
fi

# Mostrar diff del diff (qué cambió dentro de cada commit)
git range-diff --diff-filter=M main@{1}..feature main..feature
```

---

## 17.18 Git Trace2 — Perfilado de Rendimiento de Git

Git 2.22+ incluye un sistema de tracing de alto rendimiento llamado Trace2. A diferencia de `GIT_TRACE` (deprecado para diagnósticos, demasiado verboso), Trace2 está diseñado para usarse en producción sin degradar el rendimiento. Es tu herramienta principal cuando un `git status` tarda 8 segundos y no sabes por qué.

### Los Tres Modos de Trace2

```bash
# 1. GIT_TRACE2 (modo texto legible por humanos)
GIT_TRACE2=/tmp/git-trace.txt git status
cat /tmp/git-trace.txt
# Muestra: inicio, fin, duración de cada fase, regiones de código

# 2. GIT_TRACE2_PERF (métricas de rendimiento para análisis)
GIT_TRACE2_PERF=/tmp/git-perf.txt git status
# Muestra: tiempos exactos, contadores de operaciones, I/O stats

# 3. GIT_TRACE2_EVENT (formato JSON estructurado para herramientas)
GIT_TRACE2_EVENT=/tmp/git-event.json git status
# Muestra: eventos en JSON, parseables por scripts/herramientas
```

### Diagnóstico: ¿Por Qué Mi git status Es Tan Lento?

```bash
# Activar Trace2 con timestamp de alta precisión
export GIT_TRACE2_PERF=/tmp/git-perf.log
export GIT_TRACE2_PERF_BRIEF=1  # Formato compacto
export GIT_TRACE2_PERF_FLUSH=1  # Escribir inmediatamente (no buffer)

# Ejecutar el comando problemático
time git status  # Mide el tiempo real

# Analizar resultados
cat /tmp/git-perf.log

# Ejemplo de salida:
# d0 | main                     | region_enter | .. | index | label:do_read_index
# d0 | main                     | region_leave | .. | index | label:do_read_index | t:2345ms
#                                                                    ^^^^^^^^
#                                                     ¡El índice tardó 2.3 segundos!

# Si el índice es grande (>100MB), considera:
#   - sparse-index (Git 2.34+)
#   - git maintenance (optimizaciones programadas)
#   - core.untrackedCache = true

# Si es el status de archivos no rastreados:
# d0 | main                     | region_enter | .. | status | label:untracked
# d0 | main                     | region_leave | .. | status | t:4500ms
#                                                                ^^^^^^^^
#                                                           4.5 segundos

# Solución: usar un .gitignore más agresivo o sparse checkout
```

### Diagnóstico de Operaciones de Red Lentas

```bash
# ¿Tu git fetch tarda 30 segundos? Trace2 te dice dónde:

GIT_TRACE2_EVENT=/tmp/fetch-event.json git fetch origin

# Analizar con Python
python3 << 'EOF'
import json

with open('/tmp/fetch-event.json') as f:
    events = [json.loads(line) for line in f if line.strip()]

# Encontrar fases de red
for e in events:
    if e.get('event') == 'data' and 'http' in str(e.get('key', '')):
        print(f"[{e['time']}] HTTP: {e['value']}")
    
    if e.get('event') == 'region_leave':
        region = e.get('region', '')
        duration = e.get('t_rel', 0)
        if duration > 1000:  # Más de 1 segundo
            print(f"⚠️  Región lenta: {region} = {duration}ms")

# Problemas comunes detectables:
# - DNS resolution lento (>500ms) → configurar DNS cache
# - TLS handshake lento (>1s) → conexiones persistentes
# - HTTP redirect → usar URL directa en remote
# - Gran número de refs (>10000) → refspec más restrictivo
EOF
```

### Script de Diagnóstico Automatizado para CI/CD

```bash
#!/bin/bash
# diagnose-slow-git.sh: Ejecutar en CI cuando los jobs de Git son lentos
# Uso: ./diagnose-slow-git.sh git fetch origin

TRACE_FILE="/tmp/git-trace-$$.json"
export GIT_TRACE2_EVENT="$TRACE_FILE"
export GIT_TRACE2_EVENT_BRIEF=1

echo "🔍 Ejecutando con Trace2: $*"
"$@"
EXIT_CODE=$?

if [ -f "$TRACE_FILE" ]; then
    echo ""
    echo "📊 Resumen de Trace2:"
    
    # Duración total
    python3 -c "
import json, sys
events = [json.loads(l) for l in open('$TRACE_FILE') if l.strip()]
start = events[0].get('time', 0)
end = events[-1].get('time', 0)
duration_ms = (end - start) * 1000
print(f'  Duración total: {duration_ms:.0f}ms')

# Regiones más lentas
regions = {}
for e in events:
    if e.get('event') == 'region_leave':
        reg = e.get('region', 'unknown')
        dur = e.get('t_rel', 0)
        if dur > 100:  # Solo mostrar >100ms
            regions[reg] = max(regions.get(reg, 0), dur)

for reg, dur in sorted(regions.items(), key=lambda x: -x[1]):
    print(f'  Región: {reg:30s} → {dur:.0f}ms')
"
fi

exit $EXIT_CODE
```

---

## 17.19 Git Forense — Auditoría y Análisis de Repositorios

Como arquitecto, eventualmente necesitarás responder preguntas como: "¿Quién modificó este archivo sensible entre marzo y junio de 2023?" o "¿Qué archivos tocó el desarrollador que renunció en su última semana?" Git forense es el arte de extraer estas respuestas.

### Consulta 1: ¿Quién Tocó un Archivo en un Rango de Fechas?

```bash
# Todos los commits que modificaron auth/tokens.py entre enero y marzo 2024
git log --all --since="2024-01-01" --until="2024-03-31" \
    --follow -- auth/tokens.py \
    --format="%h %ad %an: %s" --date=short

# Con estadísticas de cambios:
git log --all --since="2024-01-01" --until="2024-03-31" \
    --follow -- auth/tokens.py \
    --format="%h %ad %an" --date=short \
    --numstat

# Para múltiples archivos a la vez:
git log --all --since="2024-01-01" --until="2024-03-31" \
    -- '*.env' '*.pem' 'config/secrets.*' \
    --format="%h %ad %an: %s" --date=short
```

### Consulta 2: ¿Qué Hizo un Desarrollador Específico en Su Última Semana?

```bash
# Todo lo que "maria.garcia@empresa.com" tocó en su última semana
git log --all --author="maria.garcia@empresa.com" \
    --since="2024-06-01" --until="2024-06-07" \
    --format="%h %ad %s" --date=short \
    --name-only

# Con diff completo (para revisar exactamente qué cambió)
git log --all --author="maria.garcia@empresa.com" \
    --since="2024-06-01" --until="2024-06-07" \
    -p

# Solo archivos modificados, con conteo:
git log --all --author="maria.garcia@empresa.com" \
    --since="2024-06-01" --until="2024-06-07" \
    --format="" --name-only | sort | uniq -c | sort -rn
```

### Consulta 3: ¿En Qué Momento Entró un Código Específico?

```bash
# Usar pickaxe (-S) para encontrar cuándo se introdujo una string específica
git log --all -S "API_KEY_SECRET" --source --format="%h %ad %an: %s" --date=short

# Encontrar cuándo se ELIMINÓ una string
git log --all -S "DEBUG_MODE=true" --diff-filter=D \
    --format="%h %ad %an: %s" --date=short

# Buscar con regex (-G) en diffs
git log --all -G "password\s*=\s*['\"]" \
    --format="%h %ad %an: %s" --date=short
```

### Consulta 4: Estado del Repositorio en una Fecha de Auditoría

```bash
# ¿Qué archivos existían en main el 31 de diciembre de 2023?
git ls-tree -r --name-only $(git rev-list -n1 --before="2024-01-01" main)

# ¿Qué contenía el README en esa fecha?
git show $(git rev-list -n1 --before="2024-01-01" main):README.md

# Reconstruir el árbol completo de archivos para auditoría
AUDIT_DATE="2024-03-31"
COMMIT=$(git rev-list -n1 --before="$AUDIT_DATE" main)

echo "=== Estado del repositorio al $AUDIT_DATE ==="
echo "Commit: $COMMIT"
echo ""
echo "--- Archivos y sus hashes ---"
git ls-tree -r $COMMIT | while read mode type hash path; do
    echo "$hash $path"
done

echo ""
echo "--- Personas con commits en Q1 2024 ---"
git shortlog -sn --since="2024-01-01" --until="$AUDIT_DATE"
```

### Consulta 5: Auditoría de Seguridad — Archivos con Secretos en el Historial

```bash
#!/bin/bash
# audit-secrets.sh: Buscar patrones de secretos en TODO el historial
# ADVERTENCIA: Lento en repos grandes. Usar con --since si es posible.

PATRONES=(
    "sk_live_[0-9a-zA-Z]{24,}"           # Stripe secret key
    "ghp_[0-9a-zA-Z]{36}"                # GitHub personal access token
    "-----BEGIN RSA PRIVATE KEY-----"    # Clave privada SSH/PEM
    "AKIA[0-9A-Z]{16}"                   # AWS Access Key ID
    "password\s*=\s*['\"][^'\"]+['\"]"  # password en texto plano
    "api_key\s*=\s*['\"][^'\"]+['\"]"   # api_key en texto plano
    "token\s*=\s*['\"][^'\"]+['\"]"     # token genérico
)

echo "🔍 Auditando repositorio en busca de secretos..."
echo ""

for patron in "${PATRONES[@]}"; do
    echo "--- Buscando: $patron ---"
    git log --all -G "$patron" \
        --format="  %h %ad %an: %s" \
        --date=short 2>/dev/null || echo "  (sin resultados)"
    echo ""
done

echo "✓ Auditoría completada"
echo "⚠️  Si encontraste secretos: RÓTALOS INMEDIATAMENTE."
echo "    Eliminarlos del historial NO ES SUFICIENTE."
```

---

## 17.20 Scripting Avanzado con Git Plumbing

Los comandos "porcelain" (log, show, diff) son para humanos. Los "plumbing" (rev-list, cat-file, ls-tree) son para scripts. Aquí tienes patrones avanzados que usarás como arquitecto.

### Script 1: Generar Reporte de Actividad por Desarrollador para Gerencia

```bash
#!/bin/bash
# dev-activity-report.sh: Reporte de actividad por desarrollador
# Uso: ./dev-activity-report.sh 2024-01-01 2024-03-31

SINCE="${1:-2024-01-01}"
UNTIL="${2:-2024-03-31}"

echo "=============================================="
echo "  REPORTE DE ACTIVIDAD: $SINCE → $UNTIL"
echo "=============================================="
echo ""

# Por cada autor único en el período
git shortlog -sne --since="$SINCE" --until="$UNTIL" --all | \
while read count name email; do
    email_clean=$(echo $email | tr -d '<>')
    name_clean=$(echo $name)
    
    echo "👤 $name_clean <$email_clean>"
    
    # Commits totales
    commits=$(git rev-list --count --since="$SINCE" --until="$UNTIL" \
        --author="$email_clean" --all)
    echo "   Commits: $commits"
    
    # Archivos modificados (únicos)
    files=$(git log --since="$SINCE" --until="$UNTIL" \
        --author="$email_clean" --all \
        --format="" --name-only | sort -u | wc -l | tr -d ' ')
    echo "   Archivos únicos modificados: $files"
    
    # Líneas añadidas/eliminadas
    stats=$(git log --since="$SINCE" --until="$UNTIL" \
        --author="$email_clean" --all \
        --format="" --shortstat | awk '
            /insertions/ { ins += $4 }
            /deletions/ { del += $6 }
            END { printf "+%d -%d", ins, del }
        ')
    echo "   Líneas: $stats"
    
    # Ramas creadas
    branches=$(git branch -r --format='%(refname:short)' | \
        while read branch; do
            git log --since="$SINCE" --until="$UNTIL" \
                --author="$email_clean" --oneline origin/$branch 2>/dev/null | head -1
        done | wc -l | tr -d ' ')
    echo "   Ramas con actividad: $branches"
    echo ""
done

echo "---"
echo "Total autores activos: $(git shortlog -sne --since="$SINCE" --until="$UNTIL" --all | wc -l | tr -d ' ')"
```

### Script 2: Detector de Hotspots — Archivos Más Conflictivos

```bash
#!/bin/bash
# hotspots.sh: Encuentra archivos que cambian frecuentemente
# (candidatos para refactorizacion o modularizacion)

echo "🔍 Analizando hotspots (archivos que cambian frecuentemente)..."
echo ""

# Top 20 archivos más modificados en el último año
echo "--- Top 20 archivos más modificados (último año) ---"
git log --since="1 year ago" --format="" --name-only --all | \
    sort | uniq -c | sort -rn | head -20 | \
    while read count file; do
        # Autores que modificaron este archivo
        authors=$(git log --since="1 year ago" --all --format="%an" -- "$file" | \
            sort -u | wc -l | tr -d ' ')
        echo "  $count cambios | $authors autores | $file"
    done

echo ""
echo "--- Archivos con más autores (conocimiento disperso) ---"

# Archivos que más personas han modificado (knowledge silos)
git log --since="1 year ago" --all --format="" --name-only | \
    sort -u | while read file; do
        [ -z "$file" ] && continue
        authors=$(git log --since="1 year ago" --all --format="%an" -- "$file" 2>/dev/null | \
            sort -u | wc -l | tr -d ' ')
        if [ "$authors" -gt 5 ]; then
            changes=$(git log --since="1 year ago" --all --oneline -- "$file" 2>/dev/null | wc -l | tr -d ' ')
            echo "  $changes cambios | $authors autores | $file"
        fi
    done | sort -t'|' -k2 -rn | head -15
```

### Script 3: Análisis de Acoplamiento entre Módulos

```bash
#!/bin/bash
# coupling-analysis.sh: Mide cuántas veces se modifican juntos dos archivos
# (alta co-modificación = posible acoplamiento)

SINCE="6 months ago"

echo "🔗 Análisis de co-modificación (archivos que cambian juntos)"
echo "   Período: $SINCE"
echo ""

# Obtener todos los commits con sus archivos modificados
git log --since="$SINCE" --all --format="COMMIT %H" --name-only | \
awk '
/^COMMIT/ { commit = $2; next }
/^$/ { next }
{
    files[commit] = files[commit] ? files[commit] "," $0 : $0
}
END {
    for (commit in files) {
        split(files[commit], archivos, ",")
        for (i in archivos) {
            for (j in archivos) {
                if (i < j) {
                    pair = archivos[i] "," archivos[j]
                    pairs[pair]++
                }
            }
        }
    }
    # Mostrar pares que cambian juntos frecuentemente
    for (pair in pairs) {
        if (pairs[pair] > 3) {
            printf "%d co-cambios: %s\n", pairs[pair], pair
        }
    }
}' | sort -rn | head -20
```

---

## 17.21 Scripting de Recuperación y Automatización

Ampliando la sección 17.1, `git bisect` soporta términos arbitrarios más allá de "good/bad":

```bash
# En lugar de "good" y "bad", usar "fast" y "slow" (para regresiones de rendimiento)
git bisect start --term-old=fast --term-new=slow
git bisect slow HEAD
git bisect fast v2.0.0
git bisect run ./benchmark.sh
# abc1234 is the first slow commit

# En lugar de "good" y "bad", usar "working" y "broken"
git bisect start --term-old=working --term-new=broken
git bisect broken HEAD
git bisect working v1.0.0
git bisect run ./test.sh
# def5678 is the first broken commit

# Combinar --term con paths
git bisect start --term-old=passed --term-new=failed -- src/modulo/
git bisect failed HEAD
git bisect passed v3.2.1
git bisect run npm test -- --testPathPattern=modulo
```

**Nota:** `--term-old` y `--term-new` deben usarse siempre juntos. Si omites uno, Git mantiene "good/bad" para el otro.



Los comandos avanzados de Git son herramientas de precision para escenarios complejos. `git bisect` automatiza la busqueda del commit que introdujo un bug mediante busqueda binaria, con soporte para terminos personalizados (`--term-old`/`--term-new`). `git blame` audita el origen de cada linea de codigo. `git grep` ofrece busqueda de alta velocidad en el repositorio, complementado por `git log -S` y `git log -G` (pickaxe) para buscar en el contenido de los diffs. El `reflog` es la red de seguridad para recuperar trabajo perdido. `git worktree` permite trabajar en multiples ramas simultaneamente sin clones adicionales. `git archive` y `git bundle` facilitan la transferencia de codigo. `git notes` anade metadatos sin modificar SHAs. `git range-diff` compara series de commits con control de sensibilidad (`--creation-factor`). `git interpret-trailers` parsea metadata de commits. `git send-email` y `git format-patch` soportan el workflow de patches por email estilo kernel. Los comandos de referencia como `rev-parse`, `rev-list`, `shortlog` y `describe` completan el arsenal del usuario avanzado.

---

## Ejercicios Propuestos

1. **Bisect automatico**: Crea un repositorio con 50 commits donde uno introduce un bug. Escribe un script de test que detecte el bug y usa `git bisect run` para encontrarlo automaticamente. Mide cuantos pasos tomo.

2. **Blame con deteccion de movido**: Mueve una funcion de un archivo a otro en varios commits. Usa `git blame -C -C -C` para rastrear el origen real del codigo movido y verifica que detecta correctamente la procedencia.

3. **Recuperar rama desde reflog**: Elimina intencionalmente una rama con `git branch -D`. Recuperala usando `git reflog` y vuelve a crearla. Luego elimina permanentemente entradas del reflog y confirma que ya no es recuperable.

4. **Bundle y clone offline**: Crea un bundle de una rama con commits locales no publicados. Transfierelo a otro directorio y clonalo. Verifica que el historial, ramas y tags esten completos.

5. **Range-diff post-rebase**: Crea una rama con 5 commits de feature. Haz un rebase interactivo donde combines 2 commits, edites 1, y agregues 1 nuevo. Usa `git range-diff` para comparar la version original con la rebaseada y describe las diferencias detectadas.

---
