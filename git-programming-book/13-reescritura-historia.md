# Capítulo 13: Reescritura de Historia

Reescribir la historia de Git es como viajar al pasado para corregir errores antes de que ocurran. Es una de las capacidades más poderosas de Git y, al mismo tiempo, una de las más peligrosas si no se maneja con cuidado. En este capítulo dominaremos las herramientas y protocolos para reescribir el historial de forma segura y profesional.

## 13.1 El Principio Fundamental: Poderoso pero Peligroso

Reescribir la historia significa modificar commits que ya existen. Git no borra los commits antiguos, sino que crea nuevos commits con la información corregida y abandona las referencias a los originales.

```
Antes del amend:
  A --- B --- C (HEAD)

Después del amend:
  A --- B --- C' (HEAD)
         \
          C  (commit huérfano, eventualmente eliminado por el GC)
```

> **Advertencia crítica:** Reescribir historia que ya ha sido compartida (pusheada a un repositorio remoto) causa estragos en el equipo. Cada colaborador deberá forzar la sincronización o re-clonar. La regla de oro es: solo reescribe commits que aún no han sido pusheados, o hazlo con plena coordinación del equipo.

## 13.2 Cuándo Reescribir la Historia

La reescritura no es un capricho; existen escenarios legítimos:

| Escenario                                | Herramienta Principal          |
| ---------------------------------------- | ------------------------------ |
| Limpiar commits antes del push           | `git rebase -i`                |
| Corregir el mensaje del último commit    | `git commit --amend`           |
| Eliminar datos sensibles del historial   | `git filter-repo`              |
| Cambiar autor de múltiples commits       | `git filter-repo --mailmap`    |
| Extraer un subdirectorio como nuevo repo | `git filter-repo --subdirectory-filter` |
| Reestructurar commits desordenados       | `git rebase -i`                |
| Eliminar archivos grandes del historial  | `git filter-repo` o BFG        |

### Decálogo de Seguridad para Reescritura

1. **Haz backup:** `git clone --mirror <repo> backup.git` antes de cualquier operación destructiva.
2. **Trabaja en un clon:** No reescribas en tu copia de trabajo principal.
3. **Coordina con el equipo:** Si la historia ya está compartida, todos deben saberlo.
4. **Verifica el resultado:** Usa `git log`, `git diff` y `git fsck` después de reescribir.
5. **Usa `--force-with-lease`:** Nunca `--force` a secas para empujar.

## 13.3 `git commit --amend`: La Puerta de Entrada

Ya cubierto en capítulos anteriores, pero merece mención como la forma más simple de reescritura:

```bash
# Modificar el mensaje del último commit
git commit --amend -m "Nuevo mensaje corregido"

# Añadir archivos olvidados al último commit
git add archivo_olvidado.py
git commit --amend --no-edit
```

Ambos comandos reemplazan el commit HEAD con uno nuevo que incorpora los cambios.

## 13.4 `git rebase -i`: El Taller de Escultura

El rebase interactivo, también cubierto previamente, es la herramienta de reescritura por excelencia para commits locales:

```bash
# Reescribir los últimos 5 commits
git rebase -i HEAD~5
```

Los comandos disponibles en el editor:
- `pick`: conservar el commit tal cual
- `reword`: cambiar el mensaje del commit
- `edit`: pausar para modificar el contenido del commit
- `squash`: fusionar con el commit anterior, conservando ambos mensajes
- `fixup`: fusionar con el commit anterior, descartando el mensaje
- `drop`: eliminar el commit completamente

### 13.4.1 `git rebase --onto`: Mover Porciones de Rama

`git rebase --onto` permite trasplantar una serie de commits a una nueva base, ideal para reubicar ramas o extraer solo una parte del trabajo:

```bash
# Sintaxis: git rebase --onto <nueva-base> <base-antigua> <rama>
# Trasplanta los commits desde <base-antigua>..<rama> sobre <nueva-base>

# Ejemplo: mover una rama de feature a otro punto de partida
git rebase --onto main feature/antigua-base feature/mi-tarea

# Escenario real: feature/login nació de develop, pero ahora quieres
# moverla a main porque develop tiene commits que no necesitas
git rebase --onto main develop feature/login
# Trasplanta solo los commits exclusivos de feature/login sobre main

# Recuperar colaboradores tras reescritura de historia:
git rebase --onto origin/main <commit-antes-del-rewrite> <tu-rama>
```

**Caso de uso común:** En un equipo donde main ha recibido un force-push tras limpieza de historial, `git rebase --onto` permite a cada desarrollador reubicar sus ramas locales sobre la nueva historia sin perder su trabajo.

## 13.5 `git replace`: Alternativa No Destructiva

Antes de reescribir historia permanentemente, considera `git replace`. Esta herramienta crea referencias de reemplazo que hacen que Git "vea" un objeto diferente sin modificar los SHAs reales. Es ideal para experimentar o corregir referencias sin alterar el historial del equipo:

```bash
# Crear un commit corregido (con --amend o manualmente)
git commit --amend -m "Mensaje corregido"

# Reemplazar el commit original con el corregido
git replace <SHA-original> <SHA-corregido>

# Ahora git log muestra el commit corregido, pero el original sigue intacto
git log --oneline

# Listar todos los reemplazos activos
git replace -l

# Ver el objeto reemplazado (el real)
git replace --list

# Eliminar el reemplazo (volver al original)
git replace -d <SHA-original>

# Para hacer el reemplazo permanente (reescribir historia de verdad):
git filter-repo --force  # O usar git filter-branch
```

**Ventaja clave:** Los reemplazos son locales. Puedes compartirlos con `git push origin 'refs/replace/*'` y `git fetch origin 'refs/replace/*'`, pero no reescriben la historia. Si el equipo decide adoptar los cambios, se aplica `filter-repo` definitivamente.

