# Capítulo 5: AWS: SQS, SNS y EventBridge

AWS ofrece tres servicios principales de mensajería que, combinados, cubren prácticamente cualquier necesidad de arquitectura orientada a eventos. La clave está en saber **cuándo usar cada uno** y cómo combinarlos.

> "No uses una cola cuando necesitas un bus de eventos. No uses un bus cuando necesitas una cola."

---

## 5.1 Amazon SQS (Simple Queue Service)

SQS es el servicio de **colas** de AWS. Garantiza que cada mensaje se entregue al menos una vez y que los consumidores puedan procesarlos a su ritmo.

### SQS Standard vs SQS FIFO

| Característica | Standard | FIFO |
|---------------|----------|------|
| **Throughput** | Ilimitado (casi) | 300 msg/s (petición), 3,000 msg/s (lote) |
| **Orden** | Best-effort (puede desordenarse) | Garantizado (first-in-first-out) |
| **Entrega** | Al menos una vez (puede duplicar) | Exactamente una vez (deduplicación) |
| **Colas FIFO** | No | Sí, con deduplication ID |
| **Casos de uso** | Procesamiento desacoplado, batch jobs | Eventos de negocio que requieren orden estricto |

### Anatomía de una cola SQS

```
                          ┌──────────────────────────────┐
                          │         SQS COLA              │
                          │                              │
  Productor ──SendMessage──▶  ┌────┬────┬────┬────┐     │
                          │  │ M1 │ M2 │ M3 │ M4 │     │
                          │  └────┴────┴────┴────┘     │
                          │         │         │         │
                          │    Consumidor A  Consumidor B│
                          │    (M1, M3)     (M2, M4)    │
                          └──────────────────────────────┘
```

```typescript
import { SQSClient, SendMessageCommand, ReceiveMessageCommand, DeleteMessageCommand } from "@aws-sdk/client-sqs";

const sqs = new SQSClient({ region: "us-east-1" });
const COLA_ORDENES = "https://sqs.us-east-1.amazonaws.com/123456789012/ordenes.fifo";

// ─── Productor: enviar comando ───
async function enviarComandoCrearOrden(datos: CrearOrdenDTO, idempotencyKey: string) {
  await sqs.send(new SendMessageCommand({
    QueueUrl: COLA_ORDENES,
    MessageBody: JSON.stringify({
      type: "crear_orden",
      data: datos,
      commandId: idempotencyKey,
    }),
    MessageGroupId: `orden-${datos.clienteId}`,   // FIFO: orden por cliente
    MessageDeduplicationId: idempotencyKey,         // FIFO: evitar duplicados
    MessageAttributes: {
      "event-type": { DataType: "String", StringValue: "crear_orden" },
      "content-type": { DataType: "String", StringValue: "application/json" },
    },
  }));
}

// ─── Consumidor: long polling ───
async function consumirCola(): Promise<void> {
  while (true) {
    const respuesta = await sqs.send(new ReceiveMessageCommand({
      QueueUrl: COLA_ORDENES,
      MaxNumberOfMessages: 10,        // Lote de hasta 10 mensajes
      WaitTimeSeconds: 20,            // Long polling (reduce costos y latencia)
      VisibilityTimeout: 30,          // Tiempo para procesar antes de que reaparezca
    }));

    for (const msg of respuesta.Messages ?? []) {
      try {
        const evento = JSON.parse(msg.Body!);
        await procesarEvento(evento);

        // Borrar mensaje: confirmación de procesamiento exitoso
        await sqs.send(new DeleteMessageCommand({
          QueueUrl: COLA_ORDENES,
          ReceiptHandle: msg.ReceiptHandle!,
        }));
      } catch (error) {
        logger.error("Error procesando mensaje SQS", {
          messageId: msg.MessageId,
          error,
        });
        // No borrar → el mensaje reaparece después del VisibilityTimeout
      }
    }
  }
}
```

### Visibility Timeout: el corazón de SQS

