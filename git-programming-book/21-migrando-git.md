# Capítulo 21: Migrando a Git — Desde SVN, Mercurial y Perforce

> "Migrar de VCS es como mudarse de casa. Si no empaquetas bien, pierdes cosas en el camino."

## 21.1 El Principio Universal de Migración

Toda migración de VCS sigue el mismo patrón. La herramienta específica varía, pero el proceso mental es idéntico.

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│  FASES DE MIGRACIÓN (independiente del VCS origen)               │
│                                                                   │
│  1. ANÁLISIS: ¿Qué tenemos que migrar?                           │
│     • Ramas y tags                                                │
│     • Historial de commits (autores, fechas, mensajes)           │
│     • Metadatos (svn:ignore, svn:externals)                      │
│     • ¿Qué NO vamos a migrar? (binarios gigantes, builds)        │
│                                                                   │
│  2. PREPARACIÓN: Limpiar antes de empacar                        │
│     • Eliminar binarios y builds del historial                   │
│     • Unificar convenciones de nombres                           │
│     • Mapear autores (svn_user → Name <email>)                    │
│                                                                   │
│  3. CONVERSIÓN: Ejecutar la herramienta                          │
│     • Siempre sobre un clon/espejo, NUNCA sobre el original       │
│     • Verificar cada paso                                         │
│     • Documentar el mapeo y los comandos usados                  │
│                                                                   │
│  4. VERIFICACIÓN: ¿Migró todo correctamente?                     │
│     • Comparar conteo de commits                                  │
│     • Comparar árbol de archivos en HEAD                          │
│     • Verificar ramas y tags                                      │
│     • Validar autores y fechas                                    │
│                                                                   │
│  5. CORTE: El día D                                              │
│     • Congelar VCS origen (read-only)                             │
│     • Push final a Git                                            │
│     • Equipo clona el nuevo repo                                  │
│     • Primer commit en Git por cada dev                          │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 21.2 Migrando desde Subversion (SVN)

SVN fue el rey antes de Git. Millones de repositorios legacy siguen allí. La herramienta canónica es `git svn`.

### Paso 1: Mapear Autores SVN → Git

SVN usa usernames simples (`juan`, `maria`). Git usa `Nombre <email>`. Necesitas mapearlos.

```bash
# Crear archivo de mapeo
cat > authors.txt << 'EOF'
juan = Juan Pérez <juan.perez@empresa.com>
maria = María García <maria.garcia@empresa.com>
admin = Administrador Legacy <admin@empresa.com>
(no author) = Unknown <unknown@empresa.com>
EOF
```

### Paso 2: Clonar el Repositorio SVN

```bash
# Clonar con estructura estándar de SVN (trunk/branches/tags)
git svn clone https://svn.empresa.com/repos/mi-proyecto \
    --stdlayout \
    --authors-file=authors.txt \
    --prefix=svn/ \
    mi-proyecto-git

# Si la estructura no es estándar:
git svn clone https://svn.empresa.com/repos/mi-proyecto \
    --trunk=trunk \
    --branches=branches \
    --tags=tags \
    --authors-file=authors.txt \
    mi-proyecto-git

# ⚠️  Esto puede tomar HORAS o DÍAS para repos grandes.
# Cada commit SVN se reproduce como un commit Git.
```

### Paso 3: Convertir Ramas y Tags de SVN a Git Nativo

```bash
cd mi-proyecto-git

# git svn crea ramas remotas "svn/xxx". Hay que convertirlas.

# Convertir ramas:
for branch in $(git branch -r | grep 'svn/' | grep -v 'tags/' | grep -v 'trunk' | sed 's|svn/||'); do
    echo "Convirtiendo rama: $branch"
    git branch "$branch" "svn/$branch"
done

# Convertir tags (SVN tags son ramas; Git tags son objetos inmutables):
for tag in $(git branch -r | grep 'svn/tags/' | sed 's|svn/tags/||'); do
    echo "Convirtiendo tag: $tag"
    git tag -a -m "Tag $tag (migrado desde SVN)" "$tag" "svn/tags/$tag"
done

# Limpiar referencias remotas de SVN
git branch -r | grep 'svn/' | sed 's|svn/||' | while read ref; do
    git branch -rd "svn/$ref" 2>/dev/null
done
```

