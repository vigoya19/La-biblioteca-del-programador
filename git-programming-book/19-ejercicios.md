# Capitulo 19: Ejercicios Practicos

Este capitulo presenta 28 ejercicios organizados por nivel de dificultad, mas 3 proyectos integradores. Cada ejercicio incluye un escenario detallado, los objetivos a lograr y pistas concisas con los comandos clave. Se recomienda completar todos los ejercicios antes de abordar los proyectos integradores.

**Requisitos previos**: Tener Git instalado (version 2.30+), una cuenta en GitHub/GitLab, y un editor de texto. Crear un repositorio local `git-ejercicios` para los ejercicios de nivel principiante e intermedio.

---

## 19.1 Nivel Principiante (10 Ejercicios)

### Ejercicio 1: Crear Repo, Primer Commit, Ver Log

**Escenario**: Eres un desarrollador que inicia un nuevo proyecto llamado `calculadora`.

**Objetivos**:
- Inicializar un repositorio Git.
- Configurar nombre de usuario y email.
- Crear archivo `calculadora.py` con una funcion `sumar(a, b)`.
- Hacer commit con mensaje descriptivo.
- Ver el historial de commits.

**Pistas**:
```bash
# Comandos clave
git init
git config user.name "..."
git config user.email "..."
echo 'def sumar(a, b): return a + b' > calculadora.py
git add calculadora.py
git commit -m "..."
git log --oneline
```

---

### Ejercicio 2: Crear y Fusionar una Rama de Feature

**Escenario**: Debes agregar una funcion `restar(a, b)` sin afectar el codigo en `main`.

**Objetivos**:
- Crear rama `feature/resta`.
- Hacer checkout a la nueva rama.
- Agregar la funcion `restar`, commitear.
- Volver a `main` y verificar que `restar` no existe.
- Fusionar la rama con merge fast-forward.
- Eliminar la rama fusionada.

**Pistas**:
```bash
git branch feature/resta
git switch feature/resta
# Editar archivo, agregar funcion
git add calculadora.py && git commit -m "feat: agregar funcion restar"
git switch main
git merge feature/resta
git branch -d feature/resta
```

---

### Ejercicio 3: Resolver un Conflicto Simple de Merge

**Escenario**: Dos desarrolladores (simulados por ti en dos ramas) modifican la misma linea de `calculadora.py`.

**Objetivos**:
- Crear rama `feature/multiplicar` y agregar `def multiplicar(a, b): return a * b`.
- En `main`, modificar la misma region del archivo agregando `def dividir(a, b): return a / b`.
- Intentar merge y resolver el conflicto manualmente.
- Verificar que ambas funciones coexisten tras la resolucion.

**Pistas**:
```bash
git switch -c feature/multiplicar
# Editar archivo: agregar multiplicar, commit
git switch main
# Editar archivo: agregar dividir en la misma zona, commit
git merge feature/multiplicar
# Conflicto! Editar archivo, eliminar marcadores <<< === >>>
git add calculadora.py
git commit -m "merge: integrar multiplicar y dividir"
```

---

### Ejercicio 4: Clonar un Repo, Hacer Cambios, Push

**Escenario**: Trabajas en equipo y necesitas contribuir a un repositorio remoto.

**Objetivos**:
- Crear un repositorio vacio en GitHub (sin README, sin .gitignore).
- Conectarlo como remoto `origin` a tu repo local.
- Hacer push de `main` al remoto.
- Clonar el repositorio en otro directorio.
- Hacer un cambio en el clon, commitear y pushear.
- Hacer pull en el repositorio original para traer el cambio.

**Pistas**:
```bash
git remote add origin <URL>
git push -u origin main
cd /tmp && git clone <URL> calculadora-clon
cd calculadora-clon
# Editar archivo
git add . && git commit -m "..." && git push
cd /ruta/original && git pull
```

---

### Ejercicio 5: Usar .gitignore Correctamente

**Escenario**: Tu proyecto genera archivos temporales y tiene configuracion local que no debe versionarse.

**Objetivos**:
- Crear `.gitignore` con reglas para `__pycache__/`, `*.pyc`, `.env`, `.vscode/`, y `*.log`.
- Crear esos archivos/directorios y verificar que `git status` los ignora.
- Forzar el tracking de un archivo normalmente ignorado.
- Verificar que `.gitignore` mismo SI se versiona.

**Pistas**:
```bash
# .gitignore
__pycache__/
*.pyc
.env
.vscode/
*.log
!importante.log

git status  # Debe mostrar solo .gitignore y archivos fuente
git add .gitignore && git commit -m "chore: agregar gitignore"
git add -f importante.log  # Forzar tracking de archivo ignorado
```

---

### Ejercicio 6: Deshacer Cambios en Working Directory con restore

**Escenario**: Modificaste `calculadora.py` accidentalmente y necesitas descartar los cambios.

**Objetivos**:
- Modificar `calculadora.py` agregando codigo basura.
- Usar `git restore` para descartar cambios no staggeados.
- Hacer stage de otro cambio y luego deshacer el stage con `git restore --staged`.
- Verificar en cada paso con `git status` y `git diff`.

**Pistas**:
```bash
echo "codigo basura" >> calculadora.py
git status
git restore calculadora.py
# Ahora staggear un cambio:
echo "otra funcion" >> calculadora.py
git add calculadora.py
git restore --staged calculadora.py
git restore calculadora.py
```

