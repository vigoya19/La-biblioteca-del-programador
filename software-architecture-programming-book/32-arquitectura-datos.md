# Capítulo 32: Arquitectura de Datos — Pipelines, Lagos y Data Mesh

> "Los datos son el activo más valioso de una empresa moderna. Su arquitectura merece tanta atención como la del código."

## 32.1 Más Allá de la Base de Datos Operacional

La arquitectura de datos va mucho más allá de PostgreSQL o MongoDB. Incluye:

- **OLTP** (Online Transaction Processing): Bases de datos operacionales — lo que corre el negocio.
- **OLAP** (Online Analytical Processing): Data warehouses, data lakes — lo que analiza el negocio.
- **Pipelines**: Cómo mueves y transformas datos entre sistemas.
- **Governance**: Catálogos, linaje, calidad, privacidad.
- **Serving**: Cómo expones datos para consumo (APIs, dashboards, ML).

## 32.2 OLTP vs OLAP

| | OLTP | OLAP |
|---|------|------|
| **Propósito** | Operar el negocio | Analizar el negocio |
| **Operaciones** | Muchas lecturas/escrituras pequeñas | Pocas queries muy complejas |
| **Datos** | Actualizados, normalizados | Históricos, desnormalizados |
| **Volumen por query** | KBs | GBs-TBs |
| **Latencia** | Milisegundos | Segundos-minutos |
| **Usuarios** | Miles (app) | Decenas (analistas) |
| **Ejemplo** | "Crear pedido #123" | "Ventas por categoría Q3 2023" |
| **Tecnología** | PostgreSQL, MySQL, DynamoDB | BigQuery, Snowflake, Redshift |

**Regla de oro**: Nunca hagas analytics sobre tu BD operacional. Los queries analíticos compiten por recursos con tus usuarios y degradan la experiencia.

## 32.3 El Viaje de los Datos — Pipeline Típico

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                    │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────────────┐  │
│  │ Fuentes  │────►│   Ingesta    │────►│   Transformación     │  │
│  │          │     │              │     │                      │  │
│  │ • BD     │     │ • Kafka      │     │ • dbt (data build    │  │
│  │ • APIs   │     │ • Debezium   │     │   tool)              │  │
│  │ • Logs   │     │ • Fivetran   │     │ • Spark              │  │
│  │ • Files  │     │ • Airbyte    │     │ • SQL (ELT)          │  │
│  └──────────┘     └──────────────┘     └──────────┬───────────┘  │
│                                                    │              │
│                                                    ▼              │
│                               ┌──────────────────────────────────┤
│                               │          Almacenamiento          │
│                               │                                  │
│                               │ ┌────────────┐ ┌──────────────┐ │
│                               │ │ Data Lake  │ │ Data Warehouse│ │
│                               │ │ (S3/GCS)   │ │ (BigQuery/   │ │
│                               │ │ crudo      │ │  Snowflake)  │ │
│                               │ └────────────┘ └──────────────┘ │
│                               └──────────┬───────────────────────┘
│                                          │
│                                          ▼
│                               ┌──────────────────────┐
│                               │       Consumo        │
│                               │                      │
│                               │ • Dashboards (Looker,│
│                               │   Metabase, Superset)│
│                               │ • APIs (para productos│
│                               │   y servicios)       │
│                               │ • ML Models          │
│                               │ • Reverse ETL        │
│                               └──────────────────────┘
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

## 32.4 ETL vs ELT

```
ETL (Extract → Transform → Load):
  Tradicional, para data warehouses on-premise.
  Transformas ANTES de cargar. El warehouse solo almacena.

ELT (Extract → Load → Transform):
  Moderno, cloud-native.
  Cargas datos crudos PRIMERO, transformas DESPUÉS en el warehouse.
  Ventaja: puedes reprocesar, cambiar transformaciones sin re-ingerir.

Ejemplo con dbt (ELT):
┌──────────┐     ┌──────────┐     ┌───────────────────┐
│PostgreSQL│────►│ BigQuery │────►│ dbt (transforma    │
│ (origen) │     │ (staging)│     │  SQL → modelos)    │
└──────────┘     └──────────┘     └───────────────────┘
```

### dbt (Data Build Tool) — El Estándar para Transformaciones

```sql
-- models/marts/orders_daily.sql
-- dbt model: transforma datos staging en modelo de negocio

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),
order_items AS (
    SELECT * FROM {{ ref('stg_order_items') }}
)

SELECT
    DATE_TRUNC('day', o.created_at) AS order_date,
    o.status,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    AVG(oi.quantity * oi.unit_price) AS avg_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY 1, 2
```

