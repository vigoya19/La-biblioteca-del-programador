# Capítulo 12: Estrategias de Fusión y Conflictos

Los conflictos de fusión son, sin duda, el aspecto que más temor genera en quienes trabajan con Git. Sin embargo, entender su naturaleza y dominar las herramientas para resolverlos transforma esa ansiedad en confianza. En este capítulo exploraremos a fondo qué son los conflictos, por qué ocurren, cómo resolverlos y —más importante aún— cómo prevenirlos.

## 12.1 ¿Qué es un Conflicto y Por Qué Ocurre?

Un conflicto en Git ocurre cuando el sistema no puede determinar automáticamente cómo combinar dos cambios divergentes. Git es extraordinariamente bueno fusionando modificaciones, pero cuando dos ramas alteran la misma porción de un archivo de maneras incompatibles, delega la decisión al desarrollador.

```
        A---B---C  (feature)
       /
  o---o---D---E  (main)
```

En el diagrama anterior, si tanto `C` como `E` modifican las mismas líneas de un archivo, Git no puede saber cuál versión conservar. El conflicto es una señal, no un error: Git está diciendo "necesito tu criterio humano para decidir".

> **Nota mental:** Un conflicto no significa que algo salió mal. Significa que Git te está protegiendo de una pérdida silenciosa de código. Es una red de seguridad, no un castigo.

## 12.2 Tipos de Conflictos

No todos los conflictos son iguales. Conocer los distintos tipos permite anticiparlos y resolverlos con más eficacia.

### 12.2.1 Conflicto de Contenido (Mismas Líneas)

El más común. Dos ramas modifican las mismas líneas del mismo archivo. Por ejemplo:

```bash
# En la rama feature:
echo "DEBUG=True" >> config.py
git add config.py && git commit -m "Activar debug"

# En la rama main:
echo "DEBUG=False" >> config.py
git add config.py && git commit -m "Desactivar debug"
```

Al intentar fusionar, Git detecta que la misma línea fue modificada de forma incompatible.

### 12.2.2 Archivo Modificado en Una Rama, Eliminado en Otra

También conocido como conflicto de modificación/eliminación.

```bash
# Rama feature: modifica utils.py
# Rama main: elimina utils.py porque se movió a helpers.py

git merge feature
# CONFLICT (modify/delete): utils.py deleted in HEAD and modified in feature
```

Git pregunta: ¿quieres conservar el archivo modificado o aceptar la eliminación?

### 12.2.3 Archivo Renombrado en Ambas Ramas

Conflicto de renombre/renombre:

```bash
# Rama feature: renombra old_name.py -> feature_name.py
# Rama main: renombra old_name.py -> main_name.py
```

Git no sabe cuál de los dos nombres conservar. Ambos archivos nuevos aparecerán en el área de trabajo.

### 12.2.4 Conflicto de Permisos

Menos frecuente pero posible cuando el modo de archivo (por ejemplo, 100644 vs 100755) cambia en ambas ramas:

```bash
# CONFLICT (file mode): file.sh changed mode in both branches
```

## 12.3 Anatomía de un Archivo en Conflicto

Cuando se produce un conflicto de contenido, Git modifica el archivo afectado insertando marcadores especiales:

```
<<<<<<< HEAD
def calcular_total(items):
    return sum(items) * 1.16  # IVA actual
=======
def calcular_total(items):
    return sum(items) * 1.19  # Nuevo IVA
>>>>>>> feature/nuevo-impuesto
```

La estructura es siempre la misma:

| Marcador        | Significado                                         |
| --------------- | --------------------------------------------------- |
| `<<<<<<< HEAD`  | Inicio de la sección en conflicto (rama actual)     |
| `=======`       | Separador entre las dos versiones                   |
| `>>>>>>> rama`  | Fin de la sección en conflicto (rama que se fusiona)|

Entre `<<<<<<< HEAD` y `=======` está nuestra versión (la rama donde ejecutamos `git merge`). Entre `=======` y `>>>>>>> rama` está la versión de la rama que intentamos fusionar.

> **Advertencia:** Nunca hagas commit de un archivo que contenga estos marcadores. Si accidentalmente lo haces, estarás introduciendo basura textual en el repositorio que romperá la sintaxis del código.

