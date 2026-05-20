# Capitulo 4: Repositorios Remotos

Los repositorios remotos son el corazon del trabajo colaborativo en Git. Mientras que un repositorio local te permite trabajar de forma independiente, los remotos habilitan la sincronizacion, el respaldo y la colaboracion entre multiples desarrolladores. En este capitulo exploraremos todo lo necesario para dominar el trabajo con remotos: desde la configuracion basica hasta flujos avanzados de colaboracion.

## 4.1 Concepto de Remoto

Un **remoto** es una referencia a un repositorio Git alojado en otra ubicacion, ya sea en un servidor, en otro directorio del mismo equipo o en una plataforma como GitHub, GitLab o Bitbucket. Cada remoto tiene un nombre corto (por defecto `origin`) y una URL que apunta a su ubicacion.

Cuando clonas un repositorio con `git clone`, Git automaticamente:

- Crea una copia local completa del repositorio.
- Agrega un remoto llamado `origin` que apunta al repositorio fuente.
- Configura la rama principal (`main` o `master`) para hacer tracking de `origin/main`.

El flujo tipico de trabajo con remotos sigue este ciclo:

```
git pull    # Obtener cambios remotos y fusionarlos
# ... trabajar localmente ...
git add .
git commit -m "Mensaje"
git push   # Enviar cambios al remoto
```

Los remotos son simplemente alias de URLs. Puedes tener multiples remotos configurados en un mismo repositorio local, lo cual es util para trabajar con forks o con multiples fuentes.

## 4.2 Gestion de Remotos con `git remote`

El comando `git remote` es la herramienta principal para administrar las conexiones remotas. Veamos cada subcomando en detalle.

### 4.2.1 Listar Remotos

```bash
# Listar nombres de remotos
git remote

# Listar con URLs (verbose)
git remote -v
```

En un repositorio recien clonado veras algo como:

```
origin  https://github.com/usuario/proyecto.git (fetch)
origin  https://github.com/usuario/proyecto.git (push)
```

Cada remoto puede tener URLs separadas para fetch y push, lo cual es util en flujos donde se lee de un repositorio publico y se escribe en uno privado.

### 4.2.2 Agregar un Remoto

```bash
# Agregar un nuevo remoto
git remote add <nombre> <url>

# Ejemplo: agregar el remoto de un compañero
git remote add carlos https://github.com/carlos/proyecto.git

# Ejemplo: agregar upstream en un fork
git remote add upstream https://github.com/original/proyecto.git
```

> **Convencion:** En proyectos con forks, `origin` suele apuntar a tu fork y `upstream` al repositorio original. Esta convencion es ampliamente adoptada en GitHub y GitLab.

### 4.2.3 Renombrar y Eliminar Remotos

```bash
# Renombrar un remoto
git remote rename origin upstream-original

# Eliminar un remoto
git remote remove carlos
# Equivalente:
git remote rm carlos
```

Al eliminar un remoto, tambien se eliminan las referencias de tracking asociadas (como `remotes/carlos/main`).

### 4.2.4 Inspeccionar un Remoto

```bash
# Mostrar informacion detallada de un remoto
git remote show origin
```

Este comando muestra:
- URLs de fetch y push
- Ramas remotas y su relacion con ramas locales
- Ramas configuradas para `git pull`
- Ramas locales que seran empujadas con `git push`

### 4.2.5 Gestionar URLs de Remotos

```bash
# Obtener la URL de un remoto
git remote get-url origin

# Cambiar la URL de un remoto (util al migrar repositorios)
git remote set-url origin git@github.com:nuevo-usuario/proyecto.git

# Agregar una URL de push separada
git remote set-url --add --push origin git@github.com:respaldo/proyecto.git
```

El comando `set-url --add --push` permite configurar multiples destinos de push, enviando los cambios a todos simultaneamente.

> **Tip:** Si migras un repositorio de HTTPS a SSH (o viceversa), usa `git remote set-url` en lugar de eliminar y volver a agregar el remoto. Asi conservas todas las configuraciones de tracking.

## 4.3 Obteniendo Cambios con `git fetch`

`git fetch` descarga objetos (commits, arboles, blobs, tags) del remoto sin modificar tu arbol de trabajo ni tus ramas locales. Es una operacion segura que puedes ejecutar en cualquier momento.

### 4.3.1 Fetch Basico

```bash
# Fetch de un remoto especifico
git fetch origin

# Fetch de todos los remotos
git fetch --all

# Fetch de una rama especifica
git fetch origin feature-x
```

