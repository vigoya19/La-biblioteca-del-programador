# Capítulo 9: Infrastructure as Code (IaC) con Terraform en Pipelines

> "La creación y mantenimiento manual de recursos en consolas cloud es un antipatrón destructivo que genera desviaciones de configuración imposibles de reproducir. La infraestructura debe tratarse con el mismo rigor que el software: declarativa, versionada en Git, auditable y aprovisionada exclusivamente a través de pipelines."

En el ecosistema de DevOps moderno, la infraestructura como código (**IaC**) ha transformado radicalmente el aprovisionamiento de recursos. Herramientas como **Terraform** permiten describir de forma puramente declarativa servidores, redes, bases de datos y políticas de seguridad mediante el lenguaje HCL (HashiCorp Configuration Language).

Sin embargo, correr Terraform desde la computadora local de un ingeniero representa un peligro operativo extremo: riesgo de colisión de cambios concurrentes, pérdida de control del archivo de estado y vulneración de credenciales cloud. En este capítulo, estudiaremos la gestión segura del **Estado de Terraform (State Backend & Locking)**, automatizaremos el flujo `plan` y `apply` en pipelines de CI/CD, y comprenderemos la técnica de **Drift Detection (Detección de Desviaciones)**.

---

## 9.1 El Estado de Terraform: Backend Remoto y State Locking

El corazón operativo de Terraform es el archivo **`terraform.tfstate`**. Este archivo es el registro físico único de verdad del sistema: mapea tus archivos declarativos lógicos de código HCL con los recursos físicos reales creados en tu proveedor de nube (AWS, Azure, GCP).

### El Peligro de Pérdida o Colisión:
Si dos ingenieros ejecutan `terraform apply` simultáneamente desde sus máquinas locales:
1. Ambas ejecuciones intentarán modificar el mismo archivo de estado a la vez, corrompiendo la base de datos de infraestructura física.
2. Pueden crear recursos duplicados o borrar recursos activos creados por el otro de forma accidental.

### La Solución Definitiva de Producción:
* **Backend Remoto (S3 / Azure Blob Storage)**: El archivo de estado se almacena de forma centralizada y segura en un bucket blindado en la nube, con encriptación en reposo y versionado activo de archivos.
* **State Locking (DynamoDB / Consul)**: Antes de iniciar cualquier comando de modificación, Terraform adquiere un candado lógico exclusivo (**Lock**) sobre una base de datos distribuida en caliente. Si otro pipeline o ingeniero intenta aplicar cambios, se le denegará el acceso instantáneamente hasta que el primer proceso termine y libere el candado.

---

## 9.2 Automatización Segura del Flujo en Pipelines de CI/CD

El aprovisionamiento de infraestructura mediante pipelines automatizados debe seguir una estrategia rigurosa de dos etapas de validación lógicas:

### 1. La Etapa de Planificación (`terraform plan`)
* **Qué hace**: Se dispara automáticamente ante cualquier Pull Request. Compara tu código HCL contra la realidad de la nube y genera un reporte descriptivo exacto de qué recursos se van a crear, modificar o destruir.
* **Seguridad**: Permite a los arquitectos e ingenieros senior auditar visualmente el impacto del cambio en el Pull Request antes de fusionarlo a la rama principal.

### 2. La Etapa de Aplicación (`terraform apply`)
* **Qué hace**: Se ejecuta de forma automatizada **únicamente cuando el Pull Request se fusiona con éxito a `main`**. Aprovisiona físicamente los recursos en la nube.
* **Optimización**: Se debe configurar el pipeline para reutilizar exactamente el archivo de plan compilado y aprobado en el paso anterior (`terraform apply tfplan`), evitando cambios de última hora entre la auditoría y el despliegue físico.

---

