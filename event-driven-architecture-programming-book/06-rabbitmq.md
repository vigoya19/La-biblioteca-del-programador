# Capítulo 6: RabbitMQ y AMQP

RabbitMQ implementa el protocolo **AMQP 0-9-1**, el estándar abierto más maduro para mensajería empresarial. A diferencia de Kafka (streaming) o SQS (cola simple), RabbitMQ ofrece un modelo de enrutamiento flexible que permite topologías complejas con exchanges, bindings y routing keys.

> "Kafka es para ríos de eventos. RabbitMQ es para tuberías inteligentes."

---

## 6.1 El Modelo AMQP

AMQP define un modelo estándar donde los mensajes fluyen a través de exchanges, colas y bindings:

```
┌──────────────────────────────────────────────────────────────┐
│                     RABBITMQ BROKER                           │
│                                                              │
│  Productor ──▶ [Exchange] ──binding──▶ [Queue] ──▶ Consumidor│
│                  │   ▲                     │                  │
│                  │   │    routing key      │                  │
│                  │   └─────────────────────┘                  │
│                  │                                           │
│                  ├──binding──▶ [Queue 2] ──▶ Consumidor B    │
│                  │                                           │
│                  └──binding──▶ [Queue 3] ──▶ Consumidor C    │
└──────────────────────────────────────────────────────────────┘

Productor → Exchange: publica con una routing key
Exchange → Queue: enruta según tipo y binding
Queue → Consumidor: entrega mensajes (push o pull)
```

| Componente | Descripción |
|------------|-------------|
| **Producer** | Publica mensajes al exchange |
| **Exchange** | Enruta mensajes a colas según reglas |
| **Binding** | Regla que conecta exchange con cola (routing key) |
| **Queue** | Almacena mensajes hasta que un consumidor los procesa |
| **Consumer** | Recibe mensajes de la cola |
| **Connection** | Conexión TCP con el broker |
| **Channel** | Conexión virtual dentro de una conexión TCP (multiplexación) |

---

## 6.2 Tipos de Exchange

### Direct Exchange

Enruta por coincidencia exacta de routing key. El más simple y rápido.

```
                   Direct Exchange "ordenes"
                        │
       routing key ─────┼───── routing key
       "creada"          │      "pagada"
           │             │          │
           ▼             │          ▼
    ┌──────────┐         │   ┌──────────┐
    │ Q.creadas│         │   │ Q.pagadas│
    └──────────┘         │   └──────────┘
                         │
              routing key ─── routing key
              "cancelada"     "enviada"
```

```typescript
import amqp from "amqplib";

const connection = await amqp.connect("amqp://localhost:5672");
const channel = await connection.createChannel();

// ─── Configurar topology ───
const EXCHANGE = "ordenes.direct";

await channel.assertExchange(EXCHANGE, "direct", {
  durable: true,            // Persiste tras reinicio del broker
  autoDelete: false,
});

// Colas para cada tipo de evento
await channel.assertQueue("ordenes.creadas", { durable: true });
await channel.assertQueue("ordenes.pagadas", { durable: true });
await channel.assertQueue("ordenes.enviadas", { durable: true });

// Bindings: routing key exacta
await channel.bindQueue("ordenes.creadas", EXCHANGE, "orden.creada");
await channel.bindQueue("ordenes.pagadas", EXCHANGE, "orden.pagada");
await channel.bindQueue("ordenes.enviadas", EXCHANGE, "orden.enviada");

// ─── Publicar ───
channel.publish(
  EXCHANGE,
  "orden.creada",                // routing key
  Buffer.from(JSON.stringify(evento)),
  {
    persistent: true,            // Sobrevive a reinicio del broker
    messageId: evento.id,
    headers: {
      "event-type": "orden.creada",
      "correlation-id": evento.correlationId,
    },
  }
);
```

### Topic Exchange

Enruta por patrón con wildcards (`*` una palabra, `#` cero o más palabras).

```
          Topic Exchange "ecommerce.events"
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
    ▼                 ▼                 ▼
orden.*         orden.creada.*    inventario.#

"orden.*"        → orden.creada, orden.pagada, orden.enviada
"orden.creada.*" → orden.creada.email, orden.creada.factura
"inventario.#"   → inventario.reservado, inventario.reservado.parcial, inventario.agotado.sku.123
```

