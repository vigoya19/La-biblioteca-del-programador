# Capítulo 3: GitHub Actions Internals y Sintaxis de Workflows

> "La automatización del pipeline no es un script de shell glorificado que corre a ciegas; es un sistema de orquestación reactivo distribuido que provisiona máquinas virtuales efímeras en milisegundos para compilar, probar y validar la integridad de tu software."

En la ingeniería de DevOps moderna, **GitHub Actions** se ha consolidado como el motor de CI/CD más influyente y utilizado del ecosistema de software debido a su profunda integración nativa con los repositorios de GitHub y su inmenso ecosistema de acciones reutilizables en el Marketplace.

Sin embargo, muchos desarrolladores configuran workflows copiando y pegando fragmentos de código del Marketplace sin comprender las tripas de la infraestructura: cómo se aprovisionan los runners físicos, cómo opera el aislamiento de los jobs concurrentes, cómo optimizar la velocidad mediante caché persistente y cómo escribir sintaxis YAML de grado producción. En este capítulo, desmitificaremos la arquitectura de GitHub Actions y diseñaremos un workflow optimizado para proyectos Node.js con TypeScript.

---

## 3.1 Arquitectura Física: GitHub-Hosted vs. Self-Hosted Runners

Cuando disparas un pipeline (por ejemplo, al empujar un commit a tu rama `main`), GitHub Actions provisiona un **Runner**, que es un agente físico de software encargado de ejecutar las instrucciones lógicas descritas en tu archivo YAML.

Existen dos tipos de Runners según la administración de la infraestructura:

### 1. GitHub-Hosted Runners (Alojados por GitHub)
* **Cómo funcionan**: Cada job individual de tu pipeline provisiona una **máquina virtual efímera completamente limpia y aislada** en la nube de Microsoft Azure (usando tecnologías de hypervisión ligeras).
* **Entornos**: Disponibles en Windows, macOS y Linux (Ubuntu).
* **Pros**: Cero mantenimiento, alta seguridad (la VM se destruye y borra físicamente del disco en cuanto termina el job) y escalado infinito concurrente nativo.
* **Contras**: Tiempo de arranque de la VM (handshake de red inicial de unos segundos) y costo de computación facturado por minuto de uso.

### 2. Self-Hosted Runners (Alojados por el Usuario)
* **Cómo funcionan**: Instalas el agente ligero de GitHub en tus propios servidores físicos, máquinas virtuales locales o contenedores Docker corriendo en tu propia nube (AWS, Azure, GCP o servidores Bare Metal locales).
* **Pros**: Control absoluto sobre el hardware (puedes usar CPUs potentes y GPUs para IA), acceso directo a tu red privada/VPC y almacenamiento caché local persistente ultra veloz sin cuotas de red.
* **Contras**: Responsabilidad total de la seguridad (los jobs sucesivos comparten el mismo sistema de archivos si no los limpias, arriesgando fugas de secretos) y coste operativo de mantenimiento.

---

## 3.2 Sintaxis Avanzada de Workflows YAML

Los archivos de configuración de GitHub Actions residen obligatoriamente en la ruta **`.github/workflows/`** y utilizan el formato descriptivo YAML.

### Los Tres Niveles Jerárquicos de Ejecución:
* **Workflow (Flujo de Trabajo)**: El proceso automatizado completo (representado por un archivo `.yml`). Se activa mediante eventos (**`on`**) como `push`, `pull_request` o cronogramas periódicos (`schedule`).
* **Jobs (Trabajos)**: Un conjunto de pasos ejecutados de forma secuencial en el **mismo runner físico**. Por defecto, **los diferentes jobs de un workflow corren de forma paralela en hilos aislados en runners separados**, a menos que definas dependencias utilizando la cláusula **`needs`** (ej. *el Job de Despliegue necesita que el Job de Tests culmine con éxito*).
* **Steps (Pasos)**: Tareas individuales secuenciales dentro de un mismo Job. Puede ser una instrucción de comandos de terminal (`run`) o una llamada a una plantilla reutilizable del Marketplace (`uses`).

---

