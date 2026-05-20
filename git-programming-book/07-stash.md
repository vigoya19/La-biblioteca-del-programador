# Capitulo 7: Stashing y Trabajo Temporal

Durante el desarrollo diario, es comun encontrarse en situaciones donde necesitas cambiar de contexto rapidamente: estas a mitad de una feature, llega un bug urgente, y no quieres hacer un commit a medio terminar. El **stash** de Git resuelve este problema permitiendote guardar cambios temporales y recuperarlos cuando los necesites. Complementariamente, `git worktree` te permite trabajar en multiples ramas simultaneamente sin necesidad de stashear o clonar. Este capitulo cubre ambas herramientas y sus casos de uso practicos.

## 7.1 El Problema que Resuelve `git stash`

Imagina este escenario:

```bash
# Estas trabajando en una feature compleja
git checkout -b feature/nuevo-modulo

# Modificaste varios archivos...
echo "nuevo codigo" >> src/module.js
echo "tests" >> test/module_test.js

# Pero NO has commiteado
git status
# modified: src/module.js
# modified: test/module_test.js

# De repente, bug urgente en produccion! Necesitas cambiar a main YA
git checkout main
# error: Your local changes to the following files would be overwritten
# by checkout: src/module.js
# Please commit your changes or stash them before you switch branches.
```

Git te bloquea el checkout porque tus cambios locales se perderian. Tienes tres opciones:

1. **Hacer commit** de los cambios a medio terminar (ensucia el historial).
2. **Descartar los cambios** con `git restore .` (pierdes el trabajo).
3. **Usar `git stash`** para guardar los cambios temporalmente.

`git stash` es la solucion profesional: guarda tus cambios en una pila temporal, limpia tu working directory, y te permite cambiar de contexto. Cuando terminas con el bug, recuperas tus cambios exactamente como estaban.

## 7.2 Operaciones Basicas con Stash

### 7.2.1 Guardar Cambios: `git stash push`

```bash
# Guardar todos los cambios rastreados (modificados + en staging)
git stash

# Equivalente explicito:
git stash push

# Guardar con un mensaje descriptivo (RECOMENDADO)
git stash push -m "WIP: refactor de modulo de usuarios, paso 2"

# Guardar incluyendo archivos no rastreados (untracked)
git stash push -u
git stash push --include-untracked

# Guardar TODO: rastreados, no rastreados E ignorados
git stash push -a
git stash push --all
```

Por defecto, `git stash` solo guarda:
- Archivos modificados bajo control de versiones.
- Archivos en el area de staging (commiteables).

**No** guarda por defecto:
- Archivos no rastreados (nuevos archivos sin `git add`).
- Archivos en `.gitignore` (ignorados).

> **Recomendacion:** Siempre usa `git stash push -m "descripcion"` con un mensaje claro. Cuando tienes multiples stashes, los mensajes genericos "WIP on feature-x" generados por Git no te ayudaran a identificar cual es cual semanas despues.

### 7.2.2 Listar Stashes: `git stash list`

```bash
# Ver todos los stashes guardados
git stash list

# Salida tipica:
# stash@{0}: On feature/nuevo-modulo: WIP: refactor de usuarios, paso 2
# stash@{1}: On main: fix rapido de typo en README
# stash@{2}: WIP on feature/api: avance en endpoints

# Ver con mas detalle (fechas, hashes)
git stash list --date=relative
# stash@{0}: On feature/nuevo-modulo: WIP: refactor... (2 days ago)

# Formato personalizado
git stash list --format="%gd: %h %s (%cr)"
```

Los stashes se almacenan como una **pila** (stack): `stash@{0}` es el mas reciente, `stash@{1}` el anterior, etc. Piensa en ello como una pila de platos: el ultimo que pones es el primero que sacas.

### 7.2.3 Recuperar y Eliminar: `git stash pop`

```bash
# Recuperar el stash mas reciente Y eliminarlo de la lista
git stash pop

# Recuperar un stash especifico
git stash pop stash@{2}

# Recuperar intentando restaurar el estado del STAGING AREA
# (sin --index los cambios en staging se pierden, todo va a working dir)
git stash pop --index
```

