# Capítulo 11: El Interior de Git

Comprender cómo funciona Git internamente transforma la manera en que trabajas con él. Dejas de verlo como una caja negra de comandos mágicos y empiezas a entenderlo como un sistema de archivos direccionable por contenido, con objetos, referencias y un grafo dirigido acíclico (DAG). Este capítulo te lleva desde la filosofía de diseño de Git hasta la creación manual de commits usando comandos de plumbing, revelando cada capa del modelo de datos que hace a Git tan poderoso y flexible.

---

## 11.1 Filosofía de Diseño: Git como Content-Addressable Filesystem

Git fue diseñado por Linus Torvalds en 2005 con una filosofía clara: ser un **sistema de archivos direccionable por contenido** sobre el cual se construye un sistema de control de versiones.

```
CAPA DE PORCELANA (Porcelain)
───────────────────────────────
  git add, git commit, git push, git pull, git branch, git merge
  (comandos de alto nivel, amigables para el usuario)

CAPA DE PLOMERÍA (Plumbing)
───────────────────────────────
  git hash-object, git cat-file, git update-index,
  git write-tree, git commit-tree, git update-ref
  (comandos de bajo nivel, manipulación directa de objetos)

CAPA DE ALMACENAMIENTO (Storage)
───────────────────────────────
  objects/ (blobs, trees, commits, tags)
  refs/    (heads, tags, remotes)
  index    (staging area)
```

**Principios fundamentales:**

1. **Todo es un objeto.** Archivos, directorios, commits: todos se almacenan como objetos identificados por su hash SHA-1.

2. **Integridad por hash.** El nombre de cada objeto es el hash de su contenido. Si el contenido cambia, el hash cambia. Esto hace que la corrupción de datos sea detectable instantáneamente.

3. **Snapshots, no diferencias.** Git almacena el contenido completo de cada archivo en cada commit (con compresión delta para eficiencia). A diferencia de SVN o CVS, no almacena parches entre versiones.

4. **Operaciones locales.** Casi todo es local. No necesitas conexión al servidor para ver el historial, hacer commits o crear ramas.

> **Dato histórico:** Git nació en solo 2 semanas durante abril de 2005, después de que la licencia gratuita de BitKeeper fuera revocada para el desarrollo del kernel de Linux. Torvalds diseñó Git basándose en su experiencia con sistemas de archivos y en las lecciones aprendidas de las limitaciones de los VCS existentes.

---

## 11.2 Plumbing vs Porcelain

Git fue construido como un toolkit de herramientas pequeñas que hacen una cosa bien. Los comandos que usas diariamente (`add`, `commit`, `push`) son *porcelain* (porcelana): una capa amigable sobre las herramientas de *plumbing* (plomería).

### Demostración: Crear un Commit con Solo Plumbing

Este ejercicio revela exactamente qué sucede bajo el capó cuando ejecutas `git commit`:

```bash
# Crear un repositorio vacío
mkdir demo-internals && cd demo-internals
git init

# Crear un archivo
echo "Hello, Git Internals!" > hello.txt

# ── PLUMBING: Crear el blob ──
# git hash-object calcula el SHA y almacena el contenido en objects/
BLOB_SHA=$(git hash-object -w hello.txt)
echo "Blob creado: $BLOB_SHA"

# ── PLUMBING: Añadir al index (staging area) ──
git update-index --add --cacheinfo 100644 $BLOB_SHA hello.txt

# ── PLUMBING: Crear el objeto tree ──
TREE_SHA=$(git write-tree)
echo "Tree creado: $TREE_SHA"

# ── PLUMBING: Crear el objeto commit ──
COMMIT_SHA=$(echo "Primer commit con plumbing" | git commit-tree $TREE_SHA)
echo "Commit creado: $COMMIT_SHA"

# ── PLUMBING: Actualizar la referencia HEAD ──
git update-ref refs/heads/main $COMMIT_SHA
git symbolic-ref HEAD refs/heads/main

# ¡El commit existe sin haber usado nunca 'git add' ni 'git commit'!
git log --oneline
```

> **Concepto clave:** `git add` = `git hash-object -w` + `git update-index`. `git commit` = `git write-tree` + `git commit-tree` + `git update-ref`. Los comandos de porcelana son atajos que encadenan múltiples operaciones de plumbing.

---

## 11.3 Anatomía del Directorio .git

Cada repositorio Git contiene un directorio `.git` que es el corazón del sistema. Conocer su estructura es esencial para diagnóstico y operaciones avanzadas.

```
.git/
├── HEAD                  → Puntero simbólico a la rama actual
├── config                → Configuración del repositorio
├── description           → Descripción (usado por GitWeb, cgit)
├── index                 → El staging area (archivo binario)
├── packed-refs           → Referencias empaquetadas (optimización)
│
├── objects/              → Base de datos de objetos
│   ├── info/
│   │   ├── packs/        → Índice de packfiles
│   │   └── alternates    → Referencias a otros object stores
│   ├── pack/             → Packfiles (compresión)
│   │   ├── pack-*.pack   → Datos comprimidos
│   │   └── pack-*.idx    → Índice de búsqueda rápida
│   ├── 00/ 01/ ... ff/  → Objetos sueltos (directorios por prefijo SHA)
│   │   └── *.sha         → Cada objeto es un archivo con su hash como nombre
│
├── refs/                 → Referencias (punteros a commits)
│   ├── heads/            → Ramas locales
│   │   ├── main
│   │   └── feature/xyz
│   ├── tags/             → Tags
│   │   └── v1.0.0
│   └── remotes/          → Ramas remotas
│       └── origin/
│           ├── main
│           └── develop
│
├── hooks/                → Scripts de hooks
│   └── *.sample
│
├── info/                 → Información auxiliar
│   ├── exclude           → Exclusiones locales (como .gitignore pero no versionado)
│   └── refs              → Referencias adicionales
│
├── logs/                 → Historial de referencias (reflog)
│   ├── HEAD              → Movimientos de HEAD
│   └── refs/
│       ├── heads/
│       │   └── main      → Movimientos de la rama main
│       └── remotes/
│           └── origin/
│               └── main  → Movimientos de origin/main
│
├── modules/              → Submódulos (Git 2.12+)
│   └── <nombre>/
│       └── (otro .git completo)
│
└── worktrees/            → Worktrees adicionales (Git 2.5+)
    └── <nombre>/
```

