# Capítulo 20: Bases de Datos y Persistencia a Escala

> "Elige la base de datos correcta para el trabajo correcto. No todo es PostgreSQL."

## 20.1 SQL vs NoSQL — La Decisión

| | SQL (Relacional) | NoSQL |
|---|-----------------|-------|
| **Esquema** | Fijo, migraciones | Flexible, schemaless |
| **Transacciones** | ACID completo | Limitado (o eventual) |
| **Escalabilidad** | Vertical primaria, réplicas | Horizontal nativa |
| **Consultas** | SQL poderoso, joins | Limitado, desnormalizado |
| **Consistencia** | Fuerte | Eventual (configurable) |
| **Ejemplos** | PostgreSQL, MySQL | MongoDB, DynamoDB, Cassandra |

> *La regla de oro*: Empieza con SQL (PostgreSQL). Migra a NoSQL solo cuando tengas un problema concreto que SQL no resuelva bien.

## 20.2 Tipos de Bases de Datos NoSQL

### Document Stores
```
MongoDB, Couchbase, Firestore

{
  "_id": "order-123",
  "customer": { "name": "Ana", "email": "..." },
  "items": [
    { "product": "Laptop", "qty": 1, "price": 999.99 }
  ],
  "total": 999.99,
  "status": "SHIPPED"
}
```
**Ideal para**: Catálogos, perfiles de usuario, contenido semi-estructurado.

### Key-Value Stores
```
Redis, DynamoDB, etcd

user:123 → {"name": "Juan", "email": "juan@..."}
session:abc → {"userId": 123, "expires": 1705...}
```
**Ideal para**: Sesiones, caché, configuraciones, rate limiting.

### Wide-Column Stores
```
Cassandra, ScyllaDB, HBase

Tabla "eventos_usuario":
user_123 | time_1 | evento: 'click', pagina: '/home'
user_123 | time_2 | evento: 'compra', monto: 99.99
```
**Ideal para**: Time-series, analytics, IoT, logging masivo.

### Graph Databases
```
Neo4j, Amazon Neptune, Dgraph

(Ana) -[:COMPRÓ]-> (Laptop) -[:CATEGORÍA]-> (Electrónicos)
(Ana) -[:CONOCE_A]-> (Juan) -[:COMPRÓ]-> (Monitor)
```
**Ideal para**: Redes sociales, recomendaciones, detección de fraude.

### Search Engines
```
Elasticsearch, OpenSearch, Algolia

Índice "productos" analizado para full-text search, facets, aggregations.
```
**Ideal para**: Búsqueda textual, analytics en tiempo real, logs (ELK stack).

### Time-Series Databases
```
InfluxDB, TimescaleDB, Prometheus

cpu_usage{host="web-1"} 45.2 1705300000
cpu_usage{host="web-1"} 62.1 1705300005
```
**Ideal para**: Métricas, monitoreo, IoT, datos financieros.

## 20.3 Patrones de Persistencia

### Database per Service
Cada microservicio es dueño de su base de datos. Otros servicios acceden a los datos solo a través de la API del servicio dueño.

```
Servicio Pedidos  ──► PostgreSQL (pedidos)
Servicio Usuarios ──► MongoDB (usuarios)
Servicio Análisis ──► Elasticsearch (eventos)
```

Nunca accedas directamente a la BD de otro servicio.

### Polyglot Persistence
Usar diferentes tipos de base de datos para diferentes necesidades en el mismo sistema.

```
Sistema E-commerce:
├── PostgreSQL  → Transacciones (pedidos, pagos)
├── MongoDB     → Catálogo de productos (documentos flexibles)
├── Redis       → Sesiones, carrito de compras, caché
├── Elasticsearch → Búsqueda de productos
└── Cassandra   → Eventos de comportamiento de usuarios
```

### Read Replicas + Write Primary

```
          ┌──────────┐
          │ Primary  │ ◄── Escrituras
          └────┬─────┘
               │ replication
     ┌─────────┼─────────┐
     ▼         ▼         ▼
┌────────┐┌────────┐┌────────┐
│Replica ││Replica ││Replica │ ◄── Lecturas
│1       ││2       ││3       │
└────────┘└────────┘└────────┘
```

**Latencia de replicación**: Puedes leer datos "viejos" de una réplica. Si necesitas consistencia fuerte, lee del primario.

### Sharding (Horizontal Partitioning)

```
┌──────────────────┐
│ Shard 0          │  clientes A-G
│ (PostgreSQL)     │
├──────────────────┤
│ Shard 1          │  clientes H-N
│ (PostgreSQL)     │
├──────────────────┤
│ Shard 2          │  clientes O-Z
│ (PostgreSQL)     │
└──────────────────┘
```

**Desafíos**:
- Joins entre shards → imposibles. Desnormaliza o hazlo en aplicación.
- Re-sharding → complejo, requiere migración de datos.
- Hot shards → distribuir uniformemente.

## 20.4 Migraciones de Base de Datos

### Principios
- **Versionadas**: Cada cambio es un archivo de migración con número de versión.
- **Rollback**: Cada migración "up" tiene su "down".
- **Inmutables**: Una migración aplicada en producción no se modifica; se crea una nueva.
- **Automatizadas**: Se ejecutan como parte del deploy.

### Estrategias

**Expand and Contract** (para cambios sin downtime):
```
Fase 1 (Expand): Añadir nueva columna, tabla o formato.
                 El código escribe en ambos esquemas.
                 
Fase 2 (Migrate): Migrar datos existentes al nuevo esquema.

Fase 3 (Contract): El código solo lee/escribe el nuevo esquema.
                   Eliminar el antiguo.
```

**Ejemplo**: Renombrar columna `created_at` → `created_timestamp`
1. Añadir `created_timestamp`, escribir en ambas.
2. Migrar datos históricos.
3. Cambiar código a solo `created_timestamp`.
4. Eliminar `created_at`.

## 20.5 Indexes que Importan

- **B-Tree**: Predeterminado. Igualdad y rangos (`WHERE id = 5`, `WHERE date > ...`).
- **Hash**: Solo igualdad (`WHERE id = 5`), más rápido para eso.
- **GIN (PostgreSQL)**: Full-text search, arrays, JSONB.
- **GiST (PostgreSQL)**: Datos geoespaciales, búsqueda geométrica.
- **Partial Index**: `WHERE active = true` — indexas solo lo consultado.
- **Composite Index**: `(columna_a, columna_b)` — el orden importa.

**Regla**: Indexa columnas en WHERE, JOIN y ORDER BY. No indexes todo: cada índice ralentiza escrituras.

---

> **Reflexión del capítulo**: La base de datos suele ser el cuello de botella más difícil de resolver en un sistema. No porque la tecnología sea mala, sino porque los datos son el activo más valioso y moverlos es costoso. Dedica tiempo a elegir y modelar tu persistencia. Es la decisión más difícil de revertir.