---

### Ejercicio 7: Modificar el Ultimo Commit con --amend

**Escenario**: Hiciste un commit pero olvidaste incluir un archivo y el mensaje tiene un typo.

**Objetivos**:
- Hacer commit de `calculadora.py` con un mensaje que contiene un error tipografico.
- Crear un archivo `README.md` que debia ir en ese commit.
- Usar `git commit --amend` para agregar el archivo y corregir el mensaje.
- Verificar que el SHA del commit cambio.

**Pistas**:
```bash
git commit -m "feat:agregar funcioens"  # Typo intencional
echo "# Calculadora" > README.md
git add README.md
git commit --amend -m "feat: agregar funciones basicas"
git log --oneline  # SHA diferente
```

> **Advertencia**: Solo usa `--amend` en commits que NO han sido pusheados al remoto. Reescribir historial compartido causa problemas al equipo.

---

### Ejercicio 8: Usar Stash para Guardar Trabajo Temporal

**Escenario**: Estas trabajando en una feature pero surge un bug urgente en `main` que debes arreglar primero.

**Objetivos**:
- Iniciar cambios en `feature/potencia` (sin commitear).
- Usar `git stash` para guardar el trabajo en progreso.
- Cambiar a `main` y corregir el bug.
- Volver a `feature/potencia` y recuperar el trabajo con `git stash pop`.
- Explorar `git stash list` y `git stash drop`.

**Pistas**:
```bash
git switch -c feature/potencia
echo "def potencia(a, b): return a ** b" >> calculadora.py
git stash push -m "WIP: funcion potencia"
git switch main
# Arreglar bug rapido, commit
git switch feature/potencia
git stash pop
git stash list  # Lista
git stash clear # Limpiar todos
```

---

### Ejercicio 9: Crear y Compartir un Tag Anotado

**Escenario**: La version 1.0.0 de la calculadora esta lista para release.

**Objetivos**:
- Crear un tag anotado `v1.0.0` con mensaje descriptivo.
- Verificar los detalles del tag.
- Pushear el tag al remoto.
- Clonar el repo en otro directorio y verificar que el tag esta presente.
- Crear un tag ligero y comparar la diferencia con `git show`.

**Pistas**:
```bash
git tag -a v1.0.0 -m "Release inicial: operaciones basicas"
git show v1.0.0
git push origin v1.0.0
# En otro directorio clonado:
git tag  # Debe mostrar v1.0.0

# Tag ligero vs anotado:
git tag v1.0.1-ligero  # Ligero
git show v1.0.1-ligero  # Solo muestra el commit
git show v1.0.0         # Muestra tagger, fecha, mensaje
```

---

### Ejercicio 10: Fetch y Pull con Rebase

**Escenario**: Otro desarrollador hizo cambios en `main` mientras trabajabas en tu rama.

**Objetivos**:
- Simular trabajo paralelo: clonar el repo en dos directorios.
- En el directorio A, hacer un cambio en `main` y pushear.
- En el directorio B, hacer `git fetch` y ver los cambios remotos.
- Hacer `git pull --rebase` para integrar cambios locales con remotos.
- Comparar el resultado con un `git pull` normal (merge).

**Pistas**:
```bash
# Directorio A
git switch main
echo "def info(): return 'v1.0'" >> calculadora.py
git commit -am "feat: agregar funcion info"
git push

# Directorio B
git fetch origin
git log origin/main  # Ver cambios remotos
git pull --rebase origin main
git log --oneline --graph  # Historial lineal
```

---

## 19.2 Nivel Intermedio (10 Ejercicios)

### Ejercicio 11: Rebase Interactivo: Squash 5 Commits en 1

**Escenario**: Durante el desarrollo creaste 5 commits "WIP" que necesitas consolidar en uno solo antes del PR.

**Objetivos**:
- Crear 5 commits en una rama `feature/validacion` con mensajes como "wip", "fix typo", "update", etc.
- Usar `git rebase -i HEAD~5` para hacer squash de los ultimos 4 en el primero.
- Reescribir el mensaje del commit resultante siguiendo Conventional Commits.
- Verificar con `git log --oneline` que hay un solo commit nuevo.

**Pistas**:
```bash
# Crear 5 commits...
git rebase -i HEAD~5
# Cambiar 'pick' por 'squash' (o 's') en los commits 2-5
# Guardar y salir, editar mensaje final
git log --oneline  # Debe mostrar 1 commit consolidado
```

---

### Ejercicio 12: Cherry-Pick: Mover Commits entre Ramas

**Escenario**: Un bug fix que hiciste en `main` tambien se necesita en la rama `release/1.0`.

**Objetivos**:
- Crear un commit de bug fix en `main`.
- Cambiar a la rama `release/1.0` (crearla desde un commit anterior a main).
- Usar `git cherry-pick` para aplicar el commit de bug fix en la rama release.
- Verificar que el cambio esta presente pero con diferente SHA.
- Cherry-pickear multiples commits con un rango.

**Pistas**:
```bash
# En main: fix bug
git add . && git commit -m "fix(core): corregir division por cero"

# En release/1.0:
git cherry-pick <SHA-del-fix>

# Cherry-pick de un rango (sin incluir A):
git cherry-pick A..B
```

---

### Ejercicio 13: Resolver Conflicto Complejo con Rerere

**Escenario**: Tienes una rama de feature longeva que constantemente tiene conflictos al hacer rebase con `main`.

