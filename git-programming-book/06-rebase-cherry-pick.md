# Capitulo 6: Rebase y Cherry-pick

El rebase y el cherry-pick son dos de las herramientas mas poderosas de Git para manipular el historial de commits. Mientras que `merge` integra ramas preservando la historia tal cual ocurrio, `rebase` reescribe la historia para que parezca que los cambios se hicieron en un orden diferente. `cherry-pick`, por su parte, permite aplicar commits individuales de una rama a otra, como si recogieras cerezas de un arbol. Dominar estas herramientas te permitira mantener un historial limpio, profesional y facil de revisar.

## 6.1 Entendiendo el Rebase

### 6.1.1 Concepto Fundamental

`git rebase` toma una serie de commits y los "reubica" sobre una nueva base. En lugar de crear un commit de merge, reescribe cada commit de la rama como si se hubiera hecho partiendo de la punta de la rama destino.

**Analogia visual:**

```
Antes del rebase (historial divergente):
          D---E  (feature)
         /
    A---B---C  (main)

Despues de git rebase main (historial lineal):
                D'---E'  (feature)
               /
    A---B---C  (main)
```

Cada commit de `feature` (D, E) se reescribe como un commit nuevo (D', E') con hash diferente, partiendo desde C en lugar de B.

### 6.1.2 Rebase Basico

```bash
# Estando en la rama feature:
git checkout feature
git rebase main

# Equivalente en una linea:
git rebase main feature

# Despues del rebase, el push requiere force:
git push --force-with-lease origin feature
```

Internamente, Git:
1. Identifica el ancestro comun entre `feature` y `main`.
2. Guarda los commits de `feature` que no estan en `main` (D y E en el ejemplo).
3. Mueve `feature` a la punta de `main` (fast-forward).
4. Re-aplica cada commit guardado uno por uno sobre la nueva base.

### 6.1.3 Cuando Usar Rebase

Usa rebase cuando quieras:
- **Mantener un historial lineal** sin commits de merge innecesarios.
- **Actualizar tu rama** con los cambios de `main` antes de abrir un Pull Request.
- **Limpiar tu historial local** antes de compartirlo (squash, reword, reordenar commits).
- **Evitar el "merge commit loop"**: cuando dos ramas se fusionan mutuamente creando ruido.

> **Regla de oro del rebase:** Solo haz rebase de commits que NO han sido empujados a un repositorio compartido. Si otros desarrolladores basaron su trabajo en tus commits, reescribir esos commits con rebase les causara problemas graves.

### Backends de Rebase: `--apply` vs `--merge`

Git tiene dos backends internos para aplicar commits durante el rebase. Desde Git 2.26+, `--merge` es el predeterminado para rebase interactivo:

| Aspecto | `--apply` (legacy) | `--merge` (moderno, default) |
|---|---|---|
| **Mecanismo** | `git format-patch` + `git am` | `git cherry-pick` |
| **Conflictos** | Puede fallar en conflictos complejos | Maneja renombrados y conflictos mejor |
| **Informacion de conflicto** | Generica | Muestra `REBASE_HEAD` con el commit exacto |
| **Interactivo** | No disponible | Requerido para `-i` |
| **Rendimiento** | Ligeramente mas rapido | Ligeramente mas lento (imperceptible) |

```bash
# Forzar backend apply (no interactivo)
git rebase --apply main

# Backend merge (default para -i)
git rebase --merge -i HEAD~5
```

> **Recomendacion:** No necesitas preocuparte por esto en el dia a dia. Git elige el backend correcto automaticamente. Solo usa `--apply` si encuentras un bug en `--merge` o necesitas el comportamiento legacy para scripts compatibles.

## 6.2 Rebase Interactivo (`-i`)

El rebase interactivo es una de las funcionalidades mas poderosas de Git. Te permite editar, reordenar, combinar y eliminar commits durante el proceso de rebase.

### 6.2.1 Iniciar un Rebase Interactivo

```bash
# Rebase interactivo de los ultimos N commits
git rebase -i HEAD~4

# Rebase interactivo desde un commit especifico
git rebase -i a1b2c3d

# Rebase interactivo sobre una rama
git rebase -i main
```

Git abre tu editor mostrando la lista de commits:

```
pick d4e5f6g Agregar validacion de email
pick c3d4e5f WIP: avance parcial
pick b2c3d4d Fix typo en validacion
pick a1b2c3d Tests unitarios para modulo auth

# Commands:
# p, pick <commit> = usar el commit como esta
# r, reword <commit> = usar el commit pero editar el mensaje
# e, edit <commit> = usar el commit pero detenerse para modificarlo
# s, squash <commit> = combinar con el commit anterior (conserva mensajes)
# f, fixup <commit> = como squash, pero descarta el mensaje
# x, exec <comando> = ejecutar comando (resto de la linea) usando shell
# b, break = detenerse aqui (continuar con git rebase --continue)
# d, drop <commit> = eliminar el commit
```

