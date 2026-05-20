# Git: La Guia Completa - De Principiante a Experto

## Indice General

1. [Capitulo 1: Introduccion a Git](01-introduccion.md)
   - Que es Git y por que existe
   - Historia de Git: del kernel de Linux al mundo
   - Sistemas de control de versiones: locales, centralizados y distribuidos
   - Instalacion en Windows, macOS y Linux
   - Configuracion inicial (git config)
   - Estados de Git: working directory, staging area, repository
   - Primeros pasos con la terminal

2. [Capitulo 2: Fundamentos de Git](02-fundamentos.md)
   - Inicializar un repositorio (git init)
   - Clonar repositorios (git clone)
   - El ciclo de vida de un archivo
   - Agregar cambios al staging (git add)
   - Crear commits (git commit)
   - Ver el historial (git log)
   - Ver el estado del repositorio (git status)
   - Ver diferencias (git diff)
   - Eliminar y mover archivos (git rm, git mv)
   - El archivo .gitignore

3. [Capitulo 3: Trabajando con Ramas](03-ramas.md)
   - Que es una rama y como funciona internamente
   - Crear, listar y eliminar ramas (git branch)
   - Cambiar entre ramas (git switch, git checkout)
   - Fusionar ramas (git merge)
   - Fast-forward vs three-way merge
   - Ramas remotas (remote tracking branches)
   - Estrategia de ramas en equipo
   - Borrar ramas locales y remotas

4. [Capitulo 4: Repositorios Remotos](04-remotos.md)
   - Que es un repositorio remoto
   - Agregar y gestionar remotos (git remote)
   - Enviar cambios (git push)
   - Obtener cambios (git fetch, git pull)
   - Pull request y code review
   - Forks y contribucion open source
   - Sincronizacion avanzada con upstream
   - Protocolos de red: HTTPS, SSH, Git protocol
   - Autenticacion y credenciales

5. [Capitulo 5: Deshaciendo Cambios](05-deshacer-cambios.md)
   - Deshacer cambios en el working directory (git restore, git checkout)
   - Deshacer cambios en el staging area (git restore --staged, git reset)
   - Revertir commits publicados (git revert)
   - Reescribir el ultimo commit (git commit --amend)
   - git reset: --soft, --mixed, --hard
   - git clean: eliminar archivos no rastreados
   - Recuperar commits "perdidos" con git reflog

6. [Capitulo 6: Rebase y Cherry-pick](06-rebase-cherry-pick.md)
   - Que es git rebase y como funciona
   - Rebase interactivo (git rebase -i): squash, fixup, reword, edit
   - Rebase vs merge: cuando usar cada uno
   - Resolver conflictos durante el rebase
   - Cherry-pick: aplicar commits especificos
   - Rebase sobre ramas remotas (git pull --rebase)
   - Peligros del rebase y la regla de oro

7. [Capitulo 7: Stashing y Trabajo Temporal](07-stash.md)
   - Guardar trabajo temporal (git stash)
   - Listar y recuperar stashes (git stash list, pop, apply)
   - Stash parcial y archivos especificos
   - Crear ramas desde un stash
   - Limpiar stashes (git stash drop, clear)
   - Worktrees: multiples ramas simultaneas (git worktree)

8. [Capitulo 8: Tags y Releases](08-tags.md)
   - Que son los tags en Git
   - Tags ligeros vs tags anotados
   - Crear, listar y eliminar tags
   - Compartir tags con el remoto
   - Versionado semantico (SemVer)
   - Releases en GitHub/GitLab
   - Firmar tags con GPG

9. [Capitulo 9: Submodulos y Subtree](09-submodulos.md)
   - Que son los submodulos y cuando usarlos
   - Agregar y clonar submodulos
   - Actualizar y sincronizar submodulos
   - Alternativa: git subtree
   - Monorepos: otra aproximacion
   - Comparativa: submodulos vs subtree vs monorepo

10. [Capitulo 10: Git Hooks](10-hooks.md)
    - Que son los hooks y donde se almacenan
    - Hooks del lado cliente: pre-commit, commit-msg, post-commit
    - Hooks del lado servidor: pre-receive, update, post-receive
    - Ejemplos practicos: linters, formateo, validacion de mensajes
    - Compartir hooks en el equipo
    - Alternativas modernas: Husky, Lefthook, pre-commit framework

11. [Capitulo 11: El Interior de Git](11-internals.md)
    - Modelo de objetos de Git: blobs, trees, commits, tags
    - El grafo de commits (DAG)
    - Hashing SHA-1 y direccionamiento por contenido
    - Comandos de plumbing vs porcelain
    - Explorando el directorio .git
    - Git references y HEAD
    - Packfiles y compresion delta
    - Garbage collection y git gc

