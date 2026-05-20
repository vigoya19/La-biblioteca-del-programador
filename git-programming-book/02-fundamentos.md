# Capitulo 2: Fundamentos de Git

## 2.1 Inicializar un Repositorio

Existen dos formas principales de obtener un repositorio Git: inicializar uno nuevo o clonar uno existente.

### git init

El comando `git init` crea un nuevo repositorio vacio en el directorio actual:

```bash
mkdir mi-proyecto
cd mi-proyecto
git init
# Initialized empty Git repository in /Users/usuario/mi-proyecto/.git/

ls -la
# total 0
# drwxr-xr-x   3 usuario  staff   96 Apr 19 10:00 .
# drwxr-xr-x  10 usuario  staff  320 Apr 19 09:59 ..
# drwxr-xr-x  10 usuario  staff  320 Apr 19 10:00 .git
```

### La carpeta .git

El directorio `.git` contiene **todo** lo que Git necesita para gestionar el repositorio. Es el corazon de Git:

```
.git/
  HEAD              # Referencia a la rama actual
  config            # Configuracion especifica del repositorio
  description       # Descripcion (usado por GitWeb)
  hooks/            # Scripts de gancho (pre-commit, post-commit, etc.)
  info/             # Patrones de exclusion globales
  objects/          # Base de datos de objetos (commits, trees, blobs)
  refs/             # Punteros a commits (ramas, tags)
    heads/          # Ramas locales
    tags/           # Tags
    remotes/        # Referencias a ramas remotas
  logs/             # Historial de cambios de referencias (reflog)
  index             # Staging area (archivo binario)
```

> **Advertencia:** Nunca borres manualmente el contenido de `.git` a menos que quieras perder todo el historial. Si eliminas `.git`, pierdes el repositorio (los archivos quedan, pero sin versionado).

### Inicializar en un directorio con archivos existentes

```bash
# Proyecto existente que quieres versionar
cd proyecto-existente
git init
git add .                    # Agrega todos los archivos al staging
git commit -m "Commit inicial"
```

### Opciones utiles de git init

```bash
# Especificar nombre de rama inicial
git init --initial-branch=main

# Crear un repositorio "bare" (sin working directory, para servidores)
git init --bare servidor-central.git

# Especificar directorio sin necesidad de cd
git init /ruta/a/nuevo-proyecto

# Usar una plantilla para hooks y configuraciones
git init --template=/ruta/plantilla
```

---

## 2.2 Clonar Repositorios

### git clone

`git clone` copia un repositorio remoto completo a tu maquina local, incluyendo todo el historial y todas las ramas:

```bash
# HTTPS (mas comun, funciona con firewall)
git clone https://github.com/usuario/proyecto.git

# SSH (requiere configurar llave SSH)
git clone git@github.com:usuario/proyecto.git

# Protocolo Git (solo lectura, el mas rapido pero menos comun)
git clone git://github.com/usuario/proyecto.git
```

### Clonar en un directorio especifico

```bash
# Sin especificar: crea un directorio con el nombre del repositorio
git clone https://github.com/usuario/proyecto.git

# Especificando directorio
git clone https://github.com/usuario/proyecto.git mi-nombre-personalizado

# Clonar en el directorio actual (nota el punto al final)
git clone https://github.com/usuario/proyecto.git .
```

### Clone superficial (shallow clone)

Para repositorios muy grandes donde no necesitas todo el historial:

```bash
# Solo el ultimo commit (el mas ligero)
git clone --depth 1 https://github.com/grande/repo.git

# Los ultimos 10 commits
git clone --depth 10 https://github.com/grande/repo.git

# Clone parcial: solo una rama especifica
git clone --depth 1 --branch develop https://github.com/grande/repo.git

# Clone parcial con filtro de blobs (blobless: sin archivos historicos)
git clone --filter=blob:none https://github.com/grande/repo.git
```

| Tipo de clone | Tamano relativo | Util para |
|---|---|---|
| Completo | 100% | Desarrollo activo |
| `--depth 1` | ~5-20% | CI/CD, lectura rapida |
| `--filter=blob:none` | ~30-50% | Repos enormes con historial util |
| `--filter=tree:0` | ~1-5% | Solo referencias, sin contenido |

