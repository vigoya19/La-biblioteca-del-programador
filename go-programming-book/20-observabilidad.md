# Capítulo 20: Observabilidad en Go — Tracing, Métricas y Logging Estructurado

> "No puedes arreglar lo que no puedes ver. En producción, la observabilidad no es opcional."

## 20.1 Los Tres Pilares en Go

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  LOGGING (log/slog)          → ¿Qué pasó?                   │
│  MÉTRICAS (prometheus)       → ¿Cuánto/cuántos?             │
│  TRACING (OpenTelemetry)     → ¿Por qué tardó tanto?        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

Go 1.21+ introdujo `log/slog`, reemplazando años de librerías third-party. Combinado con Prometheus y OpenTelemetry, tienes un stack de observabilidad completo sin dependencias pesadas.

---

## 20.2 Structured Logging con log/slog

```go
import "log/slog"

func main() {
    // JSON handler para producción (parseable por Grafana Loki, Elasticsearch, Datadog)
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
        // Añadir source file:line a cada log (cuesta perf, solo en debug)
        AddSource: false,
    }))
    slog.SetDefault(logger)

    // Uso básico
    slog.Info("servidor iniciado", "port", 8080, "env", "production")
    // {"time":"2024-01-15T10:30:00Z","level":"INFO","msg":"servidor iniciado","port":8080,"env":"production"}

    // Con atributos estructurados (evita string interpolation)
    slog.Warn("latencia elevada",
        "endpoint", "/api/orders",
        "p95_ms", 450,
        "threshold_ms", 200,
    )

    // Agrupar atributos (jerarquía en JSON)
    slog.Info("pedido creado",
        slog.Group("order",
            "id", "ord-123",
            "customer_id", "cust-456",
            "total", 99.99,
        ),
        slog.Group("performance",
            "db_query_ms", 12,
            "total_ms", 45,
        ),
    )
}
```

### Logger con Contexto — El Patrón Correcto

```go
// Pasar el logger por contexto, NO como variable global
type contextKey string
const loggerKey contextKey = "logger"

func WithLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func LoggerFromContext(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default()
}

// En handlers HTTP
func orderHandler(w http.ResponseWriter, r *http.Request) {
    logger := LoggerFromContext(r.Context()).With(
        "request_id", middleware.GetReqID(r.Context()),
    )

    logger.Info("procesando pedido")

    order, err := service.CreateOrder(r.Context(), req)
    if err != nil {
        logger.Error("fallo al crear pedido", "error", err)
        http.Error(w, err.Error(), 500)
        return
    }

    logger.Info("pedido creado", "order_id", order.ID)
}
```

### Niveles de Log en Producción

```go
// Configurar nivel según entorno
var logLevel slog.Level
switch os.Getenv("ENV") {
case "production":
    logLevel = slog.LevelWarn  // Solo warnings y errores
case "staging":
    logLevel = slog.LevelInfo
default:
    logLevel = slog.LevelDebug // Desarrollo: todo
}

logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: logLevel,
}))
```

---

## 20.3 Métricas con Prometheus

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

// ─── Definir métricas (registro global) ───

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total de requests HTTP",
        },
        []string{"method", "endpoint", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "Latencia de requests HTTP",
            Buckets: prometheus.DefBuckets, // .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10
        },
        []string{"method", "endpoint"},
    )

    activeRequests = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "http_requests_in_flight",
            Help: "Requests HTTP actualmente en proceso",
        },
    )

    ordersCreated = prometheus.NewCounter(
        prometheus.CounterOpts{
            Name: "orders_created_total",
            Help: "Total de pedidos creados",
        },
    )
)

func init() {
    // Registrar métricas
    prometheus.MustRegister(httpRequestsTotal)
    prometheus.MustRegister(httpRequestDuration)
    prometheus.MustRegister(activeRequests)
    prometheus.MustRegister(ordersCreated)
}
```

### Middleware de Métricas HTTP

```go
func prometheusMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        activeRequests.Inc()

        // Wrapper para capturar el status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next.ServeHTTP(wrapped, r)

        duration := time.Since(start).Seconds()
        activeRequests.Dec()

        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, strconv.Itoa(wrapped.statusCode)).Inc()
        httpRequestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

### Exponer Métricas