`pop` hace dos cosas:
1. Aplica los cambios del stash a tu working directory (como `apply`).
2. Elimina el stash de la lista (como `drop`).

**Si `pop` produce conflictos:**

```bash
git stash pop
# CONFLICT (content): Merge conflict in src/module.js
# The stash entry is kept in case you need it again.

# Resuelves los conflictos, pero el stash NO se elimina.
# Debes eliminarlo manualmente:
git stash drop
```

Este comportamiento es una proteccion: Git no elimina el stash si no pudo aplicarlo limpiamente.

### 7.2.4 Recuperar Sin Eliminar: `git stash apply`

```bash
# Aplicar el stash mas reciente (se queda en la lista)
git stash apply

# Aplicar un stash especifico
git stash apply stash@{2}

# Aplicar incluyendo el estado del staging
git stash apply --index
```

`apply` es util cuando quieres aplicar los mismos cambios en multiples ramas. El stash permanece en la lista y puedes reutilizarlo.

> **Diferencia clave:** `pop` = apply + drop (si no hay conflictos). `apply` = solo aplicar, el stash se conserva.

### 7.2.5 Eliminar Stashes: `drop` y `clear`

```bash
# Eliminar el stash mas reciente
git stash drop

# Eliminar un stash especifico
git stash drop stash@{3}

# Eliminar TODOS los stashes (IRREVERSIBLE)
git stash clear
```

| Comando | Efecto | Precaucion |
|---------|--------|------------|
| `git stash pop` | Aplica y elimina el ultimo stash | Si hay conflicto, no elimina |
| `git stash drop` | Elimina un stash sin aplicarlo | Los cambios se pierden para siempre |
| `git stash clear` | Elimina todos los stashes | IRREVERSIBLE |

### 7.2.6 Inspeccionar Stashes: `git stash show`

```bash
# Resumen de archivos modificados en el stash mas reciente
git stash show

# Resumen con estadisticas (lineas agregadas/eliminadas)
git stash show --stat

# Ver el diff completo del stash (lo que realmente contiene)
git stash show -p

# Ver diff de un stash especifico
git stash show -p stash@{2}

# Ver diff de un archivo especifico dentro del stash
git stash show -p stash@{0} -- src/module.js
```

Ejemplo de salida de `git stash show -p`:

```diff
diff --git a/src/module.js b/src/module.js
index 1234567..abcdefg 100644
--- a/src/module.js
+++ b/src/module.js
@@ -10,6 +10,14 @@
   return users.map(u => u.name);
 }

+function filterActiveUsers(users) {
+  return users.filter(u => u.active);
+}
+
+function sortUsersByName(users) {
+  return users.sort((a, b) => a.name.localeCompare(b.name));
+}
+
 module.exports = { getUsers };
```

## 7.3 Stash Parcial y Selectivo

No siempre quieres guardar todos los cambios en el stash. Git permite guardar archivos especificos o seleccionar cambios interactivamente.

### 7.3.1 Stash de Archivos Especificos

```bash
# Guardar solo un archivo especifico
git stash push src/module.js -m "WIP: cambios en module.js"

# Guardar varios archivos especificos
git stash push src/app.js test/app_test.js -m "WIP: app y tests"

# Guardar un directorio completo
git stash push src/components/ -m "WIP: refactor de componentes"
```

Los archivos no especificados permanecen en el working directory sin modificacion.

### 7.3.2 Stash Interactivo (Patch Mode)

El modo `--patch` (o `-p`) te permite seleccionar **hunks** (bloques de cambios) especificos para guardar:

```bash
# Modo interactivo: Git te pregunta hunk por hunk
git stash push -p

# O equivalente:
git stash push --patch

# Con mensaje:
git stash push -p -m "WIP: solo cambios de validacion"
```

Git muestra cada hunk y pregunta:

```
diff --git a/src/app.js b/src/app.js
@@ -20,6 +20,10 @@
   console.log("iniciando app");
+  validateConfig(config);
+  if (!config.isValid) {
+    throw new Error("Configuracion invalida");
+  }
   initDatabase(config.db);

(1/1) Stage this hunk [y,n,q,a,d,e,?]?
```