Despues de un fetch, los commits descargados estan disponibles a traves de las ramas remotas (ej. `origin/main`). Puedes inspeccionarlos antes de fusionarlos:

```bash
# Ver que cambios trajo el fetch
git log main..origin/main --oneline

# Ver el diff de los cambios entrantes
git diff main origin/main
```

### 4.3.2 Opciones Avanzadas de Fetch

```bash
# Fetch con limpieza: elimina referencias locales obsoletas
git fetch --prune
git fetch -p  # equivalente

# Fetch incluyendo tags
git fetch --tags

# Fetch forzado (sobrescribe referencias locales)
git fetch --force
```

La opcion `--prune` es especialmente util en equipos grandes donde las ramas remotas se crean y eliminan constantemente. Sin ella, tus referencias a `origin/rama-eliminada` persisten localmente.

> **Advertencia:** `--force` sobrescribe tus referencias remotas locales. Usalo solo si sabes que el historial remoto fue reescrito (por ejemplo, despues de un force push legitimado por el equipo).

| Comando | Efecto |
|---------|--------|
| `git fetch origin` | Descarga objetos de `origin` |
| `git fetch --all` | Descarga de todos los remotos |
| `git fetch --prune` | Descarga y limpia referencias obsoletas |
| `git fetch --tags` | Descarga objetos y tags |

### 4.3.3 Fetch vs Pull

```
git fetch  →  Solo descarga, no modifica tu trabajo
git pull   →  fetch + merge (o fetch + rebase)
```

Muchos desarrolladores prefieren hacer fetch explícito y luego decidir como integrar los cambios, en lugar de usar pull directamente.

## 4.4 Integrando Cambios con `git pull`

`git pull` es una operacion compuesta: primero ejecuta `git fetch` y luego integra los cambios descargados mediante `merge` o `rebase`.

### 4.4.1 Pull con Merge (Comportamiento por Defecto)

```bash
# Pull con merge
git pull origin main

# Equivalente a:
git fetch origin main
git merge origin/main
```

Si hay conflictos durante el merge, Git te pedira resolverlos y luego hacer commit.

### 4.4.2 Pull con Rebase

```bash
# Pull con rebase en lugar de merge
git pull --rebase origin main

# Configurar rebase como comportamiento por defecto
git config pull.rebase true   # para todas las ramas
git config branch.main.rebase true  # solo para main
```

El pull con rebase reubica tus commits locales sobre la punta de la rama remota, evitando merges innecesarios y manteniendo un historial lineal.

> **Recomendacion:** Configura `pull.rebase true` globalmente si prefieres historiales lineales y trabajas con ramas de feature que solo tu tocas.

### 4.4.3 Configurar Tracking de Ramas

```bash
# Al hacer push, establece upstream automaticamente
git push -u origin feature-x

# Configurar upstream manualmente
git branch --set-upstream-to=origin/main main

# Ver ramas con tracking configurado
git branch -vv
```

La salida de `git branch -vv` muestra cada rama local y su relacion con la rama remota:

```
* feature-x  a1b2c3d [origin/feature-x: ahead 2, behind 1] Agregar modulo de autenticacion
  main       d4e5f6g [origin/main] Version estable 1.2.3
```

## 4.5 Enviando Cambios con `git push`

`git push` envia los commits locales al repositorio remoto. Es la operacion que comparte tu trabajo con el equipo.

### 4.5.1 Push Basico

```bash
# Push a la rama con tracking configurado
git push

# Push a un remoto y rama especificos
git push origin main

# Push estableciendo upstream (-u)
git push -u origin feature-x
```

La primera vez que empujas una rama nueva, Git te pedira que uses `--set-upstream` (o `-u`) para establecer la relacion de tracking.

Desde Git 2.37+, `push.autoSetupRemote` elimina la necesidad de `-u`:

```bash
# Configuracion moderna (recomendado)
git config --global push.autoSetupRemote true

# Ahora, simplemente:
git push origin feature-x
# Git automaticamente configura el tracking sin necesidad de -u
```

### 4.5.2 Force Push: `--force` vs `--force-with-lease`

Cuando el historial local y el remoto han divergido (por ejemplo, despues de un rebase), un push normal sera rechazado. Existen dos opciones de fuerza:

```bash
# Force push destructivo (NO recomendado)
git push --force

# Force push seguro (RECOMENDADO)
git push --force-with-lease
```

