# Capítulo 16: Observabilidad y Monitoreo

En un sistema event-driven, un error no se manifiesta como un stack trace en un solo proceso. Atraviesa múltiples servicios, colas y brokers. Sin observabilidad adecuada, depurar es imposible. Necesitas trazas distribuidas, métricas de infraestructura de eventos y alertas inteligentes.

> "No puedes arreglar lo que no puedes ver." — Principio de Observabilidad

---

## 16.1 Los Tres Pilares + Eventos

```
┌──────────────────────────────────────────────────────────────┐
│                   OBSERVABILIDAD EDA                          │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │
│  │  Trazas  │  │ Métricas │  │   Logs   │  │   Eventos   │ │
│  │Distribuidas│ │         │  │          │  │  (Event Log)│ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬──────┘ │
│       │             │             │               │         │
│       │    ┌────────┴────────┐    │               │         │
│       └────┤ OpenTelemetry   ├────┘               │         │
│            │ (SDK + Collector)│                    │         │
│            └────────┬────────┘                    │         │
│                     │                             │         │
│           ┌─────────▼─────────┐                   │         │
│           │   Backend          │                   │         │
│           │ (Jaeger, Grafana,  │                   │         │
│           │  Honeycomb,        │                   │         │
│           │  Datadog)          │                   │         │
│           └───────────────────┘                   │         │
└──────────────────────────────────────────────────────────────┘
```

---

## 16.2 Trazas Distribuidas con OpenTelemetry

En EDA, una traza no sigue una única petición HTTP: cruza servicios mediante eventos. La clave es propagar el **trace context** en los eventos.

```
Petición HTTP (traceId: abc123)
│
├─ Servicio Órdenes: crearOrden()
│  └─ Publica OrdenCreada { spanId: s1 }
│     │
│     ├─ Servicio Pagos: onOrdenCreada { parentSpanId: s1, spanId: s2 }
│     │  └─ Publica PagoProcesado { spanId: s2 }
│     │     │
│     │     ├─ Servicio Inventario: onPagoProcesado { parentSpanId: s2, spanId: s3 }
│     │     └─ Servicio Envíos: onPagoProcesado { parentSpanId: s2, spanId: s4 }
│     │
│     └─ Servicio Notificaciones: onOrdenCreada { parentSpanId: s1, spanId: s5 }
```

```typescript
import { trace, Span, context, propagation } from "@opentelemetry/api";

const tracer = trace.getTracer("servicio-ordenes");

// ─── Propagación de contexto en eventos ───

// Helper: extraer trace context de un evento
function extraerContexto(evento: DomainEvent): Context {
  if (evento.metadata.traceParent) {
    return propagation.extract(context.active(), evento.metadata.traceParent);
  }
  return context.active();
}

// Helper: inyectar trace context en un evento
function inyectarContexto(evento: DomainEvent): void {
  const carrier = {};
  propagation.inject(context.active(), carrier);
  evento.metadata.traceParent = carrier["traceparent"];
  evento.metadata.traceState = carrier["tracestate"];
}

// ─── Productor: crear span y propagar ───
async function crearOrden(dto: CrearOrdenDTO): Promise<string> {
  return tracer.startActiveSpan("ordenes.crearOrden", async (span) => {
    try {
      span.setAttribute("orden.clienteId", dto.clienteId);
      span.setAttribute("orden.items", dto.items.length);

      const orden = Orden.crear(dto);
      await repositorio.guardar(orden);

      for (const evento of orden.eventos) {
        // Inyectar trace context en el evento
        inyectarContexto(evento);

        await eventBus.publicar(evento);
        span.addEvent("evento.publicado", {
          "event.type": evento.type,
          "event.id": evento.metadata.eventId,
        });
      }

      return orden.id;
    } catch (error) {
      span.recordException(error as Error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw error;
    } finally {
      span.end();
    }
  });
}

// ─── Consumidor: extraer contexto y crear span hijo ───
async function onOrdenCreada(evento: OrdenCreada): Promise<void> {
  const parentContext = extraerContexto(evento);

  return tracer.startActiveSpan(
    "pagos.procesarPago",
    {},
    parentContext,
    async (span) => {
      try {
        span.setAttribute("orden.id", evento.data.ordenId);
        span.setAttribute("orden.total", evento.data.total);

        const resultado = await procesador.cobrar({
          monto: evento.data.total,
          idempotencyKey: evento.metadata.eventId,
        });

        if (resultado.exitoso) {
          const pagoEvento = crearPagoProcesado(evento, resultado);
          inyectarContexto(pagoEvento);
          await eventBus.publicar(pagoEvento);
          span.setStatus({ code: SpanStatusCode.OK });
        }
      } catch (error) {
        span.recordException(error as Error);
        span.setStatus({ code: SpanStatusCode.ERROR, message: (error as Error).message });
        throw error;
      } finally {
        span.end();
      }
    },
  );
}
```

