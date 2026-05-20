# Capítulo 9: Submódulos y Subtree

A medida que los proyectos crecen, es común necesitar incluir código de otros repositorios dentro de un proyecto principal. Git ofrece dos mecanismos principales para esto: submódulos (submodules) y subtrees. Ambos resuelven el problema de dependencias entre repositorios, pero con filosofías radicalmente distintas. Este capítulo explora cada herramienta en profundidad, sus comandos, flujos de trabajo, y cómo decidir cuál usar según el contexto.

---

## 9.1 ¿Qué Son los Submódulos?

Un submódulo es un repositorio Git anidado dentro de otro repositorio Git. El repositorio principal mantiene una referencia al commit exacto del submódulo, no a su contenido. Esto permite que cada repositorio evolucione independientemente mientras el principal controla qué versión exacta se usa.

```
mi-proyecto/                    (repositorio principal, remoto: github.com/equipo/mi-proyecto)
├── .git/
├── .gitmodules                 (configuración de submódulos)
├── src/
│   └── main.c
├── lib/                        (directorio donde residen submódulos)
│   ├── parser/                 → apunta a github.com/equipo/parser.git @ commit a1b2c3d
│   └── logger/                 → apunta a github.com/equipo/logger.git @ commit e4f5g6h
└── README.md

parser/ (repositorio independiente, remoto: github.com/equipo/parser)
├── .git/
├── src/
│   └── parser.c
└── README.md
```

El repositorio principal almacena:
- La URL del remoto del submódulo (en `.gitmodules`)
- El commit exacto al que apunta (como objeto en el árbol)
- La ruta local donde se ubica (en `.gitmodules`)

> **Concepto clave:** El repositorio principal NO contiene el código del submódulo; contiene un puntero a un commit específico del submódulo. Es como un marcador de posición que dice "aquí va el repositorio X en el commit Y".

---

## 9.2 Agregar un Submódulo

```bash
# Sintaxis básica
git submodule add <url-del-repositorio> <ruta-local>

# Ejemplo: agregar una librería compartida
git submodule add https://github.com/equipo/lib-parser.git lib/parser

# Agregar un submódulo apuntando a una rama específica
git submodule add -b develop https://github.com/equipo/lib-logger.git lib/logger
```

Al ejecutar `git submodule add`, Git:

1. Clona el repositorio en la ruta especificada
2. Crea o actualiza el archivo `.gitmodules`
3. Añade la referencia al submódulo al staging area
4. Registra el submódulo en `.git/config`

### El Archivo .gitmodules

```ini
[submodule "lib/parser"]
    path = lib/parser
    url = https://github.com/equipo/lib-parser.git
    branch = main

[submodule "lib/logger"]
    path = lib/logger
    url = git@github.com:equipo/lib-logger.git
    branch = develop
```

Este archivo se versiona y se comparte con el equipo. Define la configuración de todos los submódulos.

```bash
# Ver el estado de los submódulos
git submodule status

# Salida típica:
#  a1b2c3d4e5 lib/parser (v1.2.0)
#  e4f5g6h7i8 lib/logger (heads/develop)
# +f9g0h1i2j3 lib/utils   (el + indica que el submódulo tiene cambios no sincronizados)

# Ver configuración detallada
cat .gitmodules
git submodule
```

---

## 9.3 Clonar Repositorios con Submódulos

Clonar un repositorio que contiene submódulos requiere pasos adicionales, ya que `git clone` por defecto no inicializa ni descarga los submódulos.

### Método 1: Clonar y luego Inicializar

```bash
# Clonar el repositorio (los directorios de submódulos aparecen vacíos)
git clone https://github.com/equipo/mi-proyecto.git
cd mi-proyecto

# Inicializar la configuración local de submódulos
git submodule init

# Descargar el contenido de los submódulos
git submodule update

# Alternativa: un solo paso
git submodule update --init
```

### Método 2: Clonar con Inicialización Automática (Recomendado)

```bash
# Clonar e inicializar submódulos en un solo comando
git clone --recurse-submodules https://github.com/equipo/mi-proyecto.git

# Si ya clonaste y olvidaste --recurse-submodules:
git submodule update --init --recursive
```

> **Tip:** Configura `git config --global submodule.recurse true` para que comandos como `git pull`, `git push`, y `git checkout` operen también sobre submódulos automáticamente.