## 13.6 Tabla de Migración: filter-branch → filter-repo

| Operación con `filter-branch` (obsoleto) | Equivalente moderno con `filter-repo` |
|------------------------------------------|---------------------------------------|
| `git filter-branch --index-filter 'git rm --cached ...' -- --all` | `git filter-repo --path <archivo> --invert-paths` |
| `git filter-branch --env-filter '...' -- --all` | `git filter-repo --mailmap mailmap.txt` |
| `git filter-branch --subdirectory-filter <dir> -- --all` | `git filter-repo --subdirectory-filter <dir>` |
| `git filter-branch --msg-filter 'sed ...' -- --all` | `git filter-repo --message-callback '...'` |
| `git filter-branch --tree-filter '...' -- --all` | `git filter-repo --filename-callback '...'` |
| `git filter-branch --prune-empty -- --all` | `git filter-repo --prune-empty always` (por defecto) |
| `git filter-branch --tag-name-filter cat -- --all` | `git filter-repo --refs` (automático, sin flag extra) |

## 13.7 `git filter-branch`: La Herramienta Clásica

`filter-branch` fue durante años la herramienta estándar para reescribir historiales completos. Aunque ha sido oficialmente desaconsejada en favor de `git filter-repo`, conocerla es importante porque infinidad de tutoriales y scripts aún la referencian.

### 13.5.1 Eliminar un Archivo del Historial

```bash
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch secrets/credentials.txt' \
  --prune-empty --tag-name-filter cat -- --all
```

- `--index-filter`: opera sobre el índice (más rápido que `--tree-filter`).
- `--prune-empty`: elimina commits que quedan vacíos tras la operación.
- `--tag-name-filter cat`: reescribe los tags para que apunten a los nuevos commits.
- `-- --all`: aplica a todas las ramas.

### 13.5.2 Cambiar Autor en Todos los Commits

```bash
git filter-branch --force --env-filter '
  if [ "$GIT_COMMITTER_EMAIL" = "viejo@email.com" ]; then
    export GIT_COMMITTER_NAME="Nuevo Nombre"
    export GIT_COMMITTER_EMAIL="nuevo@email.com"
  fi
  if [ "$GIT_AUTHOR_EMAIL" = "viejo@email.com" ]; then
    export GIT_AUTHOR_NAME="Nuevo Nombre"
    export GIT_AUTHOR_EMAIL="nuevo@email.com"
  fi
' --tag-name-filter cat -- --all
```

### 13.5.3 Extraer Subdirectorio como Nuevo Repositorio

```bash
git filter-branch --subdirectory-filter lib/mi-modulo -- --all
```

> **Nota:** `git filter-branch` es notoriamente lento en repositorios grandes. Para historiales de más de 1000 commits, prefiere `git filter-repo`.

## 13.8 `git filter-repo`: La Herramienta Moderna

`git filter-repo` es el reemplazo oficial recomendado por el propio equipo de Git. Es más rápido (escrito en Python, opera con streams), más seguro y más expresivo.

### 13.8.1 Instalación

```bash
# Vía pip (recomendado)
pip install git-filter-repo

# Vía gestor de paquetes del sistema
brew install git-filter-repo     # macOS
sudo apt install git-filter-repo # Ubuntu/Debian ≥ 20.04
```

### 13.8.2 Análisis Previo del Repositorio

Antes de reescribir, conviene analizar qué contiene el historial:

```bash
# Genera un reporte del repositorio
git filter-repo --analyze

# Esto crea el directorio .git/filter-repo/
# con archivos como:
#   README                - Resumen del análisis
#   directories-deleted-sorted.txt
#   extensions-deleted-sorted.txt
#   paths-deleted-sorted.txt
#   path-changes.txt
#   commits-by-size.txt
```

### 13.8.3 Eliminar Archivos del Historial

```bash
# Eliminar un archivo específico
git filter-repo --path secrets/production.env --invert-paths --force

# Eliminar múltiples archivos desde un listado
echo "archivo_grande_1.iso" > files-to-delete.txt
echo "archivo_grande_2.psd" >> files-to-delete.txt
git filter-repo --invert-paths --paths-from-file files-to-delete.txt

# Eliminar archivos por patrón glob
git filter-repo --path-glob '*.mp4' --invert-paths
git filter-repo --path-glob '*.zip' --invert-paths

# Eliminar un directorio completo
git filter-repo --path build/ --invert-paths
```

### 13.8.4 Reemplazar Texto en el Historial

Útil para limpiar contraseñas, tokens o URLs que quedaron registradas:

```bash
# Crear archivo de reemplazos (formato: texto_a_buscar==>texto_a_reemplazar)
cat > replacements.txt << 'EOF'
password=supersecreto==>password=REDACTED
api-key-12345==>api-key-REMOVED
EOF

git filter-repo --replace-text replacements.txt
```

### 13.8.5 Cambiar Autor y Email

```bash
# Crear archivo de mailmap con el mapeo de correos
cat > mailmap.txt << 'EOF'
Nuevo Nombre <nuevo@empresa.com> <viejo@personal.com>
Maria Garcia <maria@empresa.com> <maria123@gmail.com>
EOF

git filter-repo --mailmap mailmap.txt
```

También puedes usar el archivo `.mailmap` estándar de Git:

```bash
cat > .mailmap << 'EOF'
Nuevo Nombre <nuevo@empresa.com> <viejo@personal.com>
EOF
git filter-repo --use-mailmap
```

### 13.8.6 Extraer Subdirectorio como Repositorio Independiente

```bash
# Clonar el repo primero (siempre trabaja en clon)
git clone mi-repo.git repo-temp
cd repo-temp

# Extraer solo el subdirectorio
git filter-repo --subdirectory-filter src/modulo-interesante

# El historial ahora contiene solo commits que afectaron ese subdirectorio,
# con las rutas reescritas como si el subdirectorio fuera la raíz
```