> **Nota:** Un clone superficial no puede hacer `git push` a menos que lo "profundices" con `git fetch --deepen=N` o `git fetch --unshallow`.

### Clonar una rama especifica

```bash
git clone --branch develop https://github.com/usuario/proyecto.git
# Equivalente a:
git clone https://github.com/usuario/proyecto.git
cd proyecto
git checkout develop
```

---

## 2.3 El Ciclo de Vida de los Archivos

Cada archivo en tu directorio de trabajo puede estar exactamente en uno de estos estados:

```
               ┌─────────┐
               │Untracked│ ◄── Archivo nuevo que Git no conoce
               └────┬─────┘
                    │ git add
                    ▼
┌───────────┐  ┌─────────┐
│ Unmodified │  │ Staged  │ ◄── Listo para commit
└─────┬──────┘  └────┬────┘
      │              │
      │ edit     git commit
      ▼              ▼
┌───────────┐  ┌─────────────┐
│ Modified  │  │  Committed  │ ◄── Almacenado en .git
└─────┬──────┘  └─────────────┘
      │
      │ git add
      ▼
┌─────────┐
│ Staged  │
└─────────┘
```

### Estados detallados

| Estado | Significado | Como llegas ahi | Como sales |
|---|---|---|---|
| **Untracked** | Archivo nuevo, Git no lo rastrea | Crear archivo nuevo | `git add` |
| **Unmodified** | No ha cambiado desde el ultimo commit | `git commit` | Editar archivo |
| **Modified** | Cambiado pero no en staging | Editar archivo rastreado | `git add` |
| **Staged** | Listo para el proximo commit | `git add` | `git commit` o `git restore --staged` |

```bash
# Ejemplo practico del ciclo completo
echo "v1" > archivo.txt        # Untracked
git add archivo.txt             # Staged
git commit -m "primer version"  # Unmodified (committed)

echo "v2" > archivo.txt         # Modified
git add archivo.txt             # Staged
echo "v3" >> archivo.txt        # Modified AND Staged (parcialmente)
# El archivo tiene v2 en staging y v3 en working. 
# Al hacer commit, solo se incluira v2.
```

> **Concepto clave:** El mismo archivo puede estar en staging y modificado simultaneamente. `git add` toma una foto en ese momento; cambios posteriores no se incluyen hasta otro `git add`.

---

## 2.4 Agregar Archivos al Staging

### git add basico

```bash
# Agregar archivos especificos
git add archivo1.txt archivo2.js

# Agregar todo el contenido del directorio actual (recursivo)
git add .

# Agregar todo el proyecto (desde la raiz)
git add -A
git add --all

# Agregar por patron
git add *.js                    # Todos los .js en directorio actual
git add 'src/**/*.test.js'     # Todos los .test.js bajo src/
git add docs/                   # Todo el directorio docs/
```

### Modo interactivo (-i, --interactive)

```bash
git add -i
```

El modo interactivo presenta un menu:

```
*** Commands ***
  1: status      2: update      3: revert      4: add untracked
  5: patch       6: diff        7: quit        8: help
What now>
```

### Modo patch (-p, --patch)

Permite seleccionar interactivamente que partes (hunks) de cada archivo agregar. Ideal para commits atomicos:

```bash
git add -p

# Git muestra cada hunk y pregunta:
# Stage this hunk [y,n,q,a,d,e,?]?

# Opciones:
# y - agregar este hunk
# n - no agregar este hunk
# q - salir (no agregar este ni los siguientes)
# a - agregar este y todos los restantes del archivo
# d - no agregar este ni los restantes del archivo
# s - dividir el hunk en partes mas pequenas
# e - editar el hunk manualmente
```

```bash
# Ejemplo: solo quiero agregar parte de mis cambios
git add -p main.py

# Si tienes cambios de debug y cambios reales mezclados:
# 1. Usa 's' para dividir hunks
# 2. Usa 'y' solo para los cambios que pertenecen al commit
# 3. Los cambios de debug quedan en working para limpiar despues
```