### HEAD: El Puntero Actual

```bash
# Ver el contenido de HEAD
cat .git/HEAD
# ref: refs/heads/main        ← HEAD apunta a refs/heads/main

# Cuando estás en detached HEAD:
# 1a2b3c4d5e6f7...           ← HEAD apunta directamente a un commit

# Resolver HEAD al commit actual
git rev-parse HEAD
```

### Config: Configuración del Repositorio

```bash
cat .git/config
```

```ini
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
    ignorecase = true
    precomposeunicode = true

[remote "origin"]
    url = git@github.com:equipo/mi-proyecto.git
    fetch = +refs/heads/*:refs/remotes/origin/*

[branch "main"]
    remote = origin
    merge = refs/heads/main

[submodule "lib/parser"]
    url = https://github.com/equipo/parser.git
    active = true
```

### Index: El Staging Area

El index (o staging area) es un archivo binario en `.git/index` que almacena la información sobre qué archivos serán incluidos en el próximo commit.

```bash
# Ver el contenido del index (plumbing)
git ls-files --stage

# Salida:
# 100644 ce013625030ba8dba906f756967f9e9ca394464a 0    hello.txt
```

Cada entrada del index contiene: modo (permisos), SHA-1 del blob, número de stage (0=normal, 1=base en conflictos, 2=ours, 3=theirs) y el nombre del archivo.

---

## 11.4 Los Cuatro Tipos de Objetos de Git

Todo en Git se reduce a cuatro tipos de objetos almacenados en `.git/objects/`. Cada uno es inmutable y se identifica por el hash SHA-1 de su contenido.

### 11.4.1 Blob (Binary Large Object)

Un blob almacena el contenido de un archivo. **No contiene el nombre del archivo, permisos, ni metadatos.** Solo datos puros.

```bash
# Crear un blob manualmente
echo "funcion saludar() { return 'hola'; }" | git hash-object -w --stdin

# Ver el contenido de un blob
git cat-file -p <blob-sha>

# Ver el tipo de objeto
git cat-file -t <blob-sha>
# blob

# Ver el tamaño del objeto
git cat-file -s <blob-sha>
# 38 (bytes)
```

### 11.4.2 Tree (Árbol)

Un tree representa un directorio. Contiene una lista de entradas, cada una con permisos, tipo (blob o tree), SHA y nombre.

```
ESTRUCTURA DE UN TREE OBJECT
════════════════════════════════════════════

Tree: "a1b2c3..." (directorio raíz del proyecto)
├── 100644 blob "2f3e4..."    README.md
├── 100755 blob "3a4b5..."    script.sh
├── 040000 tree "4b5c6..."    src/
│   ├── 100644 blob "5c6d7..."    main.c
│   └── 100644 blob "6d7e8..."    utils.c
└── 040000 tree "7e8f9..."    lib/
    └── 100644 blob "8f9a0..."    parser.h
```

```bash
# Ver el contenido de un tree
git cat-file -p <tree-sha>

# Alternativa: git ls-tree
git ls-tree HEAD
git ls-tree -r HEAD        # Recursivo
git ls-tree -l HEAD        # Con tamaño
git ls-tree HEAD -- src/   # Filtrar por ruta
```

**Modos en los trees:**

| Modo | Significado |
|------|-------------|
| `100644` | Archivo normal (no ejecutable) |
| `100755` | Archivo ejecutable |
| `120000` | Symlink |
| `040000` | Subdirectorio (tree) |
| `160000` | Submódulo (referencia a commit en otro repo) |

### 11.4.3 Commit

Un objeto commit representa un snapshot del repositorio. Contiene la referencia al tree raíz, metadatos y referencias a commits padres.

```
ESTRUCTURA DE UN COMMIT OBJECT
════════════════════════════════════════════

Commit: "b2c3d4e5f..."

tree a1b2c3d4e5f6...          ← SHA del tree raíz
parent f5g6h7i8j9k0...        ← SHA del commit padre
author Ana López <ana@e.com> 1716100000 +0200
committer Ana López <ana@e.com> 1716100000 +0200

Primer commit con plumbing    ← Mensaje del commit
```

```bash
# Crear un commit con plumbing
COMMIT_SHA=$(echo "Segundo commit" | git commit-tree $TREE_SHA -p $PARENT_SHA)

# Ver el contenido completo de un commit
git cat-file -p $COMMIT_SHA

# Ver solo el mensaje
git log -1 --format=%B $COMMIT_SHA

# Ver solo el autor
git log -1 --format="%an <%ae>" $COMMIT_SHA

# Ver el tree de un commit
git rev-parse $COMMIT_SHA^{tree}
```

