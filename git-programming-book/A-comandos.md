# Apendice A: Referencia Rapida de Comandos

Este apendice proporciona una tabla de referencia rapida de todos los comandos de Git cubiertos en el libro, organizados por categoria funcional. Cada entrada incluye el comando, una descripcion concisa y el capitulo principal donde se explica.

---

## A.1 Configuracion

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git config --global user.name "Nombre"` | Configurar nombre de usuario | 2 |
| `git config --global user.email "email@ejemplo.com"` | Configurar email del usuario | 2 |
| `git config --global core.editor "code --wait"` | Configurar editor por defecto | 2 |
| `git config --global init.defaultBranch main` | Cambiar nombre de rama por defecto | 2 |
| `git config --global alias.co checkout` | Crear alias para comandos | 2 |
| `git config --global rerere.enabled true` | Activar reuso de resolucion de conflictos | 13 |
| `git config --list` | Listar toda la configuracion | 2 |
| `git config gc.reflogExpire` | Ver/ajustar retencion del reflog | 17 |
| `git config --global commit.gpgsign true` | Firmar commits con GPG automaticamente | 10 |

---

## A.2 Creacion y Clonacion

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git init` | Inicializar nuevo repositorio | 2 |
| `git init --bare` | Inicializar repositorio bare (servidor) | 2 |
| `git clone <url>` | Clonar repositorio remoto | 2 |
| `git clone --recurse-submodules <url>` | Clonar con submodulos | 9 |
| `git clone --branch <rama> --single-branch <url>` | Clonar solo una rama | 4 |
| `git clone --depth 1 <url>` | Clonar solo el ultimo commit (shallow) | 4 |
| `git clone --no-checkout <url>` | Clonar sin checkout inicial | 20 |
| `git svn clone <url> --stdlayout` | Clonar repositorio SVN | 13 |

---

## A.3 Cambios Basicos

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git status` | Ver estado del working directory | 2 |
| `git status -s` | Estado en formato corto | 2 |
| `git add <archivo>` | Agregar archivo al staging area | 2 |
| `git add -p` | Agregar interactivamente por hunks | 18 |
| `git add <dir>/` | Agregar todos los cambios en un directorio | 2 |
| `git commit -m "mensaje"` | Crear commit con mensaje | 2 |
| `git commit -am "mensaje"` | Add + commit de archivos trackeados | 2 |
| `git commit --amend` | Modificar el ultimo commit | 2 |
| `git commit --amend --no-edit` | Agregar cambios al ultimo commit sin cambiar mensaje | 2 |
| `git diff` | Ver cambios no staggeados | 2 |
| `git diff --staged` | Ver cambios staggeados | 2 |
| `git diff <rama1>..<rama2>` | Diferencias entre dos ramas | 4 |
| `git log` | Ver historial de commits | 2 |
| `git log --oneline --graph --all` | Historial visual compacto | 4 |
| `git log -p` | Ver diff de cada commit | 2 |
| `git log --author="Nombre"` | Filtrar por autor | 2 |
| `git log --grep="patron"` | Filtrar por mensaje | 2 |
| `git log --since="2 weeks ago"` | Filtrar por fecha | 2 |
| `git show <commit>` | Ver detalles de un commit | 2 |
| `git rm <archivo>` | Eliminar archivo y staggear eliminacion | 2 |
| `git rm --cached <archivo>` | Dejar de trackear sin eliminar archivo | 2 |
| `git mv <origen> <destino>` | Mover/renombrar archivo | 2 |

---

## A.4 Ramas

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git branch` | Listar ramas locales | 3 |
| `git branch -a` | Listar ramas locales y remotas | 3 |
| `git branch <nombre>` | Crear nueva rama | 3 |
| `git branch -d <rama>` | Eliminar rama (fusionada) | 3 |
| `git branch -D <rama>` | Forzar eliminacion de rama | 3 |
| `git branch -m <nuevo-nombre>` | Renombrar rama actual | 3 |
| `git switch <rama>` | Cambiar de rama | 3 |
| `git switch -c <rama>` | Crear y cambiar a nueva rama | 3 |
| `git checkout <rama>` | Cambiar de rama (legado) | 3 |
| `git checkout -b <rama>` | Crear y cambiar de rama (legado) | 3 |
| `git merge <rama>` | Fusionar rama en la actual | 3 |
| `git merge --no-ff <rama>` | Merge con commit explicito | 18 |
| `git merge --squash <rama>` | Squash merge (sin commit automatico) | 18 |
| `git merge --abort` | Cancelar merge en progreso | 3 |
| `git branch --merged` | Listar ramas fusionadas | 3 |
| `git branch --no-merged` | Listar ramas no fusionadas | 3 |