```typescript
const EXCHANGE = "ecommerce.events";

await channel.assertExchange(EXCHANGE, "topic", { durable: true });

// Cola para facturación: todos los eventos de órdenes
await channel.assertQueue("facturacion.ordenes", { durable: true });
await channel.bindQueue("facturacion.ordenes", EXCHANGE, "orden.#");

// Cola para analytics: todos los eventos
await channel.assertQueue("analytics.todos", { durable: true });
await channel.bindQueue("analytics.todos", EXCHANGE, "#");

// Cola para email: solo eventos de creación de orden
await channel.assertQueue("email.confirmaciones", { durable: true });
await channel.bindQueue("email.confirmaciones", EXCHANGE, "orden.creada");

// Cola para inventario: eventos de inventario + órdenes
await channel.assertQueue("inventario.eventos", { durable: true });
await channel.bindQueue("inventario.eventos", EXCHANGE, "inventario.#");
await channel.bindQueue("inventario.eventos", EXCHANGE, "orden.creada"); // Cola con múltiples bindings
```

### Fanout Exchange

Enruta a TODAS las colas vinculadas. Ignora routing key. Broadcasting puro.

```
        Fanout Exchange "orden.confirmada"
          ┌───────────┬───────────┐
          │           │           │
          ▼           ▼           ▼
     Q.facturar   Q.email    Q.analytics
```

```typescript
await channel.assertExchange("orden.confirmada", "fanout", { durable: true });

// Tres colas, todas reciben el mismo mensaje
await channel.assertQueue("orden.facturacion", { durable: true });
await channel.assertQueue("orden.email", { durable: true });
await channel.assertQueue("orden.analytics", { durable: true });

await channel.bindQueue("orden.facturacion", "orden.confirmada", ""); // routing key ignorada
await channel.bindQueue("orden.email", "orden.confirmada", "");
await channel.bindQueue("orden.analytics", "orden.confirmada", "");

// Publicar: todos los suscriptores reciben
channel.publish("orden.confirmada", "", Buffer.from(JSON.stringify(evento)));
```

### Headers Exchange

Enruta por headers en lugar de routing key. Útil para enrutamiento multidimensional.

```typescript
await channel.assertExchange("ordenes.headers", "headers", { durable: true });

await channel.assertQueue("ordenes.vip", { durable: true });

// Binding: solo eventos donde header "x-vip" = true Y "x-region" = "eu"
await channel.bindQueue("ordenes.vip", "ordenes.headers", "", {
  "x-vip": "true",
  "x-region": "eu",
  "x-match": "all",  // "all" = AND, "any" = OR
});

channel.publish(
  "ordenes.headers",
  "", // Sin routing key
  Buffer.from(JSON.stringify(evento)),
  {
    headers: {
      "x-vip": "true",
      "x-region": "eu",
      "x-priority": "high",
    },
  }
);
```

---

## 6.3 Confirmaciones, Transacciones y Confiabilidad

### Publisher Confirms

RabbitMQ puede confirmar que un mensaje fue recibido y enrutado exitosamente:

```typescript
// Activar publisher confirms en el canal
const channel = await connection.createConfirmChannel();

// Publicar y esperar confirmación
try {
  await channel.publish(EXCHANGE, routingKey, Buffer.from(JSON.stringify(evento)), {
    persistent: true,
    mandatory: true,   // Devolver mensaje si no se puede enrutar a ninguna cola
  });

  // Esperar confirmación del broker (ACK o NACK)
  await channel.waitForConfirms();
  logger.info("Mensaje confirmado por RabbitMQ", { messageId: evento.id });
} catch (error) {
  logger.error("Mensaje rechazado por RabbitMQ", { error });
  // Estrategia: reintentar o almacenar en buffer de respaldo
}

// ─── Publicación en lote con confirmación ───
async function publicarLoteConfirmado(eventos: Evento[]): Promise<Evento[]> {
  const fallidos: Evento[] = [];

  for (const evento of eventos) {
    const publicado = channel.publish(EXCHANGE, evento.routingKey, Buffer.from(JSON.stringify(evento)));
    if (!publicado) {
      // Buffer de escritura lleno: aplicar backpressure
      await new Promise(resolve => channel.once("drain", resolve));
    }
  }

  try {
    await channel.waitForConfirms();
  } catch {
    // Reintentar eventos no confirmados
    fallidos.push(...eventos);
  }

  return fallidos;
}
```