### Submódulos Anidados

Los submódulos pueden contener sus propios submódulos (anidación):

```bash
# Inicializar todos los submódulos recursivamente
git submodule update --init --recursive

# Actualizar submódulos a la rama remota configurada
git submodule update --remote

# Clonar con todos los submódulos, incluyendo los anidados
git clone --recurse-submodules <url>
```

> **Nota:** `git clone --remote-submodules` **no existe** como flag de clone. Para actualizar submódulos a sus ramas remotas despues de clonar, usa `git submodule update --remote` dentro del repositorio ya clonado.

---

## 9.4 Flujo de Trabajo Diario con Submódulos

### Actualizar Submódulos al Último Commit de su Rama

```bash
# Actualizar un submódulo específico a su último commit remoto
git submodule update --remote lib/parser

# Actualizar todos los submódulos
git submodule update --remote

# Actualizar y además hacer merge (si hay cambios locales en el submódulo)
git submodule update --remote --merge

# Actualizar y hacer rebase
git submodule update --remote --rebase
```

### Realizar Cambios en un Submódulo desde el Proyecto Principal

```bash
# Entrar al submódulo
cd lib/parser

# El submódulo está en detached HEAD por defecto
git checkout main

# Hacer cambios
echo "Nueva función" >> parser.c
git add parser.c
git commit -m "Añade función parse_json()"

# Subir los cambios al remoto del submódulo
git push origin main

# Volver al proyecto principal
cd ../..

# El proyecto principal detecta el cambio en el submódulo
git status
# modified:   lib/parser (new commits)

# Añadir y commitear la actualización de la referencia
git add lib/parser
git commit -m "Actualiza lib-parser a v1.3.0 con soporte JSON"
```

> **Advertencia:** El estado detached HEAD en submódulos es una fuente común de confusión. Si haces cambios en un submódulo sin crear una rama, esos commits pueden perderse. Siempre crea una rama antes de modificar un submódulo: `git checkout -b feature/nueva-funcion`.

---

## 9.5 Comandos Útiles para Submódulos

### git submodule foreach

Ejecuta un comando arbitrario en cada submódulo. Esencial para operaciones masivas.

```bash
# Ver la rama actual de cada submódulo
git submodule foreach 'git branch'

# Ver el estado de todos los submódulos
git submodule foreach 'git status --short'

# Actualizar todos los submódulos a su rama principal
git submodule foreach 'git checkout main && git pull origin main'

# Ejecutar tests en cada submódulo
git submodule foreach 'make test'

# Personalizar el mensaje por submódulo
git submodule foreach 'echo "Procesando $name en $path: $(git rev-parse HEAD)"'
```

### git submodule deinit

Desinicializa un submódulo (elimina su contenido del working tree pero mantiene la configuración).

```bash
# Desinicializar un submódulo
git submodule deinit lib/parser

# Eliminar el directorio de trabajo
rm -rf lib/parser

# Re-inicializar más tarde
git submodule update --init lib/parser
```

### git submodule sync

Sincroniza las URLs de submódulos cuando cambian en `.gitmodules`.

```bash
# Si .gitmodules cambió (por ej. la URL de un submódulo se actualizó)
git submodule sync

# Sincronizar solo un submódulo específico
git submodule sync lib/parser

# Después de sync, actualizar
git submodule update --init --recursive
```

### git submodule set-url (Git 2.22+)

Cambia la URL remota de un submódulo y actualiza `.gitmodules`:

```bash
# Cambiar URL de un submódulo (actualiza .gitmodules y .git/config)
git submodule set-url lib/parser https://github.com/nuevo-org/parser.git

# Cambiar a URL SSH
git submodule set-url lib/parser git@github.com:equipo/parser.git
```

### git submodule set-branch (Git 2.22+)

Configura la rama por defecto de un submódulo:

```bash
# Establecer que el submódulo siga la rama 'develop'
git submodule set-branch --branch develop lib/parser

# Cambiar a la rama 'main'
git submodule set-branch --branch main lib/logger

# Verificar configuracion
cat .gitmodules
```

### git submodule summary

Muestra un resumen de cambios pendientes entre el commit actual y el configurado:

```bash
# Resumen de cambios en submódulos
git submodule summary

# Salida tipica:
# * lib/parser a1b2c3d...e4f5g6h (2):
#   > Actualiza algoritmo de parsing
#   > Corrige memory leak en parser.c

# Resumen de un submódulo especifico
git submodule summary lib/parser

# Limitar cantidad de commits mostrados
git submodule summary -n 5
```

### git submodule absorbgitdirs (Git 2.12+)

Mueve los directorios `.git` de submódulos a `.git/modules/` del repositorio principal:

```bash
# Migrar submódulos antiguos (donde .git es un directorio completo)
# al formato moderno (donde .git es un archivo que apunta a .git/modules/)
git submodule absorbgitdirs
```

Esto es util al actualizar repositorios antiguos o al usar worktrees (los worktrees requieren este formato).



---

## 9.6 Eliminar un Submódulo

Eliminar un submódulo es un proceso de varios pasos, ya que Git registra referencias en múltiples lugares.

```bash
# Paso 1: Desinicializar el submódulo
git submodule deinit -f lib/parser

# Paso 2: Eliminar el directorio del working tree
rm -rf lib/parser

# Paso 3: Eliminar la referencia del árbol de Git
git rm -f lib/parser

# Paso 4: Limpiar el directorio de módulos interno
rm -rf .git/modules/lib/parser

# Paso 5: (Opcional) Eliminar la sección de .gitmodules
# Editar .gitmodules y eliminar el bloque [submodule "lib/parser"]

# Paso 6: Confirmar
git add .gitmodules
git commit -m "Elimina submódulo lib/parser"
```

En versiones recientes de Git (2.34+), `git submodule deinit` es más inteligente, pero verificar manualmente evita artefactos residuales.

```ascii
ANTES DE ELIMINAR UN SUBMÓDULO:
══════════════════════════════════════════════
▸ Verifica que ningún otro submódulo dependa de él
▸ Asegúrate de que el equipo está de acuerdo
▸ Los submódulos eliminados no afectan clones antiguos
▸ El historial del repositorio principal aún contiene
  referencias a los commits antiguos
```

---

## 9.7 Git Subtree: Una Alternativa a Submódulos

Mientras que los submódulos mantienen referencias externas, el mecanismo de subtree **copia el historial completo** de otro repositorio dentro del repositorio principal, fusionándolo como un subdirectorio.

### Submódulos vs Subtree: Comparativa Conceptual

```
SUBMÓDULOS                         SUBTREE
─────────                          ───────
Repositorio A                      Repositorio A
├── .git/                          ├── .git/
│   └── modules/lib/               │   └── (todo en una sola historia)
├── lib/ -> puntero a repo B       ├── lib/
└── src/                           │   ├── (código de B copiado aquí)
                                   │   └── (historial de B fusionado en A)
Repositorio B (independiente)      └── src/
├── .git/
└── src/
```

| Característica | Submódulos | Subtree |
|----------------|------------|---------|
| Código en el repo | Referencia externa | Código copiado |
| Historial | Separado | Fusionado |
| Actualizar | `git submodule update --remote` | `git subtree pull` |
| Clonar | Necesita `--recurse-submodules` | Clon normal, todo incluido |
| Contribuir upstream | Requiere push separado | `git subtree push` |
| Curva de aprendizaje | Alta | Media |
| Adecuado para | Dependencias externas con ciclo propio | Código que forma parte integral |

### Configurar `diff.submodule` para Ver Cambios en Submódulos

Por defecto `git diff` muestra solo el hash del submódulo. Puedes configurarlo para mostrar un resumen de los commits:

```bash
# Mostrar diff de commits del submódulo en lugar de solo el hash
git config diff.submodule log

# Ahora git diff muestra:
# Submodule lib/parser a1b2c3d..e4f5g6h:
#   > Actualiza algoritmo de parsing
#   > Corrige memory leak en parser.c

# Opcion 'short': muestra solo el primer commit del rango
git config diff.submodule short
```

O sobrescribir por invocación: `git diff --submodule=log`.

---

## 9.8 Trabajar con Git Subtree

### Agregar un Subtree

```bash
# Sintaxis básica
git subtree add --prefix=<ruta-local> <url-remoto> <rama> --squash

# Ejemplo: agregar una librería externa
git subtree add --prefix=lib/parser https://github.com/equipo/lib-parser.git main --squash

# Sin --squash: todo el historial del repositorio externo se copia
git subtree add --prefix=lib/parser https://github.com/equipo/lib-parser.git main
```