**Objetivos**:
- Activar `git rerere` (reuse recorded resolution).
- Crear un escenario donde una rama feature larga tenga conflictos recurrentes con main.
- Resolver el conflicto una vez y verificar que Git recuerda la resolucion.
- Hacer un segundo rebase y confirmar que Git aplica automaticamente la resolucion grabada.

**Pistas**:
```bash
git config rerere.enabled true

# Simular: crear feature longeva, modificar main
# Hacer rebase, resolver conflicto
git add . && git rebase --continue
# Git graba la resolucion

# Segundo rebase:
# Git aplica la resolucion automaticamente (o pide confirmacion)
# Ver resoluciones grabadas:
ls .git/rr-cache
```

---

### Ejercicio 14: git bisect para Encontrar un Bug

**Escenario**: La aplicacion dejo de funcionar en algun momento entre v1.0.0 y HEAD. Tienes 30 commits en ese rango.

**Objetivos**:
- Crear un script `test.sh` que ejecute el proyecto y devuelva 0 si funciona, 1 si falla.
- Insertar un bug intencional en el commit #15 de una serie de 30 commits.
- Usar `git bisect run ./test.sh` para encontrar automaticamente el commit culpable.
- Verificar que bisect identifica exactamente el commit donde se introdujo el bug.
- Usar `git bisect log` para ver el camino de busqueda.

**Pistas**:
```bash
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run ./test.sh
# "abc1234 is the first bad commit"
git bisect log  # Ver historial de la busqueda
git bisect reset
```

---

### Ejercicio 15: Reescribir Historial: Cambiar Autor de Commits

**Escenario**: Configuraste mal tu email de Git y 5 commits tienen autor incorrecto.

**Objetivos**:
- Crear 5 commits con autor incorrecto (configurar email erroneo temporalmente).
- Corregir la configuracion de email.
- Usar `git rebase -i` o `git filter-branch` para cambiar el autor de esos commits.
- Verificar con `git log --format='%an %ae'` que los autores estan corregidos.
- Hacer push force (en un repo de practica) y entender las implicaciones.

**Pistas**:
```bash
git config user.email "incorrecto@mal.com"
# Crear 5 commits...
git config user.email "correcto@bien.com"

# Opcion A: Rebase interactivo
git rebase -i HEAD~5
# Cambiar 'pick' por 'edit' en cada commit
git commit --amend --author="Nombre <correcto@bien.com>" --no-edit
git rebase --continue

# Opcion B: filter-branch (deprecated) o filter-repo
git filter-repo --name-callback '
  return b"Nombre Correcto"
' --email-callback '
  return b"correcto@bien.com"
'
```

> **Advertencia**: Reescribir el historial requiere `git push --force-with-lease`. Coordina con tu equipo antes de hacerlo en repositorios compartidos.

---

### Ejercicio 16: Configurar un Hook Pre-Commit con Linting

**Escenario**: Quieres evitar commits que no pasen el linter del proyecto.

**Objetivos**:
- Crear un script de linting simple que revise archivos Python.
- Configurar un hook `pre-commit` que ejecute el script antes de cada commit.
- Probar que el hook bloquea commits que violan el linter.
- Hacer que el hook sea compartible con el equipo (simbolicamente o via script de setup).

**Pistas**:
```bash
# .git/hooks/pre-commit
#!/bin/bash
echo "Ejecutando linter..."
# Corregido: --include y --exclude eran contradictorios
if grep -r "print(" *.py 2>/dev/null; then
    echo "ERROR: Se encontraron declaraciones print(). Usa logging en su lugar."
    exit 1
fi
```

---

### Ejercicio 17: Submodulos: Agregar y Actualizar Dependencia

**Escenario**: Tu proyecto principal depende de una libreria compartida en otro repositorio Git.

**Objetivos**:
- Crear un repositorio separado `lib-comun` con utilidades compartidas.
- Agregarlo como submódulo en el proyecto principal en `libs/comun`.
- Clonar el proyecto principal con `--recurse-submodules` y verificar.
- Hacer un cambio en `lib-comun`, pushearlo, y actualizar el submódulo en el proyecto principal.
- Explorar `git submodule update --remote` y `git submodule foreach`.

**Pistas**:
```bash
# En proyecto principal:
git submodule add <URL-lib-comun> libs/comun
git commit -m "chore: agregar submódulo lib-comun"

# Clonar con submódulos:
git clone --recurse-submodules <URL-proyecto>

# Actualizar submódulo a ultimo commit de su rama:
git submodule update --remote libs/comun

# Ejecutar comando en todos los submódulos:
git submodule foreach git pull origin main
```

---

### Ejercicio 18: Crear Release en GitHub con GitHub Actions

**Escenario**: Quieres automatizar la publicacion de releases cuando se crea un tag `v*`.

**Objetivos**:
- Crear workflow de GitHub Actions que se dispare con tags `v*`.
- El workflow debe: ejecutar tests, construir artefacto, crear release en GitHub.
- Crear un tag `v1.0.0` y verificar que se genera la release automaticamente.
- Incluir el changelog en el body de la release.

**Pistas**:
```yaml
# .github/workflows/release.yml
on:
  push:
    tags: ['v*']
jobs:
  release:
    steps:
      - uses: actions/checkout@v4
      - run: npm test
      - run: npm run build
      - uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          files: dist/**
```

---

### Ejercicio 19: git reflog para Recuperar Rama Borrada