### Estilo de Conflicto: `diff3` y `zdiff3`

Por defecto, Git muestra solo dos versiones (ours y theirs). Configurar `merge.conflictStyle diff3` añade una TERCERA seccion: el ancestro comun (base), mostrando como era el codigo antes de que ambas ramas lo modificaran. Esto acelera enormemente la resolucion de conflictos porque ves que cambio cada lado respecto al original:

```bash
# Activar diff3 GLOBALMENTE (recomendado #1 para conflictos)
git config --global merge.conflictStyle diff3

# Ahora los conflictos muestran tres secciones:
# <<<<<<< HEAD
# def calcular_total(items):
#     return sum(items) * 1.16    (nuestra version)
# ||||||| base
# def calcular_total(items):
#     return sum(items) * 1.10    (ancestro comun - como era antes)
# =======
# def calcular_total(items):
#     return sum(items) * 1.19    (version entrante)
# >>>>>>> feature/nuevo-impuesto

# Git 2.35+ ofrece zdiff3 (mejor rendimiento, mismo resultado visual)
git config --global merge.conflictStyle zdiff3
```

Con `diff3`, puedes ver inmediatamente que "nostros" cambiamos de 1.10 a 1.16, y "ellos" de 1.10 a 1.19 — sin tener que adivinar cual era el valor original.

### Anular el Estilo por Comando

```bash
# Usar diff3 solo para este merge especifico
git checkout --conflict=diff3 feature/login

# O durante un merge/rebase con conflictos, re-crear los marcadores con diff3
git checkout --conflict=diff3 -- src/auth.py
```

## 12.4 Resolución de Conflictos Paso a Paso

Veamos el proceso completo, desde la detección hasta la finalización.

### Paso 1: Detectar el Estado

```bash
git merge feature/login
# Auto-merging src/auth.py
# CONFLICT (content): Merge conflict in src/auth.py
# Automatic merge failed; fix conflicts and then commit the result.
```

### Paso 2: Diagnosticar con `git status`

```bash
git status
# On branch main
# You have unmerged paths.
#   (fix conflicts and run "git commit")
#
# Unmerged paths:
#   (use "git add <file>..." to mark resolution)
#         both modified:   src/auth.py
```

### Listar Archivos No Fusionados con Stage Numbers

`git ls-files -u` muestra los archivos en conflicto con sus stage numbers (1=base, 2=ours, 3=theirs), permitiendo inspeccionar cada version por separado:

```bash
# Listar archivos no fusionados con sus blobs
git ls-files -u

# Salida tipica:
# 100644 a1b2c3d 1   src/auth.py   (base - ancestro comun)
# 100644 e4f5g6h 2   src/auth.py   (ours - nuestra version)
# 100644 i7j8k9l 3   src/auth.py   (theirs - version entrante)

# Ver el contenido de cada version individualmente:
git show :1:src/auth.py    # version base
git show :2:src/auth.py    # version ours (HEAD)
git show :3:src/auth.py    # version theirs (rama entrante)

# Guardar cada version en archivos separados para comparacion
git show :2:src/auth.py > /tmp/auth_ours.py
git show :3:src/auth.py > /tmp/auth_theirs.py
diff /tmp/auth_ours.py /tmp/auth_theirs.py
```

Git clasifica cada archivo en conflicto con etiquetas útiles:

| Etiqueta            | Significado                                          |
| ------------------- | ---------------------------------------------------- |
| `both modified`     | Modificado en ambas ramas                            |
| `both added`        | Creado en ambas ramas con contenido diferente        |
| `both deleted`      | Eliminado en ambas, pero con modificaciones distintas |
| `deleted by us`     | Eliminado en nuestra rama, modificado en la otra     |
| `deleted by them`   | Modificado en nuestra rama, eliminado en la otra     |

### Paso 3: Editar el Archivo

Abre el archivo y busca los marcadores. La resolución puede ser:

- **Elegir una versión:** Borrar la otra sección y los marcadores.
- **Combinar ambas:** Conservar partes de cada versión, quizás en orden.
- **Reescribir:** Ninguna versión es correcta; escribir algo completamente nuevo.