### 6.2.2 Comandos del Rebase Interactivo

| Comando | Abreviatura | Efecto |
|---------|------------|--------|
| `pick` | `p` | Incluye el commit tal cual |
| `reword` | `r` | Incluye el commit pero permite cambiar el mensaje |
| `edit` | `e` | Incluye el commit pero pausa para modificarlo (amend, nuevos cambios) |
| `squash` | `s` | Combina el commit con el anterior, concatenando sus mensajes |
| `fixup` | `f` | Combina el commit con el anterior, descartando su mensaje |
| `drop` | `d` | Elimina el commit completamente |
| `break` | `b` | Inserta una pausa en ese punto |
| `exec` | `x` | Ejecuta un comando de shell |

### 6.2.3 Ejemplos de Comandos Interactivos

**Reordenar commits:**

```
pick a1b2c3d Tests unitarios para modulo auth
pick d4e5f6g Agregar validacion de email
pick b2c3d4d Fix typo en validacion
pick c3d4e5f WIP: avance parcial
```

Simplemente cambia el orden de las lineas en el editor.

**Juntar commits (squash):**

```
pick a1b2c3d Tests unitarios para modulo auth
pick d4e5f6g Agregar validacion de email
squash b2c3d4d Fix typo en validacion
squash c3d4e5f WIP: avance parcial
```

El resultado sera un solo commit combinando los ultimos tres en uno.

**Reescribir mensaje de commit (reword):**

```
pick a1b2c3d Tests unitarios para modulo auth
reword d4e5f6g Agregar validacion de email
pick b2c3d4d Fix typo en validacion
```

Git pausara en `d4e5f6g` y abrira el editor para que escribas un nuevo mensaje.

**Eliminar un commit (drop):**

```
pick a1b2c3d Tests unitarios para modulo auth
drop d4e5f6g Commit con secreto que nunca debio existir
pick b2c3d4d Fix typo en validacion
```

El commit `d4e5f6g` y sus cambios desapareceran completamente.

### 6.2.4 Squash en Detalle: Combinando Commits

El squash es util para consolidar commits pequenos y atomicos (como "fix typo", "WIP", "correccion menor") en commits coherentes y semanticamente significativos.

```bash
# Escenario: 5 commits que deberian ser 2

# Historial actual:
# a1b2c3d Implementar modulo de autenticacion
# d4e5f6g WIP: token JWT
# c3d4e5f Agregar login form
# b2c3d4d Fix: validacion de email
# e5f6g7h Setup inicial del proyecto

# Iniciar rebase interactivo
git rebase -i HEAD~5

# En el editor:
pick e5f6g7h Setup inicial del proyecto
pick c3d4e5f Agregar login form
squash b2c3d4d Fix: validacion de email
pick a1b2c3d Implementar modulo de autenticacion
squash d4e5f6g WIP: token JWT

# Git abrira el editor para combinar mensajes:
# 1er squash: combina mensajes de "login form" + "validacion"
# 2do squash: combina mensajes de "autenticacion" + "token JWT"
```

**Resultado final:** 2 commits limpios en lugar de 5.

### 6.2.5 Fixup: Squash sin Conservar Mensaje

`fixup` es como `squash` pero descarta el mensaje del commit absorbido. Ideal para commits con mensajes tipo "fix", "WIP", "typo".

```bash
# En el editor del rebase interactivo:
pick a1b2c3d Implementar modulo de autenticacion
fixup d4e5f6g fix: correccion menor
fixup c3d4e5f WIP
```

El commit final tendra solo el mensaje "Implementar modulo de autenticacion".

**Atajo rapido con `--fixup` y `--autosquash`:**

```bash
# Hacer un commit fixup que apunta a un commit especifico
git commit --fixup=a1b2c3d

# Luego, rebase con autosquash para aplicarlo automaticamente
git rebase -i --autosquash HEAD~5
# El commit fixup ya aparece con la accion 'fixup' en la posicion correcta
```

### 6.2.6 Edit: Detenerse para Modificar un Commit

```bash
# En el editor:
pick a1b2c3d Setup inicial del proyecto
edit d4e5f6g Implementar modulo de autenticacion
pick e5f6g7h Agregar documentacion

# Git se detiene en d4e5f6g. Puedes:
git add archivo-nuevo.js
git commit --amend          # Modifica el commit actual
# o incluso:
git commit --amend --no-edit  # Solo agregas archivos, sin cambiar mensaje
git rebase --continue       # Continuar

# Si quieres hacer varios commits intermedios:
git add .
git commit -m "Cambio intermedio"
git add .
git commit -m "Otro cambio"
git rebase --continue       # Los commits intermedios se insertan en la historia
```

### 6.2.7 Exec: Ejecutar Comandos en Cada Commit

`exec` ejecuta un comando de shell en cada paso del rebase interactivo. Si el comando falla (exit != 0), el rebase se detiene para que corrijas:

```bash
# Ejecutar tests en cada commit para verificar que ninguno rompe nada
git rebase -i --exec "make test" main

# O en el editor del rebase interactivo:
pick a1b2c3d Refactor: extraer modulo de usuarios
exec npm test
pick d4e5f6g Agregar endpoint de autenticacion
exec npm test
pick e5f6g7h Documentar API
exec npm test
```

**Caso practico: bisect automatico con rebase --exec**

```bash
# Encontrar que commit rompe los tests en una serie de 20 commits:
git rebase -i --exec "npm test -- --grep='login'" HEAD~20

# Git ejecuta el test en cada commit.
# Si un test falla, el rebase se detiene en ese commit exacto.
# Luego puedes corregirlo (amend) y continuar: git rebase --continue
```

## 6.3 Resolviendo Conflictos Durante el Rebase

Cuando Git no puede aplicar automaticamente un commit durante el rebase, se produce un conflicto. El proceso de resolucion es similar al de merge pero con comandos especificos.

### 6.3.1 Flujo de Resolucion de Conflictos

```bash
# Durante el rebase, Git informa de un conflicto:
# CONFLICT (content): Merge conflict in src/app.js
# error: could not apply a1b2c3d... Implementar login

# 1. Ver el estado
git status
# Muestra archivos en conflicto: "both modified: src/app.js"

# 2. Resolver el conflicto editando los archivos
# Los marcadores de conflicto son iguales que en merge:
# <<<<<<< HEAD
# <<<<<<< a1b2c3d (Implementar login)
# =======
# >>>>>>> a1b2c3d (Implementar login)

# 3. Marcar archivos como resueltos
git add src/app.js

# 4. Continuar el rebase
git rebase --continue
```

### 6.3.2 Opciones Durante Conflictos

```bash
# Ver diff del conflicto
git diff

# Abortar completamente el rebase (volver al estado inicial)
git rebase --abort

# Saltar el commit problematico (sus cambios se pierden)
git rebase --skip

# Ver que commit se esta aplicando actualmente
cat .git/rebase-merge/stopped-sha
# O:
git log --oneline -1 REBASE_HEAD

# Usar una herramienta de merge externa
git mergetool
```

### 6.3.3 Estrategia para Conflictos Multiples

```bash
# Si sabes que habra muchos conflictos, puedes resolver todos de una vez:

# 1. Dejar que el rebase falle en cada conflicto
# 2. En cada pausa, resolver y git add
# 3. git rebase --continue
# 4. Repetir hasta que termine

# Alternativa: usar rerere (reuse recorded resolution)
git config --global rerere.enabled true
# Git recuerda como resolviste conflictos y los re-aplica automaticamente
```

> **Tip:** Si estas haciendo rebase de muchos commits y anticipas conflictos repetitivos, habilita `rerere` (`git config --global rerere.enabled true`). Git recordara como resolviste cada conflicto y los reaplicara automaticamente en commits subsiguientes, ahorrandote resolver lo mismo multiples veces.

### ⚠️ CRITICO: `ours` y `theirs` estan INVERTIDOS durante el Rebase

Esta es una de las trampas mas confusas de Git. **Durante un rebase, los significados de `ours` y `theirs` estan al reves que durante un merge:**

| Operacion | `ours` (nuestro) | `theirs` (de ellos) |
|---|---|---|
| **git merge** | HEAD = la rama actual | La rama que estas fusionando |
| **git rebase** | HEAD = la NUEVA BASE (main) | TU commit que se esta aplicando |

```
Durante git rebase main (estando en feature):

NUEVA BASE (ours)         TU COMMIT (theirs)
     ↓                         ↓
A---B---C (main)        D---E (feature)
       ↓                    ↓
  git rebase aplica D sobre C

En conflicto:
  <<<<<<< HEAD (ours)  = contenido de C (la nueva base, NO tu trabajo)
  =======
  >>>>>>> D (theirs)   = tu commit D (tu trabajo que se esta re-aplicando)
```

**Regla mnemotecnica:**
- **Merge**: `ours` = lo que tienes, `theirs` = lo que traes de afuera.
- **Rebase**: `ours` = el nuevo terreno donde te paras (base), `theirs` = tu trabajo que estas trasplantando.

```bash
# Durante un conflicto en rebase:
git checkout --ours archivo.txt    # Toma la version de la BASE (NO tu trabajo)
git checkout --theirs archivo.txt  # Toma TU commit que estas aplicando (tu trabajo)

# Si quieres conservar tu cambio durante rebase, USA --theirs
git checkout --theirs archivo.txt  # ← Correcto para conservar TU trabajo en rebase
```

## 6.4 `git pull --rebase`

`git pull --rebase` es una combinacion de `fetch` + `rebase` que evita la creacion de commits de merge innecesarios al sincronizar con el remoto.

### 6.4.1 Comparativa: Pull con Merge vs Pull con Rebase

**Pull con merge (comportamiento por defecto):**