### Paso 4: Limpiar Artefactos SVN y Migrar Ignore

```bash
# Eliminar .svn directories si existen (no deberían, pero por si acaso)
find . -type d -name '.svn' -exec rm -rf {} + 2>/dev/null

# Convertir svn:ignore a .gitignore
git svn show-ignore > .gitignore
git add .gitignore
git commit -m "chore: convertir svn:ignore a .gitignore"
```

### Paso 5: Verificar la Migración

```bash
# Comparar número de commits
echo "SVN revisions: $(svn info https://svn.empresa.com/repos/mi-proyecto | grep 'Revision' | awk '{print $2}')"
echo "Git commits: $(git rev-list --count HEAD)"

# Verificar último commit: mismo contenido
svn export https://svn.empresa.com/repos/mi-proyecto/trunk /tmp/svn-head
diff -r /tmp/svn-head . --exclude='.git' --exclude='.svn'
# Sin diferencias = migración correcta

# Verificar ramas y tags
echo "Ramas:"
git branch
echo "Tags:"
git tag -l
```

### Paso 6: Push al Remoto Git

```bash
git remote add origin https://github.com/empresa/mi-proyecto.git
git push -u origin --all
git push --tags
```

### Problemas Comunes en Migración SVN

```
Problema 1: "Path not found" al clonar.
  Causa: El trunk no siempre está en /trunk.
  Solución: Usar --trunk=<ruta-correcta>.

Problema 2: Migración extremadamente lenta.
  Causa: git svn hace un fetch por cada revisión SVN.
  Solución: Si el historial tiene 50,000 revisiones, considera
            --revision START:END para migrar por partes,
            o evaluar si necesitas el historial completo.

Problema 3: Autores con caracteres especiales (ñ, ü, ç).
  Causa: El archivo authors.txt debe estar en UTF-8.
  Solución: file authors.txt → debe decir "UTF-8".

Problema 4: svn:externals.
  Causa: SVN permite "sub-repositorios" vía svn:externals.
  Solución: Migrar cada external como submódulo Git,
            o integrarlos en el monorepo, o usar git subtree.

Problema 5: Commits vacíos (solo propiedades SVN).
  git svn crea commits vacíos para cambios de svn:ignore, etc.
  Solución: git filter-repo --prune-empty=always después de migrar.
```

---

## 21.3 Migrando desde Mercurial (Hg)

Mercurial y Git comparten ADN (distribuidos, SHA-based, similares conceptualmente). La migración es más limpia que desde SVN.

### Usando hg-fast-export (Recomendado)

```bash
# 1. Clonar la herramienta
git clone https://github.com/frej/fast-export.git
cd fast-export

# 2. Crear un nuevo repo Git
git init mi-proyecto-git
cd mi-proyecto-git

# 3. Ejecutar la migración
../fast-export/hg-fast-export.sh \
    -r /ruta/al/repo/mercurial \
    --force

# 4. Verificar
git log --oneline
git branch
git tag -l

# 5. Hacer checkout de la rama default (Mercurial) → main (Git)
git checkout main 2>/dev/null || git checkout -b main default

# 6. Push
git remote add origin https://github.com/empresa/mi-proyecto.git
git push -u origin --all
git push --tags
```

### Mapeo de Autores Mercurial → Git

```bash
# Crear archivo de mapeo (mismo formato que SVN)
cat > authors.txt << 'EOF'
juan = Juan Pérez <juan.perez@empresa.com>
maria.garcia = María García <maria.garcia@empresa.com>
EOF

# Usar con hg-fast-export
hg-fast-export.sh -r /ruta/al/repo -A authors.txt --force
```