```bash
# Abrir en tu editor favorito
code src/auth.py
```

### Paso 4: Marcar como Resuelto

Una vez editado, debes informar a Git:

```bash
git add src/auth.py
```

Esto realiza dos acciones: añade el archivo al staging area y marca el conflicto como resuelto.

### Paso 5: Finalizar la Fusión

```bash
git commit
```

Al hacer commit sin mensaje (`-m`), Git abre el editor con un mensaje predefinido:

```
Merge branch 'feature/login' into main

# Conflicts:
#       src/auth.py
```

Puedes personalizar este mensaje o aceptarlo tal cual.

### Ejemplo Práctico Completo

```bash
# 1. Estamos en main, intentamos fusionar feature
git checkout main
git merge feature/pagos

# 2. Conflicto en api/pagos.py
git status
# both modified: api/pagos.py

# 3. Vemos las diferencias en conflicto
git diff

# 4. Resolvemos editando el archivo
vim api/pagos.py

# 5. Marcamos resolución
git add api/pagos.py

# 6. Confirmamos todo resuelto
git status
# All conflicts fixed but you are still merging.
#   (use "git commit" to conclude merge)

git commit -m "Merge feature/pagos: resolver conflicto en api/pagos.py"
```

## 12.5 Herramientas Visuales de Merge

Editar marcadores manualmente funciona, pero las herramientas visuales aceleran enormemente el proceso.

### 12.5.1 `git mergetool`

Git incluye un comando para lanzar herramientas externas de resolución:

```bash
git mergetool
```

Esto abre cada archivo en conflicto con la herramienta configurada. Para elegir la herramienta:

```bash
# Listar herramientas disponibles en el sistema
git mergetool --tool-help

# Configurar una herramienta específica
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
git config --global mergetool.keepBackup false
```

### 12.5.2 Visual Studio Code

VS Code tiene un editor de conflictos integrado excepcional. Al abrir un archivo con marcadores de conflicto, muestra botones para:

- **Accept Current Change:** Conservar nuestra versión (HEAD).
- **Accept Incoming Change:** Conservar la versión de la rama entrante.
- **Accept Both Changes:** Conservar ambas, una después de otra.
- **Compare Changes:** Vista lado a lado de las diferencias.

```bash
# Configurar VS Code como mergetool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait --merge $REMOTE $LOCAL $BASE $MERGED'
```

### 12.5.3 Meld

Meld es una herramienta gráfica de comparación y fusión, popular en Linux:

```bash
# Instalación
sudo apt install meld   # Debian/Ubuntu
brew install meld       # macOS

# Configuración como mergetool
git config --global merge.tool meld
git config --global mergetool.meld.path /usr/bin/meld
```

Meld muestra tres paneles: archivo base (ancestro común), tu versión y la versión remota.

### 12.5.4 KDiff3

Otra opción clásica con vista de cuatro paneles:

```bash
sudo apt install kdiff3
git config --global merge.tool kdiff3
```

### 12.5.5 P4Merge

Herramienta gratuita de Perforce, muy pulida visualmente:

```bash
# Descargar desde perforce.com
# Configurar:
git config --global mergetool.p4merge.cmd '/Applications/p4merge.app/Contents/MacOS/p4merge $BASE $LOCAL $REMOTE $MERGED'
git config --global merge.tool p4merge
```

### 12.5.6 IntelliJ IDEA / WebStorm

Los IDE de JetBrains incluyen un resolvedor de conflictos de tres paneles con resaltado de sintaxis y botones para aceptar/rechazar cambios. Se integran automáticamente con Git.

### Tabla Comparativa de Herramientas

| Herramienta  | Plataforma | Vista      | Gratuito | Curva de Aprendizaje |
| ------------ | ---------- | ---------- | -------- | -------------------- |
| VS Code      | Multi      | En línea   | Sí       | Baja                 |
| Meld         | Linux/macOS| 3 paneles  | Sí       | Baja                 |
| KDiff3       | Multi      | 4 paneles  | Sí       | Media                |
| P4Merge      | Multi      | 3 paneles  | Sí       | Baja                 |
| IntelliJ     | Multi      | 3 paneles  | Licencia | Baja                 |
| Beyond Compare| Multi     | 3 paneles  | Pago     | Baja                 |

