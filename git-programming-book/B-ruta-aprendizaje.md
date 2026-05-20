# Apendice B: Hoja de Ruta de Aprendizaje

Este apendice presenta una guia estructurada para aprender Git de forma progresiva, desde principiante absoluto hasta nivel avanzado profesional. Cada nivel incluye los capitulos relevantes del libro, las habilidades a desarrollar, estimaciones de tiempo, prerrequisitos y proyectos practicos recomendados.

---

## B.1 Visión General de la Ruta

```
Nivel 1 (Principiante)          Nivel 2 (Intermedio Bajo)
    [1-2 semanas]                   [2-3 semanas]
    Capitulos: 1, 2, 4              Capitulos: 3, 5, 8
         |                                |
         v                                v
   +-----------+                   +-----------+
   | init      |                   | branches  |
   | clone     |                   | merge     |
   | add       |                   | restore   |
   | commit    |                   | reset     |
   | push/pull |                   | tags      |
   | log/diff  |                   | amend     |
   +-----------+                   +-----------+
         |                                |
         v                                v
Nivel 3 (Intermedio)             Nivel 4 (Intermedio Alto)
    [2-3 semanas]                   [2-3 semanas]
    Capitulos: 6, 7, 9, 12          Capitulos: 10, 13, 15, 18
         |                                |
         v                                v
   +-----------+                   +-----------+
   | rebase    |                   | hooks     |
   | cherry-p  |                   | workflows |
   | stash     |                   | rewrite   |
   | submodules|                   | conv.     |
   | conflicts |                   | practices |
   +-----------+                   +-----------+
         |                                |
         v                                v
Nivel 5 (Avanzado)
    [3-4 semanas]
    Capitulos: 11, 14, 16, 17
         |
         v
   +-----------+
   | internals |
   | LFS       |
   | CI/CD     |
   | bisect    |
   | monorepo  |
   +-----------+
```

---

## B.2 Nivel 1: Principiante

**Duracion estimada**: 1-2 semanas (10-15 horas de practica)

**Prerrequisitos**: Conocimientos basicos de terminal y sistema de archivos.

### Capitulos a Estudiar

| Capitulo | Titulo | Prioridad |
|----------|--------|-----------|
| 1 | Introduccion a Git | Esencial |
| 2 | Fundamentos de Git | Esencial |
| 4 | Trabajo con Repositorios Remotos | Esencial |

### Habilidades a Desarrollar

- [ ] Inicializar y clonar repositorios.
- [ ] Configurar identidad de usuario (`user.name`, `user.email`).
- [ ] Comprender el flujo basico: working directory -> staging area -> repository.
- [ ] Usar `git status` para ver el estado del proyecto.
- [ ] Crear commits con mensajes descriptivos.
- [ ] Ver historial con `git log` y cambios con `git diff`.
- [ ] Conectar repositorios locales con remotos (`remote`, `push`, `pull`, `fetch`).
- [ ] Entender el concepto de repositorio distribuido.

### Proyecto Practico: Repositorio Personal

**Objetivo**: Crear un repositorio en GitHub que contenga todos los ejercicios de este nivel.

**Actividades**:
1. Crear repositorio local y remoto.
2. Agregar un archivo `README.md` con descripcion del proyecto.
3. Crear 5 commits con cambios incrementales.
4. Configurar `.gitignore` para excluir archivos temporales.
5. Hacer push al remoto y clonar en otro directorio.
6. Hacer un cambio en el clon y pushearlo. Hacer pull en el original.
7. Simular un escenario de `git fetch` vs `git pull`.

**Entregable**: Repositorio en GitHub con al menos 8 commits y `.gitignore` configurado.

### Checklist de Autoevaluacion

- [ ] Puedo explicar el flujo de trabajo de Git (3 areas).
- [ ] Se cuando usar `git pull` vs `git fetch`.
- [ ] Entiendo que es un commit y como escribir un buen mensaje.
- [ ] Puedo clonar repositorios y conectarme a remotos.
- [ ] Se leer la salida de `git status` y `git log --oneline`.

---

## B.3 Nivel 2: Intermedio Bajo

**Duracion estimada**: 2-3 semanas (15-20 horas de practica)

**Prerrequisitos**: Dominio del Nivel 1. Comodidad con la terminal.

### Capitulos a Estudiar

| Capitulo | Titulo | Prioridad |
|----------|--------|-----------|
| 13 | Reescribiendo la Historia | Esencial |
| 14 | Git a Gran Escala | Recomendado |
| 15 | Flujos de Trabajo | Recomendado |
| 18 | Buenas Practicas y Convenciones | Esencial |