### Configuración de OpenTelemetry

```typescript
// instrumentation.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { KafkaJsInstrumentation } from "opentelemetry-instrumentation-kafkajs";
import { PgInstrumentation } from "@opentelemetry/instrumentation-pg";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: "http://otel-collector:4318/v1/traces",
  }),
  instrumentations: [
    new KafkaJsInstrumentation({
      // Captura automáticamente produce/consume de Kafka
      producerHook: (span, info) => {
        span.setAttribute("messaging.kafka.topic", info.topic);
      },
      consumerHook: (span, info) => {
        span.setAttribute("messaging.kafka.topic", info.topic);
      },
    }),
    new PgInstrumentation(), // Trazas de queries SQL
  ],
});

sdk.start();
```

---

## 16.3 Métricas Clave para Sistemas EDA

### Lag de Consumidor (Consumer Lag)

La métrica más importante en Kafka. Mide cuántos mensajes atrás va el consumidor:

```typescript
// Métricas de lag: consumer group está atrasado
import { Gauge, Registry } from "prom-client";

const consumerLag = new Gauge({
  name: "kafka_consumer_lag",
  help: "Consumer lag en mensajes",
  labelNames: ["topic", "partition", "consumer_group"],
  registers: [registry],
});

// Actualizar métrica periódicamente
setInterval(async () => {
  const lagMetrics = await kafka.admin().fetchConsumerGroupOffsets("servicio-facturacion");

  for (const { topic, partition, offset, lag } of lagMetrics) {
    consumerLag.set({ topic, partition: String(partition), consumer_group: "servicio-facturacion" }, Number(lag));
  }
}, 15000);
```

### Métricas de negocio y sistema

```typescript
// ─── Métricas de negocio ───
const ordenesCreadas = new Counter({
  name: "ordenes_creadas_total",
  help: "Total de órdenes creadas",
  labelNames: ["estado"],
});

const montoProcesado = new Counter({
  name: "pagos_monto_total",
  help: "Monto total procesado en pagos",
  labelNames: ["metodo", "moneda"],
});

// ─── Métricas de sistema ───
const eventosPublicados = new Counter({
  name: "eventos_publicados_total",
  help: "Total de eventos publicados",
  labelNames: ["event_type", "servicio"],
});

const eventosProcesados = new Counter({
  name: "eventos_procesados_total",
  help: "Total de eventos procesados",
  labelNames: ["event_type", "servicio", "resultado"], // resultado: success, error
});

const latenciaProcesamiento = new Histogram({
  name: "eventos_procesamiento_duracion_segundos",
  help: "Duración del procesamiento de eventos",
  labelNames: ["event_type"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5, 10],
});

const dlqSize = new Gauge({
  name: "eventos_dlq_size",
  help: "Cantidad de mensajes en DLQ",
  labelNames: ["queue", "servicio"],
});

// ─── Uso en handlers ───
async function onOrdenCreada(evento: OrdenCreada): Promise<void> {
  const timer = latenciaProcesamiento.startTimer({ event_type: "orden.creada" });

  try {
    await procesarOrden(evento);
    eventosProcesados.inc({ event_type: "orden.creada", servicio: "pagos", resultado: "success" });
    ordenesCreadas.inc({ estado: "pagada" });
  } catch (error) {
    eventosProcesados.inc({ event_type: "orden.creada", servicio: "pagos", resultado: "error" });
    throw error;
  } finally {
    timer();
  }
}
```

### Dashboard recomendado (Grafana)