Opciones disponibles en modo patch:

| Opcion | Significado |
|--------|-------------|
| `y` | Si, incluir este hunk en el stash |
| `n` | No, no incluir este hunk |
| `q` | Salir; no guardar este hunk ni los siguientes |
| `a` | Incluir este hunk y todos los restantes |
| `d` | No incluir este hunk ni los siguientes |
| `e` | Editar manualmente el hunk |
| `s` | Dividir este hunk en partes mas pequeñas |
| `?` | Ayuda de opciones |

### 7.3.3 Stash Solo del Staging Area

```bash
# Guardar SOLO lo que esta en staging (cambios listos para commit)
git stash push --staged

# Caso de uso: Tienes cambios en staging listos, y otros cambios
# en working directory que aun no. Quieres guardar solo los de staging.
git stash push --staged -m "Cambios listos para commit"
```

### 7.3.4 Stash Manteniendo el Staging Area: `--keep-index`

```bash
# Guardar solo cambios del working directory, MANTENIENDO staging intacto
git stash push --keep-index

# Caso de uso critico: quieres probar solo lo que esta en staging
# (sin los cambios sucios del working directory) antes de commitear.
# --keep-index stashea los cambios del working directory pero
# DEJA en el working tree exactamente lo que esta en el index.

# Ejemplo practico:
echo "cambio en WIP" >> src/module.js        # working dir sucio
echo "cambio listo" >> src/module.js         # mas cambios
git add src/module.js                        # solo lo ultimo al staging

git stash push --keep-index -m "Guarda WIP, mantiene staged"

# Ahora tu working tree contiene solo lo que estaba en staging.
# Puedes compilar, probar, y si todo bien:
git commit -m "Cambios listos"

# Luego recuperas el WIP:
git stash pop
```

> **Diferencia con `--staged`:** `--staged` guarda EL staging y limpia TODO. `--keep-index` guarda el working directory pero MANTIENE el staging intacto en el working tree. Son complementarios: `--staged` para guardar lo commiteable; `--keep-index` para probar lo commiteable en aislamiento.

## 7.4 Creando una Rama desde un Stash

`git stash branch` crea una nueva rama a partir del commit donde se creo el stash y aplica los cambios del stash en ella. Es ideal cuando empiezas a experimentar, guardas en stash, y luego decides que esos cambios merecen su propia rama.

### 7.4.1 Uso Basico

```bash
# Crear una rama desde el stash mas reciente
git stash branch feature/experimento

# Crear rama desde un stash especifico
git stash branch feature/wip-recuperado stash@{3}
```

Esto:
1. Crea una nueva rama partiendo del commit donde se hizo el stash.
2. Hace checkout a la nueva rama.
3. Aplica el stash (pop).
4. Si tiene exito, elimina el stash de la lista.

### 7.4.2 Caso de Uso Practico

```bash
# Escenario: Hiciste cambios experimentales en main
echo "codigo experimental" >> src/experiment.js
git stash push -m "Experimento: nuevo algoritmo de cache"

# Una semana despues, decides que el experimento vale la pena
git stash list
# stash@{0}: On main: Experimento: nuevo algoritmo de cache

# Crear rama dedicada
git stash branch feature/experimento-cache stash@{0}

# Ahora estas en feature/experimento-cache
# con los cambios aplicados, listo para trabajar y commitear
git add .
git commit -m "Iniciar experimento de cache con nuevo algoritmo"
```

> **Tip:** Acostumbrate a usar `git stash branch` en lugar de `git stash pop` cuando los cambios del stash estuvieron guardados por mucho tiempo. Si tu rama principal avanzo mucho, `pop` puede causar conflictos dificiles de resolver. `branch` te aisla en una rama basada en el commit original, donde los cambios aplican limpiamente.

## 7.5 El Ciclo de Vida de un Stash

Entender como Git almacena los stashes ayuda a usarlos con confianza:

```bash
# 1. Crear stash
git stash push -m "WIP: modulo de pagos"

# El stash se almacena internamente como 2 o 3 commits:
# - Commit del estado del working directory (modificaciones)
# - Commit del estado del staging area (si habia algo en staging)
# - (Opcional) Commit con archivos untracked (si usaste -u)

# 2. Listar stashes (referencias en .git/refs/stash)
git stash list

# 3. Inspeccionar
git stash show -p

# 4. Recuperar
git stash pop  # o apply

# 5. Si ya no necesitas, eliminar
git stash drop
git stash clear
```

Los stashes son referenciados por `refs/stash` y `refs/stash@{N}`. A diferencia de los commits normales, los stashes no estan asociados a ninguna rama y no aparecen en `git log` a menos que los busques explicitamente.

### 7.5.1 Recuperar Stashes Eliminados (Reflog de Stash)

Si eliminaste un stash con `git stash drop` o `git stash clear` y necesitas recuperarlo, el reflog de la referencia `refs/stash` puede salvarte:

```bash
# Ver el historial completo de la referencia stash (incluso stashes eliminados)
git log -g refs/stash --oneline

# Salida tipica:
# a1b2c3d stash@{0}: WIP on feature: refactor de usuarios
# e4f5g6h stash@{1}: On main: hotfix urgente
# i7j8k9l stash@{2}: WIP on develop: experimento con cache

# Recuperar un stash eliminado desde su hash
git stash apply a1b2c3d

# O ver el diff de un stash eliminado antes de recuperarlo
git stash show -p a1b2c3d

# Una vez aplicado, Git lo re-registra automaticamente
git stash list
```

> **Importante:** `git stash clear` solo elimina las referencias, no los objetos. Mientras `git gc` no haya recolectado los objetos huerfanos (expiran tras 2 semanas por defecto), puedes recuperar stashes eliminados desde el reflog de `refs/stash`. Este es un mecanismo distinto al `reflog` de HEAD o de ramas — es especifico para la pila de stash.



## 7.6 Patrones y Mejores Practicas

### 7.6.1 Nombrado de Stashes

```bash
# MAL: stash sin mensaje (dificil de identificar despues)
git stash

# BIEN: mensaje descriptivo con contexto
git stash push -m "WIP: refactor auth module - extraido JWT logic"

# MEJOR: usa un prefijo consistente
git stash push -m "stash: feature/pagos - validacion de tarjetas (pendiente tests)"
git stash push -m "stash: hotfix/bug-456 - correccion SQL injection"
```

### 7.6.2 Revision Periodica

```bash
# Revisar stashes cada cierto tiempo (acumulan basura)
git stash list

# Script para ver antiguedad de stashes
git stash list --date=relative

# Eliminar stashes viejos que ya no necesitas
git stash drop stash@{5}
```

### 7.6.3 Flujo de Trabajo Recomendado

```
1. git stash push -m "descripcion"   # Guardar trabajo en progreso
2. git checkout main                  # Cambiar a la rama de produccion
3. git pull --rebase                  # Actualizar
4. [resolver bug urgente, commit, push]
5. git checkout feature/original      # Volver a tu rama
6. git stash pop                      # Recuperar tu trabajo
```

### 7.6.4 Lista de Verificacion Antes de Pop

```bash
# Antes de hacer git stash pop, verifica:
git status           # Working directory limpio?
git stash list       # Cual stash vas a recuperar?
git stash show -p    # Que contiene realmente?

# Si tienes cambios locales sin commit, pop puede causar conflictos.
# Mejor hacer stash de lo actual primero:
git stash push -m "WIP: otros cambios"
git stash pop        # Recuperar el stash original
git stash pop        # Recuperar los cambios recien guardados (o apply)
```

### 7.6.5 Stash en Scripts y Automatizacion

```bash
# Guardar y restaurar cambios en un script
#!/bin/bash
# Guardar estado actual
if ! git diff-index --quiet HEAD --; then
  git stash push -m "auto-stash: pre-script $(date)"
  STASHED=true
fi

# ... ejecutar operaciones del script ...

# Restaurar si habia cambios
if [ "$STASHED" = true ]; then
  git stash pop
fi
```

## 7.7 Worktrees: Multiples Ramas Simultaneas

