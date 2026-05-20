# Capítulo 7: NATS y Mensajería Ligera

NATS es el sistema de mensajería más ligero y rápido del ecosistema. Diseñado para cloud-native, opera con una filosofía de "siempre disponible" y latencia sub-milisegundo. Donde Kafka dice "almaceno todo para siempre", NATS dice "entrego ya mismo y me olvido".

> "NATS is to messaging what Go is to programming languages: simple, fast, and purposeful." — Comunidad NATS

---

## 7.1 Filosofía y Arquitectura NATS

### Principios de diseño

```
KAFKA:  "Almaceno todos los eventos para siempre. Puedes leerlos cuando quieras."
RABBITMQ: "Enruto mensajes inteligentemente. Dime qué quieres y te lo entrego."
NATS:    "Entrego mensajes al instante. Si no estás escuchando, lo siento."
```

```
┌──────────────────────────────────────────────────────────────┐
│                    NATS CLUSTER                               │
│                                                              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                 │
│  │ NATS Pod │◀─▶│ NATS Pod │◀─▶│ NATS Pod │   (auto-cluster)│
│  │ (leaf)   │   │ (leaf)   │   │ (leaf)   │                 │
│  └──────────┘   └──────────┘   └──────────┘                 │
│                                                              │
│  Clientes conectan a CUALQUIER nodo (full mesh awareness)    │
│  Sin líder, sin elecciones, sin ZooKeeper                    │
└──────────────────────────────────────────────────────────────┘
```

| Característica | NATS |
|----------------|------|
| **Latencia** | <1ms (sub-milisegundo) |
| **Throughput** | Millones msg/s |
| **Protocolo** | Texto plano (similar a SMTP), binario disponible |
| **Clustering** | Full mesh, auto-descubrimiento |
| **Persistencia** | JetStream (opcional, desde NATS 2.2) |
| **Seguridad** | TLS, JWT, NKeys, cuentas multi-tenant |
| **Topologías** | Leaf nodes, gateways, super-clusters globales |
| **Lenguajes** | Clientes en 40+ lenguajes |

---

## 7.2 NATS Core: Pub/Sub y Request/Reply

### Publish/Subscribe

```typescript
import { connect, StringCodec } from "nats";

const nc = await connect({ servers: ["nats://localhost:4222"] });
const sc = StringCodec();

// ─── Productor: publicar evento ───
nc.publish("ordenes.creadas", sc.encode(JSON.stringify({
  type: "orden.creada",
  data: { ordenId: "123", clienteId: "456", total: 159.98 },
})));

// ─── Consumidor: suscribirse a sujetos ───
const sub = nc.subscribe("ordenes.creadas");
console.log(`Escuchando en: ${sub.getSubject()}`);

(async () => {
  for await (const msg of sub) {
    const evento = JSON.parse(sc.decode(msg.data));
    await procesarEvento(evento);

    // NATS Core: sin ACK. El mensaje se recibe o no.
  }
})();

// ─── Sujeto con wildcards ───
// "ordenes.>"   → todas las órdenes (creadas, pagadas, enviadas...)
// "ordenes.*"   → exactamente un nivel (creadas, pagadas, enviadas)
// "eventos.*.error" → eventos.cualquier-servicio.error

const subTodas = nc.subscribe("ordenes.>");
const subUnNivel = nc.subscribe("ordenes.*");
```

### Request/Reply (RPC asíncrono)

NATS implementa request/reply nativo sin configurar colas de respuesta:

```
Cliente ──Request──▶ "ordenes.crear" ──▶ Servicio
                                           │
Cliente ◀──Reply──── "ordenes.crear" ◀─────┘
```

