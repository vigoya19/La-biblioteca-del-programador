# Capítulo 20: Git para Activos No-Código — Juegos, Diseño, Datos y Documentación

> "Git no es solo para código fuente. Es un sistema de control de versiones para cualquier archivo digital. Pero cada tipo de archivo requiere su propia estrategia."

## 20.1 El Mito: "Git No Sirve para Archivos Grandes o Binarios"

Git fue diseñado para código fuente (archivos de texto). Y es verdad que los archivos binarios masivos (>100MB) degradan el rendimiento severamente. Pero eso no significa que no puedas usar Git en proyectos con assets. Significa que necesitas las herramientas y estrategias correctas.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ¿Deberías usar Git para...                                     │
│                                                                  │
│  Código fuente:          ✓✓✓  El propósito original.            │
│  Documentación (md/rst): ✓✓   Perfecto. Docs-as-code.           │
│  Archivos de diseño:     ✓    Con Git LFS o repos separados.    │
│  Datasets pequeños (<10MB): ✓   CSV, JSON lines, Parquet.       │
│  Modelos ML (<100MB):    ✓    Con LFS. Sin LFS si son pocos.    │
│  Assets de videojuegos:  ✓    Con Git LFS + locking.            │
│  Videos y audio:         ⚠    Con LFS. Sin LFS: NO.             │
│  Binarios >500MB:        ✗    Usar almacenamiento externo.      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 20.2 Git para Desarrollo de Videojuegos

El desarrollo de juegos es el escenario más exigente para Git: repositorios de 50-200 GB, miles de archivos binarios (texturas, modelos 3D, animaciones, audio), y equipos de artistas + programadores con necesidades opuestas.

### El Problema del Merge en Archivos Binarios

```bash
# Situación: Dos artistas modifican el mismo archivo .blend (Blender) o .uasset (Unreal).

# Programador A modifica player.blend (cambiando el rigging)
# Programador B modifica player.blend (cambiando la textura)

# git merge: CONFLICTO BINARIO. Git no puede hacer merge de binarios.
# Resultado: Uno de los dos pierde su trabajo. O hay que rehacerlo manualmente.

# Esto NO es culpa de Git. Es la naturaleza de los archivos binarios.
# La solución: GIT LFS FILE LOCKING.
```

### Estrategia 1: Git LFS + File Locking (La Solución Estándar)

```bash
# 1. Instalar Git LFS
git lfs install

# 2. Definir qué archivos van a LFS (archivos binarios mergeables NO)
cat > .gitattributes << 'EOF'
# Assets del juego: LFS + bloqueo (no mergeables)
*.psd filter=lfs diff=lfs merge=lfs -text lockable
*.blend filter=lfs diff=lfs merge=lfs -text lockable
*.fbx filter=lfs diff=lfs merge=lfs -text lockable
*.png filter=lfs diff=lfs merge=lfs -text lockable
*.tga filter=lfs diff=lfs merge=lfs -text lockable
*.wav filter=lfs diff=lfs merge=lfs -text lockable
*.mp3 filter=lfs diff=lfs merge=lfs -text lockable
*.uasset filter=lfs diff=lfs merge=lfs -text lockable

# Archivos de texto/código: Git normal (sí mergeables)
*.cpp text
*.h text
*.cs text
*.py text
*.json text
EOF

git add .gitattributes
git commit -m "config: Git LFS + file locking para assets del juego"
```

### El Flujo de Trabajo con File Locking

```bash
# ─── DÍA A DÍA DEL ARTISTA ───

# 1. Antes de editar: BLOQUEAR el archivo
git lfs lock player-model.fbx
# "player-model.fbx locked by ana@studio.com"

# Si alguien más ya lo tiene bloqueado:
# git lfs lock player-model.fbx
# ERROR: already locked by carlos@studio.com

# 2. Ver quién tiene bloqueado qué
git lfs locks
# player-model.fbx  ana@studio.com    10 minutes ago
# boss-texture.psd  carlos@studio.com  2 hours ago

# 3. Editar el archivo con tranquilidad.
#    Sabes que NADIE más lo está tocando.

# 4. Commit y push como siempre
git add player-model.fbx
git commit -m "art: actualizar rig del personaje principal"
git push

# 5. Desbloquear (automático con push, o manual)
git lfs unlock player-model.fbx
# "player-model.fbx unlocked"

# ─── ROMPER UN BLOQUEO (con autorización) ───
# Si alguien bloqueó algo y se fue de vacaciones:
git lfs unlock --force player-model.fbx
# ⚠️  Usar solo en emergencia y con comunicación al equipo
```

