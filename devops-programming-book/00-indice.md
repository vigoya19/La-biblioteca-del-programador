# El Libro de DevOps: Integración y Entrega Continua (CI/CD) de Alto Rendimiento

Bienvenido al volumen de **DevOps y Tuberías de Entrega Continua de Alto Rendimiento**. Este libro ha sido diseñado como una guía curricular avanzada de ingeniería de sistemas de software para llevarte desde la cultura de integración continua hasta la orquestación multicloud, la seguridad atómica DevSecOps (Snyk, SonarQube), y la automatización híbrida (Terraform, ArgoCD, Kubernetes).

---

## Índice General

1. [Capítulo 1: Fundamentos de DevOps y Cultura de Entrega Continua](01-fundamentos-devops.md)
   - El modelo CALMS, los tres caminos de DevOps y métricas DORA críticas.
   - *Analogía didáctica*: *🏭 La Fábrica Automotriz de Montaje Continuo y Robótico*.

2. [Capítulo 2: Estrategias de Ramificación: GitFlow, Trunk-Based Development y GitLab Flow](02-estrategias-ramificacion-git.md)
   - Pros y contras de GitFlow, Trunk-Based Development con Feature Flags, y GitLab Flow por ramas de entorno.
   - *Analogía didáctica*: *🛤️ Las Agujas del Ferrocarril y las Vías de Alta Velocidad*.

3. [Capítulo 3: GitHub Actions Internals y Sintaxis de Workflows](03-github-actions-workflows.md)
   - Runners (GitHub-hosted vs. Self-hosted), matriz de jobs concurrentes, caché de dependencias en caliente.
   - *Analogía didáctica*: *🤖 El Almacén Automatizado con Robots Clasificadores (GitHub Actions Runners)*.
   - Implementación real de un workflow YAML de TypeScript con caché de Node.

4. [Capítulo 4: GitLab CI/CD: Pipelines Multi-Project y Dynamic Pipelines](04-gitlab-cicd-pipelines.md)
   - Arquitectura de GitLab CI, runners distribuidos, triggers entre repositorios y pipelines generados dinámicamente.
   - *Analogía didáctica*: *📦 La Cinta Transportadora con Brazos Mecánicos Múltiples*.
   - Implementación de tubería GitLab multiproyecto para microservicios.

5. [Capítulo 5: Azure DevOps Pipelines: YAML Reutilizable y Plantillas](05-azure-devops-pipelines.md)
   - Agent Pools, Variable Groups, Azure Key Vault integration, y plantillas YAML parametrizadas.
   - *Analogía didáctica*: *🏢 El Rascacielos Modular Pre-ensamblado (Plantillas YAML)*.
   - Código YAML de despliegue reutilizable.

6. [Capítulo 6: Seguridad de Código (DevSecOps): Integración y Configuración de Snyk](06-seguridad-snyk.md)
   - Software Composition Analysis (SCA) y Static Application Security Testing (SAST).
   - *Analogía didáctica*: *🛡️ El Inspector de Control de Ingredientes de Alimentos*.
   - Instalación paso a paso de Snyk y automatización en un workflow de GitHub Actions.

7. [Capítulo 7: Calidad de Código y Cobertura: Instalación y Configuración de SonarQube](07-calidad-sonarqube.md)
   - Deuda técnica, Code Smells, Quality Gates y arquitectura física de SonarQube Server.
   - *Analogía didáctica*: *🩺 El Escáner Médico de Chequeo Corporal Completo*.
   - Guía de instalación mediante Docker Compose con PostgreSQL persistido e integración de Sonar Scanner.

8. [Capítulo 8: Dockerización y Pipelines de Contenedores](08-docker-pipelines.md)
   - Imágenes Multi-Stage ultra optimizadas, Docker Layer Caching y despliegues automáticos a registros seguros.
   - *Analogía didáctica*: *📦 La Estandarización del Contenedor Marítimo y la Grúa del Puerto*.

9. [Capítulo 9: Infrastructure as Code (IaC) con Terraform en Pipelines](09-terraform-pipelines.md)
   - Control de estados con Terraform Backend (locks de S3/DynamoDB), plan/apply atómico en CI, y Drift Detection.
   - *Analogía didáctica*: *📐 Los Planos Impresos en 3D del Arquitecto y el Constructor Robótico*.

10. [Capítulo 10: GitOps y Despliegue Continuo con ArgoCD](10-gitops-argocd.md)
    - Reconciliación continua Pull en GitOps, ArgoCD controllers, Self-Healing y desvíos lógicos.
    - *Analogía didáctica*: *⚓ El Piloto Automático del Barco y el Puerto de Destino Fijo*.

11. [Capítulo 11: Orquestación en Kubernetes: Helm y Kustomize en CI/CD](11-helm-kustomize-kubernetes.md)
    - Empaquetamiento mediante Helm Charts, overlays de Kustomize y renderizado de manifiestos YAML.
    - *Analogía didáctica*: *🏗️ El Kit de Piezas de Construcción LEGO con Guía de Instrucciones*.

12. [Capítulo 12: Estrategias de Despliegue Avanzadas: Blue-Green y Canary Deployments](12-estrategias-despliegue.md)
    - Enrutamiento progresivo (Istio/Nginx Ingress), Blue-Green deployments, y rollbacks basados en Prometheus.
    - *Analogía didáctica*: *🔀 Los Interruptores de Desvío del Acueducto de la Ciudad*.

13. [Capítulo 13: Secretos y Seguridad de Pipeline: HashiCorp Vault e Integraciones](13-seguridad-secretos-vault.md)
    - Secretos efímeros en caliente con HashiCorp Vault y autenticación federada OIDC sin credenciales.
    - *Analogía didáctica*: *🔑 La Caja Fuerte de Combinación Dinámica de un Solo Uso*.

14. [Capítulo 14: Observabilidad en Entrega Continuo: Grafana, Prometheus y OpenTelemetry](14-observabilidad-devops.md)
    - Telemetría en pipelines CD, Golden Signals, trazas distribuidas y Alertmanager.
    - *Analogía didáctica*: *📟 La Sala de Control con Pantallas Clínicas de Pacientes*.

15. [Capítulo 15: Proyecto Integrador: Pipeline Global Híbrido Multicloud](15-proyecto-integrador-devops.md)
    - Orquestación final unificada de compilación, análisis Snyk y SonarQube, dockerización, Terraform y GitOps.
    - *Analogía didáctica*: *🚀 El Centro de Lanzamiento Espacial de Cabo Cañaveral*.
