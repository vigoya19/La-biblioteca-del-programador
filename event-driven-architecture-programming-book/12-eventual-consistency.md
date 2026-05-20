# Capítulo 12: Eventual Consistency

En sistemas distribuidos, la consistencia fuerte es cara y frágil. La consistencia eventual acepta que los datos pueden estar temporalmente desincronizados a cambio de disponibilidad, rendimiento y tolerancia a fallos. No es un bug: es una decisión arquitectónica.

> "En un sistema distribuido, vas a tener consistencia eventual. La pregunta es si la diseñas o te toma por sorpresa." — Martin Kleppmann

---

## 12.1 CAP Theorem

El teorema CAP establece que un sistema distribuido solo puede garantizar **dos** de estas tres propiedades durante una partición de red:

```
          CONSISTENCIA
          (Todos los nodos ven los mismos datos)
              /\
             /  \
            /    \
           /  CA  \
          /        \
         /          \
PARTICIÓN ──────────── DISPONIBILIDAD
(Tolera fallos    (El sistema responde
de comunicación)    aunque algunos nodos fallen)

AP: Sistemas EDA (Kafka, DynamoDB, Cassandra)
CP: Sistemas con consenso (ZooKeeper, etcd, Consul)
CA: No existe (sin particiones, es single-node)
```

**EDA elige AP**: durante una partición, el sistema sigue funcionando y acepta que los datos estarán temporalmente inconsistentes.

```
Ejemplo EDA (AP):
  Servicio de órdenes y servicio de inventario se particionan.
  → Siguen aceptando órdenes y reservas localmente.
  → Cuando se restaura la red, los eventos pendientes se sincronizan.
  → Durante la partición: disponible. Después: consistente.

Alternativa CP (no-EDA):
  Si órdenes no puede confirmar con inventario → rechaza la orden.
  → Consistente, pero indisponible durante la partición.
```

---

## 12.2 Modelos de Consistencia

### Consistencia Fuerte (Strong Consistency)

```
Escritura en nodo A ──▶ A notifica a B, C, D ──▶ Lectura en B ve el dato actualizado

Garantía: tras una escritura exitosa, cualquier lectura devuelve el último valor.
Costo: latencia alta, disponibilidad reducida en particiones.
Cuándo: sistemas financieros core, control de inventario crítico.
```

### Consistencia Eventual (Eventual Consistency)

```
Escritura en nodo A ──▶ A persiste localmente ──▶ Lectura en B: puede ser el valor antiguo
                                                       │
                                              ┌────────▼────────┐
                                              │ Tras N segundos │
                                              │ B se actualiza  │
                                              └─────────────────┘

Garantía: si no hay más escrituras, eventualmente todas las réplicas convergen.
Costo: baja latencia, alta disponibilidad.
Cuándo: catálogos, redes sociales, analytics, dashboards.
```

### Consistencia Causal (Causal Consistency)

```
Garantiza que eventos relacionados causalmente se ven en orden.
Si A responde al mensaje de B, todos los que vieron el mensaje de B verán la respuesta de A.

Ejemplo: Hilo de comentarios en redes sociales.
- Ves la publicación original (evento 1)
- Ves todos los comentarios (eventos 2..N)
- Pero los likes pueden aparecer en cualquier orden (sin relación causal)
```

### Consistencia Monotónica (Monotonic Reads)

```
Garantiza que un cliente nunca vea datos "más viejos" que los que ya vio.
Si leíste estado X en t=5, no verás estado X-1 en t=6.

Implementación: el cliente se "pega" al mismo nodo/replica para lecturas.
```

---

## 12.3 Conflict Resolution

Cuando múltiples nodos aceptan escrituras concurrentes, los conflictos son inevitables. Hay que resolverlos:

### Last-Write-Wins (LWW)

