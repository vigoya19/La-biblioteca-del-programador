# Capitulo 1: Introduccion a Git

## 1.1 Que es Git

Git es un **sistema de control de versiones distribuido** (DVCS por sus siglas en ingles) disenado para manejar desde proyectos pequenos hasta gigantescos con velocidad y eficiencia. A diferencia de otros sistemas, Git almacena el historial completo del proyecto en cada copia local, lo que permite trabajar sin conexion y realizar practicamente cualquier operacion de forma inmediata.

### Caracteristicas principales

| Caracteristica | Descripcion |
|---|---|
| **Distribuido** | Cada desarrollador tiene una copia completa del repositorio |
| **Rapido** | Operaciones locales practicamente instantaneas |
| **Integro** | Cada objeto se identifica con un hash SHA-1 |
| **No lineal** | Soporta miles de ramas paralelas de forma eficiente |
| **Seguro** | No se puede perder historial si existe al menos una copia |
| **Open source** | Licencia GPL v2, mantenido por la comunidad |

### Que es el control de versiones

El control de versiones es un sistema que registra los cambios realizados sobre un archivo o conjunto de archivos a lo largo del tiempo, de manera que puedas recuperar versiones especificas mas adelante.

```
Sin control de versiones:
  proyecto/
    informe.doc
    informe-v2.doc
    informe-v2-revisado.doc
    informe-FINAL.doc
    informe-FINAL-v2.doc
    informe-FINAL-REAL.doc

Con Git:
  proyecto/
    informe.doc  <-- un solo archivo, Git guarda el historial
```

> **Tip:** Con Git puedes revertir archivos (o un proyecto entero) a un estado anterior, comparar cambios a lo largo del tiempo, ver quien modifico algo por ultima vez, quien introdujo un bug y cuando, y mucho mas.

---

## 1.2 Historia de Git

### El nacimiento de Git (2005)

La historia de Git comienza con el desarrollo del kernel de Linux, el proyecto de software libre mas grande del mundo. Entre 1991 y 2002, los cambios en el kernel se gestionaban mediante parches y archivos comprimidos. En 2002, el equipo comenzo a usar **BitKeeper**, un sistema de control de versiones distribuido propietario.

En abril de 2005, la relacion entre la comunidad de Linux y BitMover (la empresa detras de BitKeeper) se rompio cuando la empresa revoco la licencia gratuita. **Linus Torvalds**, creador de Linux, decidio construir su propio sistema de control de versiones. Sus metas eran:

1. **Rapidez**: aplicar parches y actualizar el historial en segundos, no minutos.
2. **Diseno simple**: estructura de datos clara y robusta.
3. **Soporte para desarrollo no lineal**: miles de ramas paralelas.
4. **Totalmente distribuido**: cada copia es un repositorio completo.
5. **Manejo eficiente de proyectos enormes**: como el kernel de Linux.

### Evolucion

```bash
# Git fue lanzado en abril de 2005. El primer commit del proyecto:
commit e83c5163316f89bfbde7d9ab23ca2e25604af290
Author: Linus Torvalds <torvalds@ppc970.osdl.org>
Date:   Thu Apr 7 15:13:13 2005 -0700

    Initial revision of "git", the information manager from hell
```

| Ano | Hito |
|---|---|
| 2005 | Nace Git. Linus escribe el nucleo inicial en 10 dias |
| 2005 | Junio: Git v1.0. Jun Hamano asume el mantenimiento |
| 2007 | Git v1.5: interfaz mas amigable, `git add -i` |
| 2008 | Lanzamiento de **GitHub**, impulsa la adopcion masiva |
| 2012 | Git v1.8: mejoras en rendimiento y usabilidad |
| 2020 | Git v2.28: `main` como rama por defecto (en lugar de `master`) |
| Actualidad | Git es el VCS mas usado del mundo. +90% de los desarrolladores |

---

## 1.3 Sistemas de Control de Versiones

### Clasificacion