> **Nota:** `author` vs `committer`: El *author* es quien escribió el código. El *committer* es quien aplicó el commit. En operaciones como `cherry-pick` o `rebase`, el committer puede diferir del author.

### 11.4.4 Tag (Anotado)

Explorado en el Capítulo 8. Un objeto tag contiene metadata sobre una referencia, incluyendo el objeto al que apunta, tipo, nombre, tagger y mensaje.

### 11.4.5 El Operador de Desreferencia `^{}`

El sufijo `^{}` (dereference operator) sigue punteros hasta llegar al objeto final. Es esencial para navegar tags anotados y otras referencias indirectas:

```bash
# Sin ^{}: apunta al objeto tag (si es tag anotado)
git rev-parse v1.0.0
# a1b2c3d...  (SHA del objeto tag)

# Con ^{}: sigue el tag hasta el commit real
git rev-parse v1.0.0^{}
# e4f5g6h...  (SHA del commit al que apunta el tag)

# ^{tree}: obtiene el tree de un commit
git rev-parse HEAD^{tree}

# ^{blob}: obtiene el blob de una ruta especifica en HEAD
git rev-parse HEAD:README.md

# ^{commit}: asegura que la referencia es un commit
git rev-parse v1.0.0^{commit}
# Si v1.0.0 es un tag ligero que apunta a un tree, esto fallaria

# En packed-refs, ^{} indica que el tag anotado esta en esa linea:
# ^d4e5f6a78... refs/tags/v1.0.0^{}
```



```bash
# Crear un tag anotado con plumbing
git mktag <<EOF | git update-ref refs/tags/v1.0.0
object $COMMIT_SHA
type commit
tag v1.0.0
tagger Ana López <ana@e.com> $(date +%s) +0200

Release 1.0.0: Primera versión estable
EOF
```

---

## 11.5 Navegar la Base de Datos de Objetos

```bash
# ── Inspeccionar objetos ──

# Listar todos los objetos en el repositorio (plumbing)
git rev-list --objects --all

# Ver tipo y tamaño de un objeto
git cat-file -t <sha>
git cat-file -s <sha>
git cat-file -p <sha>       # Pretty-print (contenido legible)

# ── Recorrer la historia ──

# Obtener el tree de un commit
TREE=$(git rev-parse HEAD^{tree})

# Listar el contenido del tree (directorio raíz)
git ls-tree $TREE

# Para cada blob, ver su contenido
git ls-tree -r HEAD | while read MODE TYPE SHA FILE; do
    echo "--- $FILE ($TYPE) ---"
    if [ "$TYPE" = "blob" ]; then
        git cat-file -p $SHA
    fi
done

# ── Encontrar objetos huérfanos ──
git fsck --lost-found
```

---

## 11.6 Crear una Historia Completa con Plumbing

Este ejercicio crea un historial con múltiples archivos, directorios y commits, todo usando comandos de plumbing. Así funciona realmente Git bajo el capó.

```bash
#!/bin/bash
# Script: create-repo-with-plumbing.sh
# Demuestra cómo crear un repositorio completo usando solo plumbing

set -e

REPO="demo-plumbing"
rm -rf "$REPO" && mkdir "$REPO" && cd "$REPO"
export GIT_DIR=$(pwd)/.git

# Inicializar estructura .git manualmente
mkdir -p .git/objects .git/refs/heads .git/refs/tags
echo "ref: refs/heads/main" > .git/HEAD
git read-tree --empty   # Inicializar index vacío

# ──── Commit 1: Crear README.md y src/main.c ────

# Blob: README.md
BLOB_README=$(echo "# Mi Proyecto
Versión inicial del proyecto." | git hash-object -w --stdin)

# Blob: src/main.c
BLOB_MAIN=$(echo '#include <stdio.h>
int main() {
    printf("Hello, Git!\n");
    return 0;
}' | git hash-object -w --stdin)

# Tree: src/
git update-index --add --cacheinfo 100644 $BLOB_MAIN src/main.c
TREE_SRC=$(git write-tree)
git read-tree --empty  # Limpiar el index

# Tree: raíz (contiene README.md y src/)
git update-index --add --cacheinfo 100644 $BLOB_README README.md
git update-index --add --cacheinfo 040000 $TREE_SRC src
TREE_ROOT1=$(git write-tree)
git read-tree --empty

# Commit 1 (sin padre - commit raíz)
COMMIT1=$(echo "Commit inicial del proyecto" | git commit-tree $TREE_ROOT1)
git update-ref refs/heads/main $COMMIT1
echo "Commit 1: $COMMIT1"

# ──── Commit 2: Añadir Makefile ────

# Blob: Makefile
BLOB_MAKEFILE=$(echo 'CC=gcc
CFLAGS=-Wall -O2
all:
	$(CC) $(CFLAGS) -o app src/main.c
clean:
	rm -f app' | git hash-object -w --stdin)

# Reconstruir el tree raíz con los 3 elementos
git update-index --add --cacheinfo 100644 $BLOB_README README.md
git update-index --add --cacheinfo 100644 $BLOB_MAKEFILE Makefile
git update-index --add --cacheinfo 040000 $TREE_SRC src
TREE_ROOT2=$(git write-tree)
git read-tree --empty

# Commit 2 (hereda de Commit 1)
COMMIT2=$(echo "Añade Makefile para compilación" | git commit-tree $TREE_ROOT2 -p $COMMIT1)
git update-ref refs/heads/main $COMMIT2
echo "Commit 2: $COMMIT2"

# ──── Commit 3: Modificar src/main.c ────

BLOB_MAIN2=$(echo '#include <stdio.h>
#include <stdlib.h>
int main() {
    printf("Hello, Git Plumbing!\n");
    return EXIT_SUCCESS;
}' | git hash-object -w --stdin)

# Reconstruir el tree src/ con el nuevo blob
git update-index --add --cacheinfo 100644 $BLOB_MAIN2 src/main.c
TREE_SRC2=$(git write-tree)
git read-tree --empty

# Reconstruir tree raíz
git update-index --add --cacheinfo 100644 $BLOB_README README.md
git update-index --add --cacheinfo 100644 $BLOB_MAKEFILE Makefile
git update-index --add --cacheinfo 040000 $TREE_SRC2 src
TREE_ROOT3=$(git write-tree)
git read-tree --empty

COMMIT3=$(echo "Mejora main.c con stdlib y código de retorno" | git commit-tree $TREE_ROOT3 -p $COMMIT2)
git update-ref refs/heads/main $COMMIT3
echo "Commit 3: $COMMIT3"

# ──── Crear un tag anotado en el commit 3 ────

TAG_FILE=$(mktemp)
cat > "$TAG_FILE" << EOF
object $COMMIT3
type commit
tag v0.1.0
tagger Demo User <demo@gitbook.local> $(date +%s) +0000

Primera versión compilable del proyecto.
EOF
TAG_SHA=$(git mktag < "$TAG_FILE")
git update-ref refs/tags/v0.1.0 $TAG_SHA
rm "$TAG_FILE"
echo "Tag v0.1.0: $TAG_SHA"

# ──── Mostrar el resultado ────
echo ""
echo "═══════════════════════════════════════════"
echo "Historial creado con plumbing:"
echo "═══════════════════════════════════════════"
git log --oneline --graph --all --decorate
echo ""
echo "Contenido del repositorio:"
git ls-tree -r HEAD
echo ""
echo "Total de objetos en .git/objects:"
find .git/objects -type f | wc -l
echo ""
echo "El repositorio funciona exactamente como uno creado con porcelain."
```

