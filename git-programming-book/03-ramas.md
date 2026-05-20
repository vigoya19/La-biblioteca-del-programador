# Capitulo 3: Trabajando con Ramas

## 3.1 Que es una Rama

Una rama en Git es simplemente un **puntero movil** que apunta a un commit. Es extraordinariamente liviana: crear una rama nueva es practicamente instantaneo y no consume espacio significativo en disco.

```
main
  │
  ▼
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Commit  │◄───│ Commit  │◄───│ Commit  │
│   a1b2  │    │   c3d4  │    │   e5f6  │
└─────────┘    └─────────┘    └─────────┘
```

Cuando creas una nueva rama, Git simplemente crea un nuevo puntero:

```
main
  │
  ▼
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Commit  │◄───│ Commit  │◄───│ Commit  │
│   a1b2  │    │   c3d4  │    │   e5f6  │
└─────────┘    └─────────┘    └─────────┘
                                  ▲
                                  │
                               feature
```

### Por que usar ramas

| Proposito | Beneficio |
|---|---|
| **Aislar cambios** | Trabajar en una feature sin afectar la rama principal |
| **Experimentar** | Probar ideas sin miedo a romper nada |
| **Colaborar** | Varias personas trabajan en distintas features simultaneamente |
| **Estabilizar** | Mantener una rama de produccion y otra de desarrollo |
| **Hotfixes** | Corregir bugs en produccion sin mezclar features a medio hacer |

> **Dato clave:** A diferencia de SVN o CVS donde crear una rama significa copiar todo el proyecto (operacion costosa), en Git las ramas son solo punteros de 41 bytes (40 del SHA-1 + salto de linea). Crear 100 ramas cuesta lo mismo que crear una.

---

## 3.2 Como Git Almacena las Ramas

### El modelo de datos de Git

Git almacena sus datos como una serie de **snapshots**. Cada commit contiene:

```
┌──────────────────────────────────┐
│           Commit e5f6            │
├──────────────────────────────────┤
│ tree:     7a9b...               │  ← Apunta al arbol raiz
│ parent:   c3d4...               │  ← Commit padre
│ author:   Juan <juan@mail.com>  │
│ committer: Juan <juan@mail.com> │
│ message:  "Agregar login"       │
└──────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│           Tree 7a9b              │
├──────────────────────────────────┤
│ blob: a1b2...   index.html      │  ← Archivo (snapshot)
│ blob: c3d4...   style.css       │
│ tree: e5f6...   src/            │  ← Subdirectorio
└──────────────────────────────────┘
```

### Donde se guardan las ramas

Las ramas locales se almacenan como archivos simples en `.git/refs/heads/`:

```bash
ls .git/refs/heads/
# main
# feature-login
# bugfix-123

# Cada archivo contiene solo el SHA-1 del commit al que apunta
cat .git/refs/heads/main
# e5f6c7d8e9a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5

cat .git/refs/heads/feature-login
# 9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b
```

Las ramas remotas se guardan en `.git/refs/remotes/`:

```bash
ls .git/refs/remotes/origin/
# main
# develop

cat .git/refs/remotes/origin/main
# a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
```

### Ramas y commits: el DAG

Las ramas y commits forman un **Grafo Aciclico Dirigido (DAG)**:

```
        a1b2 ── c3d4 ── e5f6 ── g7h8  ← main
                    \
                     i9j0 ── k1l2    ← feature
```

Cada commit conoce a su(s) padre(s). Un commit puede tener:
- **Cero padres**: commit inicial (raiz)
- **Un padre**: commit normal
- **Dos padres**: merge commit (fucion de dos ramas)

---

## 3.3 HEAD: El Puntero Actual

HEAD es un puntero especial que indica en que rama (o commit) estas trabajando actualmente. Normalmente apunta a una rama, que a su vez apunta a un commit.

```
HEAD → main → commit e5f6
```

### Donde se guarda HEAD

```bash
cat .git/HEAD
# ref: refs/heads/main       (modo normal: apunta a una rama)

# Cuando haces checkout de un commit especifico (detached HEAD):
# e5f6c7d8e9a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5
```