```bash
git pull origin main

# Historial resultante:
# *   Merge branch 'main' of github.com:...  (merge commit automatico)
# |\
# | * commit remoto  (que otro hizo)
# * | commit local   (que tu hiciste)
# |/
# * commit anterior
```

**Pull con rebase:**

```bash
git pull --rebase origin main

# Historial resultante (lineal):
# * commit local (reubicado)   (tu commit)
# * commit remoto               (commit del otro)
# * commit anterior
```

### 6.4.2 Configurar Pull con Rebase como Default

```bash
# Global: para todos los repositorios
git config --global pull.rebase true

# Solo para este repositorio
git config pull.rebase true

# Solo para una rama especifica
git config branch.main.rebase true
git config branch.feature-x.rebase true

# Preservar merges locales durante pull --rebase
git config --global pull.rebase merges
```

Con `pull.rebase merges`, Git intentara preservar la estructura de merges locales que hayas hecho, en lugar de linealizarlos completamente.

### 6.4.3 Flujo de Trabajo con `pull --rebase`

```bash
# Inicio del dia: sincronizar main
git checkout main
git pull --rebase origin main

# Crear feature branch desde main actualizado
git checkout -b feature/nueva-function

# Durante el dia: trabajar...
git add .
git commit -m "Avance en funcionalidad"

# Antes de abrir PR: actualizar con main
git fetch origin
git rebase origin/main

# O en un solo paso:
git pull --rebase origin main

# Push (con force porque reescribimos historia)
git push --force-with-lease origin feature/nueva-function
```

## 6.5 Rebase vs Merge: Comparativa Completa

### 6.5.1 Diferencias Fundamentales

```
MERGE:
    A---B---C---M  (main, merge commit une las historias)
         \     /
          D---E  (feature)

REBASE:
    A---B---C---D'---E'  (main si hacemos ff, o feature con D' y E')
```

| Aspecto | Merge | Rebase |
|---------|-------|--------|
| **Historial** | Conserva la historia real (cuando y como ocurrio) | Reescribe historia lineal (como si se hubiera planificado) |
| **Commits de merge** | Genera un commit de merge (o fast-forward) | No genera commits de merge |
| **Hash de commits** | Se preservan los hashes originales | Se crean commits nuevos con nuevos hashes |
| **Conflictos** | Se resuelven una sola vez (en el merge commit) | Se resuelven por cada commit re-aplicado |
| **Git blame** | El blame apunta al autor y commit original | El blame apunta a los commits reescritos (el autor se preserva) |
| **Bisect** | Puede ser confuso en historiales con muchos merges | Mas facil al ser lineal |
| **Trabajo en equipo** | Seguro para ramas compartidas | Solo para ramas privadas/no compartidas |
| **Revert** | Facil: `git revert -m 1 <merge-commit>` | Mas complejo: hay que revertir commits individuales |

### 6.5.2 Pros y Contras de Cada Enfoque

**Ventajas del Merge:**
- Preserva el contexto historico real: que se desarrollo en paralelo y cuando se integro.
- No reescribe commits existentes (seguro para ramas compartidas).
- Los conflictos se resuelven una sola vez.
- Facil de revertir una feature completa (revert del merge commit).

**Desventajas del Merge:**
- El historial puede volverse caotico con muchos merges cruzados.
- Dificulta `git bisect` y la navegacion del historial.
- Los merge commits "vacios" (sin conflictos reales) agregan ruido.

**Ventajas del Rebase:**
- Historial limpio, lineal y facil de leer.
- Ideal para `git bisect` y `git log --oneline`.
- Cada commit se verifica individualmente (si hay conflicto, sabes exactamente cual commit lo causo).
- Sin merge commits automaticos que ensucian el historial.

**Desventajas del Rebase:**
- Reescribe la historia: peligroso en ramas compartidas.
- Los conflictos se resuelven por cada commit, potencialmente mas trabajo.
- Pierdes el contexto de que cosas se desarrollaron en paralelo.
- Requiere `push --force-with-lease`, lo que puede ser riesgoso.

### 6.5.3 Regla de Oro del Rebase

> **"No hagas rebase de commits que existen fuera de tu repositorio local y que otras personas puedan haber basado su trabajo en ellos."**

Si respetas esta regla, el rebase es una herramienta segura y poderosa. La violacion de esta regla causa:
- Duplicacion de commits (los mismos cambios con diferentes hashes).
- Confusion en el equipo al hacer pull (conflictos inesperados).
- Perdida potencial de trabajo si alguien hace force push.

### 6.5.4 Estrategia Recomendada por Equipos

Muchos equipos profesionales adoptan una estrategia hibrida:

- **Ramas de feature (privadas):** Rebase para mantener el historial limpio antes del PR.
- **Integracion a main:** Merge (con `--no-ff` para preservar la traza de la feature).
- **Pull Requests:** Squash and merge o Rebase and merge desde la interfaz web.