La opción `--squash` condensa todo el historial del repositorio externo en un único commit en el repositorio principal. Esto mantiene el historial principal limpio, a costa de perder granularidad.

```
Con --squash:

Repo principal:  A---B---C---D---S
                                 └── commit squash con todo parser

Sin --squash:

Repo principal:  A---B---C---D---p1---p2---p3---p4
                                 └─── historial completo de parser
```

### Actualizar un Subtree (Pull)

```bash
# Traer cambios del repositorio externo
git subtree pull --prefix=lib/parser https://github.com/equipo/lib-parser.git main --squash

# Si configuraste un remoto para el subtree
git remote add parser-upstream https://github.com/equipo/lib-parser.git
git subtree pull --prefix=lib/parser parser-upstream main --squash
```

### Enviar Cambios al Repositorio Externo (Push)

Una de las ventajas del subtree es que puedes modificar el código localmente y luego enviar esos cambios al repositorio de origen.

```bash
# Enviar cambios locales del subtree al repositorio externo
git subtree push --prefix=lib/parser parser-upstream main

# Con squash (crea un único commit en el remoto con todos los cambios locales)
git subtree push --prefix=lib/parser parser-upstream main --squash
```

### Dividir un Subdirectorio en un Repositorio Independiente (Split)

```bash
# Extraer el historial de un subdirectorio a una rama independiente
git subtree split --prefix=lib/parser -b parser-extracted

# Ahora parser-extracted contiene solo los commits relacionados con lib/parser
git log --oneline parser-extracted

# Push a un nuevo repositorio
git push https://github.com/equipo/nuevo-parser.git parser-extracted:main
```

**`--rejoin` en split subsiguientes:** Despues de hacer `split` y mergear cambios de vuelta, Git pierde el rastreo del historial dividido. `--rejoin` crea un merge commit que une la rama extraida de vuelta al historial principal, permitiendo que futuros `split` sean incrementales (solo exportan commits nuevos):

```bash
# Primer split (completo)
git subtree split --prefix=lib/parser -b parser-extracted

# Despues de mas trabajo, split incremental:
git subtree split --prefix=lib/parser --rejoin -b parser-extracted
# Solo exporta los commits NUEVOS, no toda la historia
```

---

## 9.9 Flujo de Trabajo con Subtree: Ejemplo Completo

```bash
# 1. Crear proyecto principal
mkdir mi-servidor && cd mi-servidor && git init

# 2. Agregar una librería de autenticación como subtree
git subtree add --prefix=vendor/libauth \
  https://github.com/equipo/libauth.git main --squash

# 3. Desarrollar el proyecto normalmente
echo "require './vendor/libauth/auth.rb'" > server.rb
git add server.rb && git commit -m "Usa libauth en server.rb"

# 4. Modificar la librería localmente
echo "def new_method; end" >> vendor/libauth/auth.rb
git add vendor/libauth/auth.rb
git commit -m "Añade new_method a libauth"

# 5. Enviar el cambio a la librería original
git subtree push --prefix=vendor/libauth \
  https://github.com/equipo/libauth.git main

# 6. Tiempo después, actualizar la librería
git subtree pull --prefix=vendor/libauth \
  https://github.com/equipo/libauth.git main --squash
```

> **Tip:** Usa `--squash` consistentemente tanto en `add` como en `pull`. Mezclar operaciones con y sin squash puede generar conflictos de historial difíciles de resolver.

---

## 9.10 Comparativa Completa: Submódulos vs Subtree vs Dependencias

La siguiente tabla amplía la comparativa para incluir alternativas modernas basadas en gestores de paquetes:

| Criterio | Submódulos | Subtree | Gestor de Paquetes (npm, pip, cargo) |
|----------|------------|---------|--------------------------------------|
| **Acoplamiento** | Bajo | Alto | Bajo |
| **Versionado** | Por commit exacto | Fusionado en historial | Por versión SemVer |
| **Tamaño del repo** | Pequeño (solo referencias) | Grande (todo el código) | Pequeño (archivo lock) |
| **Complejidad CI/CD** | Alta (necesita clonar con submódulos) | Baja (todo incluido) | Media |
| **Contribuir upstream** | Sí, con flujo separado | Sí, con subtree push | No directamente |
| **Ideal para** | Código que cambia junto al principal | Vendorización de dependencias | Dependencias externas |
| **Curva para el equipo** | Alta | Media | Baja |