> **Tip profesional:** `git add -p` es una de las herramientas mas poderosas de Git. Te permite hacer commits atomicos y semanticamente significativos incluso cuando hiciste varios cambios no relacionados. Dominala.

### git add -u (update)

Agrega solo archivos ya rastreados que han sido modificados o eliminados. No incluye archivos nuevos:

```bash
git add -u         # Solo modificados y eliminados en el dir actual
git add -u .       # Igual, en todo el arbol
```

### Deshacer git add

```bash
# Quitar un archivo del staging (version moderna, Git >= 2.23)
git restore --staged archivo.txt

# Version clasica
git reset HEAD archivo.txt

# Quitar todos los archivos del staging
git restore --staged .
```

---

## 2.5 Crear Commits

### git commit basico

```bash
# Commit con mensaje en linea (-m)
git commit -m "Agregar modulo de autenticacion"

# Commit abriendo el editor (recomendado para mensajes largos)
git commit

# Commit de todos los archivos modificados (salta el staging)
git commit -a -m "Actualizar dependencias"
# Equivalente a: git add -u && git commit -m "..."
# Difiere de git add -u en sparse checkouts: commit -a respeta
# los patrones de sparse-checkout, add -u no
```

### Mensajes de commit efectivos

Un buen mensaje de commit tiene este formato:

```
Resumen breve (max 50 caracteres, en imperativo)

Cuerpo explicativo detallando el que y el por que del cambio.
El como ya esta en el codigo. Envuelve las lineas a 72
caracteres para legibilidad en herramientas git.

- Usa bullets si es necesario
- Explica el contexto y las motivaciones

Refs: #123, #456
```

```bash
# Ejemplo de buen mensaje de commit
git commit -m "Corregir race condition en el cache de sesiones

La funcion getSession() no estaba sincronizada correctamente
cuando multiples goroutines accedian al mismo mapa de sesiones.
Se agrego un sync.RWMutex para proteger las lecturas y escrituras.

El bug se manifestaba en produccion con ~500 usuarios concurrentes.

Fixes: #452"
```

| Buenas practicas | Malas practicas |
|---|---|
| "Agregar validacion de email en formulario" | "fix" |
| "Corregir overflow en calculo de totales para ordenes > 100 items" | "cambios" |
| "Eliminar dependencia obsoleta de lodash" | "WIP" |
| Imperativo presente ("Agregar", "Corregir") | Pasado ("Agregue", "Corregi") |

> **Regla de oro:** Si aplicas `git log --oneline`, cada mensaje debe decir exactamente que hace el commit. Tu yo del futuro te lo agradecera.

### Modificar el ultimo commit (--amend)

```bash
# Corregir el mensaje del ultimo commit
git commit --amend -m "Nuevo mensaje corregido"

# Agregar archivos olvidados al ultimo commit
git add archivo-olvidado.txt
git commit --amend --no-edit     # Mantiene el mensaje original

# Modificar autor del ultimo commit
git commit --amend --author="Nombre <email@ejemplo.com>"
```

> **Advertencia:** `--amend` reescribe el historial. Solo usalo en commits que NO han sido enviados a un repositorio compartido (no han sido `push`eados).

### Commits atomicos

Un commit atomico es aquel que contiene un unico cambio logico. Si necesitas describir el commit con "y", probablemente deberian ser dos commits:

```bash
# MAL: cambios no relacionados en un solo commit
git add .
git commit -m "Corregir bug de login y actualizar estilos del footer"

# BIEN: commits atomicos
git add src/auth/login.js
git commit -m "Corregir validacion de contraseñas en login"
git add src/styles/footer.css
git commit -m "Actualizar estilos del footer para modo oscuro"
```

---

## 2.6 Verificar el Estado

### git status

Muestra el estado actual del repositorio: que rama estas, que archivos han cambiado, que hay en staging:

```bash
git status
```

Salida tipica:

```
On branch main
Your branch is up to date with 'origin/main'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   src/config.js
        new file:   tests/config.test.js

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   README.md

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        docs/notas.txt
```

### Modo corto (-s, --short)

```bash
git status -s
git status --short

# Salida:
# M  src/config.js      Modificado en staging
# A  tests/config.test.js   Agregado en staging
#  M README.md          Modificado en working (no staged)
# ?? docs/notas.txt     Sin seguimiento
```