```bash
# Flujo recomendado:
# 1. Mantener feature actualizada con rebase
git checkout feature/x
git fetch origin
git rebase origin/main
git push --force-with-lease origin feature/x

# 2. El PR se integra con merge --no-ff (conserva el "feature branch context")
# Esto lo hace GitHub/GitLab en la interfaz:
git checkout main
git merge --no-ff feature/x
git push origin main
```

## 6.6 Cherry-pick: Aplicando Commits Especificos

`git cherry-pick` permite seleccionar commits individuales de una rama y aplicarlos en otra. Es util cuando necesitas mover cambios especificos sin fusionar toda la rama.

### 6.6.1 Cherry-pick Basico

```bash
# Aplicar un commit especifico en la rama actual
git cherry-pick a1b2c3d

# Aplicar multiples commits
git cherry-pick a1b2c3d d4e5f6g e5f6g7h

# Aplicar un rango de commits (A no inclusivo, B inclusivo)
git cherry-pick a1b2c3d..d4e5f6g

# Aplicar rango incluyendo el primer commit
git cherry-pick a1b2c3d^..d4e5f6g
```

Cuando haces cherry-pick, Git:
1. Calcula el diff introducido por el commit original.
2. Aplica ese diff en la rama actual.
3. Crea un nuevo commit con el mismo mensaje y autor, pero diferente hash.

### 6.6.2 Opciones de Cherry-pick

```bash
# Cherry-pick sin crear commit automatico
git cherry-pick --no-commit a1b2c3d
# Los cambios quedan en staging/worktree, puedes modificarlos

# Cherry-pick y editar el mensaje
git cherry-pick --edit a1b2c3d

# Cherry-pick conservando el autor original
git cherry-pick -x a1b2c3d
# Agrega "(cherry picked from commit a1b2c3d)" al mensaje

# Cherry-pick con estrategia de merge especifica
git cherry-pick --strategy=recursive -X theirs a1b2c3d

# Si quieres los cambios pero NO el commit (solo diff)
git diff a1b2c3d^..a1b2c3d | git apply
```

### 6.6.3 Manejo de Conflictos en Cherry-pick

```bash
# Si el cherry-pick encuentra conflicto:
# CONFLICT (content): Merge conflict in src/app.js
# error: could not apply a1b2c3d... Mensaje del commit

# 1. Resolver el conflicto en los archivos
# ... editar archivos ...

# 2. Agregar archivos resueltos
git add src/app.js

# 3. Continuar el cherry-pick
git cherry-pick --continue

# Si quieres abortar
git cherry-pick --abort

# Si quieres saltar este commit (en cherry-pick multiple)
git cherry-pick --skip
```

### 6.6.4 Cherry-pick de Rangos

```bash
# Rango A..B (A exclusivo, B inclusivo)
git cherry-pick a1b2c3d..d4e5f6g

# Rango A^..B (A inclusivo, B inclusivo)
git cherry-pick a1b2c3d^..d4e5f6g

# Rango desde un merge base
git cherry-pick main..feature-x
# Aplica todos los commits que estan en feature-x pero no en main

# Cherry-pick de una lista guardada
git rev-list --reverse feature-x | git cherry-pick --stdin
```

### 6.6.5 Cherry-pick vs Merge vs Rebase

| Situacion | Herramienta |
|-----------|-------------|
| Integrar toda una rama en otra | `git merge` o `git rebase` |
| Mover uno o pocos commits especificos | `git cherry-pick` |
| Reordenar commits de una misma rama | `git rebase -i` |
| Traer un hotfix de main a una release branch | `git cherry-pick` |
| Sincronizar una rama con su base | `git rebase` |

## 6.7 `git rebase --onto`: Moviendo Ramas Complejas

`git rebase --onto` permite mover una rama (o parte de ella) a una nueva base, incluso saltando commits. Es la herramienta mas flexible para reestructurar ramas.

### 6.7.1 Sintaxis y Concepto

```bash
git rebase --onto <nueva-base> <base-antigua> <rama>
```

Donde:
- `<nueva-base>`: El commit donde quieres que empiece la rama.
- `<base-antigua>`: El commit desde donde actualmente empieza la rama.
- `<rama>`: La rama que quieres mover (por defecto, HEAD).

Piensa en `--onto` como: "Toma los commits desde `<base-antigua>` (exclusivo) hasta `<rama>` y pontelos encima de `<nueva-base>`."

### 6.7.2 Ejemplos Practicos

**Ejemplo 1: Mover una subrama a otra base**

```
Situacion inicial:
    A---B---C---D  (main)
         \
          E---F---G  (feature)
               \
                H---I  (subfeature)

Queremos mover subfeature directamente sobre main:
    A---B---C---D  (main)
         \       \
          \       H'---I'  (subfeature)
           \
            E---F---G  (feature)
```

```bash
git rebase --onto main feature subfeature
# Resultado: subfeature contiene H' e I' sobre D (main)
# feature no se modifica
```

**Ejemplo 2: Eliminar commits del medio de una rama**

