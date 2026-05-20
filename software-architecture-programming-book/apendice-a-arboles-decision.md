# Apéndice A: Árboles de Decisión para Arquitectos

> "Cuando no sepas qué camino tomar, sigue el árbol. Si el árbol no tiene tu respuesta, al menos te dará las preguntas correctas."

## A.1 ¿Monolito o Microservicios?

```
¿Tienes más de 15 desarrolladores?
├── NO → Monolito Modular. Reevalúa cada 6 meses.
└── SÍ → ¿Múltiples equipos independientes?
    ├── NO → Monolito Modular con módulos por equipo.
    └── SÍ → ¿Necesitas escalar partes específicas?
        ├── NO → Monolito Modular. Más simple de operar.
        └── SÍ → ¿Puedes manejar la complejidad operativa?
            ├── NO → Monolito + réplicas. Contrata DevOps.
            └── SÍ → Microservicios, extrayendo gradualmente.
```

## A.2 ¿Qué Base de Datos Usar?

```
¿Tus datos son altamente relacionales?
├── SÍ → PostgreSQL (o MySQL si ya lo usas)
└── NO → ¿Necesitas esquema flexible?
    ├── NO → ¿Solo key-value?
    │   ├── SÍ → Redis (si cabe en memoria), DynamoDB (si es masivo)
    │   └── NO → PostgreSQL con JSONB (mejor de dos mundos)
    └── SÍ → ¿Necesitas búsquedas textuales avanzadas?
        ├── SÍ → Elasticsearch (como secundario, no fuente de verdad)
        └── NO → ¿Datos de documentos con estructura variable?
            ├── SÍ → MongoDB (si necesitas consultas complejas)
            │       o DynamoDB (si escala masiva y consultas simples)
            └── NO → ¿Time-series / IoT / eventos?
                ├── SÍ → TimescaleDB (SQL) o InfluxDB
                └── NO → ¿Grafos / relaciones complejas?
                    ├── SÍ → Neo4j
                    └── NO → Vuelve a PostgreSQL. En serio.
```

## A.3 ¿REST, GraphQL o gRPC?

```
¿La API es para clientes externos / terceros?
├── SÍ → ¿Los clientes necesitan datos muy flexibles?
│   ├── SÍ → GraphQL
│   └── NO → REST (con OpenAPI)
└── NO → ¿Comunicación service-to-service?
    ├── SÍ → ¿Necesitas máxima performance y streaming?
    │   ├── SÍ → gRPC
    │   └── NO → REST (más simple, más herramientas)
    └── NO → ¿Comunicación asíncrona / eventos?
        ├── SÍ → Mensajería (Kafka/RabbitMQ) + AsyncAPI
        └── NO → REST
```

## A.4 ¿Qué Message Broker Usar?

```
¿Necesitas procesar streams de eventos?
├── SÍ → ¿Con retención a largo plazo y replay?
│   ├── SÍ → Apache Kafka
│   └── NO → ¿Serverless y baja mantención?
│       ├── SÍ → AWS Kinesis, GCP Pub/Sub
│       └── NO → Kafka de todos modos
└── NO → ¿Necesitas routing complejo (topic exchanges, headers)?
    ├── SÍ → RabbitMQ
    └── NO → ¿Solo necesitas colas simples?
        ├── SÍ → AWS SQS, GCP Pub/Sub
        └── NO → ¿Necesitas pub/sub simple?
            ├── SÍ → Redis Pub/Sub (efímero), SNS
            └── NO → RabbitMQ (balance entre simplicidad y potencia)
```

## A.5 ¿Qué Estrategia de Caché?

```
¿El dato se lee mucho más de lo que se escribe? (ratio > 10:1)
├── NO → No cachees. El costo de invalidación supera el beneficio.
└── SÍ → ¿Es contenido público y geográficamente distribuido?
    ├── SÍ → CDN (CloudFront, Cloudflare)
    └── NO → ¿Necesitas consistencia fuerte?
        ├── SÍ → Cache-Aside con invalidación explícita + TTL corto
        └── NO → ¿Son datos de sesión/efímeros?
            ├── SÍ → Redis (en memoria, TTL)
            └── NO → ¿Necesitas cachear resultados de queries BD?
                ├── SÍ → Redis con TTL (según frescura del negocio)
                └── NO → Application cache (in-memory, Caffeine)
```