| Opcion | Comportamiento | Riesgo |
|--------|---------------|--------|
| `--force` | Sobrescribe el remoto sin verificacion | **Alto**: puede borrar commits de otros |
| `--force-with-lease` | Sobrescribe solo si el remoto no ha cambiado desde tu ultimo fetch | **Bajo**: protege contra sobreescrituras accidentales |
| `--force-if-includes` | Similar a lease, pero verifica que tus commits incluyan la punta remota | **Minimo** |

```bash
# Ejemplo: despues de rebasar feature-x
git fetch origin
git rebase origin/main
git push --force-with-lease origin feature-x
```

> **Peligro:** `git push --force` puede borrar commits de tus compañeros sin previo aviso. Usa siempre `--force-with-lease` a menos que estes absolutamente seguro de lo que haces.

### Escenario: El peligro de `--force`

```
Estado inicial:
  origin/feature:  A---B---C  (Alice y Bob comparten esta base)

1. Alice hace:
   git commit -m "D"
   git push                    → origin/feature: A---B---C---D

2. Bob (que no ha hecho fetch reciente) hace:
   git commit --amend -m "C mejorado"   # Reescribe C como C'
   git push
   # ! rejected (non-fast-forward)

   Bob insiste:
   git push --force
   # RESULTADO: origin/feature: A---B---C'    (D de Alice se PERDIO)

3. Alice la proxima vez que hace pull ve que D desaparecio.
```

**Como evitarlo:**
```bash
# Bob debio usar:
git fetch origin
git log origin/feature..HEAD --oneline   # Ver que tiene que empujar
# Si ve que origin/feature avanzo, debe integrar:
git rebase origin/feature
git push --force-with-lease origin feature

# --force-with-lease habria protegido a Alice:
# ! rejected: stale info (remote tiene D, tu fetch no lo tiene)
```

**Recuperacion si ya paso:**
- El commit D de Alice sobrevive en el reflog de Bob (si Bob hizo fetch antes del force push).
- Alice tambien puede recuperarlo de su reflog local con `git reflog` y `git cherry-pick`.
- Los objetos huerfanos tardan ~90 dias en ser eliminados por `git gc`.

### 4.5.3 Push de Tags

```bash
# Empujar todos los tags
git push --tags

# Empujar un tag especifico
git push origin v2.0.0

# Empujar tags anotados solamente (recomendado)
git push --follow-tags
```

`--follow-tags` solo empuja tags anotados que sean alcanzables desde los commits que estas empujando, evitando tags accidentales.

### 4.5.4 Eliminar Ramas y Tags Remotos

```bash
# Eliminar una rama remota
git push origin --delete feature-obsoleta

# Sintaxis alternativa (menos intuitiva)
git push origin :feature-obsoleta

# Eliminar un tag remoto
git push origin --delete v1.0.0
```

### 4.5.5 Opciones de Configuracion de Push

```bash
# Configurar push por defecto (solo la rama actual)
git config push.default simple

# Push solo si el upstream coincide en nombre
git config push.default upstream

# Push de todas las ramas con tracking
git config push.default current
```

| `push.default` | Comportamiento |
|----------------|---------------|
| `simple` (defecto) | Empuja solo la rama actual a su upstream si tiene el mismo nombre |
| `upstream` | Empuja la rama actual a su upstream configurado |
| `current` | Empuja la rama actual a una rama remota con el mismo nombre |
| `matching` | Empuja todas las ramas que tengan equivalente remoto |

## 4.6 Pull Requests (PR) y Merge Requests (MR)

Los Pull Requests (GitHub, Bitbucket) o Merge Requests (GitLab) son el mecanismo principal para revisar y aprobar cambios antes de integrarlos a la rama principal.

### 4.6.1 Concepto

Un Pull Request es una solicitud para que los mantenedores del repositorio "tiren" (pull) de tus cambios hacia su rama. El flujo tipico es:

```
1. Crear una rama de feature → git checkout -b feature/nueva-funcionalidad
2. Hacer commits → git add . && git commit -m "Implementar X"
3. Empujar la rama → git push -u origin feature/nueva-funcionalidad
4. Abrir el PR desde la interfaz web
5. Revision de codigo, discusion, ajustes
6. Merge del PR (por el mantenedor o automaticamente)
```

### 4.6.2 Opciones de Merge en PRs

Las plataformas ofrecen distintas estrategias para integrar un PR:

| Estrategia | Comando Git equivalente | Resultado |
|------------|------------------------|-----------|
| **Merge commit** | `git merge --no-ff` | Crea un commit de merge explícito |
| **Squash and merge** | `git merge --squash` | Combina todos los commits del PR en uno solo |
| **Rebase and merge** | `git rebase` + `git merge --ff-only` | Reubica los commits del PR en la punta de la rama destino |