```
┌──────────────────────────────────────────────────────────────┐
│  EDA Dashboard                                                 │
│                                                              │
│  ┌─────────────────┐ ┌─────────────────┐ ┌────────────────┐ │
│  │ Event Throughput│ │ Consumer Lag    │ │ DLQ Size       │ │
│  │  1,250 msg/s    │ │ facturacion: 12 │ │ ordenes: 3     │ │
│  │  ▲ +5%          │ │ envios: 0       │ │ pagos: 0       │ │
│  └─────────────────┘ └─────────────────┘ └────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ Latencia de procesamiento (p95)                          ││
│  │ orden.creada:    120ms ████████                          ││
│  │ pago.procesado:  450ms ██████████████████████████        ││
│  │ inventario.reservado: 85ms █████                         ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ Eventos por tipo (últimos 30 min)                        ││
│  │ orden.creada:    1,250 ██████████████████████████        ││
│  │ pago.procesado:  1,180 █████████████████████████         ││
│  │ pago.rechazado:     12 █                                 ││
│  │ inventario.no_disponible: 8 ▒                            ││
│  └──────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

---

## 16.4 Logging Estructurado

```typescript
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  formatters: {
    level(label) {
      return { level: label };
    },
  },
  // Inyectar trace context automáticamente
  mixin() {
    const span = trace.getActiveSpan();
    if (span) {
      const spanContext = span.spanContext();
      return {
        traceId: spanContext.traceId,
        spanId: spanContext.spanId,
      };
    }
    return {};
  },
});

// ─── Logging estructurado en handlers ───
async function onOrdenCreada(evento: OrdenCreada): Promise<void> {
  logger.info({
    msg: "Procesando orden creada",
    eventType: evento.type,
    eventId: evento.metadata.eventId,
    ordenId: evento.data.ordenId,
    clienteId: evento.data.clienteId,
    total: evento.data.total,
    correlationId: evento.metadata.correlationId,
  });

  try {
    await procesarPago(evento);

    logger.info({
      msg: "Pago procesado exitosamente",
      ordenId: evento.data.ordenId,
      monto: evento.data.total,
      duracion: Date.now() - startTime,
    });
  } catch (error) {
    logger.error({
      msg: "Error procesando pago",
      ordenId: evento.data.ordenId,
      error: (error as Error).message,
      stack: (error as Error).stack,
    });
    throw error;
  }
}
```

---

## 16.5 Health Checks

```typescript
// Health check para servicios EDA
app.get("/health", async (req, res) => {
  const checks: HealthCheckResult[] = [];

  // 1. Conexión al broker
  try {
    await kafka.admin().listTopics();
    checks.push({ component: "kafka", status: "healthy" });
  } catch {
    checks.push({ component: "kafka", status: "unhealthy" });
  }

  // 2. Conexión BD
  try {
    await db.query("SELECT 1");
    checks.push({ component: "database", status: "healthy" });
  } catch {
    checks.push({ component: "database", status: "unhealthy" });
  }

  // 3. Consumer lag (si es muy alto, degraded)
  try {
    const lag = await obtenerConsumerLag("servicio-pagos");
    if (lag > 1000) {
      checks.push({ component: "consumer_lag", status: "degraded", detail: `lag: ${lag}` });
    } else {
      checks.push({ component: "consumer_lag", status: "healthy", detail: `lag: ${lag}` });
    }
  } catch {
    checks.push({ component: "consumer_lag", status: "unhealthy" });
  }

  const overall = checks.every(c => c.status === "healthy") ? 200
    : checks.some(c => c.status === "unhealthy") ? 503 : 200;

  res.status(overall).json({
    status: overall === 200 ? "healthy" : "unhealthy",
    checks,
  });
});