### Habilidades a Desarrollar

- [ ] Configurar hooks del lado cliente (pre-commit, commit-msg, pre-push).
- [ ] Configurar hooks del lado servidor (pre-receive, post-receive).
- [ ] Usar `git filter-repo` para limpiar/reescribir historial.
- [ ] Entender y aplicar Conventional Commits.
- [ ] Configurar proteccion de ramas en GitHub/GitLab.
- [ ] Realizar code reviews efectivas.
- [ ] Aplicar politicas de merge segun el contexto del proyecto.
- [ ] Configurar `.gitattributes` para fines de linea, merge drivers.
- [ ] Configurar `sparse-checkout` en monorepos.
- [ ] Configurar Git Worktree para trabajo paralelo.
- [ ] Mantener repositorios saludables (`git gc`, `git fsck`).

### Proyecto Practico: Configuracion de CI/CD y Proteccion de Ramas

**Objetivo**: Configurar un entorno profesional completo con CI/CD y proteccion de ramas.

**Actividades**:
1. Configurar Conventional Commits con commitlint + husky.
2. Configurar proteccion de ramas: PR requerido, CI pasando, historial lineal.
3. Implementar pre-commit hook con linting y deteccion de secrets.
4. Configurar `.gitattributes` para normalizar fines de linea.
5. Crear `CONTRIBUTING.md` y `CHANGELOG.md` con Keep a Changelog.
6. Usar `git filter-repo` para limpiar un archivo grande del historial.
7. Configurar Git worktree para trabajar en 2 features simultaneamente.
8. Mantener el repo: `git gc`, `git remote prune`, revision de tamaño.

**Entregables**:
- Repositorio configurado con protecciones y hooks.
- Documentacion completa (README, CONTRIBUTING, CHANGELOG).
- Evidencia de limpieza de historial con filter-repo.

### Checklist de Autoevaluacion

- [ ] Puedo configurar hooks que mejoran la calidad del equipo.
- [ ] Domino Conventional Commits y puedo enseñarlo a juniors.
- [ ] Se configurar protecciones de rama adecuadas al proyecto.
- [ ] Puedo hacer code review constructiva y rapida.
- [ ] Entiendo cuando usar merge commit, squash, o rebase merge.
- [ ] Puedo limpiar secretos o archivos grandes del historial con seguridad.

---

## B.6 Nivel 5: Avanzado

**Duracion estimada**: 3-4 semanas (25-35 horas de practica)

**Prerrequisitos**: Dominio completo de niveles anteriores. Experiencia liderando o manteniendo repositorios.

### Capitulos a Estudiar

| Capitulo | Titulo | Prioridad |
|----------|--------|-----------|
| 11 | Git Internals | Esencial |
| 14 | Git a Gran Escala | Esencial |
| 16 | CI/CD con Git | Esencial |
| 17 | Git Avanzado | Esencial |

### Habilidades a Desarrollar

- [ ] Entender el modelo de objetos de Git (blob, tree, commit, tag).
- [ ] Usar comandos plumbing para inspeccion y depuracion.
- [ ] Configurar y gestionar Git LFS para archivos binarios.
- [ ] Migrar repositorios existentes a LFS.
- [ ] Configurar pipelines CI/CD completos con GitHub Actions, GitLab CI.
- [ ] Integrar Docker y Kubernetes en pipelines.
- [ ] Configurar semantic release con conventional commits.
- [ ] Dominar `git bisect` para busqueda de bugs.
- [ ] Dominar `git blame` con opciones avanzadas.
- [ ] Usar `git grep` para busquedas complejas en el repositorio.
- [ ] Dominar `git reflog` para recuperacion avanzada.
- [ ] Crear y usar `git bundle` para transferencia offline.
- [ ] Usar `git notes` para metadatos.
- [ ] Comparar series de commits con `git range-diff`.
- [ ] Implementar estrategias de deploy: blue-green, canary, rolling.

### Proyecto Practico: Monorepo con CI/CD y LFS

**Objetivo**: Configurar un monorepo empresarial completo con CI/CD, LFS y automatizacion.