Los sistemas de control de versiones se clasifican en tres generaciones, cada una resolviendo las limitaciones de la anterior:

```
Evolucion de los VCS:

[Locales]  --->  [Centralizados]  --->  [Distribuidos]
   RCS              SVN, CVS            Git, Mercurial
  (1982)            (1990)               (2005)
```

### VCS Locales

Los sistemas locales almacenan los cambios en una base de datos local. El mas conocido es **RCS** (Revision Control System), que guarda conjuntos de parches (diferencias entre versiones) en un formato especial en disco.

```
┌─────────────┐
│  Base de    │
│  datos de   │  <-- guarda parches, no versiones completas
│  versiones  │
└─────────────┘
       ▲
       │ checkout
       │
┌─────────────┐
│  Archivos   │
│  locales    │
└─────────────┘
```

Problemas:
- Solo un usuario puede trabajar a la vez.
- Todo esta en una sola maquina; si falla el disco, se pierde todo.

### VCS Centralizados (CVCS)

Sistemas como **CVS**, **Subversion (SVN)** y **Perforce** tienen un unico servidor central que contiene todos los archivos versionados, y los clientes descargan los archivos desde ese servidor.

```
┌──────────────────────────┐
│    Servidor Central      │
│  (repositorio completo)  │
└──────┬────────┬──────────┘
       │        │
   checkout checkout
       │        │
       ▼        ▼
┌──────────┐ ┌──────────┐
│Cliente A │ │Cliente B │  <-- solo tienen la ultima version
└──────────┘ └──────────┘
```

Ventajas sobre los locales:
- Todos saben lo que hacen los demas.
- Los administradores tienen control fino sobre permisos.

Desventajas:
- **Punto unico de fallo**: si el servidor cae, nadie puede trabajar.
- Si el servidor se corrompe sin backups, se pierde todo el historial.
- Operaciones lentas (requieren conexion de red).

### VCS Distribuidos (DVCS)

En sistemas como **Git**, **Mercurial** o **Bazaar**, cada cliente replica completamente el repositorio, incluyendo todo el historial.

```
┌──────────────────────────┐
│    Servidor (opcional)   │
│    repositorio completo  │
└──────┬────────┬──────────┘
       │        │
   push/pull push/pull
       │        │
       ▼        ▼
┌──────────┐ ┌──────────┐
│Cliente A │ │Cliente B │
│(repo     │ │(repo     │  <-- CADA UNO tiene historial COMPLETO
│completo) │ │completo) │
└──────────┘ └──────────┘
```

Ventajas:
- **Sin punto unico de fallo**: cualquier copia puede restaurar el servidor.
- **Operaciones locales rapidisimas**: commit, diff, log, todo offline.
- **Ramas livianas**: crear y fusionar ramas es instantaneo.
- **Flujos de trabajo flexibles**: centralizado, jerarquico, pull requests, etc.

> **Nota:** Aunque Git es distribuido, la mayoria de los equipos usan un repositorio "central" (como GitHub o GitLab) como punto de sincronizacion. Esto combina lo mejor de ambos mundos: la flexibilidad del DVCS con la simplicidad del modelo centralizado.

---

## 1.4 Filosofia de Git

Entender la filosofia detras de Git es la clave para usarlo efectivamente. Git rompe con varios paradigmas de sistemas anteriores.

### Snapshots, no diferencias

La mayoria de los VCS (CVS, SVN, Perforce) almacenan la informacion como una lista de cambios por archivo (cambios basados en deltas):

```
VCS tradicionales (basados en deltas):

Archivo A
  v1 ──Δ── v2 ──Δ── v3 ──Δ── v4

Archivo B
  v1 ──Δ── v2 ──Δ── v3
```

Git, en cambio, **toma una foto instantanea (snapshot)** de todos los archivos en cada commit. Si un archivo no cambio, Git no lo vuelve a almacenar; simplemente crea un enlace al archivo identico anterior.