---

## A.5 Remotos

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git remote -v` | Listar remotos configurados | 4 |
| `git remote add <nombre> <url>` | Agregar nuevo remoto | 4 |
| `git remote remove <nombre>` | Eliminar remoto | 4 |
| `git remote rename <old> <new>` | Renombrar remoto | 4 |
| `git remote prune <remoto>` | Limpiar referencias remotas obsoletas | 18 |
| `git remote show <remoto>` | Ver detalles de un remoto | 4 |
| `git fetch <remoto>` | Descargar objetos y referencias remotas | 4 |
| `git fetch --all` | Fetch de todos los remotos | 4 |
| `git fetch --prune` | Fetch y limpiar ramas remotas eliminadas | 4 |
| `git pull` | Fetch + merge de rama remota | 4 |
| `git pull --rebase` | Fetch + rebase de cambios locales | 4 |
| `git push <remoto> <rama>` | Subir commits a remoto | 4 |
| `git push -u <remoto> <rama>` | Push con upstream tracking | 4 |
| `git push --force-with-lease` | Push force seguro | 6 |
| `git push --force` | Push force (peligroso) | 6 |
| `git push --all <remoto>` | Push de todas las ramas | 4 |
| `git push <remoto> <tag>` | Push de un tag especifico | 4 |
| `git push <remoto> --tags` | Push de todos los tags | 4 |
| `git push <remoto> --delete <rama>` | Eliminar rama remota | 4 |

---

## A.6 Deshacer Cambios

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git restore <archivo>` | Descartar cambios en working directory | 5 |
| `git restore --staged <archivo>` | Quitar archivo del staging area | 5 |
| `git restore --source=<commit> <archivo>` | Restaurar archivo desde commit especifico | 5 |
| `git reset HEAD~1` | Deshacer ultimo commit (cambios en working dir) | 5 |
| `git reset --soft HEAD~1` | Deshacer commit (cambios en staging) | 5 |
| `git reset --hard HEAD~1` | Deshacer commit y descartar cambios | 5 |
| `git reset <archivo>` | Quitar archivo del staging (legado) | 5 |
| `git reset --hard origin/main` | Resetear al estado del remoto | 5 |
| `git revert <commit>` | Crear commit que revierte cambios | 5 |
| `git revert -m 1 <merge-commit>` | Revertir un merge commit | 5 |
| `git clean -n` | Simular limpieza de archivos no trackeados | 5 |
| `git clean -fd` | Eliminar archivos y directorios no trackeados | 5 |
| `git clean -fdx` | Eliminar incluyendo archivos ignorados | 5 |

---