12. [Capitulo 12: Estrategias de Fusion y Conflictos](12-conflictos.md)
    - Tipos de conflictos: contenido, renombre, eliminacion
    - Resolver conflictos paso a paso
    - Herramientas de merge visuales (meld, VS Code, IntelliJ)
    - Estrategias de merge: recursive, ours, theirs, octopus
    - git rerere: reutilizar resoluciones de conflictos
    - Prevencion de conflictos en equipos
    - git merge-base

13. [Capitulo 13: Reescritura de Historia](13-reescritura-historia.md)
    - Cuando y por que reescribir historia
    - git filter-branch (legado)
    - git filter-repo (herramienta moderna)
    - Eliminar datos sensibles del historial
    - Migrar repositorios (extraer subdirectorios)
    - Cambiar autor de commits (git commit --amend, filter-repo)
    - Reescribir mensajes de commit masivos

14. [Capitulo 14: Git a Gran Escala](14-gran-escala.md)
    - Git LFS (Large File Storage)
    - Repositorios grandes: partial clone, shallow clone
    - Sparse checkout y sparse index
    - Monorepos: estrategias, herramientas y build systems
    - Git en la empresa: politicas y gobierno
    - Git LFS file locking para archivos binarios
    - git maintenance y optimizacion programada

15. [Capitulo 15: Flujos de Trabajo](15-workflows.md)
    - Git Flow (modelo clasico de Vincent Driessen)
    - GitHub Flow (modelo simple de PR)
    - GitLab Flow (modelo con entornos)
    - Trunk-Based Development
    - Feature flags y CI/CD
    - Escalado de equipos con CODEOWNERS
    - Convenciones de ramas y commits

16. [Capitulo 16: CI/CD con Git](16-ci-cd.md)
    - Integracion continua: conceptos fundamentales
    - Acciones de GitHub (GitHub Actions)
    - GitLab CI/CD pipelines
    - Bitbucket Pipelines
    - Jenkins, CircleCI y otras herramientas
    - Automatizacion de versionado y releases
    - Semantic Release y conventional commits

17. [Capitulo 17: Git Avanzado](17-avanzado.md)
    - git bisect: encontrar bugs con busqueda binaria
    - git blame: auditoria linea por linea
    - git grep: busqueda avanzada en el repositorio
    - git reflog: tu red de seguridad
    - git archive: exportar codigo
    - git bundle: transferir repositorios sin red
    - git notes: metadatos para commits
    - git range-diff: comparar series de commits
    - git worktree: trabajar en multiples ramas

18. [Capitulo 18: Buenas Practicas y Convenciones](18-buenas-practicas.md)
    - Como escribir buenos mensajes de commit
    - Conventional Commits 1.0.0
    - Tamaño de commits y atomicidad
    - Estrategia de branching por tipo de proyecto
    - Proteccion de ramas (branch protection rules)
    - Code review efectiva con Git
    - Documentacion del repositorio (README, CONTRIBUTING, CHANGELOG)
    - Politicas de merge (merge commit, squash, rebase)

19. [Capitulo 19: Ejercicios Practicos](19-ejercicios.md)
    - Ejercicios nivel principiante
    - Ejercicios nivel intermedio
    - Ejercicios nivel avanzado
    - Proyectos integradores
    - Simulaciones de trabajo en equipo
    - Soluciones comentadas

20. [Capítulo 20: Git para Activos No-Código](20-activos-no-codigo.md)
    - Git para desarrollo de videojuegos (Unity, Unreal)
    - Git LFS + File Locking para assets binarios
    - Git para diseñadores (Figma, Photoshop, Illustrator)
    - Git para científicos de datos (Jupyter, DVC, datasets)
    - Git para documentación (Docs-as-Code)

21. [Capítulo 21: Migrando a Git desde SVN, Mercurial y Perforce](21-migrando-git.md)
    - Principio universal de migración (5 fases)
    - Migración desde Subversion (git svn, mapeo de autores)
    - Migración desde Mercurial (hg-fast-export)
    - Migración desde Perforce (git-p4, estrategia para repos gigantes)
    - Plantilla de comunicación y checklist del equipo

---

**Apendices**

A. [Referencia Rápida de Comandos](A-comandos.md)
   - Todos los comandos del libro en un solo lugar

B. [Hoja de Ruta de Aprendizaje](B-ruta-aprendizaje.md)
   - Guía visual de progresión de temas

C. [Git Forense y Auditoría](C-forense.md)
   - 10 preguntas forenses respondidas con scripts
   - Kit de herramientas (aliases)
   - Auditoría SOC2 / ISO 27001 con script de compliance