```
Git (basado en snapshots):

commit1 ──► snapshot A (v1), B (v1), C (v1)
commit2 ──► snapshot A (v2), B (v1*), C (v1*)   * = enlace
commit3 ──► snapshot A (v2*), B (v2), C (v1*)
```

> Esto es lo que hace que las operaciones como `git diff`, `git log` y `git checkout` sean extremadamente rapidas: Git simplemente compara referencias a snapshots completos.

### Casi todo es local

La mayoria de las operaciones en Git solo necesitan archivos y recursos locales; no se requiere conexion de red. Esto significa que puedes:

- Hacer commit sin estar conectado.
- Ver el historial completo sin latencia de red.
- Comparar versiones aunque no tengas internet.
- Crear ramas y hacer merge localmente.

```bash
# Sin conexion a internet, todo esto funciona:
git log --oneline --graph --all
git diff HEAD~3..HEAD
git checkout -b feature-experimental
git commit -m "Prueba de concepto offline"
```

### Integridad SHA-1 y SHA-256

Todo en Git tiene un **checksum** (suma de verificación) antes de ser almacenado y es referenciado por ese checksum. Piensa en un checksum como la **huella digital** de tus archivos: es un código único de 40 caracteres que identifica de forma irrepetible cada cambio que guardas en Git. Si alguien altera un solo carácter de tu código, la huella digital cambia completamente y Git lo detecta al instante. Esto significa que es imposible modificar el contenido de un archivo o directorio sin que Git lo detecte.

Históricamente Git ha usado **SHA-1**:

```
SHA-1: 40 caracteres hexadecimales
Ejemplo: 24b9da6552252987aa493b52f8696cd6d3b00373
```

Desde Git 2.45+ existe soporte para **SHA-256** como alternativa criptográfica más robusta. SHA-256 genera huellas digitales más largas y seguras (64 caracteres en vez de 40). Para el uso diario de Git, la diferencia entre SHA-1 y SHA-256 es irrelevante — ambos funcionan perfectamente. SHA-256 es una preparación para el futuro, por si alguna vez SHA-1 dejara de ser suficientemente seguro:

```bash
# Crear un repositorio con hashing SHA-256
git init --object-format=sha256 mi-proyecto
```

Cada commit, cada archivo, cada directorio tiene su propio hash calculado a partir de su contenido. Esta es la base de la integridad de Git: si alguien altera un solo bit, el hash cambia y Git lo detecta inmediatamente.

> **Nota:** SHA-1 y SHA-256 no son interoperables. No puedes hacer push/pull entre repositorios con distinto formato de hash. SHA-256 es aun experimental en algunas operaciones.

### Solo agrega datos

En Git, practicamente todas las operaciones **solo agregan datos** al repositorio. Es muy dificil hacer algo que no sea recuperable o que borre informacion de forma permanente. Una vez que haces un commit, es extremadamente dificil perder esos cambios (especialmente si ya los compartiste con otro repositorio).

```bash
# Incluso si "borras" un commit con git reset, aun puedes recuperarlo
# mientras no haya pasado el garbage collection:
git reflog                    # Muestra TODAS las operaciones recientes
git reset --hard HEAD@{3}     # Recupera estado de hace 3 operaciones
```

### Los tres estados

Este es el concepto fundamental de Git. Los archivos en tu repositorio pueden estar en tres estados:

```
┌──────────────┐     git add     ┌──────────────┐    git commit    ┌──────────────┐
│  Working     │ ──────────────► │   Staging    │ ──────────────► │   .git       │
│  Directory   │                 │    Area       │                 │  Directory   │
│ (modified)   │ ◄────────────── │   (staged)    │ ◄────────────── │ (committed)  │
└──────────────┘   git restore   └──────────────┘   git checkout   └──────────────┘
```

Las tres secciones principales:

| Seccion | Descripcion | Estado del archivo |
|---|---|---|
| **Working Directory** | Tus archivos en disco. Extraes una version del repositorio para trabajar. | Modified |
| **Staging Area** | Area de preparacion (index). Construyes el proximo commit. | Staged |
| **.git directory** | Donde Git almacena los metadatos y la base de datos de objetos. | Committed |