### 13.8.7 Renombrar Archivos o Directorios

```bash
git filter-repo --path-rename old_name.py:new_name.py
git filter-repo --path-rename src/:lib/
```

### 13.8.8 Filtros por Mensaje de Commit

```bash
# Eliminar commits cuyo mensaje contenga cierta palabra
git filter-repo --invert-grep "WIP" --force

# Conservar solo commits con cierto patrón en el mensaje
git filter-repo --message-callback 'return message if b"fix" in message else None'
```

### 13.8.9 Limpiar Después de la Reescritura

`git filter-repo` ya limpia automáticamente el reflog y las referencias obsoletas. Los siguientes comandos son **opcionales** y solo necesarios si usaste otras herramientas o quieres una limpieza adicional:

```bash
# Solo necesario tras filter-branch o BFG (filter-repo ya lo hace)
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

## 13.9 Eliminar Datos Sensibles del Historial

Este es quizás el escenario más crítico de reescritura. Has commiteado accidentalmente una API key, un token o una contraseña. El procedimiento completo es:

### Procedimiento de Emergencia para Datos Sensibles

```bash
# PASO 1: Clonar el repositorio (nunca trabajes en el original)
git clone --mirror https://github.com/empresa/repo-expuesto.git repo-limpieza
cd repo-limpieza

# PASO 2: Identificar qué contiene información sensible
git filter-repo --analyze

# PASO 3: Reemplazar o eliminar
cat > secrets.txt << 'EOF'
ghp_xxxxxxxxxxxxxxxxxxxx==>***REDACTED***
sk_live_yyyyyyyyyyyyyyyy==>***REDACTED***
EOF
git filter-repo --replace-text secrets.txt

# PASO 4: Verificar que el historial está limpio
git log -p | grep -i "ghp_\|sk_live"  # No debe haber coincidencias

# PASO 5: Empujar al remoto
git remote add origin https://github.com/empresa/repo-expuesto.git
git push --force-with-lease --all
git push --force-with-lease --tags

# PASO 6: Rotar las credenciales expuestas INMEDIATAMENTE
# (La reescritura de Git no revoca las claves ya comprometidas)
```

> **Acción urgente:** Eliminar una API key del historial de Git es solo la mitad del trabajo. Debes rotar (revocar y regenerar) las credenciales expuestas inmediatamente. Asume que la clave fue comprometida desde el momento del push.

## 13.10 Cambio Masivo de Autor en Commits Históricos

Escenario común al migrar de una cuenta personal a una corporativa, o al corregir la configuración de `user.name`/`user.email`:

### Con `git filter-repo` (Recomendado)

```bash
cat > mailmap.txt << 'EOF'
Nombre Corporativo <dev@empresa.com> <nombre@gmail.com>
Nombre Corporativo <dev@empresa.com> <usuario-viejo@otromail.com>
EOF

git filter-repo --mailmap mailmap.txt --force
```

### Con Script de Bash (Alternativa Manual)

```bash
#!/bin/bash
# cambiar-autor.sh

git filter-branch --force --env-filter '
OLD_EMAIL="incorrecto@mail.com"
CORRECT_NAME="Nombre Correcto"
CORRECT_EMAIL="correcto@empresa.com"

if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]; then
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]; then
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
```

## 13.11 Migrar Subdirectorio a Repositorio Independiente

Una práctica común al extraer una biblioteca o microservicio de un monolito:

```bash
# PASO 1: Clon limpio
git clone https://github.com/empresa/monorepo.git lib-extraida
cd lib-extraida

# PASO 2: Filtrar solo el subdirectorio deseado
git filter-repo --subdirectory-filter packages/mi-libreria

# PASO 3: El historial ahora contiene solo los commits que tocaron
# packages/mi-libreria, con rutas ajustadas a la nueva raíz

# PASO 4: Crear nuevo repositorio remoto y empujar
git remote add origin https://github.com/empresa/mi-libreria.git
git push -u origin main --force

# PASO 5 (opcional): Agregar README, LICENSE, CI/CD como nuevos commits
```

## 13.12 BFG Repo-Cleaner: Alternativa Basada en Java

BFG es una alternativa a `filter-branch` más rápida y simple para casos comunes:

```bash
# Requiere Java Runtime
java -version  # Verificar que Java está instalado

# Descargar BFG (archivo .jar)
wget https://repo1.maven.org/maven2/com/madgag/bfg/1.14.0/bfg-1.14.0.jar

# PASO 1: Clonar con --mirror
git clone --mirror https://github.com/usuario/repo.git repo.git
cd repo.git

# PASO 2: Ejecutar BFG
# Eliminar archivos mayores a 100 MB
java -jar ../bfg-1.14.0.jar --strip-blobs-bigger-than 100M .

# Eliminar archivos por nombre/patrón
java -jar ../bfg-1.14.0.jar --delete-files '*.mp4' .

# Reemplazar texto (contraseñas, tokens)
java -jar ../bfg-1.14.0.jar --replace-text ../passwords.txt .

# PASO 3: Limpiar y empujar (BFG no limpia reflog automáticamente)
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force-with-lease
```

### Comparativa: filter-repo vs BFG

| Característica           | `git filter-repo`           | BFG Repo-Cleaner          |
| ------------------------ | --------------------------- | ------------------------- |
| Velocidad                | Muy rápida (Python)         | Rápida (Java)             |
| Instalación              | `pip install`               | Descargar `.jar`          |
| Flexibilidad             | Muy alta (callbacks Python) | Alta (casos predefinidos) |
| Mantenimiento            | Activo (oficial Git)        | Activo                    |
| Documentación            | Excelente                   | Buena                     |
| Curva de aprendizaje     | Media                       | Baja                      |

## 13.13 Verificar el Resultado

Después de cualquier reescritura, debes verificar la integridad del repositorio:

```bash
# Verificar el nuevo historial
git log --oneline --all --graph

