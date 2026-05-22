# Capitulo 4: Apache Kafka en Profundidad

Apache Kafka es el estandar de facto para streaming de eventos a escala empresarial. No es un simple message broker: es una **plataforma de streaming distribuida** disenada para alto throughput, persistencia duradera y procesamiento en tiempo real.

---

## 4.1 Arquitectura Interna

> [!NOTE]
> ### 🍕 Consumer Groups: El Buffet de Pizza Gigante
> 
> Si no entiendes cómo Kafka escala horizontalmente la lectura de millones de mensajes sin duplicados, imagina un buffet libre con una cinta transportadora de pizza dividida en **4 carriles independientes** (nuestras **Particiones** de un Topic):
> 
> - **Un solo comensal (Consumidor único)**: Si hay un solo comensal en la mesa, tiene que estirar los brazos y vigilar los 4 carriles de la cinta transportadora a la vez para agarrar la pizza. Puede comer tranquilo, pero si la velocidad de la cinta aumenta, se saturará rápidamente.
> - **Un grupo de 4 comensales en la misma mesa (Consumer Group)**: Para organizarse, se reparten las tareas: cada uno vigila y consume los platos de **un solo carril específico**. Nadie se cruza con nadie, y comen en perfecto orden a la velocidad de la luz.
> - **¿Qué pasa si agregas un quinto comensal al grupo?** Se quedará sentado mirando de brazos cruzados. Como solo hay 4 carriles (particiones) y cada carril solo puede asignarse a un comensal del mismo grupo a la vez para evitar que dos personas agarren la misma rebanada, el quinto queda de repuesto.
> - **Rebalanceo (El relevo)**: Si uno de los 4 comensales se llena y se retira de la mesa, el grupo se reorganiza automáticamente: uno de los comensales restantes estira el brazo para cubrir y consumir el carril que quedó vacío.

```
┌──────────────────────────────────────────────────────────────┐
│                      KAFKA CLUSTER                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  Broker 1   │  │  Broker 2   │  │  Broker 3   │          │
│  │             │  │             │  │             │          │
│  │ Particion 0 │  │ Particion 0 │  │ Particion 1 │          │
│  │ (Leader)    │  │ (Follower)  │  │ (Leader)    │          │
│  │ Particion 1 │  │             │  │ Particion 2 │          │
│  │ (Follower)  │  │ Particion 2 │  │ (Follower)  │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                   ZooKeeper / KRaft                   │    │
│  │    (Metadata, leader election, controller)           │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

**Conceptos clave:**

| Concepto | Explicacion |
|----------|-------------|
| **Topic** | Categoria logica de eventos (ej: `ordenes`, `pagos`) |
| **Particion** | Division fisica de un topic. Garantiza orden dentro de ella |
| **Offset** | Identificador unico y secuencial de cada mensaje dentro de una particion |
| **Broker** | Servidor Kafka. Un cluster tiene multiples brokers |
| **Replicacion** | Cada particion tiene 1 leader + N followers para fault tolerance |
| **Consumer Group** | Conjunto de consumidores que se reparten las particiones |

---

## 4.2 Productores: Enviando Eventos

```typescript
import { Kafka, Producer, CompressionTypes } from "kafkajs";

const kafka = new Kafka({
  clientId: "servicio-ordenes",
  brokers: ["kafka-broker-1:9092", "kafka-broker-2:9092"],
});

const producer = kafka.producer({
  maxInFlightRequests: 5,        // Max mensajes en vuelo sin ack
  idempotent: true,               // Evitar duplicados (requiere acks: "all")
  transactionalId: "ordenes-tx",  // Transacciones atomicas multi-topic
});

await producer.connect();

// Enviar evento simple
await producer.send({
  topic: "ordenes",
  compression: CompressionTypes.GZIP,
  messages: [
    {
      key: "orden-123",   // Partition key → orden garantizado
      value: JSON.stringify({
        type: "orden.creada",
        data: { ordenId: "123", clienteId: "456", total: 159.98 },
      }),
      headers: {
        "event-type": "orden.creada",
        "content-type": "application/json",
      },
    },
  ],
});
```

### Configuracion critica de productores

```typescript
const producer = kafka.producer({
  // ─── Confiabilidad ───
  acks: -1,              // -1 = ALL replicas confirman (max durability)
                          // 1 = Solo leader confirma (balance)
                          // 0 = No espera confirmacion (max throughput, datos pueden perderse)

  // ─── Ordenamiento ───
  maxInFlightRequests: 1, // Solo idempotente: garantiza orden incluso con reintentos

  // ─── Idempotencia ───
  idempotent: true,       // El broker deduplica mensajes con mismo PID + seq number
                          // Requiere: acks=-1, maxInFlightRequests <= 5, sin transactionalId si <=5

  // ─── Compresion ───
  compression: CompressionTypes.GZIP,  // GZIP (max compresion), Snappy (balance), LZ4 (velocidad)

  // ─── Batching ───
  lingerMs: 10,           // Esperar hasta 10ms para agrupar mensajes
  batchSize: 16384,       // Maximo 16KB por batch

  // ─── Retry ───
  retry: {
    initialRetryTime: 100,
    retries: 5,           // Reintentar hasta 5 veces
    maxRetryTime: 30000,
  },
});
```

---

## 4.3 Consumidores: Leyendo Eventos

```typescript
const consumer = kafka.consumer({
  groupId: "servicio-facturacion",   // Consumer Group ID
  sessionTimeout: 30000,             // Tiempo sin heartbeat → rebalance
  heartbeatInterval: 3000,           // Cada 3s
  maxBytesPerPartition: 1048576,     // Max 1MB por particion por fetch
});

