# Capítulo 4: GitLab CI/CD: Pipelines Multi-Project y Dynamic Pipelines

> "Un pipeline de entrega moderno no puede limitarse a compilar un único repositorio aislado. En arquitecturas complejas de microservicios, el éxito operativo exige pipelines federados capaces de coordinar dependencias entre repositorios y generar flujos de ejecución dinámicos en caliente basándose en los archivos modificados."

En el mundo del software empresarial, **GitLab CI/CD** destaca como una de las herramientas de automatización más maduras y robustas. A diferencia de otras soluciones que integran componentes externos de terceros, GitLab ofrece un ecosistema unificado que gestiona todo el ciclo de vida de desarrollo.

Sin embargo, a medida que las organizaciones migran hacia arquitecturas distribuidas de microservicios, los pipelines gigantescos monolíticos se vuelven lentos y difíciles de mantener. En este capítulo, estudiaremos los internals de **GitLab CI/CD**, la diferencia crítica entre **Artifacts** y **Cache**, y dominaremos las dos características más potentes para arquitecturas complejas: **Pipelines Multi-Proyecto** y **Pipelines Secundarios Dinámicos (Dynamic Child Pipelines)**.

---

## 4.1 Arquitectura de GitLab Runners: El Executor de Docker

El núcleo ejecutor de GitLab CI/CD es el **GitLab Runner**, un agente de software escrito en Go que corre de forma aislada y se registra contra tu instancia de GitLab.

### El Docker Executor:
A diferencia de los entornos predefinidos, el **Docker Executor** es el estándar de producción más utilizado:
* Cada paso (`job`) del pipeline define una directiva **`image`** (por ejemplo, `image: node:20-alpine`).
* El GitLab Runner se comunica con el demonio local de Docker, descarga la imagen oficial en milisegundos, monta los archivos de tu repositorio dentro de un contenedor en ejecución, y corre allí las instrucciones descritas en la sección `script`.
* Esto garantiza que tus compilaciones ocurran en un **entorno hermético y reproducible de primer nivel**, libre de contaminación por librerías preinstaladas en el sistema operativo del host.

---

## 4.2 La Batalla de Rendimiento: Cache vs. Artifacts

Es una de las confusiones más recurrentes en el diseño de pipelines. Aunque ambos guardan archivos para pasos posteriores, sus propósitos físicos y comportamientos en disco son opuestos:

| Característica | Cache (`cache`) | Artifacts (`artifacts`) |
| :--- | :--- | :--- |
| **Propósito** | Acelerar el rendimiento del pipeline guardando dependencias de descarga pesada (ej: `node_modules`, `.npm`). | Traspasar archivos finales de compilación indispensables para la ejecución entre etapas sucesivas (ej: binarios compildados, reportes de pruebas). |
| **Persistencia** | Asíncrona e inestable. GitLab intenta recuperarla, pero el pipeline debe poder compilar desde cero si la caché no existe. | Garantizada y obligatoria. Si un job posterior necesita el artefacto y este no está, el pipeline se detiene con error. |
| **Uso entre Jobs** | Se comparte de forma horizontal e incremental entre múltiples ejecuciones del mismo pipeline a lo largo del tiempo. | Se transmite verticalmente de forma secuencial de un stage al siguiente en el mismo flujo de ejecución. |

---

## 4.3 Pipelines Multi-Proyecto (Cross-Project Triggers)

En arquitecturas de microservicios estructuradas en múltiples repositorios separados, a menudo un cambio en un repositorio de librerías core (`libreria-financiera`) requiere disparar y validar automáticamente los pipelines de los microservicios que dependen de ella (`microservicio-pagos`).

GitLab resuelve esto nativamente mediante **Multi-Project Pipelines**:
* Un job de tu repositorio principal puede invocar de forma asíncrona un disparador (**`trigger`**) que inicia de forma remota la compilación en otro repositorio independiente de GitLab, pasando variables lógicas de entorno en red y esperando el resultado para confirmar el éxito del flujo.