## A.7 Rebase y Cherry-Pick

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git rebase <rama>` | Reaplicar commits sobre otra rama | 6 |
| `git rebase -i HEAD~N` | Rebase interactivo (ultimos N commits) | 6 |
| `git rebase --continue` | Continuar rebase tras resolver conflictos | 6 |
| `git rebase --skip` | Saltar commit problematico | 6 |
| `git rebase --abort` | Cancelar rebase | 6 |
| `git rebase --onto <base> <old> <new>` | Rebase avanzado con cambio de base | 6 |
| `git cherry-pick <commit>` | Aplicar commit especifico en rama actual | 6 |
| `git cherry-pick <A>..<B>` | Cherry-pick de rango de commits | 6 |
| `git cherry-pick --continue` | Continuar tras resolver conflicto | 6 |
| `git cherry-pick --abort` | Cancelar cherry-pick | 6 |

---

## A.8 Stash

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git stash` | Guardar cambios en stash | 7 |
| `git stash push -m "mensaje"` | Guardar con mensaje descriptivo | 7 |
| `git stash list` | Listar entradas del stash | 7 |
| `git stash pop` | Aplicar y eliminar ultimo stash | 7 |
| `git stash apply` | Aplicar sin eliminar | 7 |
| `git stash drop stash@{N}` | Eliminar entrada especifica | 7 |
| `git stash clear` | Eliminar todas las entradas | 7 |
| `git stash branch <rama> stash@{N}` | Crear rama desde stash | 7 |
| `git stash show -p stash@{N}` | Ver diff de stash | 7 |

---

## A.9 Tags

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git tag` | Listar tags | 8 |
| `git tag -a <tag> -m "mensaje"` | Crear tag anotado | 8 |
| `git tag <tag>` | Crear tag ligero | 8 |
| `git tag -d <tag>` | Eliminar tag local | 8 |
| `git tag -l "v1.*"` | Listar tags por patron | 8 |
| `git show <tag>` | Ver detalles del tag | 8 |
| `git push <remoto> <tag>` | Push de un tag | 8 |
| `git push <remoto> --delete <tag>` | Eliminar tag remoto | 8 |
| `git describe --tags` | Nombre descriptivo de commit relativo a tags | 17 |
| `git describe --tags --always --dirty` | Nombre con hash y estado dirty | 17 |

---

## A.10 Submodulos

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git submodule add <url> <path>` | Agregar submodulo | 9 |
| `git submodule init` | Inicializar submodulos | 9 |
| `git submodule update` | Actualizar submodulos | 9 |
| `git submodule update --init --recursive` | Inicializar y actualizar recursivo | 9 |
| `git submodule update --remote` | Actualizar al ultimo commit remoto | 9 |
| `git submodule status` | Ver estado de submodulos | 9 |
| `git submodule foreach <comando>` | Ejecutar comando en cada submodulo | 9 |
| `git submodule deinit <path>` | Desinicializar submodulo | 9 |
| `git clone --recurse-submodules <url>` | Clonar con submodulos | 9 |

---

## A.11 Hooks

| Comando / Archivo | Descripcion | Capitulo |
|-------------------|-------------|----------|
| `.git/hooks/pre-commit` | Ejecutar antes de crear commit | 10 |
| `.git/hooks/commit-msg` | Validar mensaje de commit | 10 |
| `.git/hooks/pre-push` | Ejecutar antes de push | 10 |
| `.git/hooks/post-commit` | Ejecutar despues de commit | 10 |
| `.git/hooks/post-receive` | Ejecutar en servidor tras push | 10 |
| `.git/hooks/pre-rebase` | Ejecutar antes de rebase | 10 |
| `chmod +x .git/hooks/<hook>` | Hacer hook ejecutable | 10 |
| `npx husky add .husky/<hook> "comando"` | Crear hook con Husky | 10 |

---