```bash
# Simulacion local de cada estrategia:

# Merge commit:
git checkout main
git merge --no-ff feature-x

# Squash:
git checkout main
git merge --squash feature-x
git commit -m "Feature X: descripcion consolidada"

# Rebase:
git checkout feature-x
git rebase main
git checkout main
git merge --ff-only feature-x
```

### 4.6.3 Buenas Practicas en Pull Requests

- **Commits atomicos:** Cada commit debe representar un cambio logico y tener un mensaje descriptivo.
- **PRs pequeños:** Prefiere PRs de 200-400 lineas; son mas faciles de revisar.
- **Descripcion clara:** Explica que hace el cambio, por que y como probarlo.
- **Referencia issues:** Vincula el PR con issues usando `Closes #123`, `Fixes #456`.
- **Responde a reviews:** Cada comentario debe ser respondido o resuelto.
- **Manten tu rama actualizada:** Haz rebase o merge de `main` antes de que te aprueben el PR.

> **Tip:** Usa `gh pr create` (GitHub CLI) o `glab mr create` (GitLab CLI) para abrir PRs directamente desde la terminal sin salir de tu flujo de trabajo.

### Draft Pull Requests

Los **Draft PRs** (GitHub) o **WIP MRs** (GitLab) permiten abrir un PR visible al equipo pero marcado como "no listo para merge". Son ideales para:
- Compartir codigo temprano para feedback informal.
- Ejecutar CI/CD sin bloquear revisiones.
- Mostrar el approach antes de refinar detalles.

```bash
# GitHub CLI: crear draft PR
gh pr create --draft --title "OAuth integration" --body "Trabajo en progreso"

# GitLab CLI: crear WIP MR
glab mr create --wip --title "Draft: OAuth"
```

### Plantillas de Pull Request

Crea un archivo `.github/pull_request_template.md` (GitHub) o `.gitlab/merge_request_templates/default.md` (GitLab) para estandarizar la informacion que debe incluir cada PR:

```markdown
## Descripcion
<!-- Explica el que y el por que de este cambio -->

## Tipo de cambio
- [ ] Bug fix
- [ ] Nueva feature
- [ ] Refactor
- [ ] Documentacion

## Checklist
- [ ] Tests agregados/actualizados
- [ ] Documentacion actualizada
- [ ] Probado localmente

## Issues relacionadas
Closes #123
```

### Stacked PRs

Los **Stacked PRs** son multiples PRs encadenados que dependen uno del otro. En lugar de un PR enorme, creas PRs incrementales basados en la rama del PR anterior:

```
PR #1: feature/base      ← main
PR #2: feature/auth       ← feature/base (PR #1)
PR #3: feature/auth-oauth ← feature/auth (PR #2)
```

**Herramientas para stacked PRs:**
- **Graphite**: CLI y web app especializada en stacks.
- **git rebase --update-refs** (Git 2.38+): mantiene las ramas del stack actualizadas durante rebases.
- **ghstack**: script Python para simplificar el flujo en GitHub.

## 4.7 Forks

Un **fork** es una copia personal de un repositorio ajeno en tu cuenta. Es el mecanismo estandar para contribuir a proyectos open source donde no tienes permisos de escritura directa.

### 4.7.1 Flujo de Trabajo con Forks

```bash
# 1. Hacer fork desde la interfaz web (GitHub/GitLab/Bitbucket)

# 2. Clonar tu fork
git clone https://github.com/tu-usuario/proyecto.git
cd proyecto

# 3. Agregar el repositorio original como upstream
git remote add upstream https://github.com/original/proyecto.git

# 4. Verificar remotos
git remote -v
# origin    https://github.com/tu-usuario/proyecto.git (fetch)
# origin    https://github.com/tu-usuario/proyecto.git (push)
# upstream  https://github.com/original/proyecto.git (fetch)
# upstream  https://github.com/original/proyecto.git (push)
```

### 4.7.2 Mantener el Fork Sincronizado

El repositorio original avanza y tu fork se desactualiza. Para sincronizarlo:

```bash
# 1. Obtener cambios del upstream
git fetch upstream

# 2. Cambiar a tu rama main
git checkout main

# 3. Fusionar los cambios del upstream
git merge upstream/main

# Alternativa: usar rebase para historial limpio
git rebase upstream/main

# 4. Actualizar tu fork en GitHub
git push origin main
```