> [!TIP]
> ### 📚 El Préstamo de Libros con Alarma de Devolución (Visibility Timeout & Heartbeat)
> 
> Imagina que vas a una biblioteca pública (la cola de mensajería SQS):
> - Pides un libro muy solicitado para resolver una tarea (el mensaje). La biblioteca te lo entrega y activa un temporizador de **30 minutos** (el **Visibility Timeout**).
> - Durante esos 30 minutos, el libro desaparece del catálogo público para que nadie más intente llevárselo a casa mientras tú trabajas en él.
> - **El problema**: Tu tarea es sumamente larga y te tomará 2 horas resolverla.
>   - **Si te quedas callado**: Al minuto 31, la biblioteca asumirá que te quedaste dormido, pondrá el libro de nuevo como "Disponible" y otro estudiante (un consumidor duplicado) se lo llevará, haciendo que ambos hagan la misma tarea dos veces.
>   - **La solución (El Heartbeat)**: Cada 25 minutos, envías un mensaje rápido a la biblioteca diciendo: *"Oye, sigo despierto y trabajando, por favor extiendan mi préstamo 30 minutos más"* (`ChangeMessageVisibility`).
> 
> Este latido constante (Heartbeat) mantiene tu exclusividad hasta que terminas, momento en el cual devuelves y destruyes la ficha de préstamo (`DeleteMessage`).

```
Tiempo ──────────────────────────────────────────────────────▶

  Consumer A recibe M1
  │
  │  ◀─── Visibility Timeout (30s) ───▶
  │  M1 invisible para otros consumidores
  │                                      │
  │                                      ├─ Consumer A procesa OK → DeleteMessage
  │                                      │
  │                                      ├─ Consumer A falla/crash → M1 reaparece
  │                                         (Consumer B puede retomarlo)
```

**Estrategia recomendada**: Visibility Timeout dinámico con `ChangeMessageVisibility`:

```typescript
async function consumirConHeartbeat(msg: Message): Promise<void> {
  const tiempoInicial = 30; // segundos

  // Extender visibilidad si el procesamiento sigue en curso
  const heartbeat = setInterval(async () => {
    await sqs.send(new ChangeMessageVisibilityCommand({
      QueueUrl: COLA_ORDENES,
      ReceiptHandle: msg.ReceiptHandle!,
      VisibilityTimeout: tiempoInicial, // Resetear a 30s
    }));
  }, (tiempoInicial - 5) * 1000); // Cada 25s

  try {
    await procesarEvento(msg);
    clearInterval(heartbeat);
    await sqs.send(new DeleteMessageCommand({
      QueueUrl: COLA_ORDENES,
      ReceiptHandle: msg.ReceiptHandle!,
    }));
  } catch (error) {
    clearInterval(heartbeat);
    throw error;
  }
}
```

### Dead-Letter Queue (DLQ)

Los mensajes que fallan repetidamente no deben bloquear la cola. Van a una DLQ:

```typescript
// ─── Configuración de cola con DLQ ───

// Cola principal: ordenes.fifo
// DLQ asociada: ordenes-dlq.fifo
// Redrive Policy: después de 3 intentos, mover a DLQ

// Lambda o consumer de DLQ: inspeccionar, registrar, alertar
async function monitorearDLQ(): Promise<void> {
  const response = await sqs.send(new ReceiveMessageCommand({
    QueueUrl: "https://sqs.us-east-1.amazonaws.com/123456789012/ordenes-dlq.fifo",
    MaxNumberOfMessages: 10,
    WaitTimeSeconds: 5,
    VisibilityTimeout: 300, // 5 minutos para inspección manual
  }));

  for (const msg of response.Messages ?? []) {
    const atributos = JSON.parse(msg.Body!);

    logger.error("Mensaje en DLQ", {
      messageId: msg.MessageId,
      eventType: msg.MessageAttributes?.["event-type"]?.StringValue,
      sentTimestamp: atributos.ts,
      body: atributos,
    });

    // Opción 1: Mover de vuelta a la cola principal tras corregir el bug
    // await sqs.send(new SendMessageCommand({
    //   QueueUrl: COLA_PRINCIPAL,
    //   MessageBody: msg.Body!,
    // }));

    // Opción 2: Registrar en tabla de "eventos fallidos" para inspección
    await registrarEventoFallido(atributos);
  }
}
```

### Delay Queues y Message Timers

```typescript
// Cola con delay: todos los mensajes se entregan con retraso fijo
// Útil para: programar reintentos, esperar consistencia eventual

await sqs.send(new SendMessageCommand({
  QueueUrl: COLA_ORDENES,
  MessageBody: JSON.stringify(evento),
  DelaySeconds: 60, // El mensaje no será visible durante 60 segundos
}));
```

---

## 5.2 Amazon SNS (Simple Notification Service)

