# Capitulo 5: Deshaciendo Cambios

Una de las habilidades mas valiosas en Git es saber deshacer cambios de forma precisa y segura. A diferencia de otros sistemas donde existe un simple boton "deshacer", Git ofrece un arsenal de comandos especializados que permiten rectificar errores a distintos niveles del modelo de objetos: el directorio de trabajo, el area de staging, el historial de commits y mas. Dominar estas herramientas te dara confianza para experimentar, sabiendo que siempre puedes volver atras.

## 5.1 La Filosofia del "Undo" en Git

Git no tiene un comando universal de "deshacer" porque opera en tres areas distintas (working directory, staging area, repositorio) y porque cada tipo de error requiere una estrategia diferente. Lo que deshaces en Git depende de:

- **Donde** esta el cambio: working directory, staging area, o ya commiteado.
- **Si el cambio** ya fue compartido (push) o es solo local.
- **Que quieres conservar** y que quieres descartar definitivamente.
- **Si necesitas** que el deshacer quede registrado en el historial.

La siguiente tabla resume que comando usar segun el escenario:

| Situacion | Comando | Pierde datos? |
|-----------|---------|---------------|
| Descartar cambios sin commit en un archivo | `git restore <file>` | **Si** (working directory) |
| Sacar un archivo del staging | `git restore --staged <file>` | No |
| Deshacer el ultimo commit (conservando cambios) | `git reset --soft HEAD~1` | No |
| Deshacer el ultimo commit (descartando cambios) | `git reset --hard HEAD~1` | **Si** (irrecuperable sin reflog) |
| Crear un commit que revierte otro | `git revert <commit>` | No (agrega commit) |
| Modificar el ultimo commit | `git commit --amend` | El commit original se pierde |
| Eliminar archivos no rastreados | `git clean -f` | **Si** |

> **Regla de oro:** Si ya hiciste push, usa `git revert` en lugar de `git reset` para no reescribir historia compartida. Si el cambio es solo local, `git reset` es mas limpio.

> **Herramienta auxiliar: `git stash`.** Aunque no es estrictamente un comando de "deshacer", `git stash` guarda temporalmente cambios no commiteados y limpia el working directory. Es la red de seguridad antes de operaciones riesgosas como `git reset --hard` o `git rebase`. Su uso se explora en profundidad en el **Capitulo 7: git stash**.
>
> ```bash
> # Antes de cualquier operacion destructiva:
> git stash push -m "Antes de experimentar"
> git reset --hard HEAD~1   # Si algo sale mal, git stash pop recupera los cambios
> ```

## 5.2 Descartando Cambios en el Working Directory

### 5.2.1 `git restore` (Moderno)

Introducido en Git 2.23, `git restore` es el comando recomendado para trabajar con el working directory y staging area.

```bash
# Descartar cambios en un archivo (vuelve al estado del ultimo commit)
git restore archivo.txt

# Descartar cambios en todos los archivos
git restore .

# Descartar cambios en un directorio completo
git restore src/

# Restaurar un archivo a una version especifica
git restore --source=HEAD~2 archivo.txt

# Restaurar desde una rama especifica
git restore --source=main archivo.txt
```

> **Advertencia:** `git restore` descarta cambios **irreversiblemente** (a menos que esten en el reflog o en otro commit). No hay confirmacion. Asegurate de que realmente quieres perder esos cambios.

### 5.2.1a `git restore -p` (Descartar por Hunks)

Igual que `git add -p` permite seleccionar que agregar al staging, `git restore -p` permite seleccionar que cambios **descartar** interactivamente:

```bash
git restore -p archivo.txt

# Git muestra cada hunk y pregunta:
# Discard this hunk from worktree [y,n,q,a,d,e,?]?
#
# Opciones:
# y - descartar este hunk
# n - no descartar este hunk
# q - salir
# a - descartar este y todos los siguientes del archivo
# d - no descartar este ni los siguientes
# e - editar el hunk manualmente
```

**Caso de uso tipico:** Hiciste varios cambios experimentales y solo quieres conservar algunos. En lugar de `git restore .` (perder todo), usas `-p` para descartar quirurgicamente los cambios que no sirven mientras conservas los buenos.

