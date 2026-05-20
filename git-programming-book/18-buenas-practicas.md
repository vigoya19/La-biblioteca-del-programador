# Capitulo 18: Buenas Practicas y Convenciones

El dominio tecnico de Git es solo una parte del desarrollo profesional. La disciplina en los mensajes de commit, la organizacion del repositorio, las convenciones de ramas y la cultura de code review determinan la calidad y mantenibilidad del proyecto a largo plazo.

---

## 18.1 El Mensaje de Commit Perfecto

Un buen mensaje de commit comunica el **que** y el **por que** de un cambio, permitiendo a cualquier miembro del equipo (incluido tu yo futuro) entender el contexto sin necesidad de leer el diff completo.

### 18.1.1 Estructura: Sujeto + Cuerpo + Pie

```
Linea de sujeto (obligatoria): maximo 50 caracteres

Cuerpo (opcional): explicacion detallada del cambio, contexto
adicional, y razonamiento. Separado del sujeto por una linea en
blanco. Maximo 72 caracteres por linea.

Pie (opcional): referencias a issues, breaking changes, co-autores.
```

Ejemplo completo:
```
fix(auth): corregir race condition en renovacion de token

El refresh token se invalidaba antes de que el access token
fuera renovado, causando errores 401 en requests concurrentes.

La solucion implementa un mutex por sesion que serializa las
renovaciones de token, garantizando que solo una renovacion
ocurra a la vez.

Closes #1247
BREAKING CHANGE: La funcion refreshSession() ahora requiere
un parametro adicional sessionId.
Co-authored-by: Bob <bob@ejemplo.com>
```

### 18.1.2 Regla 50/72

| Elemento | Longitud Maxima | Razon |
|----------|-----------------|-------|
| Sujeto | 50 caracteres | Visibilidad completa en `git log --oneline`, GitHub UI |
| Lineas del cuerpo | 72 caracteres | Legibilidad en terminales de 80 columnas con margenes |

```bash
# Verificar longitud del sujeto del ultimo commit
git log -1 --pretty=%s | wc -c
```

### 18.1.3 Lenguaje Imperativo

```
Usa:  "Add login feature"       (imperativo)
No:   "Added login feature"     (pasado)
No:   "Adding login feature"    (gerundio)
No:   "Adds login feature"      (tercera persona)

Mas ejemplos:
  fix: corregir overflow         <- BIEN
  remove: eliminar codigo muerto  <- BIEN
  update: actualizar dependencias <- BIEN
```

> **Razon**: Git mismo usa el imperativo en sus mensajes generados automaticamente: "Merge branch", "Revert commit". La convencion mantiene consistencia con el ecosistema.

### 18.1.4 Que y Por Que, No Como

| Pregunta | Contenido |
|----------|-----------|
| **Que** | Sujeto: resume el cambio |
| **Por que** | Cuerpo: contexto, problema, motivacion |
| **Como** | El diff (codigo). No duplicarlo en texto |

```
# Mal (describe el como)
fix: cambiar if por switch en parser.js

# Bien (explica el que y por que)
fix(parser): soportar tokens unicode en lexer

El parser fallaba con identificadores que contenian caracteres
unicode (emojis, acentos). El lexer asumia ASCII en la fase
de tokenizacion.
```

---

## 18.2 Conventional Commits 1.0.0

