# Capítulo 8: Dockerización y Pipelines de Contenedores

> "Un contenedor no es una máquina virtual ligera con su propio sistema operativo; es un conjunto de procesos de Linux aislados mediante namespaces y cgroups que comparten el mismo kernel físico del host. Escribir imágenes gigas cargadas de dependencias innecesarias es una irresponsabilidad que compromete la velocidad y la seguridad de tu infraestructura."

En la ingeniería de DevOps, **Docker** se ha convertido en el estándar indiscutible para empaquetar software. Permite que una aplicación se ejecute con total predictibilidad en cualquier máquina, eliminando el clásico problema de: *"En mi máquina local sí funciona"*.

Sin embargo, muchos desarrolladores escriben Dockerfiles ineficientes, generando imágenes de más de 1GB que tardan minutos en descargarse por red, contienen vulnerabilidades críticas del sistema operativo y ralentizan los despliegues. En este capítulo, aprenderemos a construir **Dockerfiles Multi-Stage optimizados** para Node.js con TypeScript, y configuraremos un pipeline de CI/CD que implementa **Docker Layer Caching** de alto rendimiento en GitHub Actions.

---

## 8.1 Optimización Multi-Stage: Imágenes de 1GB a 100MB

El principio fundamental para empaquetar software de forma segura y veloz es **reducir la superficie de ataque y el peso en disco de la imagen final**. Un Dockerfile tradicional arrastra herramientas de compilación, linters y dependencias de desarrollo (`devDependencies`) que son inútiles en caliente.

### La técnica de Multi-Stage Builds (Compilación Multietapa):
Permite definir múltiples cláusulas `FROM` independientes dentro del mismo Dockerfile, creando etapas lógicas de transición en caliente:
1. **Etapa 1: Build (El Taller de Carpintería)**: Utilizamos una imagen base completa (`node:20-alpine`) equipada con compiladores de TypeScript, linters y herramientas de desarrollo. Copiamos el código fuente, instalamos absolutamente todo, ejecutamos la compilación (`npm run build`) y obtenemos el directorio distributivo `dist/`.
2. **Etapa 2: Runner (El Salón de Exhibición Limpio)**: Iniciamos con una imagen base minúscula e impecable (`node:20-alpine`). Copiamos **únicamente** los archivos JavaScript compilados de la Etapa 1 y las dependencias estrictas de producción (`npm ci --only=production`).
* **Resultado**: La imagen final está completamente libre de compiladores pesados y dependencias de desarrollo, reduciendo el peso de disco de $1.2\text{GB}$ a apenas unos $100\text{MB}$ y mitigando vulnerabilidades críticas.

---

## 8.2 Docker Layer Caching en Pipelines de CI/CD

Cada comando (`RUN`, `COPY`, `ADD`) en tu Dockerfile genera una **Capa física de solo lectura (Layer)** en disco. Si los archivos referenciados en un comando no cambian, Docker reutiliza la capa guardada en caché en lugar de volver a ejecutar la instrucción.

### El Reto de Caché en CI/CD:
En los runners efímeros de GitHub Actions o GitLab, la RAM y el disco se destruyen al terminar el job. La caché local de Docker se pierde, forzando al pipeline a reconstruir todas las capas e instalar todas las dependencias NPM desde cero en cada ejecución.

### La Solución: BuildKit Registry Cache
Utilizando el nuevo motor de compilación **Docker BuildKit**, podemos indicarle al pipeline que exporte e ingrese la caché de capas directamente desde un registro seguro en red:
* **`cache-from`**: Lee las capas previas desde la caché persistente del workflow.
* **`cache-to`**: Exporta las nuevas capas generadas al finalizar la compilación con éxito.

---