### Consumer Acknowledgments

RabbitMQ usa ACK explícito (a diferencia de SQS que borra al recibir):

```typescript
// ─── Consumo con ACK manual (RECOMENDADO) ───
channel.consume("ordenes.creadas", async (msg) => {
  if (!msg) return;

  try {
    const evento = JSON.parse(msg.content.toString());
    await procesarEvento(evento);

    // ACK: mensaje procesado, eliminarlo de la cola
    channel.ack(msg);
  } catch (error) {
    logger.error("Error procesando", { messageId: msg.properties.messageId, error });

    // NACK con requeue: reintentar
    channel.nack(msg, false, true);  // false=requeue solo este, true=requeue

    // NACK sin requeue: descartar o mover a DLX
    // channel.nack(msg, false, false);
  }
}, { noAck: false }); // noAck: false = ACK manual (siempre en producción)
```

### Estrategias de ACK/NACK

```typescript
const MAX_REINTENTOS = 3;

channel.consume("ordenes.procesar", async (msg) => {
  if (!msg) return;

  const reintentos = (msg.properties.headers?.["x-retry-count"] ?? 0) as number;

  try {
    const evento = JSON.parse(msg.content.toString());
    await procesarOrden(evento);
    channel.ack(msg);
  } catch (error) {
    if (reintentos < MAX_REINTENTOS) {
      // Re-publicar con contador de reintentos incrementado
      channel.publish(EXCHANGE, msg.fields.routingKey, msg.content, {
        ...msg.properties,
        headers: {
          ...msg.properties.headers,
          "x-retry-count": reintentos + 1,
        },
      });
      channel.ack(msg); // ACK el mensaje original (ya lo re-publicamos)
    } else {
      // Máximo de reintentos alcanzado → no requeue → va a DLX
      logger.error("Mensaje excedió reintentos", { messageId: msg.properties.messageId, reintentos });
      channel.nack(msg, false, false); // Sin requeue
    }
  }
}, { noAck: false });
```

---

## 6.4 Dead Letter Exchanges (DLX) y TTL

RabbitMQ implementa DLQ mediante Dead Letter Exchanges: cuando un mensaje es rechazado (NACK sin requeue) o expira (TTL), se publica automáticamente en un exchange configurado.

> [!TIP]
> ### 📬 La Oficina de Cartas no Entregadas con Casilleros Temporizados (DLX & TTL)
>
> Imagina que eres un mensajero en una oficina postal de mensajería asíncrona (RabbitMQ):
> - Te entregan una carta para entregar a un cliente (un consumidor). Llegas a la casa pero no hay nadie para recibirla o la dirección tiene un error temporal (el consumidor falla).
> - **Si intentas entregarla síncronamente**: Te quedarías parado en la puerta del cliente esperando 10 horas a que regrese, lo que significa que el camión de correspondencia trasera se detendría por completo (hilo bloqueado).
> - **La solución de RabbitMQ con DLX y TTL**:
>   1. **El buzón de cartas no entregadas (Dead Letter Exchange)**: Al fallar la entrega, el cartero marca la carta con un sello de *"Rechazado temporalmente"* y la envía automáticamente a una oficina de clasificación especial llamada **Dead Letter Exchange (DLX)**.
>   2. **El Casillero Temporizado (Cola con TTL)**: El DLX coloca la carta en un casillero cerrado que tiene un temporizador físico de **10 segundos** (una cola con TTL de 10s). Nadie está autorizado a sacar la carta de allí antes de tiempo; simplemente espera.
>   3. **Re-entrega Automática**: En cuanto el temporizador de 10 segundos expira, el casillero expulsa automáticamente la carta y la redirige de vuelta a la **oficina postal principal (el exchange original)** para que el cartero intente de nuevo la entrega.
>
> **En resumen**: Combinando Dead Letter Exchanges (DLX) y Time-To-Live (TTL), RabbitMQ permite crear sofisticados bucles de reintento automatizados con tiempos de espera programados sin que tu aplicación tenga que dormir hilos de ejecución síncrones ni retener mensajes en tránsito.

