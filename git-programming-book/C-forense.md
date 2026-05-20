# Apéndice C: Git Forense y Auditoría — Respondiendo las Preguntas Difíciles

> "En una auditoría, no puedes decir 'no sé'. Git te da las respuestas, si sabes dónde buscar."

## C.1 ¿Cuándo Necesitas Git Forense?

```
Escenarios reales donde necesité Git forense en mi carrera:

1. "Recibimos una solicitud de auditoría SOC2. Necesitan saber quién
   aprobó y mergeó el PR que desplegó la versión v3.2.1 el 15 de marzo."

2. "El desarrollador Alex renunció ayer. Necesito saber qué archivos
   modificó en sus últimos 30 días para revisar su trabajo."

3. "¿En qué commit exactamente se introdujo la vulnerabilidad de
   SQL injection en el módulo de búsqueda?"

4. "El cliente dice que borramos sus datos sin permiso. Necesito
   probar que el endpoint de eliminación existe desde 2021 y
   fue aprobado por el Product Manager en el PR #2345."

5. "Estamos en due diligence para una adquisición. El comprador
   quiere saber cuántas personas contribuyeron al repositorio core,
   quiénes son, y con qué frecuencia."
```

---

## C.2 Kit de Herramientas Forenses

```bash
# Tu navaja suiza forense: alias que necesitas

alias git-authors="git shortlog -sne --all"
alias git-file-history="git log --all --follow --format='%h %ad %an: %s' --date=short"
alias git-whatchanged="git log --all --format='' --name-only | sort | uniq -c | sort -rn | head"
alias git-recent-branches="git for-each-ref --sort=-committerdate refs/heads/ --format='%(committerdate:short) %(refname:short)'"
alias git-loc-per-author="git ls-files | xargs -n1 git blame --line-porcelain | grep '^author ' | sort | uniq -c | sort -rn"
```

---

## C.3 Respondiendo las 10 Preguntas Forenses Más Comunes

### 1. ¿Quiénes contribuyeron al repositorio y cuánto?

```bash
# Por número de commits
git shortlog -sne --all | sort -rn

# Por líneas de código (aproximado)
git log --all --format='%aE' --numstat | awk '
    NF==3 { plus[$1]+=$1; minus[$1]+=$2 }
    NF==1 { author=$1 }
    END { for (a in plus) printf "%d +%d -%d %s\n", plus[a]+minus[a], plus[a], minus[a], a }
' | sort -rn

# Por fecha de primera y última contribución
git log --all --format='%aE %ad' --date=short | awk '
    !first[$1] { first[$1] = $2 }
    { last[$1] = $2 }
    END { for (a in first) printf "%s\t%s → %s\n", a, first[a], last[a] }
' | sort
```

### 2. ¿Qué archivos son los más modificados? (Hotspots)

```bash
# Top 20 archivos más modificados en el último año
git log --since="1 year ago" --format="" --name-only --all | \
    sort | uniq -c | sort -rn | head -20

# Lo mismo pero por módulo/directorio
git log --since="1 year ago" --format="" --name-only --all | \
    sed 's|/[^/]*$||' | sort | uniq -c | sort -rn | head -20
```

### 3. ¿Cuándo se introdujo una string o patrón específico?

```bash
# Encontrar el primer commit que introdujo "API_SECRET"
git log --all -S "API_SECRET" --reverse --format="%h %ad %an: %s" --date=short | head -1

# Encontrar el commit que ELIMINÓ una función
git log --all -S "def deprecated_function" --diff-filter=D \
    --format="%h %ad %an: %s" --date=short

# Buscar con regex más complejo (ej: claves API de 32 caracteres hex)
git log --all -G "[0-9a-f]{32}" --format="%h %ad %an: %s" --date=short
```

### 4. ¿Quién aprobó un merge específico?

```bash
# El autor del merge commit (quien hizo git merge o presionó "Merge PR")
git log --merges --format="%h %ad %an: %s" --date=short | head -20

# Encontrar el merge commit de un PR específico (si usan GitHub #ID en mensajes)
git log --all --merges --grep="#2345" --format="%H %ad %an: %s" --date=iso

# Ver los commits que forman parte del merge
git log <merge-commit>^..<merge-commit> --oneline
```

### 5. ¿Cuál era el estado del repositorio en una fecha/hora exacta?

```bash
# SHA del commit que era HEAD en main el 15 de marzo a las 14:00 UTC
AUDIT_COMMIT=$(git rev-list -n1 --before="2024-03-15 14:00:00 UTC" main)
echo "HEAD era: $AUDIT_COMMIT"

# Ver el árbol completo de archivos en esa fecha
git ls-tree -r --name-only $AUDIT_COMMIT

# ¿Quién era responsable de cada archivo en esa fecha?
git ls-tree -r --name-only $AUDIT_COMMIT | while read file; do
    author=$(git log -1 --format="%an" $AUDIT_COMMIT -- "$file")
    echo "$author: $file"
done

# Ver el contenido de un archivo específico en esa fecha
git show ${AUDIT_COMMIT}:ruta/al/archivo.conf
```

### 6. ¿Quién tocó este archivo sensible y cuándo?

```bash
# Historial completo de cambios en auth/secrets.js
git log --all --follow --format="%h %ad %an: %s" --date=iso -- auth/secrets.js

# Con diff incluido (para ver EXACTAMENTE qué cambió cada vez)
git log --all --follow -p -- auth/secrets.js

# Solo cambios en un rango de fechas
git log --all --follow --since="2024-01-01" --until="2024-03-31" \
    --format="%h %ad %an: %s" --date=short -- auth/secrets.js
```

### 7. ¿El desarrollador que renunció dejó código sin revisar?

