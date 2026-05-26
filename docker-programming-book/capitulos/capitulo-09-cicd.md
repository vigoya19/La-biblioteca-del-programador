# Capítulo 9: CI/CD con Docker

Docker cambia la forma de entregar software porque convierte el artefacto de despliegue en una imagen inmutable. En lugar de compilar una aplicación en CI, copiar archivos a mano y esperar que el servidor tenga exactamente las mismas dependencias, el pipeline construye una imagen versionada, la prueba, la escanea y la publica en un registry. Producción descarga ese mismo artefacto.

La idea central de este capítulo es simple: **si una imagen no fue construida, probada, firmada o escaneada por el pipeline, no debería llegar a producción**.

---

## 9.1 El Flujo Moderno de Entrega con Docker

Un pipeline Docker saludable suele tener estas etapas:

1. **Validación rápida**: lint, tests unitarios y análisis estático.
2. **Construcción de imagen**: `docker build` o `docker buildx build`.
3. **Pruebas sobre la imagen**: ejecutar tests, smoke tests o comandos de salud dentro del contenedor.
4. **Escaneo de seguridad**: vulnerabilidades en dependencias, paquetes del sistema y secretos.
5. **Publicación**: push a Docker Hub, GitHub Container Registry, Amazon ECR, Google Artifact Registry o Azure Container Registry.
6. **Despliegue**: actualización de Compose, Swarm, Kubernetes, ECS, Cloud Run u otra plataforma.

La diferencia importante frente a pipelines tradicionales es que el pipeline ya no produce "archivos sueltos"; produce una imagen con nombre, tag y digest.

```text
commit -> test -> build image -> scan -> push registry -> deploy
                                  |
                                  v
                         sha256:digest inmutable
```

El tag `latest` puede existir para comodidad, pero no debe ser la referencia confiable de producción. Para despliegues reproducibles usa tags con SHA de Git, versión semántica o ambos:

```bash
docker build -t ghcr.io/mi-org/api:1.4.2 .
docker build -t ghcr.io/mi-org/api:git-a1b2c3d .
```

## 9.2 Preparar una Imagen Amigable para CI

Un buen pipeline empieza en el `Dockerfile`. Si la imagen tarda demasiado, cambia en cada build o requiere secretos durante la construcción, CI se vuelve lento y frágil.

Un patrón sano para aplicaciones Node.js sería:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM deps AS test
COPY . .
RUN npm test

FROM deps AS build
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

Este diseño separa dependencias, pruebas, build y runtime. El pipeline puede construir el target `test` para validar y después el target final para publicar:

```bash
docker build --target test -t api:test .
docker build -t api:local .
```

También conviene mantener un `.dockerignore` agresivo:

```dockerignore
.git
node_modules
coverage
dist
.env
*.log
```

Esto reduce el contexto de build, evita copiar secretos y mejora la reutilización de caché.

## 9.3 Pipeline con GitHub Actions

Este workflow construye, escanea y publica una imagen en GitHub Container Registry. Usa el SHA del commit como tag inmutable y mantiene caché de BuildKit entre ejecuciones.

```yaml
name: docker-ci

on:
  push:
    branches: [main]
  pull_request:

env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}/api

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to registry
        if: github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build test target
        uses: docker/build-push-action@v6
        with:
          context: .
          target: test
          load: true
          tags: api:test
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: |
            ${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.IMAGE_NAME }}:main
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          ignore-unfixed: true

      - name: Upload scan report
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```

En pull requests el workflow valida que la imagen se pueda construir y que los tests pasen. En `main`, además publica la imagen.

## 9.4 Caching con BuildKit y Buildx

Sin caché, cada pipeline descarga dependencias desde cero. Con BuildKit puedes persistir capas entre ejecuciones y reducir builds de minutos a segundos.

En GitHub Actions, la combinación habitual es:

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

En registries, puedes usar una imagen de caché:

```bash
docker buildx build \
  --cache-from type=registry,ref=ghcr.io/mi-org/api:buildcache \
  --cache-to type=registry,ref=ghcr.io/mi-org/api:buildcache,mode=max \
  -t ghcr.io/mi-org/api:git-a1b2c3d \
  --push .
```

