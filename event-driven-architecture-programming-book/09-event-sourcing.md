# Capítulo 9: Event Sourcing

Event Sourcing es el patrón más poderoso —y más incomprendido— de la arquitectura orientada a eventos. En lugar de almacenar el estado actual de una entidad, almacenamos **la secuencia completa de eventos que la modificaron**. El estado actual se reconstruye reproduciendo esos eventos.

> "No guardes el estado. Guarda lo que pasó. El estado es un efecto secundario." — Greg Young

---

## 9.1 El Patrón: Almacenar Eventos, No Estado

### Enfoque tradicional (CRUD)

```sql
-- Tabla de órdenes: almacena SOLO el estado actual
CREATE TABLE ordenes (
  id UUID PRIMARY KEY,
  cliente_id UUID NOT NULL,
  estado VARCHAR(20) NOT NULL, -- 'creada', 'pagada', 'enviada'
  total DECIMAL(10,2),
  updated_at TIMESTAMP
);

-- Cuando el cliente paga, el registro se SOBRESCRIBE
UPDATE ordenes SET estado = 'pagada', total = 159.98, updated_at = NOW() WHERE id = '123';

-- El historial de cómo llegó a ese estado... se pierde.
-- ¿Estaba en 'creada' antes? ¿Quién cambió el estado? ¿Cuándo exactamente? Imposible saber.
```

### Event Sourcing

```sql
-- Tabla de eventos: almacena CADA cambio como un evento inmutable
CREATE TABLE eventos_ordenes (
  id BIGSERIAL PRIMARY KEY,
  aggregate_id UUID NOT NULL,
  aggregate_type VARCHAR(50) NOT NULL,
  event_type VARCHAR(100) NOT NULL,   -- 'orden.creada', 'orden.pagada', 'orden.enviada'
  event_data JSONB NOT NULL,           -- Payload completo del evento
  metadata JSONB NOT NULL,             -- eventId, timestamp, version, userId
  version INT NOT NULL,                -- Número de versión del aggregate
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX ON eventos_ordenes (aggregate_id, version);

-- Cada evento es un INSERT, nunca un UPDATE ni DELETE
INSERT INTO eventos_ordenes (aggregate_id, aggregate_type, event_type, event_data, metadata, version)
VALUES
  ('123', 'Orden', 'orden.creada', '{"ordenId":"123","clienteId":"456","items":[...],"total":159.98}', '{"eventId":"evt-1","timestamp":"..."}', 1),
  ('123', 'Orden', 'orden.pagada', '{"ordenId":"123","metodo":"tarjeta","monto":159.98}', '{"eventId":"evt-2","timestamp":"..."}', 2),
  ('123', 'Orden', 'orden.enviada', '{"ordenId":"123","tracking":"1Z999"}', '{"eventId":"evt-3","timestamp":"..."}', 3);

-- El estado actual se RECONSTRUYE aplicando eventos en orden:
-- Estado final = aplicar(evt1, aplicar(evt2, aplicar(evt3, estado_inicial)))
```

---

## 9.2 Event Store: Diseño e Implementación

