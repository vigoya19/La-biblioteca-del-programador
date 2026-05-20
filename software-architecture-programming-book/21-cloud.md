# Capítulo 21: Cloud Computing y Cloud-Native

> "La nube no es solo el ordenador de otro. Es un modelo completamente diferente de construir y operar software."

## 21.1 Modelos de Servicio Cloud

```
┌─────────────────────────────────────────────┐
│ On-Premise │ IaaS │  PaaS  │ FaaS │ SaaS    │
├────────────┼──────┼────────┼──────┼─────────┤
│ App        │ App  │ App    │ App  │ App     │ ← Tú
│ Data       │ Data │ Data   │ Data │ Data    │
│ Runtime    │ RT   │ RT     │ RT   │ RT      │
│ OS         │ OS   │ OS     │ OS   │ OS      │
│ Virtualiz. │ Virt │ Virt   │ Virt │ Virt    │
│ Servers    │ Serv │ Serv   │ Serv │ Serv    │
│ Storage    │ Stor │ Stor   │ Stor │ Stor    │
│ Network    │ Net  │ Net    │ Net  │ Net     │ ← Cloud Provider
└────────────┴──────┴────────┴──────┴─────────┘
```

| Modelo | Tú Gestionas | Proveedor Gestiona | Ejemplo |
|--------|-------------|-------------------|---------|
| **IaaS** | OS, runtime, app, data | Virtualización, HW, red | AWS EC2, GCE |
| **PaaS** | App, data | OS, runtime, infra | Heroku, App Engine |
| **FaaS** | Código (función) | Todo lo demás | AWS Lambda |
| **SaaS** | Configuración | Todo | Gmail, Salesforce |

## 21.2 Cloud-Native: Los 5 Pilares

Cloud-Native no es "correr en la nube". Es diseñar específicamente para el modelo cloud:

### 1. Microservicios o Servicios Desacoplados
Cada pieza puede escalar, fallar y actualizarse independientemente.

### 2. Contenedores
Empaquetado inmutable y portátil. Docker como estándar de facto.

### 3. Orquestación Dinámica
Kubernetes gestiona: despliegue, escalado, recuperación, networking.

### 4. Infraestructura como Código (IaC)
Terraform, Pulumi, CloudFormation. La infraestructura se versiona como el código.

### 5. Observabilidad y Automatización
Logging, métricas, tracing + CI/CD. Operas con datos, no con intuición.

## 21.3 Los 12-Factor App

Metodología para construir aplicaciones cloud-native (por Heroku, 2011):

| Factor | Principio |
|--------|-----------|
| 1. Codebase | Un repo por app, múltiples deploys |
| 2. Dependencies | Declarar explícitamente, aislar |
| 3. Config | En variables de entorno, no en código |
| 4. Backing Services | Servicios externos como recursos |
| 5. Build/Release/Run | Separación estricta de etapas |
| 6. Processes | Stateless, share-nothing |
| 7. Port Binding | Exportar servicios vía puerto |
| 8. Concurrency | Escalar horizontalmente |
| 9. Disposability | Arranque rápido, apagado graceful |
| 10. Dev/Prod Parity | Mismos entornos |
| 11. Logs | Tratar logs como streams |
| 12. Admin Processes | Ejecutar como one-off |

**Regla de oro**: Si tu aplicación guarda estado local, usa IPs fijas o depende del filesystem local → no es cloud-native.

## 21.4 Proveedores Cloud

### AWS (Amazon Web Services) — Líder del Mercado
Servicios clave que todo arquitecto debe conocer:

```
Compute:      EC2, Lambda, ECS/EKS (K8s), Fargate
Storage:      S3, EBS, EFS
Database:     RDS, DynamoDB, Aurora, ElastiCache
Networking:   VPC, Route 53, CloudFront, API Gateway
Messaging:    SQS, SNS, EventBridge, Kinesis
Security:     IAM, Cognito, KMS, WAF
Observability: CloudWatch, X-Ray
IaC:          CloudFormation, CDK
```

### GCP (Google Cloud Platform)
Fortaleza: Datos, ML, Kubernetes.

```
Compute:    Compute Engine, GKE, Cloud Run, Cloud Functions
Data:       BigQuery, Cloud Spanner, Firestore, Bigtable
ML/AI:      Vertex AI
```

### Azure (Microsoft)
Fortaleza: Integración con ecosistema Microsoft.

```
Compute:    Azure VMs, AKS, Azure Functions
Data:       SQL Database, Cosmos DB
DevOps:     Azure DevOps
```

## 21.5 Estrategias Multi-Cloud y Hybrid Cloud

### Multi-Cloud
Usar múltiples proveedores cloud para evitar vendor lock-in o aprovechar lo mejor de cada uno.

```
AWS: Cómputo principal + S3
GCP: BigQuery para analytics
Azure: Active Directory para identidad
```

**Realidad**: Multi-cloud es complejo y caro. Solo justificado en grandes empresas con necesidades muy específicas.

### Hybrid Cloud
Parte on-premise, parte cloud.

```
On-Premise (datacenter propio) ◄──► Cloud (AWS/GCP/Azure)
     │                                    │
  Datos sensibles                    Cómputo elástico
  Regulatorios                       CDN, Serverless
```

**Herramientas**: AWS Outposts, Google Anthos, Azure Arc.

## 21.6 Costos en la Nube: FinOps

La nube es OPEX, no CAPEX. Pero sin control, la factura se dispara.

### Estrategias de Optimización
- **Reserved Instances / Savings Plans**: Compromiso 1-3 años → 30-60% descuento.
- **Spot Instances**: Capacidad sobrante → hasta 90% descuento (para cargas interrumpibles).
- **Auto-scaling**: Escalar a cero en horas inactivas.
- **Right-sizing**: No pagues por recursos que no usas.
- **Tagging**: Etiquetar TODO para atribuir costos a equipos/productos.

### Herramientas
- **AWS Cost Explorer**, **GCP Cost Management**, **Azure Cost Management**.
- **Terceros**: CloudHealth, Spot by NetApp, Kubecost (Kubernetes).

---

> **Reflexión del capítulo**: La nube no es magia, es responsabilidad. Te da elasticidad, pero también te da la capacidad de gastar miles de dólares en minutos si no tienes controles. Un arquitecto cloud-native piensa en costos, seguridad y resiliencia desde el minuto cero.