> **Automatizacion:** GitHub ofrece un boton "Sync fork" en la interfaz web. Tambien puedes configurar GitHub Actions para sincronizar periodicamente.

### 4.7.3 Flujo Completo de Contribucion

```bash
# 1. Sincronizar main con upstream
git checkout main
git pull upstream main
git push origin main

# 2. Crear rama de feature desde main actualizado
git checkout -b feature/mi-contribucion

# 3. Trabajar y commitear
git add .
git commit -m "Implementar funcionalidad X"

# 4. Empujar a tu fork
git push -u origin feature/mi-contribucion

# 5. Abrir Pull Request desde tu fork hacia upstream/main

# 6. Si hay cambios solicitados, actualizar la rama:
git add .
git commit -m "Corregir feedback de revision"
git push origin feature/mi-contribucion
```

## 4.8 Protocolos de Conexion

Git soporta varios protocolos para comunicarse con remotos. Cada uno tiene sus ventajas y casos de uso.

### 4.8.1 HTTPS

El protocolo mas comun, especialmente en entornos corporativos donde el trafico SSH puede estar bloqueado.

```bash
# Clonar via HTTPS
git clone https://github.com/usuario/proyecto.git

# Configurar credential helper para no repetir credenciales
git config --global credential.helper cache    # Cache en memoria (defecto 15 min)
git config --global credential.helper 'cache --timeout=3600'  # Cache 1 hora
git config --global credential.helper osxkeychain  # macOS Keychain
git config --global credential.helper libsecret    # Linux (GNOME Keyring)
git config --global credential.helper manager-core # Windows (Git Credential Manager)
```

**Tokens de Acceso Personal (PAT):**

Desde 2020-2021, GitHub y GitLab requieren tokens en lugar de contraseñas para operaciones HTTPS:

```bash
# Usar token como contraseña (GitHub)
Username: tu-usuario
Password: ghp_xxxxxxxxxxxxxxxxxxxx  # Tu Personal Access Token

# Configurar token en la URL (menos seguro)
git remote set-url origin https://ghp_xxxxxxxxxxxx@github.com/usuario/proyecto.git

# Con GitLab
git remote set-url origin https://oauth2:glpat-xxxxxxxxxxxx@gitlab.com/usuario/proyecto.git
```

> **Seguridad:** Nunca incluyas tokens en URLs que se almacenaran en el historial de bash o en scripts compartidos. Usa credential helpers o variables de entorno.

### 4.8.2 SSH

SSH es el protocolo preferido por desarrolladores por su seguridad y conveniencia (sin necesidad de ingresar credenciales repetidamente, si usas ssh-agent).

**Configurar llaves SSH:**

```bash
# 1. Generar par de llaves (si no tienes una)
ssh-keygen -t ed25519 -C "tu-email@ejemplo.com"
# O RSA (compatible con sistemas antiguos):
ssh-keygen -t rsa -b 4096 -C "tu-email@ejemplo.com"

# 2. Iniciar ssh-agent y agregar la llave
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 3. Copiar llave publica
cat ~/.ssh/id_ed25519.pub
# Agregar el contenido en GitHub/GitLab: Settings → SSH Keys

# 4. Probar conexion
ssh -T git@github.com
ssh -T git@gitlab.com
```

**Configurar multiples llaves SSH:**

```bash
# ~/.ssh/config
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal

Host github.com-trabajo
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_trabajo

Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519
```

Luego ajustas las URLs de los remotos:

```bash
git remote set-url origin git@github.com:usuario/proyecto.git
git remote set-url origin git@github.com-trabajo:empresa/proyecto.git
```

### 4.8.3 Git Protocol

