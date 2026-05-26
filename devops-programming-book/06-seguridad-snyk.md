# Capítulo 6: Seguridad de Código (DevSecOps): Integración y Configuración de Snyk

> "La seguridad no puede ser un control de calidad tardío que se ejecuta el día antes de salir a producción. En la era de las dependencias open-source masivas, la seguridad debe integrarse de forma atómica en cada commit, convirtiendo el pipeline de CI/CD en un escudo infranqueable contra vulnerabilidades de código y dependencias."

En el desarrollo de software moderno, rara vez escribimos una aplicación desde cero. Un proyecto Node.js promedio importa cientos de dependencias open-source a través de `npm`, las cuales a su vez importan recursivamente miles de sub-dependencias. 

Aunque esto acelera la velocidad de desarrollo de forma espectacular, abre una superficie de ataque inmensa: **más del $80\%$ de las brechas de seguridad actuales en aplicaciones web provienen de dependencias vulnerables de terceros**. Para combatir esto, nace la cultura de **DevSecOps (Desarrollo, Seguridad y Operaciones)**, integrando análisis de seguridad automatizado en el pipeline de CI. En este capítulo, estudiaremos los conceptos de **SCA (Software Composition Analysis)** y **SAST (Static Application Security Testing)**, y aprenderemos a instalar, configurar e integrar **Snyk** en caliente.

---

## 6.1 Fundamentos de DevSecOps: SCA vs. SAST

Para blindar nuestro software de forma automatizada, combinamos dos tecnologías de escaneo estático:

### 1. SCA (Software Composition Analysis)
* **Qué hace**: Analiza la "receta de ingredientes" de tu aplicación (los manifiestos de dependencias como `package.json`, `package-lock.json`, `pom.xml` o `requirements.txt`).
* **Su valor**: Compara tus librerías y versiones instaladas contra bases de datos globales de vulnerabilidades (CVEs - *Common Vulnerabilities and Exposures*). Te avisa si estás importando una librería con una vulnerabilidad conocida (como una inyección remota de código o denegación de servicio) y te indica a qué versión segura debes actualizar.

### 2. SAST (Static Application Security Testing)
* **Qué hace**: Examina directamente la estructura lógica de tu propio código fuente en frío sin ejecutarlo.
* **Su valor**: Detecta malas prácticas de programación que causan vulnerabilidades (ej. concatenaciones de SQL vulnerables a inyección, contraseñas escritas en texto plano en el código, o configuraciones SSL/TLS inseguras).

---

## 6.2 Snyk: La Bóveda de Protección de Dependencias

**Snyk** es la herramienta líder de DevSecOps debido a su inmensa y actualizada base de datos de vulnerabilidades, su excelente soporte multi-lenguaje y su fluida integración con pipelines mediante su interfaz de comandos de terminal CLI.

### Guía de Instalación del CLI de Snyk (Local):
Antes de integrar Snyk en el pipeline, es vital que los desarrolladores puedan auditar sus dependencias de forma local en su terminal:

1. **Instalación usando NPM**:
   ```bash
   npm install -g snyk
   ```
2. **Autenticación e Inicio de Sesión**:
   ```bash
   snyk auth
   ```
   *Esto abrirá una ventana de navegador web para loguearte con tu cuenta de Snyk de forma gratuita y generar un Token de API seguro local.*
3. **Escaneo de dependencias en el proyecto local**:
   ```bash
   snyk test
   ```
   *Snyk analizará tu archivo `package-lock.json` y arrojará un reporte detallado en consola agrupando las vulnerabilidades por severidad (Crítica, Alta, Media, Baja), con su respectivo CVE y la recomendación de upgrade de versión.*
4. **Escaneo de contenedores locales (Dockerfiles)**:
   ```bash
   snyk container test mi-imagen-docker:latest
   ```

---

