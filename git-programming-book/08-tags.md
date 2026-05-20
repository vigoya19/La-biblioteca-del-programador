# Capítulo 8: Tags y Releases

Los tags (etiquetas) son uno de los mecanismos más infravalorados de Git. Mientras que las ramas se mueven constantemente, los tags permanecen anclados a un punto específico en la historia, convirtiéndolos en la herramienta ideal para marcar versiones, hitos y releases de software. Este capítulo cubre desde los tipos básicos de tags hasta estrategias de versionado semántico y publicación de releases en plataformas como GitHub y GitLab.

---

## 8.1 ¿Qué Son los Tags?

Un tag en Git es una referencia inmutable que apunta a un commit concreto. A diferencia de las ramas, que avanzan con cada nuevo commit, un tag permanece fijo, proporcionando un marcador permanente en la historia del repositorio.

```
main:     A---B---C---D---E
                      ^
                    v1.0.0 (tag)
```

Los tags se utilizan típicamente para:

- Marcar versiones de software (v1.0.0, v2.3.1)
- Identificar hitos importantes del proyecto
- Crear puntos de referencia estables para despliegues
- Facilitar la navegación por la historia del repositorio

> **Tip:** A diferencia de muchas operaciones en Git que son fácilmente reversibles, los tags están diseñados para ser permanentes. Aunque técnicamente se pueden eliminar o mover, la convención del ecosistema Git dicta que los tags no deben modificarse una vez publicados.

### Tipos de Tags

Git ofrece dos tipos de tags, cada uno con características distintas:

| Característica | Tag Ligero | Tag Anotado |
|----------------|------------|-------------|
| Almacenamiento | Puntero directo al commit | Objeto completo en la base de datos |
| Metadata | Ninguna | Autor, fecha, email, mensaje |
| Firma GPG | No | Sí |
| Uso recomendado | Marcas temporales, locales | Releases públicos, versionado |
| `git describe` | No se incluye | Sí se incluye |

---

## 8.2 Tags Ligeros (Lightweight Tags)

Un tag ligero es esencialmente un puntero fijo a un commit, almacenado como un archivo en `.git/refs/tags/`. No contiene metadata adicional ni se almacena como objeto independiente en la base de datos de Git.

### Crear un Tag Ligero

```bash
# Crear un tag ligero en el commit actual
git tag v1.0.0-ligero

# Crear un tag ligero sobre un commit específico
git tag v0.9.0-ligero a1b2c3d
```

### Verificar un Tag Ligero

```bash
# Mostrar el tag
git show v1.0.0-ligero

# Salida típica de un tag ligero:
# commit a1b2c3d4e5f6...
# Author: María García <maria@example.com>
# Date:   Mon May 19 10:00:00 2026 +0200
#
#     Mensaje del commit
```

Observa que `git show` sobre un tag ligero muestra directamente el commit al que apunta, sin información adicional del tag. Esto es útil para marcas temporales internas, como "antes-de-refactor", pero no es adecuado para releases públicos donde se necesita metadata.

---

## 8.3 Tags Anotados (Annotated Tags)

Los tags anotados se almacenan como objetos completos en la base de datos de Git. Contienen:

- Nombre del tag
- Mensaje descriptivo
- Autor y fecha de creación
- Checksum SHA-1 del commit al que apuntan
- Opcionalmente, una firma GPG

```
BASE DE DATOS DE OBJETOS

Tag Object (v2.0.0)
├── Object type: tag
├── Tag name: v2.0.0
├── Tagger: Ana López <ana@example.com> 1716100000 +0200
├── Message: Release 2.0.0 - Nueva API de autenticación
└── Points to: Commit Object (b2c3d4e)
                    │
                    ▼
            Commit Object (b2c3d4e)
            ├── Tree: f5a6b7c
            ├── Parent: a1b2c3d
            ├── Author: Ana López
            └── Message: Implementa nueva API de autenticación
```

### Crear un Tag Anotado

```bash
# Crear tag anotado con mensaje
git tag -a v2.0.0 -m "Release 2.0.0: Nueva API de autenticación"

# Crear tag anotado con editor de texto (mensaje multilínea)
git tag -a v2.0.0

# Crear tag anotado sobre un commit pasado
git tag -a v1.5.0 -m "Release 1.5.0" a1b2c3d
```

