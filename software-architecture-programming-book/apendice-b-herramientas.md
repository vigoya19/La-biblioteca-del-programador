# Apéndice B: Caja de Herramientas del Arquitecto

> "Un artesano es tan bueno como sus herramientas y su juicio para elegirlas."

## B.1 Herramientas por Categoría

### Diagramación y Documentación
| Herramienta | Tipo | Mejor Para |
|-------------|------|-----------|
| **Structurizr DSL** | Texto | C4 como código, versionable |
| **PlantUML** | Texto | UML, C4, secuencia, ER |
| **Mermaid** | Texto | Diagramas en Markdown/GitHub |
| **Excalidraw** | Visual | Diagramas rápidos estilo pizarra |
| **diagrams.net** | Visual | Diagramas visuales completos |
| **Lucidchart** | Visual | Colaboración empresarial |
| **Ilograph** | Web | Diagramas interactivos |
| **Eraser.io** | Web | Diagramas + docs colaborativos |

### API Design
| Herramienta | Propósito |
|-------------|-----------|
| **OpenAPI/Swagger** | Especificación REST |
| **AsyncAPI** | Especificación de eventos/WebSocket |
| **GraphQL Schema** | Especificación GraphQL |
| **Protobuf** | Especificación gRPC |
| **Stoplight Studio** | Editor visual de APIs |
| **Postman** | Testing, documentación, mock servers |
| **Bruno** | Cliente API open source (alternativa a Postman) |
| **Hoppscotch** | Cliente API web ligero |

### Desarrollo y Testing
| Herramienta | Propósito |
|-------------|-----------|
| **TestContainers** | BD/MQ reales para tests |
| **WireMock** | Mock de APIs HTTP |
| **Pact** | Contract testing |
| **k6** | Load testing |
| **Artillery** | Load testing YAML |
| **Gatling** | Load testing (Scala/Java DSL) |
| **Pitest** | Mutation testing (Java) |
| **Stryker** | Mutation testing (JS/.NET) |
| **OWASP ZAP** | DAST security testing |

### DevOps e Infraestructura
| Herramienta | Propósito |
|-------------|-----------|
| **Terraform** | IaC multi-cloud |
| **OpenTofu** | Fork open source de Terraform |
| **Pulumi** | IaC con lenguajes reales |
| **AWS CDK** | IaC para AWS (TS, Python, Go) |
| **Ansible** | Config management |
| **GitHub Actions** | CI/CD |
| **GitLab CI** | CI/CD integrado |
| **ArgoCD** | GitOps para Kubernetes |
| **Flux** | GitOps para Kubernetes |
| **Helm** | Package manager K8s |
| **Kustomize** | Config K8s sin templates |

### Observabilidad
| Herramienta | Tipo |
|-------------|------|
| **OpenTelemetry** | Estándar de instrumentación |
| **Grafana** | Dashboards |
| **Prometheus** | Métricas + alertas |
| **Grafana Loki** | Logs (como Prometheus pero para logs) |
| **Elastic Stack** | Logs + búsqueda |
| **Jaeger** | Distributed tracing |
| **Datadog** | Plataforma unificada ($$$) |
| **Honeycomb** | Observabilidad para high-cardinality |
| **Sentry** | Error tracking |
| **PagerDuty** | On-call management |

### Mensajería y Streaming
| Herramienta | Tipo |
|-------------|------|
| **Apache Kafka** | Streaming distribuido |
| **Redpanda** | Kafka-compatible, más rápido |
| **RabbitMQ** | Message broker AMQP |
| **NATS** | Messaging ultra-ligero |
| **AWS SQS/SNS** | Mensajería serverless |
| **AWS Kinesis** | Streaming serverless |
| **GCP Pub/Sub** | Mensajería serverless |
| **Temporal** | Workflow engine |

### Bases de Datos
| Herramienta | Tipo |
|-------------|------|
| **PostgreSQL** | Relacional (la respuesta correcta el 80% del tiempo) |
| **MySQL/MariaDB** | Relacional |
| **MongoDB** | Documentos |
| **Redis** | Key-value / caché |
| **DynamoDB** | Key-value serverless |
| **Cassandra/ScyllaDB** | Wide-column |
| **Neo4j** | Grafos |
| **Elasticsearch** | Búsqueda textual |
| **TimescaleDB** | Time-series SQL |
| **InfluxDB** | Time-series |
| **CockroachDB** | SQL distribuido |
| **Spanner** | SQL global (GCP) |

### Datos y Analytics
| Herramienta | Tipo |
|-------------|------|
| **Snowflake** | Data warehouse cloud |
| **BigQuery** | Data warehouse serverless |
| **Databricks** | Lakehouse |
| **dbt** | Transformaciones SQL |
| **Airbyte/Fivetran** | Ingesta ELT |
| **Airflow/Dagster** | Orquestación |
| **Debezium** | CDC |
| **Great Expectations** | Data quality |
| **DataHub** | Data catalog |
| **Apache Iceberg/Delta Lake** | Formatos lakehouse |

### Seguridad
| Herramienta | Tipo |
|-------------|------|
| **HashiCorp Vault** | Secrets management |
| **AWS Secrets Manager** | Secrets rotación automática |
| **Snyk** | SCA + vulnerabilidades |
| **Dependabot** | Actualización de dependencias |
| **Trivy** | Escaneo de imágenes |
| **SonarQube** | SAST + calidad de código |
| **Keycloak** | Identity provider open source |
| **Ory** | Auth (Kratos, Hydra) open source |
| **OPA (Open Policy Agent)** | Policy as code |
| **Cert-Manager** | Gestión TLS en K8s |