`git worktree` te permite tener **multiples arboles de trabajo** vinculados al mismo repositorio. Cada worktree tiene su propia rama checkout, su propio working directory, y su propio staging area. Es como tener varios clones ligeros que comparten el mismo historial y objetos.

### 7.7.1 Concepto de Worktree

```
Repositorio principal:
  .git/           ← base de datos compartida (objetos, refs)
  main/           ← worktree principal (rama main checkout)
  
Worktree adicional:
  .git/           ← puntero al .git principal (NO es un clon completo)
  feature-x/      ← worktree secundario (rama feature-x checkout)
  
Worktree adicional:
  .git/           ← otro puntero al .git principal
  hotfix/         ← worktree terciario (rama hotfix checkout)
```

Todos los worktrees comparten la misma base de datos de objetos (`.git/objects` y `.git/refs`). Solo difieren en el working directory y el HEAD checkout. Esto hace que los worktrees sean extremadamente eficientes en espacio comparados con clones.

### 7.7.2 Crear y Gestionar Worktrees

```bash
# Crear un nuevo worktree con una nueva rama
git worktree add ../proyecto-hotfix -b hotfix/bug-urgente

# Crear worktree con una rama existente
git worktree add ../proyecto-feature feature/nuevo-modulo

# Crear worktree en modo detached HEAD (commit especifico)
git worktree add ../proyecto-viejo a1b2c3d

# Listar worktrees existentes
git worktree list

# Salida tipica:
# /home/usuario/proyecto         a1b2c3d [main]
# /home/usuario/proyecto-hotfix  d4e5f6g [hotfix/bug-urgente]
# /home/usuario/proyecto-feature e5f6g7h [feature/nuevo-modulo]
```

### 7.7.3 Eliminar y Limpiar Worktrees

```bash
# Eliminar un worktree (requiere que no tenga cambios sin commit)
git worktree remove ../proyecto-hotfix

# Forzar eliminacion (aunque tenga cambios locales)
git worktree remove --force ../proyecto-hotfix

# Limpiar metadatos de worktrees que fueron eliminados manualmente
git worktree prune

# Ver worktrees huerfanos antes de limpiar
git worktree prune --dry-run
git worktree prune -v  # verbose: muestra que elimina
```

> **Importante:** Si borras un worktree manualmente (`rm -rf`), Git no se entera hasta que ejecutas `git worktree prune`. Siempre prefiere `git worktree remove` para una limpieza adecuada.

### 7.7.4 Bloquear y Desbloquear Worktrees

```bash
# Bloquear un worktree para evitar que sea podado automaticamente
git worktree lock ../proyecto-hotfix

# Especificar motivo del bloqueo
git worktree lock ../proyecto-hotfix --reason "En despliegue - no tocar"

# Desbloquear
git worktree unlock ../proyecto-hotfix
```

Los worktrees bloqueados no seran eliminados por `git worktree prune`. Util cuando un worktree esta en un dispositivo externo (USB, red) o en uso por un proceso automatizado.

### 7.7.5 Mover y Reparar Worktrees (Git 2.38+)

```bash
# Mover un worktree a una nueva ubicacion
git worktree move ../proyecto-hotfix ../nueva-ruta/hotfix

# Reparar referencias de worktree si el repositorio principal se movio
# o si las rutas internas quedaron desincronizadas
git worktree repair

# Reparar worktrees especificos
git worktree repair ../proyecto-hotfix ../proyecto-feature
```

`git worktree repair` corrige los enlaces internos cuando mueves manualmente el repositorio principal o los worktrees. Los archivos `.git` dentro de cada worktree contienen rutas absolutas que pueden romperse al mover directorios; `repair` las actualiza.

### 7.7.6 Restricciones de Worktrees

- **No puedes tener la misma rama checkout en dos worktrees distintos.** Git lo bloquea para evitar conflictos de edicion paralela en la misma rama.
- Los worktrees no se empujan ni se comparten; son puramente locales.
- Cada worktree tiene su propio `stash`, `index`, y `HEAD`.