### Estados de HEAD

| Estado | HEAD apunta a | Descripcion |
|---|---|---|
| **Normal** | Una rama | Los nuevos commits avanzan la rama |
| **Detached HEAD** | Un commit directamente | Commits sin rama, se pierden al cambiar |

```bash
# HEAD normal
git checkout main           # HEAD -> main -> commit
git commit -m "fix"         # main avanza, HEAD sigue a main

# Detached HEAD
git checkout e5f6c7d        # HEAD -> commit e5f6 (sin rama)
git commit -m "test"        # El commit queda huerfano si no creas rama
git checkout -b rescate     # Creas rama para rescatar el commit
```

> **Advertencia:** En modo detached HEAD, si haces commits y luego cambias a otra rama sin crear una rama nueva, esos commits quedaran inaccesibles y eventualmente seran eliminados por el garbage collector.

---

## 3.4 Gestion de Ramas

### Listar ramas

```bash
# Ramas locales
git branch
# * main
#   feature-login
#   bugfix-123

# Todas las ramas (locales + remotas)
git branch -a

# Solo ramas remotas
git branch -r

# Con hash abreviado y mensaje del ultimo commit
git branch -v

# Con hash y tracking info (upstream, ahead/behind)
git branch -vv
# * main           e5f6c7d [origin/main] Agregar login
#   feature-login  a1b2c3d [origin/feature-login: ahead 2] Agregar tests
```

### Crear ramas

```bash
# Crear rama (no cambia a ella)
git branch feature-pago
git branch hotfix/login-crash

# Crear en un commit especifico
git branch hotfix e5f6c7d
git branch feature-vieja HEAD~3
```

### Eliminar ramas

```bash
# Eliminar rama fusionada (segura, Git avisa si no esta fusionada)
git branch -d feature-completada

# Forzar eliminacion aunque no este fusionada (PERDIDA DE DATOS)
git branch -D feature-abandonada

# Eliminar rama remota
git push origin --delete feature-completada
git push origin :feature-completada    # Sintaxis alternativa
```

### Renombrar ramas

```bash
# Renombrar rama actual
git branch -m nuevo-nombre

# Renombrar otra rama
git branch -m viejo-nombre nuevo-nombre

# Ejemplo: migrar de master a main (antes de Git 2.28)
git branch -m master main
git push -u origin main
git push origin --delete master
```

---

## 3.5 Cambiar de Rama con git switch

`git switch` es el comando **moderno** (Git >= 2.23) para cambiar de rama. Separa las funciones de cambio de rama del checkout de archivos, haciendo la interfaz mas clara.

### Cambiar a una rama existente

```bash
git switch main
git switch feature-login
```

### Crear y cambiar a una nueva rama

```bash
git switch -c feature-carrito
# Equivalente clasico: git checkout -b feature-carrito

# Crear desde un commit o rama especifica
git switch -c hotfix HEAD~2
git switch -c experimento e5f6c7d

# Forzar cambio descartando cambios locales
# (equivalente a git checkout -f)
git switch -f main
```

### Tabla comparativa: switch vs checkout

| Accion | git switch (moderno) | git checkout (clasico) |
|---|---|---|
| Cambiar de rama | `git switch main` | `git checkout main` |
| Crear y cambiar | `git switch -c feature` | `git checkout -b feature` |
| Volver a la rama anterior | `git switch -` | `git checkout -` |
| Descartar cambios de archivo | `git restore archivo` | `git checkout -- archivo` |

> **Recomendacion:** Usa `git switch` para cambiar de rama y `git restore` para descartar cambios. Son comandos mas intuitivos y menos propensos a errores que `git checkout`.

### Cambiar con trabajo en progreso

```bash
# Si tienes cambios sin commit, Git no te dejara cambiar de rama
# a menos que los cambios no entren en conflicto con la rama destino

# Opcion 1: Hacer stash (guardado temporal)
git stash
git switch feature
# ... trabajar ...
git switch main
git stash pop

# Opcion 2: Commit temporal (luego amend)
git commit -am "WIP: cambios temporales"
git switch feature
git switch main
git reset HEAD~1           # Recupera los cambios a working

# Opcion 3: git worktree (evita stash y commits temporales)
git worktree add ../proyecto-hotfix hotfix-urgente
# Abre un segundo working directory en ../proyecto-hotfix
# en la rama hotfix-urgente. Trabajas sin tocar tu rama actual.
# Al terminar: git worktree remove ../proyecto-hotfix
```

