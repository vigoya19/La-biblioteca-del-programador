# Capítulo 10: Patrones de Mensajería y Eventos

> "Los eventos son el pegamento invisible que mantiene unidos los sistemas distribuidos."

> [!TIP]
> Si deseas profundizar en la teoría de la **Arquitectura Orientada a Eventos** y ver implementaciones avanzadas paso a paso con código real de producción, te recomendamos encarecidamente consultar la [Arquitectura Orientada a Eventos: Guía Completa](../event-driven-architecture-programming-book/00-indice.md) en tu workspace.

## 10.1 Fundamentos de Mensajería

La mensajería asíncrona desacopla productores de consumidores. Un mensaje se envía a un **canal** y uno o más consumidores lo procesan cuando pueden.

### Conceptos Clave

| Concepto | Definición |
|----------|-----------|
| **Mensaje** | Unidad de datos transmitida entre sistemas. |
| **Canal** | Medio por el que viajan los mensajes (cola, tópico). |
| **Productor** | Quien envía el mensaje. |
| **Consumidor** | Quien recibe y procesa el mensaje. |
| **Broker** | Intermediario que gestiona canales (Kafka, RabbitMQ, SQS). |

### Garantías de Entrega

| Garantía | Significado |
|----------|-------------|
| **At-most-once** | El mensaje se entrega 0 o 1 vez. Se puede perder. |
| **At-least-once** | El mensaje se entrega al menos 1 vez. Puede duplicarse. |
| **Exactly-once** | El mensaje se entrega exactamente 1 vez. La más difícil. |

> *Realidad práctica*: "Exactly-once" es un mito en sistemas distribuidos. Lo que obtienes es "efectivamente exactly-once" mediante idempotencia + deduplicación.

## 10.2 Tipos de Canales

### Point-to-Point (Colas)
Un mensaje → un consumidor. Ideal para distribuir trabajo.

```
Productor ──► [ Cola ] ──► Consumidor 1 (solo uno procesa)
                    └────► Consumidor 2 (no recibe este mensaje)
```

**Casos de uso**: Procesamiento de pedidos, envío de emails, encoding de video.

### Publish-Subscribe (Tópicos)
Un mensaje → todos los suscriptores. Ideal para notificar eventos.

```
Productor ──► [ Tópico ] ──► Consumidor 1 ✓
                      └────► Consumidor 2 ✓
                      └────► Consumidor 3 ✓
```

**Casos de uso**: Notificaciones, invalidación de caché, sincronización entre servicios.

## 10.3 Message Brokers

### RabbitMQ
**Modelo**: Colas + Exchanges. Basado en AMQP 0-9-1.

```
Productor → Exchange → Binding → Cola → Consumidor
              │                     │
              └──► Routing Key ─────┘
```

**Mejor para**: Tareas distribuidas, RPC, routing complejo.
**Garantías**: At-least-once (con confirmaciones), mensajes persistentes.

### Apache Kafka
**Modelo**: Log distribuido particionado.

```
Topic "pedidos"
├── Partition 0: [msg0, msg1, msg2, msg3, ...]
├── Partition 1: [msg0, msg1, msg2, ...]
└── Partition 2: [msg0, msg1, msg2, msg3, msg4, ...]
```

**Mejor para**: Streaming de eventos, alta throughput, retención a largo plazo.
**Garantías**: At-least-once (o exactly-once con transacciones Kafka).

### Comparativa

| | RabbitMQ | Kafka |
|---|----------|-------|
| **Modelo** | Colas inteligentes | Log distribuido |
| **Throughput** | Decenas de miles/s | Millones/s |
| **Latencia** | Baja (ms) | Muy baja |
| **Persistencia** | Consumo y descarte | Retención configurable |
| **Replay** | No nativo | Nativo (consumidores resetean offset) |
| **Orden** | Por cola | Por partición |
| **Mejor para** | Comandos, RPC, tareas | Eventos, analytics, streaming |

### Amazon SQS / SNS / EventBridge
- **SQS**: Colas (similar a RabbitMQ, sin exchanges).
- **SNS**: Pub/sub (similar a tópicos Kafka).
- **EventBridge**: Bus de eventos serverless con filtrado de patrones.

## 10.4 Eventos de Dominio vs Eventos de Integración

| | Evento de Dominio | Evento de Integración |
|---|---|---|
| **Ámbito** | Dentro del bounded context | Entre bounded contexts |
| **Datos** | Ricos en dominio | Datos mínimos necesarios |
| **Propósito** | Cambio de estado interno | Notificar a otros sistemas |
| **Formato** | Interno (objetos) | Externo (JSON, Avro, Protobuf) |

## 10.5 Patrones Clave de Mensajería

### Transactional Outbox
**Problema**: ¿Cómo publicar un evento de forma atómica con la escritura en BD?

```
❌ MAL:
BEGIN TRANSACTION
  INSERT INTO pedidos VALUES (...)
END TRANSACTION
// Si esto falla o el proceso muere, perdiste el evento
kafka.send("pedido_creado", evento)

✅ BIEN: Outbox Pattern
BEGIN TRANSACTION
  INSERT INTO pedidos VALUES (...)
  INSERT INTO outbox VALUES ('pedido_creado', payload)  // Misma transacción
END TRANSACTION
// Un proceso separado lee la tabla outbox y publica al broker
```

### Idempotent Consumer
**Problema**: El mismo mensaje puede entregarse múltiples veces.

```java
// El consumidor registra IDs ya procesados
public void consumir(Mensaje msg) {
    if (procesados.containsKey(msg.id())) {
        return; // Ya procesado → ignorar
    }
    procesar(msg);
    procesados.put(msg.id(), "procesado");
}
```

### Dead Letter Queue (DLQ)
Mensajes que no se pueden procesar van a una cola especial para inspección manual.

```
Cola Principal ──► (falla N veces) ──► Dead Letter Queue
                                           │
                                     Inspección manual
                                     Reintento o descarte
```

## 10.6 Órdenes y Ordenamiento

**Regla práctica**: No dependas del orden global de mensajes. Diseña para procesamiento desordenado.

Si necesitas orden:
- Kafka: Usa la misma clave de partición (todos los eventos de un `pedidoId` a la misma partición).
- RabbitMQ: Una sola cola con un solo consumidor.

---

> **Reflexión del capítulo**: La mensajería asíncrona es el superpoder de los sistemas distribuidos, pero también su mayor fuente de complejidad. Domina la idempotencia y el outbox pattern. El resto se aprende sobre la marcha.

---

← [Capítulo anterior](09-patrones-comunicacion.md) | [Inicio](README.md) | [Capítulo siguiente →](11-patrones-datos.md)