> [!NOTE]
> ### 🛡️ El Inspector de Ingredientes de Alimentos (Snyk)
> 
> Entendamos los conceptos de SCA, SAST y la integración de seguridad en el pipeline utilizando una analogía física e intuitiva de salud pública:
> 
> - **El Enfoque Tradicional Inseguro (El Restaurante Sin Controles)**:
>   - Imagina que eres dueño de un restaurante de hamburguesas de alta gama en la ciudad.
>   - Contratas a cocineros de todo el mundo y les permites traer ingredientes de cualquier mercado local sin revisar las marcas ni procedencias de la carne, quesos o verduras (**importar miles de node_modules de terceros a ciegas**).
>   - Si un proveedor de lechugas del mercado tiene un brote de bacterias dañinas (**una vulnerabilidad en una sub-dependencia**), tus cocineros la usarán directamente en los platillos. Los clientes se enfermarán de gravedad y tu restaurante será clausurado permanentemente por las autoridades.
> 
> - **El SCA con Snyk (El Inspector de Alimentos en la Entrada)**:
>   - Instalas un **puesto de control e inspección biológica en la puerta de la cocina (SCA con Snyk CLI)**.
>   - Cada vez que un mensajero llega con una caja de verduras, el inspector saca un manual de alertas alimenticias del gobierno nacional (**La Base de Datos de Vulnerabilidades CVE**).
>   - El inspector escanea el código de barras del queso y grita de inmediato: *"¡Detengan ese queso! Lote 45 proveniente del proveedor X tiene una alerta de salmonella. Prohibido entrar a la cocina"*. El queso infectado es tirado a la basura antes de que los cocineros siquiera abran la caja. Tu menú permanece $100\%$ libre de bacterias.
> 
> - **El SAST (El Supervisor de Técnicas de Cocina)**:
>   - Además de revisar ingredientes, contratas a un supervisor que mira a los cocineros trabajar en frío (**SAST**).
>   - El supervisor observa a un cocinero cortar pollo crudo en una tabla de madera y usar de inmediato la misma tabla sin lavar para picar los tomates de las ensaladas.
>   - El supervisor le da un manotazo antes de que termine y le dice: *"¡Mala práctica de higiene! Eso causará contaminación cruzada (inyección de código/malas prácticas). Lava la tabla de inmediato"*. Corriges el error de manipulación en caliente antes de que el cliente reciba la ensalada.

---

## 6.3 Integración en Tuberías CI/CD: Rompiendo la Build en GitHub Actions

En un pipeline de producción real, no permitimos que un commit se consolide si introduce vulnerabilidades de nivel crítico o alto en el software. Configura el pipeline para **romper la build** arrojando un error inmediato de ejecución si Snyk encuentra problemas críticos.

A continuación, implementaremos la sintaxis real y unificada para integrar Snyk en una tubería de integración continua en **GitHub Actions**:

### `tuberíaSeguridadSnyk.yml`
```yaml
name: Tubería de Seguridad y DevSecOps (Snyk)

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  escaneo-seguridad-snyk:
    name: Escaneo de Vulnerabilidades (SCA) con Snyk
    runs-on: ubuntu-latest
    steps:
      # 1. Clonar el código del repositorio
      - name: Clonar código fuente
        uses: actions/checkout@v4

      # 2. Configurar entorno de ejecución Node.js
      - name: Configurar Node.js v20
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Instalar dependencias limpias
        run: npm ci

      # 3. Integración de Snyk utilizando la Acción Oficial
      # Requiere configurar el token de Snyk de forma segura en los Secrets de tu repositorio de GitHub: SNYK_TOKEN
      - name: Ejecutar Snyk SCA para dependencias de Node.js
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          # --severity-threshold=high le dice a Snyk que ignore alertas bajas o medias,
          # pero que ROMPA y falle el pipeline inmediatamente si encuentra una vulnerabilidad de nivel ALTO o CRÍTICO.
          args: --severity-threshold=high --fail-on=all

      # 4. (Opcional) Escanear directamente el Dockerfile y código del contenedor
      # - name: Ejecutar Snyk para contenedor Docker
      #   uses: snyk/actions/docker@master
      #   env:
      #     SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      #   with:
      #     image: app-financiera:latest
      #     args: --file=Dockerfile --severity-threshold=high
```

---

## Resumen del Capítulo

* **SCA (Software Composition Analysis)** audita y escanea el árbol de dependencias externas en base a bases de datos globales de CVEs.
* **SAST (Static Application Security Testing)** inspecciona y valida la calidad lógica interna de tu propio código fuente buscando fallos estructurales de seguridad en frío.
* **Snyk** es el pilar de DevSecOps por excelencia gracias a su CLI versátil y sus integraciones nativas ágiles con pools y runners distribuidos.
* Configurar umbrales de fallo estrictos (`--severity-threshold=high`) en pipelines de CI evita que software con vulnerabilidades críticas en dependencias de terceros sea desplegado a producción.

En el próximo capítulo, escalaremos nuestra maestría en análisis de calidad de código mediante la instalación, dockerización y despliegue del servidor líder **SonarQube** en producción.

---

[← Capítulo anterior (Capítulo 5)](05-azure-devops-pipelines.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 7) →](07-calidad-sonarqube.md)