```bash
DEVELOPER="alex.garcia@empresa.com"
LAST_30_DAYS="30 days ago"

echo "=== Commits de $DEVELOPER en los últimos 30 días ==="
git log --all --author="$DEVELOPER" --since="$LAST_30_DAYS" \
    --format="%h %ad %s" --date=short

echo ""
echo "=== Archivos modificados ==="
git log --all --author="$DEVELOPER" --since="$LAST_30_DAYS" \
    --format="" --name-only | sort -u

echo ""
echo "=== Líneas añadidas/eliminadas ==="
git log --all --author="$DEVELOPER" --since="$LAST_30_DAYS" \
    --shortstat --format="" | awk '
    /files changed/ { files += $1 }
    /insertions/ { ins += $4 }
    /deletions/ { del += $6 }
    END { printf "Archivos: %d | +%d -%d líneas\n", files, ins, del }
'

echo ""
echo "=== Ramas locales de $DEVELOPER ==="
git log --all --author="$DEVELOPER" --since="$LAST_30_DAYS" \
    --format="%D" | tr ',' '\n' | grep -v '^$' | sort -u
```

### 8. ¿Un archivo fue modificado fuera del proceso de PR?

```bash
# Commits directos a main (sin merge) — posible violación de proceso
git log main --no-merges --format="%h %ad %an: %s" --date=short | head -20

# Commits directos a main en el último mes (excluyendo merges de PR)
git log main --no-merges --since="1 month ago" \
    --format="%h %ad %an: %s" --date=short
```

### 9. ¿Qué cambios se desplegaron en la versión v2.5.0?

```bash
# Si tienes un tag v2.5.0, ver qué cambió desde v2.4.0
git log v2.4.0..v2.5.0 --format="%h %ad %an: %s" --date=short

# Obtener el changelog automático entre dos versiones
git log v2.4.0..v2.5.0 --format="- %s (%an)" --date=short

# Solo las rutas de archivo modificadas (sin duplicados)
git diff --name-only v2.4.0..v2.5.0 | sort

# Estadísticas resumidas
git diff --stat v2.4.0..v2.5.0
```

### 10. ¿Hay archivos con permisos incorrectos en el historial?

```bash
# Buscar archivos con permisos 755+ (ejecutables) que no deberían serlo
git log --all --format="" --name-only --diff-filter=A | \
    sort -u | while read file; do
    mode=$(git ls-files -s "$file" 2>/dev/null | awk '{print $1}')
    if [ -n "$mode" ] && [ "${mode:0:3}" != "100" ]; then
        echo "$mode $file"
    fi
done

# Buscar cambios de permiso sospechosos
git log --all --diff-filter=M --summary | grep "mode change"
```

---

## C.4 Auditoría de Cumplimiento SOC2 / ISO 27001

```bash
#!/bin/bash
# compliance-audit.sh: Script para auditoría de cumplimiento
# Genera evidencia para SOC2, ISO 27001, PCI-DSS

REPO_NAME=$(basename $(git rev-parse --show-toplevel))
AUDIT_DATE=$(date -I)
REPORT="compliance-report-${REPO_NAME}-${AUDIT_DATE}.md"

cat > "$REPORT" << EOF
# Reporte de Auditoría Git — $REPO_NAME
**Fecha de generación:** $AUDIT_DATE

## 1. Resumen del Repositorio
EOF

# Información básica
echo "- **Repositorio:** $REPO_NAME" >> "$REPORT"
echo "- **Rama principal:** $(git symbolic-ref refs/remotes/origin/HEAD | sed 's|.*/||')" >> "$REPORT"
echo "- **Total commits:** $(git rev-list --count --all)" >> "$REPORT"
echo "- **Total contribuidores:** $(git shortlog -sne --all | wc -l | tr -d ' ')" >> "$REPORT"
echo "- **Primer commit:** $(git log --reverse --format='%ad' --date=short | head -1)" >> "$REPORT"
echo "- **Último commit:** $(git log -1 --format='%ad' --date=short)" >> "$REPORT"

cat >> "$REPORT" << EOF

## 2. Lista de Contribuidores (Identidades Verificadas)
EOF

echo '```' >> "$REPORT"
git shortlog -sne --all | sort -rn >> "$REPORT"
echo '```' >> "$REPORT"

cat >> "$REPORT" << EOF

## 3. Historial de Merges (Control de Cambios)
EOF

echo '```' >> "$REPORT"
git log --merges --since="1 year ago" --format="%h %ad %an: %s" --date=short >> "$REPORT"
echo '```' >> "$REPORT"

cat >> "$REPORT" << EOF

## 4. Tags y Releases
EOF

echo '```' >> "$REPORT"
git tag -l --format='%(refname:short) %(taggerdate:short) %(subject)' --sort=-taggerdate >> "$REPORT"
echo '```' >> "$REPORT"

cat >> "$REPORT" << EOF

## 5. Integridad del Repositorio
EOF

echo '```' >> "$REPORT"
git fsck --full --no-dangling 2>&1 >> "$REPORT"
echo '```' >> "$REPORT"

echo "" >> "$REPORT"
echo "---" >> "$REPORT"
echo "*Reporte generado automáticamente con compliance-audit.sh*" >> "$REPORT"

echo "✓ Reporte generado: $REPORT"
```

---

> **Reflexión del apéndice**: Git es una base de datos de cada cambio que ha ocurrido en tu código. Como arquitecto, tu trabajo es saber extraer la verdad de esa base de datos cuando el negocio, los auditores o los abogados la necesitan. El forense no es paranoia. Es preparación. Porque el día que necesites responder "¿quién tocó este archivo y cuándo?", la respuesta no puede ser "déjame revisar".