### 5.2.1b `git checkout -p` (Clasico)

El equivalente clasico, con el mismo comportamiento interactivo:

```bash
git checkout -p archivo.txt
# Misma interfaz de hunks que git restore -p
# Discard this hunk from worktree [y,n,q,a,d,e,?]?
```

### 5.2.2 `git checkout` (Clasico)

Antes de `git restore`, `git checkout` hacia doble funcion: cambiar de rama y restaurar archivos. Aunque aun funciona, `git restore` es mas claro semanticamente.

```bash
# Restaurar un archivo (equivalente a git restore archivo.txt)
git checkout -- archivo.txt

# Restaurar desde un commit especifico
git checkout a1b2c3d -- archivo.txt

# Restaurar desde otra rama
git checkout feature-x -- src/app.js
```

La diferencia principal es que `git checkout <commit> -- <file>` deja el archivo en staging (esta listo para commit), mientras que `git restore --source=<commit> <file>` lo deja en el working directory. Para emular el comportamiento de checkout con restore:

```bash
# restore + stage (equivalente a checkout antiguo)
git restore --source=HEAD~2 --staged --worktree archivo.txt
```

| Comando | Archivo queda en | Area de staging |
|---------|-----------------|-----------------|
| `git restore <file>` | Working directory | Sin cambios |
| `git checkout HEAD -- <file>` | Working directory + Staging | El archivo se stagea |
| `git restore --staged <file>` | Working directory (sin cambios) | Se remueve del staging |

## 5.3 Removiendo Archivos del Staging Area

A veces agregas archivos al staging (`git add`) y luego te arrepientes. Quieres conservar los cambios locales pero no incluirlos en el proximo commit.

### 5.3.1 `git restore --staged`

```bash
# Sacar un archivo del staging (conserva cambios locales)
git restore --staged archivo.txt

# Sacar todos los archivos del staging
git restore --staged .

# Ver archivos en staging antes de remover
git status
git diff --staged
```

### 5.3.2 `git reset HEAD` (Clasico)

```bash
# Forma clasica (equivalente a git restore --staged)
git reset HEAD archivo.txt
git reset HEAD .
```

Hoy en dia `git restore --staged` es la forma recomendada. `git reset HEAD` es mas confuso porque `reset` normalmente opera sobre el historial de commits.

> **Tip:** El mismo `git status` te sugiere el comando correcto. Ejecuta `git status` y lee la seccion "Changes to be committed" o "Changes not staged for commit" para ver las sugerencias.

## 5.4 Deshaciendo Commits con `git reset`

`git reset` mueve el puntero HEAD y opcionalmente modifica el staging area y el working directory. Es el comando mas potente (y peligroso) para deshacer commits locales.

### 5.4.1 Los Tres Niveles de `git reset`

```bash
# --soft:  Mueve HEAD, NO toca staging ni working directory
# --mixed: Mueve HEAD, actualiza staging, NO toca working directory (DEFAULT)
# --hard:  Mueve HEAD, actualiza staging Y working directory (PIERDE CAMBIOS)
```

**Demostracion practica de los tres niveles:**

Partimos de este estado:

```
A --- B --- C (HEAD → main)
```

Queremos deshacer el commit C (volver a B):

```bash
# --soft: C desaparece del historial, cambios de C quedan en staging
git reset --soft HEAD~1
# Resultado: archivos modificados estan en "Changes to be committed"
# Util cuando: quieres rehacer el commit con cambios adicionales

# --mixed: C desaparece, cambios de C quedan en working directory (sin staging)
git reset HEAD~1  # --mixed es el default
# Resultado: archivos modificados estan en "Changes not staged for commit"
# Util cuando: quieres volver a planificar que archivos incluir

# --hard: C desaparece, cambios de C se PIERDEN completamente
git reset --hard HEAD~1
# Resultado: working directory limpio, como si C nunca hubiera existido
# Util cuando: el commit C fue un error total y no quieres conservar nada
```