```
Situacion:
    A---B---C---D---E  (feature)

Queremos eliminar C y D:
    A---B---E'  (feature)
```

```bash
git rebase --onto B D feature
# "Toma los commits desde D (exclusivo) hasta feature y pontelos sobre B"
# Solo E se re-aplica (C y D quedan fuera)
```

**Ejemplo 3: Mover una rama a otra despues de un merge equivocado**

```bash
# Situacion: feature-x se baso en develop, pero debio basarse en main

# Ver el ancestro comun
git merge-base feature-x develop  # digamos a1b2c3d

# Mover feature-x a main
git rebase --onto main a1b2c3d feature-x

# Ahora feature-x parte de main en lugar de develop
```

### 6.7.3 Casos de Uso de `rebase --onto`

| Escenario | Comando |
|-----------|---------|
| Mover rama hija a nueva base | `git rebase --onto main feature subfeature` |
| Eliminar commits iniciales de una rama | `git rebase --onto main <commit-hasta> feature` |
| Extraer parte de una rama a otra base | `git rebase --onto release main feature` |
| Despues de un squash merge, mover trabajo restante | `git rebase --onto main <squash-commit> feature` |

### 6.7.4 Ejemplo Avanzado: Desacoplar una Rama de un Ancestro Compartido

```
Situacion: feature-b se baso en feature-a. Quieres mover feature-b
para que parta directamente de main, sin los commits de feature-a:

Antes:
    A---B---C---D---E  (main)
         \
          F---G---H  (feature-a)
                   \
                    I---J  (feature-b)

git rebase --onto main feature-a feature-b

Despues:
    I'---J'  (feature-b, ahora sobre main)
   /
  A---B---C---D---E  (main)
       \
        F---G---H  (feature-a, sin cambios)
```

Este patron es comun cuando:
- feature-a se estanco en revision de PR y quieres avanzar feature-b.
- feature-a se descarto, pero feature-b contiene trabajo valioso.
- Quieres dividir una PR grande en dos PRs independientes.

## 6.8 Escenarios Practicos

### 6.8.1 Escenario 1: Limpiar Historial Antes de un Pull Request

```bash
# Situacion: Tu rama feature tiene 15 commits desordenados
# Quieres reducirlos a 3 commits logicos antes del PR

# 1. Rebase interactivo de los ultimos 15 commits
git rebase -i HEAD~15

# 2. En el editor, reorganizar y agrupar:
pick a1b2c3d Refactor: extraer modulo de usuarios
squash b2c3d4d Fix: correccion en modulo usuarios
fixup c3d4e5f typo
pick d4e5f6g Implementar endpoint de autenticacion
squash e5f6g7h Agregar tests de autenticacion
fixup f6g7h8i Fix test flaky
pick g7h8i9j Documentar API en README
squash h8i9j0k Agregar ejemplos de uso

# Resultado: 3 commits limpios en lugar de 8

# 3. Push con force (la rama es solo tuya)
git push --force-with-lease origin feature/autenticacion

# 4. Abrir PR con historial limpio y profesional
```

### 6.8.2 Escenario 2: Mover un Hotfix entre Ramas

```bash
# Situacion: Encontraste un bug en produccion y lo arreglaste en main
# Necesitas llevar ese fix a la rama release/2.0 y develop

# 1. El fix ya esta en main como commit f1xbug
git log main --oneline -1
# f1xbug Fix: validacion de null en modulo de pagos

# 2. Cherry-pick a release/2.0
git checkout release/2.0
git cherry-pick f1xbug
# Resolver conflictos si es necesario
git push origin release/2.0

# 3. Cherry-pick a develop
git checkout develop
git cherry-pick f1xbug
git push origin develop

# 4. Documentar que el fix fue cherry-pickeado
# Git agrega automaticamente la referencia si usaste -x:
git cherry-pick -x f1xbug
# Mensaje: "Fix: validacion de null... (cherry picked from commit f1xbug)"
```

### 6.8.3 Escenario 3: Recuperar un Commit Perdido en un Rebase Fallido

```bash
# Situacion: Durante un rebase complicado, abortaste y perdiste commits

# 1. Buscar los commits en el reflog
git reflog
# a1b2c3d HEAD@{0}: rebase (abort): returning to refs/heads/feature
# d4e5f6g HEAD@{1}: commit: Cambio importante que crei perdido
# e5f6g7h HEAD@{2}: rebase: checkout main

# 2. Cherry-pick del commit perdido
git cherry-pick d4e5f6g

# O recuperar la rama completa como estaba antes del rebase
git checkout -b feature-backup HEAD@{0}
```

### 6.8.4 Escenario 4: Squash de una Rama Completa

```bash
# Situacion: Quieres hacer squash de toda una rama en un solo commit
# sin rebase interactivo paso a paso

# Metodo 1: Usando merge --squash
git checkout main
git merge --squash feature/compleja
git commit -m "Feature: descripcion consolidada de todo el trabajo"

# Metodo 2: Usando reset + commit
git checkout feature/compleja
git reset --soft main
git commit -m "Feature: descripcion consolidada"

# Metodo 3: Usando rebase interactivo
git rebase -i main
# Cambiar todos los commits (excepto el primero) a 'squash' o 'fixup'
```