El flujo basico de trabajo en Git:

```
1. Modificas archivos en tu Working Directory.
2. Seleccionas los cambios que iran en el proximo commit (Staging Area con git add).
3. Haces el commit, que toma los archivos del Staging Area y los almacena permanentemente.
```

---

## 1.5 Instalacion de Git

### macOS

Hay varias opciones para instalar Git en macOS:

```bash
# Opcion 1: Xcode Command Line Tools (incluye Git)
xcode-select --install

# Opcion 2: Homebrew (version mas reciente)
brew install git

# Opcion 3: Descarga oficial desde https://git-scm.com/download/mac
# Abre el .dmg y sigue el instalador
```

Verificar la instalacion:

```bash
git --version
# git version 2.44.0
```

### Linux

```bash
# Debian / Ubuntu
sudo apt update
sudo apt install git-all

# Fedora
sudo dnf install git-all

# Arch Linux
sudo pacman -S git

# Compilar desde fuente (ultima version)
wget https://github.com/git/git/archive/refs/tags/v2.44.0.tar.gz
tar -xzf v2.44.0.tar.gz
cd git-2.44.0
make configure
./configure --prefix=/usr/local
make all
sudo make install
```

### Windows

```bash
# Opcion 1: Instalador oficial
# Descarga desde https://git-scm.com/download/win y ejecuta el .exe

# Opcion 2: winget (Windows Package Manager)
winget install --id Git.Git -e --source winget

# Opcion 3: Chocolatey
choco install git
```

La instalacion en Windows incluye:
- **Git Bash**: terminal con comandos Unix (`bash`, `ls`, `grep`, etc.)
- **Git GUI**: interfaz grafica basica
- **Git Credential Manager**: gestion de credenciales para HTTPS

---

## 1.6 Configuracion Inicial

Git viene con una herramienta llamada `git config` que te permite personalizar su comportamiento. La configuracion se almacena en tres niveles:

| Nivel | Archivo | Alcance |
|---|---|---|
| **Sistema** | `/etc/gitconfig` | Todos los usuarios del sistema |
| **Usuario** | `~/.gitconfig` o `~/.config/git/config` | Un usuario especifico |
| **Repositorio** | `.git/config` | Un repositorio especifico |

Cada nivel sobrescribe al anterior: repositorio > usuario > sistema.

### Identidad (obligatorio)

Lo primero que debes hacer despues de instalar Git es configurar tu identidad. Esta se adjunta a cada commit que hagas:

```bash
git config --global user.name "Tu Nombre Completo"
git config --global user.email "tu-email@ejemplo.com"
```

> **Importante:** Si usas GitHub, GitLab o Bitbucket, asegurate de que el email coincida con el de tu cuenta para que los commits se asocien correctamente a tu perfil.

### Editor de texto

Git necesita un editor de texto para escribir mensajes de commit y resolver conflictos:

```bash
# VS Code (recomendado)
git config --global core.editor "code --wait"

# Vim
git config --global core.editor vim

# Neovim
git config --global core.editor nvim

# Emacs
git config --global core.editor emacs

# Sublime Text (macOS)
git config --global core.editor "subl -n -w"
```

### Repositorios Bare (Desnudos)

Un repositorio **bare** es un repositorio Git sin working directory (sin archivos visibles para editar). No contiene una copia extraída de los archivos; solo contiene el contenido de `.git/`. Piensa en ello como un **almacén puro de historial**: no tiene escritorio ni carpetas que puedas abrir y editar, solo guarda todos los cambios y versiones. Es lo que usan GitHub, GitLab y servidores Git internos para almacenar tu código. Tú como usuario nunca trabajarás directamente en un repositorio bare; simplemente haces `git push` hacia él.

Se utilizan exclusivamente como punto de sincronización central (servidor), nunca para trabajo directo:

```bash
git init --bare servidor-central.git
```

Estructura de un repositorio bare:
```
servidor-central.git/
  HEAD
  config
  objects/
  refs/
  hooks/
  ...
```

Caracteristicas:
- **Sin working directory**: no puedes editar archivos directamente en el.
- **No se hacen commits locales** en un bare (no tiene index ni working tree).
- **Sirve como destino de push**: GitHub, GitLab y servidores Git internos usan repositorios bare.
- **Convencion de nomenclatura**: suelen terminar en `.git` (ej. `proyecto.git`).

> **Dato clave:** Cuando clonas desde GitHub, estas clonando desde un repositorio bare. Tus `git push` envian cambios a ese bare, que solo almacena objetos y referencias.

### Nombre de la rama por defecto

Desde Git 2.28, puedes configurar el nombre de la rama inicial al crear un nuevo repositorio:

```bash
git config --global init.defaultBranch main
```

### Manejo de finales de línea (CRLF)

> **📖 ¿Qué es CRLF?** Cuando presionas Enter en un archivo de texto, tu sistema operativo agrega un carácter invisible al final de la línea. El problema es que Windows usa **dos caracteres** (CRLF: Carriage Return + Line Feed), mientras que macOS y Linux usan **uno solo** (LF: Line Feed). Cuando personas con diferentes sistemas operativos trabajan en el mismo proyecto, esto puede causar conflictos falsos donde Git piensa que TODAS las líneas cambiaron, cuando en realidad solo cambió el carácter invisible del final. La configuración de abajo resuelve este problema automáticamente.

Este es un punto crítico cuando trabajas en equipos multiplataforma:

```bash
# Windows: convierte CRLF a LF al commit, LF a CRLF al checkout
git config --global core.autocrlf true

# macOS/Linux: convierte CRLF a LF al commit
git config --global core.autocrlf input

# No hacer ninguna conversion (no recomendado para equipos mixtos)
git config --global core.autocrlf false
```

| Configuracion | Al commit | Al checkout |
|---|---|---|
| `true` (Windows) | CRLF -> LF | LF -> CRLF |
| `input` (macOS/Linux) | CRLF -> LF | Sin cambio |
| `false` | Sin cambio | Sin cambio |

> **Recomendacion moderna:** `core.autocrlf` aplica globalmente a todos los archivos del repositorio, lo cual puede ser problematico en binarios o formatos especificos. La alternativa preferida hoy es usar `.gitattributes` con la directiva `* text=auto`, que permite un control mas granular por tipo de archivo:
>
> ```bash
> # En .gitattributes (raiz del repositorio):
> * text=auto
> *.jpg binary
> *.png binary
> *.bat text eol=crlf
> *.sh  text eol=lf
> ```

### Colores y aliases

```bash
# Habilitar colores en la salida de Git
git config --global color.ui auto

# Crear aliases utiles
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.unstage "reset HEAD --"
git config --global alias.last "log -1 HEAD"
```

### Configuracion de proxy (si es necesario)

```bash
# Configurar proxy HTTP
git config --global http.proxy http://proxy.ejemplo.com:8080
git config --global https.proxy https://proxy.ejemplo.com:8080

# Desactivar proxy para dominios especificos
git config --global http.noProxy "ejemplo.com,.local"
```

---

## 1.7 Cuando NO usar Git

Aunque Git es la herramienta de control de versiones mas popular, no es la solucion optima para todos los escenarios. Conocer sus limitaciones te ayudara a elegir la herramienta correcta.

### Archivos binarios grandes sin LFS

Git almacena cada version completa (snapshot) de cada archivo. Para binarios grandes que cambian frecuentemente (assets de videojuegos, modelos 3D, imagenes de alta resolucion), el repositorio crece desproporcionadamente:

```bash
# Un archivo .psd de 200 MB que cambia 10 veces = ~2 GB en el repo
# Cada git clone descarga todas las versiones
```

