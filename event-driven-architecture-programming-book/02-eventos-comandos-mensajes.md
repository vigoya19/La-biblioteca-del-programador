# Capitulo 2: Eventos, Comandos y Mensajes

La diferencia entre eventos, comandos y mensajes no es solo semantica — determina la arquitectura, la confiabilidad y la mantenibilidad del sistema. Este capitulo establece las definiciones precisas y los patrones de mensajeria que todo sistema EDA necesita.

---

## 2.1 Anatomia de un Evento

Un evento bien disenado tiene tres capas:

```typescript
// Un evento bien diseñado tiene tres capas, como un sobre de correo:
// 1. ENVELOPE (capa de transporte): el sobre en sí — quién lo envió, cuándo, qué tipo de mensaje es.
// 2. METADATA (capa de infraestructura): la etiqueta de rastreo — información técnica para trazabilidad.
// 3. PAYLOAD (capa de negocio): la carta dentro — los datos reales del evento.
//
// La interfaz de abajo usa "generics" de TypeScript (<T, D>). Los generics son
// como una plantilla reutilizable: la <T> dice "el tipo de evento puede variar"
// y la <D> dice "los datos del evento también". Así, una sola interfaz sirve
// para representar cualquier tipo de evento (orden creada, pago rechazado, etc.)
// sin tener que escribir una interfaz diferente para cada uno.
interface Evento<T extends string, D = unknown> {
  // ─── Envelope (capa de transporte) ───
  id: string;                         // UUID unico del evento
  type: T;                            // Tipo de evento: "orden.creada"
  source: string;                     // Servicio/productor: "servicio-ordenes"
  timestamp: string;                  // ISO 8601 cuando ocurrio
  specversion: string;                // Version de la especificacion (CloudEvents)

  // ─── Metadata (capa de infraestructura) ───
  correlationId?: string;             // Traza distribuida: como un número de pedido que
                                      // permite rastrear TODOS los eventos relacionados
                                      // con una misma operación original.
  causationId?: string;               // ¿Qué evento CAUSÓ este? La cadena de causa-efecto.
                                      // Si el pago generó la factura, el causationId de
                                      // la factura apunta al evento del pago.
  tenant?: string;                    // Multi-tenancy (múltiples clientes en el mismo sistema)
  partitionKey?: string;              // Ordenamiento en Kafka: todos los eventos con
                                      // la misma partitionKey se procesan en orden.
                                      // Es como un código postal: todas las cartas del
                                      // mismo código van al mismo estante.

  // ─── Payload (capa de negocio) ───
  data: D;                            // El contenido real del evento
}

// Ejemplo concreto: CloudEvents spec
const evento: Evento<"orden.creada", { ordenId: string; total: number }> = {
  id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  type: "orden.creada",
  source: "/servicios/ordenes",
  timestamp: "2024-01-15T10:30:00Z",
  specversion: "1.0",
  correlationId: "trace-abc123",
  data: {
    ordenId: "ORD-2024-001",
    total: 159.98,
  },
};
```

### La regla del payload: inmutable y completo

```typescript
// ❌ MAL: payload con punteros a otros datos
{ type: "OrdenCreada", ordenId: "123" } // El consumidor tiene que hacer lookup

// ❌ MAL: payload con datos mutables
{ type: "UsuarioActualizado", camposModificados: ["email"] } // ¿Cual era el anterior?

// ✅ BIEN: payload auto-contenido
{
  type: "OrdenCreada",
  data: {
    ordenId: "123",
    clienteId: "456",
    total: 159.98,
    items: [
      { productoId: "P1", nombre: "Libro", cantidad: 2, precio: 49.99 },
    ],
    direccionEnvio: { calle: "Principal 1", ciudad: "Madrid" },
  }
}
```

---

## 2.2 Tipos de Eventos

### Domain Events (Eventos de Dominio)

Son el corazon de DDD. Representan un cambio significativo en el estado del negocio.

```typescript
// Domain Event: lenguaje ubicuo del negocio
type DomainEvent =
  | { type: "orden.creada"; data: { ordenId: string; clienteId: string; items: Item[] } }
  | { type: "orden.pagada"; data: { ordenId: string; metodo: string; monto: number } }
  | { type: "orden.enviada"; data: { ordenId: string; tracking: string } }
  | { type: "orden.cancelada"; data: { ordenId: string; motivo: string } };
```

### Integration Events (Eventos de Integracion)

Cruzan las fronteras entre bounded contexts o servicios.

```typescript
// Integration Event: version simplificada del Domain Event para otros servicios
type IntegrationEvent =
  | { type: "factura.generada"; data: { facturaId: string; ordenId: string; pdfUrl: string } }
  | { type: "inventario.reservado"; data: { reservaId: string; items: { sku: string; cantidad: number }[] } };
```

### Notification Events (Eventos de Notificacion)

Informan que algo paso SIN transferir estado. El consumidor decide si hacer lookup.

```typescript
// Notification: "algo paso, ve a buscar los datos si te interesa"
{ type: "usuario.actualizado", data: { usuarioId: "123" } }
// El consumidor llama GET /usuarios/123 si necesita los datos completos
```

### System Events (Eventos de Sistema)

Internos del sistema, no del negocio.

```typescript
{ type: "servicio.reiniciado", data: { version: "2.1.0", motivo: "deploy" } }
{ type: "circuit_breaker.abierto", data: { servicio: "pagos", duracionMs: 5000 } }
```

---

## 2.3 Comandos: Intencion Explicita

A diferencia de los eventos (que describen lo que PASO), los comandos expresan lo que QUEREMOS que pase:

```typescript
// Comando: "Quiero que crees una orden"
interface Comando<T extends string, D = unknown> {
  type: T;                    // Imperativo: "crear_orden"
  data: D;                    // Datos necesarios para ejecutar
  commandId: string;          // Idempotency key
  esperadoPor?: string;       // ID del servicio que espera respuesta
}

const comando: Comando<"crear_orden", { clienteId: string; items: Item[] }> = {
  type: "crear_orden",
  data: { clienteId: "456", items: [{ productoId: "P1", cantidad: 2 }] },
  commandId: "cmd-abc-123", // Para idempotencia y tracing
};

// Despues de ejecutar el comando exitosamente, el servicio EMITE un evento:
// { type: "orden.creada", data: { ordenId: "789", ... } }
```

### Patron: Comando asincrono con respuesta

```typescript
// Problema: ¿Como sabe el emisor que el comando se ejecuto?
// Solucion: escuchar el evento resultante

async function crearOrden(datos: CrearOrdenDTO): Promise<Orden> {
  const commandId = crypto.randomUUID();

  // 1. Emitir comando
  await kafka.send({
    topic: "comandos.ordenes",
    key: datos.clienteId,
    value: { type: "crear_orden", commandId, data: datos },
  });

  // 2. Escuchar el evento de respuesta (con timeout)
  return new Promise((resolve, reject) => {
    const timeout = setTimeout(() => reject(new Error("Timeout")), 5000);

    consumidor.on("orden.creada", (evento) => {
      if (evento.metadata.causationId === commandId) {
        clearTimeout(timeout);
        resolve(evento.data);
      }
    });
  });
}
```

---

## 2.4 Mensajes vs Eventos

| Caracteristica | Mensaje (en general) | Evento |
|---------------|---------------------|--------|
| **Proposito** | Transportar datos entre componentes | Notificar un hecho ocurrido |
| **Dirigido a** | Un destinatario especifico | Cualquier interesado |
| **Tiempo verbal** | Presente/imperativo | Pasado |
| **Vida util** | Se consume y descarta | Inmutable, historico |
| **Ejemplo** | `{ type: "procesar_pago", orderId: "123" }` | `{ type: "pago_procesado", orderId: "123" }` |

---

## 2.5 Patrones de Mensajeria

### Publish/Subscribe (Pub/Sub)

```
Productor ──▶ [TOPICO] ──▶ Consumidor A
                    ──────▶ Consumidor B
                    ──────▶ Consumidor C

Un productor, N consumidores.
Cada consumidor recibe TODOS los mensajes.
```

```typescript
// Kafka: pub/sub via consumer groups
// Cada consumer group recibe una copia de todos los mensajes
// Grupos: "servicio-facturacion", "servicio-email", "servicio-analytics"
```

### Competing Consumers

```
Productor ──▶ [COLA] ──▶ Consumidor A
                    ──▶ Consumidor B  } Compiten por mensajes
                    ──▶ Consumidor C

Un productor, N consumidores.
Cada mensaje se entrega a SOLO UN consumidor.
```

```typescript
// SQS: cola con multiples consumidores
// RabbitMQ: cola con multiple consumers en el mismo channel
// Kafka: mismo consumer group → competing consumers entre particiones
```

### Request/Reply (asincrono)

```
Cliente ──comando──▶ Servicio
Cliente ◀──evento─── Servicio  (correlationId matchea)
```

---

## 2.6 Idempotencia, Deduplicacion y Ordenamiento

### Idempotencia

En sistemas distribuidos, los mensajes PUEDEN entregarse mas de una vez. Un consumidor debe ser idempotente:

```typescript
class OrdenConsumer {
  private procesados = new Set<string>(); // En produccion: Redis/BD

  async handle(evento: Evento<"pago.procesado">): Promise<void> {
    // Idempotencia: verificar si ya procesamos este evento
    if (this.procesados.has(evento.id)) {
      logger.warn("Evento duplicado ignorado", { eventId: evento.id });
      return;
    }

    await this.actualizarEstadoOrden(evento.data.ordenId, "pagada");

    // Marcar como procesado DESPUES de procesar exitosamente
    this.procesados.add(evento.id);
  }
}
```

### Estrategias de deduplicacion

```
1. Por eventId: guardar IDs procesados en BD/Redis (TTL de 7-30 dias)
2. Por idempotency key: el productor incluye clave unica en el comando
3. Por operacion idempotente: UPSERT en vez de INSERT, operaciones conmutativas
```

### Ordenamiento

```typescript
// Kafka garantiza orden DENTRO de una particion
// Estrategia: misma partition key = misma particion = orden garantizado

// Todos los eventos de la orden "123" van a la misma particion
await producer.send({
  topic: "ordenes",
  messages: [
    { key: "orden-123", value: evento1 }, // OrdenCreada
    { key: "orden-123", value: evento2 }, // PagoProcesado
    { key: "orden-123", value: evento3 }, // Enviada
  ],
});
// Estos 3 eventos se procesan en orden para "orden-123"
```

---

## Resumen del Capitulo

- Un evento tiene 3 capas: envelope (transporte), metadata (infra), payload (negocio).
- **Domain Events** representan cambios de negocio. **Integration Events** cruzan fronteras de servicio.
- **Comandos** expresan intencion (imperativo). **Eventos** describen hechos (pasado). **Consultas** preguntan estado.
- Pub/Sub distribuye a todos. Competing Consumers balancean carga. Request/Reply asincrono para comandos.
- La **idempotencia** no es opcional: cada consumidor DEBE manejar mensajes duplicados.
- El **ordenamiento** se garantiza por partition key en Kafka. Eventos de la misma entidad = misma particion.

En el siguiente capitulo exploramos las topologias y patrones de comunicacion en detalle.

---

← [Capítulo anterior](01-fundamentos.md) | [Inicio](README.md) | [Capítulo siguiente →](03-topologias-patrones.md)