```bash
# Esto falla si main ya esta checkout en otro worktree:
git worktree add ../otro-main main
# fatal: 'main' is already checked out at '/home/usuario/proyecto'
```

### 7.7.7 Worktree con Stash Independiente

Cada worktree tiene su propia pila de stash independiente:

```bash
# En worktree A
cd ../proyecto-feature
echo "cambio en feature" >> feature.txt
git stash push -m "WIP en feature"

# En worktree B (stash separado)
cd ../proyecto-hotfix
git stash list  # No muestra el stash del worktree A
echo "cambio en hotfix" >> hotfix.txt
git stash push -m "WIP en hotfix"

# Cada worktree gestiona sus stashes de forma aislada
```

## 7.8 Flujos de Trabajo con Worktrees

### 7.8.1 Desarrollo Paralelo sin Stash

Sin worktree, cambiar entre features requiere stash o commits temporales:

```bash
# Flujo SIN worktree (frustrante)
# En feature/pagos:
git stash push -m "WIP: pagos"
git checkout feature/notificaciones
# ... trabajar en notificaciones ...
git stash push -m "WIP: notificaciones"
git checkout feature/pagos
git stash pop
```

Con worktrees, cada feature tiene su propio directorio permanente:

```bash
# Configuracion inicial (una vez por feature)
git worktree add ../proyecto-pagos -b feature/pagos
git worktree add ../proyecto-notificaciones -b feature/notificaciones

# Trabajo diario: simplemente cambia de terminal/directorio
cd ../proyecto-pagos    # Trabajar en pagos
cd ../proyecto-notificaciones  # Trabajar en notificaciones
# Sin stash, sin commits temporales, sin friccion
```

### 7.8.2 Hotfix sin Interrumpir el Desarrollo

```bash
# Escenario: Estas en feature/compleja y surge un hotfix urgente

# 1. Crear worktree para el hotfix desde el worktree actual
git worktree add ../proyecto-hotfix -b hotfix/critico main

# 2. Trabajar en el hotfix en el nuevo worktree
cd ../proyecto-hotfix
# ... corregir bug, commit, push, crear PR ...

# 3. Mientras tanto, tu feature compleja sigue intacta
cd ../proyecto  # tu worktree original
git status      # tus cambios siguen ahi, sin tocar

# 4. Cuando el hotfix esta en main, sincronizar tu feature
git rebase main  # o git merge main
```

### 7.8.3 Revision de Codigo Local con Worktrees

```bash
# Revisar un PR localmente sin afectar tu trabajo actual

# 1. Fetch de la rama del PR
git fetch origin pull/42/head:pr-42

# 2. Crear worktree para la revision
git worktree add ../proyecto-review-pr42 pr-42

# 3. Revisar, probar, ejecutar tests
cd ../proyecto-review-pr42
npm test
npm run lint

# 4. Cuando terminas, limpiar
cd ../proyecto
git worktree remove ../proyecto-review-pr42
git branch -D pr-42
```

### 7.8.4 Compilacion o Tests en Paralelo

```bash
# Mientras trabajas en feature-x, ejecutar tests de main sin interrupcion

# Crear worktree para main
git worktree add ../proyecto-main main

# En el worktree de main, ejecutar tests largos
cd ../proyecto-main
npm run test:integration  # Tarda 20 minutos

# Mientras, sigues trabajando en tu worktree principal
cd ../proyecto
# ... seguir desarrollando feature-x sin esperar ...
```

## 7.9 Worktree vs Stash vs Clone

Cada herramienta tiene su lugar. Aqui una comparativa para elegir la correcta:

| Aspecto | Worktree | Stash | Clone |
|---------|----------|-------|-------|
| **Espacio en disco** | Minimo (~comparte .git) | Minimo (commits en .git) | Alto (copia completa) |
| **Persistencia** | Permanente | Temporal (pila) | Permanente |
| **Aislamiento** | Medio (mismo .git, distintas ramas) | N/A (es guardado temporal) | Alto (repositorio independiente) |
| **Cambio de contexto** | Cambiar de directorio/terminal | stash → checkout → pop | Cambiar de directorio |
| **Trabajo paralelo** | Si, nativo | No (una rama a la vez) | Si, pero con sincronizacion manual |
| **Uso tipico** | Features paralelas, hotfixes | Interrupciones breves | Entornos aislados, CI/CD |