## 12.6 Abortar una Fusión

Si el conflicto es demasiado complejo o te das cuenta de que iniciaste la fusión equivocada:

```bash
git merge --abort
```

Este comando restaura el estado exacto anterior al `git merge`. No deja rastro alguno.

```bash
# Para abortar un rebase en conflicto:
git rebase --abort

# Para abortar un cherry-pick:
git cherry-pick --abort
```

> **Tip:** Si hiciste cambios locales durante la resolución del conflicto y quieres conservarlos antes de abortar, haz `git stash` primero.

## 12.7 Estrategias de Fusión (`git merge -s`)

Git ofrece múltiples estrategias de fusión para diferentes situaciones:

### 12.7.1 `recursive` (Predeterminada)

La estrategia estándar para fusionar dos ramas. Detecta renombres y resuelve fusiones "criss-cross".

```bash
git merge -s recursive feature/login
```

Características:
- Detecta renombres.
- Resuelve fusiones "criss-cross" (cuando dos ramas se han fusionado entre sí previamente).
- Es la más inteligente y la recomendada para el 99% de los casos.
- **No fusiona 3+ ramas.** Para fusionar multiples ramas simultaneamente, usa `octopus`.

### 12.7.2 `ours`

Descarta completamente los cambios de la rama entrante y conserva solo nuestra versión:

```bash
git merge -s ours feature/deprecated
```

> **Importante:** No confundir `-s ours` (estrategia que descarta TODO el contenido de la otra rama) con `-X ours` (opción que, en caso de conflicto, elige nuestra versión archivo por archivo).

`-s ours` es útil para marcar una rama como "fusionada" sin incorporar sus cambios, por ejemplo al deprecar una rama de feature que ya no será integrada.

### 12.7.3 `theirs` (Vía Opción `-X`)

Aunque no existe `-s theirs`, puedes lograr el efecto con:

```bash
git merge -X theirs feature/login
```

Esto resuelve todos los conflictos automáticamente a favor de la rama entrante.

### 12.7.4 `octopus`

Fusiona tres o más ramas simultáneamente. Es la UNICA estrategia que puede fusionar 3+ ramas:

```bash
git merge -s octopus feature/a feature/b feature/c
```

Ideal para consolidar múltiples ramas de features pequeñas. Si ocurre algún conflicto durante una fusión octopus, la operación falla completamente; no permite resolución interactiva.

### 12.7.5 `subtree`

Útil cuando un repositorio contiene otro como subdirectorio (merge de subtrees). Ajusta automáticamente las rutas:

```bash
git merge -s subtree -X subtree=lib/vendor feature/vendor-update
```

## 12.8 Merge vs Rebase: Estrategias de Conflicto Diferentes

Merge y rebase manejan los conflictos de forma radicalmente distinta:

| Aspecto | Merge | Rebase |
|---------|-------|--------|
| **Cantidad de conflictos** | Todos los conflictos a la vez | N conflictos (uno por cada commit) |
| **Contexto disponible** | Solo el diff final entre HEAD y la rama | El diff de CADA commit individual |
| **Resolucion** | Una sola ronda de resolucion | N rondas (`git rebase --continue` por cada commit conflictivo) |
| **Ventaja** | Ves el panorama completo | Aislas cada conflicto en el commit que lo causo |
| **Desventaja** | El merge commit oculta como resolviste | Si abortas a mitad, pierdes resoluciones (usa rerere) |

```bash
# Merge: un solo conflicto con todo el diff acumulado
git merge feature/login
# CONFLICT en 3 archivos. Resuelves todo de una vez.
git add .
git commit

# Rebase: un conflicto por cada commit de feature/login
git rebase main
# CONFLICT en commit 1/5
# ... resuelves ...
git add . && git rebase --continue
# CONFLICT en commit 2/5 (mismos archivos, distinto contexto)
# ... resuelves ...
git add . && git rebase --continue
# ... etc.
```

**Recomendacion:** Usa merge cuando la rama tiene muchos commits que tocan los mismos archivos (resuelves una vez). Usa rebase cuando quieres mantener el historial lineal y rerere esta activo para reutilizar resoluciones entre commits.

