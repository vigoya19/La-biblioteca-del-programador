# Capítulo 23: Serverless y FaaS

> "Serverless no significa 'sin servidores'. Significa 'sin preocuparte por servidores'."

## 23.1 ¿Qué es Serverless?

Modelo de ejecución donde el proveedor cloud gestiona totalmente la infraestructura: aprovisionamiento, escalado, parcheo, disponibilidad. Solo escribes y despliegas código.

### Principios Serverless
1. **Sin gestión de servidores**: No hay instancias que mantener.
2. **Escalado automático**: De 0 a miles de requests sin configuración.
3. **Pago por uso**: Pagas solo por lo que ejecutas (no por capacidad ociosa).
4. **Event-driven por naturaleza**: Las funciones responden a eventos.
5. **Stateless**: Cada invocación es independiente.

## 23.2 FaaS (Function as a Service)

El corazón de serverless. Funciones efímeras que responden a triggers.

```
Triggers → Función → Respuesta

Triggers:
├── HTTP (API Gateway)
├── Eventos (S3 upload, DynamoDB stream)
├── Colas (SQS, Kafka)
├── Cron (CloudWatch Events / Scheduler)
└── Otros servicios cloud
```

### Anatomía de una Función

```python
# AWS Lambda
import json

def handler(event, context):
    """
    event:    Datos del trigger (HTTP body, S3 event, etc.)
    context:  Metadatos de ejecución (requestId, timeout, memory)
    """
    # Lógica de negocio
    pedido_id = event['pathParameters']['id']
    pedido = obtener_pedido(pedido_id)

    return {
        'statusCode': 200,
        'body': json.dumps(pedido)
    }

# Cold start: ~100ms-2s (según lenguaje y dependencias)
# Warm start: ~microsegundos
```

### Límites y Restricciones

| | AWS Lambda | GCP Cloud Functions | Azure Functions |
|---|-----------|-------------------|-----------------|
| **Timeout máx** | 15 min | 60 min | 10 min (ilimitado en Premium) |
| **Memoria** | 128MB - 10GB | 128MB - 32GB | 128MB - 14GB |
| **Tamaño código** | 250MB (descomprimido) | 500MB | Variable |
| **Concurrencia** | 1000 (por defecto, escalable) | Variable | Variable |
| **Cold start** | Sí (mejora con Provisioned Concurrency) | Sí | Sí (Premium plan evita) |

## 23.3 Cold Start — El Gran Problema

**Cold start**: Tiempo que tarda el proveedor en inicializar un nuevo entorno de ejecución para tu función (crear contenedor, cargar runtime, inicializar dependencias).

```
Factores que aumentan cold start:
├── Java/JVM > Node.js > Python > Go (por orden de lentitud)
├── Muchas dependencias > pocas dependencias
├── VPC (ENI provisioning) añade ~2-3 segundos
└── Funciones poco invocadas (sin warm instances)
```

### Estrategias de Mitigación

1. **Provisioned Concurrency** (AWS Lambda): Mantiene N instancias siempre calientes.
2. **Lenguajes rápidos**: Go, Rust, Python para funciones con requisitos de latencia.
3. **Empaquetado ligero**: Minimiza dependencias, usa árbol shaking.
4. **Warmers**: Pings periódicos para mantener instancias calientes.
5. **Lambda SnapStart** (AWS, Java): Restaura desde snapshot pre-inicializado.
6. **WebAssembly (Wasm)**: Cold starts de microsegundos (Cloudflare Workers, Fermyon Spin).

## 23.4 Serverless ≠ Solo FaaS

El ecosistema serverless va mucho más allá de funciones:

```
Serverless Services:
├── Compute:     Lambda, Fargate, Cloud Run
├── API:         API Gateway, AppSync (GraphQL)
├── Storage:     S3 (serverless storage)
├── Database:    DynamoDB, Aurora Serverless
├── Messaging:   SQS, SNS, EventBridge
├── Workflows:   Step Functions
├── Auth:        Cognito
└── Edge:        CloudFront + Lambda@Edge / CloudFront Functions
```

## 23.5 ¿Cuándo Usar Serverless?

### Perfecto para:
- **APIs con tráfico variable o impredecible** (escala a 0 cuando no hay tráfico).
- **Procesamiento de eventos** (upload de archivos, cambios en BD, mensajes).
- **Tareas programadas** (cron jobs).
- **ETL y procesamiento de datos** (transformaciones, validaciones).
- **Microservicios pequeños** con responsabilidades bien definidas.
- **Prototipos y MVPs** (sin inversión en infraestructura).

### NO Usar Serverless para:
- **Cargas de trabajo constantes y predecibles** (reservar EC2/VM es más barato).
- **Procesos de larga duración** (streaming continuo, cálculos de 30+ minutos).
- **Latencia ultra-baja predecible** (sin cold starts garantizados).
- **Aplicaciones con estado local** (WebSockets puros, gaming servers).
- **Migraciones lift-and-shift** de aplicaciones monolíticas (adoptar serverless requiere rediseño).

## 23.6 Serverless Framework y Herramientas

```
Herramientas de desarrollo serverless:
├── Serverless Framework (CLI + YAML)
├── AWS SAM (Serverless Application Model)
├── AWS CDK (Infrastructure as actual Code)
├── Terraform (Multi-cloud IaC)
├── SST (Serverless Stack, construido sobre CDK)
└── Pulumi
```

### Ejemplo Serverless Framework

```yaml
service: mi-api-pedidos
provider:
  name: aws
  runtime: python3.11
  region: us-east-1

functions:
  crearPedido:
    handler: handlers/pedidos.crear
    events:
      - http: POST /pedidos
    environment:
      TABLE_NAME: !Ref PedidosTable

resources:
  PedidosTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      KeySchema:
        - AttributeName: id
          KeyType: HASH
```

## 23.7 Observabilidad en Serverless

El debugging de funciones serverless es más complejo que en servidores tradicionales.

- **Structured logging**: JSON logs con requestId, coldStart, duration.
- **Distributed tracing**: AWS X-Ray, OpenTelemetry.
- **Métricas**: CloudWatch Metrics (invocaciones, errores, duración, throttles).
- **Alertas**: CloudWatch Alarms en errores y throttling.

---

> **Reflexión del capítulo**: Serverless es el futuro de gran parte del cómputo, pero no es una solución universal. Entender el cold start, los límites de tiempo y el modelo de costos es esencial. La arquitectura serverless requiere un cambio de mentalidad: piensa en eventos y funciones efímeras, no en servidores y procesos persistentes.

---

← [Capítulo anterior](22-contenedores.md) | [Inicio](README.md) | [Capítulo siguiente →](24-iac-cicd.md)