### Ver un Tag Anotado

```bash
# Ver información completa del tag
git show v2.0.0

# Salida:
# tag v2.0.0
# Tagger: Ana López <ana@example.com>
# Date:   Mon May 19 10:30:00 2026 +0200
#
# Release 2.0.0: Nueva API de autenticación
#
# commit b2c3d4e5f6a7...
# Author: Ana López <ana@example.com>
# Date:   Mon May 19 10:25:00 2026 +0200
#
#     Implementa nueva API de autenticación
```

Observa cómo `git show` en un tag anotado muestra primero la información del tag (tagger, fecha, mensaje) y después el commit asociado.

---

## 8.4 Gestionar Tags: Listar, Filtrar y Buscar

### Listar Tags

```bash
# Listar todos los tags
git tag

# Listar tags con información adicional (ordenados por versión)
git tag -l
git tag --list           # sinonimo moderno de -l

# Listar tags con patrón de filtro
git tag -l "v2.*"
git tag --list "v2.*"

# Listar tags que contienen un commit específico
git tag --contains a1b2c3d

# Listar tags que NO contienen un commit (util para releases pendientes)
git tag --no-contains a1b2c3d

# Listar tags alcanzables desde una rama (merged)
git tag --merged main

# Listar tags NO alcanzables desde main (no merged)
git tag --no-merged main

# Listar tags con múltiples líneas de información
git tag -n
git tag -n3        # Muestra hasta 3 líneas del mensaje de cada tag anotado

# Ordenar por versión (útil con SemVer)
git tag --sort=version:refname

# Ordenar por fecha de creación
git tag --sort=creatordate
```

### Filtrar por Patrones

```bash
# Filtrar por prefijo de versión
git tag -l "v2.1.*"

# Filtrar por versión mayor específica
git tag -l "v[13].*"    # Muestra tags v1.* y v3.*

# Excluir patrones (con grep, ya que git tag no soporta negación nativa)
git tag -l "v*" | grep -v "rc"
```

### Formato Personalizado de Tags

Git 2.23+ permite formatear la salida de `git tag` con especificadores de `--format`, similares a los de `git log`:

```bash
# Listar tags con nombre y fecha de creacion
git tag --format='%(refname:short) %(creatordate:short)'
# v2.0.0 2026-05-19
# v1.7.3 2026-04-15

# Tags con autor, fecha y mensaje (primera linea)
git tag --format='%(refname:short) | %(taggername) | %(creatordate:iso) | %(subject)'

# Solo tags anotados con su hash de objeto
git tag --format='%(refname:short) %(objectname) %(*objectname)'

# Ordenar por fecha del commit (no del tag)
git tag --sort='*creatordate' --format='%(refname:short) %(*creatordate:short) %(subject)'

# Tags ligeros no tienen metadata de tag, usa * para acceder al commit referenciado:
# %(creatordate)     = fecha del objeto tag (solo tags anotados)
# %(*creatordate)    = fecha del commit al que apunta (ambos tipos)
```



> **Tip:** Los patrones en `git tag -l` usan globbing de shell (`*`, `?`, `[abc]`), no expresiones regulares completas. Para filtros más complejos, combina `git tag` con `grep`.

### Buscar Tags por Commit

```bash
# Ver qué tags apuntan a un commit específico
git tag --points-at a1b2c3d

# Ver qué tags contienen un commit en su historia
git tag --contains a1b2c3d

# Combinado con log para ver tags y ramas
git log --oneline --decorate -10
```

---

## 8.5 Tags Retrospectivos

Puedes crear tags sobre commits pasados en cualquier momento. Esto es útil cuando olvidaste etiquetar una versión antigua o necesitas marcar un commit importante después del hecho.

```bash
# Ver el historial para identificar el commit
git log --oneline --graph -20

# Crear tag anotado sobre un commit pasado
git tag -a v0.5.0 -m "Release 0.5.0: Primera versión estable" f6a7b8c

# Crear tag ligero retrospectivo
git tag pre-refactor e3d2f1a

# Verificar que el tag apunta al commit correcto
git log --oneline --graph --decorate --all | head -30
```

```ascii
Consideraciones sobre tags retrospectivos:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ No alteran la historia existente
✓ Son seguros de crear en cualquier momento
✗ Pueden confundir si la fecha del tag es muy posterior al commit
✗ No pueden cambiar el orden de releases si ya se publicaron
```