```typescript
// La escritura con timestamp más reciente gana. Simple pero peligroso.

// ❌ PROBLEMA: relojes no sincronizados entre nodos
// Nodo A (timestamp 1000) escribe: cantidad = 5
// Nodo B (timestamp 0999) escribe: cantidad = 10
// LWW: gana A (1000 > 999) → cantidad = 5
// PERO: B escribió DESPUÉS en tiempo real. El reloj de A está adelantado.

// Solo usar LWW cuando:
// 1. Las escrituras son inmutables (eventos)
// 2. Usas relojes sincronizados (NTP, AWS Time Sync)
// 3. La pérdida de datos es aceptable
```

### CRDTs (Conflict-free Replicated Data Types)

Estructuras de datos que garantizan convergencia sin coordinación:

```typescript
// ─── GCounter: contador distribuido (solo incrementa) ───
class GCounter {
  private counters: Map<string, number> = new Map(); // nodoId → valor

  incrementar(nodoId: string, delta: number): void {
    this.counters.set(nodoId, (this.counters.get(nodoId) ?? 0) + delta);
  }

  valor(): number {
    return Array.from(this.counters.values()).reduce((a, b) => a + b, 0);
  }

  merge(otro: GCounter): void {
    for (const [nodoId, valor] of otro.counters) {
      this.counters.set(
        nodoId,
        Math.max(this.counters.get(nodoId) ?? 0, valor),
      );
    }
  }
}

// Uso en EDA: contar "likes" en un post
// Cada réplica incrementa su contador local
// Al sincronizar, se mergean usando max (conmutativo, asociativo, idempotente)

// ─── PNCounter: contador con incrementos y decrementos ───
class PNCounter {
  private inc = new GCounter();
  private dec = new GCounter();

  incrementar(nodoId: string, delta: number): void {
    this.inc.incrementar(nodoId, delta);
  }

  decrementar(nodoId: string, delta: number): void {
    this.dec.incrementar(nodoId, delta);
  }

  valor(): number {
    return this.inc.valor() - this.dec.valor();
  }

  merge(otro: PNCounter): void {
    this.inc.merge(otro.inc);
    this.dec.merge(otro.dec);
  }
}

// Uso: inventario distribuido
// inc = entradas, dec = salidas
// stock = inc - dec (siempre converge, sin importar orden de merge)
```

### Custom Merge Strategy

```typescript
// Cuando las reglas de negocio dictan la resolución de conflictos

interface CarritoCompra {
  items: { productoId: string; cantidad: number; precio: number }[];
  version: number; // Vector clock simplificado
}

class CarritoMergeStrategy {
  merge(version1: CarritoCompra, version2: CarritoCompra): CarritoCompra {
    // Regla de negocio: sumar cantidades, mantener precio más reciente
    const itemsMap = new Map<string, { cantidad: number; precio: number }>();

    // Combinar items de ambas versiones
    for (const item of [...version1.items, ...version2.items]) {
      const existente = itemsMap.get(item.productoId);
      if (existente) {
        itemsMap.set(item.productoId, {
          cantidad: existente.cantidad + item.cantidad,
          precio: Math.max(existente.precio, item.precio), // Precio más reciente
        });
      } else {
        itemsMap.set(item.productoId, { cantidad: item.cantidad, precio: item.precio });
      }
    }

    return {
      items: Array.from(itemsMap.entries()).map(([productoId, data]) => ({
        productoId,
        ...data,
      })),
      version: Math.max(version1.version, version2.version),
    };
  }
}
```

---

## 12.4 Diseñando UX para Consistencia Eventual

La consistencia eventual no es solo un problema técnico: el usuario debe entender lo que está pasando.

### Patrones de UX