**Escenario**: Borraste accidentalmente la rama `feature/importante` con `git branch -D`.

**Objetivos**:
- Crear una rama con 3 commits valiosos.
- Borrarla agresivamente con `git branch -D`.
- Usar `git reflog` para encontrar el ultimo commit de la rama.
- Recrear la rama desde ese commit.
- Verificar que los 3 commits estan recuperados.

**Pistas**:
```bash
git branch feature/importante
# Crear 3 commits en esa rama...
git switch main
git branch -D feature/importante

git reflog
# Buscar HEAD@{N} donde se hizo el ultimo commit de la rama
git branch feature/importante <SHA-del-ultimo-commit>
git switch feature/importante
git log --oneline  # 3 commits recuperados
```

---

### Ejercicio 20: Sparse Checkout en Monorepo

**Escenario**: Trabajas en un monorepo enorme con 50 paquetes, pero solo necesitas 2.

**Objetivos**:
- Crear un repositorio que simule un monorepo con 5 directorios de paquetes.
- Configurar `sparse-checkout` para ver solo el paquete `pkg-a` y `pkg-c`.
- Verificar que `ls` solo muestra esos dos paquetes.
- Agregar un tercer paquete al sparse-checkout.
- Desactivar sparse-checkout y ver todos los paquetes nuevamente.

**Pistas**:
```bash
git clone --no-checkout <URL> monorepo
cd monorepo
git sparse-checkout init --cone
git sparse-checkout set pkg-a pkg-c
git checkout main  # Solo descarga esos directorios

# Agregar mas directorios:
git sparse-checkout add pkg-e

# Ver configuracion:
git sparse-checkout list

# Desactivar:
git sparse-checkout disable
```

---

## 19.3 Nivel Avanzado (5 Ejercicios)

### Ejercicio 21: git filter-repo: Extraer Subdirectorio a Nuevo Repo

**Escenario**: `src/auth-module` ha crecido tanto que merece su propio repositorio, preservando su historial.

**Objetivos**:
- Instalar `git-filter-repo`.
- Extraer `src/auth-module` a un nuevo repositorio, manteniendo solo los commits que afectaron ese directorio.
- Reescribir paths para que el contenido quede en la raiz del nuevo repo.
- Agregar el nuevo repo como remoto y verificar el historial preservado.
- Opcional: extraer con exclusion de archivos especificos.

**Pistas**:
```bash
pip install git-filter-repo

# Extraer subdirectorio a nuevo repo
git clone <repo-original> repo-filtrado
cd repo-filtrado
git filter-repo --subdirectory-filter src/auth-module

# Extraer con paths reescritos
git filter-repo --path src/auth-module/ --path-rename src/auth-module/:

# Verificar
git log --oneline  # Solo commits que tocaron ese dir
ls  # Contenido en la raiz
```

---

### Ejercicio 22: Automatizar Semantic Release con Conventional Commits

**Escenario**: Configurar un pipeline completo de semantic release.

**Objetivos**:
- Configurar conventional commits con commitlint y husky.
- Configurar semantic-release en GitHub Actions.
- Realizar commits `feat`, `fix` y `BREAKING CHANGE` para ver el versionado.
- Verificar que se generan releases, tags y CHANGELOG automaticamente.
- Probar el flujo completo: commit -> push -> release automatica.

**Pistas**:
```bash
npm init -y
npm install --save-dev @commitlint/cli @commitlint/config-conventional husky
npx commitlint --from HEAD~1 --verbose

npm install --save-dev semantic-release @semantic-release/git

# Crear .releaserc.json con plugins: commit-analyzer, release-notes-generator,
# changelog, npm, git, github
# Crear .github/workflows/release.yml que ejecute npx semantic-release
```

---

### Ejercicio 23: Configurar CI/CD Completa con Test + Build + Deploy

**Escenario**: Configurar un pipeline profesional completo para una aplicacion web.

**Objetivos**:
- Pipeline con stages: lint, test (matrix), build, security scan, deploy staging, deploy prod.
- Deploy staging automatico en push a `develop`.
- Deploy produccion manual con aprobacion en push a `main`.
- Notificaciones en Slack o email en caso de fallo.
- Cache de dependencias entre ejecuciones.

**Pistas**:
```yaml
# Usar GitHub Actions o GitLab CI con:
# - Matrix builds para test en multiples versiones
# - Artifacts para compartir build entre jobs
# - Environments con proteccion para produccion
# - Slack notification action
# - actions/cache para node_modules

# Estructura de stages:
# lint (independiente)
# test (matrix: node 18, 20, 22)
# build (needs: lint, test)
# security (independiente, paralelo)
# deploy-staging (needs: build, if: develop)
# deploy-prod (needs: build, environment: production, manual approval)
```

---

### Ejercicio 24: Escribir un Hook Post-Receive para Auto-Deploy

**Escenario**: Tienes un servidor con Git y quieres que cada push a `main` despliegue automaticamente.

**Objetivos**:
- Crear un repositorio bare en el servidor.
- Escribir un hook `post-receive` que haga checkout en `/var/www/app`.
- Configurar el hook para que solo despliegue pushes a `main`.
- Reiniciar el servidor web despues del deploy.
- Probar el flujo: push local -> deploy automatico.