### ¿Cuándo Usar Cada Estrategia?

```
DECISIÓN: ¿La dependencia se modifica junto con el proyecto principal?

SÍ ──────────────────────────────────────── NO
 │                                           │
 ▼                                           ▼
¿El equipo necesita contribuir           ¿Es una dependencia
al upstream frecuentemente?              estándar del ecosistema?

SÍ ────── NO ──────                    SÍ ────────── NO
 │         │                             │             │
 ▼         ▼                             ▼             ▼
SUBTREE   SUBMÓDULOS              GESTOR DE        SUBMÓDULOS
          (referencia            PAQUETES         (cuevas de
           fija, bajo            (npm, pip,         librería
           acoplamiento)          cargo, etc.)       interna)
```

---

## 9.11 Monorepos: La Alternativa Arquitectónica

Antes de decidir entre submódulos y subtree, considera si realmente necesitas múltiples repositorios. Un **monorepo** (repositorio monolítico único) es una alternativa cada vez más popular.

### Ventajas del Monorepo

- **Código atómico**: Un commit puede abarcar cambios en toda la codebase
- **Refactorización masiva**: Refactorizar APIs es trivial con herramientas como `codemod`
- **CI/CD unificado**: Una sola pipeline de integración
- **Sin problemas de versionado entre componentes**: Todo usa HEAD

### Desventajas del Monorepo

- **Escala**: Repositorios muy grandes requieren herramientas especializadas (Git LFS, sparse checkout, Scalar)
- **Herramientas**: Necesitas build systems que soporten monorepos (Bazel, Nx, Turborepo, Lerna)
- **Control de acceso**: Difícil limitar permisos por componente
- **Acoplamiento**: Puede llevar a dependencias circulares si no hay disciplina

### Herramientas para Monorepos

```bash
# Turborepo (JavaScript/TypeScript)
npx create-turbo@latest mi-monorepo

# Nx (multi-lenguaje)
npx create-nx-workspace@latest mi-monorepo

# Bazel (multi-lenguaje, gran escala)
# Archivo WORKSPACE y BUILD por componente

# Lerna (JavaScript, clásico - DEPRECADO)
# NOTA: Lerna dejó de recibir mantenimiento activo en 2022.
# El equipo de Nrwl absorbió Lerna y recomienda migrar a Nx.
# Para proyectos nuevos, usa Turborepo o Nx en lugar de Lerna.
npx lerna init  # solo para proyectos existentes que ya lo usan
```

---

## 9.12 Submódulos y Worktrees

### Compatibilidad

Los submódulos y worktrees pueden coexistir, pero con consideraciones:

```bash
# Crear worktree en un repo con submódulos
git worktree add ../proyecto-feature feature/nueva-ui

# Los submódulos NO se inicializan automaticamente en el nuevo worktree.
# Debes inicializarlos manualmente:
cd ../proyecto-feature
git submodule update --init --recursive
```

### Worktree + Submódulo: Misma Rama, Distinto Worktree

Cada worktree puede tener el submódulo en un commit distinto:

```bash
# Worktree A: submódulo en v1.0.0
cd ../proyecto-main
cd lib/parser && git checkout v1.0.0 && cd ../..
git add lib/parser && git commit -m "Fijar parser a v1.0.0"

# Worktree B: submódulo en v2.0.0 (develop)
cd ../proyecto-feature
cd lib/parser && git checkout main && git pull && cd ../..
git add lib/parser && git commit -m "Actualizar parser a v2.0.0"
```

Cada worktree gestiona su propio checkout del submódulo de forma independiente.

### Precaución: Submódulos en Worktrees Antiguos

Worktrees creados con Git < 2.12 almacenan el `.git` del submódulo como directorios reales, no como archivos que apuntan a `.git/modules/`. Si encuentras problemas:

```bash
# Migrar al formato moderno
git submodule absorbgitdirs
```

---

## 9.13 Casos de Uso Prácticos

### Caso 1: Librería Compartida entre Múltiples Proyectos

**Contexto:** La empresa mantiene `lib-core` (autenticación, logging, utilidades) usada en 3 microservicios.

**Solución con submódulos:**
```bash
# Cada microservicio agrega lib-core como submódulo
git submodule add git@github.com:empresa/lib-core.git vendor/core
git config -f .gitmodules submodule.vendor/core.branch main
```