### 5.4.2 Sintaxis de Referencias para `reset`

```bash
# Deshacer el ultimo commit
git reset --soft HEAD~1    # Usar ~1, ~2, ~3...

# Volver a un commit especifico por hash
git reset --soft a1b2c3d

# Volver al estado de una rama remota
git reset --hard origin/main

# Deshacer los ultimos N commits
git reset --soft HEAD~3

# Volver al commit inicial del repositorio
git reset --soft $(git rev-list --max-parents=0 HEAD)
```

### 5.4.3 `git reset` con Archivos Especificos

Cuando se usa con rutas de archivo, `git reset` solo afecta el staging area (comportamiento `--mixed` parcial):

```bash
# Sacar un archivo del staging
git reset archivo.txt

# Sacar un directorio del staging
git reset src/

# Esto NO mueve HEAD, solo actualiza el staging
```

> **Peligro:** `git reset --hard` es el comando mas destructivo de Git. Antes de ejecutarlo, considera: (1) usar `git stash` para guardar cambios, (2) verificar con `git status` que no perderas trabajo valioso, (3) recordar que el `reflog` puede salvarte si te arrepientes.

### 5.4.4 Ejemplo Practico: "Commitee en la rama equivocada"

```bash
# Escenario: Hiciste 3 commits en main pero debian ir en feature-x

# 1. Guardar el estado actual (el hash del commit mas reciente)
git log --oneline -1  # anotar: a1b2c3d

# 2. Crear la rama correcta desde donde estas
git branch feature-x

# 3. Volver main a su estado original
git reset --hard origin/main

# 4. Continuar trabajando en feature-x
git checkout feature-x
```

Si ya habias empujado los commits a `origin/main`... estas en problemas. En ese caso, mejor usar `git revert` o coordinar con el equipo antes de un force push.

## 5.5 Revirtiendo con `git revert`

A diferencia de `git reset`, que borra historia, `git revert` crea un **nuevo commit** que deshace los cambios de un commit anterior. Es la forma segura de deshacer cambios que ya han sido compartidos.

### 5.5.1 Revertir un Commit Simple

```bash
# Revertir el ultimo commit
git revert HEAD

# Revertir un commit especifico
git revert a1b2c3d

# Revertir un rango de commits (cada uno genera su propio revert)
git revert HEAD~3..HEAD

# Revertir sin crear commit automatico
git revert --no-commit a1b2c3d
# Ahora puedes revisar, modificar, y hacer commit manualmente
```

### 5.5.2 Revertir un Merge Commit

Los merge commits tienen dos padres (o mas). Para revertirlos necesitas especificar cual "lado" del merge quieres conservar:

```bash
# Ver los padres del merge commit
git show --summary <merge-commit>
# Merge: a1b2c3d d4e5f6g (padre1 padre2)

# Revertir conservando el primer padre (la rama donde se hizo el merge)
git revert -m 1 <merge-commit>

# Revertir conservando el segundo padre (la rama que fue fusionada)
git revert -m 2 <merge-commit>
```

### Entendiendo `-m 1` con un Diagrama

```
Antes del merge:
    main (padre 1)        feature (padre 2)
    A---B---C              D---E
           \               /
            F---G---H---M  (M es el merge commit: padre1=H, padre2=E)

git revert -m 1 M:
  -m 1 = "conservar el linaje del PRIMER padre (main)"
  Resultado: se deshacen los cambios de feature (E), volviendo al estado de H

git revert -m 2 M:
  -m 2 = "conservar el linaje del SEGUNDO padre (feature)"  
  Resultado: se deshacen los cambios de main desde la divergencia
```

### El problema de "revertir el revert"

Despues de revertir un merge, Git considera que la rama feature ya fue integrada (el revert esta en el historial de main). Si intentas volver a fusionar la misma rama:

```bash
git merge feature
# Already up to date.  ← Git cree que feature ya esta en main
```

Esto ocurre porque los commits D y E son ancestros de M, que aun esta en el historial. Para volver a fusionar despues de un revert, debes **revertir el revert**:

```
main:   A---B---C---H---M---R  (R = revert de M)
                       / 
feature:          D---E

# Para reintroducir los cambios de feature:
git revert -m 1 R   # Revierte el revert, restaurando los cambios de feature
# O alternativamente:
git checkout feature
git rebase main      # Reescribe feature sobre main, nuevos commits con nuevos hashes
git checkout main
git merge feature    # Ahora si funciona
```

> **Importante:** Despues de revertir un merge, no podras volver a fusionar la misma rama directamente (Git cree que ya esta fusionada). Necesitas primero revertir el revert: `git revert <hash-del-revert>`.
>
> **Mejor practica:** Si el merge de feature resulto en bugs, prefiere repararlos en un nuevo commit en main en lugar de revertir el merge completo. El revert de merge es la opcion nuclear cuando la rama entera debe desaparecer.

### 5.5.3 `reset` vs `revert`: Como Elegir

```
¿Los commits ya fueron empujados (push) y alguien mas podria haberlos bajado?
├── SI → Usa git revert (seguro, no reescribe historia)
└── NO → Evalua:
    ├── ¿Quieres que quede registro de que deshiciste algo?
    │   ├── SI → Usa git revert
    │   └── NO → Usa git reset
    └── ¿Quieres conservar los cambios para reusarlos?
        ├── SI → Usa git reset --soft o --mixed
        └── NO → Usa git reset --hard
```

## 5.6 Modificando el Ultimo Commit con `--amend`

`git commit --amend` te permite modificar el commit mas reciente. Es ideal para corregir errores tontos inmediatamente despues de commitear.

### 5.6.1 Usos Comunes de Amend

```bash
# Cambiar el mensaje del ultimo commit
git commit --amend -m "Nuevo mensaje corregido"

# Agregar un archivo que olvidaste incluir
git add archivo-olvidado.txt
git commit --amend --no-edit   # conserva el mensaje original

# Agregar archivo Y cambiar mensaje
git add archivo-olvidado.txt
git commit --amend -m "Mensaje completo con archivo faltante"

# Solo cambiar el mensaje (sin modificar archivos)
git commit --amend
# Se abre el editor para editar el mensaje
```

### 5.6.2 Que Hace Realmente `--amend`

`git commit --amend` **no modifica** el commit original. Crea un commit nuevo con un hash diferente y mueve HEAD al nuevo commit. El commit anterior queda "huerfano" y eventualmente sera recolectado por el garbage collector.

```
Antes:  A --- B --- C (HEAD → main)

Despues de amend:
        A --- B --- C' (HEAD → main)
                 \
                  C (huerfano, sin referencia)
```

### 5.6.3 Precauciones con Amend

> **Regla critica:** NUNCA hagas amend de un commit que ya fue empujado (push) y que otras personas podrian haber descargado. Reescribiras historia compartida y causaras conflictos graves en el equipo.

Para verificar si el ultimo commit ya fue empujado:

```bash
# Verifica el tracking
git status
# "Your branch is ahead of 'origin/main' by 1 commit." → Seguro hacer amend
# "Your branch is up to date with 'origin/main'." → NO hacer amend
```

Si necesitas modificar un commit ya empujado, el flujo seguro es:

```bash
git revert HEAD  # Crear un nuevo commit que deshaga el anterior
# ... o si insistes en amend:
git commit --amend
git push --force-with-lease  # Coordina con tu equipo antes
```

## 5.7 Eliminando Archivos No Rastreados con `git clean`

`git clean` elimina archivos y directorios que no estan bajo control de versiones. Es util para limpiar artefactos de compilacion, dependencias instaladas localmente, o archivos generados.

### 5.7.1 Opciones de `git clean`

```bash
# Ver que archivos se eliminarian (simulacion)
git clean -n
git clean --dry-run

# Eliminar archivos no rastreados (sin directorios)
git clean -f

# Eliminar archivos y directorios no rastreados
git clean -fd

# Eliminar incluyendo archivos en .gitignore
git clean -fx

# Eliminar directorios con archivos ignorados
git clean -fdx

# Modo interactivo
git clean -i
```

### 5.7.2 Combinaciones de Flags