---

## 3.6 Cambiar de Rama con git checkout

`git checkout` es el comando **clasico** para cambiar de rama. Aunque `git switch` es preferible para cambiar de rama, `git checkout` sigue siendo ubicuo en documentacion, scripts y herramientas.

### Cambiar de rama

```bash
git checkout main
git checkout feature-login

# Crear y cambiar a nueva rama
git checkout -b feature-pagos
git checkout -b hotfix-urgente e5f6c7d
```

### Checkout de archivos (restaurar version anterior)

```bash
# Descartar cambios locales y volver al estado del ultimo commit
git checkout -- archivo.txt
git checkout -- src/

# Traer un archivo de otra rama o commit
git checkout feature-login -- src/login.js
git checkout e5f6c7d -- config.old.yml
```

> **Precaucion:** `git checkout -- archivo.txt` **sobrescribe** los cambios locales sin confirmacion. No hay forma de recuperarlos. Si tienes dudas, usa primero `git stash` o `git diff` para revisar.

### Desambiguar checkout

El mismo comando `git checkout` puede cambiar de rama o restaurar archivos. Git decide segun el argumento:

```bash
git checkout main             # Cambiar a rama "main"
git checkout e5f6c7d          # Detached HEAD al commit e5f6c7d
git checkout -- main          # Restaurar ARCHIVO "main" (nota el --)
```

Si existe una rama y un archivo con el mismo nombre, Git prefiere la rama. Usa `--` para forzar el archivo:

```bash
git checkout main      # Cambia a la rama main
git checkout -- main   # Restaura el archivo main
```

---

## 3.7 Fusionar Ramas

`git merge` combina los cambios de una rama en otra. Es una de las operaciones mas importantes de Git.

### Fast-forward merge

Cuando la rama destino NO tiene commits nuevos desde que se creo la rama fuente, Git simplemente mueve el puntero hacia adelante:

```
Antes del merge:
      main
        │
        ▼
  a1b2 ── c3d4
              \
               e5f6 ── g7h8
                         ▲
                         │
                      feature

Despues de git merge feature (fast-forward):
                          main
                           │
                           ▼
  a1b2 ── c3d4 ── e5f6 ── g7h8
```

```bash
git switch main
git merge feature
# Updating c3d4..g7h8
# Fast-forward
```

### Three-way merge

Cuando ambas ramas tienen commits nuevos, Git crea un **merge commit** que une las historias:

```
Antes del merge:
                        main
                         │
                         ▼
  a1b2 ── c3d4 ── e5f6 ── g7h8
              \
               i9j0 ── k1l2
                         ▲
                         │
                      feature

Despues del merge (three-way):
                        main
                         │
                         ▼
  a1b2 ── c3d4 ── e5f6 ── g7h8 ── m3n4 (merge commit)
              \                   /
               i9j0 ─── k1l2 ────
```

```bash
git switch main
git merge feature
# Merge made by the 'ort' strategy.
# Aparece el editor para el mensaje de merge commit
```

### Opciones de git merge

```bash
# Forzar un merge commit incluso si podria ser fast-forward
git merge --no-ff feature

# Solo hacer merge si es fast-forward (abortar si no)
git merge --ff-only feature

# Abortar un merge en progreso (si hay conflictos)
git merge --abort

# Hacer squash: combinar todos los commits de la rama en uno solo
git merge --squash feature
git commit -m "Agregar funcionalidad de pagos"
# El historial de feature no se conserva, solo el resultado final
```

### Estrategias de merge

| Estrategia | Comando | Resultado |
|---|---|---|
| **Fast-forward** | `git merge feature` | Historial lineal, puntero avanza |
| **Merge commit** | `git merge --no-ff feature` | Historial bifurcado, merge commit explicito |
| **Squash** | `git merge --squash feature` | Todos los cambios en un solo commit lineal |