```typescript
// ─── Interfaz del Event Store ───
interface EventStore {
  guardarEventos(
    aggregateType: string,
    aggregateId: string,
    eventos: DomainEvent[],
    expectedVersion: number, // Control de concurrencia optimista
  ): Promise<void>;

  obtenerEventos(
    aggregateType: string,
    aggregateId: string,
    desdeVersion?: number,
  ): Promise<DomainEvent[]>;

  // Para proyecciones: leer todos los eventos de un tipo
  obtenerEventosPorTipo(
    eventType: string,
    desdeTimestamp?: Date,
  ): AsyncIterable<DomainEvent>;
}

// ─── Implementación con PostgreSQL ───
class PostgresEventStore implements EventStore {
  constructor(private db: Pool) {}

  async guardarEventos(
    aggregateType: string,
    aggregateId: string,
    eventos: DomainEvent[],
    expectedVersion: number,
  ): Promise<void> {
    const client = await this.db.connect();

    try {
      await client.query("BEGIN");

      // ─── Control de concurrencia optimista ───
      const { rows } = await client.query(
        `SELECT MAX(version) as version_actual
         FROM eventos
         WHERE aggregate_id = $1 AND aggregate_type = $2`,
        [aggregateId, aggregateType],
      );

      const versionActual = rows[0]?.version_actual ?? 0;

      if (versionActual !== expectedVersion) {
        throw new ConcurrencyError(
          `Conflicto de concurrencia en ${aggregateType}:${aggregateId}. ` +
          `Esperada v${expectedVersion}, actual v${versionActual}`
        );
      }

      // ─── Insertar eventos en lote ───
      for (let i = 0; i < eventos.length; i++) {
        const evento = eventos[i];
        const version = expectedVersion + i + 1;

        await client.query(
          `INSERT INTO eventos (aggregate_id, aggregate_type, event_type, event_data, metadata, version)
           VALUES ($1, $2, $3, $4, $5, $6)`,
          [
            aggregateId,
            aggregateType,
            evento.type,
            JSON.stringify(evento.data),
            JSON.stringify({
              eventId: evento.metadata.eventId,
              timestamp: evento.metadata.timestamp,
              userId: evento.metadata.userId,
            }),
            version,
          ],
        );
      }

      await client.query("COMMIT");
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }

  async obtenerEventos(
    aggregateType: string,
    aggregateId: string,
    desdeVersion: number = 1,
  ): Promise<DomainEvent[]> {
    const { rows } = await this.db.query(
      `SELECT event_type, event_data, metadata, version
       FROM eventos
       WHERE aggregate_id = $1 AND aggregate_type = $2 AND version >= $3
       ORDER BY version ASC`,
      [aggregateId, aggregateType, desdeVersion],
    );

    return rows.map(row => ({
      type: row.event_type,
      data: row.event_data,
      metadata: { ...row.metadata, aggregateType, aggregateId, version: row.version },
    }));
  }

  async *obtenerEventosPorTipo(
    eventType: string,
    desdeTimestamp?: Date,
  ): AsyncIterable<DomainEvent> {
    let cursor: number | null = null;
    const BATCH_SIZE = 1000;

    while (true) {
      const { rows } = await this.db.query(
        `SELECT id, aggregate_id, aggregate_type, event_type, event_data, metadata, version
         FROM eventos
         WHERE event_type = $1
           AND ($2::BIGINT IS NULL OR id > $2)
           AND ($3::TIMESTAMP IS NULL OR created_at > $3)
         ORDER BY id ASC
         LIMIT $4`,
        [eventType, cursor, desdeTimestamp, BATCH_SIZE],
      );

      if (rows.length === 0) break;

      for (const row of rows) {
        yield {
          type: row.event_type,
          data: row.event_data,
          metadata: { ...row.metadata, aggregateType: row.aggregate_type, aggregateId: row.aggregate_id, version: row.version },
        };
        cursor = parseInt(row.id);
      }
    }
  }
}
```

---

## 9.3 Reconstrucción de Estado

El estado actual de un aggregate se reconstruye aplicando todos sus eventos en orden:

```typescript
// ─── Repository que usa Event Sourcing ───
class OrdenEventSourcingRepository {
  constructor(private eventStore: EventStore) {}

  async obtener(ordenId: string): Promise<Orden> {
    const eventos = await this.eventStore.obtenerEventos("Orden", ordenId);

    if (eventos.length === 0) {
      throw new NotFoundError(`Orden ${ordenId} no encontrada`);
    }

    // Reconstruir la orden desde eventos
    return Orden.reconstruirDesde(eventos);
  }

  async guardar(orden: Orden): Promise<void> {
    const eventos = orden.drenarEventos();
    if (eventos.length === 0) return;

    await this.eventStore.guardarEventos(
      "Orden",
      orden.id,
      eventos,
      orden.versionActual, // Concurrencia optimista
    );
  }
}

// ─── Aggregate con reconstrucción ───
class Orden {
  private id: string;
  private estado: OrdenEstado = "inicial";
  private clienteId: string = "";
  private items: ItemOrden[] = [];
  private total: number = 0;
  private versionActual: number = 0;
  private eventosPendientes: DomainEvent[] = [];

  // ─── Reconstruir desde eventos ───
  static reconstruirDesde(eventos: DomainEvent[]): Orden {
    const orden = new Orden();

    for (const evento of eventos) {
      orden.aplicarEvento(evento);
    }

    return orden;
  }

  // ─── Método "when": aplica un evento al estado ───
  private aplicarEvento(evento: DomainEvent): void {
    this.versionActual = evento.metadata.version;

    switch (evento.type) {
      case "orden.creada":
        this.id = evento.data.ordenId;
        this.clienteId = evento.data.clienteId;
        this.items = evento.data.items;
        this.total = evento.data.total;
        this.estado = "creada";
        break;

      case "orden.pagada":
        this.estado = "pagada";
        break;

      case "orden.enviada":
        this.estado = "enviada";
        break;

      case "orden.cancelada":
        this.estado = "cancelada";
        break;

      case "items.modificados":
        this.items = evento.data.items;
        this.total = evento.data.nuevoTotal;
        break;
    }
  }

  // ─── Métodos de comando: validan y emiten eventos ───
  pagar(metodo: string, transaccionId: string, monto: number): void {
    if (this.estado !== "creada") {
      throw new InvalidStateError(`No se puede pagar orden en estado ${this.estado}`);
    }

    // Emitir evento (no modifica estado directamente)
    this.emitirEvento({
      type: "orden.pagada",
      data: { ordenId: this.id, metodo, transaccionId, monto },
    });

    // Aplicar evento al estado local
    this.aplicarEvento(this.eventosPendientes[this.eventosPendientes.length - 1]);
  }

  private emitirEvento(evento: DomainEvent): void {
    this.eventosPendientes.push({
      ...evento,
      metadata: {
        eventId: crypto.randomUUID(),
        timestamp: new Date().toISOString(),
        aggregateType: "Orden",
        aggregateId: this.id,
        version: this.versionActual + 1,
      },
    });
  }
}
```

---

## 9.4 Snapshots (Rendimiento)

Para aggregates con muchos eventos, reconstruir desde el inicio es ineficiente. Los snapshots guardan el estado en un punto y solo se aplican eventos posteriores:

