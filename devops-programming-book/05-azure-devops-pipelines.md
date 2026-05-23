# Capítulo 5: Azure DevOps Pipelines: YAML Reutilizable y Plantillas

> "El copy-paste de código YAML entre pipelines es la mayor fuente de deudas técnicas de infraestructura en la ingeniería de DevOps. Los pipelines de grado empresarial deben tratarse como software: modulares, tipados estrictamente, reutilizables y parametrizables."

En corporaciones de gran escala, **Azure DevOps (anteriormente TFS)** es uno de los pilares de automatización más extendidos debido a su robustez empresarial, su soporte nativo para proyectos híbridos y sus capacidades para modularizar arquitecturas complejas de pipelines.

A medida que una organización crece y pasa de tener 10 a tener 500 aplicaciones en producción, la mantenimiento manual de 500 archivos YAML idénticos se vuelve insostenible. Cualquier cambio (como actualizar una versión de Node.js o añadir un escáner de seguridad) exigiría editar 500 repositorios de forma repetitiva. En este capítulo, estudiaremos los internals de **Azure DevOps Pipelines**, aprenderemos a integrar de forma segura variables desde **Azure Key Vault**, y dominaremos el estándar definitivo de escalabilidad mediante **YAML Templates (Plantillas Reutilizables)**.

---

## 5.1 Anatomía de Azure Pipelines: Agent Pools y Variables de Key Vault

La arquitectura lógica de un pipeline en Azure DevOps se divide en tres niveles jerárquicos:
1. **Stages (Etapas)**: Grandes fases del ciclo de vida (ej. `Build`, `Test`, `Deploy_Staging`, `Deploy_Production`). Pueden requerir aprobaciones manuales y controles de entornos lógicos en caliente.
2. **Jobs (Trabajos)**: Unidades de ejecución agrupadas sobre un mismo **Agent** (equivalente al runner de GitHub).
3. **Tasks (Tareas)**: Instrucciones procedimentales secuenciales (ej. instalar Node, ejecutar script).

### Agent Pools:
Los agentes se organizan en **Agent Pools** (grupos de agentes). Puedes usar agentes compartidos administrados por Microsoft (`Microsoft-hosted`) o aprovisionar tus propios agentes locales privados (`Self-hosted`) instalados en máquinas físicas o clústeres de Kubernetes (`VMSS Agents`) que escalan de forma automática bajo demanda.

### Integración de Secretos con Azure Key Vault:
En entornos de nivel bancario o corporativo, nunca se almacenan contraseñas o llaves criptográficas directamente en el archivo YAML de repositorio.
* Los secretos se guardan en el servicio blindado de gestión de identidades y claves **Azure Key Vault**.
* Azure DevOps permite enlazar un **Variable Group (Grupo de Variables)** directamente a tu bóveda de Key Vault mediante credenciales seguras.
* El pipeline carga los secretos bajo demanda en memoria de forma cifrada durante la duración del Job, enmascarando cualquier intento de impresión de logs con asteriscos (`***`) de forma automática en consola.

---

## 5.2 YAML Templates: La Filosofía Dry (Don't Repeat Yourself)

La característica más potente de Azure Pipelines es su capacidad de modularización mediante **Plantillas (Templates)**. Esto permite separar la lógica procedimental pesada de la configuración básica específica de la aplicación.

### Tipos de Plantillas en Azure DevOps:
* **Step Templates (Plantillas de Pasos)**: Encapsulan una serie de tareas recurrentes (ej. compilar, ejecutar linters y subir reporte).
* **Job Templates (Plantillas de Trabajos)**: Definen un entorno completo (ej. levantar un agente Docker específico de base de datos y correr tests).
* **Stage Templates (Plantillas de Etapas)**: Definen flujos completos de despliegue a entornos con compuertas lógicas complejas de seguridad.
* **Extends Templates (Plantillas de Extensión)**: La política de seguridad corporativa por excelencia. La plantilla corporativa define la estructura obligatoria de todo el pipeline (ej. obliga a ejecutar un escáner de seguridad antes de cualquier despliegue), e impide que los desarrolladores editen la lógica del flujo, limitándolos únicamente a pasar parámetros como el nombre de su app.

---

> [!NOTE]
> ### 🏢 El Rascacielos Modular Pre-ensamblado (Plantillas YAML)
> 
> Entendamos el poder de las plantillas parametrizadas de Azure DevOps utilizando una analogía física de ingeniería civil:
> 
> - **El Enfoque de Pipelines Duplicados (La Construcción Tradicional de Ladrillos)**:
>   - Imagina que tu empresa constructora es contratada para construir 100 rascacielos residenciales idénticos en la ciudad.
>   - Si utilizas el enfoque tradicional duplicado, los albañiles van a cada terreno y comienzan a pegar ladrillo por ladrillo a mano, haciendo el cemento e instalando las tuberías de agua de forma independiente en cada edificio.
>   - Si al cabo de 6 meses el ingeniero civil descubre que el diámetro de la tubería de agua elegida viola las leyes de presión de la ciudad y debe cambiarse, tendrás que mandar cuadrillas de mecánicos a picar paredes y cambiar tuberías en los 100 rascacielos de forma manual y secuencial. El coste operativo y el riesgo de error humano te arruinarán.
> 
> - **El Enfoque YAML Templates (El Rascacielos Modular de Fábrica)**:
>   - Decides industrializar el proceso. Montas una gran planta de prefabricación de departamentos modulares en las afueras de la ciudad (**La Plantilla Central de Pipeline**).
>   - En esa fábrica, bajo condiciones hiper-controladas por robots ingenieros, ensamblas los módulos completos de cocina, baño y salas con sus tuberías de agua perfectas y seguras.
>   - Para construir los 100 rascacielos, los albañiles solo van a los terrenos a montar y encastrar los módulos prefabricados traídos de la fábrica central mediante grúas. Solo necesitan pasar parámetros lógicos específicos para cada edificio (**Los Parámetros de la Plantilla**: *Color de fachada = "Azul", Número de pisos = 20*).
>   - Si el diámetro de la tubería de agua debe actualizarse, no tocas ningún rascacielos de la ciudad. Vas a la **fábrica central de prefabricación modulares**, ajustas la máquina robótica de moldes una sola vez, y todos los siguientes módulos producidos y desplegados contarán automáticamente con la nueva tubería segura sin necesidad de picar una sola pared física en caliente en la ciudad.