Significado de las dos columnas:

```
XY archivo.txt

X = estado en staging
Y = estado en working directory

?? = Untracked
A  = Added (en staging)
 M = Modified (en working)
M  = Modified (en staging)
MM = Modified en staging Y en working (dos versiones distintas)
D  = Deleted (en staging)
 D = Deleted (en working)
R  = Renamed
C  = Copied
```

```bash
# Ver estado en formato corto con rama remota
git status -sb
# ## main...origin/main [ahead 2, behind 1]
# M  src/config.js
# ?? nuevo-archivo.txt
```

---

## 2.7 Historial de Commits

### git log basico

```bash
git log                    # Completo (enter para avanzar, q para salir)
git log --oneline          # Un commit por linea (resumen)
git log -5                 # Ultimos 5 commits
git log -5 --oneline       # Ultimos 5, formato corto
```

### Opciones de formato mas usadas

```bash
# Mostrar cambios (diff) de cada commit
git log -p
git log -p -2             # Cambios de los ultimos 2 commits

# Mostrar estadisticas (archivos cambiados, lineas agregadas/eliminadas)
git log --stat

# Formato personalizado
git log --pretty=format:"%h - %an, %ar : %s"
# abc1234 - Juan Perez, hace 3 dias : Corregir bug de login

git log --pretty=format:"%C(yellow)%h%Creset %C(blue)%ad%Creset | %s %C(green)(%an)%Creset" --date=short
# abc1234 2025-01-15 | Corregir bug de login (Juan Perez)
```

### Placeholders para --pretty=format

| Placeholder | Descripcion | Ejemplo |
|---|---|---|
| `%H` | Hash completo del commit | `abc1234def5678...` |
| `%h` | Hash abreviado | `abc1234` |
| `%an` | Nombre del autor | `Juan Perez` |
| `%ae` | Email del autor | `juan@ejemplo.com` |
| `%ad` | Fecha del autor | `Wed Jan 15 14:30 2025` |
| `%ar` | Fecha relativa | `hace 3 dias` |
| `%s` | Asunto (subject) | `Corregir bug de login` |
| `%b` | Cuerpo (body) | Texto completo del mensaje |
| `%d` | Decoraciones (ramas, tags) | `(HEAD -> main, origin/main)` |

### Visualizacion grafica

```bash
# Historia grafica de todas las ramas
git log --graph --oneline --decorate --all

# Alias practico (usar en git config)
git log --graph --pretty=format:'%C(red)%h%Creset -%C(yellow)%d%Creset %s %C(green)(%cr) %C(blue)<%an>%Creset' --all
```

### Filtros utiles

```bash
# Por autor
git log --author="Juan Perez"
git log --author="juan@ejemplo.com"

# Por fecha
git log --since="2025-01-01" --until="2025-01-31"
git log --since="2 weeks ago"
git log --after="2025-01-15" --before="2025-02-01"

# Por mensaje de commit (busqueda)
git log --grep="bug"
git log --grep="fix" --grep="login" --all-match  # AND logico

# Por contenido (que archivos tocaron cierta funcion)
git log -S "function calcularTotal"     # Busca string en los diffs
git log -G "calcularTotal\(.*\)"        # Busca con expresion regular

# Por rango de commits
git log main..feature     # Commits en feature que no estan en main
git log HEAD~10..HEAD     # Ultimos 10 commits
git log main...feature    # Commits en main O feature pero no en ambos

# Por archivo
git log -- src/config.js
git log --oneline -- src/ tests/
```

```bash
# Ejemplo combinado: commits de Juan en la ultima semana sobre tests/
git log --author="Juan" --since="1 week ago" --oneline -- tests/
```

---

## 2.8 Ver Cambios

### git diff: comparaciones

```bash
# Working Directory vs Staging Area (lo que aun no esta staged)
git diff

# Staging Area vs Ultimo Commit (lo que se incluira en el proximo commit)
git diff --staged
git diff --cached

# Working Directory vs Ultimo Commit (todo lo modificado desde el ultimo commit)
git diff HEAD
```