## 12.9 Custom Merge Drivers via `.gitattributes`

Puedes definir estrategias de merge por tipo de archivo usando `.gitattributes` y un merge driver personalizado:

```bash
# .gitattributes
# Usar 'ours' para archivos generados automaticamente
*.lock merge=ours

# Usar un merge driver personalizado para archivos de configuracion
*.xml merge=xmlmerge
*.json merge=jsonmerge

# Usar union merge para CHANGELOG (ambas lineas se conservan)
CHANGELOG.md merge=union

# PO files: usar merge tool especializado para traducciones
*.po merge=pomerge
```

```bash
# Definir el driver personalizado en .git/config (o ~/.gitconfig)
git config merge.xmlmerge.name "XML merge driver"
git config merge.xmlmerge.driver "python3 scripts/xml_merge.py %O %A %B %L %P"

git config merge.jsonmerge.name "JSON merge driver"
git config merge.jsonmerge.driver "python3 scripts/json_merge.py %O %A %B"
# Donde: %O=ancestro, %A=ours, %B=theirs, %L=tamaño conflicto, %P=ruta
```

**Drivers built-in utiles:**
- `merge=ours`: siempre elige nuestra version (ideal para archivos generados)
- `merge=union`: concatena ambas versiones (ideal para CHANGELOG, requirements.txt)
- `merge=binary`: aborta si hay conflicto (no intenta merge de binarios)

## 12.10 `git merge-file`: Plumbing para Resolucion de Conflictos

`git merge-file` es el comando de plumbing que Git usa internamente para fusionar archivos. Puedes usarlo directamente para resolver conflictos desde scripts o herramientas:

```bash
# Extraer las tres versiones de un archivo en conflicto
git show :1:archivo.txt > /tmp/base.txt
git show :2:archivo.txt > /tmp/ours.txt
git show :3:archivo.txt > /tmp/theirs.txt

# Fusionar usando merge-file (plumbing)
git merge-file -p /tmp/ours.txt /tmp/base.txt /tmp/theirs.txt > archivo_resuelto.txt

# Opciones utiles:
git merge-file --ours /tmp/ours.txt /tmp/base.txt /tmp/theirs.txt     # elegir ours
git merge-file --theirs /tmp/ours.txt /tmp/base.txt /tmp/theirs.txt   # elegir theirs
git merge-file --union /tmp/ours.txt /tmp/base.txt /tmp/theirs.txt    # concatenar ambos
```

El comando modifica el primer archivo in-place con la resolucion. Si hay conflictos no resueltos, inserta los marcadores estandar.

## 12.11 Evil Merges: Cuando la Resolucion Introduce Codigo Nuevo

Un "evil merge" ocurre cuando la resolucion manual de un conflicto **introduce codigo que no existia en ninguna de las dos ramas**. El commit de merge contiene cambios adicionales mas alla de la fusion textual.

```bash
# Ejemplo tipico: ambas ramas cambiaron la misma funcion
# Resuelves el conflicto, pero ademas refactorizas la funcion
# para usar un nuevo patron que no estaba en ninguna rama.

git merge feature/login
# Editas auth.py resolviendo el conflicto Y refactorizando
git add auth.py
git commit -m "Merge feature/login"
```

```bash
# Detectar evil merges: commits de merge que introducen cambios propios
git log --merges --diff-filter=M -p

# Ver solo los cambios del merge que no vienen de ninguna rama
git show <merge-commit>
# git diff <merge-commit>^1 <merge-commit>   # cambios vs primer padre
# git diff <merge-commit>^2 <merge-commit>   # cambios vs segundo padre
# Lo que NO esta en ninguno de esos dos diffs = evil merge
```

**Problemas de los evil merges:**
- `git bisect` puede apuntar al merge en lugar del commit real que introdujo el bug
- `git revert` de un evil merge es complejo (¿a cual padre volver?)
- Dificulta la revision de codigo porque el diff del merge mezcla fusion + cambios nuevos

**Alternativa recomendada:** Resuelve el conflicto minimamente para que compile. Haz un commit separado en la rama mergeada con la refactorizacion, luego mergea limpiamente SIN cambios adicionales.

