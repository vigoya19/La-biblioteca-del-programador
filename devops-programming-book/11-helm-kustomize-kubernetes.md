# Capítulo 11: Orquestación en Kubernetes: Helm y Kustomize en CI/CD

> "El mantenimiento de manifiestos duplicados de Kubernetes para diferentes entornos es una fuente masiva de inconsistencias y fallos operativos en caliente. La infraestructura de contenedores exige motores de empaquetamiento y plantillas dinámicas que permitan abstraer la configuración específica de cada entorno de forma limpia y mantenible."

Cuando desplegamos aplicaciones en **Kubernetes (K8s)**, nos encontramos con que cada microservicio requiere múltiples manifiestos declarativos: un `Deployment` para las réplicas, un `Service` para la red, un `Ingress` para exponerlo al exterior y un `ConfigMap` para las variables de entorno.

Si gestionas de forma imperativa estos archivos por cada entorno físico (`desarrollo`, `staging`, `producción`), terminarás con cientos de archivos YAML idénticos dispersos y difíciles de actualizar. En este capítulo, desglosaremos las dos herramientas de orquestación y plantillas líderes en Kubernetes: **Helm** (el gestor de paquetes por excelencia) y **Kustomize** (la solución declarativa de overlays nativa de `kubectl`), y diseñaremos una configuración de overlays con Kustomize lista para producción.

---

## 11.1 Helm: El Gestor de Paquetes de Kubernetes

**Helm** opera de forma análoga a administradores de paquetes como `npm` de Node.js o `pip` de Python, pero diseñado para aplicaciones complejas de Kubernetes.

### Conceptos Clave de Helm:
* **Chart**: Un paquete empaquetado de Helm que contiene todos los manifiestos YAML parametrizados mediante el motor de plantillas de Go (Go Templates).
* **`values.yaml`**: El archivo que contiene las variables y parámetros por defecto que alimentan los manifiestos dinámicos.
* **Release**: Una instancia de ejecución física del Chart instalada en el clúster de K8s. Puedes actualizar o hacer rollback de una release en caliente en microsegundos con un solo comando.

---

## 11.2 Kustomize: Overlays Sin Plantillas (Template-Free Customization)

A diferencia de Helm (que requiere modificar los archivos YAML convirtiéndolos en plantillas ilegibles con llaves dobles `{{ .Values.replicaCount }}`), **Kustomize** adopta una filosofía **libre de plantillas (Template-Free)**.

Kustomize se basa en la herencia y superposición de capas lógicas (**Base y Overlays**):
* **Base (Cimiento)**: Contiene los manifiestos YAML ordinarios y estándar de Kubernetes sin ninguna plantilla ni variable especial.
* **Overlays (Capas de Entorno)**: Carpetas dedicadas a cada entorno físico (ej. `staging`, `production`) que contienen parches lógicos minúsculos. Kustomize toma los parches de la capa del entorno y los funde atómicamente sobre los manifiestos Base en caliente.

---