Ejecutando este script se crea un repositorio Git completo **sin usar `git add` ni `git commit` ni una sola vez**.

---

## 11.7 SHA-1: Integridad, Colisiones y Transición

### Cómo se Calcula el SHA-1 de un Objeto

```
FÓRMULA: SHA-1("<tipo> <tamaño_en_bytes>\0<contenido>")

Ejemplo para un blob con contenido "hello\n":
  SHA-1("blob 6\0hello\n")

Ejemplo para un commit:
  SHA-1("commit <size>\0tree <sha>\nparent <sha>\nauthor ...\n\nmessage")
```

```bash
# Calcular SHA-1 manualmente (para verificar la fórmula)
CONTENT="hello, world"

# Método portable (funciona en macOS con shasum y Linux con sha1sum):
SIZE=$(printf '%s' "$CONTENT" | wc -c | tr -d ' ')
printf "blob %d\0%s" "$SIZE" "$CONTENT" | shasum -a 1 | awk '{print $1}'
# En Linux: reemplazar shasum por sha1sum

# Comparar con git hash-object
printf '%s' "$CONTENT" | git hash-object --stdin
# Ambos deben dar el mismo resultado
```

> **Nota:** El ejemplo anterior evita `echo -n` (que no es portable puro entre bash y zsh/macOS) usando `printf` en su lugar. La formula correcta es `"<tipo> <size>\0<contenido>"` donde `\0` es el byte NUL, no la cadena `\0`.

### Integridad en Cadena

La integridad de los datos en Git se basa en el hash: cada objeto se referencia por su hash, y cada commit referencia su tree por hash. Si un solo bit cambia en cualquier objeto, el hash cambia y toda la cadena se rompe.

```
CADENA DE INTEGRIDAD
══════════════════════════════════════════

commit → tree → [blob1, blob2, tree/subdir → blob3]
  │        │          │
  SHA      SHA        SHA
  │        │          │
  Si alguno cambia, todo lo que lo referencia
  cambia también. Git detecta la corrupción
  inmediatamente.
```

### Colisiones y SHAttered (2017)

En 2017, investigadores de Google y CWI demostraron la primera colisión práctica de SHA-1 (ataque SHAttered). Esto **no rompe Git** porque:

1. Git no solo usa SHA-1: también verifica el tamaño y contenido de los objetos.
2. La colisión requiere manipular intencionadamente ambos archivos, no es accidental.
3. Git está migrando a SHA-256 como objetivo a largo plazo.

### Formato de Almacenamiento de Objetos

Cada objeto en `.git/objects/` se almacena con el siguiente formato binario:

```
Formato: zlib(<tipo> <tamaño>\0<contenido>)

Ejemplo blob con "hello, world\n":
  "blob 13\0hello, world\n"  → comprimido con zlib → archivo en .git/objects/

Los archivos se nombran con el SHA-1 del contenido sin comprimir.
El prefijo de 2 caracteres del SHA es el subdirectorio,
los 38 caracteres restantes son el nombre del archivo:

  SHA: ce013625030ba8dba906f756967f9e9ca394464a
  Ruta: .git/objects/ce/013625030ba8dba906f756967f9e9ca394464a
```

```bash
# Ver el formato crudo de un objeto
git cat-file -p <sha>         # contenido legible
git cat-file -t <sha>         # tipo (blob, tree, commit, tag)

# Ver los bytes exactos sin interpretacion (incluye header)
# python3 -c "
# import zlib, sys
# with open('.git/objects/ce/013625...', 'rb') as f:
#     print(repr(zlib.decompress(f.read())))
# "
# Salida: b'blob 13\x00hello, world\n'
```