```go
func main() {
    // Endpoint de métricas para Prometheus
    http.Handle("/metrics", promhttp.Handler())

    // Tu servidor normal
    http.Handle("/api/", prometheusMiddleware(yourHandler))

    http.ListenAndServe(":8080", nil)
}

// Prometheus scrapea http://localhost:8080/metrics cada 15s
// Grafana visualiza las métricas
```

### Métricas de Negocio — Las que Importan

```go
// Las métricas técnicas (latencia, requests) son necesarias.
// Pero las métricas de NEGOCIO son las que el CEO entiende.

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    order, err := s.repo.Create(ctx, req)
    if err != nil {
        // Métrica de negocio: pedidos fallidos por razón
        orderFailures.WithLabelValues(err.Type()).Inc()
        return nil, err
    }

    ordersCreated.Inc()
    orderTotal.WithLabelValues(req.Currency).Add(req.Total)
    orderItemsPerOrder.Observe(float64(len(req.Items)))

    return order, nil
}
```

---

## 20.4 Distributed Tracing con OpenTelemetry

OpenTelemetry es el estándar de la CNCF para tracing, métricas y logging. En Go, el SDK te permite instrumentar tu aplicación una vez y enviar datos a cualquier backend (Jaeger, Datadog, Honeycomb, Grafana Tempo).

### Setup Básico

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/trace"
)

func initTracer() (*trace.TracerProvider, error) {
    // Exportar a un collector OTLP (Jaeger, Grafana Agent, etc.)
    exporter, err := otlptracegrpc.New(context.Background(),
        otlptracegrpc.WithEndpoint("localhost:4317"),
        otlptracegrpc.WithInsecure(), // Solo dev. En producción: TLS
    )
    if err != nil {
        return nil, err
    }

    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("order-service"),
            semconv.ServiceVersion("1.0.0"),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}
```

### Instrumentar Código

```go
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    // Crear un span para esta operación
    tracer := otel.Tracer("order-service")
    ctx, span := tracer.Start(ctx, "CreateOrder")
    defer span.End()

    // Añadir atributos al span
    span.SetAttributes(
        attribute.String("customer_id", req.CustomerID),
        attribute.Int("items_count", len(req.Items)),
    )

    // Llamar a BD (hereda el span padre automáticamente)
    order, err := s.repo.Create(ctx, req)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }

    span.SetAttributes(attribute.String("order_id", order.ID))
    span.SetStatus(codes.Ok, "pedido creado")
    return order, nil
}
```

### Middleware HTTP Automático

```go
import (
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

func main() {
    // otelhttp instrumenta automáticamente: crea span por request,
    // propaga W3C Trace Context headers, registra status code y duración
    handler := otelhttp.NewHandler(yourMux, "order-service-api")

    http.ListenAndServe(":8080", handler)
}
```

---

## 20.5 El Stack Completo en main()

```go
func main() {
    // 1. Inicializar tracer
    tp, err := initTracer()
    if err != nil {
        log.Fatal(err)
    }
    defer tp.Shutdown(context.Background())

    // 2. Configurar logger estructurado
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    slog.SetDefault(logger)

    // 3. Mux con métricas + tracing + logging
    mux := http.NewServeMux()
    mux.HandleFunc("POST /api/orders", createOrderHandler)

    // Encadenar middlewares: tracing → métricas → logging
    handler := prometheusMiddleware(
        otelhttp.NewHandler(mux, "order-service"),
    )

    // 4. Health check (sin métricas, sin tracing para no contaminar)
    mux.HandleFunc("/health", healthHandler)

    // 5. Métricas endpoint para Prometheus
    mux.Handle("/metrics", promhttp.Handler())

    // 6. Servidor con graceful shutdown
    server := &http.Server{
        Addr:         ":8080",
        Handler:      handler,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
    }

    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
        <-sigCh
        slog.Info("apagando servidor...")
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        server.Shutdown(ctx)
    }()

    slog.Info("servidor iniciado", "port", 8080)
    server.ListenAndServe()
}
```

---

> **Reflexión del capítulo**: La observabilidad en Go se construye con tres bibliotecas: `log/slog` (stdlib), `prometheus/client_golang`, y `go.opentelemetry.io/otel`. No necesitas agentes externos ni sidecars pesados. Instrumenta desde el día 1. El día que tengas un outage en producción, cada log, cada métrica y cada trace que tengas te ahorrará horas de debugging. Lo barato sale caro.

---

← [Capítulo anterior](19-http-client-web.md) | [Inicio](README.md)