> [!NOTE]
> ### 🤖 El Almacén Automatizado con Robots Clasificadores (GitHub Actions Runners)
> 
> Entendamos el comportamiento de los Runners, Jobs concurrentes y la caché de dependencias utilizando una analogía física cotidiana de logística:
> 
> - **El Runner Alojado (El Alquiler Temporal de Cabina de Trabajo)**:
>   - Imagina que necesitas armar un rompecabezas gigante de 10,000 piezas (tu proceso de compilación y testing).
>   - En lugar de comprar una mesa enorme y dejarla armada en tu pequeña casa, decides ir a un mega centro de alquiler de cabinas de trabajo vacías (**GitHub-Hosted Runners**).
>   - Llegas y te prestan una cabina de metal limpia de 3x3 metros con herramientas estándar listas en las paredes. Armas tu rompecabezas en 15 minutos, sacas una foto del resultado (**los artefactos**) y te vas. 
>   - Inmediatamente entra un equipo de limpieza que destruye la mesa, barre todo a la basura y desinfecta la cabina para el siguiente cliente. Tienes la certeza de que tu trabajo nunca se mezclará con el de los demás.
> 
> - **Los Jobs Concurrentes (El Trabajo en Equipo en Cabinas Aisladas)**:
>   - Tienes que armar 3 rompecabezas diferentes al mismo tiempo.
>   - En lugar de hacerlo todo tú solo en la misma mesa (**jobs secuenciales lentos**), el centro de logística te abre **3 cabinas de trabajo separadas y concurrentes** al mismo microsegundo con un robot operario en cada una. 
>   - Cada robot trabaja a máxima velocidad de forma independiente sin estorbarse. Si necesitas que el Robot 3 pegue los marcos del rompecabezas (Job de Despliegue), le configuras una regla: *"No procedas hasta que el Robot 1 y 2 te entreguen las piezas armadas por la ventanilla de comunicación"* (**la cláusula `needs`**).
> 
> - **La Caché de Dependencias (El Cajón de Herramientas de Uso Frecuente)**:
>   - Para armar el rompecabezas, necesitas descargar e instalar 500 mini herramientas especiales de Internet en cada visita (**los `node_modules`**). Descargar esto de la red en cada entrada a la cabina te consume 5 minutos completos de tu tiempo.
>   - Instalas un **Cajón Inteligente de Almacenamiento Persistente en el Lobby (la Acción `actions/cache`)**.
>   - Al terminar tu sesión de trabajo, el robot toma tu caja de herramientas, le pega una etiqueta con un código de barra único basado en tu lista de compras (**el hash del archivo `package-lock.json`**) y la guarda en el cajón del lobby.
>   - En tu siguiente visita, antes de encender Internet, el robot escanea tu lista de compras, busca la coincidencia exacta en el cajón del lobby, y recupera tu caja de herramientas local en 3 segundos, ahorrándote 5 minutos de descarga de red.

---

## 3.3 Código YAML de Producción: CI/CD Optimizado para TypeScript

A continuación, crearemos un archivo YAML de producción para automatizar la integración continua de un backend Node.js estructurado en TypeScript. El pipeline realiza triggers en `push` a la rama principal, implementa una estrategia de **Matrix Build** para probar en múltiples versiones de Node simultáneamente, y optimiza los tiempos de compilación al mínimo aplicando caché persistente nativa sobre `npm`:

### `integracionContinua.yml`
```yaml
name: Tubería de Integración Continua (TypeScript)

# 1. Configurar los eventos desencadenantes (Triggers) de forma defensiva
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # Job 1: Control de Estilo, Linters y Calidad Estática
  calidad-codigo:
    name: Análisis de Calidad y Linters
    runs-on: ubuntu-latest
    steps:
      - name: Clonar código fuente del repositorio
        uses: actions/checkout@v4

      - name: Configurar entorno de Node.js v20
        uses: actions/setup-node@v4
        with:
          node-version: 20

      # Utilizar la caché integrada de setup-node para node_modules basada en el package-lock.json
      - name: Aprovisionar dependencias con caché nativa
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Instalar dependencias limpias
        run: npm ci

      - name: Ejecutar validador de sintaxis y linters (ESLint)
        run: npm run lint

  # Job 2: Pruebas unitarias concurrentes en múltiples versiones de Node usando Matrix Strategy
  pruebas-unitarias:
    name: Test Unitarios en Node.js v${{ matrix.node-version }}
    needs: calidad-codigo # Este job se ejecuta únicamente si el job de linter pasa con éxito
    runs-on: ubuntu-latest
    strategy:
      # matrix permite probar el código en paralelo bajo múltiples configuraciones
      matrix:
        node-version: [ 18, 20, 22 ]
      # fail-fast = true cancelará todos los jobs de la matriz en curso si uno de ellos falla
      fail-fast: true

    steps:
      - name: Clonar código fuente
        uses: actions/checkout@v4

      - name: Configurar Node.js v${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Instalar dependencias limpias
        run: npm ci

      - name: Ejecutar suite de pruebas unitarias
        run: npm run test

  # Job 3: Compilación final y empaquetamiento de TypeScript a Javascript
  compilacion-empaquetamiento:
    name: Compilar TypeScript (build)
    needs: pruebas-unitarias
    runs-on: ubuntu-latest
    steps:
      - name: Clonar código fuente
        uses: actions/checkout@v4

      - name: Configurar Node.js v20
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      - name: Compilar código fuente TypeScript
        run: npm run build

      # Guardamos el resultado transaccional de la compilación en caliente para futuros jobs
      - name: Persistir artefacto de distribución (dist)
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: dist/
          retention-days: 7 # Purgar automáticamente el archivo en 7 días para ahorrar storage
```

---

## Resumen del Capítulo

* **GitHub Actions** provisiona entornos virtuales de ejecución aislados llamados **Runners**, que pueden ser efímeros y administrados (`GitHub-hosted`) o dedicados y privados (`Self-hosted`).
* Los workflows se dividen jerárquicamente en **Jobs** (que corren de forma paralela en runners independientes por defecto) y **Steps** (tareas secuenciales sobre la misma máquina).
* La directiva **`matrix`** nos permite escalar e internacionalizar nuestras pruebas ejecutándolas de forma concurrente sobre múltiples versiones de entornos de compilación.
* Implementar **caching de dependencias** en caliente basado en hashes de archivos de control (`package-lock.json`) es la optimización clave para reducir los tiempos de ciclo en producción.

En el próximo capítulo, escalaremos a tuberías avanzadas y multinivel estudiando la potencia de **GitLab CI/CD: Pipelines Multi-Project y Dynamic Pipelines** en producción.

---

[← Capítulo anterior (Capítulo 2)](02-estrategias-ramificacion-git.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 4) →](04-gitlab-cicd-pipelines.md)