SNS es el servicio de **pub/sub** de AWS. Un mensaje publicado en un tópico se distribuye a **todos** los suscriptores.

### Modelo Pub/Sub de SNS

```
┌──────────────────────────────────────────────────────────────┐
│                         SNS TOPIC                            │
│                    "ordenes.procesadas"                       │
│                                                              │
│  ┌─────────┐    ┌──────────────┐    ┌──────────────────┐    │
│  │ Ordenes │───▶│              │───▶│ SQS: facturacion │    │
│  └─────────┘    │  Fan-out a   │───▶│ SQS: envios      │    │
│                 │  TODOS los   │───▶│ Lambda: analytics│    │
│                 │  suscriptores│───▶│ HTTP: webhook    │    │
│                 └──────────────┘    └──────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

```typescript
import { SNSClient, PublishCommand, SubscribeCommand } from "@aws-sdk/client-sns";

const sns = new SNSClient({ region: "us-east-1" });
const TOPIC_ARN = "arn:aws:sns:us-east-1:123456789012:ordenes-procesadas";

// ─── Publicar evento ───
async function publicarEventoOrdenProcesada(evento: OrdenProcesada): Promise<void> {
  await sns.send(new PublishCommand({
    TopicArn: TOPIC_ARN,
    Message: JSON.stringify(evento),
    MessageAttributes: {
      "event-type": { DataType: "String", StringValue: "orden.procesada" },
      "version": { DataType: "String", StringValue: "1.0" },
    },
  }));
}
```

### Fan-Out Pattern: SNS + SQS

El patrón más común en AWS: SNS distribuye a múltiples colas SQS. Cada servicio consume de su propia cola.

```
                    ┌───────────────┐
                    │   SNS Topic    │
                    │ "orden-creada" │
                    └───┬───┬───┬───┘
                        │   │   │
            ┌───────────┘   │   └───────────┐
            ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ SQS Cola     │ │ SQS Cola     │ │ SQS Cola     │
    │ facturacion  │ │ inventario   │ │ notificaciones│
    └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
           │                │                │
    Servicio        Servicio         Servicio
    Facturación     Inventario       Notificaciones
```

```typescript
// Crear tópico y suscribir colas (Infraestructura como código)
// CDK / CloudFormation / Terraform:

const topic = new sns.Topic(this, "OrdenCreadaTopic");

// Cada servicio tiene su propia cola, suscrita al mismo topic
topic.addSubscription(new snsSub.SqsSubscription(colafacturacion));
topic.addSubscription(new snsSub.SqsSubscription(colaInventario));
topic.addSubscription(new snsSub.SqsSubscription(colaNotificaciones));
```

**Ventajas del patron SNS + SQS:**
- Cada servicio consume a su ritmo (colas independientes)
- Si un consumidor falla, los demás no se ven afectados
- Reintentos nativos por consumidor gracias a SQS
- DLQ independiente por servicio

### Message Filtering

SNS permite filtrar mensajes para que cada suscriptor reciba solo lo que le interesa:

```typescript
// Suscribir con filtro: solo eventos de tipo "orden.pagada"
await sns.send(new SubscribeCommand({
  TopicArn: TOPIC_ARN,
  Protocol: "sqs",
  Endpoint: COLA_FACTURACION_ARN,
  Attributes: {
    FilterPolicy: JSON.stringify({
      "event-type": ["orden.pagada"],  // Solo este tipo
    }),
  },
}));

// Suscriptor de analytics recibe TODOS los tipos de orden
await sns.send(new SubscribeCommand({
  TopicArn: TOPIC_ARN,
  Protocol: "sqs",
  Endpoint: COLA_ANALYTICS_ARN,
  Attributes: {
    FilterPolicy: JSON.stringify({
      "event-type": [
        { prefix: "orden." },  // Todos los que empiecen con "orden."
      ],
    }),
  },
}));