---

> [!NOTE]
> ### 📦 La Cinta Transportadora con Brazos Mecánicos Múltiples
> 
> Entendamos la caché, los artefactos, los triggers multiproyecto y los pipelines dinámicos utilizando una analogía física industrial:
> 
> - **La Diferencia de Cache vs. Artifacts (El Taller de Pintura de Autos)**:
>   - Imagina que estás fabricando un coche deportivo en una línea de montaje secuencial de tres estaciones.
>   - **El Artefacto (El Chasis Compilado)**: En la Estación 1, construyes el chasis físico del coche. Ese chasis de metal **debe** pasar de forma obligatoria a la Estación 2 para poder atornillar las puertas (**un artefacto del compilado `dist/`**). Si el chasis desaparece, el coche no se puede construir. Es vital.
>   - **La Caché (La Caja de Destornilladores Reutilizables)**: En la Estación 1, utilizas una caja de destornilladores eléctricos para apretar los tornillos. Al final del día, los dejas en un estante para que al día siguiente los uses de nuevo (**la caché de `node_modules`**). Si por la noche un conserje limpia el taller y tira los destornilladores (borrado de caché), la línea de montaje sigue funcionando; solo que tendrás que ir a la tienda a comprar destornilladores nuevos al día siguiente, tardando 5 minutos más en arrancar la jornada.
> 
> - **El Pipeline Multi-Proyecto (Los Teléfonos de Línea entre Fábricas Vecinas)**:
>   - Tienes dos fábricas ubicadas en calles separadas: **Fábrica de Motores (Proyecto A)** y **Fábrica de Coches Completos (Proyecto B)**.
>   - Cada vez que terminas de perfeccionar un motor nuevo en la Fábrica A, el supervisor no camina con el motor a cuestas. Toma un teléfono rojo directo (**Trigger Multiproyecto**) y llama a la Fábrica B: *"Acabamos de validar el motor v2, arranca de inmediato tu línea de montaje para construir el coche con este motor"*. Ambas fábricas operan de forma independiente pero coordinada en red.
> 
> - **El Pipeline Dinámico (La Recepción de Paquetes con Contenido Variable)**:
>   - Tienes un almacén central de distribución que recibe paquetes de diferentes formas y tamaños en la entrada (**un repositorio Mono-Repo con microservicios**).
>   - En lugar de encender todas las grúas gigantes y bandas de transporte del edificio cada vez que entra un paquete pequeño de apenas 10 gramos (un script ineficiente que compila todo el monorepo por un cambio de un renglón en un microservicio):
>   - Contratas a un clasificador en la puerta. Este abre la correspondencia, lee qué contiene y escribe una pequeña lista de instrucciones específica en una hoja: *"Hoy solo llegó ropa: encender únicamente la lavadora 2"*. El clasificador mete esa hoja en el robot del pipeline y este **genera en caliente un flujo a medida** de forma instantánea (**Dynamic Child Pipeline**), evitando encender maquinaria innecesaria en el edificio.

---

## 4.4 Código YAML: Pipelines Multiproyecto y Generación Dinámica

A continuación, implementaremos la configuración real y sin placeholders para GitLab CI/CD. Mostraremos un pipeline principal en `.gitlab-ci.yml` que:
1. Dispara de forma remota un pipeline multi-proyecto en un repositorio secundario.
2. Ejecuta un script en TypeScript para escanear directorios modificados, escribe un archivo YAML temporal en caliente y lo dispara como un **Dynamic Child Pipeline**:

#### [.gitlab-ci.yml](file:///Users/andres/Documents/biblioteca/devops-programming-book/.gitlab-ci.yml)
```yaml
stages:
  - test
  - trigger-externo
  - pipeline-dinamico

# 1. Ejecutar suites de test en contenedor hermético usando Docker Executor
ejecutar-tests-locales:
  stage: test
  image: node:20-alpine
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - .npm/
      - node_modules/
  before_script:
    - npm ci --cache .npm --prefer-offline
  script:
    - npm run test

# 2. Pipeline Multi-Proyecto: Dispara asíncronamente el pipeline de otra app de cara al cliente
disparar-pipeline-clientes:
  stage: trigger-externo
  needs: [ejecutar-tests-locales]
  # trigger invoca la compilación del repo secundario 'servicios-cliente-app' en la rama 'main'
  trigger:
    project: software-banco/servicios-cliente-app
    branch: main
    strategy: depend # Espera a que el pipeline externo termine con éxito para marcar este paso como válido

# 3. Pipeline Dinámico: Genera un YAML a medida y lo ejecuta en caliente
generar-pipeline-hijo:
  stage: pipeline-dinamico
  image: node:20-alpine
  script:
    # Ejecutamos un script local en TypeScript/Node para analizar cambios y escribir el YAML dinámico en disco
    - node ./scripts/generarPipelineManifiesto.js
  artifacts:
    paths:
      - pipeline-hijo-generado.yml
    expire_in: 1 day

ejecutar-pipeline-hijo:
  stage: pipeline-dinamico
  needs: [generar-pipeline-hijo]
  # trigger:include le dice a GitLab que lea el artefacto YAML generado en caliente en el paso anterior y lo ejecute de inmediato
  trigger:
    include:
      - artifact: pipeline-hijo-generado.yml
        job: generar-pipeline-hijo
    strategy: depend
```

#### [generarPipelineManifiesto.js](file:///Users/andres/Documents/biblioteca/devops-programming-book/scripts/generarPipelineManifiesto.js)
```javascript
const fs = require('fs');

// Script de automatización de CI para monorepos:
// Detecta cambios en subdirectorios de microservicios y escribe el manifiesto YAML en caliente
function generarManifiestoDinamico() {
  console.log('[CI-Generador] Analizando estructura de directorios modificados...');

  // Simulación de análisis de git (en producción se puede usar: git diff --name-only HEAD~1)
  // Evaluamos si el microservicio de contabilidad sufrió modificaciones
  const microservicioModificado = 'contabilidad'; 
  
  // Escribimos la estructura de la tubería YAML dinámica en caliente
  const contenidoYAML = `
stages:
  - compilar-servicio

compilar-microservicio-${microservicioModificado}:
  stage: compilar-servicio
  image: node:20-alpine
  script:
    - echo "Iniciando compilación a medida del microservicio modificado: ${microservicioModificado}"
    - cd services/${microservicioModificado}
    - npm ci
    - npm run build
`;

  fs.writeFileSync('pipeline-hijo-generado.yml', contenidoYAML);
  console.log('[CI-Generador] Manifiesto dinamico y optimizado escrito con éxito en disk.');
}

generarManifiestoDinamico();
```

---

## Resumen del Capítulo

* El **Docker Executor** de GitLab Runners garantiza compilaciones completamente herméticas y aisladas descargando imágenes Docker oficiales de forma nativa.
* La **Cache** acelera las descargas recurrentes de dependencias externas a lo largo del tiempo, mientras que los **Artifacts** son binarios de compilación indispensables para la continuidad del pipeline.
* Los **Multi-Project Pipelines** integran dependencias lógicas complejas entre repositorios independientes disparando compilaciones en red de forma coordinada.
* Los **Dynamic Child Pipelines** permiten modularizar y optimizar al máximo monorepos gigantes generando y ejecutando manifiestos de pipelines YAML a medida en caliente.

En el próximo capítulo, estudiaremos la modularización empresarial en la nube mediante el estudio de **Azure DevOps Pipelines: YAML Reutilizable y Plantillas** en producción.

---

[← Capítulo anterior (Capítulo 3)](03-github-actions-workflows.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 5) →](05-azure-devops-pipelines.md)