La regla práctica: copia primero los archivos que cambian menos. En Node, `package-lock.json` cambia menos que `src/`; en Go, `go.mod` y `go.sum` cambian menos que el código; en Java, `pom.xml` o `build.gradle` deberían copiarse antes del árbol completo.

## 9.5 Tags, Digests y Promoción entre Ambientes

Un tag es mutable: `api:main` puede apuntar hoy a una imagen y mañana a otra. Un digest es inmutable:

```text
ghcr.io/mi-org/api@sha256:0f8c...
```

Para ambientes críticos, despliega por digest. Puedes construir una sola vez y promover el mismo artefacto:

```text
build once:
  api:git-a1b2c3d -> sha256:0f8c...

staging:
  deploy sha256:0f8c...

production:
  deploy sha256:0f8c...
```

Esto evita el error clásico de "funcionó en staging, pero producción recibió otra imagen".

## 9.6 Escaneo de Seguridad en el Pipeline

El escaneo no reemplaza buenas prácticas de construcción, pero ayuda a detectar paquetes vulnerables, imágenes base obsoletas y dependencias peligrosas.

Herramientas comunes:

| Herramienta | Uso principal |
|---|---|
| Trivy | Vulnerabilidades, secrets, misconfigurations |
| Grype | Vulnerabilidades en imágenes y SBOM |
| Docker Scout | Análisis integrado al ecosistema Docker |
| Syft | Generación de SBOM |
| Cosign | Firma y verificación de imágenes |

Un gate simple puede fallar el pipeline si aparecen vulnerabilidades críticas:

```bash
trivy image --exit-code 1 --severity CRITICAL ghcr.io/mi-org/api:git-a1b2c3d
```

Para equipos maduros, el siguiente paso es generar SBOM y firmar:

```bash
syft ghcr.io/mi-org/api:git-a1b2c3d -o spdx-json > sbom.spdx.json
cosign sign ghcr.io/mi-org/api:git-a1b2c3d
```

## 9.7 Despliegue desde CI/CD

Docker no prescribe una plataforma de despliegue. El mismo artefacto puede ir a varios destinos.

Con Docker Compose en un servidor:

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

Con Kubernetes:

```bash
kubectl set image deployment/api api=ghcr.io/mi-org/api:git-a1b2c3d
kubectl rollout status deployment/api
```

Con Helm:

```bash
helm upgrade --install api ./charts/api \
  --set image.repository=ghcr.io/mi-org/api \
  --set image.tag=git-a1b2c3d
```

El pipeline debe esperar confirmación del rollout. Publicar una imagen no significa que la aplicación esté sana.

## 9.8 Checklist de Producción

- Usa tags inmutables basados en SHA o versión.
- Evita desplegar `latest` en producción.
- Ejecuta tests antes de publicar.
- Escanea la imagen y define una política clara de severidad.
- No pases secretos con `ARG`; usa secrets del runtime o BuildKit secrets.
- Publica solo desde ramas protegidas o releases.
- Mantén `.dockerignore` actualizado.
- Despliega por digest cuando necesites trazabilidad fuerte.
- Registra el digest desplegado en logs, releases o changelog.
- Automatiza rollback: Compose, Swarm, Kubernetes o tu plataforma deben tener una ruta de reversión.

## Resumen del Capítulo

CI/CD con Docker consiste en tratar la imagen como el artefacto principal de entrega. El pipeline construye, prueba, escanea, publica y despliega una unidad inmutable. BuildKit y Buildx hacen que el proceso sea rápido; los tags y digests hacen que sea trazable; los escaneos, SBOM y firmas agregan confianza.

En el siguiente capítulo profundizaremos en seguridad de contenedores: usuarios no root, capacidades Linux, imágenes mínimas, secretos, escaneo y hardening del runtime.

---

← [Capítulo anterior](capitulo-08-orquestacion.md) | [Inicio](../README.md) | [Capítulo siguiente →](capitulo-10-seguridad.md)