```bash
# Forzar fast-forward aun cuando hay merge commit posible
# (reescribe feature para que este sobre main con rebase, NO merge)
git switch feature
git rebase main
git switch main
git merge feature   # Ahora es fast-forward
```

---

## 3.8 Resolver Conflictos de Merge Basicos

Un conflicto ocurre cuando Git no puede determinar automaticamente como combinar cambios porque dos ramas modificaron las mismas lineas de un archivo.

### Anatomia de un conflicto

Cuando hay conflicto, Git modifica el archivo mostrando ambas versiones:

```
<<<<<<< HEAD
const PORT = 3000;
const DEBUG = true;
=======
const PORT = process.env.PORT || 8080;
const LOG_LEVEL = 'info';
>>>>>>> feature
```

- `<<<<<<< HEAD`: inicio de la version en tu rama actual
- `=======`: separador entre las dos versiones
- `>>>>>>> feature`: fin de la version de la rama que estas fusionando

### Resolver el conflicto

```bash
# 1. Editar el archivo para conservar la version deseada
#    (o una combinacion de ambas)

# 2. Marcar como resuelto
git add archivo-conflictivo.js

# 3. Completar el merge
git commit
# O si prefieres un mensaje especifico:
git commit -m "Resolver conflicto: combinar configuracion de puerto y logging"
```

### Flujo completo de resolucion

```bash
git switch main
git merge feature
# CONFLICT (content): Merge conflict in config.js
# Automatic merge failed; fix conflicts and then commit the result.

# Ver que archivos tienen conflicto
git status
# Unmerged paths:
#   both modified:   config.js

# Ver el conflicto
cat config.js

# Herramienta visual para resolver (si configuraste difftool)
git mergetool

# Despues de editar:
git add config.js
git status
# All conflicts fixed but you are still merging.
# Changes to be committed:
#   modified:   config.js

git commit -m "Merge feature: integracion de nueva configuracion"
```

### Abortar el merge

```bash
# Si el conflicto es demasiado complejo y quieres empezar de nuevo
git merge --abort
# Vuelve al estado exacto antes del merge
```

### Estrategias de resolucion

```bash
# Aceptar solo nuestra version (la rama actual)
git checkout --ours archivo.txt
git add archivo.txt

# Aceptar solo la version de ellos (la rama que mergeas)
git checkout --theirs archivo.txt
git add archivo.txt

# Ver conflictos lado a lado
git diff
git diff --ours
git diff --theirs
git diff --base    # Ancestro comun
```

> **Tip:** Para minimizar conflictos, haz merge frecuentemente de la rama principal a tu rama de feature. Un merge de 10 archivos con 2 conflictos cada dia es mucho mas manejable que un merge de 100 archivos con 50 conflictos al final de la semana.

### Configuracion recomendada: diff3

Por defecto Git muestra marcadores de conflicto con dos secciones (ours / theirs). Configurar `diff3` agrega una tercera seccion que muestra la version **base** (el ancestro comun):

```bash
git config --global merge.conflictStyle diff3
```

Con diff3, los conflictos muestran:

```
<<<<<<< HEAD
const PORT = 3000;
||||||| merged common ancestors
const PORT = 8080;
=======
const PORT = process.env.PORT || 8080;
>>>>>>> feature
```

La seccion `|||||||` es invaluable: muestra como era el codigo antes de que ninguna rama lo modificara, permitiendote entender mejor que cambio hizo cada lado y cual conservar.

---

## 3.9 Ramas Fusionadas y No Fusionadas

Git permite consultar que ramas ya fueron integradas y cuales no, lo cual es util para limpieza:

```bash
# Ramas ya fusionadas en la rama actual
git branch --merged
# * main
#   feature-completada
#   hotfix-enviado

# Ramas NO fusionadas en la rama actual
git branch --no-merged
#   feature-en-progreso
#   experimento

# Ramas fusionadas en una rama especifica
git branch --merged develop

# Ramas no fusionadas en una rama especifica
git branch --no-merged main
```