// Filtro por atributos anidados (solo órdenes de alto valor)
await sns.send(new SubscribeCommand({
  TopicArn: TOPIC_ARN,
  Protocol: "sqs",
  Endpoint: COLA_FRAUDE_ARN,
  Attributes: {
    FilterPolicy: JSON.stringify({
      "event-type": ["orden.pagada"],
      "total": [{ numeric: [">", 1000] }],  // Solo montos > 1000
    }),
  },
}));
```

---

## 5.3 Amazon EventBridge

EventBridge es el **bus de eventos serverless** de AWS. Es la evolución de CloudWatch Events y el servicio más potente para EDA en AWS.

### EventBridge vs SNS vs SQS

| | SQS | SNS | EventBridge |
|---|-----|-----|-------------|
| **Propósito** | Cola de trabajo | Pub/Sub simple | Bus de eventos empresarial |
| **Consumidores** | Compiten por mensajes | Todos reciben cada mensaje | Reglas definen quién recibe qué |
| **Schema** | No | No | Schema Registry, validación |
| **Origen** | Cualquiera (SDK) | Cualquiera (SDK) | SDK + servicios AWS nativos |
| **Destino** | Consumidores polling | Suscriptores push | 20+ destinos AWS + API Destinations |
| **Filtrado** | No (solo queue-level) | Message Attributes | Content-based filtering (reglas) |
| **Transformación** | No | No | Input Transformer |
| **Mejor para** | Desacoplar y amortiguar | Broadcast simple | Enrutamiento complejo, eventos AWS |

### Arquitectura de EventBridge

```
┌───────────────────────────────────────────────────────────────┐
│                     EVENTBRIDGE BUS                            │
│                   "ecommerce-events"                            │
│                                                                │
│  ┌─────────────┐      ┌──────────────────────┐                │
│  │   Eventos   │      │      REGLAS           │                │
│  │             │─────▶│                        │               │
│  │ • orden     │      │ Regla 1: "orden.*"     │──▶ Facturación│
│  │   .creada   │      │    → Lambda facturar   │               │
│  │ • pago      │      │                        │               │
│  │   .procesado│      │ Regla 2: "pago.procesado"│──▶ Envíos    │
│  │ • envio     │      │    → Step Function envio│              │
│  │   .prog     │      │                        │               │
│  └─────────────┘      │ Regla 3: "orden.creada" │──▶ Analytics │
│                        │    → Kinesis Firehose   │              │
│                        └──────────────────────┘                │
└───────────────────────────────────────────────────────────────┘
```

### Publicar eventos en EventBridge

```typescript
import { EventBridgeClient, PutEventsCommand } from "@aws-sdk/client-eventbridge";

const eventBridge = new EventBridgeClient({ region: "us-east-1" });

// ─── Evento siguiendo formato EventBridge ───
async function emitirOrdenCreada(orden: Orden): Promise<void> {
  await eventBridge.send(new PutEventsCommand({
    Entries: [
      {
        Source: "com.ecommerce.ordenes",        // Origen del evento
        DetailType: "OrdenCreada",               // Tipo (PascalCase por convención AWS)
        Detail: JSON.stringify({                 // Payload: string JSON
          ordenId: orden.id,
          clienteId: orden.clienteId,
          total: orden.total,
          items: orden.items,
          timestamp: new Date().toISOString(),
        }),
        EventBusName: "ecommerce-events",        // Bus personalizado
        // Time: solo si el evento ocurrió en el pasado
        // TraceHeader: para trazas distribuidas (X-Ray)
      },
    ],
  }));
}

// ─── Envío en lote (hasta 10 eventos por llamada) ───
const eventos = ordenes.map(orden => ({
  Source: "com.ecommerce.ordenes",
  DetailType: "OrdenCreada",
  Detail: JSON.stringify(orden),
  EventBusName: "ecommerce-events",
}));