**Ventaja:** Cada microservicio puede fijar una versión exacta de `lib-core`. Las actualizaciones son explícitas y controladas.

### Caso 2: Tema de Documentación Compartido

**Contexto:** Documentación en docs/ con un tema Hugo/Jekyll mantenido centralizadamente.

**Solución con submódulos:**
```bash
# Agregar tema de documentación
git submodule add https://github.com/empresa/tema-docs.git themes/tema-empresa

# CI/CD clona con submódulos para buildear la documentación
git clone --recurse-submodules <repo-docs>
hugo --theme themes/tema-empresa
```

### Caso 3: Vendorización de una Dependencia Modificada

**Contexto:** Necesitas usar una versión modificada de una librería open-source. Los cambios no son aceptados upstream pero son críticos para tu proyecto.

**Solución con subtree:**
```bash
# Forkear la librería
# Agregar como subtree
git subtree add --prefix=vendor/lib-external \
  git@github.com:usuario/lib-external-fork.git main --squash

# Modificar localmente
vim vendor/lib-external/src/core.c
git add vendor/lib-external
git commit -m "Parche: corrige race condition en core.c"

# Periódicamente, actualizar desde upstream
git subtree pull --prefix=vendor/lib-external \
  https://github.com/original/lib-external.git main --squash
```

### Caso 4: Extraer un Componente a su Propio Repositorio

```bash
# El componente estaba en lib/auth y ahora será su propio proyecto
git subtree split --prefix=lib/auth -b auth-extracted

# Crear nuevo repositorio y subir la rama extraída
gh repo create empresa/lib-auth --private --source=. --push auth-extracted:main

# Reemplazar el código local con un submódulo (opcional)
git rm -r lib/auth
git commit -m "Extrae lib/auth a repositorio independiente"
git submodule add git@github.com:empresa/lib-auth.git lib/auth
```

---

## 9.14 Problemas Comunes y Soluciones

### Problema 1: Submódulo en Estado "detached HEAD"

```bash
# Síntoma: cd lib/parser && git status muestra detached HEAD

# Solución: Crear o cambiar a una rama
cd lib/parser
git checkout main        # o la rama que corresponda

# Para evitar: configurar la rama por defecto
git config -f .gitmodules submodule.lib/parser.branch main
```

### Problema 2: "fatal: reference is not a tree" al Actualizar Submódulos

```bash
# Causa: El commit referenciado no está disponible localmente
# Solución: Hacer fetch del submódulo primero
git submodule foreach git fetch
git submodule update --remote
```

### Problema 3: Conflicts al Hacer Subtree Pull con --squash

```bash
# El merge squash crea un commit que puede no conectar bien con el historial
# Solución: Usar estrategia de merge subtree
git pull -s subtree parser-upstream main

# Alternativa: Usar --no-squash consistentemente
git subtree pull --prefix=vendor/lib parser-upstream main
```

### Problema 4: Submódulo con Cambios Locales No Deseados

```bash
# Verificar estado
git submodule status

# Si aparece + al inicio, hay cambios locales no commiteados
# +b2c3d4e5 lib/parser (v1.2.0)

# Restaurar al commit esperado
git submodule update --init --force lib/parser

# Si quieres descartar todos los cambios locales en submódulos
git submodule foreach git reset --hard HEAD
git submodule foreach git clean -fd
```

---

## 9.15 Automatización y CI/CD

### Submódulos en GitHub Actions

```yaml
name: CI with Submodules

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
          token: ${{ secrets.SUBMODULE_PAT }}

      - name: Check submodule status
        run: git submodule status

      - name: Build
        run: make build
```

> **Advertencia:** Si los submódulos están en repositorios privados, necesitas un Personal Access Token (PAT) con los permisos adecuados. Configúralo como secreto en tu CI/CD y pásalo a `actions/checkout`.

### Submódulos en GitLab CI

```yaml
variables:
  GIT_SUBMODULE_STRATEGY: recursive
  # Para repositorios privados:
  GIT_SUBMODULE_DEPTH: 1
  GIT_SUBMODULE_FORCE_HTTPS: "true"

build:
  script:
    - git submodule status
    - make build
```

### Verificación Automática de que los Submódulos Están Actualizados