```typescript
// ─── Snapshot Store ───
interface SnapshotStore {
  guardarSnapshot(aggregateType: string, aggregateId: string, version: number, estado: unknown): Promise<void>;
  obtenerSnapshot(aggregateType: string, aggregateId: string): Promise<{ version: number; estado: unknown } | null>;
}

class PostgresSnapshotStore implements SnapshotStore {
  async guardarSnapshot(aggregateType: string, aggregateId: string, version: number, estado: unknown): Promise<void> {
    await this.db.query(
      `INSERT INTO snapshots (aggregate_type, aggregate_id, version, estado)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (aggregate_type, aggregate_id)
       DO UPDATE SET version = $3, estado = $4, created_at = NOW()`,
      [aggregateType, aggregateId, version, JSON.stringify(estado)],
    );
  }

  async obtenerSnapshot(aggregateType: string, aggregateId: string): Promise<{ version: number; estado: unknown } | null> {
    const { rows } = await this.db.query(
      `SELECT version, estado FROM snapshots
       WHERE aggregate_type = $1 AND aggregate_id = $2`,
      [aggregateType, aggregateId],
    );
    if (rows.length === 0) return null;
    return { version: rows[0].version, estado: rows[0].estado };
  }
}

// ─── Repository con snapshots ───
class OrdenRepositoryConSnapshots {
  constructor(
    private eventStore: EventStore,
    private snapshotStore: SnapshotStore,
  ) {}

  async obtener(ordenId: string): Promise<Orden> {
    // 1. Intentar cargar snapshot
    const snapshot = await this.snapshotStore.obtenerSnapshot("Orden", ordenId);

    let orden: Orden;
    let desdeVersion: number;

    if (snapshot) {
      // Reconstruir desde snapshot
      orden = Orden.desdeSnapshot(snapshot.estado as OrdenSnapshot);
      desdeVersion = snapshot.version + 1;
    } else {
      orden = new Orden();
      desdeVersion = 1;
    }

    // 2. Aplicar eventos posteriores al snapshot
    const eventos = await this.eventStore.obtenerEventos("Orden", ordenId, desdeVersion);
    for (const evento of eventos) {
      orden.aplicarEvento(evento);
    }

    return orden;
  }

  async guardar(orden: Orden): Promise<void> {
    const eventos = orden.drenarEventos();
    if (eventos.length === 0) return;

    await this.eventStore.guardarEventos("Orden", orden.id, eventos, orden.versionActual);

    // Crear snapshot cada N versiones
    if (orden.versionActual % 10 === 0) {
      await this.snapshotStore.guardarSnapshot(
        "Orden",
        orden.id,
        orden.versionActual,
        orden.toSnapshot(),
      );
    }
  }
}
```

---

## 9.5 Proyecciones (Modelo de Lectura)

Las proyecciones transforman eventos en modelos optimizados para consultas:

```typescript
// ─── Proyección: Órdenes Activas ───
// Escucha eventos y mantiene una tabla desnormalizada para consultas rápidas

class ProyeccionOrdenesActivas {
  async onOrdenCreada(evento: OrdenCreada): Promise<void> {
    await this.db.query(
      `INSERT INTO ordenes_activas (orden_id, cliente_id, estado, total, items_count, creada_en)
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [
        evento.data.ordenId,
        evento.data.clienteId,
        "creada",
        evento.data.total,
        evento.data.items.length,
        evento.metadata.timestamp,
      ],
    );
  }

  async onOrdenPagada(evento: OrdenPagada): Promise<void> {
    await this.db.query(
      `UPDATE ordenes_activas
       SET estado = 'pagada',
           metodo_pago = $2,
           pagada_en = $3
       WHERE orden_id = $1`,
      [evento.data.ordenId, evento.data.metodo, evento.metadata.timestamp],
    );
  }

  async onOrdenEnviada(evento: OrdenEnviada): Promise<void> {
    await this.db.query(
      `UPDATE ordenes_activas
       SET estado = 'enviada',
           tracking_number = $2,
           enviada_en = $3
       WHERE orden_id = $1`,
      [evento.data.ordenId, evento.data.trackingNumber, evento.metadata.timestamp],
    );
  }

  async onOrdenCancelada(evento: OrdenCancelada): Promise<void> {
    await this.db.query(
      `UPDATE ordenes_activas
       SET estado = 'cancelada', cancelada_en = $2
       WHERE orden_id = $1`,
      [evento.data.ordenId, evento.metadata.timestamp],
    );
  }
}

// ─── Suscribir proyección a eventos ───
const proyeccion = new ProyeccionOrdenesActivas(db);

eventBus.on("orden.creada", proyeccion.onOrdenCreada.bind(proyeccion));
eventBus.on("orden.pagada", proyeccion.onOrdenPagada.bind(proyeccion));
eventBus.on("orden.enviada", proyeccion.onOrdenEnviada.bind(proyeccion));
eventBus.on("orden.cancelada", proyeccion.onOrdenCancelada.bind(proyeccion));

// Ahora las consultas son rápidas:
// SELECT * FROM ordenes_activas WHERE cliente_id = '456' AND estado = 'pagada';
```

### Reconstrucción de proyecciones

```typescript
// Si la proyección se corrompe o necesita un nuevo campo,
// se reconstruye desde cero reproduciendo todos los eventos

