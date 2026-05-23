# Bases de Datos Relacionales y SQL: Diseño de Sistemas de Alto Rendimiento

Bienvenido al libro de **Bases de Datos Relacionales y SQL**. Este volumen ha sido estructurado como una guía de ingeniería avanzada "con esteroides" para llevarte desde la álgebra relacional y la física de los planificadores de consultas, hasta la optimización microscópica de índices, control de transacciones concurrentes complejas, RLS multi-tenant de nivel de producción y migraciones sin bloqueos.

---

## Índice General

1. [Capítulo 1: Fundamentos del Modelo Relacional e Internals del Motor](01-fundamentos-y-arquitectura.md)
   - Álgebra relacional y el enfoque declarativo SQL frente a la programación imperativa.
   - Anatomía interna de un motor relacional (PostgreSQL): Parser, Analyzer, Planner, Optimizer, Executor.
   - *Analogía didáctica*: *🎭 El Guionista, el Director de Teatro y el Elenco de Actores (Parser, Planner, Executor)*.
   - Implementación segura de consultas nativas con TypeScript.

2. [Capítulo 2: Anatomía de la Indexación (B-Trees, Hash y GIN)](02-anatomia-indexacion.md)
   - Estructura física en disco de los árboles balanceados B-Trees (Páginas, nodos hoja, punteros).
   - Índices Hash (búsquedas exactas) e Índices Invertidos Generalizados (GIN) para JSONB y Full-Text.
   - *Analogía didáctica*: *🔍 El Fichero de Biblioteca de Ancho Variable y el Índice Temático (B-Tree vs GIN)*.
   - Creación de índices en caliente sin bloqueos en base de datos.

3. [Capítulo 3: Transacciones y Garantías ACID Estrictas](03-transacciones-y-acid.md)
   - Desglose microscópico de las garantías transaccionales ACID (Atomicity, Consistency, Isolation, Durability).
   - Durabilidad física mediante logs secuenciales: Write-Ahead Logging (WAL) y Checkpoint threads.
   - *Analogía didáctica*: *✈️ La Caja Negra del Avión y la Bitácora de Vuelo (WAL)*.
   - Transacciones y manejo de rollbacks seguros en TypeScript.

4. [Capítulo 4: Niveles de Aislamiento y Anomalías de Concurrencia](04-niveles-aislamiento-y-mvcc.md)
   - Lecturas sucias, lecturas no repetibles, lecturas fantasma y Write Skew.
   - Multi-Version Concurrency Control (MVCC) en PostgreSQL: deconstruct de versiones en tuplas.
   - *Analogía didáctica*: *🎥 La Cámara del Tiempo de Múltiples Dimensiones (MVCC)*.

5. [Capítulo 5: Mecanismos de Bloqueos (Locking)](05-bloqueos-locking.md)
   - Bloqueos implícitos y explícitos de fila (`SELECT ... FOR UPDATE`, `FOR SHARE`) y bloqueos de tabla.
   - Escalado de bloqueos, detección y resolución automática de Deadlocks.
   - *Analogía didáctica*: *🔐 Los Candados del Hotel y los Huéspedes Hambrientos*.

6. [Capítulo 6: Consultas Avanzadas: CTEs y Window Functions](06-ctes-y-window-functions.md)
   - Common Table Expressions (CTEs) recursivas para estructuras jerárquicas y grafos.
   - Window Functions en profundidad (`ROW_NUMBER()`, `RANK()`, `LEAD()`, `LAG()`, `PARTITION BY`).
   - *Analogía didáctica*: *🪟 La Ventana Deslizante del Autobús y el Espejo Retrovisor (Window Functions)*.

7. [Capítulo 7: Modelado de Datos y Normalización Avanzada](07-modelado-y-normalizacion.md)
   - Normalización relacional (1NF, 2NF, 3NF, BCNF) frente a la desnormalización estratégica.
   - Patrón Entity-Attribute-Value (EAV) frente al tipo de datos documental JSONB en PostgreSQL.
   - *Analogía didáctica*: *🧳 El Armario Inteligente de Ropa Clasificada*.

8. [Capítulo 8: Optimización de Consultas con EXPLAIN ANALYZE](08-explain-analyze-y-joins.md)
   - Lectura e interpretación de planes de ejecución: Seq Scan, Index Scan, Index Only Scan, Bitmap Scan.
   - Algoritmos físicos de JOIN: Nested Loops, Hash Joins, Merge Joins.
   - *Analogía didáctica*: *🩺 El Escáner de Rayos X de la Fila de Peajes*.

9. [Capítulo 9: Particionamiento y Sharding de Tablas](09-particionamiento-y-sharding.md)
   - Particionamiento nativo en PostgreSQL (rango, lista, hash).
   - Escalabilidad horizontal relacional: Sharding cruzado de datos.
   - *Analogía didáctica*: *📂 El Archivador por Carpetas Mensuales y los Edificios de Sucursales*.

10. [Capítulo 10: Replicación y Alta Disponibilidad](10-replicacion-y-alta-disponibilidad.md)
    - Replicación física y lógica Maestro-Réplica de fondo síncrona/asíncrona.
    - Connection Pooling (PgBouncer) y tolerancia a fallos automáticos.
    - *Analogía didáctica*: *📠 La Copia de Fax en Tiempo Real y el Conserje del Teléfono*.

11. [Capítulo 11: Seguridad, Privilegios y SQL Injection](11-seguridad-y-rls.md)
    - Roles, esquemas, permisos `GRANT` y políticas de Row-Level Security (RLS) para aislamiento multi-tenant.
    - Prevención robusta de ataques de inyección SQL mediante parametrización a nivel de driver.
    - *Analogía didáctica*: *🛡️ Los Guardias con Llaves de Habitación y la Sanitización de Correspondencia*.

12. [Capítulo 12: Procedimientos Almacenados, Triggers y PL/pgSQL](12-stored-procedures-y-triggers.md)
    - Programación en el servidor: triggers reactivos en caliente y lógica estructurada en PL/pgSQL.
    - *Analogía didáctica*: *🤖 El Robot Automatizado de la Cocina (Triggers)*.

13. [Capítulo 13: Migraciones de Esquema Zero-Downtime](13-migraciones-zero-downtime.md)
    - Alteración de tablas en producción sin bloqueos y creación concurrente de índices (`CONCURRENTLY`).
    - *Analogía didáctica*: *🏎️ Cambiar las Llantas del Coche de F1 en Plena Carrera*.

14. [Capítulo 14: Proyecto Práctico Integrador: SaaS Multi-Tenant](14-proyecto-practico-saas.md)
    - Diseño e implementación de un backend robusto en TypeScript utilizando PostgreSQL con aislamiento RLS.
    - *Analogía didáctica*: *🏢 El Rascacielos Compartido con Departamentos Aislados*.

15. [Capítulo 15: Apéndice: Cheat Sheet y Ejercicios Resueltos Paso a Paso](15-ejercicios-y-cheat-sheet.md)
    - Guía de ejercicios desde filtros relacionales básicos hasta consultas de ventana recursivas complejas resueltas paso a paso.