await consumer.connect();
await consumer.subscribe({
  topic: "ordenes",
  fromBeginning: false, // false = procesar solo eventos nuevos
});

await consumer.run({
  eachMessage: async ({ topic, partition, message, heartbeat }) => {
    const evento = JSON.parse(message.value!.toString());

    try {
      await procesarEvento(evento);

      // Commit manual del offset (MAS SEGURO)
      await consumer.commitOffsets([
        { topic, partition, offset: (Number(message.offset) + 1).toString() },
      ]);
    } catch (error) {
      // No commitear → se reprocesa
      logger.error("Error procesando evento", { offset: message.offset, error });
      throw error; // Kafka reintenta
    }

    // Heartbeat: avisar que el consumidor sigue vivo
    await heartbeat();
  },
});
```

### Estrategias de commit

```typescript
// ─── Auto commit (por defecto) ───
// autoCommit: true, autoCommitInterval: 5000
// Riesgo: mensajes procesados pero no commiteados se reprocesan si el consumidor muere

// ─── Commit manual sincrono ───
await consumer.commitOffsets([...]); // Mas lento, mas seguro

// ─── Commit manual asincrono ───
consumer.commitOffsets([...]).catch(logger.error); // Mas rapido, puede perder commits

// ─── Commit por lotes (RECOMENDADO) ───
async function procesarLote(mensajes: KafkaMessage[]): Promise<void> {
  for (const msg of mensajes) {
    await procesarEvento(msg);
  }
  // Commit al final del lote → todos o ninguno
  await consumer.commitOffsets([...]);
}
```

### Consumer Groups y Rebalance

```
Consumer Group: "servicio-facturacion"
Topic: "ordenes" (4 particiones)

              ┌──────────────┐
              │  Consumer 1  │ → Particion 0, Particion 1
              │  Consumer 2  │ → Particion 2, Particion 3
              └──────────────┘

Si Consumer 1 muere → Rebalance → Consumer 2 toma P0 y P1
Si Consumer 3 se agrega → Rebalance → se redistribuyen
```

---

## 4.4 Kafka Streams (Procesamiento dentro de Kafka)

```java
// Kafka Streams: procesamiento stateful sin cluster externo
// Transforma, agrega, une topics en tiempo real

KStream<String, Orden> ordenes = builder.stream("ordenes");

// Agrupar ordenes por cliente y contar
KTable<String, Long> conteoPorCliente = ordenes
    .groupBy((key, orden) -> orden.getClienteId())
    .count();

// Unir con tabla de clientes
KTable<String, Cliente> clientes = builder.table("clientes");
KStream<String, ClienteConOrden> enriquecido = ordenes
    .join(clientes, (orden, cliente) -> new ClienteConOrden(cliente, orden));

enriquecido.to("ordenes.enriquecidas");
```

---

## 4.5 Schema Registry y Avro

```json
// Schema Registry: gobierna esquemas de eventos
// Compatibilidad: backward, forward, full

// Evento en Avro (schema en Schema Registry)
{
  "type": "record",
  "name": "OrdenCreada",
  "namespace": "com.ecommerce.ordenes",
  "fields": [
    { "name": "ordenId", "type": "string" },
    { "name": "clienteId", "type": "string" },
    { "name": "total", "type": "double" },
    { "name": "items", "type": { "type": "array", "items": "string" } }
  ]
}
```

```typescript
// Node.js: producir con Schema Registry
import { SchemaRegistry, AvroSerializer } from "@kafkajs/confluent-schema-registry";

const registry = new SchemaRegistry({ host: "http://schema-registry:8081" });
const serializer = new AvroSerializer(registry);

const encoded = await serializer.encode("ordenes-value", {
  ordenId: "123",
  clienteId: "456",
  total: 159.98,
  items: ["P1", "P2"],
});

await producer.send({ topic: "ordenes", messages: [{ value: encoded }] });
```

---

## 4.6 Kafka en Produccion

```yaml
# Configuracion de cluster (3 brokers minimo)
num.partitions: 12                 # Particiones por topic por defecto
default.replication.factor: 3      # 3 copias de cada particion
min.insync.replicas: 2             # Minimo 2 replicas en sync para escribir
unclean.leader.election.enable: false # No elegir leaders desactualizados

# Retencion
log.retention.hours: 168           # 7 dias
log.segment.bytes: 1073741824      # 1GB por segmento

# Seguridad
ssl.keystore.location: /var/private/ssl/kafka.keystore.jks
ssl.truststore.location: /var/private/ssl/kafka.truststore.jks
sasl.enabled.mechanisms: SCRAM-SHA-512
```

**Metricas clave a monitorear:**
- `kafka.consumer.lag`: cuantos mensajes atras esta el consumidor
- `kafka.producer.record-error-rate`: tasa de errores al enviar
- `kafka.server.broker.bytesin/out`: throughput del broker
- `kafka.controller.active.count`: debe ser exactamente 1

---

## Resumen del Capitulo

- Kafka es una plataforma de streaming distribuida, no un simple broker de mensajes.
- **Particiones** garantizan orden y permiten escalado horizontal.
- Productores configuran `acks=-1` + `idempotent=true` para maxima confiabilidad.
- Consumidores usan Consumer Groups para balanceo; commit manual para control preciso.
- **Schema Registry** + Avro gobiernan la evolucion de esquemas.
- En produccion: `min.insync.replicas=2`, `unclean.leader.election=false`, monitoreo de lag.

En el siguiente capitulo exploramos los servicios de mensajeria de AWS.

---

← [Capítulo anterior](03-topologias-patrones.md) | [Inicio](README.md) | [Capítulo siguiente →](05-aws-messaging.md)