## A.6 ¿Serverless o Contenedores?

```
¿Tu carga de trabajo es constante y predecible?
├── SÍ → Contenedores (ECS, GKE, EKS). Más barato a largo plazo.
└── NO → ¿Es variable, con picos impredecibles?
    ├── SÍ → ¿La latencia de cold start es aceptable?
    │   ├── SÍ → Lambda / Cloud Functions
    │   └── NO → ¿Puedes usar Provisioned Concurrency?
    │       ├── SÍ → Lambda con Provisioned Concurrency
    │       └── NO → Contenedores con auto-scaling agresivo
    └── NO → ¿Necesitas ejecutar tareas de larga duración (>15 min)?
        ├── SÍ → Contenedores (Fargate, Cloud Run, GKE)
        └── NO → ¿Estás haciendo un MVP?
            ├── SÍ → Serverless (velocidad, sin ops)
            └── NO → Evalúa caso por caso
```

## A.7 ¿Cuándo Adoptar Kubernetes?

```
¿Tienes más de 10 microservicios?
├── NO → ECS Fargate, Cloud Run, o App Runner. K8s es overkill.
└── SÍ → ¿Tu equipo tiene experiencia en K8s?
    ├── NO → ¿Estás dispuesto a invertir 3-6 meses en aprender?
    │   ├── NO → Usa un managed service más simple
    │   └── SÍ → K8s gestionado (EKS, GKE, AKS)
    └── SÍ → ¿Necesitas portabilidad multi-cloud?
        ├── SÍ → Kubernetes (con herramientas multi-cloud)
        └── NO → ¿Necesitas features avanzadas de K8s?
            ├── SÍ → (Service Mesh, Operators, CRDs) → K8s
            └── NO → Evalúa alternativas más simples
```

## A.8 ¿Cuándo Usar CQRS?

```
¿Tus modelos de lectura y escritura son radicalmente diferentes?
├── NO → No uses CQRS. CRUD es suficiente.
└── SÍ → ¿La carga de lecturas es mucho mayor que escrituras? (>10:1)
    ├── NO → CQRS probablemente es overkill.
    └── SÍ → ¿Necesitas escalar lecturas y escrituras independientemente?
        ├── NO → Índices + read replicas es más simple y efectivo.
        └── SÍ → CQRS. Pero considera el costo de la consistencia eventual.
```

## A.9 ¿Qué Protocolo de Autenticación?

```
¿Es una API que consumirán aplicaciones de terceros?
├── SÍ → OAuth 2.0 + OIDC (Authorization Code + PKCE)
└── NO → ¿Es service-to-service?
    ├── SÍ → ¿En la nube?
    │   ├── SÍ → IAM Roles / Workload Identity (sin managed secrets)
    │   └── NO → mTLS o API Keys con rotación
    └── NO → ¿Es una SPA o móvil?
        ├── SÍ → OAuth 2.0 + PKCE + BFF (Backend For Frontend)
        └── NO → ¿Es una web tradicional server-side?
            ├── SÍ → Sesiones con cookies (HttpOnly, Secure, SameSite)
            └── NO → JWT si stateless, sesiones si stateful
```

## A.10 ¿Particionar o Replicar?

```
¿Tu base de datos tiene problemas de rendimiento?
├── NO → No toques nada. "If it ain't broke..."
└── SÍ → ¿El problema son las lecturas?
    ├── SÍ → Réplicas de lectura + cache.
    └── NO → ¿El problema son las escrituras?
        ├── SÍ → ¿Tu carga de escritura supera la capacidad vertical?
        │   ├── NO → Scale up (máquina más grande) primero.
        │   └── SÍ → ¿Puedes particionar los datos?
        │       ├── NO → Optimiza queries, revisa índices, archiva datos.
        │       └── SÍ → Sharding. Pero prepárate para la complejidad.
        └── NO → ¿Consultas lentas?
            ├── SÍ → Optimiza queries, añade índices, desnormaliza.
            └── NO → Monitorea y vuelve a diagnosticar.
```

---

> **Reflexión del apéndice**: Estos árboles no reemplazan el análisis profundo. Son atajos mentales, heurísticas basadas en experiencia. Úsalos como punto de partida para la conversación, no como el final de la decisión. Cada sistema es único y merece su propio análisis.