### Comparativa: Unity vs Unreal Engine con Git

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  UNITY:                                                          │
│                                                                  │
│  ✓ Usa archivos de texto para escenas (.unity = YAML)            │
│  ✓ Prefabs y assets serializados como texto (configurable)      │
│  ✓ Merge de escenas parcialmente posible (con cuidado)           │
│                                                                  │
│  Configuración necesaria:                                        │
│    Edit → Project Settings → Editor → Asset Serialization       │
│    Modo: Force Text                                              │
│                                                                  │
│  .gitignore recomendado:                                         │
│    /Library/                                                     │
│    /Temp/                                                        │
│    /Obj/                                                         │
│    /Build/                                                       │
│    .vs/                                                          │
│    *.csproj                                                      │
│    *.sln                                                         │
│                                                                  │
│  .gitattributes:                                                 │
│    *.unity text merge=unityyamlmerge                             │
│    *.prefab text merge=unityyamlmerge                            │
│    *.asset text merge=unityyamlmerge                             │
│    *.mat text                                                    │
│    *.png binary                                                  │
│    *.fbx binary                                                  │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  UNREAL ENGINE:                                                  │
│                                                                  │
│  ✗ .uasset son binarios. NO mergeables.                         │
│  ✗ Repositorios ENORMES (>50 GB típico).                         │
│                                                                  │
│  Estrategia recomendada:                                         │
│    1. Git LFS para TODOS los archivos de /Content/               │
│    2. File locking OBLIGATORIO para archivos .uasset             │
│    3. Uso de Unreal Game Sync (UGS) para distribución interna   │
│    4. Considerar Perforce para equipos >10 personas               │
│       (industria AAA estándar, mejor manejo de binarios)         │
│                                                                  │
│  .gitignore:                                                     │
│    /Binaries/                                                    │
│    /Intermediate/                                                │
│    /DerivedDataCache/                                            │
│    /Saved/                                                       │
│    *.pdb                                                         │
│                                                                  │
│  Alternativa: Unreal + Perforce (estándar AAA)                   │
│  Alternativa: Unreal + Plastic SCM (Unity)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 20.3 Git para Diseñadores — Figma, Photoshop y Assets Visuales

Los diseñadores viven en herramientas visuales. Su "código fuente" no es texto. Pero necesitan versionado tanto como los devs. Aquí va la estrategia por herramienta.

### Figma: Versionado Incorporado (No Necesitas Git)

```
Figma tiene historial de versiones nativo (⌘⌥S para guardar versión).
No exportes PNGs a Git para "versionar" diseños.

Estrategia:
  • Figma: versionado de diseños.
  • Git: versionado de tokens de diseño (design-tokens.json),
         documentación de componentes, specs de handoff.

Ejemplo de repo de diseño:
  design-system/
  ├── tokens/
  │   ├── colors.json
  │   ├── typography.json
  │   └── spacing.json
  ├── components/
  │   ├── button.md
  │   └── modal.md
  └── README.md
```

### Photoshop / Illustrator / Sketch: Git LFS

```bash
# Los archivos .psd, .ai, .sketch van a Git LFS

cat >> .gitattributes << 'EOF'
*.psd filter=lfs diff=lfs merge=lfs -text lockable
*.ai filter=lfs diff=lfs merge=lfs -text lockable
*.sketch filter=lfs diff=lfs merge=lfs -text lockable
*.xd filter=lfs diff=lfs merge=lfs -text lockable
EOF
```

**El truco para diffs visuales de diseño**:

```bash
# Configurar un diff tool que muestre los archivos visualmente
# (no el diff binario, que es inútil para humanos)

# En .git/config:
[diff "psd"]
    textconv = ./scripts/diff-psd.sh
    cachetextconv = true

# scripts/diff-psd.sh:
#!/bin/bash
# Convierte PSD a PNG para que el diff sea visual
convert "$1[0]" -resize 800x800 PNG:-

# Ahora git diff player.psd muestra una miniatura PNG en vez de basura binaria
```

---

## 20.4 Git para Científicos de Datos — Notebooks, Datasets y Modelos

El Jupyter notebook (.ipynb) es el peor enemigo de Git. Cada ejecución cambia metadatos (execution_count, timestamps), generando conflictos falsos en cada merge. Y los datasets/modelos son archivos grandes que no pertenecen a Git.

### Jupyter Notebooks: La Guerra Puede Terminar

```bash
# 1. Limpiar outputs antes de commit (AUTOMÁTICO con filtro)

cat >> .gitattributes << 'EOF'
*.ipynb filter=nbstripout
EOF

# Instalar nbstripout
pip install nbstripout
nbstripout --install

# Ahora cada git add limpia los outputs automáticamente.
# El notebook en Git solo contiene código, no outputs ni metadatos volátiles.

# 2. Para diffs legibles de notebooks:
# Configurar Jupyter Lab con extensión de Git
# jupyter labextension install @jupyterlab/git

# O usar nbdime (herramienta oficial de Jupyter)
pip install nbdime
nbdime config-git --enable --global

# Ahora git diff de notebooks muestra diferencias semánticas
# (qué celdas cambiaron, outputs nuevos), no JSON ilegible.
```