## A.12 Plumbing / Internals (Porcelana Baja)

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git cat-file -p <sha>` | Mostrar contenido de objeto | 11 |
| `git cat-file -t <sha>` | Mostrar tipo de objeto | 11 |
| `git ls-files --stage` | Listar archivos en el indice | 11 |
| `git hash-object -w <archivo>` | Crear blob y devolver SHA | 11 |
| `git update-index --add <archivo>` | Agregar blob al indice | 11 |
| `git write-tree` | Escribir arbol desde indice actual | 11 |
| `git commit-tree <tree> -p <parent> -m "msg"` | Crear commit manualmente | 11 |
| `git count-objects -vH` | Estadisticas de objetos | 11 |
| `git verify-pack -v <pack>` | Inspeccionar contenido de packfile | 11 |
| `git ls-tree <tree-ish>` | Listar contenido de arbol | 11 |
| `git rev-parse <ref>` | Convertir referencia a SHA | 17 |
| `git rev-list <rango>` | Listar SHAs de commits | 17 |
| `git merge-base <A> <B>` | Encontrar ancestro comun | 11 |
| `git show-ref` | Listar referencias y sus SHAs | 11 |
| `git for-each-ref` | Iterar sobre referencias con formato | 11 |

---

## A.13 Avanzado

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git bisect start` | Iniciar sesion de bisect | 17 |
| `git bisect bad <commit>` | Marcar commit como malo | 17 |
| `git bisect good <commit>` | Marcar commit como bueno | 17 |
| `git bisect run <script>` | Automatizar bisect con script | 17 |
| `git bisect reset` | Finalizar sesion de bisect | 17 |
| `git blame <archivo>` | Ver autor por linea | 17 |
| `git blame -L <inicio>,<fin>` | Rango de lineas | 17 |
| `git blame -C -C -C` | Detectar codigo movido/copiado | 17 |
| `git blame -w` | Ignorar cambios de whitespace | 17 |
| `git grep <patron>` | Buscar en archivos versionados | 17 |
| `git grep -n <patron>` | Buscar mostrando numero de linea | 17 |
| `git grep -E <regex>` | Buscar con regex extendido | 17 |
| `git reflog` | Ver historial de HEAD | 17 |
| `git reflog show <rama>` | Ver reflog de rama especifica | 17 |
| `git reflog expire --expire=now --all` | Expirar entradas de reflog | 17 |
| `git archive --format=zip --output=<file> HEAD` | Exportar codigo sin .git | 17 |
| `git bundle create <file> --all` | Crear bundle de todo el repo | 17 |
| `git bundle verify <file>` | Verificar bundle | 17 |
| `git bundle unbundle <file>` | Desempaquetar bundle | 17 |
| `git notes add -m "nota" <commit>` | Agregar nota a commit | 17 |
| `git notes show <commit>` | Ver nota de commit | 17 |
| `git notes remove <commit>` | Eliminar nota | 17 |
| `git range-diff <viejo>..<nuevo>` | Comparar series de commits | 17 |
| `git shortlog -sne` | Resumen de contribuciones | 17 |
| `git describe` | Nombre descriptivo de commit | 17 |

---

## A.14 Git LFS (Large File Storage)

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git lfs install` | Inicializar LFS en repositorio | 14 |
| `git lfs track "*.psd"` | Registrar patron para LFS | 14 |
| `git lfs track --lockable "*.xlsx"` | Track con bloqueo | 14 |
| `git lfs ls-files` | Listar archivos en LFS | 14 |
| `git lfs migrate import --include="*.psd" --everything` | Migrar archivos existentes a LFS | 14 |
| `git lfs migrate export` | Migrar de LFS a Git normal | 14 |
| `git lfs prune` | Limpiar cache local de LFS | 14 |
| `git lfs fetch --all` | Descargar todos los objetos LFS | 14 |
| `git lfs checkout` | Descargar objetos LFS al working tree | 14 |
| `git lfs lock <archivo>` | Bloquear archivo (edicion exclusiva) | 14 |
| `git lfs unlock <archivo>` | Desbloquear archivo | 14 |
| `git lfs locks` | Listar archivos bloqueados | 14 |

---

## A.15 Worktree

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git worktree list` | Listar worktrees | 15 |
| `git worktree add <path> <rama>` | Crear nuevo worktree | 15 |
| `git worktree add -b <nueva-rama> <path>` | Crear worktree con nueva rama | 15 |
| `git worktree remove <path>` | Eliminar worktree | 15 |
| `git worktree prune` | Limpiar worktrees huerfanos | 15 |