### Diferencias Conceptuales que Confunden al Equipo

```
┌─────────────────────────────────────────────────────────────────┐
│  Mercurial                         Git                          │
│  ─────────                         ───                          │
│  hg commit                         git commit + git push        │
│  (commit local = publicado)        (commit local ≠ remoto)      │
│                                                                    │
│  hg branches (nombres visibles)    git branch (local remoto)    │
│  Ramas son parte del historial     Ramas son punteros ligeros   │
│                                                                    │
│  hg log (limitado)                 git log (extremadamente      │
│                                    flexible con --format)        │
│                                                                    │
│  hg revert                         git restore / git reset       │
│  (comportamiento diferente)        (más granular, más opciones)  │
│                                                                    │
│  No tiene staging area             git add (staging area)        │
│  (commit = todo modificado)        (commit = lo que eliges)     │
│                                                                    │
│  Bookmarks ≈ Git branches          Branches son ciudadanos       │
│  (en Mercurial son opcionales)     de primera clase en Git       │
└─────────────────────────────────────────────────────────────────┘
```

### Checklist de Preparación del Equipo Mercurial

```
Antes del corte:

[ ] Workshop: "Conceptos de Git para usuarios de Mercurial" (1 día).
[ ] El equipo instala Git y configura user.name/user.email.
[ ] Se comparte el nuevo URL del repo Git y se verifica acceso.
[ ] Se documenta el cheat sheet "Hg → Git" (equivalencias de comandos).
[ ] Los hooks de CI/CD se migran primero (Jenkins/GitHub Actions).
[ ] Se congela el repo Mercurial (read-only) el día del corte.
[ ] Primer sprint en Git: expectativa de fricción. Dar soporte extra.
```

---

## 21.4 Migrando desde Perforce (P4)

Perforce (Helix Core) es el estándar en la industria AAA de videojuegos y en algunas empresas enterprise. Migrar a Git es el movimiento más complejo de los tres, principalmente por el tamaño de los repositorios (frecuentemente >100 GB) y porque Perforce no es distribuido.

### ¿Migrar de Perforce a Git es la Decisión Correcta?

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  ¿Cuándo SÍ migrar de Perforce a Git?                        │
│                                                               │
│  ✓ El repositorio es <10 GB total.                           │
│  ✓ Mayoría de archivos son texto (no solo binarios).         │
│  ✓ Equipo <50 personas.                                      │
│  ✓ Necesitan branching/merging más ágil.                     │
│  ✓ Quieren PR workflows modernos (GitHub/GitLab).            │
│                                                               │
│  ¿Cuándo NO migrar?                                          │
│                                                               │
│  ✗ Repositorio >100 GB de assets binarios.                   │
│  ✗ Equipo >100 artistas que dependen de file locking de P4. │
│  ✗ Infraestructura de CI/CD altamente acoplada a P4.        │
│  ✗ No hay presupuesto para Git LFS hosting masivo.          │
│                                                               │
│  Considerar: Git + Git LFS para código, mantener P4 para     │
│  assets. O evaluar Plastic SCM (Unity) como puente.          │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Usando git-p4 (Herramienta Oficial de Git)

```bash
# 1. Clonar de Perforce
git p4 clone //depot/mi-proyecto/...@all mi-proyecto-git

# @all = trae todo el historial. Para repos grandes, usar @YYYY/MM/DD
# para limitar la profundidad histórica.

# 2. Si el depot tiene múltiples subdirectorios:
git p4 clone --detect-branches //depot/mi-proyecto/...@all .

# 3. Mapeo de usuarios P4 → Git
# git-p4 usa el archivo de mapeo automáticamente si existe
cat > ~/.gitp4usermap << 'EOF'
juanp Juan Pérez <juan.perez@empresa.com>
mariag María García <maria.garcia@empresa.com>
EOF
```

### Estrategia para Repositorios Perforce Enormes

