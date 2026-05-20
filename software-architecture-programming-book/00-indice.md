# Arquitectura de Software: De la Teoría a la Práctica

## La Guía Definitiva para Convertirte en Arquitecto de Software

---

## Índice General

### Parte I: Fundamentos
1. [¿Qué es la Arquitectura de Software?](01-fundamentos.md) — Definición, historia (1960s→2020s), leyes, síntomas de mala arquitectura, kata práctica.
2. [El Rol del Arquitecto de Software](02-rol-arquitecto.md) — Responsabilidades, día completo, técnicas de negociación, stakeholders, mentoría, carrera.
3. [Atributos de Calidad (Quality Attributes)](03-atributos-calidad.md) — Rendimiento, escalabilidad, disponibilidad, seguridad, mantenibilidad, trade-offs, QAW.

### Parte II: Principios de Diseño
4. [Principios SOLID](04-principios-solid.md) — SRP, OCP, LSP, ISP, DIP con ejemplos, señales de violación, cuándo ignorarlos.
5. [Principios de Diseño Esenciales](05-principios-diseno.md) — DRY, KISS, YAGNI, SoC, composición sobre herencia, fail fast, POLA, Postel's Law.
6. [Domain-Driven Design (DDD)](06-ddd.md) — Ubiquitous Language, Bounded Contexts, Aggregates, Value Objects, Event Storming, Strategic Design.
7. [Arquitectura Hexagonal y Clean Architecture](07-hexagonal-clean.md) — Ports & Adapters, regla de dependencia, capas, estructura de paquetes, cuándo NO usarlas.

### Parte III: Estilos y Patrones
8. [Estilos Arquitectónicos](08-estilos-arquitectonicos.md) — Monolito, Modular Monolith, Microservicios, Event-Driven, SOA, comparativa, cómo elegir.
9. [Patrones de Comunicación](09-patrones-comunicacion.md) — REST, GraphQL, BFF, API Composition, Service Discovery, Service Mesh.
10. [Patrones de Mensajería y Eventos](10-mensajeria-eventos.md) — Kafka vs RabbitMQ, Outbox, Idempotent Consumer, DLQ, ordenamiento.
11. [Patrones de Datos: CQRS y Event Sourcing](11-patrones-datos.md) — Separación lectura/escritura, event sourcing, snapshots, Saga Pattern.
12. [Patrones de Resiliencia](12-resiliencia.md) — Circuit Breaker, Retry con backoff, Bulkhead, Timeout, Fallback, Rate Limiter, Health Checks.

### Parte IV: Protocolos y APIs
13. [Protocolos de Red y Comunicación](13-protocolos.md) — OSI, TCP/UDP, HTTP/1/2/3, QUIC, WebSocket, TLS/mTLS, DNS.
14. [Diseño de APIs](14-diseno-apis.md) — Principios, paginación, errores, idempotencia, webhooks, bulk ops, 8 anti-patrones, checklist.
15. [gRPC y Protocol Buffers](15-grpc.md) — Streaming (4 tipos), interceptors, comparativa con REST, casos de uso, deadlines.

### Parte V: Seguridad
16. [Seguridad en Arquitectura de Software](16-seguridad.md) — STRIDE, OWASP Top 10, Defense in Depth, Secrets Management, Supply Chain.
17. [Autenticación y Autorización](17-auth.md) — JWT, OAuth 2.0 (PKCE), OIDC, RBAC/ABAC, SSO, OPA (Rego policies).

### Parte VI: Rendimiento y Datos
18. [Escalabilidad y Rendimiento](18-escalabilidad.md) — Vertical vs Horizontal, sharding, réplicas, percentiles, load testing, Amdahl, connection pooling, backpressure, consistent hashing, request hedging, checklist.
19. [Estrategias de Caching](19-caching.md) — Cache-Aside, Write-Through, Write-Behind, Redis, CDN, invalidación, anti-patrones.
20. [Bases de Datos y Persistencia a Escala](20-bases-datos.md) — SQL vs NoSQL, polyglot persistence, sharding, migraciones (expand/contract), índices.