**Pistas**:
```bash
# En el servidor:
git init --bare /srv/git/mi-app.git

# /srv/git/mi-app.git/hooks/post-receive
#!/bin/bash
while read oldrev newrev refname; do
    if [ "$refname" = "refs/heads/main" ]; then
        echo "Deployando main..."
        GIT_WORK_TREE=/var/www/app git checkout -f main
        cd /var/www/app
        npm ci --production
        systemctl restart mi-app
        echo "Deploy completado."
    fi
done

chmod +x hooks/post-receive
```

---

### Ejercicio 25: Analizar y Optimizar un Repo Grande

**Escenario**: Un repositorio de 2GB necesita ser optimizado para mejorar el rendimiento.

**Objetivos**:
- Crear un repositorio con commits grandes intencionalmente (archivos binarios).
- Identificar los objetos mas grandes con `git rev-list` y `git cat-file`.
- Usar `git gc --aggressive` para comprimir.
- Configurar Git LFS para archivos binarios existentes (migrar).
- Usar `git filter-repo` para eliminar archivos grandes del historial.
- Verificar la reduccion de tamaño con `du -sh .git`.

**Pistas**:
```bash
# Encontrar objetos grandes
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  awk '/^blob/ {print $0}' | sort -k3 -nr | head -20

# Garbage collection agresivo
git gc --aggressive --prune=now

# Instalar LFS
git lfs install
git lfs track "*.psd" "*.zip" "*.mp4"
git add .gitattributes
git commit -m "chore: configurar LFS"

# Migrar archivos existentes a LFS
git lfs migrate import --include="*.psd,*.zip" --everything

# Eliminar archivos grandes del historial
git filter-repo --path-glob '*.iso' --invert-paths
```

---

### Ejercicio 26: Recuperar un Detached HEAD

**Escenario**: Estabas explorando un commit antiguo y accidentalmente hiciste cambios valiosos en estado detached HEAD.

**Objetivos**:
- Hacer checkout de un commit antiguo (no una rama).
- Crear 2 commits en estado detached HEAD.
- Darte cuenta del problema y recuperar los commits.
- Crear una rama apuntando a los commits recuperados.

**Pistas**:
```bash
git checkout <SHA-antiguo>  # Entras en detached HEAD
echo "cambio valioso" >> archivo.txt
git add . && git commit -m "fix importante"
# Git advierte: "You are in 'detached HEAD' state"
echo "otro cambio" >> archivo.txt
git add . && git commit -m "fix adicional"

# Recuperación:
git branch recuperado HEAD  # Crea rama en el commit actual
git checkout main
git merge recuperado
# O con reflog si ya cambiaste de HEAD:
git reflog  # Buscar los commits del detached HEAD
git branch recuperado <SHA-del-segundo-commit>
```

---

### Ejercicio 27: git rebase --onto para Reubicar una Rama

**Escenario**: Tu rama `feature/api-v2` nació de `develop` pero ahora quieres moverla a `main` porque `develop` tiene cambios experimentales que no necesitas.

**Objetivos**:
- Crear rama `feature/api-v2` desde `develop`.
- Hacer 3 commits en `feature/api-v2`.
- Usar `git rebase --onto` para mover esos 3 commits sobre `main`.
- Verificar que los commits ahora tienen `main` como base.
- Eliminar `develop` del historial de la rama.

**Pistas**:
```bash
# Estado inicial: feature/api-v2 nace de develop
git checkout develop
git checkout -b feature/api-v2
# ... 3 commits ...
echo "API v2 changes" >> api.js
git add . && git commit -m "feat(api): v2 endpoint"

# Mover a main con --onto
git rebase --onto main develop feature/api-v2
# Trasplanta commits desde develop..feature/api-v2 sobre main

git log --oneline --graph --all
# feature/api-v2 ahora sale de main, no de develop
```

---

### Ejercicio 28: git bisect con Términos Personalizados

**Escenario**: El rendimiento de la aplicación se degradó entre v2.0.0 y HEAD. Usa bisect con términos "fast" y "slow".

**Objetivos**:
- Crear un script `benchmark.sh` que mida rendimiento y devuelva 0 si es rápido, 1 si es lento.
- Insertar un commit que degrade rendimiento intencionalmente.
- Usar `git bisect --term-old=fast --term-new=slow` para encontrarlo.
- Verificar que bisect identifica correctamente el commit con términos personalizados.

**Pistas**:
```bash
cat > benchmark.sh << 'EOF'
#!/bin/bash
# Simula benchmark: compara tiempo de ejecución
# En un caso real: tiempo=$(time ./app 2>&1)
# if [ $tiempo -gt 1000 ]; then exit 1; else exit 0; fi
grep -q "slow_code" main.js && exit 1 || exit 0
EOF
chmod +x benchmark.sh

git bisect start --term-old=fast --term-new=slow
git bisect slow HEAD
git bisect fast v2.0.0
git bisect run ./benchmark.sh
# abc1234 is the first slow commit
git bisect reset
```

---

### Ejercicio 29: git worktree para Trabajo Paralelo

**Escenario**: Estás desarrollando `feature/auth` cuando surge un hotfix urgente en producción. No quieres stashear ni perder tu contexto de editor.

**Objetivos**:
- Crear worktree para `feature/auth` en desarrollo.
- Crear un segundo worktree para el hotfix desde `main`.
- Arreglar el bug en el worktree del hotfix, commitear, pushear.
- Volver al worktree original y verificar que todo sigue intacto.
- Eliminar el worktree del hotfix al terminar.