**Alternativas:**
- **Git LFS** (Large File Storage): extension oficial que reemplaza binarios por punteros.
- **Perforce** o **Plastic SCM**: disenados para activos de juegos.
- **DVC** (Data Version Control): para datasets de machine learning.

### Assets de videojuegos

Los motores de juego generan miles de archivos binarios (texturas, mapas de iluminacion, modelos compilados). Git no maneja bien:
- Archivos binarios muy grandes (> 100 MB).
- Directorios con cientos de miles de archivos.
- Escenas de Unity/Unreal que son binarias y generan conflictos de merge imposibles de resolver manualmente.

**Alternativas:** Perforce (estandar en AAA), Plastic SCM (Unity), Git LFS, o Anchorpoint.

### Datasets de Machine Learning

Un dataset de entrenamiento de 50 GB que se versiona regularmente:
- Git se vuelve extremadamente lento (hashear 50 GB de datos).
- Los clones consumen ancho de banda y disco innecesariamente.
- No necesitas diffs linea-por-linea en CSVs binarios o imagenes.

**Alternativas:**
- **DVC**: versiona punteros ligeros en Git, almacena datos en S3/GCS/Azure.
- **LakeFS**: versionado estilo Git sobre data lakes.
- **DVC + Git**: combinacion ideal: Git para codigo, DVC para datos.

### Otras limitaciones

| Escenario | Problema | Alternativa |
|---|---|---|
| **Archivos enormes individuales (>500 MB)** | Lento, alta memoria | Git LFS, asset managers |
| **Repos monolito (+1M archivos)** | `git status` lento, alto uso de RAM | Scalar (Microsoft), GVFS |
| **Documentos ofimaticos** | Diffs no legibles (.docx, .xlsx) | SharePoint, Google Docs |
| **Versionado de base de datos** | No versiona esquemas SQL | Flyway, Liquibase |
| **Archivos con permisos estrictos** | Git no almacena permisos Unix completos | Herramientas de configuracion (Ansible, etc.) |

### Cuando Git SÍ es la herramienta correcta

Git sobresale con:
- Codigo fuente (texto plano).
- Documentacion en formato texto (Markdown, AsciiDoc, LaTeX).
- Archivos de configuracion.
- Proyectos con colaboracion distribuida.
- Historial de cambios rastreable y auditable.

> **Regla practica:** Si tus archivos son texto plano que se beneficia de `diff` linea por linea, Git es la opcion correcta. Si trabajas principalmente con binarios grandes y cambiantes, complementa Git con LFS o evalua alternativas especializadas.

---

## 1.8 Verificar la Configuracion

### Listar toda la configuracion

```bash
# Muestra toda la configuracion activa (todos los niveles combinados)
git config --list

# Configuracion por nivel
git config --system --list      # Solo sistema
git config --global --list      # Solo usuario
git config --local --list       # Solo repositorio actual
```

Salida tipica:

```
user.name=Tu Nombre Completo
user.email=tu-email@ejemplo.com
core.editor=code --wait
init.defaultbranch=main
color.ui=auto
core.autocrlf=input
```

### Editar la configuracion directamente

```bash
# Abre el archivo de configuracion en el editor configurado
git config --global --edit
```

### Ver un valor especifico

```bash
git config user.name         # Busca en el nivel mas cercano
git config --global user.name # Especifica el nivel
```

El archivo `~/.gitconfig` tiene este formato:

```ini
[user]
    name = Tu Nombre Completo
    email = tu-email@ejemplo.com

[core]
    editor = code --wait
    autocrlf = input

[init]
    defaultBranch = main

[alias]
    co = checkout
    br = branch
    lg = log --oneline --graph --decorate --all
```

---

## 1.9 Ayuda y Documentacion

Git tiene una documentacion integrada excepcional. Saber como acceder a ella es esencial.

### Ayuda desde la terminal

```bash
# Ayuda general (resumen de comandos)
git help

# Ayuda de un comando especifico (tres formas equivalentes)
git help <comando>
git <comando> --help
man git-<comando>
```