Protocolo nativo de Git (git://). Solo lectura, sin autenticacion. Practicamente en desuso porque GitHub lo desactivo en 2022 y la mayoria de plataformas lo han abandonado en favor de HTTPS/SSH.

```bash
# Ya no funcional en la mayoria de servidores:
git clone git://github.com/usuario/proyecto.git
```

### 4.8.4 Comparativa de Protocolos

| Protocolo | Autenticacion | Encriptacion | Velocidad | Puertos |
|-----------|--------------|-------------|-----------|---------|
| **HTTPS** | Usuario/Token | SSL/TLS | Rapido | 443 |
| **SSH** | Llaves SSH | SSH | Rapido | 22 |
| **Git** | Ninguna | No | Muy rapido | 9418 |

## 4.9 Autenticacion y Herramientas CLI

### 4.9.1 GitHub CLI (`gh`)

```bash
# Instalar en macOS
brew install gh

# Instalar en Linux (Ubuntu/Debian)
sudo apt install gh

# Autenticarse
gh auth login

# Operaciones comunes
gh repo clone usuario/proyecto
gh pr create --title "Nueva feature" --body "Descripcion del cambio"
gh pr list
gh pr view 42
gh pr checkout 42
gh pr review --approve
gh pr merge --squash
gh issue create --title "Bug report" --body "Detalles..."
gh release create v1.0.0
```

### 4.9.2 GitLab CLI (`glab`)

```bash
# Instalar en macOS
brew install glab

# Autenticarse
glab auth login

# Operaciones comunes
glab repo clone usuario/proyecto
glab mr create --title "Nueva feature" --description "Descripcion"
glab mr list
glab mr view 15
glab mr checkout 15
glab mr approve 15
glab mr merge 15
```

### 4.9.3 Credential Helpers por Plataforma

```bash
# macOS: Keychain (integrado)
git config --global credential.helper osxkeychain

# Linux: GNOME Keyring
sudo apt install libsecret-1-0 libsecret-1-dev
git config --global credential.helper libsecret

# Linux: archivo plano (inseguro, solo para desarrollo)
git config --global credential.helper store

# Windows: Git Credential Manager (viene con Git for Windows)
git config --global credential.helper manager-core

# Helper personalizado (script)
git config --global credential.helper "/ruta/a/mi-helper.sh"
```

## 4.10 Colaboracion Basica

### 4.10.1 Sincronizacion Diaria

El ciclo diario de un desarrollador trabajando en equipo:

```bash
# Al inicio del dia
git checkout main
git pull --rebase origin main

# Crear rama de trabajo
git checkout -b feature/tarea-123

# Trabajar...
git add .
git commit -m "Avance parcial"

# Antes de terminar, actualizar con cambios del equipo
git fetch origin
git rebase origin/main

# Si hay conflictos, resolverlos:
# ... editar archivos ...
git add .
git rebase --continue

# Push final
git push -u origin feature/tarea-123
```

### 4.10.2 Resolver Conflictos Remotos

Cuando dos personas modifican las mismas lineas y una ya empujo:

```bash
# Intento de push es rechazado
git push origin main
# ! [rejected] main -> main (non-fast-forward)

# 1. Obtener cambios remotos
git fetch origin

# 2. Integrar (merge o rebase)
git rebase origin/main
# CONFLICT (content): Merge conflict in src/app.js

# 3. Resolver conflicto editando el archivo
# Archivo con marcadores:
# <<<<<<< HEAD
#   tu version
# =======
#   version remota
# >>>>>>> origin/main

# 4. Marcar como resuelto
git add src/app.js

# 5. Continuar
git rebase --continue

# 6. Empujar
git push origin main
```

### 4.10.3 Estrategia de Integracion en Equipos

**Trunk-Based Development (ramas de corta vida):**

```bash
# Ramas de feature de 1-2 dias maximo
git checkout -b feature/pequena main
# ... 1-2 commits ...
git fetch origin
git rebase origin/main
git push -u origin feature/pequena
# Abrir PR → merge rapido
```

**GitFlow (ramas de larga vida):**

```bash
# Ramas develop, release, hotfix
git checkout -b release/1.2.0 develop
# ... estabilizacion ...
git checkout develop
git merge release/1.2.0
```

## 4.11 Refspecs: Especificaciones de Referencia

Un **refspec** es el mecanismo interno que usa Git para mapear referencias entre repositorios locales y remotos. Define que ramas/tags se transfieren durante `fetch`, `push` y `pull`.

### Anatomia de un Refspec

```
[+]<fuente>:<destino>
```

- `+`: prefijo opcional para actualizaciones non-fast-forward (equivalente a `--force`).
- `<fuente>`: referencia en el repositorio origen.
- `<destino>`: referencia en el repositorio destino.

### Refspecs por defecto

```bash
# Fetch: el refspec por defecto configurado en .git/config
[remote "origin"]
    fetch = +refs/heads/*:refs/remotes/origin/*
    #     ↑                                  ↑
    #   todas las ramas remotas         ramas de tracking locales

# Push (cuando haces git push origin main):
# Implicitamente: refs/heads/main:refs/heads/main
```

### Ejemplos practicos

```bash
# Fetch de una rama especifica con nombre local diferente
git fetch origin refs/heads/main:refs/remotes/origin/main-custom

# Fetch solo tags estables (no tags de CI), ignorando el resto
git fetch origin '+refs/tags/v*:refs/tags/v*'

# Push de una rama local a una remota con distinto nombre
git push origin refs/heads/feature-x:refs/heads/feature-beta

# Eliminar rama remota usando refspec
git push origin :refs/heads/feature-obsoleta

# Fetch todo pero no crear ramas remotas
git fetch origin refs/heads/*:refs/heads/*
# ⚠️ Sobrescribe ramas locales - peligroso - prefiere el default
```

### Caso practico: solo fetch de ramas de release

```bash
# En .git/config:
[remote "origin"]
    url = https://github.com/usuario/proyecto.git
    fetch = +refs/heads/release/*:refs/remotes/origin/release/*

# git fetch origin ahora solo descarga ramas release/*
# Ignora main, develop, feature/*, etc.
```

> **Dato clave:** Los refspecs son la razon por la que `git fetch` no sobrescribe tus ramas locales: el `+` permite non-fast-forward en `remotes/origin/*` pero tus ramas en `refs/heads/` estan protegidas.

---

## 4.12 git bundle: Transferencias Offline

`git bundle` empaqueta un repositorio (o parte de el) en un solo archivo que puede transferirse por USB, red interna sin internet, o correo electronico. Es ideal para entornos air-gapped o con conectividad limitada.

### Crear un bundle

```bash
# Crear bundle de una rama especifica
git bundle create repo.bundle main

# Bundle de todo el repositorio (todas las ramas y tags)
git bundle create repo.bundle --all

# Bundle incremental: solo commits nuevos desde cierto punto
git bundle create actualizacion.bundle origin/main..main

# Bundle con un rango especifico
git bundle create feature.bundle origin/main..feature-x
```

### Verificar un bundle

```bash
# Inspeccionar contenido del bundle antes de usarlo
git bundle verify repo.bundle
# repo.bundle is okay

# Listar referencias contenidas
git bundle list-heads repo.bundle
# a1b2c3d refs/heads/main
# d4e5f6g refs/heads/develop
```

### Clonar desde un bundle

```bash
# Clonar desde bundle exactamente como desde una URL
git clone repo.bundle mi-copia-local
cd mi-copia-local
git remote add origin https://github.com/usuario/proyecto.git
```

### Caso practico: red air-gapped

```
Entorno con internet (dev machine):
  git bundle create actualizacion.bundle origin/main..main
  # Copiar actualizacion.bundle a USB

Entorno air-gapped (servidor de produccion):
  git remote add usb /media/usb/actualizacion.bundle
  git fetch usb
  git merge usb/main
```

---

## 4.13 Mirror Clones

Un **mirror clone** (`git clone --mirror`) crea una replica exacta del repositorio remoto, incluyendo TODAS las referencias (ramas, tags, refs/notes, etc.) sin configurar un working directory.

### clone --mirror vs clone --bare

| Aspecto | `--bare` | `--mirror` |
|---|---|---|
| Working directory | No | No |
| Todas las ramas | No, solo la principal | Si, todas |
| Tags | Solo los alcanzables | Todos |
| Refspec configurado | fetch normal | `fetch = +refs/*:refs/*` |
| Update automatico | No | `git remote update` |

```bash
# Crear mirror completo
git clone --mirror https://github.com/usuario/proyecto.git

# Actualizar mirror con cambios remotos
cd proyecto.git
git remote update
git fetch --prune
```

### Casos de uso

| Escenario | Comando |
|---|---|
| **Backup completo** de un repositorio | `git clone --mirror` |
| **Migracion** entre plataformas (GitHub → GitLab) | `git clone --mirror` + `git push --mirror` |
| **Servidor local** de cache para equipos grandes | Mirror + `git remote update` cron |
| **CI/CD auto-escalado**: clon rapido desde mirror local | Mirror en NAS/red local |

```bash
# Migracion completa de GitHub a GitLab
git clone --mirror https://github.com/usuario/proyecto.git
cd proyecto.git
git remote add gitlab https://gitlab.com/empresa/proyecto.git
git push --mirror gitlab
```

---

## 4.14 Inspeccionando Remotos

### 4.14.1 `git ls-remote`

Este comando lista las referencias de un repositorio remoto sin necesidad de clonarlo:

```bash
# Listar todas las referencias (ramas, tags)
git ls-remote origin

# Listar solo ramas
git ls-remote --heads origin

# Listar solo tags
git ls-remote --tags origin

# Buscar una rama especifica
git ls-remote origin refs/heads/feature-x

# Desde cualquier URL (sin remoto configurado)
git ls-remote https://github.com/usuario/proyecto.git
```

Salida tipica:

```
a1b2c3d4e5f6...  refs/heads/main
b2c3d4e5f6a7...  refs/heads/develop
c3d4e5f6a7b8...  refs/tags/v1.0.0
d4e5f6a7b8c9...  refs/tags/v1.1.0
```

### 4.14.2 Comparar Ramas Locales y Remotas

```bash
# Ver ramas locales y su tracking
git branch -vv

# Ver ramas remotas
git branch -r

# Ver todas (locales + remotas)
git branch -a

# Commits locales no empujados
git log origin/main..HEAD --oneline

# Commits remotos no integrados
git log HEAD..origin/main --oneline

# Diferencia detallada
git diff origin/main
```

### 4.14.3 Verificar Estado de Sincronizacion

```bash
# Script rapido para ver estado de todas las ramas locales
for branch in $(git branch --format='%(refname:short)'); do
  upstream=$(git rev-parse --abbrev-ref "$branch@{upstream}" 2>/dev/null)
  if [ -n "$upstream" ]; then
    ahead=$(git rev-list --count "$upstream..$branch")
    behind=$(git rev-list --count "$branch..$upstream")
    echo "$branch → $upstream (ahead $ahead, behind $behind)"
  fi
done
```

---

## Resumen del Capitulo 4

- Un **remoto** es una referencia a un repositorio Git en otra ubicacion; `origin` es el nombre por defecto.
- `git remote` permite agregar, renombrar, eliminar e inspeccionar conexiones remotas.
- `git fetch` descarga objetos sin modificar el arbol de trabajo; es siempre seguro.
- `git pull` combina `fetch` con `merge` o `rebase`; `--rebase` mantiene un historial lineal.
- `git push` envia cambios al remoto; usa `--force-with-lease` en lugar de `--force`. `push.autoSetupRemote` (Git 2.37+) configura tracking automatico.
- Los **Pull Requests** son el mecanismo estandar para revision de codigo; existen draft PRs, plantillas y stacked PRs para flujos mas complejos.
- Los **Forks** permiten contribuir a proyectos sin permisos de escritura; `upstream` mantiene la sincronizacion.
- **HTTPS** usa tokens/credential helpers; **SSH** usa llaves y `ssh-agent` para autenticacion sin friccion.
- **Refspecs** (`+refs/heads/*:refs/remotes/origin/*`) definen el mapeo de referencias entre repositorios.
- **`git bundle`** permite transferir repositorios offline (air-gapped, USB) como archivos unicos.
- **Mirror clones** (`--mirror`) replican todas las referencias para backups, migraciones o cache local.
- `git ls-remote` permite inspeccionar un remoto sin clonarlo.
- La sincronizacion diaria eficiente combina `fetch`, `rebase` y `push --force-with-lease`.

## Ejercicios Propuestos

1. **Configuracion de remotos:** Crea un repositorio local. Agrega dos remotos distintos (pueden ser dos repositorios bare locales o dos repositorios en GitHub). Verifica con `git remote -v` y `git remote show`. Luego renombra uno de ellos y finalmente eliminalo.

2. **Simulacion de fork:** Clona un repositorio publico de GitHub. Agrega un segundo remoto llamado `upstream` que apunte al repositorio original. Simula el flujo de contribucion: crea una rama, haz un commit, y empuja a `origin`. Luego sincroniza `main` con `upstream`.

3. **Force push seguro:** Crea una rama, haz 3 commits, empuja. Luego haz un rebase interactivo local para modificar los mensajes de los commits. Intenta hacer `git push` normal y observa el rechazo. Resuelve con `git push --force-with-lease`. Luego simula la perdida de datos usando `--force` y explica la diferencia.

4. **Pull con rebase vs merge:** En un repositorio compartido con otra persona (o simulandolo con dos clones locales), haz cambios divergentes. Practica resolver el mismo escenario con `git pull` (merge) y con `git pull --rebase`. Compara el historial resultante con `git log --oneline --graph --all`.

5. **Inspeccion de remotos:** Usa `git ls-remote` para listar todas las ramas y tags de varios repositorios publicos populares. Escribe un script que compare las ramas locales con sus upstreams y muestre cuales estan desincronizadas (ahead/behind).

---

← [Capítulo anterior](03-ramas.md) | [Inicio](README.md) | [Capítulo siguiente →](05-deshacer-cambios.md)
