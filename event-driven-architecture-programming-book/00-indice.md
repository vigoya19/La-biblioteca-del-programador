# Arquitectura Orientada a Eventos: Guia Completa

## Indice General

### Parte I: Fundamentos Teoricos

1. [Capitulo 1: Fundamentos de la Arquitectura Orientada a Eventos](01-fundamentos.md)
   - ¿Que es Event-Driven Architecture (EDA)?
   - Eventos vs comandos vs consultas
   - Paradigmas: request-driven vs event-driven
   - El Manifiesto Reactivo y EDA
   - Topologias: broker, mediator, choreography vs orchestration
   - Modelo de madurez EDA: de eventos simples a event sourcing

2. [Capitulo 2: Eventos, Comandos y Mensajes](02-eventos-comandos-mensajes.md)
   - Anatomia de un evento: envelope, payload, metadata
   - Tipos de eventos: domain, integration, notification, system
   - Comandos: intencion explicita de cambiar estado
   - Mensajes vs eventos: diferencias semanticas
   - Patrones de mensajeria: pub/sub, point-to-point, request/reply
   - Idempotencia, deduplicacion y ordenamiento

3. [Capitulo 3: Topologias y Patrones de Comunicacion](03-topologias-patrones.md)
   - Choreography vs Orchestration: pros y contras
   - Event Collaboration: servicios colaboran via eventos
   - Event Notification: notificar sin compartir estado
   - Event-Carried State Transfer: eventos como fuente de verdad
   - Competing Consumers y fan-out
   - Dead Letter Queue y retry patterns

### Parte II: Infraestructura y Tecnologias

4. [Capitulo 4: Apache Kafka en Profundidad](04-apache-kafka.md)
   - Arquitectura interna: brokers, particiones, replicacion
   - Productores: acks, idempotencia, compresion
   - Consumidores: grupos, offset management, rebalanceo
   - Kafka Streams y ksqlDB
   - Schema Registry y Avro
   - Kafka Connect para integracion
   - Configuracion de produccion y operaciones

5. [Capitulo 5: AWS: SQS, SNS y EventBridge](05-aws-messaging.md)
   - SQS: colas standard vs FIFO, visibilidad, dead-letter
   - SNS: notificaciones pub/sub, filtrado de mensajes
   - EventBridge: bus de eventos serverless, reglas, schema registry
   - Lambda como consumidor de eventos
   - Step Functions para orquestacion
   - Comparativa: cuando usar cada servicio

6. [Capitulo 6: RabbitMQ y AMQP](06-rabbitmq.md)
   - Modelo AMQP: exchanges, queues, bindings, routing keys
   - Tipos de exchange: direct, fanout, topic, headers
   - Confirmaciones, transacciones y confiabilidad
   - Dead Letter Exchanges y TTL
   - Clustering y alta disponibilidad
   - Comparativa RabbitMQ vs Kafka vs SQS

7. [Capitulo 7: NATS y Mensajeria Ligera](07-nats.md)
   - NATS Core: pub/sub y request/reply
   - JetStream: persistencia, streaming, consumers
   - Key-Value Store y Object Store
   - Comparativa NATS vs Kafka para edge/IoT

### Parte III: Patrones Avanzados

8. [Capitulo 8: Domain Events y Domain-Driven Design](08-domain-events-ddd.md)
   - Domain Events como ciudadanos de primera clase
   - Event Storming: descubrir eventos del negocio
   - Aggregates que emiten eventos
   - Integration Events vs Domain Events
   - Bounded Contexts y relaciones via eventos
   - Anti-Corruption Layer para traduccion de eventos

9. [Capitulo 9: Event Sourcing](09-event-sourcing.md)
   - El patron: almacenar eventos, no estado
   - Event Store: requisitos y diseno
   - Snapshotting para rendimiento
   - Reconstruccion de estado y proyecciones
   - Ventajas: auditoria, temporal queries, debugging
   - Desventajas: complejidad, eventual consistency, versionado

10. [Capitulo 10: CQRS (Command Query Responsibility Segregation)](10-cqrs.md)
    - Separar lecturas de escrituras
    - Modelo de escritura: comandos + event sourcing
    - Modelo de lectura: proyecciones materializadas
    - Sincronizacion entre modelos
    - CQRS sin Event Sourcing (simplificado)
    - Casos de uso reales: fintech, e-commerce, IoT

11. [Capitulo 11: Sagas y Process Managers](11-sagas.md)
    - El problema de las transacciones distribuidas
    - Saga: secuencia de transacciones locales con compensacion
    - Coreografia vs Orquestacion de sagas
    - Implementacion de saga orquestada con AWS Step Functions
    - Process Manager como alternativa a la saga
    - Manejo de fallos, reintentos y compensacion

12. [Capitulo 12: Eventual Consistency](12-eventual-consistency.md)
    - CAP Theorem: Consistency, Availability, Partition Tolerance
    - Modelos de consistencia: fuerte, eventual, causal, monotonic
    - Conflict Resolution: last-write-wins, CRDTs, custom merge
    - Lecturas monotonicas y consistencia de prefijo
    - Como disenar UX para consistencia eventual
    - Patrones: read-your-writes, session consistency

### Parte IV: Implementacion Practica

13. [Capitulo 13: Sistema de E-Commerce Event-Driven (Node.js + TypeScript)](13-implementacion-ecommerce.md)
    - Arquitectura completa: servicios, eventos, bases de datos
    - Implementacion con Kafka + Node.js
    - Implementacion con AWS EventBridge + Lambda
    - Codigo paso a paso: ordenes, pagos, inventario, envios
    - Eventos: OrdenCreada, PagoProcesado, InventarioReservado, EnvioProgramado
    - Idempotencia, reintentos y DLQ en la practica

14. [Capitulo 14: Schema Evolution y Compatibilidad](14-schema-evolution.md)
    - El problema del versionado de eventos
    - Schema Registry: Avro, Protobuf, JSON Schema
    - Compatibilidad: backward, forward, full
    - Estrategias de migracion: upcasting, versionado dual, transform-on-read
    - Eventos auto-descriptivos con CloudEvents
    - Gobierno de esquemas en equipos grandes

15. [Capitulo 15: Testing en Sistemas Event-Driven](15-testing.md)
    - Test pyramid para sistemas de eventos
    - Unit testing de handlers de eventos
    - Integration testing con Testcontainers (Kafka, PostgreSQL)
    - Consumer-driven contract testing con Pact
    - Simulacion de fallos y chaos engineering
    - Testing de sagas y flujos end-to-end

16. [Capitulo 16: Observabilidad y Monitoreo](16-observabilidad.md)
    - Trazas distribuidas con OpenTelemetry
    - Metricas: lag de consumidor, throughput, DLQ size, event latency
    - Logging estructurado con contexto de traza
    - Health checks para infrastructura de eventos
    - Alertas: que monitorear y thresholds
    - Debugging: reconstruir flujos desde eventos

17. [Capitulo 17: Anti-Patrones y Buenas Practicas](17-anti-patrones.md)
    - Anti-patrones: event god, callback hell, data overload, ghost events
    - Event-first design: disenar pensando en eventos
    - Granularidad de eventos: ni muy finos ni muy gruesos
    - Nombrado de eventos: convenciones y taxonomia
    - Owner de esquemas y gobernanza de eventos
    - Decision framework: cuando usar EDA y cuando no