## 12.12 Opciones de Estrategia (`-X`)

### `-X ours`

Ante cada conflicto, elige automáticamente nuestra versión:

```bash
git merge -X ours feature/login
```

### `-X theirs`

Ante cada conflicto, elige la versión entrante:

```bash
git merge -X theirs feature/login
```

### `-X patience`

Usa el algoritmo "patience diff" para detectar cambios. Es más lento pero produce resultados más precisos cuando hay muchas líneas que no coinciden exactamente:

```bash
git merge -X patience feature/refactor
```

### `-X ignore-space-change`

Ignora cambios de espacios en blanco (tabulaciones, espacios al final de línea). Los conflictos de cambios reales se resuelven mejor:

```bash
git merge -X ignore-space-change feature/format
```

### `-X ignore-all-space`

Ignora completamente todos los espacios en blanco, incluyendo los cambios dentro de las líneas:

```bash
git merge -X ignore-all-space feature/reformat
```

### Tabla de Opciones de Estrategia

| Opción                  | Efecto en Conflicto                                      |
| ----------------------- | -------------------------------------------------------- |
| `-X ours`               | Elige automáticamente la versión HEAD                    |
| `-X theirs`             | Elige automáticamente la versión de la rama entrante     |
| `-X patience`           | Algoritmo más preciso para coincidencias difusas         |
| `-X ignore-space-change`| Ignora cambios de whitespace adyacentes                  |
| `-X ignore-all-space`   | Ignora todos los cambios de whitespace                   |
| `-X diff-algorithm=`    | Algoritmo: patience, minimal, histogram, myers           |
| `-X renormalize`        | Aplica reglas de EOL y filtros antes de comparar         |

## 12.13 `git rerere`: Reutilización de Resoluciones

`rerere` significa **Re**use **R**ecorded **R**esolution. Es una funcionalidad infravalorada que graba cómo resolviste un conflicto y la reaplica automáticamente si el mismo conflicto vuelve a aparecer.

### 12.13.1 Activación

```bash
git config --global rerere.enabled true
```

O en un repositorio específico:

```bash
git config rerere.enabled true
```

### 12.13.2 Donde se Almacenan las Resoluciones

Las resoluciones de rerere se guardan en `.git/rr-cache/`:

```bash
# Ver resoluciones almacenadas
ls -R .git/rr-cache/
# .git/rr-cache/ce013625030ba8dba906f756967f9e9ca394464a/
#   preimage    (conflicto original, con marcadores)
#   postimage   (resolucion final, sin marcadores)

# Las resoluciones se identifican por un hash del conflicto
# (combinacion de los SHAs de ours + theirs + base)
```

### 12.13.3 Compartir Resoluciones con el Equipo

Las resoluciones de rerere son locales, pero puedes compartirlas:

```bash
# Exportar resoluciones del repo actual
mkdir -p .git/rr-cache
tar czf rerere-cache.tar.gz .git/rr-cache/

# Otro miembro del equipo las importa:
tar xzf rerere-cache.tar.gz -C /path/to/repo/
# O mas simple: copiar directorio
cp -r .git/rr-cache /path/to/other/repo/.git/
```

Para equipos, considera versionar `.git/rr-cache/` como un asset compartido (no dentro del repo Git, pero disponible via script de setup) para que todos se beneficien de las resoluciones del equipo.

### 12.13.4 El Ciclo de Rerere

```
                 +-------------------+
                 | Ocurre un conflicto|
                 +--------+----------+
                          |
                          v
                 +--------+----------+
                 | Resuelves manual-  |
                 | mente el conflicto |
                 +--------+----------+
                          |
                          v
                 +--------+----------+
                 | git add + commit    |
                 | rerere graba la     |
                 | resolución         |
                 +--------+----------+
                          |
                          v
           +-------------+-------------+
           | Mismo conflicto reaparece  |
           | (ej. durante rebase)      |
           +-------------+-------------+
                          |
                          v
                 +--------+----------+
                 | rerere reaplica la  |
                 | resolución automá-  |
                 | ticamente          |
                 +-------------------+
```

### 12.13.5 Caso de Uso Típico: Rebase con Rerere