// EventBridge puede fallar parcialmente: verificar FailedEntryCount
const response = await eventBridge.send(new PutEventsCommand({ Entries: eventos }));
if (response.FailedEntryCount && response.FailedEntryCount > 0) {
  response.Entries!.forEach((entry, i) => {
    if (entry.ErrorMessage) {
      logger.error("Evento rechazado por EventBridge", {
        index: i,
        error: entry.ErrorMessage,
        event: eventos[i],
      });
    }
  });
}
```

### Reglas de EventBridge: filtrado y enrutamiento

```typescript
// ─── Regla con event pattern (filtrado por contenido) ───
// Infraestructura como código (CloudFormation/CDK):
const reglaFacturacion = new events.Rule(this, "FacturacionRegla", {
  eventBus: busPersonalizado,
  eventPattern: {
    source: ["com.ecommerce.ordenes"],
    detailType: ["OrdenCreada", "OrdenPagada", "OrdenCancelada"],
    detail: {
      total: [{ numeric: [">", 0] }],     // Solo eventos con total > 0
    },
  },
  targets: [
    new targets.LambdaFunction(funcionFacturacion),
  ],
});
```

**Event Pattern avanzado:**

```json
{
  "source": ["com.ecommerce.ordenes", "com.ecommerce.pagos"],
  "detail-type": [
    { "prefix": "Orden" },
    { "prefix": "Pago" }
  ],
  "detail": {
    "total": [{ "numeric": [">=", 100, "<=", 5000] }],
    "items": {
      "categoria": ["electronica", "libros"]
    },
    "cliente": {
      "pais": [{ "anything-but": "IR" }],
      "vip": [true]
    }
  }
}
```

### Input Transformer: transformar el evento para el destino

```typescript
// El consumidor no necesita conocer la estructura completa del evento de origen
const reglaInventario = new events.Rule(this, "InventarioRegla", {
  eventPattern: {
    detailType: ["OrdenCreada"],
  },
  targets: [
    new targets.LambdaFunction(funcionInventario, {
      event: events.RuleTargetInput.fromObject({
        accion: "RESERVAR",
        ordenId: events.EventField.fromPath("$.detail.ordenId"),
        items: events.EventField.fromPath("$.detail.items"),
        // El consumidor recibe un objeto limpio, no el evento crudo
      }),
    }),
  ],
});
```

### Schema Registry de EventBridge

```typescript
// EventBridge mantiene un registro de esquemas por tipo de evento.
// Permite:
//  - Descubrir qué eventos existen en el bus
//  - Validar eventos contra el esquema
//  - Generar bindings de código (Java, TypeScript, Python)

// Publicar esquema para "OrdenCreada":
const schema = {
  type: "object",
  properties: {
    ordenId: { type: "string" },
    clienteId: { type: "string" },
    total: { type: "number" },
    items: {
      type: "array",
      items: {
        type: "object",
        properties: {
          productoId: { type: "string" },
          cantidad: { type: "number" },
          precio: { type: "number" },
        },
        required: ["productoId", "cantidad", "precio"],
      },
    },
  },
  required: ["ordenId", "clienteId", "total", "items"],
};

// Con el schema registrado, EventBridge valida eventos y rechaza los que no cumplan.
// También genera código para TypeScript automáticamente.
```

### API Destinations: llamadas HTTP desde eventos

```typescript
// Conectar EventBridge con APIs externas sin Lambda intermediario

// 1. Definir la conexión (credenciales)
const conexionSAP = new events.Connection(this, "ConexionSAP", {
  authorization: events.Authorization.basic("usuario-sap", secrets.SecretValue.secretsManager("sap-password")),
});

// 2. Definir el destino API
const destinoSAP = new events.ApiDestination(this, "DestinoSAP", {
  connection: conexionSAP,
  endpoint: "https://api.sap.com/v1/ordenes",
  httpMethod: events.HttpMethod.POST,
  rateLimitPerSecond: 10,
});

// 3. Regla que dispara la llamada
new events.Rule(this, "SyncSAP", {
  eventPattern: { detailType: ["OrdenCreada"] },
  targets: [new targets.ApiDestination(destinoSAP)],
});
```

---

## 5.4 Lambda como Consumidor de Eventos

Lambda es el consumidor de eventos por excelencia en AWS. La integración con EventBridge, SQS y SNS es nativa y configurable.

### Lambda + EventBridge

```typescript
// serverless.yml / SAM / CDK

// Lambda se subscribe a una regla de EventBridge
const funcionFacturacion = new lambda.Function(this, "FacturacionFn", {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset("src/facturacion"),
});

// EventBridge invoca Lambda sincrónicamente (push)
new events.Rule(this, "OrdenCreadaRegla").addTarget(
  new targets.LambdaFunction(funcionFacturacion)
);
```

```typescript
// handler.ts: código de la Lambda
import { EventBridgeEvent } from "aws-lambda";

interface OrdenCreadaDetail {
  ordenId: string;
  clienteId: string;
  total: number;
  items: { productoId: string; cantidad: number; precio: number }[];
}

export const handler = async (event: EventBridgeEvent<"OrdenCreada", OrdenCreadaDetail>): Promise<void> => {
  const { ordenId, clienteId, total, items } = event.detail;

  logger.info("Procesando orden para facturación", { ordenId, total });

  await servicioFacturacion.generarFactura({
    ordenId,
    clienteId,
    total,
    items,
  });
};
```

### Lambda + SQS (event source mapping)

```typescript
// Lambda sondea SQS automáticamente (polling)
// AWS gestiona el long polling, batching, scaling