> [!NOTE]
> ### 📦 La Estandarización del Contenedor Marítimo
> 
> Entendamos la dockerización multi-stage y el caché de capas con una analogía física e histórica del transporte de mercancías:
> 
> - **El Enfoque Antiguo Caótico (El Transporte a Granel)**:
>   - En los años 1950, para transportar 1,000 mercancías diferentes (sacos de harina, pianos, barriles de petróleo) en un barco, los estibadores debían acomodar a mano cada pieza en los almacenes del barco.
>   - Si el barco cambiaba de puerto, descargar y volver a acomodar las piezas requería días completos de esfuerzo físico caótico, y las mercancías solían romperse o perderse por el camino.
> 
> - **El Contenedor Docker (El Contenedor Marítimo Estandarizado)**:
>   - En 1956, Malcolm McLean inventó **el contenedor de acero estandarizado** de 20 y 40 pies.
>   - No importa si dentro llevas pianos, harina o juguetes. El contenedor tiene las mismas esquinas de encastre universales y las grúas de cualquier puerto del mundo lo mueven en 10 segundos de forma segura.
>   - Tu aplicación se encapsula en este contenedor estandarizado. Al sistema operativo del servidor no le importa qué lenguaje o base de datos lleva dentro; lo ejecuta con total predictibilidad en caliente.
> 
> - **La Optimización Multi-Stage (La Fábrica y la Caja de Regalo)**:
>   - Imagina que fabricas osos de peluche de felpa.
>   - En tu fábrica tienes sierras, taladros, rollos gigantescos de tela, retazos de basura y aserrín en el suelo (**las herramientas de desarrollo y `node_modules`**).
>   - Si empaquetas el oso de peluche metiendo toda la fábrica y la basura dentro de una caja de acero gigante (**imagen monolítica sin multi-stage**), el paquete pesará 1 tonelada y el cliente recibirá aserrín y herramientas peligrosas en su casa.
>   - Utilizando **Multi-Stage**, creas el oso en la fábrica, tomas **únicamente el peluche terminado**, lo limpias con un cepillo y lo colocas dentro de una pequeña caja de regalo impecable de cartón ligero (**imagen final de producción**). El envío pesa 200 gramos y es seguro para un niño.

---

## 8.3 Implementación Práctica: Dockerfile Multi-Stage y Workflow de Compilación

A continuación, implementaremos primero el **Dockerfile Multi-Stage optimizado** para Node/TypeScript, y posteriormente el archivo YAML de **GitHub Actions** que compila la imagen, aplica caching avanzado de capas BuildKit y la publica en Docker Hub:

### `Dockerfile`
```dockerfile
# ==========================================
# ETAPA 1: Compilación de Código (Builder)
# ==========================================
FROM node:20-alpine AS builder

WORKDIR /usr/src/app

# Copiamos manifiestos primero para optimizar la caché de capas de Docker
COPY package*.json ./

# Instalamos todas las dependencias (incluyendo devDependencies)
RUN npm ci

# Copiamos el código fuente de TypeScript
COPY tsconfig.json ./
COPY src/ ./src/

# Compilamos de TypeScript a JavaScript nativo
RUN npm run build

# ==========================================
# ETAPA 2: Entorno de Ejecución (Runner)
# ==========================================
FROM node:20-alpine AS runner

# Definir variable de entorno para optimizar librerías en caliente
ENV NODE_ENV=production

WORKDIR /usr/src/app

COPY package*.json ./

# Instalamos únicamente dependencias estrictas de producción para ahorrar espacio
RUN npm ci --only=production

# Copiamos únicamente el código compilado de la etapa Builder
COPY --from=builder /usr/src/app/dist ./dist

# Principio de Menor Privilegio: Cambiar al usuario sin privilegios root predefinido de node
USER node

EXPOSE 3000

CMD ["node", "dist/app.js"]
```

### `publicarContenedor.yml`
```yaml
name: Tubería de Compilación y Publicación de Contenedor Docker

on:
  push:
    branches: [ main ]

jobs:
  construir-y-publicar:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    steps:
      - name: Clonar código fuente
        uses: actions/checkout@v4

      # 1. Configurar el motor de compilación moderno Buildx de Docker
      - name: Configurar Docker Buildx
        uses: docker/setup-buildx-action@v3

      # 2. Iniciar sesión en Docker Hub
      # Requires DOCKER_USERNAME y DOCKER_PASSWORD en los Secrets de tu repositorio
      - name: Iniciar sesión en Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      # 3. Compilar y publicar imagen usando BuildKit Registry Cache
      - name: Compilar y publicar imagen optimizada
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/api-financiera:latest
          # Configuración de caching de capas persistente de BuildKit en GitHub Actions
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Resumen del Capítulo

* **Multi-Stage Builds** dividen el empaquetamiento en etapas de compilación (`builder`) y ejecución (`runner`), reduciendo drásticamente el peso de las imágenes Docker finales en disco.
* Cambiar al usuario **`node`** predefinido en la etapa de ejecución evita problemas graves de seguridad asociados a correr procesos del contenedor con privilegios root.
* El motor **Buildx (BuildKit)** permite implementar caché persistente de capas de Docker en workflows CI/CD a través del almacenamiento nativo de las acciones de GitHub.
* La organización lógica del Dockerfile (copiar manifiestos de dependencias antes de copiar el código fuente completo) maximiza la reutilización de capas locales disminuyendo los tiempos de build.

En el próximo capítulo, ingresaremos a la automatización de la infraestructura física mediante el estudio de **Infrastructure as Code (IaC) con Terraform en Pipelines** en producción.

---

[← Capítulo anterior (Capítulo 7)](07-calidad-sonarqube.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 9) →](09-terraform-pipelines.md)