### 6.8.5 Escenario 5: Dividir un Commit en Varios

```bash
# Situacion: Un commit contiene cambios de dos features distintas
# y quieres separarlos en commits independientes

# 1. Iniciar rebase interactivo marcando el commit como 'edit'
git rebase -i HEAD~3
# Cambiar 'pick' por 'edit' en el commit problematico

# 2. Git se detiene en ese commit. Deshacer el commit conservando cambios:
git reset HEAD~1
# Ahora los cambios estan en el working directory

# 3. Hacer commits separados por cada feature:
git add src/feature-a.js
git commit -m "Feature A: implementacion"

git add src/feature-b.js
git commit -m "Feature B: implementacion"

# 4. Continuar el rebase
git rebase --continue
```

---

## 6.9 git range-diff: Comparando Series de Commits

`git range-diff` compara dos series de commits (antes y despues de un rebase, por ejemplo) mostrando cuales commits se mapean a cuales, cuales fueron agregados, cuales eliminados y cuales modificados. Es la herramienta perfecta para verificar que un rebase no rompio nada.

### Uso basico

```bash
# Comparar una rama antes y despues de rebase
git range-diff main..feature  main..feature-rebased

# Usando referencias del reflog (comun despues de rebase)
git range-diff HEAD@{1}..HEAD  feature@{1}..feature

# Notacion simplificada (Git 2.19+)
git range-diff @{1}...@
```

### Interpretacion de la salida

```
1:  a1b2c3d = 1:  x1y2z3w  Agregar validacion de email
2:  d4e5f6g ! 2:  x4y5z6w  Refactor modulo auth
    - Antes: "Refactor modulo auth (WIP)"
    + Ahora: "Refactor: extraer logica de autenticacion"
3:  f6g7h8i - 3:  <none>    Fix typo (se hizo squash)
4:  -        > 4:  x7y8z9w  Agregar tests de integracion (nuevo)
```

| Simbolo | Significado |
|---|---|
| `=` | Commit se mapea directamente (mismo diff, hash diferente) |
| `!` | Commit se mapea pero fue MODIFICADO (cambio mensaje, contenido) |
| `-` | Commit fue ELIMINADO en la nueva version |
| `>` | Commit fue AGREGADO en la nueva version |

### Caso practico: verificar un rebase

```bash
# 1. Antes del rebase, guardar la punta actual
OLD_HEAD=$(git rev-parse HEAD)

# 2. Rebase interactivo
git rebase -i main
# Reordenas, haces squash...

# 3. Verificar que no perdiste cambios
git range-diff $OLD_HEAD..feature main..feature

# 4. Si algo se perdio, puedes recuperarlo del reflog
```

---

## 6.10 git rebase --update-refs: Stacked Branches Automatizados

Introducido en Git 2.38, `--update-refs` mantiene automaticamente actualizadas las ramas que dependen de la rama que estas rebaseando. Es la funcionalidad clave para **stacked branch workflows**.

### El problema que resuelve

```
Situacion tipica de stacked branches:
    main
      └── feature-x (rama base)
            └── feature-y (depende de feature-x)
                  └── feature-z (depende de feature-y)
```

Si haces rebase de `feature-x`, las ramas `feature-y` y `feature-z` quedan apuntando a commits antiguos (los previos al rebase). Sin `--update-refs`, debes actualizar cada rama manualmente con rebase --onto.

### Como funciona

```bash
# Hacer rebase de feature-x y actualizar todas las ramas que dependen de ella
git checkout feature-x
git rebase --update-refs -i main
```

Git automaticamente:
1. Rebasea `feature-x` sobre `main`.
2. Detecta que `feature-y` se basaba en `feature-x` (via reflog o config de branch).
3. Actualiza `feature-y` al nuevo `feature-x` rebaseado.
4. Repite para `feature-z` basada en `feature-y`.

### Salida del comando

```
Successfully rebased and updated refs/heads/feature-x.
Updated the following refs with --update-refs:
        refs/heads/feature-y
        refs/heads/feature-z
```

### Caso practico completo

```bash
# 1. Crear stack de branches
git checkout -b feature/auth main
echo "auth" > auth.js && git add auth.js && git commit -m "Auth module"

git checkout -b feature/auth-oauth feature/auth
echo "oauth" > oauth.js && git add oauth.js && git commit -m "OAuth support"

git checkout -b feature/auth-oauth-google feature/auth-oauth
echo "google" > google.js && git add google.js && git commit -m "Google OAuth"

# 2. Despues de que feature/auth recibe feedback, hacer rebase
git checkout feature/auth
git commit --amend -m "Auth module (improved)"

# 3. Actualizar TODO el stack en un solo comando
git checkout feature/auth-oauth-google
git rebase --update-refs -i main

# Resultado: las 3 ramas apuntan a los nuevos commits rebaseados.
# feature/auth → auth' (nuevo hash)
# feature/auth-oauth → oauth'' (basado en auth')
# feature/auth-oauth-google → google'' (basado en oauth'')
```