# Verificar integridad de objetos
git fsck --full --strict

# Verificar que no quedaron referencias a los datos eliminados
git log --all --full-history -- <ruta-del-archivo-eliminado>
# Debe devolver: (nada)

# Comparar conteo de commits entre antes y después
git rev-list --count HEAD

# Inspeccionar tamaño del repositorio
git count-objects -vH
du -sh .git
```

## 13.14 `git push --force-with-lease` Después de Reescribir

Una reescritura cambia los SHAs de los commits. Para reflejar esto en el remoto:

```bash
# NUNCA uses --force a secas en repositorios compartidos
# git push --force  # PELIGROSO

# USA --force-with-lease (más seguro)
git push --force-with-lease origin main

# También empujar tags si fueron reescritos
git push --force-with-lease origin --tags
```

`--force-with-lease` verifica que tu copia local de la rama remota coincida con lo que realmente está en el servidor. Si alguien más empujó commits mientras tanto, el push será rechazado, evitando que sobrescribas trabajo ajeno.

```bash
# Simulación del peligro de --force
# Colega A: git push feature  (empuja commits X, Y)
# Tú:       git rebase -i     (reescribes la rama, generando X', Y')
# Tú:       git push --force  (SOBRESCRIBE X, Y sin preguntar)
# Colega A: git pull          (HISTORIAL INCONSISTENTE, trabajo perdido)

# Con --force-with-lease:
# Tú:       git push --force-with-lease
# Si colega A empujó algo nuevo, tu push será RECHAZADO
```

## 13.15 Coordinación con el Equipo

Si debes reescribir historia que ya fue compartida, el protocolo es:

```
1. ANUNCIAR en el canal del equipo:
   "Voy a reescribir la historia de main a las 18:00 UTC.
    Todos deben pushear su trabajo antes y re-clonar después."

2. VERIFICAR que nadie tiene trabajo sin pushear.

3. EJECUTAR la reescritura.

4. NOTIFICAR finalización:
   "Reescritura completada. Ejecuten:
    git fetch origin
    git reset --hard origin/main
    git clean -fdx"
```

Para los colaboradores después de una reescritura forzada:

```bash
# Opción 1: Reset duro (si no tienes trabajo local propio)
git fetch origin
git reset --hard origin/main
git clean -fdx

# Opción 2: Re-clonar (lo más seguro)
cd ..
rm -rf repo
git clone https://github.com/empresa/repo.git

# Opción 3: Rebase sobre la historia reescrita
git fetch origin
git rebase --onto origin/main <commit-antes-del-rewrite> <tu-rama>
```

---

## 13.16 `git replace` — La Alternativa No Destructiva

`git replace` te permite reemplazar un objeto Git (commit, blob, tree) por otro SIN modificar el historial original. El objeto original sigue existiendo. Solo cambias lo que Git muestra cuando lo referencias. Es como ponerle una máscara a un commit sin tocar el commit real.

### Cuándo Usar git replace en Vez de filter-repo o rebase

| Escenario | filter-repo | rebase | git replace |
|-----------|------------|--------|-------------|
| **¿Modifica SHAs?** | Sí (todo) | Sí (desde el punto cambiado) | No (solo visual) |
| **¿Rompe clones de otros?** | Sí | Sí | No |
| **Rollback** | Difícil (re-clonar backup) | Medio (reflog) | Trivial (git replace -d) |
| **Compartible** | Vía --force-with-lease | Vía --force-with-lease | Vía refs/replace (push normal) |
| **Rendimiento 100k+ commits** | Rápido (C) | Lento (shell) | Instantáneo (un reemplazo) |

**El caso de uso perfecto**: necesitas "corregir" un puñado de commits específicos en un repositorio compartido masivo donde un rebase o filter-repo sería traumático para el equipo.

### El Modelo Mental — Cómo Funciona Realmente

```
Antes de git replace:

refs/heads/main → commit AAAA (original, con bug en README)

Después de git replace:

refs/heads/main → commit AAAA (original, intacto en .git/objects)
refs/replace/AAAA → commit BBBB (reemplazo, con README corregido)

Git muestra BBBB cuando le pides AAAA, pero AAAA sigue existiendo.
Los clones de otros devs siguen viendo AAAA hasta que compartas refs/replace.
```

### Reemplazar un Commit Completo

```bash
# Escenario: Un commit histórico contiene un archivo con datos sensibles.
# No quieres reescribir TODO el historial porque 50 devs tienen clones.
# Solo necesitas que futuros clones y CI/CD vean la versión limpia.

# 1. Crear una rama temporal desde el commit problemático
git checkout -b fix-sensible <commit-problematico>

# 2. Corregir el archivo
echo "DATOS_REEMPLAZADOS" > archivo-sensible.txt
git add archivo-sensible.txt

# 3. Crear un NUEVO commit con los mismos metadatos pero contenido corregido
git commit --allow-empty -C <commit-problematico> --amend --no-edit

# 4. Guardar el SHA del nuevo commit
COMMIT_CORREGIDO=$(git rev-parse HEAD)

# 5. Volver a la rama original
git checkout main

# 6. Decirle a Git: "cuando veas <commit-problematico>, muestra <commit_corregido>"
git replace <commit-problematico> $COMMIT_CORREGIDO

# 7. Limpiar la rama temporal
git branch -D fix-sensible

# Ahora cualquier comando de Git (log, show, diff)
# mostrará el commit corregido en lugar del original.
git log --oneline  # Verás contenido corregido, mismo SHA

# Pero el objeto original sigue existiendo:
git cat-file -p <commit-problematico>  # Muestra el contenido ORIGINAL
git --no-replace-objects log --oneline  # Muestra la historia SIN reemplazos
```

### Reemplazar un Blob (Archivo Individual)

```bash
# Escenario: Un archivo específico en un commit histórico contiene una API key.
# Solo necesitas reemplazar ESE archivo, no todo el commit.