const funcionProcesarOrdenes = new lambda.Function(this, "ProcesarOrdenesFn", {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset("src/procesar"),
  timeout: Duration.seconds(30),
  reservedConcurrentExecutions: 10, // Limitar concurrencia
});

// Event Source Mapping: Lambda lee de SQS
funcionProcesarOrdenes.addEventSource(new SqsEventSource(colaOrdenes, {
  batchSize: 10,                    // Procesar hasta 10 mensajes por invocación
  maxBatchingWindow: Duration.seconds(5), // Esperar hasta 5s para completar lote
  reportBatchItemFailures: true,    // Solo reintentar mensajes fallidos
}));
```

```typescript
// handler.ts: procesamiento por lotes con reportBatchItemFailures
import { SQSEvent, SQSRecord } from "aws-lambda";

export const handler = async (event: SQSEvent): Promise<{ batchItemFailures: { itemIdentifier: string }[] }> => {
  const fallidos: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    try {
      const evento = JSON.parse(record.body);
      await procesarEvento(evento);
    } catch (error) {
      logger.error("Error procesando mensaje", { messageId: record.messageId, error });
      fallidos.push({ itemIdentifier: record.messageId });
    }
  }

  // Solo los mensajes fallidos vuelven a la cola
  return { batchItemFailures: fallidos };
};
```

### Lambda + SNS (suscripción directa)

```typescript
// SNS invoca Lambda directamente (push), sin cola intermedia
const TOPIC_ARN = "arn:aws:sns:us-east-1:123456789012:ordenes-procesadas";

import { SNSEvent } from "aws-lambda";

export const handler = async (event: SNSEvent): Promise<void> => {
  for (const record of event.Records) {
    const mensaje = JSON.parse(record.Sns.Message);
    logger.info("Notificación recibida", { subject: record.Sns.Subject });
    await procesarNotificacion(mensaje);
  }
};

// ⚠️ SNS → Lambda sin cola: si Lambda falla, SNS reintenta (3 veces, por defecto).
// Para mayor confiabilidad: SNS → SQS → Lambda
```

---

## 5.5 AWS Step Functions para Orquestación

Step Functions implementa el patron **Mediator (Orquestación)** como servicio gestionado. Ideal para sagas y flujos de negocio complejos.

### Máquina de estados para Saga de Orden

```json
{
  "Comment": "Saga: Procesar Orden de Compra",
  "StartAt": "ValidarOrden",
  "States": {
    "ValidarOrden": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:validarOrden",
      "Next": "ReservarInventario",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "OrdenFallida" }]
    },
    "ReservarInventario": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:reservarInventario",
      "Next": "ProcesarPago",
      "Catch": [{
        "ErrorEquals": ["SinStockError"],
        "Next": "NotificarSinStock"
      }]
    },
    "ProcesarPago": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:procesarPago",
      "Next": "VerificarPago"
    },
    "VerificarPago": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.pagoExitoso",
          "BooleanEquals": true,
          "Next": "ConfirmarOrden"
        }
      ],
      "Default": "CompensarInventario"
    },
    "ConfirmarOrden": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:confirmarOrden",
      "Next": "ProgramarEnvio"
    },
    "ProgramarEnvio": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:programarEnvio",
      "End": true
    },

    "─── Estados de compensación ───": "────────────",
    "CompensarInventario": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:liberarInventario",
      "Next": "OrdenFallida"
    },
    "NotificarSinStock": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:notificarSinStock",
      "Next": "OrdenFallida"
    },
    "OrdenFallida": {
      "Type": "Fail",
      "Cause": "La orden no pudo ser procesada"
    }
  }
}
```

```typescript
import { SFNClient, StartExecutionCommand } from "@aws-sdk/client-sfn";

const sfn = new SFNClient({ region: "us-east-1" });
const SAGA_ARN = "arn:aws:states:us-east-1:123456789012:stateMachine:ProcesarOrdenSaga";