**Actividades**:
1. Crear monorepo con 3 paquetes (web, API, shared).
2. Configurar Git LFS para archivos de diseño (`.psd`, `.sketch`).
3. Migrar archivos binarios existentes a LFS.
4. Configurar CI/CD por paquete usando path filtering.
5. Implementar `CODEOWNERS` para revision automatica.
6. Configurar semantic release con versionado independiente por paquete.
7. Optimizar repositorio con `git gc --aggressive`.
8. Implementar estrategia blue-green deploy en el pipeline.
9. Configurar `sparse-checkout` para desarrolladores de un solo paquete.
10. Usar `git range-diff` para verificar la integridad tras rebases.

**Entregables**:
- Monorepo completamente configurado en GitHub/GitLab.
- Pipelines CI/CD funcionales con path filtering.
- Releases automatizadas con semantic-release.
- Documentacion de arquitectura de CI/CD y monorepo.

### Checklist de Autoevaluacion

- [ ] Entiendo el modelo de objetos de Git a nivel de implementacion.
- [ ] Puedo configurar y migrar a Git LFS en un repositorio real.
- [ ] Puedo diseñar pipelines CI/CD complejos desde cero.
- [ ] Domino `git bisect` para encontrar bugs en minutos.
- [ ] Puedo recuperar cualquier situacion de perdida de datos con `reflog`.
- [ ] Puedo transferir repositorios sin acceso a red.
- [ ] Puedo implementar estrategias de deploy avanzadas.
- [ ] Estoy preparado para liderar la adopcion de Git en un equipo/empresa.

---

## B.7 Resumen de Niveles y Tiempos

| Nivel | Semanas | Capitulos | Enfoque Principal |
|-------|---------|-----------|-------------------|
| 1 - Principiante | 1-2 | 1, 2, 4 | Operaciones basicas, remotos |
| 2 - Intermedio Bajo | 2-3 | 3, 5, 8 | Ramas, deshacer, tags |
| 3 - Intermedio | 2-3 | 6, 7, 9, 12 | Rebase, stash, submodulos, conflictos |
| 4 - Intermedio Alto | 2-3 | 10, 13, 14, 15, 18 | Hooks, reescritura, gran escala, workflows, practicas |
| 5 - Avanzado | 3-4 | 11, 14, 16, 17 | Internals, LFS, CI/CD, comandos avanzados |

**Tiempo total estimado**: 10-15 semanas (100-120 horas) para completar toda la ruta de aprendizaje.

---

## B.8 Combinaciones de Aprendizaje por Rol

### Desarrollador Junior

**Enfoque**: Niveles 1 y 2.
- Comandos basicos diarios.
- Trabajo con ramas de feature.
- Colaboracion via PRs.

### Desarrollador Senior / Tech Lead

**Enfoque**: Niveles 3, 4 y 5.
- Diseno de workflows de equipo.
- Configuracion de CI/CD y proteccion de ramas.
- Mentoria en buenas practicas.
- Recuperacion de incidentes con reflog/bisect.

### DevOps / SRE

**Enfoque**: Niveles 4 y 5, capitulos 10, 14, 16, 17.
- Automatizacion de pipelines CI/CD.
- Hooks del lado servidor.
- Git LFS.
- Integracion con Docker y Kubernetes.

### Tech Writer / Documentador

**Enfoque**: Nivel 4, capitulo 18.
- README, CONTRIBUTING, CHANGELOG.
- Conventional Commits.
- Documentacion de workflows.

### Open Source Maintainer

**Enfoque**: Niveles 3, 4 y 5.
- Gestion de forks y PRs.
- Proteccion de ramas.
- Semantic release.
- Code review y politicas de merge.

### Estudiante / Autodidacta