---

## 8.6 Compartir Tags con Remotos

Por defecto, `git push` **no** transfiere tags al remoto. Debes enviarlos explícitamente.

```bash
# Subir un tag específico a origin
git push origin v2.0.0

# Subir todos los tags de una vez
git push --tags

# Subir tags anotados y ligeros (comportamiento por defecto)
git push origin --tags

# Subir solo tags anotados (recomendado para releases)
git push origin --follow-tags
```

> **Advertencia:** `git push --tags` envía todos los tags sin distinción, incluyendo tags ligeros temporales que quizás no deberían publicarse. Para releases, prefiere `git push origin <tag>` o configura `push.followTags = true` para que `git push` envíe automáticamente los tags anotados que sean alcanzables desde la rama.

### Configurar Follow Tags

```bash
# Configurar a nivel global
git config --global push.followTags true

# Configurar solo para este repositorio
git config push.followTags true
```

Con esta configuración, un simple `git push` enviará también los tags anotados que referencien commits que estás subiendo.

### Descargar Tags de Remotos

```bash
# Los tags se descargan automáticamente con fetch
git fetch

# Si un tag fue eliminado en el remoto, limpiar localmente
git fetch --prune --prune-tags

# Forzar actualización de tags locales
git fetch --tags --force
```

### Push Atomico con Tags

Cuando necesitas que tanto el commit como el tag lleguen juntos al remoto (o nada), usa `--atomic`:

```bash
# Push atomico: commit + tag llegan juntos, o ninguno llega
git push --atomic origin main v2.0.0

# Si el push de la rama falla, el tag tampoco se sube.
# Si el tag falla, la rama tampoco se sube.
# Esto evita tags huerfanos que apuntan a commits no publicados.
```

---

## 8.7 Eliminar Tags

### Eliminación Local

```bash
# Eliminar un tag local
git tag -d v1.0.0

# Eliminar múltiples tags locales
git tag -d v0.1.0 v0.2.0

# Eliminar por patrón (con shell scripting)
git tag -l "rc-*" | xargs git tag -d
```

### Eliminación Remota

```bash
# Sintaxis explícita (Git 1.7+)
git push origin --delete v1.0.0

# Sintaxis alternativa (push con prefijo vacío)
git push origin :refs/tags/v1.0.0

# Eliminar múltiples tags remotos
git push origin --delete v0.1.0 v0.2.0
```

> **Advertencia:** Eliminar un tag publicado es una mala práctica en el ecosistema Git. Los tags son contratos públicos: otros desarrolladores pueden haber basado trabajo en ellos. Si publicaste un tag erróneo, la convención recomienda crear uno nuevo (ej. v1.0.1) en lugar de eliminar o mover el existente.

---

## 8.8 Checkout de Tags y Estado Detached HEAD

Cuando haces checkout de un tag (en lugar de una rama), entras en estado `detached HEAD`. Esto significa que HEAD apunta directamente a un commit, no a una rama.

```bash
# Checkout de un tag (entra en detached HEAD)
git checkout v2.0.0

# Git advierte:
# You are in 'detached HEAD' state.
# Example:
#   git switch -c <new-branch-name>

# Explorar el código en ese tag
ls -la
cat VERSION

# Crear una rama desde el tag si necesitas hacer cambios
git switch -c hotfix-from-v2.0.0 v2.0.0

# Alternativa moderna con git switch
git switch --detach v2.0.0
```

```ascii
ESTADO DETACHED HEAD
═══════════════════════════════════════════

main:     A---B---C---D---E (HEAD -> main)
               \
                F---G (feat/new-ui)

Después de "git checkout v1.0.0":

main:     A---B---C---D---E
               ^
          HEAD (detached at v1.0.0)

Cualquier commit nuevo se crea sin rama.
Si cambias de rama, esos commits se pierden.
```

---

## 8.9 Versionado Semántico (SemVer 2.0.0)

El versionado semántico (`https://semver.org`) define un esquema estandarizado para asignar números de versión con significado. Una versión SemVer tiene el formato:

```
MAJOR.MINOR.PATCH

v2 . 1  . 5
 │   │    └── PATCH: Corrección de bugs compatible hacia atrás
 │   └─────── MINOR: Nueva funcionalidad compatible hacia atrás
 └─────────── MAJOR: Cambios incompatibles con la API anterior
```

### Reglas de SemVer 2.0.0

| Incremento | Cuándo usarlo | Ejemplo |
|------------|---------------|---------|
| **MAJOR** | Cambios que rompen la API existente | v1.7.3 → v2.0.0 |
| **MINOR** | Nueva funcionalidad compatible hacia atrás | v1.7.3 → v1.8.0 |
| **PATCH** | Correcciones de bugs compatibles | v1.7.3 → v1.7.4 |

### Pre-release y Build Metadata

```bash
# Pre-release: alpha, beta, rc (release candidate)
git tag -a v2.0.0-alpha.1 -m "Primera alpha de 2.0"
git tag -a v2.0.0-beta.3 -m "Tercera beta"
git tag -a v2.0.0-rc.2 -m "Release candidate 2"

# Build metadata (no se recomienda en tags de Git)
# v1.0.0+20130313144700
```

> **Tip:** Los sufijos de pre-release tienen precedencia: `alpha < beta < rc < versión final`. Así, `v2.0.0-alpha < v2.0.0-beta < v2.0.0-rc.1 < v2.0.0`.

### Automatizar SemVer con Git Describe

```bash
# git describe genera un identificador único basado en el tag más cercano
git describe --tags

# Salida: v1.7.3-15-gb2c3d4e
#          │      │  └── hash abreviado del commit
#          │      └─── commits desde el tag
#          └────────── tag más cercano
#          La "g" antes del hash significa "git" (prefijo estandar
#          para indicar que es un hash de commit de Git, no de otro VCS)

# Solo si estamos exactamente en un tag
git describe --exact-match

# Generar para scripts
git describe --tags --abbrev=0    # Solo el tag más cercano
git describe --tags --long        # Formato largo siempre
```

---

## 8.10 Firmar Tags con GPG

Firmar un tag con GPG permite verificar criptográficamente que el tag fue creado por una persona o entidad de confianza. Esto añade una capa de seguridad para los consumidores de tu software.

### Configuración Inicial

```bash
# Generar un par de claves GPG (si no tienes una)
gpg --full-generate-key

# Listar tus claves
gpg --list-secret-keys --keyid-format LONG

# Configurar Git para usar tu clave
git config --global user.signingkey B3A7C9E12345F67A

# Configurar Git (opcional) para firmar todos los tags por defecto
git config --global tag.gpgSign true
```

### Crear Tags Firmados

```bash
# Firmar un tag (con la clave configurada)
git tag -s v2.0.0 -m "Release 2.0.0 firmada"

# Firmar con una clave específica
git tag -u B3A7C9E12345F67A -s v2.0.0 -m "Release firmada"

# Firmar un tag anotado ya existente (sobrescribiendo)
git tag -s v2.0.0 -f -m "Release 2.0.0 firmada"
```

### Verificar Firmas

```bash
# Verificar la firma de un tag
git tag -v v2.0.0

# Salida exitosa:
# object b2c3d4e5f6a7... (commit)
# type commit
# tag v2.0.0
# tagger Ana López <ana@example.com> 1716100000 +0200
#
# Release 2.0.0
# gpg: Signature made Mon May 19 10:30:00 2026 CEST
# gpg:                using RSA key B3A7C9E12345F67A
# gpg: Good signature from "Ana López <ana@example.com>"
```

### Firmar Tags con SSH (Git 2.34+)

Ademas de GPG, Git 2.34+ soporta firmar tags con claves SSH, lo que elimina la necesidad de GPG:

```bash
# Configurar Git para usar SSH como formato de firma
git config --global gpg.format ssh

# Configurar la clave SSH de firma
git config --global user.signingkey ~/.ssh/id_ed25519.pub

# Si usas una clave con passphrase, configura el agente SSH
ssh-add ~/.ssh/id_ed25519

# Crear tag firmado con SSH
git tag -s v3.0.0 -m "Release 3.0.0 firmada con SSH"

# Verificar firma SSH
git tag -v v3.0.0
# Good "git" signature for ana@example.com with ED25519 key SHA256:...

# Permitir que otros verifiquen tus firmas SSH
# Configurar allowed_signers file:
echo "$(git config user.email) namespaces=\"git\" $(cat ~/.ssh/id_ed25519.pub)" >> ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

**Ventajas de SSH sobre GPG:**
- Los desarrolladores ya tienen claves SSH para GitHub/GitLab.
- No requiere instalar ni configurar GPG.
- Las mismas claves sirven para autenticacion y firma.
- Integracion nativa con GitHub (desde 2022) y GitLab.



### Exportar Clave Pública GPG

Para que otros puedan verificar tus tags firmados con GPG, tu clave pública debe estar disponible:

```bash
# Exportar clave pública
gpg --armor --export B3A7C9E12345F67A > ana-lopez.gpg

# Subir a un keyserver
gpg --send-keys B3A7C9E12345F67A

# GitHub también puede alojar claves GPG en la configuración de usuario
```

---

## 8.11 Releases en GitHub

GitHub extiende el concepto de tags con el sistema de Releases, que añade documentación, assets binarios y notas de versión sobre los tags anotados.

### Crear un Release desde un Tag Existente

```bash
# Primero, crear y subir el tag
git tag -a v2.0.0 -m "Release 2.0.0: Nueva API de autenticación"
git push origin v2.0.0

# Crear release con GitHub CLI
gh release create v2.0.0 \
  --title "v2.0.0: Nueva API de autenticación" \
  --notes-file CHANGELOG.md \
  --draft \
  --prerelease
```

### Opciones de `gh release create`

| Opción | Descripción |
|--------|-------------|
| `--title` | Título del release |
| `--notes` | Notas de versión inline |
| `--notes-file` | Cargar notas desde un archivo (CHANGELOG.md) |
| `--draft` | Crear como borrador (no visible públicamente) |
| `--prerelease` | Marcar como pre-release |
| `--latest` | Marcar como última versión estable |
| `--discussion-category` | Vincular con categoría de Discussions |

### Generar Notas de Release Automáticamente

```bash
# GitHub puede generar notas automáticamente desde los PRs mergeados
gh release create v2.0.0 --generate-notes

# Esto incluye:
# - Lista de PRs mergeados desde el release anterior
# - Autores de cada PR
# - Enlaces a cada PR
# - Nuevos contribuidores (si los hay)
```

### Gestionar Releases Existentes con `gh`

```bash
# Listar releases del repositorio
gh release list
# v2.0.0  Latest  v2.0.0  2026-05-19T10:30:00Z
# v1.7.3          v1.7.3  2026-04-15T08:00:00Z

# Limitar cantidad
gh release list --limit 5

# Ver detalles de un release especifico
gh release view v2.0.0
# Muestra: titulo, notas, assets, fecha, autor, stats de descargas

# Ver release en formato JSON para scripting
gh release view v2.0.0 --json name,tagName,publishedAt,assets

# Descargar assets de un release
gh release download v2.0.0                    # todos los assets
gh release download v2.0.0 --pattern '*.dmg'  # solo macOS
gh release download v2.0.0 --dir ./binaries   # directorio destino

# Eliminar un release (y opcionalmente su tag)
gh release delete v2.0.0           # elimina el release, conserva el tag
gh release delete v2.0.0 --cleanup-tag  # elimina release + tag
# NOTA: Esto solo elimina el release de GitHub, no el tag local.
# Para eliminar tambien el tag local: git tag -d v2.0.0
```

### Adjuntar Assets Binarios

```bash
# Compilar binarios antes
make build-linux build-macos build-windows

# Adjuntar al release
gh release upload v2.0.0 \
  dist/app-linux.tar.gz \
  dist/app-macos.dmg \
  dist/app-windows.exe
```

### Release Notes Efectivas

```
# Release v2.0.0