// Iniciar saga desde cualquier servicio
async function iniciarSagaOrden(datos: CrearOrdenDTO): Promise<string> {
  const response = await sfn.send(new StartExecutionCommand({
    stateMachineArn: SAGA_ARN,
    name: `orden-${Date.now()}-${crypto.randomUUID().slice(0, 8)}`,
    input: JSON.stringify({
      clienteId: datos.clienteId,
      items: datos.items,
      direccionEnvio: datos.direccionEnvio,
      sagaId: crypto.randomUUID(),
    }),
  }));

  return response.executionArn!; // Para consultar el estado después
}
```

### Step Functions + EventBridge: orquestación híbrida

```
Step Functions puede:
1. Esperar un callback (Task Token) antes de continuar
2. Publicar/consumir eventos de EventBridge

Patrón híbrido:
  Orquestador SF coordina el flujo principal
  EventBridge distribuye notificaciones a servicios secundarios
```

---

## 5.6 Comparativa: Cuándo Usar Cada Servicio

| Escenario | Servicio | Por qué |
|-----------|----------|---------|
| **Desacoplar productor y consumidor, con buffer** | SQS | Cola amortigua picos. Consumidores van a su ritmo. |
| **Distribuir un evento a múltiples servicios** | SNS | Fan-out nativo a SQS, Lambda, HTTP, email. |
| **Enrutar eventos por contenido, con validación de esquema** | EventBridge | Reglas con filtrado, schema registry, input transformer. |
| **Orquestar un flujo de negocio complejo (saga)** | Step Functions | Visibilidad del flujo, reintentos, compensación, auditing. |
| **Baja latencia, sin servidor** | EventBridge → Lambda | Push directo, sin polling, latencia <100ms. |
| **Alta durabilidad, no perder mensajes** | SQS + DLQ | Mensajes persisten hasta 14 días. DLQ para fallos. |
| **Eventos nativos de AWS (EC2, S3, RDS)** | EventBridge | 200+ servicios AWS emiten a EventBridge automáticamente. |
| **Notificaciones push a usuarios (email, SMS, push)** | SNS | Integración nativa con múltiples protocolos de notificación. |

### Matriz de decisión combinada

```
¿El mensaje necesita orden estricto?
├── SÍ → SQS FIFO o Kafka (partición con key)
└── NO → Continuar
    ¿El mensaje va a MÚLTIPLES consumidores diferentes?
    ├── SÍ → ¿Necesitas filtrar por contenido del mensaje?
    │   ├── SÍ → EventBridge
    │   └── NO → SNS (fan-out a SQS por consumidor)
    └── NO → ¿Necesitas amortiguar carga entre productor y consumidor?
        ├── SÍ → SQS Standard
        └── NO → EventBridge → Lambda (push directo)
```

### Arquitectura recomendada para EDA en AWS

```
                            ┌───────────────────┐
                            │   EVENTBRIDGE BUS  │
                            │ "ecommerce-events" │
                            └───────┬───────────┘
                                    │
          ┌─────────────────────────┼──────────────────────┐
          │                         │                      │
          ▼                         ▼                      ▼
   ┌──────────────┐        ┌──────────────┐       ┌──────────────┐
   │ SNS + SQS    │        │ Step         │       │ EventBridge  │
   │ (fan-out)    │        │ Functions    │       │ → Lambda     │
   │              │        │ (sagas)      │       │ (push)       │
   │ Servicios    │        │              │       │              │
   │ que necesitan│        │ Flujos de    │       │ Eventos      │
   │ buffer + DLQ │        │ negocio      │       │ efímeros     │
   └──────────────┘        └──────────────┘       └──────────────┘
```

---

## Resumen del Capítulo

- **SQS** es una cola. Usala para desacoplar, amortiguar carga y garantizar procesamiento confiable. Standard para throughput, FIFO para orden estricto.
- **SNS** es pub/sub. Usalo para distribuir un evento a múltiples destinos. Combinado con SQS (fan-out) es el patrón más común de EDA en AWS.
- **EventBridge** es el bus de eventos serverless. Reglas de enrutamiento por contenido, schema registry, y conexión nativa con servicios AWS.
- **Lambda** consume eventos de los tres. SQS = polling gestionado. SNS/EventBridge = push directo.
- **Step Functions** orquestan sagas y flujos complejos con visibilidad y compensación integrada.
- La arquitectura recomendada combina los tres: EventBridge como bus central, SNS+SQS para servicios que necesitan buffer, y Step Functions para orquestación de sagas.

En el siguiente capítulo exploramos RabbitMQ y el protocolo AMQP en profundidad.

---

← [Capítulo anterior](04-apache-kafka.md) | [Inicio](README.md) | [Capítulo siguiente →](06-rabbitmq.md)