```
┌─────────────────────────────────────────────────────────────┐
│ Cola "ordenes.creadas"                                      │
│   x-dead-letter-exchange: "ordenes.dlx"                     │
│   x-dead-letter-routing-key: "creada.fallida"               │
│   x-message-ttl: 86400000 (24h)                              │
│                                                             │
│   Mensaje falla → NACK sin requeue → publicado en DLX       │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ Dead Letter Exchange "ordenes.dlx"                           │
│   └── Cola "ordenes.creadas.dlq"                            │
│       │                                                     │
│       └── Consumidor DLQ: alertar, inspeccionar, reenviar   │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// ─── Configurar cola con DLX ───
await channel.assertExchange("ordenes.dlx", "direct", { durable: true });
await channel.assertQueue("ordenes.creadas.dlq", { durable: true });
await channel.bindQueue("ordenes.creadas.dlq", "ordenes.dlx", "creada.fallida");

// Cola principal con políticas de DLX y TTL
await channel.assertQueue("ordenes.creadas", {
  durable: true,
  arguments: {
    "x-dead-letter-exchange": "ordenes.dlx",       // A dónde van los mensajes fallidos
    "x-dead-letter-routing-key": "creada.fallida",  // Con qué routing key
    "x-message-ttl": 86400000,                      // TTL de 24h en milisegundos
    "x-max-length": 100000,                         // Máximo 100k mensajes
    "x-overflow": "reject-publish",                 // Rechazar al exceder capacidad
  },
});

// ─── Consumidor de DLQ ───
channel.consume("ordenes.creadas.dlq", async (msg) => {
  if (!msg) return;

  const muerte = msg.properties.headers?.["x-death"]; // RabbitMQ registra metadata de muerte
  logger.error("Mensaje en DLQ", {
    messageId: msg.properties.messageId,
    routingKey: msg.fields.routingKey,
    reason: muerte?.[0]?.reason,     // "rejected", "expired", "maxlen"
    count: muerte?.[0]?.count,       // Cuántas veces murió
    originalQueue: muerte?.[0]?.queue,
  });

  // Almacenar en base de datos para inspección manual
  await registrarEventoMuerto(msg, muerte);

  channel.ack(msg); // ACK para eliminarlo de DLQ
});
```

### Retry con TTL por mensaje y DLX

```typescript
// Patrón: reintentos con backoff exponencial usando TTL
const COLAS_REINTENTO = [
  { queue: "reintentos.10s", ttl: 10000, exchange: "reintentos.dlx" },
  { queue: "reintentos.30s", ttl: 30000, exchange: "reintentos.dlx" },
  { queue: "reintentos.1m", ttl: 60000, exchange: "reintentos.dlx" },
];

// Configurar colas de reintento encadenadas
for (const { queue, ttl } of COLAS_REINTENTO) {
  await channel.assertQueue(queue, {
    durable: true,
    arguments: {
      "x-dead-letter-exchange": EXCHANGE,        // Vuelve al exchange principal
      "x-dead-letter-routing-key": "orden.creada",
      "x-message-ttl": ttl,
    },
  });
}

// Al fallar, publicar en la cola de reintento correspondiente
function reintentarConBackoff(msg: amqp.ConsumeMessage, intento: number): void {
  const colaReintento = COLAS_REINTENTO[Math.min(intento, COLAS_REINTENTO.length - 1)];

  channel.publish("", colaReintento.queue, msg.content, {
    persistent: true,
    messageId: msg.properties.messageId,
    headers: {
      ...msg.properties.headers,
      "x-retry-count": intento,
    },
  });
  channel.ack(msg);
}

// Flujo:
// 1. Mensaje falla → va a reintentos.10s (TTL=10s)
// 2. 10s después → DLX lo devuelve al exchange principal
// 3. Si vuelve a fallar → reintentos.30s (TTL=30s)
// 4. 30s después → DLX lo devuelve...
// 5. Si agota reintentos → cola.dlq final
```

---

## 6.5 Clustering y Alta Disponibilidad

### Quorum Queues (RabbitMQ 3.8+)

Las Quorum Queues reemplazan a las mirrored queues clásicas. Usan el algoritmo Raft para consistencia:

```typescript
// Quorum Queue: tolerante a fallos, basada en consenso Raft
await channel.assertQueue("ordenes.criticas", {
  durable: true,
  arguments: {
    "x-queue-type": "quorum",         // Tipo de cola: quorum
    "x-quorum-initial-group-size": 3, // Tamaño de grupo Raft (3 nodos = tolera 1 fallo)
    "x-delivery-limit": 3,            // Máximo de entregas antes de DLQ
  },
});

// Ventajas de Quorum Queues:
// ✅ Consistencia fuerte (no se pierden mensajes confirmados)
// ✅ Recuperación automática tras fallo de nodo
// ✅ Tolerancia a particiones de red
// ❌ Mayor latencia que colas clásicas
// ❌ No soportan TTL por mensaje ni prioridades
```

### Streams (RabbitMQ 3.9+)

RabbitMQ Streams compite con Kafka: logs inmutables, consumers replay, fan-out sin colas:

```typescript
// RabbitMQ Stream: log inmutable similar a topic de Kafka
// Útil para event sourcing, replay de eventos, múltiples consumidores independientes

// Crear stream (comando rabbitmq-streams o plugin management)
// rabbitmqadmin declare queue name=eventos.ordenes queue_type=stream

// Consumir desde offset específico (replay)
await channel.consume("eventos.ordenes", (msg) => {
  const evento = JSON.parse(msg.content.toString());
  procesarEvento(evento);
}, {
  arguments: {
    "x-stream-offset": "first",  // Desde el principio
    // "x-stream-offset": "last", // Solo eventos nuevos
    // "x-stream-offset": 42,     // Offset específico
  },
});
```

### Topología de cluster en producción

```yaml
# rabbitmq.conf
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
cluster_formation.k8s.address_type = hostname
cluster_formation.k8s.service_name = rabbitmq-headless

# Quorum queues para datos críticos
quorum_commands_soft_limit = 256
quorum_commands_hard_limit = 512

# Límites de recursos
vm_memory_high_watermark.relative = 0.6
disk_free_limit.absolute = 2GB
```

---

## 6.6 RabbitMQ vs Kafka vs SQS

| | RabbitMQ | Apache Kafka | Amazon SQS |
|---|----------|-------------|------------|
| **Modelo** | Message broker (AMQP) | Streaming platform | Cola gestionada |
| **Enrutamiento** | Muy flexible (4 tipos de exchange) | Topics + particiones | Simple (1 cola) |
| **Consumidores** | Push + pull | Pull (offsets) | Polling (pull) |
| **Orden** | Por cola (FIFO) | Por partición | Standard: best-effort / FIFO: garantizado |
| **Replay** | No nativo (Streams 3.9+) | Nativo (offsets) | No |
| **Persistencia** | En disco (transciente/durable) | En disco (siempre) | Hasta 14 días |
| **Throughput** | ~50K msg/s (moderado) | Millones msg/s (muy alto) | Ilimitado (escala automático) |
| **Latencia** | Muy baja (<1ms) | Baja (ms) | Baja-media (ms) |
| **Operaciones** | Autogestionado (o CloudAMQP) | Autogestionado (o Confluent/MSK) | Totalmente gestionado |
| **Ecosistema** | Plugins (MQTT, STOMP, web) | Kafka Streams, Connect, KSQL | Integración nativa AWS |
| **Mejor para** | Enrutamiento complejo, RPC, baja latencia | Event streaming, analytics, big data | Work queues, desacoplamiento serverless |

---

## Resumen del Capítulo

- **AMQP** define un modelo de exchanges, colas y bindings que permite enrutamiento complejo.
- **4 tipos de exchange**: Direct (exacta), Topic (wildcards), Fanout (broadcast), Headers (metadata).
- **Publisher confirms** aseguran que el broker recibió el mensaje. **Consumer ACK** aseguran que se procesó.
- **Dead Letter Exchanges (DLX)** + **TTL** implementan DLQ y reintentos con backoff sin lógica adicional.
- **Quorum Queues** proporcionan alta disponibilidad con consistencia fuerte (Raft).
- **Streams** (3.9+) ofrecen logs inmutables al estilo Kafka dentro de RabbitMQ.
- RabbitMQ brilla en enrutamiento flexible, baja latencia y ecosistema de plugins. Kafka domina en streaming de alto volumen. SQS en simplicidad serverless.

En el siguiente capítulo exploramos NATS: mensajería ligera para edge, IoT y microservicios.

---

← [Capítulo anterior](05-aws-messaging.md) | [Inicio](README.md) | [Capítulo siguiente →](07-nats.md)
