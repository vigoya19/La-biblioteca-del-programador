# Capítulo 7: Calidad de Código y Cobertura: Instalación y Configuración de SonarQube

> "El código que compila y pasa las pruebas de negocio no es necesariamente código de alta calidad. La acumulación silenciosa de complejidad cognitiva, duplicaciones y deuda técnica degrada la velocidad de desarrollo en el mediano plazo hasta paralizar por completo la entrega de valor."

Hasta ahora hemos estudiado la entrega continua de código, la ramificación y la seguridad de las dependencias. Sin embargo, una vez asegurado que las librerías externas están libres de vulnerabilidades críticas, debemos inspeccionar la salud interna de **nuestro propio código fuente**.

Un código mal estructurado, con alta complejidad o sin pruebas unitarias es difícil de mantener, incomprensible para nuevos desarrolladores y propenso a revivir bugs del pasado. Para automatizar este análisis y aplicar umbrales mínimos de aceptación lógicos, la industria se apoya en **SonarQube**. En este capítulo, desmitificaremos las métricas de calidad de código, estudiaremos la arquitectura de SonarQube y aprovisionaremos un servidor de producción completo utilizando **Docker Compose con base de datos PostgreSQL persistida**.

---

## 7.1 Métricas de Calidad de Código y Quality Gates

SonarQube escanea de forma estática el código y genera telemetría sobre cuatro pilares fundamentales:

1. **Bugs e Inseguridades (Reliability & Security)**: Detecta fallos lógicos evidentes (bucles infinitos, variables no inicializadas) y malas prácticas de seguridad (vulnerabilidades SAST).
2. **Code Smells (Malos Olores de Código)**: Fragmentos de código que indican un mal diseño de software. No impiden que la app funcione, pero dificultan la legibilidad y mantenimiento (ej. funciones de 500 líneas o variables mal nombradas).
3. **Complejidad Cognitiva (Cognitive Complexity)**: A diferencia de la complejidad ciclomática tradicional (que mide la cantidad de bifurcaciones lógicas de la CPU), la complejidad cognitiva **mide qué tan difícil es para un cerebro humano comprender la lógica del flujo de código**.
4. **Deuda Técnica (Technical Debt)**: El tiempo estimado (en minutos u horas) requerido por un programador para refactorizar y resolver todos los Code Smells detectados en el repositorio.

### ¿Qué es un Quality Gate (Compuerta de Calidad)?
Es el conjunto de reglas y umbrales mínimos obligatorios que un proyecto debe pasar con éxito para ser aceptado en el pipeline de CI (por ejemplo: *"Cobertura de tests > 80%", "Cero Bugs nuevos de severidad bloqueante" y "Rating de confiabilidad = A"*). Si la rama no cumple el Quality Gate, el pipeline falla y se detiene el flujo de entrega.

---

## 7.2 Arquitectura y Despliegue de SonarQube con Docker Compose

Un servidor SonarQube local o de producción consta de tres componentes lógicos:
* **SonarQube Server (Web & Search)**: El panel de administración que aloja la UI de usuario y el motor de búsqueda (Elasticsearch) para consultar métricas.
* **Database (PostgreSQL)**: El motor relacional que almacena persistentemente todas las métricas históricas de escaneo, configuraciones y usuarios.
* **SonarScanner**: El agente ligero de comandos CLI que corre en los runners del pipeline, lee tu código fuente local y envía los metadatos analizados al servidor a través de APIs de red.

### El Archivo Docker Compose de Producción:
A continuación, implementaremos la configuración real y sin placeholders para levantar un servidor de SonarQube autónomo y persistente en caliente:

### `docker-compose.yml`
```yaml
version: '3.8'

services:
  # 1. Base de datos PostgreSQL persistente
  db-sonarqube:
    image: postgres:15-alpine
    container_name: postgres_sonarqube
    environment:
      POSTGRES_USER: sonar_admin
      POSTGRES_PASSWORD: SecureSonarPassword2026$
      POSTGRES_DB: sonarqube_metadata
    volumes:
      # Persistencia física de los archivos de base de datos
      - pg_data_sonar:/var/lib/postgresql/data
    networks:
      - red-sonar
    restart: always

  # 2. Servidor principal de SonarQube
  sonarqube:
    image: sonarqube:10.5-community
    container_name: sonarqube_server
    depends_on:
      - db-sonarqube
    ports:
      - "9000:9000" # Puerto HTTP expuesto para la UI
    environment:
      # Conexión JDBC nativa apuntando a nuestro contenedor de PostgreSQL
      - SONAR_JDBC_USERNAME=sonar_admin
      - SONAR_JDBC_PASSWORD=SecureSonarPassword2026$
      - SONAR_JDBC_URL=jdbc:postgresql://db-sonarqube:5432/sonarqube_metadata
    volumes:
      # Persistencia física de datos recopilados y configuraciones
      - sonar_data:/opt/sonarqube/data
      - sonar_extensions:/opt/sonarqube/extensions
      - sonar_logs:/opt/sonarqube/logs
    ulimits:
      # Requisito físico de Elasticsearch integrado en SonarQube para control de memoria en hilos de red
      nofile:
        soft: 65536
        hard: 65536
    sysctls:
      - net.core.somaxconn=1024
    networks:
      - red-sonar
    restart: always

networks:
  red-sonar:
    name: red_privada_sonar
    driver: bridge

volumes:
  pg_data_sonar:
    driver: local
  sonar_data:
    driver: local
  sonar_extensions:
    driver: local
  sonar_logs:
    driver: local
```