### Limpieza de ramas

```bash
# Eliminar ramas ya fusionadas (seguro)
git branch --merged | grep -v "\*" | xargs -n 1 git branch -d

# En macOS/Linux:
git branch --merged | grep -v "^\*" | xargs git branch -d

# Forzar eliminacion de todas menos main
git branch | grep -v "main" | xargs git branch -D  # CUIDADO
```

### Navegacion con Referencias Relativas

Git ofrece multiples formas de referenciar commits en el historial:

| Referencia | Significado | Ejemplo (sobre C4) |
|---|---|---|
| `HEAD^` | Primer padre del commit | `HEAD^` = C3 |
| `HEAD~` | Un commit atras (alias de `HEAD~1`) | `HEAD~` = C3 |
| `HEAD~2` | Dos commits atras (primer padre siempre) | `HEAD~2` = C2 |
| `HEAD~2^2` | Dos atras, luego el segundo padre | Util en merge commits |
| `HEAD@{1}` | Donde estaba HEAD en la entrada 1 del reflog | `git reflog` para ver indices |
| `@{u}` o `@{upstream}` | La rama remota de tracking asociada | `main@{u}` = `origin/main` |
| `@{push}` | Donde se hace push esta rama | `main@{push}` = `origin/main` |
| `feature~3..feature` | Rango: commits de feature que no estan en su ancestro 3 atras | Utiles con `git log` |

```bash
# Ejemplos practicos
git log HEAD~5..HEAD          # Ultimos 5 commits
git diff HEAD^ HEAD           # Cambios en el ultimo commit
git rebase -i HEAD~4          # Rebase interactivo de los ultimos 4
git merge feature@{u}         # Merge con la version remota de feature
```

### Convenciones de Nombrado de Ramas

Un esquema consistente de nombres mantiene el repositorio organizado:

| Prefijo | Uso | Ejemplo |
|---|---|---|
| `feature/` | Nueva funcionalidad | `feature/autenticacion-oauth` |
| `bugfix/` | Correccion de bug | `bugfix/login-null-pointer` |
| `hotfix/` | Correccion urgente en produccion | `hotfix/1.2.1-crash-inicio` |
| `release/` | Preparacion de release | `release/2.0.0` |
| `chore/` | Tareas de mantenimiento | `chore/actualizar-dependencias` |
| `experiment/` | Pruebas descartables | `experiment/redis-cache` |
| `docs/` | Cambios de documentacion | `docs/api-guia-instalacion` |

```bash
# Incluir numero de ticket
git checkout -b feature/PROJ-234-agregar-filtros
git checkout -b bugfix/PROJ-567-corregir-off-by-one
```

---

## 3.10 Ramas Remotas

Las ramas remotas son referencias al estado de las ramas en tus repositorios remotos. Son inmutables localmente: solo se mueven cuando haces `git fetch` o `git pull`.

### Ver ramas remotas

```bash
git branch -r
# origin/main
# origin/develop
# origin/feature-login

git remote show origin
# Remote branches:
#   main            tracked
#   develop         tracked
#   feature-login   tracked
# Local branches configured for 'git pull':
#   main    merges with remote main
# Local refs configured for 'git push':
#   main    pushes to main (up to date)
```

### Tracking branches (ramas de seguimiento)

Una rama local que "sigue" a una rama remota se llama **tracking branch**:

```bash
# Al clonar, main automaticamente sigue a origin/main
git branch -vv
# * main e5f6c7d [origin/main] Ultimo commit

# Crear rama local que siga a una remota
git checkout -b feature-login origin/feature-login
# o con switch:
git switch -c feature-login origin/feature-login

# Establecer seguimiento en una rama existente
git branch -u origin/develop
git branch --set-upstream-to=origin/develop
```

### Sincronizacion con remotos

