# Capítulo 10: CQRS (Command Query Responsibility Segregation)

CQRS es la separación de responsabilidades entre **comandos** (escritura, modifican estado) y **consultas** (lectura, devuelven datos). Aunque puede usarse de forma independiente, con Event Sourcing alcanza su máximo potencial.

> "La razón por la que CQRS existe es porque el modelo que necesitas para escribir no es el modelo que necesitas para leer." — Greg Young

---

## 10.1 Separar Lecturas de Escrituras

```
┌──────────────────────────────────────────────────────────────┐
│                    ARQUITECTURA TRADICIONAL                   │
│                                                              │
│  Cliente ──▶ [POST /ordenes]    ──▶ Mismo modelo ──▶ BD     │
│  Cliente ──▶ [GET /ordenes/123] ──▶ Mismo modelo ──▶ BD     │
│                                                              │
│  Problema: la misma tabla "ordenes" sirve para TODO.         │
│  - Índices para lecturas penalizan escrituras.                │
│  - JOINs complejos para reportes ralentizan todo.             │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    ARQUITECTURA CQRS                          │
│                                                              │
│  Comandos ──▶ [Modelo Escritura] ──▶ Event Store             │
│                                           │                  │
│                                     Proyecciones             │
│                                           │                  │
│  Consultas ◀── [Modelo Lectura]   ◀── BD Lectura            │
│                                                              │
│  Cada lado usa el modelo óptimo para su propósito.           │
└──────────────────────────────────────────────────────────────┘
```

---

## 10.2 El Modelo de Escritura (Command Side)

El lado de escritura usa el **modelo de dominio**: aggregates, domain events, invariantes de negocio.

```typescript
// ─── Command: CrearOrden ───
interface CrearOrdenComando {
  type: "crear_orden";
  commandId: string;     // Idempotencia
  data: {
    ordenId: string;
    clienteId: string;
    items: { productoId: string; cantidad: number; precioUnitario: number }[];
    direccionEnvio: Direccion;
  };
}

// ─── Command Handler ───
class CrearOrdenCommandHandler {
  constructor(
    private repositorio: OrdenEventSourcingRepository,
    private eventBus: EventBus,
  ) {}

  async handle(comando: CrearOrdenComando): Promise<void> {
    // Validar que no exista ya (idempotencia por commandId)
    const existente = await this.repositorio.obtener(comando.data.ordenId).catch(() => null);
    if (existente) {
      logger.warn("Orden ya existe, ignorando comando duplicado", { commandId: comando.commandId });
      return;
    }

    // Crear aggregate (valida reglas de negocio)
    const orden = Orden.crear(
      comando.data.ordenId,
      comando.data.clienteId,
      comando.data.items,
    );

    // Persistir eventos
    await this.repositorio.guardar(orden);

    // Publicar eventos (dispara proyecciones y otros servicios)
    for (const evento of orden.drenarEventos()) {
      await this.eventBus.publicar(evento);
    }

    logger.info("Orden creada", { ordenId: comando.data.ordenId, commandId: comando.commandId });
  }
}

// ─── API de comandos ───
app.post("/api/ordenes", async (req, res) => {
  const commandId = crypto.randomUUID();
  const comando: CrearOrdenComando = {
    type: "crear_orden",
    commandId,
    data: { ...req.body, ordenId: crypto.randomUUID() },
  };

  try {
    await commandHandler.handle(comando);
    res.status(202).json({
      commandId,
      ordenId: comando.data.ordenId,
      status: "accepted",
      _links: {
        consultar: `/api/consultas/ordenes/${comando.data.ordenId}`,
      },
    });
  } catch (error) {
    if (error instanceof ValidationError) {
      res.status(400).json({ error: error.message });
    } else {
      throw error;
    }
  }
});
```

---

## 10.3 El Modelo de Lectura (Query Side)

El lado de lectura usa **modelos desnormalizados** optimizados para cada tipo de consulta:

```typescript
// ─── Proyecciones: mantienen actualizado el modelo de lectura ───

// Tabla desnormalizada para consultas de órdenes
class ProyeccionOrdenesConsulta {
  async onOrdenCreada(evento: OrdenCreada): Promise<void> {
    await this.db.query(
      `INSERT INTO ordenes_consulta
         (orden_id, cliente_id, cliente_nombre, estado, total, items, creada_en, actualizada_en)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8)`,
      [
        evento.data.ordenId,
        evento.data.clienteId,
        evento.data.clienteNombre,   // Enriquecido al proyectar
        "creada",
        evento.data.total,
        JSON.stringify(evento.data.items),
        evento.metadata.timestamp,
        evento.metadata.timestamp,
      ],
    );
  }

  async onOrdenPagada(evento: OrdenPagada): Promise<void> {
    await this.db.query(
      `UPDATE ordenes_consulta
       SET estado = 'pagada', metodo_pago = $2, actualizada_en = $3
       WHERE orden_id = $1`,
      [evento.data.ordenId, evento.data.metodo, evento.metadata.timestamp],
    );
  }

  async onOrdenEnviada(evento: OrdenEnviada): Promise<void> {
    await this.db.query(
      `UPDATE ordenes_consulta
       SET estado = 'enviada', tracking = $2, transportista = $3, actualizada_en = $4
       WHERE orden_id = $1`,
      [evento.data.ordenId, evento.data.trackingNumber, evento.data.transportista, evento.metadata.timestamp],
    );
  }
}

// ─── API de consultas ───
app.get("/api/consultas/ordenes/:id", async (req, res) => {
  const { rows } = await db.query(
    `SELECT * FROM ordenes_consulta WHERE orden_id = $1`,
    [req.params.id],
  );

  if (rows.length === 0) {
    return res.status(404).json({ error: "Orden no encontrada" });
  }

  res.json({
    ordenId: rows[0].orden_id,
    estado: rows[0].estado,
    total: rows[0].total,
    items: rows[0].items,
    tracking: rows[0].tracking,
    creadaEn: rows[0].creada_en,
    actualizadaEn: rows[0].actualizada_en,
  });
});

// ─── Consultas especializadas: diferentes modelos de lectura ───

// Modelo 1: Dashboard de ventas (agregado)
app.get("/api/consultas/dashboard/ventas", async (req, res) => {
  const { rows } = await db.query(
    `SELECT
       DATE(creada_en) as fecha,
       COUNT(*) as total_ordenes,
       SUM(total) as monto_total,
       AVG(total) as ticket_promedio
     FROM ordenes_consulta
     WHERE creada_en >= NOW() - INTERVAL '30 days'
     GROUP BY DATE(creada_en)
     ORDER BY fecha DESC`
  );
  res.json(rows);
});

// Modelo 2: Órdenes por cliente (diferente estructura)
app.get("/api/consultas/clientes/:id/ordenes", async (req, res) => {
  const { rows } = await db.query(
    `SELECT orden_id, estado, total, actualizada_en
     FROM ordenes_consulta
     WHERE cliente_id = $1
     ORDER BY creada_en DESC
     LIMIT 50`,
    [req.params.id],
  );
  res.json(rows);
});
```

---

## 10.4 Sincronización entre Modelos

Hay dos estrategias para mantener sincronizados el modelo de escritura y lectura:

### Sincronización asíncrona (recomendada)

```
┌─────────────────┐
│  Command Side   │
│                 │
│  1. Guarda      │
│     eventos     │──▶ Event Store
│                 │
│  2. Publica      │
│     eventos     │──▶ Event Bus ──▶ Proyecciones ──▶ BD Lectura
└─────────────────┘
                                     │
                           Latencia: ms a segundos
```

```typescript
// La proyección se actualiza de forma asíncrona
// Ventaja: escrituras nunca se bloquean por lecturas
// Desventaja: consistencia eventual

eventBus.on("orden.creada", async (evento) => {
  await proyeccion.onOrdenCreada(evento);
});
```

### Sincronización síncrona (simplificada)

```
┌─────────────────┐
│  Command Side   │
│                 │
│  1. Guarda en   │
│     BD lectura  │──▶ BD Lectura
│  2. Guarda en   │
│     BD escritura│──▶ BD Escritura
│                 │
│  (Misma transacción)     │
└─────────────────┘
         │
  Ventaja: consistencia fuerte
  Desventaja: acoplamiento, latencia
```

---

## 10.5 CQRS sin Event Sourcing

CQRS puede implementarse sin Event Sourcing, usando una base de datos relacional con modelos separados:

```typescript
// ─── CQRS Simplificado (sin Event Sourcing) ───

// Escritura: tabla normalizada
class OrdenesCommandRepository {
  async crearOrden(comando: CrearOrdenComando): Promise<void> {
    const client = await this.db.connect();
    try {
      await client.query("BEGIN");

      await client.query(
        `INSERT INTO ordenes (id, cliente_id, estado, total, version)
         VALUES ($1, $2, $3, $4, 1)`,
        [comando.data.ordenId, comando.data.clienteId, "creada", comando.data.total],
      );

      for (const item of comando.data.items) {
        await client.query(
          `INSERT INTO orden_items (orden_id, producto_id, cantidad, precio)
           VALUES ($1, $2, $3, $4)`,
          [comando.data.ordenId, item.productoId, item.cantidad, item.precioUnitario],
        );
      }

      await client.query("COMMIT");

      // Publicar evento para actualizar modelo de lectura
      await eventBus.publicar({
        type: "orden.creada",
        data: {
          ordenId: comando.data.ordenId,
          clienteId: comando.data.clienteId,
          items: comando.data.items,
          total: comando.data.total,
        },
      });
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }
}

// Lectura: tabla desnormalizada (misma BD, diferente esquema)
class OrdenesQueryService {
  async obtenerOrden(ordenId: string): Promise<OrdenDTO> {
    const { rows } = await this.db.query(
      `SELECT * FROM ordenes_consulta WHERE orden_id = $1`,
      [ordenId],
    );
    return rows[0];
  }

  async buscarOrdenes(filtros: OrdenFiltros): Promise<OrdenDTO[]> {
    let query = "SELECT * FROM ordenes_consulta WHERE 1=1";
    const params: unknown[] = [];

    if (filtros.clienteId) {
      params.push(filtros.clienteId);
      query += ` AND cliente_id = $${params.length}`;
    }
    if (filtros.estado) {
      params.push(filtros.estado);
      query += ` AND estado = $${params.length}`;
    }

    query += " ORDER BY creada_en DESC LIMIT 50";
    const { rows } = await this.db.query(query, params);
    return rows;
  }
}
```