### Parte VII: Cloud y DevOps
21. [Cloud Computing y Cloud-Native](21-cloud.md) — IaaS/PaaS/FaaS/SaaS, 12-Factor App, AWS/GCP/Azure, FinOps, multi-cloud.
22. [Contenedores y Orquestación](22-contenedores.md) — Docker, Kubernetes, Helm, estrategias de deploy, resource management.
23. [Serverless y FaaS](23-serverless.md) — Lambda, cold start (causas y mitigaciones), cuándo usarlo y cuándo no.
24. [Infrastructure as Code y CI/CD](24-iac-cicd.md) — Terraform, GitOps, pipelines, estrategias de deploy, Feature Flags, Platform Engineering.

### Parte VIII: Operaciones
25. [Observabilidad: Logging, Métricas y Tracing](25-observabilidad.md) — Logs estructurados, RED/USE, SLO/SLI/SLA, Error Budget, OpenTelemetry, Alerting.
26. [Estrategia, Trade-offs y Toma de Decisiones](26-tradeoffs.md) — ADRs, matriz de decisión, Strangler Fig, deuda técnica, ética.

### Parte IX: Casos Prácticos
27. [El Arquitecto del Futuro](27-futuro.md) — WebAssembly, eBPF, IA, Edge, FinOps, principios atemporales, epílogo.
28. [Caso de Estudio: ShopFlow desde Cero](28-caso-estudio.md) — E-commerce completo: discovery, QAW, Event Storming, ADRs, pagos con Saga, Terraform, Black Friday, microservicios, métricas reales.
29. [Escenarios de Entrevistas de Arquitectura](29-entrevistas.md) — Framework de respuesta, 4 escenarios resueltos con código (TinyURL, WhatsApp, YouTube, Rate Limiter distribuido).

### Parte X: Temas Avanzados
30. [Documentación de Arquitectura](30-documentacion.md) — Modelo C4 (4 niveles), ADRs detallados, diagramas como código, living documentation, anti-patrones.
31. [Estrategias de Testing para Arquitectos](31-testing.md) — Pirámide y honeycomb, contract tests, tests de resiliencia, mutation testing, ambientes efímeros, checklist.
32. [Arquitectura de Datos](32-arquitectura-datos.md) — OLTP vs OLAP, pipelines ETL/ELT, dbt, Data Lake/Warehouse/Lakehouse, Data Mesh, CDC, calidad de datos.

### Parte XI: Gobernanza, Equipos y Resiliencia Organizacional
33. [Gobernanza de Arquitectura y Fitness Functions](33-gobernanza.md) — Fitness functions (con código), ArchUnit, gobernanza federada, architecture reviews, métricas de gobernanza.
34. [Team Topologies — Diseñando Equipos para la Arquitectura Deseada](34-team-topologies.md) — Conway's Law en profundidad, 4 tipos de equipos, 3 modos de interacción, migración gradual, anti-patrones de estructura.
35. [Disaster Recovery y Continuidad de Negocio](35-disaster-recovery.md) — RPO y RTO con ejercicios de cálculo, 4 estrategias (Backup a Active-Active), arquitectura multi-región, DR Plan template, simulacros.
36. [Catálogo de Anti-Patrones Arquitectónicos](36-anti-patrones.md) — 14 anti-patrones con causas, síntomas, prevención y escape: Big Ball of Mud, Distributed Monolith, Nanoservices, Death Star, Framework Obsession, Secretos hardcodeados, y más.

### Apéndices
- [Apéndice A: Árboles de Decisión para Arquitectos](apendice-a-arboles-decision.md) — 10 árboles para las elecciones más comunes (monolito/microservicios, SQL/NoSQL, REST/GraphQL/gRPC, message broker, caché, serverless, K8s, CQRS, auth, particionar/replicar).

- [Apéndice B: Caja de Herramientas del Arquitecto](apendice-b-herramientas.md) — 100+ herramientas por 12 categorías, biblioteca de 15 libros esenciales, newsletters, podcasts, conferencias, roadmap de aprendizaje de 5 años.

---

**Total**: 36 capítulos + 2 apéndices. ~360 KB de contenido diseñado para convertirte en arquitecto de software desde cero.