| Flag | Significado |
|------|-------------|
| `-n` | Dry run: muestra que se eliminaria sin hacerlo |
| `-f` | Force: necesario para que git clean funcione (medida de seguridad) |
| `-d` | Directorios: incluye directorios no rastreados |
| `-x` | Ignored: incluye archivos listados en `.gitignore` |
| `-X` | Solo archivos ignorados (los listados en `.gitignore`) |
| `-i` | Interactivo: pregunta antes de eliminar |

### 5.7.3 Ejemplos Practicos

```bash
# Limpiar despues de una compilacion (build artifacts)
git clean -fd

# Resetear completamente al estado del repositorio
git reset --hard HEAD
git clean -fd

# Limpiar node_modules u otras dependencias locales
git clean -fdx
npm install  # reinstalar desde cero

# Ver que pasaria antes de ejecutar
git clean -nfd
```

> **Advertencia:** `git clean -fdx` es extremadamente destructivo. Elimina TODO lo que no esta en el repositorio, incluyendo archivos de configuracion local, `.env`, dependencias instaladas, etc. Usalo con precaucion.

## 5.8 Recuperando Trabajo Perdido con `git reflog`

El **reflog** (reference log) es un diario de todos los movimientos de HEAD y las ramas locales. Cada vez que haces commit, reset, checkout, amend, rebase, etc., se registra una entrada. El reflog es tu red de seguridad: te permite recuperar commits que parecian perdidos.

### 5.8.1 Fundamentos del Reflog

```bash
# Ver el reflog de HEAD
git reflog

# Ver el reflog de una rama especifica
git reflog show main

# Ver con fechas
git reflog --date=iso

# Formato personalizado
git reflog --format="%h %gd %gs %s" --date=relative
```

Entradas tipicas del reflog:

```
a1b2c3d HEAD@{0}: commit: Agregar modulo de usuarios
d4e5f6g HEAD@{1}: commit: Corregir typo en documentacion
b7c8d9e HEAD@{2}: reset: moving to HEAD~1
f0a1b2c HEAD@{3}: commit (amend): Mejorar logging
c3d4e5f HEAD@{4}: commit: Mejorar logging
g6h7i8j HEAD@{5}: checkout: moving from feature-x to main
```

### 5.8.2 Recuperar Commits "Perdidos"

```bash
# Escenario: Hiciste git reset --hard y perdiste commits

# 1. Buscar el commit perdido en el reflog
git reflog
# Encuentra: a1b2c3d HEAD@{3}: commit: Gran feature que borre

# 2. Recuperar el commit (varias opciones):

# Opcion A: Crear una rama apuntando al commit
git branch recuperado a1b2c3d

# Opcion B: Hacer checkout directo (estado detached HEAD)
git checkout a1b2c3d

# Opcion C: Resetear la rama actual al commit
git reset --hard a1b2c3d

# Opcion D: Cherry-pick del commit
git cherry-pick a1b2c3d
```

### 5.8.3 Ejemplo Completo de Recuperacion

```bash
# 1. Escenario desastroso: reset --hard incorrecto
git log --oneline
# a1b2c3d (HEAD -> main) Version 3.0
# d4e5f6g Version 2.0
git reset --hard HEAD~2  # UPS! Queria volver solo 1

# 2. Verificar que perdimos algo
git log --oneline
# g6h7i8j (HEAD -> main) Version 1.0 (perdimos 2.0 y 3.0!)

# 3. El reflog al rescate
git reflog
# g6h7i8j HEAD@{0}: reset: moving to HEAD~2
# a1b2c3d HEAD@{1}: commit: Version 3.0   <-- aqui esta!
# d4e5f6g HEAD@{2}: commit: Version 2.0

# 4. Recuperar
git reset --hard a1b2c3d
# git log --oneline vuelve a mostrar los 3 commits
```

### 5.8.4 Limitaciones del Reflog

- El reflog es **local**. No se comparte con clones ni se empuja al remoto.
- Las entradas expiran (por defecto 90 dias para objetos inalcanzables, nunca para alcanzables).
- `git gc` (garbage collection) puede limpiar entradas expiradas.
- Solo registra movimientos de referencias, no cambios en archivos individuales.