### Transicion a SHA-256

```bash
# Verificar el algoritmo de hash en uso
git rev-parse --short HEAD            # SHA-1 por defecto

# En versiones experimentales con SHA-256:
# git init --object-format=sha256     # (Git 2.29+, experimental)
```

```bash
# Verificar la integridad del repositorio
git fsck --full

# Verificar conectividad de objetos
git fsck --connectivity-only

# Verificar sin recuperar objetos sueltos
git fsck --no-reflogs
```

---

## 11.8 Packfiles: Compresión y Optimización

Con el tiempo, Git empaqueta objetos sueltos en packfiles para ahorrar espacio y mejorar el rendimiento. Un packfile almacena objetos comprimidos, incluyendo deltas (diferencias) contra otros objetos.

### Ver el Estado de los Objetos

```bash
# Ver cuántos objetos sueltos y cuántos en packfiles
git count-objects -v

# Salida típica:
# count: 24          ← objetos sueltos
# size: 48           ← tamaño en KB de objetos sueltos
# in-pack: 152       ← objetos en packfiles
# packs: 1           ← número de packfiles
# size-pack: 128     ← tamaño en KB de packfiles
# prune-packable: 0  ← objetos que serán eliminados con prune
# garbage: 0         ← objetos no referenciados
```

### Generar Packfiles Manualmente

```bash
# Git ejecuta gc automáticamente cuando es necesario, puedes forzarlo
git gc --aggressive --prune=now

# Ver los packfiles generados
ls -lh .git/objects/pack/

# Verificar integridad del packfile
git verify-pack -v .git/objects/pack/pack-*.pack | head -20
```

### Compresión Delta

Git elige objetos similares como bases para deltas. Por ejemplo, versiones consecutivas de un mismo archivo:

```
Packfile: [Objeto A (completo)] [Objeto B (delta contra A)] [Objeto C (completo)] ...

Ventaja: Los archivos grandes que cambian poco se almacenan
muy eficientemente (solo se guarda la diferencia).

Desventaja: Reconstruir B requiere leer A + aplicar delta.
Por eso git gc almacena los objetos más recientes completos
(para acceso rápido) y versiones antiguas como deltas.
```

---

## 11.9 Commit-Graph: Acelerar Operaciones de Historia

El archivo `.git/objects/info/commit-graph` es un indice binario que acelera operaciones como `git log`, `git merge-base` y `git tag --contains`. Sin el, Git debe recorrer el DAG secuencialmente; con el, las operaciones son O(1) o O(log n).

```bash
# Generar commit-graph
git commit-graph write --reachable

# Ver estadisticas del commit-graph
git commit-graph verify
# Salida tipica:
# commit-graph file is valid
# 152 commits
# 3 merges
# 0 generation numbers

# Escribir con numeros de generacion mejorados
git commit-graph write --reachable --changed-paths

# Verificar si un commit esta en el grafo
git commit-graph verify --shallow
```

Git ejecuta `commit-graph write` automaticamente durante `git gc` y `git fetch` (si `fetch.writeCommitGraph` esta activo).

---

## 11.10 Multi-Pack Index (MIDX)

El archivo `.git/objects/pack/multi-pack-index` (MIDX) permite a Git buscar objetos a traves de multiples packfiles simultaneamente, mejorando el rendimiento en repositorios con muchos packfiles.

```bash
# Generar multi-pack index
git multi-pack-index write

# Verificar integridad
git multi-pack-index verify

# Con bitmap (acelera clones y fetches)
git multi-pack-index write --bitmap

# Estadisticas
ls -lh .git/objects/pack/multi-pack-index
```

El MIDX es especialmente util en repositorios grandes donde `git gc` no ha compactado todos los packfiles en uno solo. `git maintenance` (seccion siguiente) lo gestiona automaticamente.

---

## 11.11 `git maintenance`: Garbage Collection Moderno (Git 2.31+)

`git maintenance` es el reemplazo moderno de `git gc` que divide las tareas de mantenimiento en tareas programables por frecuencia:

```bash
# Registrar el repositorio para mantenimiento automatico
git maintenance start

# Esto configura tareas en cron/launchd/systemd segun la plataforma:
# - Hourly: prefetch remoto, commit-graph incremental
# - Daily: loose object GC, pack-refs
# - Weekly: full repack, MIDX write

# Listar tareas registradas
git maintenance list

# Ejecutar tareas manualmente
git maintenance run                    # todas las tareas necesarias
git maintenance run --task=gc          # solo gc
git maintenance run --task=commit-graph  # solo commit-graph
git maintenance run --task=prefetch    # solo prefetch

# Detener mantenimiento automatico
git maintenance stop

# Configurar por repositorio
git config maintenance.auto true
git config maintenance.strategy incremental  # optimizacion progresiva
```

**Diferencias con `git gc`:**
- `git gc` es una operacion unica e intensiva. `git maintenance` distribuye el trabajo.
- `git maintenance` incluye prefetch automatico (mantiene `origin/*` actualizados).
- El prefetch periodico acelera `git fetch` y `git pull` porque los objetos ya estan locales.

---

## 11.12 Reachability Bitmaps

Los bitmaps de alcanzabilidad (`.bitmap` en packfiles) aceleran clones y fetches al permitir a Git determinar rapidamente que objetos necesita enviar:

```bash
# Generar bitmap para un packfile existente
git repack -A -d --write-bitmap-index

# Verificar que los bitmaps existen
ls .git/objects/pack/*.bitmap

# Verificar estadisticas de bitmaps
git count-objects -v | grep bitmap
```

Sin bitmaps, Git debe recorrer todo el DAG para cada fetch/clone. Con bitmaps, el servidor puede responder en tiempo constante determinando exactamente que objetos son alcanzables desde las referencias del cliente.

---

## 11.13 `git replace`: Reemplazar Objetos sin Reescribir Historia

`git replace` permite crear referencias que sustituyen temporalmente un objeto por otro, util para:
- Corregir un commit antiguo sin `rebase` masivo.
- Dividir un commit grande en partes mas pequenas.
- Experimentar con reescritura de historia sin perder la original.

```bash
# Crear un commit de reemplazo
REPLACEMENT=$(echo "Commit corregido" | git commit-tree <tree> -p <parent>)
git replace <commit-original> $REPLACEMENT

# Ahora git log muestra el commit de reemplazo en lugar del original
git log --oneline

# Ver reemplazos activos
git replace -l

# Ver el objeto real detras de un reemplazo
git --no-replace-objects log --oneline

# Eliminar reemplazo
git replace -d <commit-original>

# Convertir reemplazo en permanente (reescribe la historia)
git filter-branch -- --all  # o git filter-repo
```

Los reemplazos se almacenan en `refs/replace/` y no se comparten con el remoto a menos que los pushes explicitamente.

---

## 11.14 Referencias: El Sistema de Punteros

Las referencias son archivos que apuntan a objetos. Son la razón por la que puedes usar nombres como `main` en lugar de hashes hexadecimales.

### Referencias Directas

```bash
# Una rama es un archivo con un SHA
cat .git/refs/heads/main
# b2c3d4e5f6a7890abcdef1234567890abcdef12

# Crear una rama manualmente
echo "b2c3d4e5f6a7890abcdef1234567890abcdef12" > .git/refs/heads/experimental

# Actualizar una referencia con verificación
git update-ref refs/heads/main <nuevo-sha> <sha-antiguo>

# Si el sha-antiguo no coincide, la actualización falla
# (protección contra race conditions)
```

### Referencias Simbólicas (HEAD)

HEAD no apunta a un commit directamente, sino a otra referencia:

```bash
cat .git/HEAD
# ref: refs/heads/main

# Cambiar HEAD a otra rama
git symbolic-ref HEAD refs/heads/develop

# O directamente (mejor usar el comando):
echo "ref: refs/heads/develop" > .git/HEAD
```

### Packed Refs

Para optimizar repositorios con muchas referencias (cientos o miles de ramas/tags), Git las empaqueta en `.git/packed-refs`:

```bash
cat .git/packed-refs

# Formato:
# b2c3d4e5f... refs/heads/main
# c3d4e5f6a... refs/tags/v1.0.0
# ^d4e5f6a78... refs/tags/v1.0.0^{}
#   (^{} indica que es un tag anotado y apunta al objeto tag, no al commit)
```

---

## 11.15 El Grafo de Commits (DAG)

El historial de Git forma un **Grafo Dirigido Acíclico (DAG)** donde:
- Los nodos son commits
- Las aristas son relaciones parentales
- No hay ciclos (un commit no puede ser su propio ancestro)

```
GRAFO DE COMMITS (DAG)
═══════════════════════════════════════════════════════════

A ← B ← C ← H (main)
     ↑       ↑
     │       └── F ← G (feature/login)
     │
     └── D ← E (release/1.0)

Leyenda:
- A: commit inicial (sin padres)
- B: primer hijo de A
- C: merge commit (tiene dos padres: B y E)
- H: HEAD de main (merge de G y E)

Propiedades fundamentales:
- Cada commit conoce a sus padres (no a sus hijos)
- Las ramas son punteros móviles a commits
- El DAG permite encontrar el merge base
  (mejor ancestro común) eficientemente
```

### Consultar el Grafo

```bash
# Ver padres de un commit
git rev-parse HEAD^@     # Todos los padres (para merges)

# Ver ancestros entre dos commits
git rev-list --ancestry-path A..H

# Encontrar merge base (ancestro común más cercano)
git merge-base feature/login main
git merge-base --all feature/login main

# Verificar si un commit es ancestro de otro
git merge-base --is-ancestor A H && echo "Sí, A es ancestro de H"

# Ver la topología del grafo
git log --oneline --graph --all --decorate -n 30
```

---

## 11.16 git fsck: Verificador de Integridad

`git fsck` (File System Check) verifica la integridad y conectividad de los objetos en la base de datos de Git.

```bash
# Verificación básica
git fsck

# Verificación completa (incluye objetos antiguos)
git fsck --full

# Solo verificar conectividad
git fsck --connectivity-only

# Modo estricto (más detallado)
git fsck --strict

# Buscar objetos no referenciados (huérfanos)
git fsck --unreachable

# Buscar objetos dangling (sin referencias)
git fsck --dangling

# Recuperar objetos dangling
git fsck --lost-found
# Los objetos se guardan en .git/lost-found/
```

### Interpretación de la Salida de git fsck

```bash
# Checking object directories: 100% (256/256), done.

# dangling blob 1a2b3c4d5e...
#   Un blob que no pertenece a ningún commit
#   (posiblemente creado con git add y luego eliminado)

# dangling commit a1b2c3d4...
#   Un commit sin rama
#   (posiblemente después de un reset o rebase abortado)

# missing tree 5e6f7a8b...
#   ¡CORRUPCIÓN! Se referencia un tree que no existe
#   Esto es grave y requiere recuperación desde backup
```