### Datasets: NUNCA en Git Directamente

```bash
# Estrategia por tamaño:

# <1 MB: CSV, JSON lines en Git es aceptable
# 1-10 MB: Git LFS
# 10-100 MB: Git LFS + .gitignore parcial
# >100 MB: DVC (Data Version Control) o almacenamiento externo

# ─── DVC: Git Para Datos ───
# DVC versiona datasets como Git versiona código

pip install dvc
dvc init
git commit -m "Initialize DVC"

# Agregar un dataset al versionado
dvc add data/training.csv
# Crea data/training.csv.dvc (puntero ligero en Git)
# El archivo real va a .dvc/cache/

git add data/training.csv.dvc .gitignore
git commit -m "data: agregar dataset de entrenamiento v1"

# Sincronizar datasets con almacenamiento remoto
dvc remote add -d storage s3://mi-bucket/datasets
dvc push  # Sube el dataset a S3

# Otro cientifico clona y obtiene los datos:
git clone <repo>
dvc pull  # Descarga el dataset exacto que usó el commit actual

# Cambiar el dataset:
dvc add data/training.csv  # Nueva versión
git commit -m "data: actualizar dataset (más datos de Perú)"

# Reproducibilidad total: cada commit de Git apunta
# a una versión específica del dataset.
```

---

## 20.5 Git para Documentación — Docs-as-Code

Tratar la documentación como código es una de las mejores decisiones que puede tomar un equipo.

```
Beneficios:
  ✓ La documentación se versiona junto al código que documenta.
  ✓ PRs incluyen cambios de documentación (sin excusa).
  ✓ CI/CD publica la documentación automáticamente al mergear.
  ✓ Toda la potencia de Git (blame, history, diff) en tus docs.
```

### Estructura de Monorepo con Documentación

```
proyecto/
├── docs/                    ← Documentación (Git normal)
│   ├── architecture/        ← ADRs, diagramas C4
│   │   ├── adr-001.md
│   │   └── c4-context.puml
│   ├── api/                 ← OpenAPI specs
│   │   └── openapi.yaml
│   └── guides/              ← Guías para desarrolladores
│       ├── getting-started.md
│       └── contributing.md
├── src/                     ← Código fuente
└── README.md
```

### CI/CD para Documentación — Publicación Automática

```yaml
# .github/workflows/docs.yml
name: Deploy Documentation
on:
  push:
    branches: [main]
    paths:
      - 'docs/**'
      - 'README.md'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Ejemplo con MkDocs (Python)
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install mkdocs-material
      - run: mkdocs build
      
      # Publicar a GitHub Pages
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./site
      
      # O publicar a Netlify, Vercel, S3, etc.
```

### Revisión de Documentación como Código

```markdown
<!-- PR template con checklist de documentación -->

## Documentación
- [ ] He actualizado los docs relevantes en /docs/
- [ ] Los ejemplos de código en los docs funcionan
- [ ] He revisado con `markdownlint` los cambios en .md
- [ ] Los diagramas C4/PlantUML renderizan correctamente
- [ ] Las specs OpenAPI pasan validación
```

---

## 20.6 Git para Bases de Datos — Migraciones y Esquemas

```bash
# Las migraciones de BD SON CÓDIGO. Deben estar en Git.

# Estructura típica:
migrations/
├── 001_create_users.sql
├── 002_add_email_column.sql
├── 003_create_orders.sql
└── ...

# Estrategia: Expand and Contract (cero downtime)
# Capítulo 20 del libro de Arquitectura para más detalle

# Validación de migraciones en CI:
# .github/workflows/db-check.yml
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check migration naming convention
        run: |
          for file in migrations/*.sql; do
            if ! [[ $(basename "$file") =~ ^[0-9]{3}_[a-z_]+\.sql$ ]]; then
              echo "ERROR: $file no sigue el formato ###_descripcion.sql"
              exit 1
            fi
          done
```

---

> **Reflexión del capítulo**: Git no discrimina tipos de archivo. Almacena bytes. La diferencia entre un proyecto con assets que funciona bien en Git y uno que es un infierno está en la configuración: `.gitattributes`, Git LFS, file locking, y las herramientas de diff adecuadas para cada formato. Un game studio que domina Git LFS + file locking puede manejar terabytes de assets. Un equipo de datos que ignora DVC llena el repo de CSVs de 500MB y hace el clone imposible. La herramienta es la misma. La diferencia es saber usarla.