```
1. Mostrar el estado "intencional" inmediatamente
   "Tu orden ha sido recibida y está siendo procesada"
   → El usuario ve "En proceso" aunque internamente no esté confirmada

2. Indicadores de progreso
   [✓] Orden creada  [●] Pago en proceso  [ ] Envío pendiente

3. Optimistic UI + rollback
   Mostrar el resultado esperado. Si falla, notificar y revertir.

4. Polling + WebSocket para actualización en tiempo real
   GET /ordenes/123/estado → "pagada"
   WebSocket → evento "orden.enviada" → actualizar UI inmediatamente

5. "Read-your-writes"
   Después de escribir, las lecturas del mismo usuario siempre muestran
   sus propios cambios (aunque otras réplicas no estén actualizadas)
```

```typescript
// ─── Implementación: Read-Your-Writes ───

class OrdenesQueryService {
  // Mapa en memoria (o Redis) de escrituras recientes por usuario
  private escriturasRecientes = new Map<string, OrdenDTO>();

  async crearOrden(dto: CrearOrdenDTO, usuarioId: string): Promise<string> {
    const ordenId = crypto.randomUUID();

    // Guardar localmente para "read-your-writes"
    this.escriturasRecientes.set(ordenId, {
      ordenId,
      estado: "creada",
      total: dto.total,
      items: dto.items,
      creadaEn: new Date().toISOString(),
    });

    // Publicar evento para procesamiento asíncrono
    await eventBus.publicar({
      type: "orden.creada",
      data: { ordenId, clienteId: usuarioId, ...dto },
    });

    return ordenId;
  }

  async obtenerOrden(ordenId: string, usuarioId: string): Promise<OrdenDTO | null> {
    // 1. Verificar escrituras recientes (read-your-writes)
    const reciente = this.escriturasRecientes.get(ordenId);
    if (reciente) {
      return reciente;
    }

    // 2. Consultar modelo de lectura (puede tener pequeña latencia)
    const { rows } = await this.db.query(
      `SELECT * FROM ordenes_consulta WHERE orden_id = $1`,
      [ordenId],
    );

    if (rows.length > 0) {
      // Ya está en el modelo de lectura → limpiar caché local
      this.escriturasRecientes.delete(ordenId);
      return rows[0];
    }

    return null;
  }
}
```

---

## 12.5 Patrones de Consistencia en EDA

### Read-Your-Writes (Lee tus propias escrituras)

```typescript
// Garantiza que un usuario siempre ve sus propios cambios
// Incluso si el modelo de lectura aún no está actualizado

// Estrategia: mantener un cache de "mis escrituras recientes"
class ReadYourWritesCache {
  private cache = new Map<string, { data: unknown; timestamp: number }>();

  guardar(clave: string, data: unknown): void {
    this.cache.set(clave, { data, timestamp: Date.now() });
  }

  obtener<T>(clave: string): T | undefined {
    const entry = this.cache.get(clave);
    if (!entry) return undefined;

    // Expirar después de 30s (suficiente para que la proyección se actualice)
    if (Date.now() - entry.timestamp > 30000) {
      this.cache.delete(clave);
      return undefined;
    }

    return entry.data as T;
  }
}
```

### Session Consistency (Consistencia de Sesión)

```typescript
// Dentro de una sesión, garantiza lecturas monotónicas
// El cliente siempre lee de la misma réplica

class SessionConsistentRouter {
  private sessionReplica = new Map<string, string>(); // sessionId → replicaId

  async query(sessionId: string, query: string, params: unknown[]): Promise<unknown> {
    const replicaId = this.sessionReplica.get(sessionId);

    if (replicaId) {
      // Leer de la misma réplica que antes
      return this.leerDeReplica(replicaId, query, params);
    }

    // Primera consulta: elegir réplica y guardarla para la sesión
    const replica = await this.elegirReplica();
    this.sessionReplica.set(sessionId, replica.id);
    return this.leerDeReplica(replica.id, query, params);
  }
}
```

### Eventual Consistency con Versionado