```bash
#!/bin/bash
# Script: check-submodules.sh
# Verifica que todos los submódulos apuntan a commits existentes en sus remotos

set -e

git submodule foreach --recursive '
  REMOTE_URL=$(git config --get remote.origin.url)
  HEAD_SHA=$(git rev-parse HEAD)

  echo "Verificando $name en $REMOTE_URL..."

  if ! git ls-remote --exit-code "$REMOTE_URL" "$HEAD_SHA" > /dev/null 2>&1; then
    echo "ERROR: El submódulo $name apunta a un commit ($HEAD_SHA) que no existe en $REMOTE_URL"
    exit 1
  fi

  echo "  $name: OK"
'

echo "Todos los submódulos verificados correctamente."
```

---

## 9.16 Buenas Prácticas

1. **Documenta la política de submódulos/subtree** en el README o CONTRIBUTING.md del proyecto. Los nuevos miembros del equipo deben saber cómo trabajar con estas herramientas.

2. **Usa submódulos para dependencias con ciclo de desarrollo independiente.** Si el equipo de la librería es distinto al equipo del proyecto principal, los submódulos son más apropiados.

3. **Usa subtree para vendorización.** Cuando necesitas incluir código externo que puedes llegar a modificar y mantener localmente, subtree evita la complejidad de los submódulos.

4. **Nunca modifiques código de un submódulo sin crear una rama.** El detached HEAD es la causa #1 de pérdida de trabajo en submódulos.

5. **Automatiza en CI/CD.** Configura la inicialización de submódulos en tus pipelines y añade verificaciones de consistencia.

6. **Considera el monorepo primero.** Antes de introducir la complejidad de múltiples repositorios, evalúa si un monorepo con herramientas modernas cubre tus necesidades.

7. **Mantén los submódulos actualizados regularmente.** Un submódulo que apunta a un commit de hace 6 meses acumula deuda técnica rápidamente.

---

## Resumen del Capítulo 9

- Los **submódulos** permiten incluir repositorios Git dentro de otros, manteniendo una referencia al commit exacto. Requieren comandos específicos para clonar (`--recurse-submodules`), inicializar (`init`) y actualizar (`update`).
- El archivo `.gitmodules` almacena la configuración de submódulos (URL, rama, ruta) y se versiona con el proyecto.
- `git submodule foreach` ejecuta comandos en todos los submódulos simultáneamente.
- **Git subtree** copia el historial de un repositorio externo dentro del principal, fusionándolo como un subdirectorio. Los comandos principales son `add`, `pull`, `push` y `split`.
- La opción `--squash` en subtree condensa el historial externo en un único commit.
- Los **monorepos** son una alternativa arquitectónica que evita la complejidad de múltiples repositorios, usando herramientas como Nx, Turborepo o Bazel.

---

## Ejercicios Propuestos

1. **Crear y gestionar submódulos:** Crea dos repositorios: `lib-calc` (con una función `sumar`) y `app-principal`. Agrega `lib-calc` como submódulo de `app-principal`. Modifica `lib-calc` para añadir una función `restar`, haz commit y push en el submódulo, y luego actualiza la referencia en `app-principal`. Verifica con `git submodule status`.

2. **Simular un clon con submódulos:** Clona `app-principal` (del ejercicio 1) en otro directorio usando `git clone` normal. Observa que los submódulos están vacíos. Luego usa `--recurse-submodules` para clonar correctamente y compara los resultados.

3. **Extraer un componente con subtree split:** En un repositorio existente con múltiples subdirectorios (por ejemplo, `src/`, `lib/`, `docs/`), extrae el historial del subdirectorio `lib/` a una rama independiente usando `git subtree split`. Crea un nuevo repositorio y sube esa rama para crear un proyecto independiente.

4. **Vendorizar una dependencia con subtree:** Crea un proyecto `mi-app` y "vendoriza" una librería externa (puedes simularla con un repositorio público pequeño) usando `git subtree add --squash`. Realiza modificaciones locales en la librería y envíalas de vuelta al repositorio original con `git subtree push`.

5. **Comparativa y decisión:** Investiga un proyecto open-source real que use submódulos (ej. `.vim` bundles, o themes de Hugo). Analiza el archivo `.gitmodules` y explica por qué los mantenedores eligieron submódulos en lugar de subtree o gestores de paquetes. Escribe un breve informe de 200 palabras justificando la decisión.