**Enfoque**: Niveles 1 al 3 en orden, complementado con herramientas visuales.
- Seguir los ejercicios del Cap 19 de forma progresiva.
- Usar [Learn Git Branching](https://learngitbranching.js.org) para visualizar ramas y rebase.
- Explorar [Oh My Git!](https://ohmygit.org) para practicar con un juego interactivo.
- Crear un portafolio en GitHub con proyectos personales versionados correctamente.
- Leer `git log` de proyectos open source grandes (React, VS Code, Linux) para ver Conventional Commits en accion.

### Migrando desde SVN / Mercurial

**Enfoque**: Nivel 1 y 2 acelerado + Proyecto Integrador 2 del Cap 19.
- Cap 1-4: Fundamentos (la filosofía distribuida es el mayor cambio mental).
- Cap 6: Rebase es el equivalente funcional de `svn update` pero más potente.
- Proyecto Integrador 2 (Cap 19): Migración real de SVN a Git preservando historial.
- Conceptos equivalentes: `git svn clone`, `git svn rebase`, `git svn dcommit`.
- Recursos específicos: [git-scm.com/book/en/v2/Git-and-Other-Systems](https://git-scm.com/book/en/v2/Git-and-Other-Systems)

---

## B.9 Recomendaciones de Practica Diaria

| Duracion | Actividad | Frecuencia |
|----------|-----------|------------|
| 15 min | Ejercicio del capitulo actual | Diaria |
| 30 min | Proyecto practico del nivel | 3-4 veces/semana |
| 1 hora | Sesion de repaso y dudas | Semanal |
| 10 min | Leer `git log` de un proyecto open source | Diaria (opcional) |
| 5 min | Revisar y practicar un alias o atajo nuevo | Semanal |

### Recursos Complementarios

- **Documentacion oficial**: https://git-scm.com/doc
- **Pro Git Book (Scott Chacon)**: https://git-scm.com/book/es/v2
- **Conventional Commits**: https://www.conventionalcommits.org
- **Keep a Changelog**: https://keepachangelog.com
- **GitHub Actions Docs**: https://docs.github.com/actions
- **GitLab CI Docs**: https://docs.gitlab.com/ee/ci/
- **Learn Git Branching** (visual, interactivo): https://learngitbranching.js.org
- **Oh My Git!** (juego educativo de Git): https://ohmygit.org
- **Katacoda Git Scenarios** (práctica en navegador): https://katacoda.com/courses/git
- **Visualizing Git** (visualización de commits): https://git-school.github.io/visualizing-git/
- **ProGit 2ª edición (español)**: https://github.com/progit/progit2-es

---

## B.10 Mapa Conceptual de Dependencias

```
Conceptos Fundamentales
    |
    +-- Working Dir, Staging, Repository (Cap 2)
    |       |
    |       +-- Log, Diff, Show (Cap 2)
    |       +-- Undo: Restore, Reset, Revert (Cap 5)
    |
    +-- Remotos (Cap 4)
    |       |
    |       +-- Fetch, Pull, Push
    |       +-- Colaboracion
    |
    +-- Ramas (Cap 3)
            |
            +-- Merge (Cap 3)
            +-- Rebase (Cap 6)
            +-- Cherry-pick (Cap 6)
            +-- Stash (Cap 7)
            +-- Conflictos (Cap 12)
            +-- Workflows (Cap 15)

Conceptos Avanzados
    |
    +-- Hooks (Cap 10)
    |       +-- Pre-commit, Post-receive
    |       +-- CI/CD triggers (Cap 16)
    |
    +-- Internals (Cap 11)
    |       +-- Blob, Tree, Commit, Tag
    |       +-- Plumbing commands
    |
    +-- History Rewriting (Cap 13)
    |       +-- Filter-repo
    |       +-- Bisect (Cap 17)
    |
    +-- Large Files (Cap 14)
    |       +-- LFS
    |       +-- Bundle (Cap 17)
    |
    +-- Monorepos / LFS / Gran Escala (Cap 14)
    |       +-- Sparse checkout
    |       +-- Worktree
    |
    +-- CI/CD (Cap 16)
    |       +-- GitHub Actions
    |       +-- GitLab CI
    |       +-- Semantic Release
    |
    +-- Buenas Practicas (Cap 18)
            +-- Conventional Commits
            +-- Code Review
            +-- Seguridad
```

---

## B.11 Posibles Certificaciones Complementarias

Aunque no existen certificaciones oficiales de Git ampliamente reconocidas, las siguientes opciones pueden complementar tu formacion:

| Certificacion | Organizacion | Enfoque |
|---------------|--------------|---------|
| GitHub Foundations | GitHub | Uso basico de Git y GitHub |
| GitHub Actions | GitHub | Automatizacion CI/CD |
| GitHub Advanced Security | GitHub | Seguridad en repositorios |
| GitLab Certified Associate | GitLab | Git y GitLab CI/CD |
| AZ-400 (DevOps) | Microsoft | Git integrado en Azure DevOps |

*Nota: Ninguna certificacion reemplaza la practica real. Un portafolio de GitHub con proyectos bien documentados y PRs a proyectos open source tiene mas valor que cualquier certificacion.*

---

*Fin del Apendice B. Este documento es una guia viva; ajusta los tiempos y el orden segun tu ritmo de aprendizaje y contexto profesional.*

---

← [Capítulo anterior](A-comandos.md) | [Inicio](README.md) | [Capítulo siguiente →](C-forense.md)