> **Tip:** Puedes cambiar la expiracion del reflog: `git config gc.reflogExpire "180 days"`. Para objetos inalcanzables: `git config gc.reflogExpireUnreachable "30 days"`.

### Recuperacion de Ultimo Recurso: `git fsck --lost-found`

Cuando el reflog ya no tiene la entrada (expiraron los 90 dias o se ejecuto `git gc`), aun puedes recuperar commits huerfanos con `git fsck`:

```bash
# Buscar objetos inalcanzables (dangling)
git fsck --lost-found

# Salida tipica:
# dangling commit a1b2c3d...
# dangling commit d4e5f6g...
# dangling blob    f7e8d9c...
```

Los commits `dangling` son commits que no son alcanzables desde ninguna referencia (rama, tag). Git los preserva un tiempo pero no los muestra en el reflog.

**Procedimiento de recuperacion:**

```bash
# 1. Encontrar commits huerfanos
git fsck --lost-found | grep "dangling commit"

# 2. Para cada hash candidato, inspeccionar
git show <hash> --oneline --no-patch
# "Corregir race condition..." ← reconoces el mensaje

# 3. Ver el historial completo del commit
git log --oneline <hash>

# 4. Recuperarlo
git branch recuperado <hash>
git checkout recuperado
# O cherry-pickear sus cambios a tu rama actual
git cherry-pick <hash>
```

> **Dato:** `git fsck --lost-found` tambien guarda los objetos encontrados en `.git/lost-found/` para inspeccionarlos sin necesidad de recordar hashes.
>
> **Advertencia:** Despues de `git gc --prune=now`, los objetos dangling se eliminan permanentemente. Esto es irreversible. No ejecutes `git gc --prune` agresivo si crees que puedes necesitar recuperar algo.

## 5.9 Casos Practicos de Deshacer Cambios

### 5.9.1 Caso 1: "Commitee en la rama equivocada"

```bash
# Situacion: Estas en main, hiciste 2 commits que debian ir en feature-x

# Solucion A: Si los commits NO han sido empujados
git branch feature-x         # Crear rama en el punto actual
git reset --hard HEAD~2      # Retroceder main 2 commits
git checkout feature-x       # Seguir trabajando en feature-x

# Solucion B: Si los commits YA fueron empujados
git branch feature-x
git push -u origin feature-x  # Empujar la nueva rama
git reset --hard origin/main  # Volver main al estado remoto
# Si main tuvo cambios locales adicionales, usa git revert en su lugar
```

### 5.9.2 Caso 2: "Olvide un archivo en el commit"

```bash
# Situacion: Hiciste commit pero olvidaste incluir config/database.yml

# Solucion: Amend
git add config/database.yml
git commit --amend --no-edit

# Verificar que el commit ahora incluye el archivo
git show --name-only HEAD
```

### 5.9.3 Caso 3: "Cambie algo que no debia en el working directory"

```bash
# Situacion: Modificaste varios archivos experimentando, solo quieres
# conservar los cambios en src/app.js

# Solucion A: Restaurar archivos especificos
git restore src/config.js
git restore test/test_app.js
git restore README.md

# Solucion B: Stash selectivo (guardar solo lo bueno, descartar el resto)
git stash push src/app.js -m "Cambios que si quiero conservar"
git restore .
git stash pop

# Solucion C: Patch inverso
git diff > mis-cambios.patch   # Guardar cambios actuales
git restore .                  # Limpiar todo
# Editar mis-cambios.patch para dejar solo lo deseado
git apply mis-cambios.patch    # Aplicar solo lo bueno
```

### 5.9.4 Caso 4: "El merge salio mal y quiero volver atras"

```bash
# Situacion: Hiciste git merge feature-x y resulto en conflictos o bugs

# Si el merge NO ha sido commiteado (conflictos sin resolver):
git merge --abort

# Si el merge YA fue commiteado (merge commit creado):
git reset --hard HEAD~1         # Si es local
git revert -m 1 HEAD            # Si ya fue empujado

# Si quieres intentar el merge de nuevo desde cero:
git reset --hard HEAD~1
git merge feature-x  # Reintentar
```