La especificacion [Conventional Commits](https://www.conventionalcommits.org) define un formato estandar para mensajes de commit que permite automatizar versionado y changelogs.

### 18.2.1 Formato

```
<tipo>[alcance opcional]: <descripcion>

[cuerpo opcional]

[pie(s) opcionales]
```

### 18.2.2 Tipos Principales

| Tipo | Proposito | Versionado |
|------|-----------|------------|
| `feat` | Nueva funcionalidad | MINOR |
| `fix` | Correccion de bug | PATCH |
| `docs` | Cambios en documentacion | - |
| `style` | Formato, whitespace, punto y coma | - |
| `refactor` | Cambio de codigo sin fix ni feat | - |
| `perf` | Mejora de rendimiento | PATCH |
| `test` | Agregar o corregir tests | - |
| `chore` | Tareas de mantenimiento, deps | - |
| `ci` | Cambios en CI/CD | - |
| `build` | Sistema de build, dependencias externas | - |
| `revert` | Reversion de commit anterior | - |

```bash
# Ejemplos de conventional commits bien formados
feat(api): agregar endpoint de autenticacion
feat(ui)!: rediseñar componente de navegacion
fix(db): corregir deadlock en conexiones MySQL
docs(readme): actualizar instrucciones de instalacion
chore(deps): actualizar lodash a 4.17.21
perf(parser): reducir tiempo de parseo en 40%
ci: configurar matrix build para Node 18/20/22
```

### 18.2.3 BREAKING CHANGE y !

Dos formas de indicar cambios que rompen compatibilidad:

```bash
# Forma 1: ! despues del tipo/alcance
feat(api)!: cambiar firma de authenticate()

# Forma 2: BREAKING CHANGE en el pie
feat(api): cambiar firma de authenticate()

BREAKING CHANGE: authenticate() ya no acepta callback;
ahora retorna una Promise.
```

Ambas formas incrementan la version **MAJOR** en Semantic Versioning.

### 18.2.4 Relacion con Semantic Versioning

```
feat -> MINOR increment (1.2.3 -> 1.3.0)
fix  -> PATCH increment (1.2.3 -> 1.2.4)

feat! o BREAKING CHANGE -> MAJOR increment (1.2.3 -> 2.0.0)
```

```bash
# Validar conventional commits en CI
npm install --save-dev @commitlint/cli @commitlint/config-conventional
echo "module.exports = { extends: ['@commitlint/config-conventional'] };" > commitlint.config.js

# Hook de commit que valida
npx husky add .husky/commit-msg "npx --no -- commitlint --edit \$1"
```

---

## 18.3 Atomicidad de Commits

Un commit atomico contiene **un solo cambio logico** que puede revertirse o cherry-pickearse de forma independiente.

### Principios

- Un commit por concepto.
- No mezclar formateo con cambios funcionales.
- No mezclar feature nueva con bug fix.
- Si la descripcion necesita "y", probablemente son dos commits.

```
# Mal: commit no atomico
git commit -m "Add login, fix navbar bug, and update README"

# Bien: tres commits atomicos
git add src/login.js && git commit -m "feat(auth): add login page"
git add src/navbar.js && git commit -m "fix(navbar): fix mobile menu overlap"
git add README.md && git commit -m "docs: update setup instructions"
```

### Como Lograr Atomicidad

```bash
# Agregar interactivamente por hunks
git add -p

# Crear commits selectivos
git commit -m "feat(api): add user endpoint" -- src/api/users.js

# Separar cambios en staging area
git stash push --keep-index  # Guarda lo no staggeado
# Testear lo staggeado
git stash pop
```

> **Tip**: Si al escribir el mensaje de commit necesitas enumerar varios cambios no relacionados, es señal de que deberias dividirlo. Usa `git add -p` para construir commits atomicos quirurgicamente.

---

## 18.4 Frecuencia de Commits

### Principio: Commit Early, Commit Often

- Haz commits frecuentes durante el desarrollo, en tu rama local.
- Antes de crear un PR, reorganiza los commits con rebase interactivo.
- Cada commit final debe ser atomico y pasar los tests.

```
Flujo de trabajo tipico:

[Desarrollo local: commits frecuentes]
wip: empezar formulario
wip: agregar validacion
wip: conectar API
wip: corregir CORS

[Antes del PR: squash/rebase]
feat(ui): agregar formulario de registro con validacion y conexion API
```

```bash
# Reorganizar commits antes del PR
git rebase -i HEAD~5
# squash los wip en un solo commit bien descrito
```

---

## 18.5 Estrategia de Branching

### 18.5.1 Por Tipo de Proyecto

| Tipo de Proyecto | Estrategia Recomendada |
|------------------|----------------------|
| Web / SaaS | Trunk-based development con feature flags |
| Libreria / Paquete | GitFlow o GitHub Flow |
| Monorepo | Trunk-based, cada subproyecto tiene su pipeline |
| Enterprise / Regulado | GitFlow con ramas de release y hotfix |
| Open Source | GitHub Flow: fork + PR desde rama de feature |

### 18.5.2 Convencion de Nombres

```
# Estructura recomendada
feature/<descripcion>    # Nueva funcionalidad
bugfix/<id-issue>        # Correccion de bug
hotfix/<descripcion>     # Correccion urgente en produccion
release/<version>        # Preparacion de release
chore/<descripcion>      # Tareas de mantenimiento
experiment/<descripcion> # Pruebas y prototipos
docs/<descripcion>       # Cambios de documentacion
refactor/<descripcion>   # Refactorizaciones

# Ejemplos validos
feature/login-oauth
bugfix/123-login-redirect
hotfix/2.1.1-token-expiry
release/2.2.0
chore/update-deps
```

---

## 18.6 Proteccion de Ramas

### GitHub Branch Protection Rules

Configurables en Settings > Branches > Add rule.

```yaml
# Reglas tipicas para main/master
- Require a pull request before merging: ON
- Require approvals: 1  (o 2+ para proyectos criticos)
- Dismiss stale pull request approvals: ON
- Require status checks to pass before merging: ON
  - CI / Test
  - CI / Lint
  - CI / Build
- Require conversation resolution before merging: ON
- Require linear history: ON  (solo squash/rebase merge)
- Do not allow bypassing the above settings: ON
- Restrict who can push: Admins only
- Allow force pushes: OFF
- Allow deletions: OFF
```

### GitLab Protected Branches

```yaml
# Settings > Repository > Protected Branches
main:
  Allowed to merge: Maintainers
  Allowed to push: No one
  Allowed to force push: OFF
  Code owner approval: ON (1 approval)
```

> **Advertencia**: La proteccion de ramas sin CI configurado puede bloquear todos los merges. Asegurate de que los checks requeridos existan y esten funcionando antes de activar la regla.

---

## 18.7 Code Review Efectiva

### 18.7.1 Como Hacer Buenos Pull Requests

```markdown
## Titulo
feat(api): agregar endpoint POST /api/users

## Descripcion
### Que
Nuevo endpoint REST para creacion de usuarios con validacion
de email y hash de contraseña bcrypt.

### Por que
El frontend necesita registro de usuarios para la funcionalidad de
perfiles. Issue relacionado: #456

### Cambios
- Nuevo controlador UsersController::create
- Validacion de email unico
- Hash bcrypt con salt rounds configurable
- Tests: unitarios + integracion

### Screenshots / Evidencia
| Antes | Despues |
|-------|---------|
| (no existia) | POST /api/users -> 201 Created |

## Como Testear
1. POST /api/users con body {email, password, name}
2. Verificar 201 y header Location
3. Verificar que password en BD esta hasheado

## Checklist
- [x] Tests unitarios cubren casos feliz y de error
- [x] Documentacion de API actualizada (Swagger)
- [x] Migraciones de BD preparadas
- [x] Variables de entorno documentadas
```

### 18.7.2 Como Revisar: Checklist

```
[ ] El codigo resuelve el problema descrito
[ ] La arquitectura/solucion es adecuada
[ ] No introduce bugs evidentes
[ ] Los tests cubren casos principales y bordes
[ ] El estilo coincide con las guias del proyecto
[ ] No hay codigo muerto, comentado, ni debug
[ ] No hay secrets hardcodeados
[ ] Las dependencias nuevas estan justificadas
[ ] La documentacion esta actualizada
[ ] Los mensajes de commit siguen la convencion
```

### 18.7.3 Comentarios Constructivos

```
# Mal
"Esto esta mal"

# Bien
"Este bucle recorre todo el array para cada request.
Sugiero usar un Map pre-construido: O(n+m) -> O(m).
Ver ejemplo en src/utils/lookup.js:42"

# Mal
"No me gusta este patron"

# Bien
"Entiendo que quieras usar Singleton aqui, pero complica el
testing. ¿Consideraste usar dependency injection? Asi los
tests pueden inyectar mocks sin hackear el singleton."
```

### 18.7.4 Tamaño de PRs

> **Regla de oro**: Un PR debe ser revisable en < 30 minutos. Idealmente **< 400 lineas** cambiadas. PRs grandes deben dividirse en varios PRs mas pequeños y enfocados.

- PRs de 1-100 lineas: review en minutos, alta calidad.
- PRs de 100-400 lineas: buena practica, revisables en una sesion.
- PRs de 400-800 lineas: considerar dividir.
- PRs de 800+ lineas: dividir obligatoriamente si es posible.

---

## 18.8 Documentacion del Repositorio

### 18.8.1 README.md

```markdown
# Nombre del Proyecto

[![CI](https://github.com/org/proyecto/actions/workflows/ci.yml/badge.svg)]
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)]

Descripcion corta y clara del proyecto (1-3 oraciones).

## Instalacion

git clone https://github.com/org/proyecto.git
cd proyecto
npm install

## Uso

import { Componente } from 'proyecto';

## Documentacion

Link a docs completas.

## Contribucion

Ver CONTRIBUTING.md.

## Licencia

MIT - Ver LICENSE.
```

### 18.8.2 CONTRIBUTING.md

```markdown
# Guia de Contribucion

## Flujo de Trabajo
1. Fork del repositorio
2. Crear rama: feature/descripcion
3. Commits con Conventional Commits
4. PR a main, describiendo cambios

## Configuracion Local
npm install
cp .env.example .env

## Tests
npm test            # Unitarios
npm run test:e2e    # End-to-end

## Linting y Formato
npm run lint
npm run format

## Guia de Estilo
- TypeScript strict mode
- Tests obligatorios para nuevas features
- Commits atomicos
```

### 18.8.3 CHANGELOG.md (Keep a Changelog)

```markdown
# Changelog

## [2.1.0] - 2026-05-19

### Added
- Soporte para autenticacion OAuth2 (#234)

### Changed
- Mejora de rendimiento en modulo de parsing (40% mas rapido)

### Deprecated
- API v1: /api/v1/users sera removida en v3.0.0

### Fixed
- Race condition en renovacion de token (#567)

### Security
- Actualizacion de dependencia con CVE critico (#890)
```

### 18.8.4 LICENSE

```markdown
MIT License

Copyright (c) 2026 Nombre del Autor

Permission is hereby granted, free of charge...
```

---

## 18.9 Politicas de Merge

| Metodo | Resultado | Mejor para |
|--------|-----------|------------|
| **Merge commit** (`--no-ff`) | Commit de merge explicito, conserva historial | Proyectos enterprise, trazabilidad maxima |
| **Squash merge** | Un solo commit en main por PR | Feature branches, historial limpio en main |
| **Rebase merge** (`--ff-only`) | Historial lineal, sin commits de merge | Trunk-based development |

```bash
# Merge commit (conserva contexto de feature branch)
git merge --no-ff feature/nueva

# Squash merge (aplana rama en un commit)
git merge --squash feature/nueva
git commit -m "feat: descripcion unificada"

# Rebase merge (historial lineal)
git rebase main
git checkout main
git merge --ff-only feature/nueva
```

| Politica | Historial | Revertir PR | Cherry-pick desde PR | Impacto en bisect |
|----------|-----------|-------------|---------------------|-------------------|
| Merge commit | Completo | Facil (`git revert -m 1`) | Facil | Bajo: cada commit es independiente |
| Squash merge | Limpio | Facil (un commit) | Imposible (todo en uno) | Medio: pierde granularidad de cambios |
| Rebase merge | Lineal | Facil | Posible pero frágil | Alto: commits aislados, ideal para bisect |

```bash
# Revertir un merge commit requiere -m (mainline)
git revert -m 1 <SHA-del-merge>  # -m 1 = mantener el primer padre (main)

# Squash merge: revertir es simple porque es un solo commit
git revert <SHA-del-squash>

# Rebase merge: cada commit es individual, bisect funciona óptimamente
git bisect start HEAD v1.0.0
git bisect run npm test
# Los commits lineales permiten aislar el bug con precisión
```

> **Impacto en flujo de trabajo:** Squash merge hace imposible cherry-pickear un fix individual de un PR a otra rama. Si tu equipo hace backports frecuentes (hotfixes a releases antiguas), prefiere merge commit o rebase merge. Si priorizas git bisect, el historial lineal del rebase merge es óptimo.

> **Recomendacion**: Squash merge para equipos pequeños/startups. Merge commit para enterprise/regulado. Rebase merge solo si todo el equipo domina rebase.

---

## 18.10 .gitattributes

```bash
# .gitattributes

# Normalizar fines de linea
* text=auto

# Forzar fines de linea especificos
*.sh text eol=lf
*.bat text eol=crlf

# Archivos binarios (no hacer diff)
*.png binary
*.jpg binary
*.pdf binary
*.zip binary

# Estrategia de merge para archivos especificos
CHANGELOG.md merge=union
package-lock.json merge=ours

# Export-ignore (archivos excluidos de git archive)
.gitattributes export-ignore
.gitignore export-ignore
tests/ export-ignore
.github/ export-ignore

# Diff drivers para formatos no estandar
*.ipynb diff=jupyternotebook
*.lock diff=lockfile
```

```bash
# Verificar configuracion de gitattributes
git check-attr -a src/app.js
# src/app.js: text: set
```

---

## 18.11 Mantener el Repositorio Limpio

```bash
# Recolectar objetos inalcanzables y comprimir
git gc --aggressive --prune=now

# Verificar integridad de la base de datos de objetos
git fsck --full

# Ver objetos grandes (método correcto)
git rev-list --objects --all | \
  git cat-file --batch-check='%(objectsize) %(objectname) %(rest)' | \
  sort -nr | head -20

# Alternativa más simple con script incluido en Git
git rev-list --objects --all | \
  git cat-file --batch-check='%(objectname) %(objecttype) %(objectsize)' | \
  awk '$2 == "blob" {print $3, $1}' | \
  sort -nr | head -20
```

### Análisis de Tamaño

```bash
# Tamaño del directorio .git
du -sh .git

# Objetos por tipo y tamaño
git count-objects -vH
```

### Limpieza de Referencias

```bash
# Limpiar referencias obsoletas
git remote prune origin

# Eliminar ramas locales ya mergeadas
git branch --merged main | grep -v "main" | xargs git branch -d
```

> **Tip**: Ejecuta `git gc --aggressive` trimestralmente en repositorios activos. Para repos con miles de commits, considera git-lfs para archivos binarios y `git filter-repo` para limpiar objetos grandes del historial.

---

## 18.12 Seguridad

### 18.12.1 No Commitear Secretos

```bash
# .gitignore DEBE incluir:
.env
.env.local
.env.*.local
*.pem
*.key
credentials.json
service-account.json
secrets.yaml
id_rsa
*.pfx
*.p12
```

### 18.12.2 Pre-commit Hooks

```bash
# Instalar pre-commit framework
pip install pre-commit

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: detect-private-key
      - id: detect-aws-credentials
        args: [--allow-missing-credentials]

  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

### 18.12.3 git-secrets (AWS)

```bash
# Instalar y configurar git-secrets
brew install git-secrets
git secrets --install
git secrets --register-aws

# Agregar patrones personalizados
git secrets --add 'sk-[a-zA-Z0-9]{32}'  # OpenAI keys
git secrets --add 'ghp_[a-zA-Z0-9]{36}'  # GitHub tokens

# Escanear todo el historial
git secrets --scan-history
```

### 18.12.4 Escaneo en CI/CD

```yaml
# GitHub Action para detectar secrets
- name: Secret Scanning
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

> **Advertencia**: Si accidentalmente commiteaste un secreto, considera que ya esta comprometido. No basta con hacer un nuevo commit quitandolo; rota el secreto inmediatamente y luego limpia el historial con `git filter-repo`.

---

## 18.13 `.gitallowed`: Falsos Positivos en Gitleaks

Cuando herramientas como Gitleaks o TruffleHog detectan patrones que parecen secretos pero no lo son (ejemplo: claves de ejemplo en documentación), puedes crear un archivo `.gitallowed` en la raíz:

```
# .gitallowed: patrones que gitleaks debe ignorar
# Ejemplos/documentación con claves ficticias
ghp_example_fake_key_12345678901234567890
sk_test_demo_key_for_documentation_only

# Reglas glob para excluir archivos completos
docs/examples/secrets.md
**/test/fixtures/**
```

```bash
# Configurar gitleaks para respetar .gitallowed
gitleaks detect --gitleaks-ignore-path .gitallowed

# En pre-commit:
repos:
  - repo: https://github.com/zricethezav/gitleaks
    hooks:
      - id: gitleaks
        args: ['--gitleaks-ignore-path', '.gitallowed']
```

## 18.14 GitHub Secret Scanning y Push Protection

GitHub ofrece detección de secretos a nivel de plataforma (no requiere configuración en el repo):

- **Secret scanning (gratis en repos públicos):** Escanea el historial en busca de patrones conocidos (tokens de GitHub, AWS, GCP, npm, etc.).
- **Push protection (gratis en repos públicos):** Bloquea pushes que contengan secretos detectados **antes** de que lleguen al repositorio remoto.

```bash
# Si GitHub bloquea tu push:
# remote: error: GH013: Repository commit push blocked by push protection.
# remote: error: We found a secret in commit abc1234.
# remote: error: Secret type: GitHub Personal Access Token

# Opción 1: Eliminar el secreto del commit y hacer force-push
git filter-repo --replace-text <(echo "ghp_xxx==>***REDACTED***")
git push --force-with-lease

# Opción 2: Si es un falso positivo, marcar como "seguro" en la UI de GitHub
# Settings > Code Security > Secret scanning > "Allow" en el alert
```

> **Push protection evita el error más común:** commitear un secret y tener que rotarlo después. Actívalo en todos los repositorios de la organización.

---

## 18.15 Git Aliases: Productividad en el Terminal

Los aliases convierten comandos frecuentes en atajos rápidos:

```bash
# Aliases básicos de log
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
git config --global alias.ll "log --oneline --graph --decorate"
git config --global alias.last "log -1 HEAD"

# Aliases de flujo de trabajo
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage "reset HEAD --"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.amend "commit --amend --no-edit"

# Aliases de inspección
git config --global alias.d diff
git config --global alias.ds "diff --staged"
git config --global alias.who "shortlog -sne"
git config --global alias.hist "log --follow --patch"

# Funciones shell complejas como aliases
git config --global alias.cleanup "!git branch --merged main | grep -v 'main' | xargs git branch -d"
git config --global alias.prune-all "!git remote prune origin && git branch --merged main | grep -v 'main' | xargs git branch -d"
```

### Compartir Aliases con el Equipo

```bash
# Opción 1: Script de setup que configure aliases comunes
# tools/setup-git-aliases.sh
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.co checkout
# Comparte este script en el repo

# Opción 2: Copiar sección [alias] de ~/.gitconfig
cat ~/.gitconfig
# [alias]
#     lg = log --oneline --graph --decorate --all
#     co = checkout
#     ci = commit

# Opción 3: Documentar aliases recomendados en CONTRIBUTING.md
```

## 18.16 Commit Signing (GPG/SSH): Verificación de Autoría

Firmar commits y tags garantiza que el código proviene de quien dice venir, protegiendo contra suplantación:

### Configuración GPG

```bash
# Generar clave GPG
gpg --full-generate-key
# Elegir RSA, 4096 bits, nombre y email igual a git config

# Listar claves
gpg --list-secret-keys --keyid-format LONG
# sec   rsa4096/3AA5C34371567BD2 2026-01-01 [SC]

# Configurar Git para firmar
git config --global user.signingkey 3AA5C34371567BD2
git config --global commit.gpgsign true    # Firmar todos los commits
git config --global tag.gpgsign true       # Firmar todos los tags

# Firmar un commit individual
git commit -S -m "feat: cambio firmado"

# Verificar firma
git log --show-signature
# commit abc1234 (HEAD -> main)
# gpg: Good signature from "Alice <alice@ejemplo.com>"

# Exportar clave pública para GitHub
gpg --armor --export 3AA5C34371567BD2
# Agregar output en GitHub: Settings > SSH and GPG keys > New GPG key
```

### Configuración SSH (Git 2.34+)

Git también soporta firmar commits con claves SSH (más simple si ya usas SSH para autenticación):

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true

# El commit aparecerá como "Verified" (SSH) en GitHub
```

### GitHub Vigilant Mode

En Settings > SSH and GPG keys, activa **Vigilant Mode** (modo vigilante). Con esto, cualquier commit sin firma de un colaborador aparecerá marcado como "Unverified" en la UI de GitHub, alertando visualmente sobre autoría no confirmada.

## 18.17 `includeIf` en `.gitconfig`: Identidad por Proyecto

Permite usar configuración diferente según el directorio del repositorio. Ideal para separar trabajo/personal o empresas/clientes:

```ini
# ~/.gitconfig
[user]
    name = Alice Personal
    email = alice@gmail.com

[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/clientes/"]
    path = ~/.gitconfig-clientes

# ~/.gitconfig-work
[user]
    name = Alice Corp
    email = alice@empresa.com
```

```bash
# Todos los repos en ~/work/ usarán automáticamente la identidad corporativa
cd ~/work/proyecto-a && git config user.email  # alice@empresa.com
cd ~/personal/proyecto-b && git config user.email  # alice@gmail.com

# También puedes incluir por rama o URL remota:
[includeIf "hasconfig:remote.*.url:git@github.com:empresa/**"]
    path = ~/.gitconfig-work
```

## 18.18 `.editorconfig` + `.gitattributes`: Complemento de Formato

Mientras `.gitattributes` controla cómo Git maneja los archivos, `.editorconfig` asegura que todos los editores del equipo usen el mismo formato:

```ini
# .editorconfig (raíz del proyecto)
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.{py,go,rs}]
indent_size = 4

[Makefile]
indent_style = tab

[*.md]
trim_trailing_whitespace = false
```

**Complementariedad:**
- `.editorconfig` → Formato en el editor (espacios, tabs, encoding, EOL)
- `.gitattributes` → Comportamiento de Git (EOL en checkout, diff drivers, merge drivers, LFS, export-ignore)

Ambos deben versionarse en el repositorio para que todo el equipo tenga las mismas reglas.

---

Las buenas practicas transforman Git de una herramienta tecnica a un pilar de la cultura de ingenieria. El mensaje de commit perfecto sigue la estructura sujeto-cuerpo-pie con lenguaje imperativo y la regla 50/72. Conventional Commits estandariza el formato para automatizar versionado. La atomicidad y frecuencia adecuada de commits facilitan la revision y revertibilidad. La estrategia de branching y la proteccion de ramas protegen la rama principal. La code review efectiva con PRs pequeños y comentarios constructivos eleva la calidad del codigo. La documentacion del repositorio (README, CONTRIBUTING, CHANGELOG) guia a contribuidores. Las politicas de merge impactan en revertibilidad, cherry-pick y bisect; elige segun tu contexto. `.gitattributes` y `.editorconfig` normalizan el formato entre editores. Los aliases de Git aceleran el flujo diario. La firma de commits (GPG/SSH) y GitHub Vigilant Mode verifican autoria. `includeIf` separa identidades por proyecto (trabajo/personal). El mantenimiento regular y la deteccion de objetos grandes mantienen el repositorio saludable. La seguridad (`.gitallowed`, GitHub Secret Scanning, Push Protection) previene fugas de secretos en todo el pipeline.

---

## Ejercicios Propuestos

1. **Auditoria de mensajes de commit**: Revisa los ultimos 20 commits de un proyecto real o de practica. Clasifica cada uno segun Conventional Commits. Reescribe aquellos que no cumplan el formato o la regla 50/72.

2. **Configuracion de proteccion de ramas**: En un repositorio de GitHub o GitLab, configura proteccion para la rama `main`: requiere PR con al menos 1 aprobacion, CI pasando, historial lineal, y sin push directo. Prueba que las reglas se aplican correctamente.

3. **Creacion de .gitattributes**: Crea un archivo `.gitattributes` para un proyecto multi-plataforma que normalice fines de linea, marque archivos binarios, configure merge drivers para `CHANGELOG.md` y `package-lock.json`, y excluya archivos de CI/CD de `git archive`.

4. **Simulacion de code review**: Escribe un PR que contenga 3 tipos de problemas: un bug logico, un problema de seguridad (secret hardcodeado), y codigo no idiomatico. Intercambialo con un compañero y revisenlo mutuamente usando el checklist de la seccion 18.7.2.

5. **Implementacion de pre-commit hooks**: Configura pre-commit con al menos 4 hooks: `trailing-whitespace`, `end-of-file-fixer`, `check-yaml`, y `detect-private-key`. Ejecutalo sobre un repositorio existente y corrige los problemas detectados.

---