```bash
# Descargar referencias remotas (no modifica ramas locales)
git fetch
git fetch origin
git fetch --all                # Todos los remotos
git fetch --prune              # Eliminar referencias a ramas remotas borradas

# Descargar e integrar (fetch + merge)
git pull
git pull origin main
git pull --rebase              # fetch + rebase en vez de merge

# Enviar cambios
git push origin main
git push -u origin feature     # Push y establecer seguimiento (-u = --set-upstream)
git push --all origin          # Todas las ramas
git push --force               # Peligroso: sobrescribe remoto
git push --force-with-lease    # Mas seguro: verifica que nadie haya pusheado
```

> **Diferencia clave fetch vs pull:** `git fetch` solo descarga datos, permitiendote revisar antes de integrar. `git pull` hace fetch + merge automaticamente. En equipos, muchos prefieren `fetch` + `merge` manual para controlar que entra.

### Ver diferencias con remoto

```bash
# Commits locales que no estan en remoto
git log origin/main..HEAD

# Commits remotos que no tienes localmente
git log HEAD..origin/main

# Resumen
git status -sb
# ## main...origin/main [ahead 2, behind 1]
```

---

## 3.11 git bisect: Encontrar Regresiones

`git bisect` usa **busqueda binaria** para encontrar el commit exacto que introdujo un bug. Dado un rango de commits donde sabes que uno era "bueno" y otro "malo", bisect va probando el punto medio eficientemente hasta aislar el commit culpable.

### Flujo Basico

```bash
# Iniciar sesion de bisect
git bisect start

# Marcar el commit actual como malo (tiene el bug)
git bisect bad

# Marcar un commit conocido como bueno (sin el bug)
git bisect good v1.0.0
# O: git bisect good a1b2c3d

# Git te mueve automaticamente a un commit en el punto medio.
# Pruebas si el bug existe ahi:
# - Si el bug esta presente: git bisect bad
# - Si el bug NO esta presente: git bisect good

# Repetir hasta que Git anuncie:
# a1b2c3d is the first bad commit

# Terminar la sesion (vuelve a la rama original)
git bisect reset
```

### Automatizando bisect

Si puedes escribir un script que verifique el bug (exit 0 = bueno, exit 1-127 = malo):

```bash
git bisect start HEAD v1.0.0
git bisect run npm test -- --testPathPattern=login
# Git ejecuta el script en cada paso automaticamente
# y encuentra el commit culpable sin intervencion manual
```

### Caso practico

```
Commits:  A --- B --- C --- D --- E --- F --- G (HEAD)
           ^bueno                            ^malo

Bisect prueba D → bueno  (bug no esta en D)
           descarta A..D
           prueba F → malo (bug esta en F)
           descarta F..G
           prueba E → malo (bug esta en E)
           RESULTADO: E es el primer commit malo
```

En solo 3 pasos (log2(7) ≈ 3), bisect aislo el commit. En un historial de 1000 commits, solo necesitas ~10 pasos.

> **Tip:** `git bisect` tambien se puede usar para encontrar el commit que **corrigio** un bug: intercambia `good` y `bad`. El commit donde un comportamiento paso de "presente" a "ausente" es el que introdujo la correccion.

---

## 3.12 Estrategias de Merge

Git ofrece multiples estrategias y opciones para controlar exactamente como se fusionan dos ramas. Entenderlas te permite resolver merges complejos con precision quirurgica.

### Estrategia ort (por defecto)

