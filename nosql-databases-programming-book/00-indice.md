# Bases de Datos NoSQL: Guía Completa de Programación y Diseño de Sistemas

Bienvenido al libro de **Bases de Datos NoSQL**. Este libro ha sido diseñado para llevarte desde los fundamentos teóricos del almacenamiento no relacional y los límites de la física de redes (Teorema de CAP) hasta el modelado avanzado de datos de alto rendimiento y la integración de arquitecturas distribuidas multi-base de datos (Polyglot Persistence) listas para producción.

## Índice General

1. [Capítulo 1: Fundamentos de NoSQL y el Teorema de CAP](01-introduccion-y-teorema-cap.md)
   - ¿Qué es NoSQL y por qué surge? Límites de escalabilidad de SQL
   - Arquitecturas distribuidas y el Teorema de CAP (Consistencia, Disponibilidad, Tolerancia a Partición)
   - Clasificación de modelos NoSQL: Documental, Clave-Valor, Ancho de Columna (Wide-column) y Grafos
   - Transacciones ACID clásicas frente a BASE (Basic Availability, Soft state, Eventual consistency)
   - Escalar de forma horizontal (Sharding) frente a vertical (Replicación)

2. [Capítulo 2: MongoDB y el Almacenamiento Orientado a Documentos](02-mongodb.md)
   - Anatomía de un documento: del formato JSON al almacenamiento binario BSON
   - Operaciones CRUD fundamentales y APIs dinámicas en TypeScript con Mongoose
   - Aggregation Framework avanzado: pipelines, stages (`$match`, `$group`, `$lookup`, `$unwind`)
   - Replicación y alta disponibilidad: Replica Sets, árbitros y elecciones automáticas de primario
   - Escalado horizontal masivo: arquitectura de Sharding (Config Servers, Query Routers, Shards)

3. [Capítulo 3: Amazon DynamoDB y el Almacenamiento Clave-Valor](03-dynamodb.md)
   - La arquitectura distribuida de Amazon DynamoDB: escalabilidad predecible a cualquier escala
   - Diseño de llaves primarias: Partition Key (PK) y Sort Key (SK) para búsquedas directas
   - Índices secundarios: Global Secondary Indexes (GSIs) e Local Secondary Indexes (LSIs)
   - Particiones calientes (Hot Partitions): causas, prevención y diseño con alta entropía
   - AWS SDK en TypeScript: operaciones atómicas, condicionales y transacciones nativas

4. [Capítulo 4: Redis y la Caché en Memoria de Alto Rendimiento](04-redis.md)
   - Redis como base de datos en memoria ultra-rápida (sub-milisegundo)
   - Estructuras de datos avanzadas: Strings, Lists, Sets, Hashes, Sorted Sets y HyperLogLogs
   - Opciones de persistencia física: RDB (snapshots rápidos) e AOF (registro incremental)
   - Mensajería reactiva en tiempo real: Pub/Sub y Redis Streams para flujos de eventos
   - Redis Cluster: particionamiento automático de datos en red y slots de hash

5. [Capítulo 5: Apache Cassandra y el Almacenamiento Wide-Column](05-cassandra.md)
   - Cassandra y la arquitectura distribuida masterless sin punto único de fallo
   - El anillo de hashing (Consistent Hashing) y el factor de replicación
   - Motores de almacenamiento LSM-Tree: escrituras rápidas y el rol de Memtables y SSTables
   - Consistencia tuneable (Tunable Consistency): cálculo matemático del quórum ($R + W > N$)
   - CQL (Cassandra Query Language) y modelado orientado estrictamente a las consultas

6. [Capítulo 6: Neo4j y las Bases de Datos Orientadas a Grafos](06-neo4j.md)
   - Cuándo usar grafos: el problema del rendimiento de las consultas JOIN relacionales recursivas
   - Elementos del grafo: Nodos, Propiedades, Relaciones direccionadas y etiquetas
   - Motor de consultas Cypher: patrones declarativos de grafos
   - Index-Free Adjacency: navegación por punteros físicos de memoria en lugar de tablas índice
   - Algoritmos clásicos de grafos: PageRank, camino más corto (Dijkstra) y detección de comunidades

7. [Capítulo 7: Modelado de Datos NoSQL Avanzado](07-modelado-avanzado.md)
   - La desnormalización estratégica: cuándo duplicar datos para priorizar lecturas rápidas
   - Patrones de esquemas en MongoDB: Referencias (1:N) frente a Documentos Embebidos
   - DynamoDB Single-Table Design: modelar múltiples entidades de negocio en una sola tabla física
   - Patrones de diseño documental: Bucket pattern, Polymorphic pattern y Extended Reference

8. [Capítulo 8: Consistencia Eventual y Transacciones Distribuidas](08-consistencia-y-transacciones.md)
   - El problema de la consistencia en sistemas distribuidos
   - Relojes de Vector y conflictos de concurrencia: resolución en servidor vs. en cliente
   - Transacciones distribuidas multi-documento en MongoDB y su costo de rendimiento
   - Orquestación de transacciones asíncronas: el patrón transaccional Saga (Coreografía vs. Orquestación)
   - El patrón transaccional Outbox para garantizar entregas "exactly-once" de eventos

9. [Capítulo 9: Indexación Avanzada y Búsqueda Full-Text](09-indexacion-y-busqueda.md)
   - Índices compuestos en MongoDB: ordenamiento de llaves y regla de igualdad-rango-orden (ESR)
   - Índices geoespaciales ($2dsphere$) para consultas de geolocalización y proximidad
   - Índices TTL para expirar datos de forma automática (sesiones de usuario, logs)
   - Integración de NoSQL con motores de búsqueda dedicados como Elasticsearch u OpenSearch

10. [Capítulo 10: Seguridad y Control de Accesos](10-seguridad.md)
    - Autenticación y control de accesos basados en roles (RBAC) en bases de datos NoSQL
    - Encriptación de datos en tránsito (TLS/SSL) y en reposo (AES-256)
    - Políticas de seguridad IAM robustas para accesos seguros a Amazon DynamoDB
    - Prevención de inyección NoSQL y sanitización de consultas dinámicas en MongoDB

11. [Capítulo 11: Monitoreo, Profiling y Optimización de Consultas](11-monitoreo-y-optimizacion.md)
    - Perfilado de consultas lentas en MongoDB mediante el comando `.explain("executionStats")`
    - Optimización de costos en AWS: monitoreo de RCU (Read Capacity Units) y WCU (Write Capacity Units)
    - Detección de memory leaks en Redis y estrategias de desalojo de llaves (LRU, LFU)
    - Análisis de latencia de red, IOPS y balanceo de carga en clusters distribuidos

12. [Capítulo 12: Proyecto Práctico Multi-NoSQL Integrador](12-proyecto-practico.md)
    - Arquitectura de Persistencia Políglota (Polyglot Persistence) para una aplicación SaaS
    - MongoDB para el catálogo flexible de productos con Aggregation Pipelines
    - Amazon DynamoDB para el registro inmutable de transacciones e historial de pedidos
    - Redis como caché de alta velocidad sub-milisegundo para tokens de sesión
    - Neo4j para el motor de recomendación inteligente de compras personalizadas

## Apéndices

- [Apéndice A: Ejercicios Prácticos Resueltos Paso a Paso](apendice-ejercicios.md)