# 1. Crear una versión limpia del archivo
echo "API_KEY=redacted" > config/.env

# 2. Crear un nuevo blob con el contenido limpio
BLOB_LIMPIO=$(git hash-object -w config/.env)

# 3. Encontrar el blob original en el commit problemático
# (El blob de config/.env en ese commit específico)
BLOB_ORIGINAL=$(git ls-tree <commit-problematico> config/.env | awk '{print $3}')

# 4. Reemplazar: cuando Git vea el blob original, usará el limpio
git replace $BLOB_ORIGINAL $BLOB_LIMPIO

# 5. Verificar: el archivo se ve limpio
git show <commit-problematico>:config/.env
# Muestra: API_KEY=redacted

# 6. El blob original sigue intacto
git --no-replace-objects show <commit-problematico>:config/.env
# Muestra: API_KEY=sk_live_abc123
```

### Reemplazar Referencias de Padres (Para "Eliminar" Commits)

```bash
# Escenario: Un commit intermedio introdujo un archivo gigante
# que fue eliminado en el commit siguiente. Los clones son lentos.
# Quieres que Git "salte" ese commit visualmente.

# Commits: A → B → C → D (B introdujo el archivo gigante, C lo eliminó)
# Quieres:  A → D (saltar B y C, pero sin reescribir historia)

# 1. Crear un commit que reemplace a B, pero con padre = A (saltando B-C)
# (Esto es avanzado: usa git commit-tree con plumbing)
COMMIT_D=$(git rev-parse D)
COMMIT_A=$(git rev-parse A)

# Crear un commit con el tree de D pero padre = A
TREE_D=$(git cat-file -p $COMMIT_D | grep "^tree" | awk '{print $2}')
COMMIT_FALSO=$(echo "commit $TREE_D" | git commit-tree $TREE_D -p $COMMIT_A -m "Skipped B and C")

# 2. Reemplazar D con la versión "saltada"
git replace $COMMIT_D $COMMIT_FALSO

# Ahora el log muestra: A → D  (B y C invisibles)
# Pero los objetos B y C siguen existiendo para quien los necesite
```

### Compartir Reemplazos con el Equipo

```bash
# Los reemplazos son LOCALES por defecto. Para compartirlos:

# Publicar reemplazos al remoto
git push origin refs/replace/*

# O publicar un reemplazo específico
git push origin refs/replace/<sha-reemplazado>

# Los compañeros los obtienen con fetch (no clone, no pull):
git fetch origin refs/replace/*:refs/replace/*

# Configurar para que fetch siempre traiga reemplazos:
git config --add remote.origin.fetch '+refs/replace/*:refs/replace/*'

# Ver todos los reemplazos activos
git replace -l

# Listar en formato detallado
git replace -l --format='%h → %H (%s)'

# Eliminar un reemplazo (volver a ver la realidad):
git replace -d <sha-reemplazado>

# Eliminar TODOS los reemplazos
git replace -d $(git replace -l)
```

### Cuándo NO Usar git replace

```
❌ No uses git replace cuando:
  • Necesitas eliminar permanentemente datos sensibles.
    Los objetos originales siguen en .git. 'git replace' es cosmético.
    Para eliminación real: git filter-repo + rotar credenciales.

  • El commit problemático es el HEAD actual o cercano.
    Un simple 'git commit --amend' o 'git rebase -i' es más simple.
    git replace brilla con commits ANTIGUOS en historial compartido.

  • Necesitas cambiar el orden de commits o mezclarlos.
    git replace reemplaza objetos puntuales, no reestructura historia.

  • Todo el equipo está de acuerdo en reescribir.
    Si todos van a re-clonar, filter-repo es más limpio.
    git replace es para cuando NO PUEDES reescribir.
```

---

## 13.17 Programación de Callbacks en `git filter-repo` — Poder Total

`git filter-repo` no es solo una herramienta de línea de comandos. Es una biblioteca Python que puedes importar y programar. Los callbacks te permiten ejecutar lógica arbitraria durante el filtrado, desde renombrar archivos con regex hasta reescribir mensajes de commit basados en los cambios.

### Instalación y Setup para Callbacks

```bash
# filter-repo como biblioteca Python (no solo CLI)
pip install git-filter-repo

# Verificar que puedes importarlo
python3 -c "import git_filter_repo; print(git_filter_repo.__version__)"
```

### Callback 1: Renombrar Archivos con Regex

```python
#!/usr/bin/env python3
"""Renombra archivos según patrones regex durante el filtrado."""
import re
from git_filter_repo import RepoFilter

class ArchivoRenamer(RepoFilter):
    def __init__(self, patrones):
        super().__init__()
        self.patrones = patrones
        
    def filename_callback(self, filename):
        """Se llama para CADA archivo en CADA commit."""
        for patron, reemplazo in self.patrones:
            nuevo = re.sub(patron, reemplazo, filename)
            if nuevo != filename:
                print(f"  Renombrando: {filename} → {nuevo}")
                return nuevo
        return filename  # Sin cambios

# Uso: renombrar archivos .jsx a .tsx en TODO el historial
filtro = ArchivoRenamer([
    (r'\.jsx$', '.tsx'),
    (r'src/legacy/(.*)', r'src/\1'),  # Mover fuera de legacy/
])
filtro.run()
```

### Callback 2: Reescribir Mensajes de Commit Basados en Contenido

```python
#!/usr/bin/env python3
"""Reescribe mensajes de commit basados en qué archivos se modificaron."""
import re
from git_filter_repo import CommitFilter, Blob

class MensajeCommitRewriter(CommitFilter):
    def message_callback(self, message, commit):
        """Se llama para CADA commit. Puedes modificar el mensaje."""
        cambios = commit.file_changes
        
        # Detectar si se modificaron tests
        archivos_test = [c for c in cambios if 'test' in c.filename.lower()]
        
        # Detectar si se modificó configuración
        archivos_config = [c for c in cambios 
                          if c.filename.startswith('config/')]
        
        nuevo_mensaje = message.decode('utf-8', errors='replace')
        
        # Añadir trailer de convención si no existe
        if archivos_test and 'test:' not in nuevo_mensaje.lower():
            # Insertar scope de test
            if ':' in nuevo_mensaje:
                tipo, resto = nuevo_mensaje.split(':', 1)
                nuevo_mensaje = f"{tipo}(test):{resto}"
        
        return nuevo_mensaje.encode('utf-8')

filtro = MensajeCommitRewriter()
filtro.run()
```

### Callback 3: Filtrar por Autor con Lógica Condicional

```python
#!/usr/bin/env python3
"""Elimina commits de autores específicos pero preserva el trabajo."""
from git_filter_repo import RepoFilter

class FiltroAutorCondicional(RepoFilter):
    def __init__(self, autor_prohibido, autor_reemplazo):
        super().__init__()
        self.autor_prohibido = autor_prohibido
        self.autor_reemplazo = autor_reemplazo
        self.commits_cambiados = 0
        
    def commit_callback(self, commit):
        """Se llama por cada commit. Retorna None para ELIMINARLO."""
        autor = commit.author_email.decode('utf-8', errors='replace')
        
        if self.autor_prohibido in autor:
            # Opción 1: Reatribuir el commit a otro autor
            commit.author_name = self.autor_reemplazo[0].encode()
            commit.author_email = self.autor_reemplazo[1].encode()
            commit.committer_name = self.autor_reemplazo[0].encode()
            commit.committer_email = self.autor_reemplazo[1].encode()
            self.commits_cambiados += 1
            return commit  # Mantener con nuevo autor
            
            # Opción 2 (alternativa): Eliminar el commit
            # return None  # El commit desaparece
        
        return commit  # Mantener sin cambios
    
    def done(self):
        print(f"Total commits reatribuidos: {self.commits_cambiados}")

filtro = FiltroAutorCondicional(
    autor_prohibido="renunciado@empresa.com",
    autor_reemplazo=("Equipo Ingeniería", "dev@empresa.com")
)
filtro.run()
```

### Callback 4: Extraer Subdirectorio Preservando Tags

```python
#!/usr/bin/env python3
"""
Extrae un subdirectorio como repositorio independiente,
preservando tags que tocaron ese subdirectorio.
"""
from git_filter_repo import RepoFilter, Tag

class ExtractWithTags(RepoFilter):
    def __init__(self, subdirectorio):
        super().__init__()
        self.subdirectorio = subdirectorio
        self.tags_preservadas = 0
        self.tags_omitidas = 0
        
    def tag_callback(self, tag):
        """Decide qué tags preservar en el nuevo repositorio."""
        commit_tags = tag.ref  # El commit al que apunta el tag
        
        # Verificar si este commit toca el subdirectorio
        # (simplificado; en producción usarías git plumbing)
        archivos = commit_tags.file_changes if hasattr(commit_tags, 'file_changes') else []
        
        toca_subdirectorio = any(
            f.filename.startswith(self.subdirectorio) 
            for f in archivos
        )
        
        if toca_subdirectorio:
            self.tags_preservadas += 1
            return tag  # Mantener el tag
        else:
            self.tags_omitidas += 1
            return None  # Eliminar el tag
        
    def done(self):
        print(f"Tags preservadas: {self.tags_preservadas}")
        print(f"Tags omitidas (no tocaban el subdirectorio): {self.tags_omitidas}")

# Uso
filtro = ExtractWithTags("packages/auth")
filtro.run()
```

### Callback 5: Script de Migración Complejo — Monorepo a Múltiples Repos

```python
#!/usr/bin/env python3
"""
Divide un monorepo en múltiples repos, cada uno con:
- Su historial filtrado (solo commits que tocan su directorio)
- Sus tags (solo las que apuntan a commits preservados)
- Sus ramas (solo las que sobreviven al filtro)
"""
import subprocess
import sys
from pathlib import Path

def extraer_servicio(repo_origen, directorio, nombre_repo):
    """Extrae un directorio del monorepo como repo independiente."""
    print(f"\n{'='*60}")
    print(f"Extrayendo: {directorio} → {nombre_repo}")
    print(f"{'='*60}")
    
    # 1. Crear clon temporal del mirror
    temp_mirror = f"/tmp/{nombre_repo}_mirror.git"
    subprocess.run([
        "git", "clone", "--mirror", repo_origen, temp_mirror
    ], check=True)
    
    # 2. Filtrar: solo archivos DENTRO del directorio
    #    y reubicarlos a la raíz
    subprocess.run([
        "git", "-C", temp_mirror, "filter-repo",
        "--subdirectory-filter", directorio,
        "--force"
    ], check=True)
    
    # 3. Limpiar referencias huérfanas
    subprocess.run([
        "git", "-C", temp_mirror, "filter-repo",
        "--prune-empty=always",
        "--prune-degenerate=always",
        "--force"
    ], check=True)
    
    # 4. Crear repositorio usable
    output_dir = Path(nombre_repo)
    subprocess.run([
        "git", "clone", temp_mirror, str(output_dir)
    ], check=True)
    
    # 5. Limpiar mirror temporal
    subprocess.run(["rm", "-rf", temp_mirror])
    
    # 6. Estadísticas
    commits = subprocess.run(
        ["git", "-C", str(output_dir), "rev-list", "--count", "HEAD"],
        capture_output=True, text=True
    )
    print(f"  ✓ Repositorio creado: {output_dir}")
    print(f"  ✓ Commits preservados: {commits.stdout.strip()}")
    
    return output_dir


# Script principal: dividir monorepo en 3 repos
SERVICIOS = [
    ("packages/auth", "auth-service"),
    ("packages/payments", "payments-service"), 
    ("packages/notifications", "notifications-service"),
]

for directorio, nombre in SERVICIOS:
    extraer_servicio(".", directorio, nombre)

print(f"\n✓ Monorepo dividido en {len(SERVICIOS)} repositorios independientes")
```

---

## 13.18 Recuperación de Desastres en Reescritura

Las reescrituras de historia son una de las operaciones más peligrosas en Git. Aquí tienes el protocolo de emergencia para cuando algo sale mal.

### Antes de Reescribir — Tu Seguro de Vida

```bash
#!/bin/bash
# PRE-REESCRITURA: Ejecutar SIEMPRE antes de filter-repo o rebase masivo
# Guarda TODO lo necesario para revertir

BACKUP_DIR="/tmp/git-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p $BACKUP_DIR

echo "🔒 Creando backup pre-reescritura en: $BACKUP_DIR"

# 1. Backup de todas las referencias (ramas, tags, notes, etc.)
git for-each-ref --format='%(refname)' refs/ > $BACKUP_DIR/all-refs.txt
git for-each-ref --format='%(refname) %(objectname)' refs/ > $BACKUP_DIR/refs-shas.txt

# 2. Backup de reflog (contiene operaciones recientes)
cp .git/logs/HEAD $BACKUP_DIR/head-reflog 2>/dev/null
cp -r .git/logs/refs $BACKUP_DIR/refs-logs 2>/dev/null

# 3. Backup de la configuración actual
git config --list > $BACKUP_DIR/git-config.txt

# 4. Clon mirror (el backup DEFINITIVO)
git clone --mirror . $BACKUP_DIR/mirror-backup.git
echo "✓ Mirror backup creado"

# 5. Guardar los SHAs de TODOS los objetos (para fsck después)
git rev-list --all --objects > $BACKUP_DIR/all-objects.txt

echo "✓ Backup completo. Para restaurar:"
echo "  git clone $BACKUP_DIR/mirror-backup.git repo-restaurado"
```

### Escenario 1: filter-repo Se Ejecutó con los Argumentos Equivocados

```bash
# PÁNICO: Ejecutaste esto y borró el directorio equivocado
# git filter-repo --path docs/ --invert-paths  # Querías borrar docs/
# Pero borraste todo MENOS docs/. Horror.

# RECUPERACIÓN INMEDIATA (si hiciste backup mirror):
cd ..
git clone /tmp/git-backup-*/mirror-backup.git repo-rescatado
cd repo-rescatado