```
Problema: Repositorio de 500 GB con 15 años de historial.
Solución: Estrategia de migración parcial.

FASE 1: Migrar solo el historial reciente (últimos 2 años)
  git p4 clone //depot/...@2022/01/01,@now .
  (2 años de historial en Git para trabajo diario)

FASE 2: Archivar el historial antiguo
  git p4 clone //depot/...@2009/01/01,2022/01/01 repo-historico.git
  Mantener como referencia read-only en un servidor aparte.

FASE 3: Para auditorías que requieran historial completo:
  Clonar repo-historico.git (solo cuando se necesita).
```

### Post-Migración: Reemplazar File Locking de Perforce

```bash
# Perforce usa "p4 edit" para bloquear archivos (checkout exclusivo).
# En Git, el equivalente es Git LFS file locking:

# Activar locking en archivos binarios
cat >> .gitattributes << 'EOF'
*.psd filter=lfs diff=lfs merge=lfs -text lockable
*.fbx filter=lfs diff=lfs merge=lfs -text lockable
*.uasset filter=lfs diff=lfs merge=lfs -text lockable
EOF

# Workflow: antes de editar un binario
git lfs lock archivo.psd
# ... editar ...
git add archivo.psd
git commit -m "art: actualizar textura"
git push
# El push DESBLOQUEA automáticamente (git lfs unlock)
```

### Checklist Post-Migración P4 → Git

```
[ ] Todos los changelists de P4 se convirtieron en commits de Git.
[ ] Los autores están correctamente mapeados (Name <email>).
[ ] Los archivos binarios grandes usan Git LFS (verificar .gitattributes).
[ ] Las ramas de P4 se convirtieron a ramas de Git.
[ ] Los labels de P4 se convirtieron a tags de Git.
[ ] El file locking está configurado en .gitattributes para binarios.
[ ] Los hooks de CI/CD funcionan contra el nuevo repo Git.
[ ] El equipo recibió training "P4 → Git" antes del corte.
[ ] Perforce se dejó en modo read-only (por si acaso).
```

---

## 21.5 Plantilla de Correo de Anuncio de Migración

```markdown
Asunto: Migración de [SVN/Mercurial/Perforce] a Git — Fecha: [FECHA]

Hola equipo,

El [FECHA] migraremos nuestro control de versiones a Git.

═══ ¿POR QUÉ? ═══
• Pull requests con code review integrado.
• Ramas más rápidas y ligeras.
• Integración nativa con GitHub Actions / GitLab CI.
• Ecosistema moderno (LFS para archivos grandes, GitHub Codespaces).

═══ ¿CUÁNDO? ═══
• [FECHA-1 semana]: Workshop "Introducción a Git" (1 hora).
• [FECHA-3 días]: El repositorio Git estará disponible en modo lectura.
• [FECHA-DÍA-D]: CORTE. El repo [VCS-origen] pasa a read-only.
  El repo Git es el nuevo source of truth.

═══ ¿QUÉ NECESITO HACER? ═══
1. Asistir al workshop (obligatorio).
2. Antes del corte: pushear TODO tu trabajo a [VCS-origen].
3. Después del corte: clonar el nuevo repo Git.
4. Leer la guía de migración: [LINK A WIKI/CONFLUENCE]

═══ ¿DÓNDE PIDO AYUDA? ═══
• Canal Slack: #git-migration
• Office hours: [FECHAS Y HORARIOS] con el equipo de DevOps/Arquitectura.

Saludos,
[Nombre del Arquitecto]
```

---

> **Reflexión del capítulo**: Una migración de VCS exitosa no se mide por la herramienta que usaste para convertir los commits. Se mide por cuántos días tardó el equipo en volver a ser productivo después del corte. La diferencia entre una migración traumática y una fluida está en: el mapeo de autores, la preparación del equipo, el plan de rollback, y la comunicación constante. La herramienta de conversión es el 20% del trabajo. El 80% es gestión del cambio.

---

← [Capítulo anterior](20-activos-no-codigo.md) | [Inicio](README.md) | [Capítulo siguiente →](A-comandos.md)