---

> [!NOTE]
> ### 🩺 El Escáner Médico de Chequeo Corporal Completo
> 
> Entendamos la calidad de código, la deuda técnica y SonarQube a través de una analogía médica cotidiana:
> 
> - **El Enfoque Tradicional Descuidado (El Paciente que Evita el Médico)**:
>   - Imagina a una persona que come comida rápida todos los días, nunca hace ejercicio y fuma (**escribir código sucio con alta deuda técnica**).
>   - Como no tiene ningún dolor de muelas inmediato, la persona asume que está $100\%$ sana y sigue corriendo maratones (**ejecutar el software en producción sin medirlo**).
>   - Sin embargo, sus arterias se están tapando de grasa silenciosamente (**acumulación de Code Smells y duplicación**). 
>   - De pronto, en medio de un esfuerzo físico ordinario, sufre un paro cardíaco repentino e inesperado que lo deja hospitalizado por semanas (**caída catastrófica del servidor por un bug de concurrencia**). El paciente se salvó por milagro, pero el coste de recuperación es astronómico.
> 
> - **El Escaneo con SonarQube (El Chequeo Médico Anual Obligatorio)**:
>   - Decides someterte a una revisión exhaustiva en una clínica de alto nivel (**La Tubería de CI con SonarScanner**).
>   - Te acuestan en una máquina de **Resonancia Magnética y Rayos X completa (SonarQube Server)**.
>   - El escáner no solo revisa si tienes huesos rotos (errores de compilación). Realiza un análisis microscópico de tu sangre, mide tu porcentaje de grasa y tu capacidad pulmonar (**Complejidad Cognitiva y Deuda Técnica**).
>   - El médico te entrega un reporte de "Quality Gate": *"Felicidades, tus niveles de colesterol están en A, pero tienes una deuda técnica de 30 minutos de ejercicio diario debido a tu Rating de mantenibilidad en B. Te prescribo cambiar tus hábitos antes de que salgas a correr el maratón de la semana que viene"*. 
>   - Corriges tu salud de forma controlada y preventiva, garantizando que tu cuerpo resista cualquier esfuerzo físico en producción de forma predecible y saludable.

---

## 7.3 Código YAML: Integración de SonarScanner en GitHub Actions

En el pipeline de desarrollo, una vez compilada la aplicación, disparamos el **SonarScanner** para analizar el repositorio y enviar las métricas a nuestro servidor SonarQube.

A continuación, implementaremos la sintaxis real y unificada para integrar el escáner en **GitHub Actions**:

### `tuberíaSonarQube.yml`
```yaml
name: Tubería de Calidad de Código (SonarQube)

on:
  push:
    branches: [ main ]
  pull_request:
    types: [ opened, synchronize, reopened ]

jobs:
  analisis-calidad-sonar:
    name: Escaneo de Calidad Estática con SonarQube
    runs-on: ubuntu-latest
    steps:
      # 1. Clonar el repositorio
      - name: Clonar código fuente
        uses: actions/checkout@v4
        with:
          # SonarQube requiere un historial completo de git para calcular
          # de forma precisa los autores de las deudas técnicas y los commits modificados
          fetch-depth: 0

      # 2. Configurar entorno de Node y ejecutar tests con reporte de cobertura
      - name: Configurar Node.js v20
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Instalar dependencias
        run: npm ci

      # Es fundamental generar un reporte de cobertura (ej: en formato LCOV)
      # para que SonarScanner pueda enviarlo al servidor y calcular el porcentaje de test coverage
      - name: Ejecutar pruebas unitarias con reporte de cobertura LCOV
        run: npm run test:coverage -- --coverageReporters=lcov

      # 3. Disparar el escáner oficial de SonarQube
      # Requiere configurar SONAR_TOKEN y SONAR_HOST_URL de forma segura en los Secrets de GitHub
      - name: Ejecutar SonarQube Scanner
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }} # ej: https://sonarqube.mi-empresa.com
        with:
          # Argumentos de configuración del análisis estático
          args: >
            -Dsonar.projectKey=api-finanzas-service
            -Dsonar.projectName=api-finanzas-service
            -Dsonar.sources=src
            -Dsonar.tests=src/__tests__
            -Dsonar.typescript.lcov.reportPaths=coverage/lcov.info
            -Dsonar.exclusions=node_modules/**,dist/**,coverage/**
```

---

## Resumen del Capítulo

* **SonarQube** analiza y clasifica la salud lógica del software mediante métricas de bugs, vulnerabilidades, **Code Smells**, **Complejidad Cognitiva** y **Deuda Técnica**.
* Un **Quality Gate** es la compuerta de control lógica que restringe la fusión de ramas si el commit no cumple con los porcentajes mínimos exigidos de cobertura o ratings de mantenibilidad.
* Levantar SonarQube mediante **Docker Compose con base de datos PostgreSQL** garantiza la persistencia, velocidad de indexación y aislamiento de los datos físicos en caliente.
* El análisis de cobertura requiere la generación previa de reportes estándar (ej. **LCOV** o XML) por parte del framework de pruebas unitarias para que el escáner pueda transmitirlos al servidor.

En el próximo capítulo, estudiaremos el empaquetamiento estandarizado de software a gran escala mediante el estudio de **Dockerización y Pipelines de Contenedores** en producción.

---

[← Capítulo anterior (Capítulo 6)](06-seguridad-snyk.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 8) →](08-docker-pipelines.md)