**ORT** (Ostensibly Recursive's Twin) es la estrategia predeterminada desde Git 2.34. Reemplazo a `recursive` con mejor rendimiento y correccion de casos extremos:

```bash
git merge feature                     # Usa ort por defecto
git merge -s ort feature              # Explicito
```

### Estrategia recursive

La estrategia clasica, aun disponible. Maneja merges con ancestros comunes multiples (criss-cross merges):

```bash
git merge -s recursive feature
```

### Opciones de estrategia (`-X`)

Las opciones de estrategia controlan como se resuelven conflictos automaticamente:

```bash
# Preferir nuestra version en conflictos
git merge -X ours feature
# Si hay conflicto, toma automaticamente la version de la rama actual

# Preferir la version de la rama fusionada
git merge -X theirs feature
# Si hay conflicto, toma automaticamente la version de feature

# Usar ours en archivos completos (ignorar cambios de feature en conflicto)
git merge -X ours archivo.txt feature

# Ignorar cambios de espacios en blanco al resolver
git merge -X ignore-space-change feature
git merge -X ignore-all-space feature

# Intentar renombrar deteccion mejorada
git merge -X rename-threshold=70 feature
```

### `-X ours` vs `-X theirs`

| Opcion | En conflicto, conserva | Caso de uso |
|---|---|---|
| `-X ours` | Version de la rama actual (HEAD) | "Se que mi version es la correcta" |
| `-X theirs` | Version de la rama fusionada | "La feature completa reemplaza esto" |

> **Precaucion:** `-X theirs` NO es lo mismo que `-s ours` (estrategia ours, que descarta la otra rama completamente). `-X` son opciones de estrategia; `-s` cambia la estrategia completa.

### Estrategia ours (diferente a `-X ours`)

```bash
git merge -s ours feature-para-descartar
# Crea un merge commit pero IGNORA todos los cambios de la rama fusionada.
# La rama queda "fusionada" en el historial pero sus cambios no se aplican.
# Util para marcar una rama como integrada sin aplicar su contenido.
```

### Comparativa de estrategias

| Estrategia | Cuando usarla |
|---|---|
| `ort` (default) | Caso general: rapida, correcta |
| `recursive` | Compatibilidad con versiones antiguas |
| `ours` | Marcar rama como fusionada sin aplicar sus cambios |
| `subtree` | Merge de proyectos con distinta jerarquia de directorios |

---

## 3.13 Flujo de Trabajo con Ramas

### Flujo basico: Feature Branch Workflow

El flujo mas comun y punto de partida para equipos:

```
main (produccion, estable)
  │
  ├── feature/login    (aislado, desarrollo)
  ├── feature/pagos    (aislado, desarrollo)
  ├── hotfix/urgente   (correccion directa en produccion)
  └── ...
```

```bash
# 1. Actualizar main
git switch main
git pull origin main

# 2. Crear rama de feature
git switch -c feature/nueva-funcionalidad

# 3. Trabajar: modificar, commit, repetir
echo "funcionalidad" > feature.js
git add feature.js
git commit -m "Agregar base de la funcionalidad"

echo "mejora" >> feature.js
git add feature.js
git commit -m "Refinar implementacion"

# 4. Actualizar con cambios de main (si otros avanzaron)
git switch main
git pull origin main
git switch feature/nueva-funcionalidad
git merge main                 # o git rebase main
# Resolver conflictos si los hay

# 5. Push y crear Pull Request (GitHub/GitLab) o merge local
git push -u origin feature/nueva-funcionalidad
# ... crear PR en la plataforma, revision de codigo ...

# 6. Merge a main (despues de aprobacion)
git switch main
git merge feature/nueva-funcionalidad
git push origin main

# 7. Limpiar
git branch -d feature/nueva-funcionalidad
git push origin --delete feature/nueva-funcionalidad
```

### Ejemplo de ciclo completo

```bash
# Escenario: agregar una pagina de "Acerca de" al sitio web

# Inicio
git switch main
git pull origin main
git switch -c feature/pagina-acerca

# Desarrollo: 3 commits atomicos
echo "<h1>Acerca de</h1>" > acerca.html
git add acerca.html
git commit -m "Agregar estructura base de pagina Acerca de"

echo "<p>Nuestra historia...</p>" >> acerca.html
git add acerca.html
git commit -m "Agregar contenido de la pagina Acerca de"

# Agregar estilos
echo ".about { font-family: sans-serif; }" >> style.css
git add style.css
git commit -m "Agregar estilos para pagina Acerca de"

# Verificar estado antes del merge
git log --oneline main..feature/pagina-acerca

# Sincronizar con main por si hubo cambios
git switch main
git pull origin main
git switch feature/pagina-acerca
git merge main          # Sin conflictos, bien

# Merge final
git switch main
git merge feature/pagina-acerca
git push origin main

# Limpiar
git branch -d feature/pagina-acerca
git push origin --delete feature/pagina-acerca
```

### Arbol resultante

```
* 8f3a2c1 (HEAD -> main, origin/main) Merge branch 'feature/pagina-acerca'
|\
| * b2c3d4e Agregar estilos para pagina Acerca de
| * f1e2d3c Agregar contenido de la pagina Acerca de
| * a1b2c3d Agregar estructura base de pagina Acerca de
|/
* e5f6g7h Commit anterior en main
```

> **Principio fundamental:** La rama `main` (o `master`) debe estar siempre en un estado desplegable. Nunca hagas commits directamente en main; siempre usa ramas de feature o hotfix.

---

## Resumen del Capitulo

- Una rama es un **puntero movil** a un commit, almacenado como un archivo de 41 bytes en `.git/refs/heads/`.
- **HEAD** apunta a la rama actual. En modo detached HEAD se apunta directamente a un commit.
- `git branch` crea, lista (`-a`, `-r`, `-v`, `-vv`), elimina (`-d`, `-D`) y renombra (`-m`) ramas.
- `git switch` (moderno) y `git checkout` (clasico) cambian de rama. `switch -c` crea y cambia.
- `git merge` integra ramas: fast-forward (lineal), three-way (merge commit). `--no-ff` fuerza merge commit.
- Los conflictos se resuelven editando los marcadores `<<<<<<<`, `=======`, `>>>>>>>`, luego `git add` y `git commit`.
- `git branch --merged` y `--no-merged` permiten identificar ramas integradas y pendientes. Referencias como `HEAD^`, `HEAD~3`, `@{u}` ayudan a navegar.
- Convenciones de nombres (`feature/`, `bugfix/`, `hotfix/`, `release/`, `chore/`) mantienen el repositorio organizado.
- Las ramas remotas (`origin/main`) se sincronizan con `git fetch`, `git pull` y `git push`. `-u` establece seguimiento.
- `git bisect` realiza busqueda binaria para encontrar el commit exacto que introdujo una regresion.
- Las **estrategias de merge** (`-X ours`, `-X theirs`, `ort`, `recursive`) permiten control fino sobre la fusion de ramas.
- El flujo basico recomendado: crear rama de feature desde main, desarrollar, actualizar con main, mergear de vuelta y limpiar.

---

## Ejercicios Propuestos

1. **Creacion y navegacion de ramas**: Crea un repositorio con 3 commits en main. Crea 3 ramas (`feature-a`, `feature-b`, `experimento`) desde distintos commits. Lista todas las ramas con `git branch -v` y navega entre ellas con `git switch`. Documenta como cambia HEAD en cada paso.

2. **Simulacion de merge con conflictos**: Crea dos ramas desde main que modifiquen las mismas lineas de un archivo. Intenta fusionarlas. Resuelve el conflicto manualmente usando las tres estrategias: a) Conservando la version de la rama actual, b) Conservando la version de la otra rama, c) Escribiendo una combinacion manual de ambas. Usa `git merge --abort` al menos una vez.

3. **Flujo feature branch completo**: Simula el flujo de trabajo completo: a) Crea main con 2 commits, b) Crea rama `feature/api`, agrega 2 commits, c) Mientras tanto, agrega un commit a main (simulando que otro desarrollador avanzo), d) Mergea main en feature, e) Mergea feature en main, f) Elimina la rama feature. Entrega el `git log --graph --oneline --all` final.

4. **Exploracion de ramas remotas**: Clona un repositorio publico que tenga varias ramas (o usa uno propio en GitHub). Investiga: a) Cuantas ramas remotas existen, b) Que ramas locales tienes, c) Cual es la diferencia entre `git branch -v` y `git branch -vv`, d) Simula `git fetch` y analiza `git status -sb`.

5. **Comparacion merge vs squash vs rebase**: En un repositorio nuevo, crea una rama feature con 3 commits. Realiza 3 experimentos independientes (usa `git reset --hard` para volver al punto de partida): a) Merge con merge commit (`--no-ff`), b) Merge con squash, c) Rebase. Compara el `git log --graph --oneline --all` resultante de cada uno y explica las diferencias en el historial.

---

← [Capítulo anterior](02-fundamentos.md) | [Inicio](README.md) | [Capítulo siguiente →](04-remotos.md)