### Visualizacion de diferencias

```
diff --git a/config.js b/config.js
index 8c2a1b9..d7f3e2c 100644
--- a/config.js
+++ b/config.js
@@ -10,7 +10,8 @@
 const config = {
-  port: 3000,
+  port: process.env.PORT || 3000,
   debug: true,
+  timeout: 5000,
 };
```

Lectura de un diff:
- `--- a/config.js`: archivo original
- `+++ b/config.js`: archivo modificado
- `@@ -10,7 +10,8 @@`: linea 10 del original (7 lineas mostradas), linea 10 del nuevo (8 lineas)
- Lineas con `-`: eliminadas
- Lineas con `+`: agregadas

### Opciones de git diff

```bash
# Comparar dos commits especificos
git diff HEAD~3..HEAD
git diff abc123..def456

# Comparar un archivo especifico
git diff -- src/config.js

# Mostrar estadisticas (resumen)
git diff --stat
git diff --stat HEAD~5..HEAD

# Mostrar nombres de archivos cambiados
git diff --name-only HEAD~1..HEAD
git diff --name-status HEAD~1..HEAD   # Con estado (M, A, D)

# Diff por palabras (util para textos, no codigo)
git diff --word-diff
git diff --word-diff-regex="\w+|[^[:space:]]"

# Ignorar cambios de espacios en blanco
git diff -w
git diff --ignore-all-space

# Diff entre ramas
git diff main..feature
git diff origin/main..HEAD

# Diff con merge-base (util para ver cambios de una rama completa)
git diff main...feature     # Cambios en feature desde que divergio de main
# A...B = diff entre el ancestro comun de A y B, y B
# A..B  = diff directo entre A y B
```

### git diff con herramientas externas

```bash
# Configurar herramienta visual de diff
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'

# Usar la herramienta configurada
git difftool
git difftool HEAD~1..HEAD
```

### Configurar estilo de conflictos recomendado

```bash
# Activar diff3 como estilo de marcadores de conflicto (altamente recomendado)
git config --global merge.conflictStyle diff3
```

Con `diff3`, los marcadores de conflicto incluyen una seccion adicional `|||||||` que muestra la version **base** (el ancestro comun), ademas de HEAD (ours) y la rama fusionada (theirs):

```
<<<<<<< HEAD
const PORT = 3000;
||||||| merged common ancestors
const PORT = 8080;
=======
const PORT = process.env.PORT || 8080;
>>>>>>> feature
```

Esto te permite ver no solo las dos versiones en conflicto, sino tambien como era originalmente antes de que ambas ramas lo modificaran, facilitando enormemente la decision de como resolver.

---

## 2.9 Eliminar y Mover Archivos

### git rm

Elimina archivos del working directory y del staging, preparando la eliminacion para el proximo commit:

```bash
# Eliminar archivo rastreado
git rm archivo.txt
git commit -m "Eliminar archivo obsoleto"

# Eliminar solo del staging, mantener en disco (dejar de rastrear)
git rm --cached archivo.txt
git rm --cached -r directorio/     # Recursivo

# Forzar eliminacion aunque el archivo tenga cambios sin commit
git rm -f archivo-modificado.txt

# Eliminar por patron
git rm '*.log'                     # Elimina todos los .log rastreados
git rm -r --cached node_modules/   # Dejar de rastrear node_modules
```

> **Uso comun:** `git rm --cached` es util cuando agregaste accidentalmente archivos al repositorio que deberian estar en `.gitignore` (como `node_modules/`, `.env`, o binarios compilados).

### git mv

Renombra o mueve archivos. Git detecta el renombrado automaticamente:

```bash
# Renombrar archivo
git mv viejo-nombre.txt nuevo-nombre.txt
# Equivalente a: mv viejo-nombre.txt nuevo-nombre.txt && git add --all

# Mover archivo a otro directorio
git mv src/helper.js src/utils/helper.js

# Git detecta renombrados incluso sin usar git mv
mv viejo-nombre.txt nuevo-nombre.txt
git add -A
# Git detectara que es un renombrado, no una eliminacion + creacion
```

Deteccion de renombrados en `git log`:

```bash
# Ver historial siguiendo renombrados
git log --follow archivo.txt

# Detectar renombrados en diff
git diff --find-renames HEAD~5..HEAD

# Ajustar umbral de similitud para deteccion (default 50%)
git diff -M90% HEAD~5..HEAD   # Solo detecta si 90% similar
```

---

## 2.10 Ignorar Archivos

### .gitignore

El archivo `.gitignore` le dice a Git que archivos o directorios debe ignorar intencionalmente. Normalmente se coloca en la raiz del repositorio y se versiona junto con el proyecto:

```bash
# Crear .gitignore
touch .gitignore
git add .gitignore
git commit -m "Agregar reglas de ignorados"
```

### Patrones de .gitignore

```gitignore
# Comentarios con #
# Lineas en blanco se ignoran

# Ignorar archivos especificos
*.log
.DS_Store
Thumbs.db

# Ignorar directorios completos
node_modules/
build/
dist/
.vscode/

# Negacion: NO ignorar un archivo que coincide con un patron anterior
*.log
!importante.log          # Este archivo no se ignora

# Ignorar archivos en un directorio especifico
logs/*.log               # Solo .log en logs/, no en subdirectorios
logs/**/*.log            # .log en logs/ y todos sus subdirectorios

# Ignorar archivo solo en la raiz
/config.yml              # Solo /config.yml, no subdir/config.yml

# Patrones con comodines
temp-*                   # Cualquier archivo que empiece con temp-
debug?.log               # debug0.log, debug1.log, etc. (? = un caracter)
```

### Ejemplo practico de .gitignore

```gitignore
# Dependencias
node_modules/
vendor/
__pycache__/
*.pyc

# Compilados y artefactos
build/
dist/
*.exe
*.dll
*.so
*.o

# Entorno y configuracion
.env
.env.local
*.secret
config/local.yml

# IDE y editores
.idea/
.vscode/
*.swp
*.swo
*~

# Sistema operativo
.DS_Store
Thumbs.db
desktop.ini

# Logs y temporales
*.log
*.tmp
tmp/
```

### .gitignore en Subdirectorios y Reglas en Cascada

Puedes tener multiples archivos `.gitignore` en diferentes directorios del proyecto. Las reglas se aplican en cascada:

```bash
# .gitignore en la raiz
*.log                        # Ignora .log en todo el proyecto

# src/.gitignore
*.log                        # Ignora .log solo en src/ y subdirectorios
!importante.log              # Excepcion local a src/
```

Reglas de precedencia:
1. Los patrones del `.gitignore` en un subdirectorio solo aplican a ese directorio y sus hijos.
2. Los patrones de un `.gitignore` mas cercano al archivo tienen prioridad sobre uno mas lejano.
3. Las negaciones (`!`) revierten patrones de nivel superior.
4. Los archivos ya rastreados (committed) nunca son ignorados, aunque los agregues a `.gitignore`.

> **Tip:** Para dejar de rastrear un archivo que ya fue commiteado, usa `git rm --cached <archivo>` primero, luego agregalo a `.gitignore`.

### Archivos globales de ignore

Para patrones personales que aplican a todos tus repositorios (archivos de tu editor, del SO, etc.):

```bash
# Crear un gitignore global
git config --global core.excludesfile ~/.gitignore_global

# Crear el archivo
cat > ~/.gitignore_global << 'EOF'
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Linux
*~
.directory

# Windows
Thumbs.db
ehthumbs.db
Desktop.ini

# Editores
.idea/
.vscode/
*.swp
*.swo
EOF
```

### Plantillas de .gitignore

GitHub mantiene plantillas para practicamente cualquier lenguaje y framework:

```bash
# Ver plantillas disponibles (si tienes acceso al repo git)
ls $(git --exec-path)/../share/git-core/templates/

# O descargar desde GitHub
# https://github.com/github/gitignore

# Ejemplo: .gitignore para Node.js
# https://raw.githubusercontent.com/github/gitignore/main/Node.gitignore
```

> **Regla de oro del .gitignore:** Solo debes ignorar archivos generados (compilados, dependencias, logs, datos de entorno). Nunca ignores archivos de codigo fuente o configuracion que otros desarrolladores necesiten.

