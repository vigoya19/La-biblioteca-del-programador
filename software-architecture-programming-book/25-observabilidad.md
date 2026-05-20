# Capítulo 25: Observabilidad — Logging, Métricas y Tracing

> "No puedes arreglar lo que no puedes ver. No puedes mejorar lo que no mides."

## 25.1 Los Tres Pilares (y el Cuarto)

```
┌───────────────┐  ┌───────────────┐  ┌──────────────┐
│    Logging    │  │   Métricas    │  │    Tracing    │
│  (eventos)    │  │  (números)    │  │  (recorridos) │
└───────────────┘  └───────────────┘  └──────────────┘
         │                 │                  │
         └─────────────────┼──────────────────┘
                           │
                  ┌────────▼────────┐
                  │   Alerts        │  ← El cuarto pilar
                  │  (acción)       │
                  └─────────────────┘
```

| Pilar | Responde a | Ejemplo |
|-------|-----------|---------|
| **Logging** | ¿Qué pasó? | `ERROR: Payment failed for order 123` |
| **Métricas** | ¿Cuánto/cuántos? | `p95 latency = 230ms, error rate = 0.5%` |
| **Tracing** | ¿Dónde y por qué? | Request cruzó 5 servicios, 3er servicio tardó 2s |
| **Alerting** | ¿Qué debo hacer? | `p95 > 500ms → página al on-call` |

## 25.2 Logging Moderno

### Structured Logging
```
❌ MAL (no estructurado):
2024-01-15 10:30:00 Error processing order 12345 for user 678

✅ BIEN (JSON estructurado):
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "ERROR",
  "message": "Payment processing failed",
  "service": "payment-service",
  "traceId": "abc-123-def-456",
  "spanId": "span-789",
  "userId": "678",
  "orderId": "12345",
  "error": {
    "type": "PaymentGatewayTimeout",
    "gateway": "stripe",
    "duration_ms": 3200
  }
}
```

### Niveles de Log y su Uso

| Nivel | Uso en Producción |
|-------|-------------------|
| **ERROR** | Algo falló que requiere atención humana inmediata. |
| **WARN** | Algo inesperado pero el sistema se recuperó (retry exitoso, degradación). |
| **INFO** | Eventos de negocio significativos (pedido creado, pago completado, deploy). |
| **DEBUG** | Información útil para debugging. No en producción por defecto. |
| **TRACE** | Máximo detalle. Nunca en producción. |

### El Stack de Logging Moderno

```
App → stdout/stderr → Agente (Fluentd/Fluent Bit) → Centralización → Visualización
                                                         │
                                              Elasticsearch / Loki
                                                         │
                                                   Kibana / Grafana
```

**Principio**: Las aplicaciones no deben saber dónde terminan sus logs. Solo escriben a stdout/stderr. La infraestructura se encarga de capturar, enriquecer y enrutar.

## 25.3 Métricas

### Los Cuatro Tipos Esenciales

| Tipo | Descripción | Ejemplo |
|------|-------------|---------|
| **Counter** | Valor que solo incrementa | `http_requests_total`, `errors_total` |
| **Gauge** | Valor que sube y baja | `memory_usage_bytes`, `active_connections` |
| **Histogram** | Distribución en buckets | `request_duration_seconds` (p50, p95, p99) |
| **Summary** | Similar a histogram, cuantiles | `request_duration_quantiles` |

### Métricas que DEBES tener (USE + RED)

**USE Method** (para recursos: CPU, memoria, disco):
- **U**tilization: % de uso
- **S**aturation: Cola de trabajo pendiente
- **E**rrors: Tasa de errores

**RED Method** (para servicios):
- **R**ate: Requests por segundo
- **E**rrors: Tasa de errores
- **D**uration: Latencia (p50, p95, p99)

### SLOs, SLIs y SLAs

```
SLI (Service Level Indicator): La métrica que mides.
  Ej: "Porcentaje de requests con status 2xx"

SLO (Service Level Objective): El objetivo que cumples.
  Ej: "99.9% de requests en 30 días son 2xx"

SLA (Service Level Agreement): La promesa contractual.
  Ej: "Si la disponibilidad baja de 99.5%, devolvemos $$"
```

### Error Budget
```
Error Budget = 100% - SLO

Ej: Si SLO = 99.9% de disponibilidad mensual
Error Budget = 0.1% = 43 minutos de downtime al mes

Si gastas el error budget → freeze de features, solo fiabilidad.
```

## 25.4 Distributed Tracing

En microservicios, un request cruza múltiples servicios. El tracing muestra el camino completo.

```
Request: POST /orders

Trace: ───────────────────────────────────────────── (traceId: abc123)
  ├─ Span: API Gateway ────────── 5ms
  │    └─ Span: Auth Service ──── 15ms
  ├─ Span: Order Service ──────── 120ms
  │    ├─ Span: Validate ──────── 5ms
  │    ├─ Span: DB Insert ─────── 30ms
  │    └─ Span: Emit Event ────── 15ms (a Kafka)
  └─ Span: Notification Svc ───── 80ms
       └─ Span: Send Email ────── 75ms
```

### Propagación de Contexto

Cada servicio debe propagar headers de tracing:

```
HTTP Headers (W3C Trace Context):
traceparent: 00-abc123def456-789012-spanid-01
tracestate: vendor=opaquevalue
```

### Herramientas
- **Jaeger** (open source)
- **Zipkin** (open source)
- **AWS X-Ray**
- **Google Cloud Trace**
- **Datadog APM**, **New Relic**, **Honeycomb**
- **OpenTelemetry** (estándar para instrumentación, vendor-neutral)

### OpenTelemetry

> "El estándar abierto para observabilidad. Úsalo."

```
OpenTelemetry SDK → Colector → Backend (Jaeger, Datadog, etc.)

Un solo estándar de instrumentación para logs, métricas y trazas.
```

## 25.5 Alerting Efectivo

### Anti-patrones de Alerting
- **Alertar por todo** → fatiga de alertas → ignorar alertas.
- **Alertas sin acción**: Si la alarma suena y no hay nada que hacer, ¿para qué existe?
- **Umbrales estáticos** para tráfico variable.

### Buenas Prácticas
```
Reglas para alertas:
1. Toda alerta debe requerir una acción humana.
2. Prioriza síntomas sobre causas (p95 > 500ms, no "CPU > 80%").
3. Agrupa alertas relacionadas (no dispares 50 alertas por un mismo outage).
4. Documenta el runbook para cada alerta.
5. Alertas basadas en SLO burn rate, no en umbrales arbitrarios.
```

### On-Call y Escalación
```
Alerta → On-Call Primario (5 min) → On-Call Secundario (10 min) → Manager (15 min)
```

---

> **Reflexión del capítulo**: La observabilidad no es para debugging, es para entender. Un sistema sin observabilidad es un avión sin instrumentos: puedes volar, pero no sabes a qué altura, velocidad o si te estás quedando sin combustible. Invierte en observabilidad desde el día 1. El día que tengas un outage en producción, cada dólar invertido se paga solo.