---

## 10.6 Casos de Uso Reales

### Fintech: Transacciones Bancarias

```typescript
// Escritura: Event Sourcing para auditoría regulatoria
// Cada transacción es un evento inmutable
interface TransaccionIniciada {
  type: "transaccion.iniciada";
  data: {
    transaccionId: string;
    cuentaOrigen: string;
    cuentaDestino: string;
    monto: number;
    moneda: string;
    concepto: string;
  };
}

// Lectura: Dashboard de saldos actualizado al instante
// Proyección rápida para consultas de clientes
class ProyeccionSaldoCuenta {
  async onTransaccionCompletada(evento: TransaccionCompletada): Promise<void> {
    await this.db.query(
      `UPDATE saldos_cuentas
       SET saldo = saldo - $2,
           ultima_actualizacion = $3
       WHERE cuenta_id = $1`,
      [evento.data.cuentaOrigen, evento.data.monto, new Date()],
    );

    await this.db.query(
      `INSERT INTO saldos_cuentas (cuenta_id, saldo, ultima_actualizacion)
       VALUES ($1, $2, $3)
       ON CONFLICT (cuenta_id) DO UPDATE
       SET saldo = saldos_cuentas.saldo + $2`,
      [evento.data.cuentaDestino, evento.data.monto, new Date()],
    );
  }
}
```

### E-Commerce: Catálogo y Búsqueda

```typescript
// Escritura: productos gestionados por administradores (pocos cambios)
// Lectura: búsquedas complejas por múltiples criterios (muchas consultas)

// Modelo de lectura para búsqueda full-text
class ProyeccionCatalogoBusqueda {
  async onProductoCreado(evento: ProductoCreado): Promise<void> {
    await this.db.query(
      `INSERT INTO catalogo_busqueda
         (producto_id, nombre, descripcion, precio, categoria, etiquetas, search_vector)
       VALUES ($1, $2, $3, $4, $5, $6,
         to_tsvector('spanish', $2 || ' ' || $3 || ' ' || $6))`,
      [
        evento.data.productoId,
        evento.data.nombre,
        evento.data.descripcion,
        evento.data.precio,
        evento.data.categoria,
        evento.data.etiquetas.join(" "),
      ],
    );
  }
}

// Búsqueda optimizada
app.get("/api/buscar", async (req, res) => {
  const { rows } = await db.query(
    `SELECT producto_id, nombre, precio, categoria
     FROM catalogo_busqueda
     WHERE search_vector @@ plainto_tsquery('spanish', $1)
     ORDER BY ts_rank(search_vector, plainto_tsquery('spanish', $1)) DESC
     LIMIT 20`,
    [req.query.q],
  );
  res.json(rows);
});
```

### IoT: Telemetría de Sensores

```typescript
// Escritura: millones de eventos de sensores por segundo
// Event Sourcing en Kafka/Kinesis

// Lectura 1: Último valor conocido (para dashboards en tiempo real)
const ultimoValor = await redis.get(`sensor:${sensorId}:ultimo`);

// Lectura 2: Agregaciones horarias (para reportes)
const { rows } = await db.query(
  `SELECT
     DATE_TRUNC('hour', timestamp) as hora,
     AVG(valor) as promedio,
     MIN(valor) as minimo,
     MAX(valor) as maximo
   FROM telemetria_horaria
   WHERE sensor_id = $1 AND timestamp >= $2
   GROUP BY DATE_TRUNC('hour', timestamp)`,
  [sensorId, inicio],
);
```

---

## Resumen del Capítulo

- **CQRS** separa el modelo de escritura (comandos, invariantes, domain events) del modelo de lectura (consultas, proyecciones desnormalizadas).
- El **modelo de escritura** usa Event Sourcing y aggregates. Optimizado para integridad transaccional.
- El **modelo de lectura** usa proyecciones que transforman eventos en tablas/materializaciones optimizadas para consultas.
- La **sincronización** entre modelos es asíncrona (consistencia eventual) o síncrona (consistencia fuerte, pero acoplada).
- **CQRS sin Event Sourcing** es válido para casos simples: usar la misma BD con modelos de lectura y escritura separados.
- Casos de uso reales: fintech (transacciones vs saldos), e-commerce (catálogo vs búsqueda), IoT (eventos crudos vs agregaciones).

En el siguiente capítulo exploramos Sagas: transacciones distribuidas en el mundo event-driven.

---

← [Capítulo anterior](09-event-sourcing.md) | [Inicio](README.md) | [Capítulo siguiente →](11-sagas.md)