```yaml
# dbt lineage: dbt genera automáticamente el grafo de dependencias
# sources → staging → intermediate → marts → exposure
```

## 32.5 Data Lake, Data Warehouse, Lakehouse

### Data Lake
Almacenamiento masivo de datos en formato crudo (S3, GCS, ADLS).

```
Ventajas:
  + Barato (object storage).
  + Schema-on-read (flexible).
  + Guardas TODO (nunca sabes qué vas a necesitar).

Desventajas:
  - Puede convertirse en "data swamp" sin governance.
  - Rendimiento de queries inferior al warehouse.
  - Necesitas catálogo (Hive Metastore, Glue, Unity Catalog).
```

**Tecnologías**: Apache Iceberg, Delta Lake, Apache Hudi sobre S3.

### Data Warehouse
Datos estructurados, optimizados para queries analíticas.

```
Ventajas:
  + Rendimiento de queries excelente.
  + SQL estándar.
  + Gobernanza y acceso integrado.

Desventajas:
  - Más caro por GB.
  - Schema-on-write (menos flexible).
  - Típicamente no guardas datos crudos.
```

### Data Lakehouse
Combina lo mejor de ambos mundos: storage barato + rendimiento de warehouse + ACID.

```
Delta Lake / Iceberg sobre S3:
  - Datos en formato Parquet/ORC.
  - Transacciones ACID en data lake.
  - Time travel (leer versiones anteriores de los datos).
  - Schema evolution.

Arquitectura Lakehouse:
┌──────────────────────────────────────────┐
│              Databricks / Spark          │
│  ┌──────────────────────────────────────┤
│  │  Delta Lake (tablas con ACID)        │
│  │  sobre S3 (storage barato)           │
│  └──────────────────────────────────────┤
│  Gobernanza: Unity Catalog               │
│  Queries: Photon Engine, Spark SQL       │
│  ML: MLflow integrado                    │
└──────────────────────────────────────────┘
```

## 32.6 Data Mesh — La Revolución Organizacional

Propuesto por Zhamak Dehghani (ThoughtWorks, 2019). Cambia el paradigma: los datos no son un monolito centralizado.

### Principios del Data Mesh

```
1. Domain Ownership:
   Cada equipo es dueño de SUS datos, como es dueño de SU código.
   El equipo de pedidos publica datos de pedidos.

2. Data as a Product:
   Los datos se tratan como producto: con SLAs, documentación,
   versionado, consumidores identificados.

3. Self-Serve Data Platform:
   Plataforma que permite a cualquier equipo publicar, descubrir
   y consumir datos sin depender de un equipo central.

4. Federated Computational Governance:
   Estándares de interoperabilidad globales, ejecución local.
```

```
Data Mesh vs Data Warehouse Centralizado:

Centralizado:                           Data Mesh:
┌────────────────┐                  ┌────┐ ┌────┐ ┌────┐
│ Data Warehouse │                  │Ped │ │Usr │ │Inv │
│     Team       │                  │DP  │ │DP  │ │DP  │
└───────┬────────┘                  └──┬──┘ └──┬──┘ └──┬──┘
        │                              │       │       │
  ┌─────┴─────┐                   ┌────┴───────┴───────┴────┐
  ▼     ▼     ▼                   │  Data Platform (self-serve)│
Equipo Equipo Equipo              └────────────────────────────┘
(esperan datos del DW team)

"Data Product" (DP) = Datasets publicados por cada equipo con:
  - Schema definido y versionado.
  - SLAs (frescura, disponibilidad, calidad).
  - Documentación (significado de cada campo).
  - Linaje (de dónde vienen los datos).
```

### Cuándo Adoptar Data Mesh

- **Organización grande** (múltiples dominios de negocio independientes).
- **Cuello de botella** del equipo central de datos (semanas para un dataset nuevo).
- **Madurez de ingeniería** alta (equipos pueden gestionar sus propios pipelines).
- **NO adoptar** si eres una startup de 20 personas. Un warehouse centralizado funciona bien.

## 32.7 Cambio de Datos en Tiempo Real (CDC)

```
Change Data Capture (CDC):

PostgreSQL ──► Debezium ──► Kafka ──► Consumidores
   │               │                      │
   │ WAL           │ Lee el log           │
   │ (Write-Ahead  │ de transacciones     ├──► Elasticsearch (búsqueda)
   │  Log)         │                      ├──► Data Warehouse (analytics)
   │               │                      ├──► Cache invalidation
   │               │                      └──► Otros servicios
```