**Pistas**:
```bash
# Worktree principal: estás en feature/auth
git worktree list

# Crear worktree para hotfix
git worktree add -b hotfix/critico ../proyecto-hotfix main
cd ../proyecto-hotfix
echo "fix urgente" >> archivo.txt
git add . && git commit -m "hotfix: corregir error crítico"
git push origin hotfix/critico

# Volver al worktree original
cd ../proyecto  # O la ruta original
# Todo sigue como estaba: archivos sin guardar, editor abierto, etc.

# Limpiar
git worktree remove ../proyecto-hotfix
git worktree list  # Solo queda el worktree principal
```

---

## 19.4 Proyectos Integradores

### Proyecto Integrador 1: Simulacion Completa de Trabajo en Equipo

**Escenario**: Simula un equipo de 3 personas (tres clones locales) trabajando en el mismo proyecto durante una semana.

**Roles**:
- **Alice** (Lead): Revisa PRs, mantiene `main`, maneja releases.
- **Bob** (Developer): Desarrolla features, crea PRs.
- **Carlos** (Junior): Corrige bugs simples, actualiza documentacion.

**Actividades**:
1. Configurar proteccion de ramas en GitHub/GitLab (main requiere PR + review).
2. Bob crea `feature/auth` con login, crea PR, Alice lo aprueba y mergea.
3. Carlos crea `bugfix/typo-readme`, pero hace el PR apuntando a `develop`. Corregir.
4. Alice crea `release/1.0.0`, Carlos encuentra un bug, Bob hace `hotfix/urgente`.
5. Resolver conflictos de merge cuando dos PRs tocan el mismo archivo.
6. Alice hace release final con tag `v1.0.0` y semantic release.
7. Simular un rollback: Alice revierte el release anterior usando `git revert`.

**Pistas generales**:
```bash
# Configurar proteccion de ramas desde la UI de GitHub/GitLab

# Trabajar con fork + PR para simular colaboradores externos
git remote add alice <url-repo-alice>
git fetch alice

# Releases
git tag -a v1.0.0 -m "v1.0.0: primera release"
git push origin v1.0.0

# Rollback
git revert -m 1 <SHA-del-merge>
```

---

### Proyecto Integrador 2: Migracion de SVN a Git Preservando Historia

**Escenario**: Una empresa legacy usa SVN y quiere migrar a Git sin perder 5 años de historial.

**Actividades**:
1. Crear un repositorio SVN local de prueba con branches, tags y trunk.
2. Usar `git svn clone` para migrar a Git con todo el historial.
3. Mapear branches y tags de SVN a branches y tags de Git.
4. Limpiar el historial migrado (eliminar archivos binarios grandes, commits vacios).
5. Configurar un archivo `authors.txt` para mapear usuarios SVN a usuarios Git.
6. Verificar la integridad del historial: commits totales, autores, fechas.
7. Pushear el repositorio migrado a GitHub/GitLab.

**Pistas**:
```bash
# Preparar authors.txt
# svn-user = Git User <git-email@example.com>
alice = Alice <alice@ejemplo.com>
bob = Bob <bob@ejemplo.com>

# Clonar SVN
git svn clone <svn-url> --stdlayout --authors-file=authors.txt --prefix=svn/ repo-git

# Migrar branches de SVN a branches de Git
git for-each-ref --format='%(refname:short)' refs/remotes/svn/ | \
  while read ref; do
    git branch "$ref" "remotes/$ref"
  done

# Migrar tags
git for-each-ref --format='%(refname:short)' refs/remotes/svn/tags | \
  while read tag; do
    git tag "$tag" "refs/remotes/svn/tags/$tag"
  done

# Limpiar remotos SVN
git remote add origin <nuevo-url-git>
git push origin --all
git push origin --tags
```

---

### Proyecto Integrador 3: Configuracion de un Monorepo con CI/CD y Proteccion de Ramas

**Escenario**: Una organizacion quiere migrar 3 repositorios separados a un monorepo con CI/CD optimizado.

**Estructura**:
```
monorepo/
  packages/
    web/         # React app
    api/         # Express API
    shared/      # Libreria comun
  .github/
    workflows/
      web-ci.yml
      api-ci.yml
      shared-ci.yml
      release.yml
```

**Actividades**:
1. Migrar los 3 repositorios existentes al monorepo (preservar historial de cada uno).
2. Configurar proteccion de ramas en `main` para todos los paquetes.
3. Crear pipelines CI independientes por paquete usando filtros de paths.
4. Configurar pipeline CD que solo despliegue el paquete que cambio.
5. Implementar versionado independiente por paquete.
6. Configurar codeowners para que cada paquete tenga reviewers especificos.
7. Implementar cache compartida de dependencias entre pipelines.
8. Configurar pre-commit hooks que ejecuten linters por paquete.