```typescript
// ─── Servicio: manejar requests ───
const servicio = nc.subscribe("ordenes.crear");

(async () => {
  for await (const msg of servicio) {
    const { clienteId, items } = JSON.parse(sc.decode(msg.data));

    try {
      const orden = await crearOrden({ clienteId, items });

      // Responder al cliente que hizo el request
      msg.respond(sc.encode(JSON.stringify({
        success: true,
        ordenId: orden.id,
      })));
    } catch (error) {
      msg.respond(sc.encode(JSON.stringify({
        success: false,
        error: error.message,
      })));
    }
  }
})();

// ─── Cliente: hacer request con timeout ───
const respuesta = await nc.request("ordenes.crear", sc.encode(JSON.stringify({
  clienteId: "456",
  items: [{ productoId: "P1", cantidad: 2 }],
})), { timeout: 5000 }); // 5 segundos

const resultado = JSON.parse(sc.decode(respuesta.data));
console.log(resultado.ordenId);

// Si ningún servicio responde en 5s → Error: timeout
// Si múltiples servicios responden → el cliente recibe SOLO la primera
```

### Queue Groups (Competing Consumers)

```typescript
// Mismo sujeto, mismo queue group → solo UN consumidor recibe cada mensaje

// Consumidor A
nc.subscribe("ordenes.procesar", {
  queue: "workers-ordenes", // Queue group name
});

// Consumidor B
nc.subscribe("ordenes.procesar", {
  queue: "workers-ordenes", // Mismo queue group
});

// Consumidor C
nc.subscribe("ordenes.procesar", {
  queue: "workers-ordenes",
});

// Productor publica 100 mensajes → se distribuyen entre A, B, C
// Ideal para escalado horizontal: añadir más workers sin cambiar código
```

---

## 7.3 JetStream: Persistencia y Streaming

JetStream es la capa de persistencia de NATS (desde 2.2). Agrega garantías de entrega, replay, streams y consumidores duraderos.

### Streams: tópicos persistentes

```typescript
import { connect, StringCodec, JetStreamClient } from "nats";

const nc = await connect({ servers: ["nats://localhost:4222"] });
const jsm = await nc.jetstreamManager();
const js = nc.jetstream();

// ─── Crear stream ───
await jsm.streams.add({
  name: "ORDENES",
  subjects: ["ordenes.>"],           // Todos los eventos con prefijo "ordenes."
  storage: "file",                    // File (durable) o memory (efímero)
  retention: "limits",                // Limits, interest o workqueue
  max_msgs: 1_000_000,               // Máximo 1M mensajes
  max_bytes: 1024 * 1024 * 1024,     // Máximo 1GB
  max_age: 7 * 24 * 60 * 60,         // Retención 7 días
  replicas: 3,                        // Replicación para HA
  deny_delete: true,                  // Proteger contra borrado accidental
  allow_rollup_hdrs: true,            // Permitir compactación (último valor)
});

// ─── Productor JetStream: publicar con garantía ───
const ack = await js.publish("ordenes.creadas", sc.encode(JSON.stringify({
  type: "orden.creada",
  data: { ordenId: "123", total: 159.98 },
})), {
  msgID: "evt-orden-123-creada", // Idempotencia
});

logger.info("Evento persistido", { stream: ack.stream, seq: ack.seq });

// ─── Consumidor JetStream: con ACK ───
const consumer = await js.pullSubscribe("ordenes.>", {
  mack: true,           // ACK manual
  config: {
    durable_name: "servicio-facturacion", // Consumidor duradero (sobrevive reinicios)
    ack_policy: "explicit",               // ACK explícito
    max_deliver: 5,                       // Reintentar hasta 5 veces
    ack_wait: 30_000_000_000,             // 30 segundos para ACK
    filter_subject: "ordenes.creadas",    // Solo eventos de creación
  },
});

// Pull: el consumidor pide mensajes explícitamente
(async () => {
  while (true) {
    const batch = await consumer.fetch({ max_messages: 10, expires: 10_000 });

    for (const msg of batch) {
      try {
        const evento = JSON.parse(sc.decode(msg.data));
        await procesarEvento(evento);
        msg.ack(); // Confirmar procesamiento
      } catch (error) {
        msg.nak(); // Reintentar (hasta max_deliver)
      }
    }
  }
})();
```

### Políticas de retención

