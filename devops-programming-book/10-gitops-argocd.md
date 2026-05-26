# Capítulo 10: GitOps y Despliegue Continuo con ArgoCD

> "El modelo tradicional de empujar cambios mediante scripts imperativos en pipelines de CI/CD que poseen credenciales administrativas completas del clúster de producción es un riesgo de seguridad inaceptable. El despliegue continuo moderno debe ser declarativo, reconciliado de forma asíncrona y operado bajo el modelo Pull."

Históricamente, los despliegues se realizaban mediante la automatización basada en empujar (**Push Model**): un pipeline en Jenkins o GitHub Actions ejecutaba comandos imperativos tipo `kubectl apply -f manifest.yaml`. Este modelo requería almacenar credenciales de administración del clúster con privilegios root (`kubeconfig`) dentro de las variables del pipeline, abriendo una brecha crítica ante posibles infiltraciones en el repositorio.

Para resolver este talón de Aquiles de seguridad, nació la filosofía **GitOps**. En este capítulo, desmitificaremos el modelo **Pull vs. Push**, estudiaremos la arquitectura de **ArgoCD** para Kubernetes y diseñaremos un manifiesto de aplicación GitOps real y automatizado.

---

## 10.1 La Revolución de GitOps: Modelo Pull vs. Push

**GitOps** es un paradigma de entrega continua definido por Weaveworks en 2017 que establece que **Git es la única fuente de verdad para la infraestructura y las aplicaciones**.

### 1. El Modelo Push Tradicional (Pipelines Imperativos)
* **Mecánica**: El pipeline de CI/CD compila, crea la imagen Docker y "empuja" el cambio iniciando sesión en el servidor destino de producción a través de la terminal.
* **Problema**: El pipeline tiene demasiados privilegios. Si un hacker compromete tu servidor de CI, comprometerá de forma inmediata todos tus clústeres físicos de producción.

### 2. El Modelo Pull Moderno (Reconciliación Basada en Agentes)
* **Mecánica**: Un agente ligero de software (como **ArgoCD**) corre **dentro** de tu propio clúster de Kubernetes en producción.
* **El Flujo**:
  1. El pipeline de CI/CD solo se encarga de compilar, testear y subir el contenedor al registro. Al finalizar, escribe un pequeño cambio en un repositorio Git separado (**El repositorio de configuración de infraestructura**), indicando la nueva tag de versión de la imagen.
  2. El agente interno (ArgoCD) vigila constantemente el repositorio Git.
  3. Al detectar que el manifiesto en Git cambió, **jala (Pull)** los cambios y los aplica localmente en el clúster.
* **Ventaja**: Ningún sistema externo al clúster de producción tiene credenciales de acceso. La seguridad es total.

---

## 10.2 ArgoCD: Reconciliación Dinámica y Self-Healing

ArgoCD implementa un bucle continuo de reconciliación en Kubernetes:

```
┌────────────────────────────────┐
│ Repositorio Git (Estado Deseado)│
└──────────────┬─────────────────┘
               │
               ▼  (Monitoreo Continuo - GitOps Loop)
        [ ArgoCD Controller ]
               ▲
               │  (Reconciliación y Autorreparación)
┌──────────────┴─────────────────┐
│ Clúster K8s (Estado Físico Real)│
└────────────────────────────────┘
```

* **Out-of-Sync (Fuera de Sincronía)**: Si el manifiesto en Git indica que la aplicación debe tener 5 réplicas pero el clúster solo tiene 3, ArgoCD detecta el desfase lúdico y cambia el estado a *Out-of-Sync*.
* **Self-Healing (Autorreparación)**: Si un desarrollador descuidado entra por ssh al clúster de producción y borra un despliegue de forma manual con `kubectl delete`, el controlador de ArgoCD detecta la desviación en milisegundos y **vuelve a aprovisionar automáticamente el recurso original descrito en Git**, revirtiendo el error humano de forma instantánea.

---