```bash
# Ejemplos
git help config       # Abre la pagina del manual de configuracion
git commit --help     # Abre la ayuda del comando commit
man git-branch        # Manual de git branch
```

### Ayuda resumida (-h)

Si solo necesitas un recordatorio rapido de las opciones:

```bash
# Guia de referencia rapida (mas corta que --help)
git commit -h
git log -h
git branch -h
```

Salida de ejemplo:

```
usage: git commit [-a | --interactive | --patch] [-s] [-v] [-u<mode>] [--amend]
                  [--dry-run] [(-c | -C | --squash) <commit> | --fixup [(amend|reword):]<commit>)]
                  [-F <file> | -m <msg>] [--allow-empty-message] [--no-verify]
                  [--author=<author>] [--date=<date>] [--cleanup=<mode>]
                  [--status | --no-status] [-i | -o] [--pathspec-from-file=<file>]
                  [-S[<keyid>]] [--] [<pathspec>...]
```

### Documentacion oficial

- **git-scm.com**: sitio oficial con documentacion completa y libro Pro Git gratuito.
- **Pro Git** (libro): disponible gratis en [git-scm.com/book/es](https://git-scm.com/book/es).
- **Man pages**: `man gittutorial`, `man giteveryday`, `man gitglossary`.
- **Comunidad**: Stack Overflow, GitHub Discussions, foros especializados.

> **Tip profesional:** Acostumbrate a usar `git <comando> -h` antes de buscar en internet. El 90% de las veces la respuesta esta en la documentacion integrada.

---

## Resumen del Capitulo

- Git es un sistema de control de versiones **distribuido**, rapido y seguro.
- Fue creado por **Linus Torvalds** en 2005 para el desarrollo del kernel de Linux.
- Los VCS evolucionaron de locales (RCS) a centralizados (SVN, CVS) a distribuidos (Git, Mercurial).
- La filosofia de Git se basa en: **snapshots** (no diferencias), operaciones **locales**, **integridad SHA-1/SHA-256**, **solo agrega datos**, y los **tres estados** (working, staging, committed).
- La instalacion es sencilla en macOS (`brew`), Linux (`apt`/`yum`) y Windows (instalador).
- La configuracion inicial minima es `user.name` y `user.email`. Para finales de linea, `.gitattributes` es preferible a `core.autocrlf`.
- Los repositorios **bare** (sin working directory) son la base de servidores Git como GitHub y GitLab.
- Git no es optimo para archivos binarios grandes, assets de videojuegos, o datasets de ML sin herramientas complementarias como **LFS** o **DVC**.
- Git tiene ayuda integrada accesible con `git help`, `man` y `-h`.

---

## Ejercicios Propuestos

1. **Instalacion y configuracion**: Instala Git en tu sistema operativo y configura tu nombre, email, editor y alias personalizados. Verifica la configuracion con `git config --list`. Documenta que comandos usaste.

2. **Exploracion de ayuda**: Investiga las 5 opciones mas importantes de `git config` usando `git config -h`. Describe cada una con un ejemplo practico. Haz lo mismo para `git init`.

3. **Comparacion de VCS**: Investiga brevemente sobre **Mercurial** y escribe una tabla comparativa con Git. Incluye: modelo de ramas, rendimiento, curva de aprendizaje, comunidad y herramientas de hosting.

4. **Configuracion avanzada**: Configura `core.autocrlf` de forma adecuada para tu sistema operativo. Investiga que otros parametros de `core.*` existen y escribe un breve resumen de 3 que consideres utiles.

5. **Laboratorio de los tres estados**: Crea un repositorio nuevo y un archivo. Sigue este flujo: modifica el archivo, agregalo al staging, haz commit, modificalo de nuevo sin agregar. Usa `git status` entre cada paso y documenta como cambia el estado del archivo en cada etapa.

---

[Inicio](README.md) | [Capítulo siguiente →](02-fundamentos.md)