// ─── Liveness vs Readiness ───
// /health/live: ¿El proceso está vivo? (siempre 200 mientras el proceso corra)
// /health/ready: ¿Puede recibir tráfico? (200 solo si Kafka, BD, y conexiones OK)
```

---

## 16.6 Alertas: Qué Monitorear

```
┌──────────────────────────────────────────────────────────────┐
│                    SISTEMA DE ALERTAS                         │
│                                                              │
│  CRÍTICAS (P1 - despiertan on-call):                         │
│  🔴 Consumer lag > 10,000 por más de 5 minutos               │
│  🔴 DLQ size > 100 mensajes nuevos en 1 minuto               │
│  🔴 Tasa de error > 5% en cualquier servicio                 │
│  🔴 Kafka broker caído                                       │
│                                                              │
│  WARNING (P2 - notificar en horario laboral):                │
│  🟡 Consumer lag > 1,000 por más de 10 minutos               │
│  🟡 P95 latencia > 1s para cualquier event type              │
│  🟡 DLQ size > 10 mensajes nuevos en 5 minutos               │
│  🟡 Rebalances de consumer group frecuentes (>2/min)         │
│                                                              │
│  INFO (P3 - dashboard, sin notificación):                    │
│  🔵 Throughput anómalo (bajón del 50% vs promedio)           │
│  🔵 Aumento de reintentos                                    │
│  🔵 Crecimiento de retención de disco (riesgo de llenado)    │
└──────────────────────────────────────────────────────────────┘
```

```yaml
# Ejemplo: Reglas de alerta en Prometheus/Alertmanager
groups:
  - name: kafka_alerts
    rules:
      - alert: HighConsumerLag
        expr: kafka_consumer_lag > 10000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Consumer lag alto en {{ $labels.consumer_group }}"
          description: "El grupo {{ $labels.consumer_group }} tiene lag de {{ $value }} mensajes en topic {{ $labels.topic }} partición {{ $labels.partition }}"

      - alert: DeadLetterQueueGrowing
        expr: rate(eventos_dlq_size[1m]) > 100
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "DLQ creciendo rápidamente en {{ $labels.queue }}"
```

---

## 16.7 Debugging: Reconstruir Flujos desde Eventos

Cuando algo falla, los eventos te permiten reconstruir exactamente qué pasó:

```typescript
// ─── Herramienta de debugging: reconstruir flujo de una orden ───
async function reconstruirFlujoOrden(ordenId: string): Promise<FlujoOrden> {
  const eventos = await db.query(
    `SELECT event_type, event_data, metadata, created_at
     FROM eventos
     WHERE (event_type LIKE 'orden.%' AND event_data->>'ordenId' = $1)
        OR metadata->>'causationId' IN (
          SELECT metadata->>'eventId' FROM eventos WHERE event_data->>'ordenId' = $1
        )
     ORDER BY created_at ASC`,
    [ordenId],
  );

  console.log(`\nFlujo de la orden ${ordenId}:`);
  console.log("─".repeat(60));

  for (const evt of eventos.rows) {
    const ts = new Date(evt.created_at).toISOString();
    const delta = eventos.rows.indexOf(evt) > 0
      ? `(+${new Date(evt.created_at).getTime() - new Date(eventos.rows[0].created_at).getTime()}ms)`
      : "";

    switch (evt.event_type) {
      case "orden.creada":
        console.log(`  ✅ ${ts} ${delta} | ORDEN CREADA | Total: ${evt.event_data.total} EUR`);
        break;
      case "orden.pagada":
      case "pago.procesado":
        console.log(`  💰 ${ts} ${delta} | PAGO PROCESADO | ${evt.event_data.monto} EUR`);
        break;
      case "pago.rechazado":
        console.log(`  ❌ ${ts} ${delta} | PAGO RECHAZADO | ${evt.event_data.motivo}`);
        break;
      case "inventario.reservado":
        console.log(`  📦 ${ts} ${delta} | INVENTARIO RESERVADO`);
        break;
      case "inventario.no_disponible":
        console.log(`  ⚠️  ${ts} ${delta} | INVENTARIO NO DISPONIBLE`);
        break;
      case "orden.cancelada":
        console.log(`  🗑️  ${ts} ${delta} | ORDEN CANCELADA | ${evt.event_data.motivo}`);
        break;
      case "orden.enviada":
        console.log(`  🚚 ${ts} ${delta} | ORDEN ENVIADA | Tracking: ${evt.event_data.trackingNumber}`);
        break;
    }
  }

  console.log("─".repeat(60));
}
```

---

## Resumen del Capítulo

- **Trazas distribuidas** con OpenTelemetry conectan eventos entre servicios. El trace context se propaga en el metadata del evento.
- **Métricas clave**: consumer lag (la más importante), throughput de eventos, DLQ size, latencia de procesamiento, tasa de error.
- **Logging estructurado** con traceId y spanId inyectados automáticamente para correlacionar logs con trazas.
- **Health checks** verifican conexión al broker, BD, y monitorizan consumer lag para readiness.
- **Alertas**: consumer lag > 10K por 5 min = P1 critical. DLQ creciendo > 100 msg/min = P1 critical.
- Los **eventos como log inmutable** permiten reconstruir flujos completos para debugging post-mortem.

En el siguiente capítulo final exploramos anti-patrones y buenas prácticas en EDA.

---

← [Capítulo anterior](15-testing.md) | [Inicio](README.md) | [Capítulo siguiente →](17-anti-patrones.md)