> **Tip:** Si `git fsck` reporta objetos dangling, puedes recuperarlos antes de que `git gc` los elimine. Los blobs dangling suelen ser contenido que se perdió después de un `git reset` o `git add` abortado.

---

## 11.17 git gc: Garbage Collection

`git gc` (Garbage Collection) optimiza y limpia la base de datos de objetos, mejorando el rendimiento y reduciendo el uso de disco.

```bash
# Ejecutar gc básico
git gc

# gc agresivo (más lento, mejor compresión)
git gc --aggressive

# Eliminar objetos no referenciados inmediatamente
git gc --prune=now

# Ver cuándo se ejecutó el último gc
cat .git/gc.log 2>/dev/null
git count-objects -v | grep prune

# Configurar gc automático
git config gc.auto 6700        # Objetos sueltos que disparan gc
git config gc.autopacklimit 50 # Packfiles que disparan gc
git config gc.pruneexpire "2.weeks.ago"
```

### ¿Qué Hace git gc?

1. **Recolecta objetos sueltos** y los empaqueta en packfiles comprimidos
2. **Elimina objetos no referenciados** que hayan expirado (por defecto, >2 semanas)
3. **Compacta packfiles múltiples** en menos archivos para eficiencia
4. **Limpia el reflog** de entradas antiguas (configurable con `gc.reflogExpire`)
5. **Actualiza referencias empaquetadas** en `.git/packed-refs`

---

## 11.18 Reflog: El Registro de Movimientos

El reflog registra cada movimiento de las referencias locales (HEAD, ramas). Es la red de seguridad que te permite recuperar commits "perdidos" después de un reset, rebase o eliminación de rama.

```bash
# Ver el reflog de HEAD
git reflog

# Salida típica:
# 1a2b3c4 HEAD@{0}: commit: Arregla bug en parser
# 5e6f7a8 HEAD@{1}: reset: moving to HEAD~1
# 8b9c0d1 HEAD@{2}: merge feature/nueva-ui: Merge made by...
# 2c3d4e5 HEAD@{3}: checkout: moving from main to feature/nueva-ui

# Ver el reflog de una rama específica
git reflog main

# Recuperar un commit desde el reflog
git checkout HEAD@{3}
git branch recuperado HEAD@{5}

# Ver reflog con fechas legibles
git reflog --date=iso

# Configurar expiración del reflog
git config gc.reflogExpire "90.days.ago"              # Objetos alcanzables
git config gc.reflogExpireUnreachable "30.days.ago"   # Objetos inalcanzables
```

### Anatomía de una Entrada de Reflog

```bash
# El reflog es un archivo de texto plano
cat .git/logs/HEAD

# Formato de cada línea:
# <sha-antiguo> <sha-nuevo> <autor> <timestamp> <tabulador><acción>
# Ejemplo:
# 1a2b3c4d 5e6f7a8e Ana López <ana@e.com> 1716100000 +0200	commit: Arregla bug
```

### Recuperación de Desastres con Reflog

```bash
# Escenario: hiciste git reset --hard y perdiste commits
git reflog

# Encuentra el commit perdido (el que estaba antes del reset)
# 5e6f7a8 HEAD@{1}: commit: Commit que se perdió
# 1a2b3c4 HEAD@{2}: reset: moving to HEAD~1

# Recuperarlo creando una rama
git branch recover-branch HEAD@{1}

# O mover HEAD directamente
git reset --hard HEAD@{1}
```

> **Concepto clave:** El reflog es **estrictamente local**. No se transfiere con push/fetch, no se comparte con el equipo. Solo existe en tu máquina. Por eso, `git reflog` es tu red de seguridad personal, pero no puede salvar a otros de sus propios errores.

---

## 11.19 Partial Clone: Como Funciona a Nivel de Objetos

Cuando clonas con `--filter=blob:none`, Git no descarga los blobs (contenido de archivos) inicialmente. En su lugar:

```
Clon normal:
  Cliente: "Dame todo"
  Servidor: Envia TODOS los objetos (commits, trees, blobs)

Partial clone (--filter=blob:none):
  Cliente: "Dame commits y trees, NO blobs"
  Servidor: Envia solo estructura. Los blobs se descargan bajo demanda.

Cuando accedes a un archivo:
  git checkout main
  → Falta el blob para README.md
  → Git contacta al servidor: "Necesito blob SHA ce013625..."
  → Servidor envia solo ese blob
  → El archivo aparece en tu working tree
```

```bash
# Crear partial clone
git clone --filter=blob:none https://github.com/grande/repo.git

# Verificar que es partial
git config remote.origin.partialclonefilter
# blob:none

# Al hacer checkout, Git descarga blobs bajo demanda
# (sin conexion, los archivos faltantes dan error)

# Fetch tambien es parcial
git fetch --filter=blob:none

# Convertir a clone completo si es necesario
git fetch --unshallow
```

Internamente, Git usa el protocolo `filter` que permite al servidor omitir objetos del dialogo. El cliente registra el filtro en `.git/config` y contacta al servidor via `git fetch` con `--filter` cada vez que necesita un blob faltante. Esto es la base de como funcionan los clones shallow (`--depth`) y sparse checkout a nivel de protocolo.

## 11.20 Git y el Sistema de Archivos: Rendimiento

### Cuándo el Tamaño del Repositorio se Vuelve un Problema