### 5.9.5 Caso 5: "Hice git add de archivos que no debia"

```bash
# Situacion: Agregaste archivos al staging que tienen secretos o basura

# Remover archivos especificos del staging
git restore --staged .env
git restore --staged credentials.json

# Remover todo del staging
git restore --staged .

# Si los archivos son delicados y quieres que git los ignore permanentemente:
echo ".env" >> .gitignore
echo "credentials.json" >> .gitignore
git add .gitignore
git commit -m "Agregar archivos sensibles a .gitignore"
```

### 5.9.6 Caso 6: "Necesito recuperar un archivo borrado hace varios commits"

```bash
# 1. Encontrar el commit donde existia el archivo
git log --all --full-history -- "ruta/al/archivo.txt"
# O usar:
git log --diff-filter=D --summary | grep archivo.txt

# 2. Recuperar el archivo del commit donde aun existia
git checkout a1b2c3d -- ruta/al/archivo.txt

# 3. Commitear la recuperacion
git add ruta/al/archivo.txt
git commit -m "Recuperar archivo.txt borrado accidentalmente"
```

---

## Resumen del Capitulo 5

- Git no tiene un "undo" universal; el comando correcto depende de **donde** esta el cambio y **si fue compartido**.
- `git restore` descarta cambios en el working directory; `git restore --staged` saca del staging.
- `git reset --soft` mueve HEAD conservando staging y working; `--mixed` limpia staging; `--hard` pierde todo.
- `git revert` crea un nuevo commit que deshace cambios; es la unica opcion segura para commits ya empujados.
- `git commit --amend` modifica el ultimo commit local; nunca lo uses en commits compartidos.
- `git clean -f` elimina archivos no rastreados; `-fd` incluye directorios; `-fx` incluye archivos ignorados.
- El `git reflog` es la red de seguridad: registra todos los movimientos de HEAD y permite recuperar commits perdidos (hasta ~90 dias).
- Los casos practicos cubren escenarios comunes: commit en rama equivocada, archivo olvidado, merge fallido, add accidental.

## Ejercicios Propuestos

1. **Los tres niveles de reset:** Crea un repositorio de practica con 3 commits en secuencia. Para cada nivel (`--soft`, `--mixed`, `--hard`), haz `git reset HEAD~1` desde un estado limpio (vuelve a crear los commits antes de cada prueba). Documenta el estado del working directory y staging area despues de cada uno usando `git status` y `git diff`.

2. **Revert de merge:** Crea dos ramas desde `main`: `feature-a` y `feature-b`. Haz cambios en ambas y fusiona `feature-a` en `main` (creando un merge commit). Luego aplica `git revert -m 1 HEAD` para revertir el merge. Intenta volver a fusionar `feature-a` y explica por que no funciona. Solucionalo revirtiendo el revert.

3. **Recuperacion con reflog:** Crea 5 commits en secuencia. Ejecuta `git reset --hard HEAD~3` (pierdes los ultimos 3 commits). Usa `git reflog` para encontrarlos y recupera cada uno creando ramas separadas. Luego usa `git cherry-pick` para traerlos de vuelta a `main` en orden inverso.

4. **Limpieza y amend:** Crea un escenario con archivos no rastreados, archivos en staging, y un commit recien hecho. Practica: (a) usar `git clean -nfd` para ver que se eliminaria, (b) ejecutar `git clean -fd` para limpiar, (c) modificar el ultimo commit con `--amend` para agregar un archivo olvidado y cambiar el mensaje. Verifica cada paso con `git log --oneline` y `git show`.

5. **Simulacion de desastre y recuperacion:** Escribe un script bash que: crea un repositorio, hace 5 commits, simula un `reset --hard` accidental a 4 commits atras, ejecuta `git reflog` para encontrar el commit perdido, y lo recupera. Incluye verificaciones con `git log` antes y despues. Este script debe ser reutilizable como herramienta de aprendizaje.