**Pistas**:
```yaml
# Ejemplo de path filtering en GitHub Actions
on:
  push:
    paths:
      - 'packages/web/**'
      - '.github/workflows/web-ci.yml'

# CODEOWNERS
packages/web/ @equipo-frontend
packages/api/ @equipo-backend
packages/shared/ @equipo-frontend @equipo-backend

# Cache de dependencias
- uses: actions/cache@v4
  with:
    path: |
      packages/*/node_modules
      ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

```bash
# Migrar cada repo preservando historial:
# Para cada repositorio origen:
cd repo-original
git filter-repo --to-subdirectory-filter packages/web
git remote add monorepo <url-monorepo>
git push monorepo main
```

---

## Tabla Resumen de Comandos por Ejercicio

| Ejercicio | Comandos Clave |
|-----------|---------------|
| 1 | `init`, `config`, `add`, `commit`, `log` |
| 2 | `branch`, `switch`, `merge`, `branch -d` |
| 3 | `switch -c`, `merge`, resolucion manual |
| 4 | `remote add`, `push -u`, `clone`, `pull` |
| 5 | `.gitignore`, `status`, `add -f` |
| 6 | `restore`, `restore --staged`, `diff`, `status` |
| 7 | `commit --amend` |
| 8 | `stash push`, `stash pop`, `stash list` |
| 9 | `tag -a`, `show`, `push origin <tag>` |
| 10 | `fetch`, `pull --rebase` |
| 11 | `rebase -i`, squash |
| 12 | `cherry-pick`, cherry-pick con rango |
| 13 | `config rerere.enabled`, `rebase` |
| 14 | `bisect start/bad/good/run/reset` |
| 15 | `rebase -i` + `commit --amend --author`, `filter-repo` |
| 16 | hook `pre-commit`, `chmod +x` |
| 17 | `submodule add`, `clone --recurse-submodules`, `update --remote` |
| 18 | GitHub Actions `on push tags`, `softprops/action-gh-release` |
| 19 | `reflog`, `branch <nombre> <SHA>` |
| 20 | `sparse-checkout init/set/add/disable` |
| 21 | `git-filter-repo --subdirectory-filter` |
| 22 | semantic-release, commitlint, husky |
| 23 | GitHub Actions matrix, environments, artifacts, cache |
| 24 | hook `post-receive`, `GIT_WORK_TREE` |
| 25 | `rev-list`, `gc`, `lfs migrate`, `filter-repo` |

---

## Resumen del Capitulo 19

Este capitulo presenta 29 ejercicios practicos y 3 proyectos integradores que cubren todo el espectro de Git, desde comandos basicos hasta configuracion avanzada de CI/CD y monorepos. Cada ejercicio esta diseñado para reforzar un concepto especifico con escenarios realistas y pistas concisas. Los proyectos integradores simulan situaciones profesionales completas: trabajo en equipo, migracion de SVN, y configuracion de monorepo empresarial. La practica sistematica de estos ejercicios garantiza el dominio practico de Git.

---

## Soluciones Comentadas

A continuacion se presentan soluciones detalladas para los ejercicios mas representativos de cada nivel. Se recomienda intentar cada ejercicio antes de consultar la solucion.

### Ejercicio 1: Primer Repositorio

```bash
mkdir calculadora && cd calculadora
git init
git config user.name "Tu Nombre"
git config user.email "tu@email.com"
echo 'def sumar(a, b): return a + b' > calculadora.py
git add calculadora.py
git commit -m "feat: agregar funcion sumar basica"
git log --oneline
# abc1234 feat: agregar funcion sumar basica
```

### Ejercicio 3: Resolver Conflicto de Merge

```bash
# Rama feature/multiplicar
git switch -c feature/multiplicar
echo -e 'def sumar(a, b): return a + b\ndef multiplicar(a, b): return a * b' > calculadora.py
git add calculadora.py && git commit -m "feat: agregar multiplicar"

# En main
git switch main
echo -e 'def sumar(a, b): return a + b\ndef dividir(a, b): return a / b' > calculadora.py
git add calculadora.py && git commit -m "feat: agregar dividir"

# Merge con conflicto
git merge feature/multiplicar
# CONFLICT (content): Merge conflict in calculadora.py
# Editar archivo: conservar ambas funciones
cat > calculadora.py << 'EOF'
def sumar(a, b): return a + b
def multiplicar(a, b): return a * b
def dividir(a, b): return a / b
EOF
git add calculadora.py
git commit -m "merge: integrar multiplicar y dividir en calculadora"
```

### Ejercicio 11: Rebase Interactivo con Squash

```bash
# Crear 5 commits WIP
for i in 1 2 3 4 5; do
  echo "cambio $i" >> feature.txt
  git add . && git commit -m "wip: cambio $i"
done

# Squash interactivo
git rebase -i HEAD~5
# En el editor, cambiar:
# pick abc1234 wip: cambio 1  (dejar como pick)
# squash def5678 wip: cambio 2 (cambiar a squash)
# squash ghi9012 wip: cambio 3
# squash jkl3456 wip: cambio 4
# squash mno7890 wip: cambio 5

# En el segundo editor, escribir el mensaje consolidado:
# feat(validacion): implementar validacion de formularios
git log --oneline  # Un solo commit consolidado
```

### Ejercicio 14: git bisect

```bash
# Script de test que detecta el bug
cat > test.sh << 'EOF'
#!/bin/bash
# El bug: si existe el archivo bug.txt devuelve 1
[ -f bug.txt ] && exit 1 || exit 0
EOF
chmod +x test.sh

# El commit #15 (de 30) introduce el archivo bug.txt
# El commit #14 no lo tiene -> test pasa (exit 0)
# El commit #15 lo crea -> test falla (exit 1)