| Síntoma | Causa Probable | Solución |
|---------|---------------|----------|
| `git status` lento | Index grande o muchos archivos | `git gc`, `git sparse-checkout` |
| `git clone` lento | Packfile grande | `git clone --depth 1`, Git LFS |
| `.git/` ocupa mucho | Archivos binarios en historia | `git gc --aggressive`, BFG Repo-Cleaner |
| `git log` lento | Historia muy larga | `--since`, `-n`, shallow clone |

### Git LFS (Large File Storage)

Para archivos binarios grandes (imágenes, videos, modelos ML, datasets):

```bash
# Instalar Git LFS
git lfs install

# Rastrear archivos grandes por extensión
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "datasets/*.csv"

# El archivo .gitattributes se versiona
cat .gitattributes
# *.psd filter=lfs diff=lfs merge=lfs -text

# Git LFS almacena punteros en el repo y los archivos reales en un servidor LFS
```

### Sparse Checkout

Para monorepos muy grandes donde solo necesitas una parte:

```bash
# Clonar sin checkout
git clone --no-checkout --filter=blob:none <url>
cd repo
git sparse-checkout init --cone

# Especificar qué directorios necesitas
git sparse-checkout set src/app docs
git checkout main

# Solo src/app y docs/ estarán en tu working tree
```

---

## 11.21 Buenas Prácticas sobre los Internals

1. **No modifiques `.git/` manualmente** a menos que sepas exactamente qué haces. Usa los comandos de plumbing para operaciones de bajo nivel.

2. **Conoce el reflog.** Es tu herramienta de recuperación más poderosa. Siempre verifica `git reflog` antes de declarar trabajo como perdido.

3. **Ejecuta `git gc` periódicamente** en repositorios grandes o con mucha historia, pero no abuses de `--aggressive` (es muy lento y el beneficio marginal es pequeño).

4. **Verifica la integridad con `git fsck`** después de operaciones de riesgo (clonaciones parciales, recuperación de backups, manipulación de objetos).

5. **Entiende el modelo de objetos.** Saber qué es un blob, tree, commit y tag te permite diagnosticar problemas, escribir herramientas y entender por qué Git se comporta como lo hace.

6. **No temas al plumbing.** Los comandos de bajo nivel (`cat-file`, `hash-object`, `ls-tree`) son excelentes para scripting, automatización y aprendizaje.

7. **Usa Git LFS para binarios** desde el principio. Intentar limpiar binarios grandes de la historia después es mucho más doloroso que configurar LFS al inicio.

---

## Resumen del Capítulo 11

- Git es un **filesystem direccionable por contenido** sobre el que se construye un VCS. Los comandos de **porcelain** (alto nivel) son interfaces amigables sobre los de **plumbing** (bajo nivel).
- El directorio `.git/` contiene toda la base de datos: `objects/` (blobs, trees, commits, tags), `refs/` (ramas, tags, remotos), `HEAD` (puntero actual) y el `index` (staging area).
- Los **cuatro tipos de objetos** son: **blob** (contenido de archivo, sin metadatos), **tree** (directorio con permisos y referencias), **commit** (snapshot con tree, padres, autor y mensaje) y **tag** (referencia anotada con metadata).
- Los objetos se identifican por su **hash SHA-1**, calculado a partir del tipo, tamaño y contenido del objeto. Esto garantiza la integridad en cadena.
- Los **packfiles** comprimen objetos usando deltas para ahorrar espacio. `git gc` gestiona la compactación y limpieza automática.
- El **reflog** registra cada movimiento de referencias locales, siendo la herramienta principal de recuperación ante resets, rebases o eliminaciones de rama.
- El historial de commits forma un **DAG** (grafo dirigido acíclico) donde cada commit conoce a sus padres y las ramas son punteros a nodos del grafo.
- `git fsck` verifica la integridad del repositorio y puede recuperar objetos huérfanos.

---

## Ejercicios Propuestos

1. **Crear un commit con plumbing puro:** Crea un repositorio vacío. Sin usar `git add` ni `git commit`, crea un archivo, calcula su blob, constrúyelo en la base de datos, actualiza el index, crea el tree y finalmente crea el commit. Verifica con `git log` que el commit es visible.

2. **Explorar el DAG manualmente:** En un repositorio con al menos 3 ramas y un merge, usa únicamente comandos de plumbing (`git rev-parse`, `git cat-file`, `git merge-base`) para: (a) encontrar el merge base entre dos ramas, (b) listar todos los ancestros comunes, (c) verificar que los commits del merge base son ancestros de ambas ramas.

3. **Simular pérdida y recuperación con reflog:** Crea una rama con 3 commits. Haz `git reset --hard HEAD~2` (perdiendo 2 commits). Sin mirar el log normal, usa `git reflog` para identificar y recuperar los commits perdidos creando una rama `recuperados`.

4. **Auditar objetos con git fsck y cat-file:** Ejecuta `git fsck --lost-found` en un repositorio existente. Para cada objeto dangling reportado, identifica su tipo (`git cat-file -t`), muestra su contenido (`git cat-file -p`) si es posible, y determina si es valioso o puede descartarse.

5. **Analizar la compresión de un repositorio:** Ejecuta `git count-objects -v` antes y después de `git gc --aggressive`. Documenta la diferencia en: número de objetos sueltos, número de packfiles, tamaño total del directorio `.git/`. Explica qué optimizaciones realizó `git gc`.

---

← [Capítulo anterior](10-hooks.md) | [Inicio](README.md) | [Capítulo siguiente →](12-conflictos.md)