> [!NOTE]
> ### 📐 Los Planos en 3D del Arquitecto y el Constructor Robótico
> 
> Entendamos el comportamiento de Terraform en Pipelines y el State Lock a través de una analogía física de ingeniería civil:
> 
> - **El Enfoque Tradicional Inseguro (El Aprovisionamiento Manual en la Nube)**:
>   - Imagina que quieres construir una red de acueductos en una ciudad.
>   - En lugar de dibujar planos, le das instrucciones vagas de palabra a cuadrillas de trabajadores informales: *"Excaven un pozo aquí, pongan un tubo allá"* (**crear servidores con clics en la consola web**).
>   - Si al cabo de un año el pozo se rompe, nadie sabrá por dónde pasan los tubos ni qué diámetro tienen. Repararlo implicará excavar a ciegas destruyendo calles y cortando el agua por accidente.
> 
> - **El Enfoque Terraform (Los Planos de Impresión 3D)**:
>   - Diseñas los planos milimétricos del acueducto en un software CAD digital (**El código descriptivo HCL**).
>   - Compras un **Constructor Robótico Autónomo (Terraform Engine)** que lee tus planos CAD y esculpe los acueductos exactamente como están dibujados con total precisión física.
> 
> - **El State Backend y el State Locking (La Bóveda del Plano Único y el Candado del Terreno)**:
>   - Para evitar que los planos se mojen o se pierdan en el fango de la construcción, decides guardar la copia oficial de planos en una **baja fuerte blindada en el banco (El Backend en Amazon S3)**.
>   - Además, para evitar catástrofes, el banco instala un **dispositivo de alerta digital (State Locking con DynamoDB)**.
>   - Si el Diseñador A y el Diseñador B intentan hacer modificaciones al acueducto, el robot exige que se introduzca la llave única. El Diseñador A toma la llave y el sistema bloquea la puerta de la bóveda (**adquiere el Lock**). El Diseñador B tiene prohibido abrir los planos o excavar hasta que el Diseñador A termine su turno y devuelva la llave a la base, protegiendo al acueducto de demoliciones concurrentes absurdas.

---

## 9.3 Código YAML: Automatización Segura de Terraform en GitHub Actions

A continuación, implementaremos la configuración real para automatizar el aprovisionamiento de infraestructura mediante **Terraform** en **GitHub Actions**. El pipeline implementa el flujo de seguridad auditando cambios en Pull Requests (`plan`) y aplicando modificaciones únicamente al fusionar a `main` (`apply`) con persistencia segura de estado:

#### [despliegueTerraform.yml](file:///Users/andres/Documents/biblioteca/devops-programming-book/.github/workflows/despliegueTerraform.yml)
```yaml
name: Tubería de Aprovisionamiento de Infraestructura (Terraform)

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  terraform-pipeline:
    name: Terraform Plan & Apply
    runs-on: ubuntu-latest
    
    # Configuramos variables de entorno para que el pipeline se autentique contra AWS de forma segura
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_DEFAULT_REGION: 'us-east-1'

    steps:
      - name: Clonar código fuente
        uses: actions/checkout@v4

      # 1. Instalar la versión exacta de Terraform CLI en el Runner
      - name: Configurar Terraform CLI
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0

      # 2. Inicializar Terraform (Descarga providers y configura el backend remoto en S3 con DynamoDB lock)
      - name: Inicializar Terraform (Init)
        run: terraform init

      # 3. Comprobar que los archivos HCL cumplan con las reglas de estilo y formato oficiales
      - name: Validar Formato de Código (Fmt)
        run: terraform fmt -check

      # 4. Fase Plan (Se ejecuta en Pull Requests y Push de main para auditar la infraestructura)
      - name: Generar Plan de Infraestructura (Plan)
        run: terraform plan -out=tfplan
        id: plan

      # 5. Fase Apply (Se ejecuta ÚNICAMENTE al fusionar a la rama 'main' de forma automatizada)
      - name: Aplicar Cambios en la Nube (Apply)
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

---

## Resumen del Capítulo

* **Terraform** mapea la infraestructura declarativa en caliente con los recursos cloud físicos mediante el archivo centralizado **`terraform.tfstate`**.
* Almacenar el archivo de estado en un **Backend Remoto** y configurar **State Locking** previene colisiones y corrupciones accidentales ante ejecuciones concurrentes del equipo.
* Automatizar IaC requiere la bifurcación lógica: **`terraform plan`** en Pull Requests para auditoría y aprobación visual, y **`terraform apply`** con plan precargado en merges a `main`.
* **Drift Detection (Detección de Desviaciones)** compara de forma continua el estado deseado contra la realidad física del cloud, detectando modificaciones directas no documentadas de los desarrolladores.

En el próximo capítulo, ingresaremos al modelo moderno de despliegue continuo mediante el estudio de **GitOps y Despliegue Continuo con ArgoCD** en producción.

---

[← Capítulo anterior (Capítulo 8)](08-docker-pipelines.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 10) →](10-gitops-argocd.md)