# Verificar que está todo
git log --oneline --all --graph
git fsck --full --strict

# Ahora vuelve a intentar con los argumentos CORRECTOS
```

### Escenario 2: force-with-lease Rechazado — Alguien Empujó Durante Tu Reesritura

```bash
# Intentaste empujar historia reescrita pero alguien más empujó antes:
# ! [rejected] main -> main (stale info)
# error: failed to push some refs to 'origin'

# OPCIÓN 1: Si los commits nuevos del colega deben preservarse
# (son valiosos y tu reescritura es descartable)
git fetch origin
git reset --hard origin/main  # Vuelves a la realidad compartida
git cherry-pick <tus-commits-no-pusheados>  # Re-aplicas tu trabajo

# OPCIÓN 2: Si tu reescritura es CRÍTICA y los commits del colega son pocos
# (coordinar con el colega para que re-aplique sobre tu historia)
git fetch origin
# Ver qué empujó el colega
git log HEAD..origin/main --oneline
git log origin/main..HEAD@{1} --oneline  # Lo que perdiste de tu lado

# Traer los commits del colega a tu historia reescrita
git cherry-pick <commits-del-colega>
git push --force-with-lease origin main  # Ahora sí
```

### Escenario 3: Reescritura Parcial — Algunas Ramas Quedaron Desincronizadas

```bash
# Reescritura afectó main pero feature/xyz todavía tiene commits
# basados en la historia ANTIGUA

# Diagnóstico: encontrar ramas que no han sido actualizadas
git branch --contains <sha-del-commit-antes-de-reescritura>

# Solución: para cada rama, rebasear sobre la nueva historia
for branch in $(git branch --contains <sha-viejo> | sed 's/\*//'); do
    echo "Reparando: $branch"
    git checkout $branch
    git rebase --onto main <sha-viejo> $branch || {
        echo "⚠️  Conflicto en $branch. Resuelve manualmente y haz:"
        echo "  git rebase --continue"
    }
done
```

### Escenario 4: Se Eliminó Información que Era Necesaria

```bash
# Usaste filter-repo para eliminar un archivo, pero ahora resulta
# que NECESITAS buscar algo en una versión antigua de ese archivo.

# Si tienes el mirror backup:
cd /tmp/git-backup-*/mirror-backup.git
# El archivo EXISTE en el mirror (es un clon anterior al filter-repo)
git log --all --full-history -- <archivo-eliminado>

# Recuperar una versión específica del mirror:
git show <commit>:<archivo> > archivo-rescatado.txt

# Si NO tienes backup... reza. Los objetos podrían no haber sido
# recolectados por git gc aún. Intenta:
git fsck --lost-found
# Busca en .git/lost-found/other/
# Pero no hay garantías. DE AHÍ LA IMPORTANCIA DEL BACKUP.
```

---

## 13.19 Caso de Estudio: Limpiar un Repositorio Antes de Publicarlo como Open Source

**Contexto:** La empresa "FintechXYZ" desarrolló durante 3 años un SDK de pagos en un repositorio privado. Ahora quiere publicarlo como open source. El historial contiene:

- Claves API y tokens de desarrollo en archivos `.env` commiteados
- Archivos binarios grandes (videos de demostración de 200 MB)
- Commits con mensajes tipo "wip", "test", "asdf"
- Correos personales de desarrolladores que ya no están en la empresa
- Un subdirectorio `internal/tools/` con herramientas internas que no deben publicarse

**Estrategia aplicada:**

```bash
# 1. Clonar mirror para trabajar (paso 1 de seguridad)
git clone --mirror repo-privado.git repo-limpieza.git
cd repo-limpieza.git