```bash
# Activamos rerere
git config rerere.enabled true

# Primera vez: resolvemos el conflicto manualmente
git checkout feature
git rebase main
# CONFLICT en src/config.js
# ... resolvemos manualmente ...
git add src/config.js
git rebase --continue
# Rerere graba la resolución automáticamente

# Si abortamos y reintentamos, rerere la reaplica:
git rebase --abort
git rebase main
# Resolved 'src/config.js' using previous resolution.  <-- Rerere en acción
```

### 12.13.6 Comandos de Rerere

```bash
# Ver las resoluciones grabadas
git rerere status

# Ver la diferencia antes/después de una resolución grabada
git rerere diff

# Olvidar una resolución específica
git rerere forget <ruta-del-archivo>

# Limpiar todas las resoluciones grabadas
git rerere clear

# Ver estadísticas de rerere
git rerere remaining
```

> **Tip:** Activa `rerere.enabled` globalmente. El costo es insignificante y el beneficio durante rebases largos es enorme. Te ahorrará resolver el mismo conflicto docenas de veces.

## 12.14 `git merge-base`: Encontrar el Ancestro Común

Entender dónde divergieron dos ramas es crucial para diagnosticar conflictos:

```bash
# Mostrar el commit del ancestro común
git merge-base main feature

# Mostrar todos los ancestros comunes (en caso de múltiples)
git merge-base --all main feature

# Ver si un commit es ancestro de otro
git merge-base --is-ancestor <commit-a> <commit-b>
echo $?  # 0 si es ancestro, 1 si no lo es
```

### Aplicación Práctica

```bash
# Ver qué cambió en cada rama desde el punto de divergencia
BASE=$(git merge-base main feature)
git diff $BASE..main
git diff $BASE..feature
```

## 12.15 Estrategias de Equipo para Prevenir Conflictos

Los conflictos no son inevitables. Un equipo disciplinado puede reducirlos drásticamente:

### 12.15.1 Sincronización Frecuente

```bash
# Actualizar la rama con main diariamente
git fetch origin
git merge origin/main
```

### 12.15.2 Commits Pequeños y Enfocados

Un commit que modifica 20 archivos tiene altísima probabilidad de conflicto.
Un commit que modifica 1 archivo tiene bajísima probabilidad.

### 12.15.3 Comunicación del Equipo

- **"Estoy tocando X archivo"**: Avisa en el canal del equipo.
- **Revisión de PR temprana**: Abre el PR como draft apenas tengas código.
- **Pair programming**: Reduce conflictos porque dos personas no editan lo mismo por separado.

### 12.15.4 Modularización del Código

Un código bien modularizado reduce la probabilidad de que dos features toquen los mismos archivos:

```
# Mala estructura (alta probabilidad de conflicto)
src/
  utils.py          # 5000 líneas, todos meten mano aquí

# Buena estructura (baja probabilidad de conflicto)
src/
  utils/
    auth.py         # Responsabilidad del equipo A
    payments.py     # Responsabilidad del equipo B
    formatting.py   # Responsabilidad del equipo C
```

### 12.15.5 CODEOWNERS y Revisión de Código

Configurar dueños de código asegura que los conflictos sean detectados y resueltos por quien mejor conoce cada área (ver Capítulo 15).

## 12.16 Pull con Conflictos: `fetch` + `merge` vs `pull` Directo

Muchos conflictos aparecen al hacer `git pull`. La práctica recomendada es separar `fetch` de `merge`:

### Enfoque Recomendado (fetch + merge)

```bash
# Paso 1: Traer cambios remotos sin fusionar
git fetch origin

# Paso 2: Ver qué ha cambiado
git log HEAD..origin/main --oneline

# Paso 3: Ver las diferencias
git diff HEAD origin/main

# Paso 4: Fusionar (o rebasar) con visibilidad completa
git merge origin/main
```

### Enfoque No Recomendado (pull directo)

```bash
# Fusiona inmediatamente sin oportunidad de revisar
git pull origin main
```

### Manejo de Conflictos Durante Pull

