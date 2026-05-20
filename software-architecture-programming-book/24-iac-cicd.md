# Capítulo 24: Infrastructure as Code y CI/CD

> "Si no está en código, no existe. Si no está versionado, no pasó."

## 24.1 Infrastructure as Code (IaC)

Gestionar y aprovisionar infraestructura mediante archivos de configuración legibles por máquinas, en lugar de procesos manuales o consolas web.

### Principios de IaC
1. **Infraestructura versionada**: Mismo repo, mismo branching, mismos PRs.
2. **Idempotencia**: Aplicar 1 o 100 veces produce el mismo resultado.
3. **Declarativo sobre imperativo**: Describes el estado deseado, no los pasos.
4. **Inmutable**: No modificas servidores; los reemplazas.

### Terraform (HashiCorp) — El Estándar

```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_ecs_cluster" "main" {
  name = "mi-cluster"
}

resource "aws_ecs_service" "api" {
  name            = "api-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 3
  launch_type     = "FARGATE"
}
```

### Estado de Terraform
Terraform mantiene un archivo de estado (`terraform.tfstate`) que mapea recursos declarados a recursos reales.

```
NUNCA edites el estado a mano.
USA remote state (S3 + DynamoDB lock) para equipos.
NUNCA comitees el state file al repo.
```

### Herramientas de IaC

| Herramienta | Enfoque | Cloud |
|-------------|---------|-------|
| **Terraform** | Declarativo, HCL | Multi-cloud |
| **Pulumi** | Declarativo, lenguajes reales (TS, Python, Go) | Multi-cloud |
| **AWS CDK** | Declarativo, lenguajes reales | AWS |
| **CloudFormation** | Declarativo, YAML/JSON | AWS |
| **Ansible** | Procedural, YAML | Multi-cloud + on-prem |
| **Bicep** | Declarativo, DSL simple | Azure |

## 24.2 GitOps

Evolución de IaC: Git es la fuente única de verdad. Un operador (ArgoCD, Flux) reconcilia el estado declarado en Git con el estado real del clúster.

```
Developer ──► Git (PR/Merge) ──► ArgoCD detecta diff ──► Aplica a K8s
                ▲                                               │
                └─── Si alguien toca K8s manualmente ──────────┘
                     ArgoCD lo revierte automáticamente
```

**Principios GitOps**:
1. Todo en Git (código, config, infra).
2. Git es la fuente de verdad (no el clúster, no la consola).
3. El operador reconcilia automáticamente.
4. Cambios = PR + Merge. Nada manual en producción.

## 24.3 CI/CD (Continuous Integration / Continuous Delivery)

### CI (Continuous Integration)
Integrar cambios frecuentemente (varias veces al día). Cada push dispara: build → test → análisis estático.

```yaml
# GitHub Actions CI
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21' }
      - run: ./gradlew test
      - run: ./gradlew checkstyle
      - run: ./gradlew jacocoTestReport  # Code coverage
      - uses: sonarsource/sonarqube-scan-action@v2  # SAST
```

### CD (Continuous Delivery vs Continuous Deployment)

```
Continuous Delivery:
Código → Build → Test → Staging → (aprobación manual) → Production

Continuous Deployment:
Código → Build → Test → Staging → (test automático) → Production
                                                 ↑
                                          Sin intervención humana
```

**Regla general**: Continuous Delivery para todos los equipos. Continuous Deployment solo cuando la confianza en los tests es absoluta.

### Pipeline Típica

```
┌──────┐   ┌──────┐   ┌────────┐   ┌──────────┐   ┌────────┐
│ Lint │──►│ Test │──►│ Build  │──►│ Security │──►│ Deploy │
│      │   │      │   │Docker  │   │  Scan    │   │ Staging│
└──────┘   └──────┘   └────────┘   └──────────┘   └────┬───┘
                                                        │
                                              ┌─────────▼──────┐
                                              │ Smoke/API Tests│
                                              └─────────┬──────┘
                                                        │
                                              ┌─────────▼──────┐
                                              │Deploy Production│
                                              └────────────────┘
```

### Estrategias de Deploy

| Estrategia | Downtime | Rollback | Complejidad |
|-----------|----------|----------|-------------|
| **Rolling** | 0 | Medio | Baja |
| **Blue/Green** | 0 | Instantáneo | Media |
| **Canary** | 0 | Gradual | Alta |
| **All-at-Once** | Sí | Instantáneo | Muy Baja |
| **Feature Flags** | 0 | Instantáneo | Alta (código) |

### Feature Flags (Feature Toggles)

Separan deploy de release. Despliegas código inactivo que activas por configuración.

```java
if (featureFlagService.isEnabled("nuevo-checkout")) {
    return nuevoCheckoutService.procesar(pedido);
} else {
    return checkoutLegacy.procesar(pedido);
}
```

**Beneficios**: Deploy de día, release cuando quieras. Rollback instantáneo (apaga el flag).

## 24.4 DevOps vs Platform Engineering

```
DevOps tradicional:
Equipo de desarrollo ──► CI/CD + Ops (mismo equipo)

Platform Engineering (evolución moderna):
┌──────────────────────────────┐
│ Internal Developer Platform  │  (Construido por Platform Team)
│                              │
│ "Golden Paths" para deploys  │
│ Templates, CI/CD, Monitoring │
│ Self-service para devs       │
└──────────────────────────────┘
        ▲           ▲
Equipo A        Equipo B   (Consumidores de la plataforma)
```

---

> **Reflexión del capítulo**: IaC y CI/CD no son herramientas, son cultura. La diferencia entre un equipo que despliega cada 3 meses y uno que despliega 20 veces al día no es técnica; es confianza en el pipeline. Construye pipelines que den confianza, y la velocidad llegará sola.