### Cuando usar cada uno:

**Usa `git stash` cuando:**
- Es una interrupcion breve (minutos u horas).
- Los cambios son experimentales y no sabes si los conservaras.
- Necesitas cambiar de rama rapidamente.
- Estas en medio de algo que no merece un commit.

**Usa `git worktree` cuando:**
- Trabajas en multiples features simultaneamente.
- Necesitas hacer un hotfix sin interrumpir tu feature actual.
- Quieres revisar un PR localmente.
- Ejecutas tareas largas (tests, builds) en paralelo.
- Una feature tomara varios dias y no quieres stashear constantemente.

**Usa `git clone` cuando:**
- Necesitas aislamiento total (diferentes configuraciones).
- Trabajas en entornos CI/CD.
- Necesitas probar con diferentes versiones de dependencias.
- El worktree no es suficiente (necesitas otro remote, otra config).

## 7.10 Worktrees y Herramientas Externas

### 7.10.1 Configuracion de Entorno por Worktree

```bash
# Cada worktree puede tener configuracion especifica
cd ../proyecto-feature
git config user.email "dev@feature.com"
git config core.editor "vim"

cd ../proyecto-hotfix
git config user.email "hotfix@produccion.com"

# Las configuraciones de worktree se almacenan en .git/config
# y son independientes entre worktrees
```

### 7.10.2 Worktrees y Hooks

Los hooks de Git se comparten entre worktrees porque residen en `.git/hooks/` del repositorio principal. Si necesitas hooks diferentes por worktree, puedes usar scripts condicionales:

```bash
# .git/hooks/pre-commit (compartido)
#!/bin/bash
CURRENT_WORKTREE=$(git rev-parse --show-toplevel)
case "$CURRENT_WORKTREE" in
  */proyecto-feature)
    npm run lint:feature
    ;;
  */proyecto-hotfix)
    npm run lint:strict
    ;;
  *)
    npm run lint
    ;;
esac
```

### 7.10.3 Worktrees y Editores/IDEs

```bash
# Abrir cada worktree en una ventana separada de VS Code
code ../proyecto-feature
code ../proyecto-hotfix

# Los worktrees son directorios normales, cualquier editor funciona.
# Cada ventana del editor tiene su propia rama activa.
```

## 7.11 Resolucion de Problemas Comunes

### 7.11.1 "Stash pop causa conflictos"

```bash
# Problema: git stash pop genera conflictos
# Causa: la rama avanzo y los cambios del stash ya no aplican limpiamente

# Solucion 1: Crear rama desde el stash (recomendado)
git stash branch recuperar-cambios stash@{0}

# Solucion 2: Resolver conflictos manualmente
git stash pop
# Resolver conflictos en los archivos
git add .
git stash drop  # El stash no se elimino automaticamente

# Solucion 3: Aplicar stash en un worktree basado en el commit original
COMMIT_BASE=$(git stash list --format="%H" | head -1)
git worktree add ../temp-stash "$COMMIT_BASE"
cd ../temp-stash
git stash pop   # Aplica limpiamente
# Luego cherry-pick o merge a tu rama actual
```

### 7.11.2 "Tengo demasiados stashes acumulados"

```bash
# Ver cuantos stashes tienes
git stash list | wc -l

# Revisar cada uno antes de eliminar
git stash list
for i in $(seq 0 10); do
  echo "=== stash@{$i} ==="
  git stash show -p stash@{$i} | head -20
  echo ""
done

# Limpiar selectivamente
git stash drop stash@{5}
git stash drop stash@{3}

# O limpiar todo (si estas seguro)
git stash clear
```

### 7.11.3 "Stash no guardo mis archivos nuevos"

```bash
# Problema: Creaste archivos nuevos, hiciste git stash, y los archivos no se guardaron
# Causa: git stash por defecto ignora untracked files

# Solucion preventiva:
git stash push -u -m "WIP: incluye archivos nuevos"

# Solucion para stash ya creado (no incluyo los archivos):
# Agrega los archivos al staging primero
git add nuevos-archivos/
git stash push --staged -m "Archivos nuevos que olvide incluir"
```