**Herramientas**: Debezium, AWS DMS, Fivetran, Airbyte.

### Evento CDC de Ejemplo

```json
{
  "before": null,
  "after": {
    "id": "order-123",
    "status": "CONFIRMED",
    "total": 150.00,
    "updated_at": "2024-01-15T10:30:00Z"
  },
  "source": {
    "db": "shopflow",
    "table": "orders",
    "lsn": 123456789
  },
  "op": "u",
  "ts_ms": 1705312200000
}
```

## 32.8 Gobernanza y Calidad de Datos

### Data Catalog
El "índice" de todos los datasets de la empresa.

```
Herramientas:
  - Amundsen (Lyft, open source)
  - DataHub (LinkedIn, open source)
  - Alation, Collibra (enterprise)
  - AWS Glue Catalog

Información por dataset:
  - Schema + descripción de cada columna.
  - Owner (equipo responsable).
  - Frecuencia de actualización.
  - SLA (frescura, calidad).
  - Linaje (de dónde viene, quién lo consume).
```

### Data Quality Dimensions

| Dimensión | Pregunta | Ejemplo de Test |
|-----------|----------|----------------|
| **Completitud** | ¿Faltan datos? | `orders` sin `customer_id`: no deben existir |
| **Unicidad** | ¿Hay duplicados? | `order_id` debe ser único |
| **Frescura** | ¿Qué tan actualizados están? | Datos no deben tener >1h de retraso |
| **Consistencia** | ¿Es consistente entre fuentes? | `total` = SUM(`order_lines.price`) |
| **Precisión** | ¿Refleja la realidad? | `status` solo valores del enum |
| **Validez** | ¿Formato correcto? | `email` cumple regex, `phone` es E.164 |

```yaml
# Great Expectations: tests de calidad como código
expectations:
  - expectation: expect_column_values_to_not_be_null
    column: order_id
  - expectation: expect_column_values_to_be_unique
    column: order_id
  - expectation: expect_column_values_to_be_in_set
    column: status
    value_set: ["PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"]
  - expectation: expect_column_mean_to_be_between
    column: total
    min_value: 0
    max_value: 100000
```

## 32.9 Catálogo de Tecnologías de Datos (2024)

```
Ingesta / CDC:
├── Debezium (CDC open source)
├── Fivetran / Airbyte / Stitch (conectores SaaS)
├── Kafka Connect (framework de conectores)
└── AWS DMS / GCP Datastream

Transformación:
├── dbt (SQL, el estándar)
├── Apache Spark (big data, código)
└── SQL en el warehouse (BigQuery, Snowflake)

Orquestación:
├── Airflow (Python DAGs, el más usado)
├── Dagster (asset-based, moderno)
├── Prefect (Python, mejor DX)
└── Temporal (workflows complejos)

Almacenamiento:
├── S3 / GCS / ADLS (data lake)
├── Snowflake / BigQuery / Redshift (warehouse)
├── Databricks (lakehouse)
└── Delta Lake / Iceberg (formatos abiertos)

Catálogo y Gobernanza:
├── DataHub / Amundsen (catálogo open source)
├── Great Expectations (calidad)
├── dbt (documentación + linaje)
└── Monte Carlo / Soda (calidad + observabilidad)
```

## 32.10 El Rol del Arquitecto en Datos

- **No necesitas ser Data Engineer**, pero sí entender el panorama.
- **Asegura que los datos operacionales estén disponibles** para analytics.
- **Diseña eventos de dominio** que alimenten los pipelines de datos.
- **Define la estrategia CDC**: qué tablas, qué frecuencia, qué destino.
- **Exige catálogo de datos**: si un dataset no está documentado, no existe.
- **Garantiza privacidad y compliance**: PII, GDPR, right to deletion en todos los sistemas.

---

> **Reflexión del capítulo**: La arquitectura de software y la arquitectura de datos son dos caras de la misma moneda. Tu sistema produce datos que alguien necesita analizar. Si no diseñas para eso desde el principio, terminarás con un ETL frágil que extrae datos de réplicas de producción a las 3 AM y falla todos los lunes. Piensa en datos desde el día 1.

---

← [Capítulo anterior](31-testing.md) | [Inicio](README.md) | [Capítulo siguiente →](33-gobernanza.md)