---

## 5.3 Código YAML: Estructura Multicloud Reutilizable

A continuación, implementaremos un pipeline de producción modular en Azure DevOps. Consiste en dos archivos YAML reales:
1. Una **Plantilla Reutilizable (`template-compilacion.yml`)** parametrizada que encapsula la lógica compleja de compilar, testear y validar caché.
2. El **Pipeline de Aplicación (`azure-pipelines.yml`)** que simplemente importa la plantilla pasándole sus variables específicas:

#### [template-compilacion.yml](file:///Users/andres/Documents/biblioteca/devops-programming-book/pipelines/templates/template-compilacion.yml)
```yaml
# 1. Definir los parámetros de entrada y sus tipos de datos estrictos para la plantilla
parameters:
  - name: versionNode
    type: string
    default: '20.x'
  - name: entornoEjecucion
    type: string
    values:
      - 'desarrollo'
      - 'staging'
      - 'produccion'
  - name: ejecutarPruebas
    type: boolean
    default: true

jobs:
  - job: CompilarAplicacion
    displayName: 'Trabajo de Compilación y Test en ${{ parameters.entornoEjecucion }}'
    pool:
      vmImage: 'ubuntu-latest' # Microsoft-hosted agent

    steps:
      # Instalamos Node.js parametrizando dinámicamente la versión seleccionada
      - task: NodeTool@0
        displayName: 'Instalando Node.js version ${{ parameters.versionNode }}'
        inputs:
          versionSpec: '${{ parameters.versionNode }}'

      # Tarea para caching persistente de node_modules en Azure Pipelines
      - task: Cache@2
        displayName: 'Aprovisionando caché persistente de dependencias NPM'
        inputs:
          key: 'npm | "$(Agent.OS)" | package-lock.json'
          restoreKeys: |
            npm | "$(Agent.OS)"
          path: $(npm_config_cache)

      - script: |
          npm ci
        displayName: 'Instalar dependencias NPM limpias'

      # Condicional lógica en caliente evaluando parámetros booleanos de plantilla
      - ${{ if eq(parameters.ejecutarPruebas, true) }}:
        - script: |
            npm run test
          displayName: 'Ejecutar suite de pruebas unitarias críticas'

      - script: |
          npm run build
        displayName: 'Ejecutar compilación distributiva (build)'
```

#### [azure-pipelines.yml](file:///Users/andres/Documents/biblioteca/devops-programming-book/azure-pipelines.yml)
```yaml
# Tubería principal de la aplicación

trigger:
  branches:
    include:
      - main

variables:
  # Vinculamos un Grupo de Variables enlazado de forma atómica a Azure Key Vault
  - group: GrupoCuentasKeyVault

stages:
  - stage: BuildStage
    displayName: 'Etapa de Compilación'
    jobs:
      # Consumimos la plantilla reutilizable externa pasando los parámetros requeridos
      - template: pipelines/templates/template-compilacion.yml
        parameters:
          versionNode: '20.x'
          entornoEjecucion: 'staging'
          ejecutarPruebas: true

  - stage: DeployStage
    displayName: 'Etapa de Despliegue'
    dependsOn: BuildStage
    jobs:
      - deployment: DesplegarServidor
        displayName: 'Despliegue controlado en Staging'
        environment: 'entorno-staging'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          # Estrategia runOnce básica para despliegue
          runOnce:
            deploy:
              steps:
                - script: |
                    # Consumimos el secreto $(ContraseñaBaseDatos) importado directamente
                    # de forma invisible y enmascarada desde el grupo de Key Vault
                    echo "Iniciando despliegue seguro con credenciales de Key Vault..."
                    node ./scripts/deploy.js --token $(ContraseñaBaseDatos)
                  displayName: 'Ejecutar script de subida en caliente'
```

---

## Resumen del Capítulo

* Azure DevOps organiza sus flujos en **Stages** (grandes fases de entorno), **Jobs** (unidades de ejecución agrupadas) y **Tasks** (pasos procedimentales de terminal).
* La integración con **Azure Key Vault** mediante *Variable Groups* garantiza el enmascaramiento y la inmutabilidad de secretos y credenciales de producción sin subirlos a repositorios.
* Las **YAML Templates** implementan la arquitectura *Don't Repeat Yourself (DRY)*, permitiendo empaquetar lógicas complejas de pipeline en plantillas centrales parametrizables.
* El uso de condicionales dinámicas y validación estricta de parámetros en plantillas previene fallos humanos y estandariza la seguridad en miles de proyectos de software en masa.

En el próximo capítulo, ingresaremos al área de DevSecOps para blindar la seguridad del código mediante el estudio de **Seguridad de Código (DevSecOps): Integración y Configuración de Snyk** en producción.

---

[← Capítulo anterior (Capítulo 4)](04-gitlab-cicd-pipelines.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 6) →](06-seguridad-snyk.md)