> [!NOTE]
> ### ⚓ El Piloto Automático del Barco y el Puerto de Destino Fijo
> 
> Entendamos el modelo Push vs Pull y el comportamiento de reconciliación utilizando una analogía física náutica:
> 
> - **El Modelo Push Tradicional (El Barco Controlado por Hilo)**:
>   - Imagina que quieres mover un barco de carga gigante desde una isla central (**El Servidor de CI**) hasta el puerto del continente (**El Clúster de Producción**).
>   - En el modelo Push, atas un cable metálico de 10 kilómetros desde la isla al timón del barco. Un motor en la isla tira del cable de forma violenta para forzar el rumbo del barco.
>   - Si una tormenta desvía el barco de su curso o rompe el cable metálico, el motor en la isla seguirá tirando a ciegas rompiendo el timón y estrellando el barco contra los muelles del puerto. El controlador en la isla no tiene visibilidad en tiempo real de las olas ni del viento del mar de producción.
> 
> - **El Modelo Pull de GitOps (El Piloto Automático con Coordenadas Fijas)**:
>   - Decides no usar cables externos. Instalas una **computadora con piloto automático inteligente dentro del barco (El Agente de ArgoCD en Kubernetes)**.
>   - En lugar de controlarlo desde la isla, le dejas una bitácora digital de navegación fija en un cofre sellado de seguridad en la cabina (**El Repositorio Git de Configuración**).
>   - Las instrucciones son claras: *"Coordenadas físicas de destino = Puerto Sur, réplicas de timones activos = 2"*.
>   - El piloto automático lee las instrucciones (**Pull**) y timonea el barco de forma autónoma con total conocimiento del oleaje local.
>   - **La Reconciliación (Self-Healing)**: Si una ola masiva golpea el barco y apaga uno de los timones, la computadora interna detecta la discrepancia con su bitácora en milisegundos, enciende el timón de emergencia de forma autónoma y estabiliza el barco en el rumbo óptimo sin requerir que ningún operador humano en la isla mueva un solo dedo.

---

## 10.3 Manifiesto YAML de ArgoCD de Nivel de Producción

A continuación, implementaremos un manifiesto de aplicación real en Kubernetes para configurar **ArgoCD**. El archivo describe una **ArgoCD Application** de forma declarativa, vinculando un repositorio Git de configuración con el destino físico del clúster, con políticas de autorreparación (`Self-Healing`) y sincronización automática activas:

### `aplicacionArgoCD.yml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-financiera-saas
  namespace: argocd # Namespace donde reside instalado el operador de ArgoCD
  finalizers:
    # Garantiza una limpieza ordenada eliminando los recursos creados en K8s si se borra este manifiesto
    - resources-finalizer.argocd.argoproj.io
spec:
  # 1. Definir la Fuente de la Verdad (El repositorio Git de Configuración)
  source:
    repoURL: 'https://github.com/software-banco/infraestructura-gitops.git'
    targetRevision: HEAD # Apunta al último commit de la rama por defecto
    path: environments/staging/api-service # Ruta interna donde residen los manifiestos de K8s

  # 2. Definir el Destino Físico (El clúster de Kubernetes objetivo)
  destination:
    server: 'https://kubernetes.default.svc' # Apunta al clúster local donde corre ArgoCD
    namespace: staging-apps # Namespace donde se desplegará físicamente la aplicación

  # 3. Políticas de Sincronización Automática (GitOps Loop)
  syncPolicy:
    automated:
      prune: true       # Borra del clúster cualquier recurso que haya sido eliminado de Git
      selfHeal: true    # Revierte de forma automática y asíncrona cualquier cambio manual hecho en caliente
    syncOptions:
      - CreateNamespace=true # Crea automáticamente el namespace de destino si no existe
```

---

## Resumen del Capítulo

* **GitOps** define a Git como la única fuente de verdad para el estado de la infraestructura y aplicaciones, implementando un modelo seguro de **Pull** basado en agentes.
* **ArgoCD** opera dentro del clúster de Kubernetes resolviendo la reconciliación continua entre el estado deseado en Git y el estado físico real de los recursos.
* La política **`selfHeal`** (autorreparación) de ArgoCD erradica desvíos manuales accidentales de configuración en producción, asegurando la consistencia e inmutabilidad del sistema.
* Centralizar la configuración en repositorios de Git separados e independientes de los repositorios de código fuente simplifica los permisos lógicos del pipeline de CI.

En el próximo capítulo, escalaremos nuestra destreza en empaquetamiento y personalización de recursos en Kubernetes mediante el estudio de **Orquestación en Kubernetes: Helm y Kustomize en CI/CD** en producción.

---

[← Capítulo anterior (Capítulo 9)](09-terraform-pipelines.md) | [Inicio (README.md)](README.md) | [Capítulo siguiente (Capítulo 11) →](11-helm-kustomize-kubernetes.md)