## B.2 Libros Esenciales (Biblioteca del Arquitecto)

### Fundamentos
- *Fundamentals of Software Architecture* — Mark Richards, Neal Ford ← **Empieza aquí**
- *Software Architecture: The Hard Parts* — Ford, Richards, et al. ← **Decisiones difíciles**
- *Building Evolutionary Architectures* — Ford, Parsons, Kua ← **Cómo evolucionar**
- *Designing Data-Intensive Applications* — Martin Kleppmann ← **La biblia de datos**

### Especializados
- *Domain-Driven Design* — Eric Evans ← **El libro azul**
- *Implementing DDD* — Vaughn Vernon ← **DDD práctico**
- *Building Microservices* — Sam Newman ← **Microservicios pragmáticos**
- *Monolith to Microservices* — Sam Newman ← **Migración**
- *Building Event-Driven Microservices* — Adam Bellemare ← **Kafka + eventos**
- *The DevOps Handbook* — Gene Kim et al. ← **DevOps práctico**
- *Accelerate* — Nicole Forsgren ← **DORA, métricas**
- *Team Topologies* — Matthew Skelton, Manuel Pais ← **Organización**
- *Continuous Delivery* — Jez Humble, Dave Farley ← **CD desde cero**

### Cultura y Pensamiento
- *The Phoenix Project* — Gene Kim ← **Novela sobre DevOps**
- *The Unicorn Project* — Gene Kim ← **Novela sobre desarrollo**
- *The Pragmatic Programmer* — Hunt, Thomas ← **Sabiduría atemporal**
- *A Philosophy of Software Design* — John Ousterhout ← **Simplicidad**
- *Staff Engineer* — Will Larson ← **Liderazgo técnico**

## B.3 Newsletters y Blogs

| Recurso | Frecuencia | Enfoque |
|---------|-----------|---------|
| **Pragmatic Engineer** | Semanal | Ingeniería, carrera, industria |
| **InfoQ Architecture** | Semanal | Tendencias, casos de estudio |
| **Martin Fowler's Blog** | Variable | Patrones, arquitectura, agile |
| **ThoughtWorks Radar** | Semestral | Tecnologías evaluadas |
| **High Scalability** | Semanal | Casos de sistemas a escala |
| **Bytes.dev** | Semanal | JavaScript/TypeScript |
| **SRE Weekly** | Semanal | Reliability, operaciones |
| **Data Engineering Weekly** | Semanal | Datos, pipelines |
| **Last Week in AWS** | Semanal | AWS, humor |
| **Cloud Native Now** | Variable | CNCF, Kubernetes |

## B.4 Podcasts

| Podcast | Enfoque |
|---------|---------|
| **Software Engineering Daily** | Temas técnicos profundos |
| **InfoQ Podcast** | Arquitectura, tendencias |
| **Architecture Weekly** | Patrones, casos reales |
| **The Changelog** | Open source, comunidad |
| **Kubernetes Podcast** | K8s, cloud-native |
| **Data Engineering Podcast** | Datos, pipelines |
| **Honeycomb.io Podcast** | Observabilidad |
| **ThoughtWorks Podcast** | Tecnología + negocio |

## B.5 Conferencias

| Evento | Ubicación | Enfoque |
|--------|-----------|---------|
| **QCon** | SF, London, NY | Arquitectura general |
| **GOTO** | Copenhagen, Chicago | Desarrollo + arquitectura |
| **NDC** | Oslo, London, Sydney | .NET + polyglot |
| **KubeCon** | Global | Cloud-native, K8s |
| **re:Invent** | Las Vegas | AWS |
| **Google Cloud Next** | Global | GCP |
| **Strange Loop** | St. Louis | Lenguajes, sistemas |
| **DDD Exchange** | London | Domain-Driven Design |
| **Craft Conference** | Budapest | Software craftsmanship |
| **LeadDev** | Global | Liderazgo técnico |

## B.6 Skills Autodidacta — Roadmap de Aprendizaje

```
Año 1-2: Fundamentos sólidos
├── Leer: Fundamentals of Software Architecture + Designing Data-Intensive Apps
├── Patrones: SOLID, Gang of Four, Enterprise Integration Patterns
├── Data: PostgreSQL en profundidad (índices, explain analyze, tuning)
├── Protocolos: HTTP/2, TCP/IP, TLS (hacer handshake con Wireshark)
└── Proyecto: Construir sistema simple con Clean Architecture

Año 3-4: Profundización
├── Leer: DDD + Building Microservices + Building Event-Driven Microservices
├── Cloud: Certificar en AWS/GCP/Azure (Solutions Architect)
├── K8s: Desplegar clúster, entender operators, service mesh
├── Eventos: Kafka (productores, consumidores, Streams API)
├── SRE: Implementar SLOs, alertas, runbooks
└── Proyecto: Migrar monolito a microservicios (simulado)

Año 5+: Maestría
├── Leer: Software Architecture: The Hard Parts + Team Topologies
├── Organizaciones: Cómo la estructura de equipos afecta la arquitectura
├── Negocio: Finanzas, producto, estrategia — hablar con C-level
├── Mentoría: Enseñar a la siguiente generación
├── Especialización: Elegir 1-2 áreas de profundidad (datos, seguridad, edge)
└── Contribución: Charlas en conferencias, open source, escribir
```

---

> **Reflexión del apéndice**: Las herramientas van y vienen. La habilidad del arquitecto no está en conocer 100 herramientas, sino en saber elegir la correcta para el problema correcto. Construye tu caja de herramientas con criterio, no con moda.

---

← [Capítulo anterior](apendice-a-arboles-decision.md) | [Inicio](README.md)