### Verificar reglas de ignore

```bash
# Por que un archivo esta siendo ignorado?
git check-ignore -v archivo.log
# .gitignore:1:*.log   archivo.log

# Con -v, la salida muestra: <fuente>:<linea>:<patron> <archivo>
# La fuente puede ser:
#   .gitignore          → archivo local
#   ~/.gitignore_global → configuración global (core.excludesfile)
#   .git/info/exclude   → exclusiones especificas del repo (no versionado)

git check-ignore -v dist/bundle.js
# .gitignore:3:dist/   dist/bundle.js

git check-ignore -v src/temp.log
# src/.gitignore:1:*.log   src/temp.log

# Verificar si un archivo NO es ignorado (sin salida = no ignorado)
git check-ignore -v src/app.js
# (sin salida: el archivo no coincide con ningun patron)
```

---

## 2.11 Atributos de Archivo: `.gitattributes`

El archivo `.gitattributes` (colocado en la raiz del repositorio y versionado) permite configurar el comportamiento de Git **por tipo de archivo**. Es mas granular y preferible a configuraciones globales como `core.autocrlf`.

### Que controla `.gitattributes`

| Directiva | Proposito | Ejemplo |
|---|---|---|
| `text` | Normalizacion de finales de linea (LF/CRLF) | `* text=auto` |
| `eol` | Forzar final de linea especifico | `*.sh text eol=lf` |
| `binary` | Marcar archivo como binario | `*.png binary` |
| `diff` | Driver de diff personalizado | `*.docx diff=word` |
| `merge` | Driver de merge personalizado | `*.json merge=json-tool` |
| `linguist-language` | Override de lenguaje para GitHub stats | `*.ec linguist-language=Go` |
| `export-ignore` | Excluir de `git archive` | `tests/ export-ignore` |

### Ejemplo practico

```gitattributes
# Normalizar todos los archivos de texto a LF
* text=auto

# Forzar shell scripts a LF (Unix)
*.sh text eol=lf

# Forzar batch files a CRLF (Windows)
*.bat text eol=crlf

# Marcar binarios explicitamente
*.png binary
*.jpg binary
*.pdf binary

# Usar herramienta externa para diffs de Word
*.docx diff=word

# No incluir tests en git archive
tests/ export-ignore
```

> **Recomendacion:** Siempre prefiere `.gitattributes` sobre `core.autocrlf` para el manejo de finales de linea. Da control fino por tipo de archivo y se versiona con el proyecto, garantizando consistencia en todo el equipo.

---

## 2.12 Firma de Commits

Firmar commits y tags garantiza criptograficamente que el autor es quien dice ser. Es esencial en proyectos que requieren verificacion de identidad o cumplen con estandares de seguridad (SLSA, FedRAMP).

### Firmado con GPG

```bash
# 1. Generar una clave GPG (si no tienes)
gpg --full-generate-key
# Elegir RSA and RSA, 4096 bits, y una fecha de expiracion

# 2. Listar tus claves GPG
gpg --list-secret-keys --keyid-format=long
# /Users/usuario/.gnupg/pubring.kbx
# sec   rsa4096/3AA5C34371567BD2 2025-01-15 [SC]
# uid         [ultimate] Tu Nombre <tu-email@ejemplo.com>
# ssb   rsa4096/42B317FD4BA89E7A 2025-01-15 [E]

# 3. Configurar Git con tu clave
git config --global user.signingKey 3AA5C34371567BD2

# 4. Firmar un commit
git commit -S -m "Commit firmado con GPG"
# -S = firmar, -s = agregar Signed-off-by (son distintos)

# 5. Firmar un tag
git tag -s v1.0.0 -m "Release v1.0.0"

# 6. Verificar firmas en el historial
git log --show-signature
git log --show-signature -5
```

Salida de `git log --show-signature`:
```
commit a1b2c3d4e5f6...
gpg: Signature made Wed Jan 15 14:30:00 2025 CST
gpg:                using RSA key 3AA5C34371567BD2
gpg: Good signature from "Tu Nombre <tu-email@ejemplo.com>"
Author: Tu Nombre <tu-email@ejemplo.com>
Date:   Wed Jan 15 14:30:00 2025 -0600
    Mensaje del commit
```