### 7.11.4 "Worktree no me deja checkout de una rama"

```bash
# Error: fatal: 'main' is already checked out at '/home/...'

# Solucion: Usar una rama diferente, o crear nueva rama
git worktree add ../otro-directorio -b main-copia main

# Ver que worktrees existen y que ramas tienen
git worktree list
```

---

## Resumen del Capitulo 7

- **`git stash`** guarda cambios temporales en una pila, permitiendo cambiar de contexto sin commits a medio terminar.
- **`git stash push -m "mensaje"`** guarda cambios con descripcion; `-u` incluye archivos no rastreados; `-a` incluye ignorados.
- **`git stash pop`** recupera y elimina el ultimo stash; **`git stash apply`** recupera sin eliminar.
- **`git stash list`** lista los stashes; **`git stash show -p`** muestra el contenido detallado de un stash.
- **`git stash drop`** elimina un stash especifico; **`git stash clear`** elimina todos.
- **Stash parcial** permite guardar archivos especificos o usar `-p` para seleccion interactiva hunk por hunk.
- **`git stash branch`** crea una rama desde el commit base del stash y aplica los cambios alli. Ideal para stashes antiguos o experimentales.
- **`git worktree`** permite multiples arboles de trabajo compartiendo el mismo repositorio, cada uno con su propia rama checkout.
- **Worktree** se crea con `git worktree add`, se lista con `git worktree list`, se elimina con `git worktree remove`, y se limpia con `git worktree prune`.
- **Worktree vs Stash vs Clone:** Stash para interrupciones breves; Worktree para features paralelas y hotfixes sin friccion; Clone para aislamiento total.

## Ejercicios Propuestos

1. **Ciclo completo de stash:** Crea una rama `feature` y modifica 3 archivos. Agrega uno al staging, deja otro modificado y crea un archivo nuevo sin rastrear. Practica: (a) `git stash push -u -m "WIP completo"`, (b) verifica que el working directory quedo limpio, (c) cambia a `main`, haz un commit trivial, (d) vuelve a `feature` y `git stash pop`, (e) verifica que los tres archivos (staged, modified, untracked) se recuperaron correctamente.

2. **Stash parcial e interactivo:** Modifica 5 archivos en una rama. Usa `git stash push -p` para guardar solo 2 de los cambios en el stash. Verifica con `git stash show -p` y `git status` que los otros 3 archivos permanecen modificados. Luego usa `git stash push archivo1.txt archivo2.txt` para guardar archivos especificos. Finalmente recupera todo y verifica el estado original.

3. **Worktree para hotfix:** Configura un repositorio con `main` y una rama `feature/larga` donde tengas cambios sin commitear. Crea un worktree para un hotfix urgente usando `git worktree add`. En el worktree del hotfix, corrige el bug, haz commit y push. Vuelve al worktree principal y sincroniza `feature/larga` con los cambios del hotfix usando merge o rebase. Elimina el worktree del hotfix con `git worktree remove`.

4. **Multiples worktrees simultaneos:** Crea 3 worktrees adicionales para simular: una feature nueva, una revision de PR, y una rama de release. En cada worktree, haz al menos un commit. Usa `git worktree list` para ver la configuracion completa. Ejecuta `git log --oneline --all --graph` desde cualquier worktree y verifica que todos los commits son visibles. Practica la eliminacion con `git worktree prune`.

5. **Script de automatizacion de stash:** Escribe un script bash que: (a) verifique si hay cambios sin commit, (b) ofrezca hacer stash automatico con mensaje basado en fecha y rama actual, (c) ejecute una operacion (como `git pull --rebase`), (d) restaure el stash automaticamente si existia. El script debe ser idempotente (puede ejecutarse multiples veces sin causar errores). Incluye manejo de errores para cuando el stash pop produce conflictos.

---

← [Capítulo anterior](06-rebase-cherry-pick.md) | [Inicio](README.md) | [Capítulo siguiente →](08-tags.md)