# 2. Analizar el repositorio
git filter-repo --analyze
cat .git/filter-repo/commits-by-size.txt

# 3. Eliminar directorio interno
git filter-repo --path internal/tools/ --invert-paths

# 4. Eliminar archivos grandes (>50MB)
git filter-repo --strip-blobs-bigger-than 50M

# 5. Reemplazar datos sensibles en todo el historial
cat > replacements.txt << 'EOF'
sk_live_.*==>***REDACTED***
ghp_.*==>***REDACTED***
password=.*==>password=***REDACTED***
EOF
git filter-repo --replace-text replacements.txt

# 6. Corregir autores con mailmap
cat > mailmap.txt << 'EOF'
Jane Doe <jane@fintech.com> <jane123@gmail.com>
Jane Doe <jane@fintech.com> <jane@old-laptop.local>
EOF
git filter-repo --mailmap mailmap.txt

# 7. Verificar integridad
git fsck --full --strict
git log -p | grep -i "sk_live\|ghp_"  # Sin resultados

# 8. Re-clonar para tener un repo no-bare
cd ..
git clone repo-limpieza.git sdk-publico
cd sdk-publico

# 9. Agregar README, LICENSE, CONTRIBUTING, CODE_OF_CONDUCT
echo "MIT License" > LICENSE
git add LICENSE README.md CONTRIBUTING.md CODE_OF_CONDUCT.md
git commit -m "docs: agregar archivos de proyecto open source"

# 10. Publicar en GitHub
git remote add origin https://github.com/fintech/sdk-publico.git
git push -u origin main --force-with-lease
```

**Resultado:** Repositorio reducido de 2.1 GB a 45 MB. Historial limpio, sin datos sensibles, autores correctos, listo para la comunidad open source.

---

 del Capítulo 13

La reescritura de historia es una herramienta poderosa que debe manejarse con respeto y protocolos claros. Los puntos clave son:

1. **Reescribe solo antes del push,** o con el equipo completamente coordinado.
2. **`git filter-repo` es la herramienta canónica moderna.** Sustituye a `filter-branch` con mejor rendimiento y seguridad.
3. **La eliminación de datos sensibles requiere dos pasos:** reescritura de Git + rotación de credenciales. El primero sin el segundo es insuficiente.
4. **`--force-with-lease` sobre `--force` siempre.** Protege contra la sobrescritura accidental de trabajo ajeno.
5. **Verifica todo después de reescribir** con `git fsck`, `git log` y comparaciones de integridad.
6. **La comunicación con el equipo es indispensable** cuando la historia compartida será modificada.

## Ejercicios Propuestos

### Ejercicio 1: Limpieza de Datos Sensibles

1. Crea un repositorio con 10 commits, donde en el commit #3 accidentalmente incluiste un archivo `secrets.json` con claves API ficticias.
2. Usa `git filter-repo` para eliminar completamente `secrets.json` del historial.
3. Verifica con `git log --all --full-history -- secrets.json` que el archivo ya no existe en ningún commit.
4. Simula que el repositorio está en GitHub: escribe los comandos que usarías para empujar la historia limpia.

### Ejercicio 2: Extracción de Subdirectorio

1. Crea un repositorio con estructura de monorepo:
   ```
   /
   ├── packages/
   │   ├── auth/
   │   ├── payments/
   │   └── notifications/
   └── README.md
   ```
2. Realiza 5 commits que modifiquen `packages/auth/` y 3 commits que modifiquen `packages/payments/`.
3. Extrae `packages/auth/` como un repositorio independiente usando `git filter-repo --subdirectory-filter`.
4. Verifica que el historial del nuevo repositorio solo contenga los 5 commits de `auth` y que las rutas estén ajustadas a la raíz.

### Ejercicio 3: Cambio Masivo de Autor

1. Configura `user.name` y `user.email` con valores ficticios (`Invitado <invitado@example.com>`).
2. Crea 10 commits en un repositorio de prueba.
3. Cambia la configuración a tus datos reales y crea 5 commits más.
4. Usa `git filter-repo --mailmap` para cambiar todos los commits del autor `invitado@example.com` a tu identidad real.
5. Verifica con `git log --format='%an <%ae>'` que todos los commits tienen los nuevos datos.

### Ejercicio 4: Simulación de Fuerza de Trabajo en Equipo

1. Crea un repositorio "central" (`git init --bare server.git`).
2. Clónalo en dos directorios: `dev-a` y `dev-b`.
3. En `dev-a`, reescribe los últimos 3 commits con `git rebase -i` y empuja con `--force-with-lease`.
4. En `dev-b`, simula tener trabajo local no pusheado e intenta sincronizar tras la reescritura usando `git rebase --onto`.
5. Explica qué habría pasado si en `dev-a` hubieras usado `--force` en lugar de `--force-with-lease` y `dev-b` hubiera empujado algo en el ínterin.

### Ejercicio 5: BFG Repo-Cleaner

1. Descarga BFG Repo-Cleaner en un repositorio de prueba.
2. Crea commits que incluyan archivos `.mp4` y `.zip` de gran tamaño (puedes usar `dd` para generar archivos grandes).
3. Usa BFG para eliminar todos los archivos `.mp4` del historial.
4. Compara la experiencia y el rendimiento con `git filter-repo` para la misma tarea.
5. Documenta en qué escenarios preferirías cada herramienta.

---

← [Capítulo anterior](12-conflictos.md) | [Inicio](README.md) | [Capítulo siguiente →](14-gran-escala.md)