### Firmado con SSH

Desde Git 2.34, puedes firmar con tu llave SSH existente (sin GPG):

```bash
# Configurar firma SSH
git config --global gpg.format ssh
git config --global user.signingKey ~/.ssh/id_ed25519.pub

# Firmar commits con SSH
git commit -S -m "Commit firmado con SSH"
```

### Commit signing en GitHub/GitLab

| Estado | Significado |
|---|---|
| **Verified** (verde) | Firma valida, clave asociada a cuenta |
| **Unverified** (gris) | Firma valida pero clave no asociada |
| **Unsigned** | Sin firma |

> **Tip:** Configura `git config --global commit.gpgSign true` para firmar todos los commits automaticamente (o `git config --global tag.gpgSign true` para tags).

---

## Resumen del Capitulo

- `git init` crea un repositorio nuevo; `git clone` copia uno existente (incluyendo clones superficiales con `--depth`).
- Los archivos pasan por 4 estados: **Untracked -> Staged -> Committed -> Modified**.
- `git add` prepara archivos para commit; `git add -p` permite seleccionar cambios parciales.
- `git commit` guarda snapshots permanentes. Usa `--amend` solo en commits locales no compartidos.
- `git status` muestra el estado actual; `-s` da formato compacto con codigos de dos columnas.
- `git log` ofrece multiples formatos (`--oneline`, `--graph`, `--stat`) y filtros (`--author`, `--since`, `--grep`).
- `git diff` compara entre working, staging y commits; `--staged` muestra lo que se va a commitear.
- `git rm` y `git mv` gestionan eliminacion y renombrado; `--cached` solo afecta al staging.
- `.gitignore` define patrones de archivos a ignorar global o localmente. Se pueden colocar `.gitignore` en subdirectorios con reglas en cascada.
- `.gitattributes` permite configurar finales de linea, diffs y merges por tipo de archivo, siendo preferible a `core.autocrlf`.
- La **firma de commits** (`git commit -S`) con GPG o SSH garantiza la autoria criptografica, configurable con `user.signingKey`.

---

## Ejercicios Propuestos

1. **Laboratorio de ciclo de vida**: Crea un repositorio y sigue este flujo paso a paso documentando `git status -s` en cada etapa: a) Crea 3 archivos, b) Agrega 2 al staging, c) Haz commit, d) Modifica uno de los committed y crea un cuarto archivo, e) Interpreta la salida de `git status -s` en cada paso.

2. **Commits atomicos con patch**: Crea un archivo con 5 funciones. Modifica 3 de ellas con cambios no relacionados (ej: corrige un bug, agrega un log, cambia un valor por defecto). Usa `git add -p` para separar los cambios en 3 commits atomicos, cada uno con su mensaje descriptivo. Verifica con `git log --oneline -3`.

3. **Exploracion de historial**: Clona el repositorio de Git mismo (`https://github.com/git/git.git` con `--depth 50`). Usa `git log` para responder: a) Los 5 commits mas recientes en una linea, b) Cuantos commits de "Junio C Hamano" hay, c) Commits de la ultima semana que mencionan "fix" en el mensaje. Documenta los comandos usados.

4. **Gestion de .gitignore**: Crea un proyecto simulado con archivos de varios tipos (`.js`, `.log`, `.env`, `.swp`, `.exe`, `.DS_Store`). Escribe un `.gitignore` que ignore todo excepto los `.js`. Verifica que funciona con `git check-ignore -v` y `git status`.

5. **Comparacion de versiones**: En un repositorio con al menos 5 commits, usa `git diff` para: a) Ver los cambios en el working directory, b) Ver que se incluira en el proximo commit, c) Comparar HEAD con hace 3 commits, d) Ver solo los nombres de archivos cambiados entre los 2 ultimos commits. Explica la diferencia entre `git diff`, `git diff --staged` y `git diff HEAD`.

---

← [Capítulo anterior](01-introduccion.md) | [Inicio](README.md) | [Capítulo siguiente →](03-ramas.md)