> **Dato clave:** `--update-refs` hace que los stacked branch workflows sean practicos en Git nativo, sin necesidad de herramientas como Graphite. Combinalo con `push.autoSetupRemote` y `--force-with-lease` para un flujo completo.

---

## Resumen del Capitulo 6

- **Rebase** reubica commits sobre una nueva base, creando un historial lineal. Es ideal para limpiar trabajo local antes de compartir.
- **Rebase interactivo** (`-i`) permite reordenar, combinar (squash/fixup), reescribir (reword), modificar (edit), y eliminar (drop) commits. Es la herramienta principal para curar el historial.
- **Squash** combina multiples commits en uno, concatenando mensajes. **Fixup** hace lo mismo pero descarta el mensaje.
- **Conflictos en rebase** se resuelven por cada commit re-aplicado. Usa `git add` + `git rebase --continue`, `--skip`, o `--abort`.
- **`git pull --rebase`** evita merge commits automaticos, reubicando commits locales sobre los remotos entrantes.
- **Rebase vs Merge:** El merge preserva la historia real; el rebase la reescribe lineal. Usa rebase en ramas privadas, merge para integrar ramas compartidas. La regla de oro: **nunca hagas rebase de commits publicos**.
- **Cherry-pick** aplica commits individuales de una rama a otra. Util para hotfixes, mover cambios especificos y rescatar commits perdidos.
- **`git rebase --onto`** permite mover una rama completa a una nueva base, incluso saltando commits. Es la herramienta mas flexible para reestructuracion.
- **`git range-diff`** compara series de commits (pre/post rebase) mostrando cuales se mapean, cuales cambiaron y cuales se perdieron.
- **`git rebase --update-refs`** (Git 2.38+) actualiza automaticamente ramas dependientes en stacked branch workflows.
- ⚠️ **CRITICO:** Durante rebase, `ours` = nueva base, `theirs` = tu commit. Es lo OPUESTO a merge. Usar `--ours` en rebase descarta tu propio trabajo.
- **`git rebase --exec`** ejecuta comandos (tests, linters) en cada commit durante un rebase, util para verificar integridad.
- Los escenarios practicos cubren: limpiar historial antes de PR, mover hotfixes entre ramas, recuperar commits perdidos y squash de ramas completas.

## Ejercicios Propuestos

1. **Rebase interactivo completo:** Crea un repositorio con 7 commits que incluyan: commits con mensajes "WIP", "fix typo", "avance", y commits bien nombrados. Usa `git rebase -i HEAD~7` para: (a) reordenar commits, (b) hacer squash de los WIP y fixes en sus commits padres, (c) usar `reword` para mejorar mensajes, (d) usar `drop` para eliminar un commit con codigo de debug. El resultado debe ser 3 commits limpios con mensajes profesionales.

2. **Resolucion de conflictos en rebase:** Crea `main` con un archivo `data.txt`. Crea `feature-a` que modifica las primeras 10 lineas y `feature-b` que modifica las mismas lineas de forma diferente. Haz merge de `feature-a` en `main`. Luego intenta `git rebase main` desde `feature-b`. Resuelve los conflictos manualmente y completa el rebase. Habilita `rerere` y repite el ejercicio para ver como Git reutiliza las resoluciones.

3. **Cherry-pick entre ramas:** Crea tres ramas: `develop`, `release/1.0`, y `hotfix/bug-123`. Simula: (a) un bug fix en `hotfix/bug-123` (2 commits), (b) cherry-pick de ambos commits a `release/1.0` y `develop`, (c) simula un conflicto en uno de los cherry-picks y resuelvelo. Verifica que los commits en las tres ramas tengan diferentes hashes pero los mismos cambios semanticos usando `git diff`.

4. **Rebase --onto complejo:** Crea esta estructura de ramas: `main` → `feature` → `subfeature`. Haz commits en las tres ramas. Luego usa `git rebase --onto main feature subfeature` para mover `subfeature` directamente sobre `main`. Explica que paso con los commits de `feature` que estaban en el medio. Repite el ejercicio creando una situacion donde `--onto` sea la unica solucion (sin cherry-pick ni merge).

5. **Simulacion de flujo de equipo:** Trabaja con dos clones locales del mismo repositorio (simulando dos desarrolladores). Cada uno crea una rama feature con 5 commits. Ambos hacen rebase interactivo para limpiar su historial (dejar 2 commits cada uno). Ambos hacen `git pull --rebase origin main` para actualizarse. Simula un conflicto entre las dos features al integrarlas. Finalmente, haz cherry-pick de un commit especifico de una feature a la otra. Documenta todo el flujo con `git log --oneline --graph --all` en cada paso.