```bash
# Si aparece un conflicto durante pull:
git fetch origin
git merge origin/main
# ... resolver conflictos ...
git add <archivos>
git commit

# Alternativa: rebase en lugar de merge
git fetch origin
git rebase origin/main
# Si hay conflicto:
# ... resolver ...
git add <archivo>
git rebase --continue
```

### Pull con Rebase (Configuración Global)

```bash
# Hacer que pull use rebase por defecto
git config --global pull.rebase true

# O solo en este repositorio
git config pull.rebase true
```

---

## Resumen del Capítulo 12

En este capítulo hemos cubierto en profundidad los conflictos de fusión en Git: su naturaleza, tipos, anatomía, herramientas de resolución y estrategias de prevención. Los puntos clave son:

1. **Los conflictos son normales y esperables.** No indican error, indican que Git protege tu código de fusiones automáticas potencialmente destructivas.
2. **Existen múltiples tipos de conflictos:** contenido (mismas líneas), modificación/eliminación, renombre/renombre y permisos. Cada uno requiere un abordaje diferente.
3. **La resolución sigue el ciclo:** detectar (`git status`), editar, marcar (`git add`), finalizar (`git commit`).
4. **Las herramientas visuales** (VS Code, Meld, KDiff3, IntelliJ) aceleran la resolución y reducen errores humanos.
5. **`git rerere`** es una joya oculta: graba tus resoluciones y las reaplica automáticamente, ideal para rebases prolongados.
6. **Prevenir es mejor que curar:** sincronización frecuente, comunicación del equipo, código modularizado y CODEOWNERS reducen los conflictos drásticamente.

## Ejercicios Propuestos

### Ejercicio 1: Simulación Completa de Conflicto

1. Crea un repositorio nuevo con un archivo `calculadora.py` que contenga una función `sumar(a, b)`.
2. Crea dos ramas desde `main`: `feature/iva` y `feature/descuento`.
3. En `feature/iva`, modifica `calculadora.py` para aplicar un IVA del 21% al resultado.
4. En `feature/descuento`, modifica `calculadora.py` para aplicar un descuento del 15%.
5. Fusiona ambas ramas a `main` y resuelve los conflictos manualmente sin herramientas visuales.
6. Verifica que el archivo final contenga ambas funcionalidades correctamente.

### Ejercicio 2: Uso de Herramientas Visuales

1. Configura `git mergetool` para usar VS Code.
2. Reproduce un conflicto entre dos ramas que modifiquen 5 líneas diferentes del mismo archivo.
3. Resuelve el conflicto usando solo los botones de VS Code.
4. Documenta los pasos y compara la experiencia con la resolución manual del Ejercicio 1.

### Ejercicio 3: Dominando `git rerere`

1. Activa `git rerere` en tu configuración global.
2. Crea una rama `feature/database` con 3 commits que modifiquen `db.py`.
3. En `main`, modifica las mismas secciones de `db.py` en 1 commit.
4. Realiza un rebase de `feature/database` sobre `main`, resolviendo los conflictos.
5. Aborta el rebase (`git rebase --abort`) y vuélvelo a iniciar. Observa cómo `rerere` reaplica las resoluciones.
6. Usa `git rerere diff` para inspeccionar las resoluciones grabadas.

### Ejercicio 4: Estrategias de Merge

1. Crea tres ramas de feature que modifiquen archivos distintos (sin conflictos entre sí).
2. Fusiona las tres simultáneamente usando `git merge -s octopus`.
3. Crea una rama que modifique 10 archivos y usa `git merge -s ours` para marcarla como fusionada sin incorporar cambios.
4. Explica en qué escenario real usarías `-s ours` y en cuál usarías `-X ours`.

### Ejercicio 5: Simulación de Conflicto Modificación/Eliminación

1. Crea un repositorio con un archivo `legacy.js`.
2. En la rama `feature`, modifica `legacy.js` añadiendo una nueva función.
3. En la rama `main`, elimina `legacy.js` con `git rm`.
4. Fusiona `feature` en `main` y resuelve el conflicto de tipo "modify/delete".
5. Documenta qué comandos usaste para decidir entre conservar el archivo o aceptar la eliminación.

---

← [Capítulo anterior](11-internals.md) | [Inicio](README.md) | [Capítulo siguiente →](13-reescritura-historia.md)