## 🚀 Nuevas Funcionalidades
- Implementado sistema de autenticación OAuth 2.0 (#234)
- Añadido soporte para autenticación por huella biométrica (#245)
- Nueva API REST para gestión de sesiones (#251)

## 🐛 Correcciones
- Solucionado timezone en tokens JWT (#260)
- Corregido race condition en renovación de tokens (#267)

## ⚠️ Breaking Changes
- La API de login ahora requiere header `X-Client-ID`
- Deprecado endpoint /api/v1/auth (usar /api/v2/auth)

## 📦 Dependencias
- Actualizado crypto-library a v3.1.0
- Actualizado http-client a v2.5.0

## 🙏 Agradecimientos
Gracias a @dev_externo por el reporte de seguridad #255
```

---

## 8.12 Releases en GitLab

GitLab tiene un sistema de releases estrechamente integrado con su CI/CD.

### Crear Release desde CI/CD

```yaml
# .gitlab-ci.yml
release_job:
  stage: release
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  script:
    - echo "Creando release v${CI_COMMIT_TAG}"
  release:
    tag_name: $CI_COMMIT_TAG
    name: 'Release $CI_COMMIT_TAG'
    description: ./CHANGELOG.md
    assets:
      links:
        - name: 'linux-binary'
          url: 'https://artifacts.example.com/app-linux.tar.gz'
  only:
    - tags
```

### Con GitLab CLI (glab)

```bash
# Crear release directamente
glab release create v2.0.0 \
  --name "Release v2.0.0" \
  --notes "Nueva versión con API de autenticación" \
  --assets-links '[
    {"name": "app-linux.tar.gz", "url": "https://..."},
    {"name": "app-macos.dmg", "url": "https://..."}
  ]'
```

---

## 8.13 Estrategias de Versionado en Proyectos

La elección de la estrategia de versionado depende del tipo de proyecto, su madurez y su audiencia. No existe una solución única para todos los casos.

### Comparativa de Estrategias

| Estrategia | Esquema | Ideal para | Ejemplo |
|------------|---------|------------|---------|
| **SemVer** | MAJOR.MINOR.PATCH | Librerías, APIs, paquetes | v2.1.5 |
| **CalVer** | AAAA.MM.DD o YYYY.MINOR | Proyectos con ciclos temporales | 2026.05.19 |
| **0ver** | 0.MAJOR.MINOR | Proyectos en desarrollo inicial | v0.15.2 |
| **RomVer** | MAJOR.MINOR (sin patch) | Aplicaciones end-user | v15, v16 |
| **Commit-based** | Sin versiones explícitas | Rolling release, CI/CD | (usa hash de commit) |

### Estrategia de Ramas para Versionado

```
main ─────────────────────○────○────○ v2.1.0
                             \    \
release/2.0  ────○──○──○ v2.0.3   \
                     \              \
release/1.x  ────○ v1.7.5            \
                                       \
hotfix/2.0.4 ──────────────────────────○ v2.0.4
```

```bash
# Flujo típico de versionado con ramas de release
git checkout -b release/2.1.0 main
# Ajustar versión en archivo VERSION o package.json
echo "2.1.0" > VERSION
git add VERSION
git commit -m "Bump version to 2.1.0"
git tag -a v2.1.0 -m "Release 2.1.0"
git push origin release/2.1.0
git push origin v2.1.0
```

> **Tip:** Mantén un archivo `CHANGELOG.md` adherido al estándar [Keep a Changelog](https://keepachangelog.com). Esto asegura que los cambios entre versiones sean legibles tanto para humanos como para herramientas automáticas.

### Cuándo Crear el Tag

Existen dos enfoques principales:

**Enfoque 1: Tag en la rama de release (antes del merge)**
```
- Asegura que el tag existe antes de integrar
- Permite CI/CD sobre el tag sin esperar merge
```

**Enfoque 2: Tag en main después del merge (recomendado)**
```bash
# Merge de release a main
git checkout main
git merge release/2.1.0

# Crear tag en main
git tag -a v2.1.0 -m "Release 2.1.0"

# El tag está en main, no en release/
git push origin main --follow-tags
```
```
- El tag está en la rama principal, no en una efímera
- Más limpio: main contiene todos los tags de releases
- Mejor integración con herramientas que asumen tags en main
```

---

## 8.14 Operaciones Avanzadas con Tags

### Mover un Tag (NO Recomendado para Tags Públicos)

```bash
# Forzar la recreación de un tag en otro commit
git tag -a v1.0.0 -f -m "Release corregida" <nuevo-commit>

# Subir el tag forzado al remoto
git push origin v1.0.0 --force

# NOTA: Esto rompe el principio de inmutabilidad de tags
# Cualquiera que ya tenga el tag antiguo debe ejecutar:
git fetch --tags --force
```

### Comparar Contenido entre Tags

```bash
# Ver diferencias entre dos versiones
git diff v1.7.3..v2.0.0

# Ver archivos cambiados (solo nombres)
git diff --name-only v1.7.3..v2.0.0

# Estadísticas de cambios
git diff --stat v1.7.3..v2.0.0

# Listar commits entre tags
git log --oneline v1.7.3..v2.0.0

# Generar changelog entre tags
git log --oneline --no-merges v1.7.3..v2.0.0 > CHANGELOG_2.0.0.txt
```

### Tag como Base para Cherry-pick

```bash
# Identificar un commit desde un tag
git log v1.5.0 -1 --format="%H"

# Cherry-pick de un commit específico desde un tag
git cherry-pick v1.5.0~2    # Dos commits antes del tag

# Ver todos los commits en la historia de un tag
git log v2.0.0 --oneline --graph
```

### Verificar el Estado de Tags Locales vs Remotos

```bash
# Comparar tags locales con remotos
git ls-remote --tags origin | sort > /tmp/remotes.txt
git tag -l | sed 's/^/refs\/tags\//' | sort > /tmp/locals.txt
diff /tmp/locals.txt /tmp/remotes.txt
```

---

## 8.15 Buenas Prácticas con Tags

1. **Usa tags anotados para releases públicos.** Los tags ligeros son para uso interno y temporal.

2. **Sigue SemVer** si tu proyecto expone una API pública o es una librería.

3. **Nunca elimines o muevas tags publicados.** Si cometiste un error, publica una nueva versión que lo corrija.

4. **Firma tus tags** con GPG, especialmente en proyectos de seguridad o infraestructura crítica.

5. **Documenta cada release** con notas de versión claras, incluyendo breaking changes y migraciones necesarias.

6. **Automatiza** la creación de releases en CI/CD para reducir errores humanos.

7. **Usa `--follow-tags`** en lugar de `--tags` para evitar publicar tags temporales accidentalmente.

8. **Incluye el tag en el binario**, por ejemplo mediante `git describe --tags` en tiempo de compilación:

```bash
# En Makefile o script de build
VERSION=$(git describe --tags --always --dirty)
ldflags="-X main.Version=${VERSION}"
go build -ldflags "$ldflags" -o app .
```

---

## Resumen del Capítulo 8

- Los **tags** son referencias inmutables a commits, ideales para marcar versiones. Existen dos tipos: **ligeros** (punteros simples) y **anotados** (objetos completos con metadata, recomendados para releases).
- Los comandos principales son `git tag` (listar, crear, eliminar), `git show` (ver detalles) y `git push` con `--tags` o `--follow-tags` para compartir.
- Los tags **no se transfieren** con `git push` normal; deben enviarse explícitamente.
- El **versionado semántico (SemVer 2.0.0)** define el esquema MAJOR.MINOR.PATCH con reglas claras de compatibilidad.
- Las plataformas como **GitHub y GitLab** extienden los tags con sistemas de Releases que incluyen notas de versión, assets binarios y firmas GPG.
- Los tags **no deben modificarse ni eliminarse** una vez publicados.

---

## Ejercicios Propuestos

1. **Crear y explorar tags:** En un repositorio de prueba, crea un tag ligero y otro anotado. Usa `git show` en cada uno y explica las diferencias. Luego elimina ambos tags local y remotamente.

2. **Simular un flujo de release:** Crea un repositorio con varios commits. Crea una rama `release/1.0.0`, ajusta un archivo `VERSION`, mergea a `main` y crea el tag `v1.0.0`. Usa `git describe` para verificar que el tag se generó correctamente.

3. **Versionado SemVer:** Dada la siguiente secuencia: v1.0.0 → añadir funcionalidad compatible → corregir bug → cambio que rompe API → corregir otro bug. Asigna manualmente la versión SemVer correcta a cada paso y justifica tu decisión.

4. **Generar changelog entre tags:** En un repositorio con múltiples tags, genera un changelog listando todos los commits no-merge entre v1.0.0 y v2.0.0. Formatea la salida incluyendo autor y fecha de cada commit.

5. **Configurar GPG para firmar tags:** Genera un par de claves GPG, configúralo en Git, crea un tag firmado y verifica la firma. Exporta la clave pública y simula cómo otro desarrollador verificaría tu firma.