```typescript
// ─── Limits (por defecto): retiene según max_msgs, max_bytes, max_age ───
retention: "limits"
// Cuando se alcanza el límite, se descartan los mensajes más antiguos

// ─── Interest: retiene mientras haya consumidores activos ───
retention: "interest"
// Si un consumidor reconoce un mensaje, NATS puede descartarlo
// Para streams donde los consumidores definen el ciclo de vida

// ─── WorkQueue: cada mensaje va a SOLO UN consumidor ───
retention: "workqueue"
// Comportamiento tipo cola: un mensaje procesado se elimina
// Ideal para procesamiento de trabajos (competing consumers)
```

### Consumidores Push vs Pull

```typescript
// ─── Pull consumer (recomendado): cliente controla el ritmo ───
const pullConsumer = await js.pullSubscribe("ordenes.creadas", {
  config: { durable_name: "facturacion-pull" },
});

// El consumidor pide mensajes cuando está listo
const mensajes = await pullConsumer.fetch({ max_messages: 10, expires: 10_000 });

// ─── Push consumer: NATS empuja mensajes al consumidor ───
const pushConsumer = await js.subscribe("ordenes.creadas", {
  config: {
    durable_name: "facturacion-push",
    deliver_subject: "_INBOX.facturacion", // Sujeto interno de entrega
    flow_control: true,                     // Backpressure
    idle_heartbeat: 5_000_000_000,          // Heartbeat cada 5s
  },
});

for await (const msg of pushConsumer) {
  await procesarEvento(msg);
  msg.ack();
}
```

---

## 7.4 Key-Value Store y Object Store

JetStream incluye KV Store y Object Store, eliminando la necesidad de Redis o S3 para casos simples.

### Key-Value Store

```typescript
import { connect, nuid } from "nats";

const nc = await connect({ servers: ["nats://localhost:4222"] });
const js = nc.jetstream();
const kv = await js.views.kv("configuracion", {
  history: 10, // Guardar 10 versiones históricas por clave
  ttl: 0,       // Sin expiración
  replicas: 3,
});

// ─── Operaciones KV ───

// Escribir
await kv.put("feature_flag.nuevo_checkout", sc.encode("true"));
await kv.put("limit_rate.usuarios", sc.encode("1000"));

// Leer (último valor)
const entry = await kv.get("feature_flag.nuevo_checkout");
console.log(sc.decode(entry!.value)); // "true"

// Leer histórico (versiones anteriores)
const history = await kv.history("feature_flag.nuevo_checkout");
for await (const e of history) {
  console.log(`Revision ${e.revision}: ${sc.decode(e.value)}`);
}

// Watch: reactivo a cambios
const watch = await kv.watch("feature_flag.nuevo_checkout");
(async () => {
  for await (const e of watch) {
    console.log(`Flag cambió a: ${sc.decode(e.value)}`);
    actualizarConfiguracionServicio("nuevo_checkout", sc.decode(e.value) === "true");
  }
})();

// Caso de uso EDA: estado de saga
await kv.put(`saga.${sagaId}.estado`, sc.encode("orden_creada"));
await kv.put(`saga.${sagaId}.estado`, sc.encode("pago_procesado"));
await kv.put(`saga.${sagaId}.estado`, sc.encode("completada"));
```

### Object Store

```typescript
const os = await js.views.os("facturas", {
  storage: "file",
  replicas: 3,
  max_bytes: 5 * 1024 * 1024 * 1024, // 5GB
});

// Guardar objeto
const pdfBytes = await generarFacturaPDF(ordenId);
const info = await os.put({
  name: `factura-${ordenId}.pdf`,
  description: `Factura de orden ${ordenId}`,
}, pdfBytes);

// Leer objeto
const factura = await os.get(`factura-${ordenId}.pdf`);
await fs.writeFile(`/tmp/factura-${ordenId}.pdf`, factura!);

// Listar y eliminar
const objetos = await os.list();
for (const obj of objetos) {
  console.log(`${obj.name}: ${obj.size} bytes`);
}
```