git bisect start HEAD v1.0.0
git bisect run ./test.sh
# Bisecting: 14 revisions left (roughly 4 steps)
# abc1234 is the first bad commit
git show abc1234  # Revisar qué introdujo el bug
git bisect reset
```

### Ejercicio 16: Hook Pre-Commit (Corregido)

```bash
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
echo "Ejecutando linter..."
# Verificar que no haya print() en archivos Python
if grep -r "print(" *.py 2>/dev/null; then
    echo "ERROR: Se encontraron declaraciones print(). Usa logging."
    exit 1
fi
EOF
chmod +x .git/hooks/pre-commit
```

### Ejercicio 19: Recuperar Rama Borrada

```bash
# Borrar rama
git branch -D feature/importante

# Recuperar con reflog
git reflog
# abc1234 HEAD@{3}: commit: feat: ultimo cambio importante
git branch feature/importante abc1234
git switch feature/importante
git log --oneline  # 3 commits recuperados
```

### Ejercicio 21: filter-repo Extraer Subdirectorio

```bash
pip install git-filter-repo
git clone repo-original.git repo-filtrado
cd repo-filtrado
git filter-repo --subdirectory-filter src/auth-module
# Historial: solo commits que tocaron src/auth-module
# Rutas reescritas: src/auth-module/file.js → file.js
git log --oneline
```

### Ejercicio 24: Hook Post-Receive Auto-Deploy

```bash
# En el servidor, crear hook
cat > /srv/git/mi-app.git/hooks/post-receive << 'EOF'
#!/bin/bash
while read oldrev newrev refname; do
    if [ "$refname" = "refs/heads/main" ]; then
        echo "[$(date)] Deployando main (${newrev:0:7})..."
        GIT_WORK_TREE=/var/www/app git checkout -f main
        cd /var/www/app || exit 1
        npm ci --production
        systemctl reload nginx
        echo "[$(date)] Deploy completado."
    fi
done
EOF
chmod +x /srv/git/mi-app.git/hooks/post-receive
```

### Ejercicio 25: Optimizar Repo Grande

```bash
# Identificar objetos grandes
git rev-list --objects --all | \
  git cat-file --batch-check='%(objectsize) %(objectname) %(rest)' | \
  sort -nr | head -20

# Si hay binarios: migrar a LFS
git lfs track "*.psd" "*.zip"
git lfs migrate import --include="*.psd,*.zip" --everything

# Si son archivos a eliminar:
git filter-repo --path-glob '*.iso' --invert-paths

# GC final
git gc --aggressive --prune=now
du -sh .git  # Comparar antes y después
```

### Ejercicio 26: Recuperar Detached HEAD

```bash
git checkout abc1234  # Commit antiguo, detached HEAD
echo "cambio valioso" >> fix.txt
git add . && git commit -m "fix: correccion en estado detached"
echo "otro cambio" >> fix.txt
git add . && git commit -m "fix: mejora adicional"
# Git: "You are in 'detached HEAD' state"

# Recuperar
git branch rama-recuperada HEAD
git checkout main
git merge rama-recuperada
```

### Ejercicio 27: git rebase --onto

```bash
# feature/api-v2 nace de develop (que tiene commits experimentales)
git checkout develop
git checkout -b feature/api-v2
echo "API v2 endpoint" > api.js
git add . && git commit -m "feat(api): v2 base"
echo "API v2 auth" >> api.js
git add . && git commit -m "feat(api): v2 auth"
echo "API v2 tests" >> api.js
git add . && git commit -m "test(api): v2 tests"

# Mover a main, descartando dependencia de develop
git rebase --onto main develop feature/api-v2
# Ahora feature/api-v2 se basa en main, no en develop
git log --oneline --graph --all
```

### Ejercicio 29: git worktree

```bash
# Estás en feature/auth trabajando
# Surge hotfix urgente

# Crear worktree para hotfix sin perder tu contexto
git worktree add -b hotfix/critico ../proyecto-hotfix main
cd ../proyecto-hotfix

# Arreglar el bug
echo "fix: validar input nulo" >> server.js
git add . && git commit -m "hotfix: validar input nulo en endpoint"
git push origin hotfix/critico

# Volver al trabajo original
cd ../proyecto  # Ruta del worktree principal
# Todo sigue exactamente como lo dejaste

# Limpiar
git worktree remove ../proyecto-hotfix
```

---

## Ejercicios Propuestos

1. **Completa los 10 ejercicios de nivel principiante** en orden y registra el tiempo que te toma cada uno. Identifica cuales comandos necesitas reforzar.

2. **Ejercicios intermedios guiados por un mentor**: Realiza los ejercicios 11 al 20 con un compañero. Uno actua como "desarrollador" y otro como "revisor de PR", alternando roles. El revisor debe verificar atomicidad de commits y formato de mensajes.

3. **Proyecto integrador personalizado**: Adapta el Proyecto Integrador 1 a un stack tecnologico que uses (Python/Django, Java/Spring, Node.js/Express). Implementa las protecciones de rama y simulacion de equipo real.

4. **Automatizacion completa**: Basandote en el Proyecto Integrador 3, configura un monorepo real con al menos 2 paquetes, CI/CD con path filtering, y semantic release. Documenta todo el proceso en un `CONTRIBUTING.md`.

5. **Auditoria de repositorio existente**: Toma un repositorio real en el que trabajes y aplica los ejercicios 14 (bisect), 19 (reflog), y 25 (optimizacion). Identifica al menos una mejora concreta que puedas implementar.

---

---

← [Capítulo anterior](18-buenas-practicas.md) | [Inicio](README.md) | [Capítulo siguiente →](20-activos-no-codigo.md)