---

## A.16 Administracion y Mantenimiento

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git gc` | Recolector de basura (compresion) | 18 |
| `git gc --aggressive --prune=now` | GC agresivo con limpieza inmediata | 18 |
| `git fsck` | Verificar integridad de objetos | 18 |
| `git fsck --full` | Verificacion completa (mas lenta) | 18 |
| `git prune` | Eliminar objetos inalcanzables | 18 |
| `git count-objects -vH` | Estadisticas de objetos (legible) | 18 |
| `git rev-list --objects --all | sort -k2 -nr | head` | Listar objetos por tamaño | 18 |
| `git remote prune <remoto>` | Limpiar referencias remotas | 18 |
| `du -sh .git` | Tamaño total del repositorio | 18 |
| `git filter-repo --analyze` | Analizar repositorio | 13 |
| `git filter-repo --path <dir> --path-rename <dir>:<root>` | Filtrar y renombrar paths | 13 |
| `git filter-repo --path-glob '*.iso' --invert-paths` | Eliminar archivos del historial | 13 |

---

## A.17 Workflows y Colaboracion

| Comando / Archivo | Descripcion | Capitulo |
|-------------------|-------------|----------|
| `.github/workflows/ci.yml` | Definir CI con GitHub Actions | 16 |
| `.gitlab-ci.yml` | Definir CI con GitLab | 16 |
| `bitbucket-pipelines.yml` | Definir CI con Bitbucket | 16 |
| `Jenkinsfile` | Definir pipeline de Jenkins | 16 |
| `.gitignore` | Excluir archivos del versionado | 2 |
| `.gitattributes` | Configurar atributos por archivo | 18 |
| `.pre-commit-config.yaml` | Configurar hooks pre-commit | 18 |
| `COMMIT_EDITMSG` | Mensaje de commit en progreso | 10 |
| `CODEOWNERS` | Definir propietarios de codigo | 15 |
| `CHANGELOG.md` | Registro de cambios del proyecto | 18 |
| `CONTRIBUTING.md` | Guia de contribucion | 18 |

---

## A.18 Comandos Raros pero Utiles

| Comando | Descripcion | Capitulo |
|---------|-------------|----------|
| `git diff --word-diff` | Diff a nivel de palabra | 2 |
| `git log -S "funcion"` | Buscar commits que agregaron/quitaron string | 7 |
| `git add -p` | Agregar cambios por hunks interactivo | 18 |
| `git checkout --orphan <rama>` | Crear rama sin historial | 11 |
| `git replace` | Reemplazar referencias a objetos | 11 |
| `git interpret-trailers` | Parsear trailers de mensaje de commit | 11 |
| `git mailinfo` | Extraer parche de email | 11 |
| `git sparse-checkout init --cone` | Checkout parcial de monorepo | 15 |
| `git switch --detach <commit>` | Entrar en modo detached HEAD | 3 |
| `git merge --strategy-option theirs` | Resolver conflictos automaticamente | 3 |

---

## A.19 Indice Alfabetico Rapido

| Letra | Comandos |
|-------|----------|
| A | add, archive, amend |
| B | bisect, blame, branch, bundle |
| C | checkout, cherry-pick, clean, clone, commit, config |
| D | describe, diff |
| F | fetch, format-patch |
| G | gc, grep |
| I | init |
| L | log, lfs (ext) |
| M | merge, mv |
| N | notes |
| P | pull, push |
| R | range-diff, rebase, reflog, remote, reset, restore, revert, rev-list, rev-parse, rm |
| S | shortlog, show, sparse-checkout, stash, status, submodule, switch |
| T | tag |
| W | worktree |

---

---

← [Capítulo anterior](21-migrando-git.md) | [Inicio](README.md) | [Capítulo siguiente →](B-ruta-aprendizaje.md)