---

## 7.5 NATS para Edge e IoT

NATS brilla en escenarios de edge computing gracias a su protocolo ligero y topología de leaf nodes:

```
┌─────────────────────────────────────────────────────────────┐
│                    Super Cluster Global                      │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Región EU    │  │ Región US    │  │ Región ASIA  │       │
│  │ (gateway)    │  │ (gateway)    │  │ (gateway)    │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                 │               │
│    ┌────▼────┐       ┌────▼────┐       ┌────▼────┐          │
│    │ Leaf    │       │ Leaf    │       │ Leaf    │          │
│    │ Fábrica │       │ Tienda  │       │ Sucursal│          │
│    └────┬────┘       └────┬────┘       └────┬────┘          │
│         │                 │                 │               │
│    ┌────▼────┐       ┌────▼────┐       ┌────▼────┐          │
│    │Sensores │       │  POS    │       │ Cámaras │          │
│    │IoT      │       │  Local  │       │  Local  │          │
│    └─────────┘       └─────────┘       └─────────┘          │
└─────────────────────────────────────────────────────────────┘
```

```typescript
// Leaf node: conecta una ubicación remota al cluster central
// Se ejecuta en el edge (fábrica, tienda, vehículo)

// Configuración de leaf node (nats-server.conf)
// leaf {
//   remotes = [
//     { url: "nats://hub-central:7422" }
//   ]
// }

// En el edge, publicar eventos de sensores
// Estos eventos se propagan al cluster central automáticamente
nc.publish("sensores.temperatura", sc.encode(JSON.stringify({
  sensorId: "temp-001",
  ubicacion: "fabrica-berlin",
  valor: 23.5,
  timestamp: new Date().toISOString(),
})));

// En la nube central, consumir eventos de todos los edges
const sub = nc.subscribe("sensores.>");
for await (const msg of sub) {
  const lectura = JSON.parse(sc.decode(msg.data));
  await almacenarEnDataLake(lectura);
}
```

---

## 7.6 NATS vs Kafka: Cuándo Usar Cada Uno

| | NATS + JetStream | Apache Kafka |
|---|-----------------|-------------|
| **Filosofía** | Simple, rápido, siempre disponible | Plataforma de streaming completa |
| **Persistencia** | JetStream (opcional) | Siempre persistente |
| **Latencia** | <1ms | ~5-10ms |
| **Throughput** | Extremadamente alto (<1M msg/s) | Muy alto (millones msg/s) |
| **Retención** | Días/semanas (configurable) | Semanas/meses/años |
| **Ordenamiento** | Por sujeto (similar a partición) | Por partición dentro de topic |
| **Replay** | JetStream consumers | Nativo con offsets |
| **Operaciones** | Muy simple (un binario) | Complejo (ZooKeeper/KRaft, brokers, tuning) |
| **Stream processing** | No nativo | Kafka Streams, ksqlDB |
| **Ecosistema** | Ligero, menos integraciones | Masivo: Connect, Schema Registry, ksqlDB |
| **Ideal para** | Microservicios, edge/IoT, cloud-native | Big data, analytics, event sourcing, integración |

---

## Resumen del Capítulo

- **NATS Core** es pub/sub y request/reply con latencia sub-milisegundo. "Siempre disponible, siempre rápido."
- **JetStream** agrega persistencia, streaming, consumidores duraderos, ACK y replay.
- **KV Store y Object Store** sustituyen Redis/S3 en casos simples, simplificando la infraestructura.
- **Leaf nodes** permiten extender el cluster a edge/IoT con propagación automática de eventos.
- NATS es ideal para **microservicios cloud-native**, **IoT/edge**, y sistemas que priorizan **latencia y simplicidad** sobre retención masiva de eventos.
- Kafka sigue siendo la elección para **event sourcing**, **big data** y ecosistemas que requieren Kafka Connect, Streams y Schema Registry.

En el siguiente capítulo conectamos EDA con Domain-Driven Design: Domain Events como ciudadanos de primera clase.