```typescript
// Cada registro tiene un número de versión
// Los clientes envían la versión que leyeron
// El servidor rechaza escrituras sobre versiones obsoletas

interface ActualizarOrdenDTO {
  version: number; // "Estoy modificando la versión 3 de este registro"
  cambios: Partial<OrdenDTO>;
}

async function actualizarOrden(ordenId: string, dto: ActualizarOrdenDTO): Promise<void> {
  const { rows } = await db.query(
    `UPDATE ordenes
     SET ${buildSetClause(dto.cambios)}, version = version + 1
     WHERE orden_id = $1 AND version = $2
     RETURNING version`,
    [ordenId, dto.version],
  );

  if (rows.length === 0) {
    throw new ConflictError(
      `La orden fue modificada por otro usuario. Recarga e intenta de nuevo.`
    );
  }
}
```

---

## 12.6 Cuándo la Consistencia Eventual NO es Aceptable

Hay escenarios donde la consistencia eventual no es suficiente y necesitas consistencia fuerte:

```
✅ CONSISTENCIA EVENTUAL ACEPTABLE:
  - Catálogo de productos (actualizaciones poco frecuentes)
  - Likes/comentarios en redes sociales
  - Analytics dashboards (datos agregados)
  - Recomendaciones personalizadas
  - Perfil de usuario (último cambio gana)

❌ REQUIERE CONSISTENCIA FUERTE:
  - Transferencias bancarias (no duplicar, no perder)
  - Reserva de asientos/vuelos (doble venta inaceptable)
  - Subastas (última puja gana, orden estricto)
  - Control de inventario crítico (no vender sin stock)
  - Autenticación/autorización (no debe haber ventana de inseguridad)
```

### Soluciones híbridas

```typescript
// Combinar consistencia fuerte para operaciones críticas
// y eventual para el resto

class ServicioPagos {
  // CRÍTICO: procesar pago usa consistencia fuerte
  async procesarPago(ordenId: string): Promise<void> {
    // Transacción ACID en base de datos relacional
    await db.transaction(async (tx) => {
      const orden = await tx.ordenes.findUnique({ where: { id: ordenId }, lock: "FOR UPDATE" });
      if (orden.estado !== "pendiente") throw new Error("Orden no pagable");

      await tx.pagos.create({ data: { ordenId, monto: orden.total } });
      await tx.ordenes.update({ where: { id: ordenId }, data: { estado: "pagada" } });
    });

    // Evento para lecturas eventuales (analytics, notificaciones)
    await eventBus.publicar({
      type: "orden.pagada",
      data: { ordenId, monto: orden.total },
    });
  }

  // NO CRÍTICO: dashboard de ventas lee de proyección eventual
  async obtenerDashboardVentas(): Promise<VentasDashboard> {
    return this.db.query(
      `SELECT SUM(total) as ingresos, COUNT(*) as ordenes
       FROM ordenes_consulta
       WHERE estado = 'pagada' AND actualizada_en > NOW() - INTERVAL '24 hours'`,
    );
  }
}
```

---

## Resumen del Capítulo

- **CAP Theorem**: los sistemas EDA eligen AP (disponibilidad + tolerancia a particiones), aceptando consistencia eventual.
- **Modelos de consistencia**: fuerte, eventual, causal y monotónica. Cada una con sus trade-offs.
- **Resolución de conflictos**: Last-Write-Wins (simple, arriesgado), CRDTs (convergencia garantizada), merge strategies custom (reglas de negocio).
- **UX para consistencia eventual**: mostrar intención inmediata, indicadores de progreso, optimistic UI, read-your-writes.
- **Patrones**: Read-Your-Writes, Session Consistency, versionado optimista.
- **No todo es eventual**: para operaciones críticas (pagos, reservas), usar consistencia fuerte con transacciones ACID locales.

En el siguiente capítulo implementamos un sistema E-Commerce event-driven completo.

---

← [Capítulo anterior](11-sagas.md) | [Inicio](README.md) | [Capítulo siguiente →](13-implementacion-ecommerce.md)