> [!NOTE]
> ### 🏗️ El Kit de Piezas de Construcción LEGO con Guía de Instrucciones
> 
> Entendamos la diferencia conceptual entre Helm y Kustomize en Kubernetes utilizando una analogía física de juguetes infantiles:
> 
> - **El Enfoque Tradicional Monolítico (El Castillo de Arcilla Rígido)**:
>   - Imagina que quieres construir 3 castillos de juguete idénticos pero de diferentes colores (Tus entornos `Dev`, `Staging` y `Prod`).
>   - Moldeas los tres castillos completos en arcilla sólida y los horneas (**Manifiestos lógicos YAML duplicados**). 
>   - Si al cabo de un mes decides que la puerta de entrada de los tres castillos debe ser redonda en lugar de cuadrada, tendrás que tomar un cincel y esculpir físicamente las tres puertas por separado en la arcilla dura arriesgándote a romper las murallas por completo.
> 
> - **El Enfoque Helm (El Molde de Plástico Inyectado con Filtros)**:
>   - Construyes un **Molde Industrial de Plástico Inyectado (Helm Chart)**.
>   - El molde tiene espacios dinámicos que se pueden rellenar insertando diferentes filtros y embudos a la entrada antes de inyectar el plástico (**Las variables del `values.yaml`**).
>   - Si inyectas plástico azul con el filtro de 3 torres, obtienes el Castillo Dev. Si inyectas plástico rojo con el filtro de 5 torres, obtienes el Castillo Prod. 
>   - Modificar la forma del castillo exige ir al molde principal e inyectar el parámetro correcto.
> 
> - **El Enfoque Kustomize (El Kit de Piezas de Construcción LEGO y los Adhesivos)**:
>   - En lugar de moldear plástico, decides construir el castillo utilizando un **Kit de Piezas de Construcción LEGO estándar e inalterable (El Manifiesto Base)**.
>   - Sigues la guía y construyes la estructura estándar de ladrillos grises.
>   - Para crear los diferentes entornos, no desmontas los LEGOs. Compras **Kits de Adhesivos y Piezas Decorativas Especiales (Los Overlays)**:
>     - Para el entorno de *Desarrollo (Dev)*, le pegas adhesivos de grafitis a las paredes y colocas 2 soldados de juguete.
>     - Para el entorno de *Producción (Prod)*, le colocas un puente levadizo dorado de metal, cambias los ladrillos grises por ladrillos blindados negros (**Parches DML**), y agregas 50 soldados fuertemente armados.
>     - El cimiento del castillo de LEGO sigue siendo exactamente el mismo para todos; solo cambian las capas decorativas que aplicas encima con total limpieza y modularidad.

---

## 11.3 Código YAML: Configuración de Overlays con Kustomize

A continuación, implementaremos la estructura de archivos real y unificada para configurar **Kustomize** en caliente. Definiremos la carpeta del overlay de **Staging** que aplica parches lógicos de replicas, nombres y parches JSON directos sobre la tabla de manifiestos base de Kubernetes:

### `kustomization.yaml`
```yaml
# 1. Vincular el cimiento físico común (Base)
resources:
  - ../../base

# 2. Agregar un prefijo automático de nombre a todos los recursos de este entorno
namePrefix: staging-

# 3. Inyectar etiquetas y labels comunes de forma automática a todos los manifiestos
commonLabels:
  environment: staging
  managed-by: kustomize

# 4. Parche Lógico: Mutar la configuración específica del deployment de staging
patches:
  - target:
      kind: Deployment
      name: api-financiera
    patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: api-financiera
      spec:
        # Aumentamos a 3 réplicas físicas exclusivamente en este entorno
        replicas: 3
        template:
          spec:
            containers:
              - name: app-container
                # Inyectamos variables de entorno específicas para staging
                env:
                  - name: NODE_ENV
                    value: "staging"
                  - name: MONGO_URL
                    value: "mongodb://staging-db:27017/finanzas"
```

---

## Resumen del Capítulo

* **Helm** unifica y paquetiza manifiestos de Kubernetes mediante variables lógicas en archivos `values.yaml` procesados por el motor de plantillas de Go.
* **Kustomize** implementa un enfoque libre de plantillas (*Template-Free*) dividiendo la arquitectura en manifiestos comunes (**Base**) y capas decorativas específicas (**Overlays**).
* La directiva **`patches`** de Kustomize permite realizar fusiones atómicas de configuración YAML en caliente sin necesidad de reescribir ni duplicar los archivos base originales.
* Integrar Kustomize con herramientas GitOps (como ArgoCD) permite una promoción transparente de infraestructura por entornos sin duplicaciones de código en masa.

En el próximo capítulo, abordaremos la orquestación del tráfico de red a gran escala mediante el estudio de **Estrategias de Despliegue Avanzadas: Blue-Green y Canary Deployments** en producción.

---

[← Capítulo anterior (Capítulo 10)](10-gitops-argocd.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 12) →](12-estrategias-despliegue.md)