class ReconstructorProyecciones {
  async reconstruirProyeccionOrdenes(): Promise<void> {
    // Vaciar tabla de proyección
    await this.db.query("TRUNCATE ordenes_activas");

    const proyeccion = new ProyeccionOrdenesActivas(this.db);

    // Leer todos los eventos de órdenes en orden
    for await (const evento of this.eventStore.obtenerEventosPorTipo("orden.creada")) {
      await proyeccion.onOrdenCreada(evento);
    }
    for await (const evento of this.eventStore.obtenerEventosPorTipo("orden.pagada")) {
      await proyeccion.onOrdenPagada(evento);
    }
    for await (const evento of this.eventStore.obtenerEventosPorTipo("orden.enviada")) {
      await proyeccion.onOrdenEnviada(evento);
    }

    logger.info("Proyección de órdenes reconstruida exitosamente");
  }
}
```

---

## 9.6 Ventajas y Desventajas

### Ventajas

```
✅ Auditoría completa: cada cambio está registrado
  → "¿Quién cambió el estado de la orden #123 a 'cancelada' y cuándo?"

✅ Temporal queries: consultar el estado en cualquier momento del pasado
  → "¿Cuál era el total acumulado de ventas el 15 de enero a las 14:30?"

✅ Debugging: reconstruir exactamente lo que pasó
  → Reproducir bugs en desarrollo con la secuencia real de eventos

✅ Extensibilidad: nuevas proyecciones sin modificar código existente
  → Añadir "dashboard de velocidad de envío" leyendo eventos antiguos

✅ Origen único de verdad: los eventos son inmutables e irrebatibles
  → Nunca hay conflicto sobre "cuál es el estado correcto"
```

### Desventajas

```
❌ Complejidad: no es CRUD. Requiere cambio de mentalidad.
  → Curva de aprendizaje alta para el equipo

❌ Eventual Consistency: las proyecciones van con retraso
  → La UI puede mostrar datos desactualizados por milisegundos

❌ Versionado de eventos: los schemas evolucionan
  → Necesitas Schema Registry + estrategias de migración (Capítulo 14)

❌ Almacenamiento: más datos que CRUD
  → Compensado con snapshots y retención selectiva

❌ Eliminación de datos: GDPR "derecho al olvido"
  → Crypto-shredding, eventos de olvido, borrado de snapshots
```

---

## 9.7 Cuándo Usar Event Sourcing

| ✅ Usar Event Sourcing | ❌ No usar Event Sourcing |
|------------------------|--------------------------|
| Auditoría es requisito legal/financiero | CRUD simple sin necesidad de historial |
| Necesitas trazabilidad completa | Los datos cambian raramente |
| El negocio requiere temporal queries | El equipo no está preparado |
| Múltiples modelos de lectura (CQRS) | La consistencia eventual es inaceptable |
| Debugging de bugs complejos en producción | El volumen de eventos es bajo (<1000/día) |
| Fintech, e-commerce, healthcare, supply chain | Blog, landing page, CMS |

---

## Resumen del Capítulo

- **Event Sourcing** almacena la secuencia de eventos que modifican una entidad, no su estado actual.
- El **Event Store** es una base de datos append-only donde cada evento es inmutable.
- Los **aggregates** se reconstruyen aplicando eventos en orden. Los **snapshots** optimizan el rendimiento.
- Las **proyecciones** transforman eventos en modelos de lectura optimizados para consultas.
- El **control de concurrencia optimista** (expected version) evita conflictos en escrituras concurrentes.
- Ventajas: auditoría completa, temporal queries, debugging preciso, extensibilidad.
- Desventajas: complejidad, eventual consistency, versionado de schemas, volumen de almacenamiento.

En el siguiente capítulo exploramos CQRS: separar lecturas de escrituras.
