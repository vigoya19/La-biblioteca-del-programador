# Capítulo 19: HTTP Client, Templating y Web Frameworks en Go

> "La biblioteca estándar de Go cubre el 80% de las necesidades web. Conocer QUÉ usar y CUÁNDO usar un framework es lo que te separa del junior."

## 19.1 El Lado Cliente de net/http — Lo Que Todo el Mundo Olvida

El capítulo 16 cubrió el servidor HTTP. Pero en microservicios, tu servidor ES TAMBIÉN un cliente de otros servicios. El HTTP client de Go es poderoso pero está lleno de defaults peligrosos que TODO arquitecto debe conocer.

### El Default que Te Va a Morder

```go
// ❌ ESTO FUNCIONA EN DEV. EXPLOTA EN PRODUCCIÓN.
resp, err := http.Get("https://api.externa.com/datos")
// http.DefaultClient NO TIENE TIMEOUT.
// Si la API externa se cuelga, tu goroutine se bloquea PARA SIEMPRE.
// En 3 horas tienes 10,000 goroutines bloqueadas y OOM.
```

```go
// ✅ Cliente HTTP con timeouts y connection pooling correctos
var httpClient = &http.Client{
    Timeout: 30 * time.Second, // Timeout TOTAL del request
    Transport: &http.Transport{
        MaxIdleConns:        100,              // Conexiones idle totales
        MaxIdleConnsPerHost: 10,               // Por host (importante para microservicios)
        IdleConnTimeout:     90 * time.Second, // Cuánto mantener conexiones idle
        DisableCompression:  false,            // gzip por defecto
        TLSHandshakeTimeout: 10 * time.Second, // Timeout de TLS handshake
        ResponseHeaderTimeout: 5 * time.Second, // Esperar headers de respuesta
        ExpectContinueTimeout: 1 * time.Second,
    },
}

// Incluso mejor: reutilizar UNA instancia de cliente para toda la app.
// NUNCA crear un http.Client por request.
```

### Request con Context — Cancelación y Trazabilidad

```go
func fetchUserData(ctx context.Context, userID string) (*User, error) {
    // El contexto propaga: timeout, cancelación, tracing headers (si usas OpenTelemetry)
    req, err := http.NewRequestWithContext(ctx, "GET",
        "https://api.usuarios.com/v1/users/"+userID, nil)
    if err != nil {
        return nil, fmt.Errorf("crear request: %w", err)
    }

    // Headers estándar que TODO servicio debería enviar
    req.Header.Set("User-Agent", "mi-servicio/1.0")
    req.Header.Set("Accept", "application/json")
    req.Header.Set("X-Request-ID", getRequestID(ctx)) // Propagación de tracing

    resp, err := httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("ejecutar request: %w", err)
    }
    defer resp.Body.Close()

    // SIEMPRE leer y cerrar el body, incluso en errores
    // Si no lees el body, la conexión NO se reutiliza (connection leak)

    if resp.StatusCode >= 400 {
        body, _ := io.ReadAll(resp.Body)
        return nil, fmt.Errorf("API externa error %d: %s", resp.StatusCode, string(body))
    }

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, fmt.Errorf("decodificar respuesta: %w", err)
    }
    return &user, nil
}
```

### Retry con Backoff Exponencial

```go
func fetchWithRetry(ctx context.Context, url string, maxRetries int) (*http.Response, error) {
    var lastErr error

    for attempt := 0; attempt <= maxRetries; attempt++ {
        if attempt > 0 {
            // Backoff exponencial con jitter
            backoff := time.Duration(math.Pow(2, float64(attempt))) * time.Second
            jitter := time.Duration(rand.Int63n(int64(backoff / 2)))
            select {
            case <-time.After(backoff + jitter):
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }

        req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
        resp, err := httpClient.Do(req)
        if err == nil {
            if resp.StatusCode < 500 { // Solo reintentar errores de servidor
                return resp, nil
            }
            resp.Body.Close()
            lastErr = fmt.Errorf("status %d", resp.StatusCode)
        } else {
            lastErr = err
        }
    }

    return nil, fmt.Errorf("agotados %d reintentos: %w", maxRetries, lastErr)
}
```

---

## 19.2 Templating — html/template y text/template

Go tiene dos motores de template en su biblioteca estándar. No necesitas Handlebars, Pug, ni EJS.

### text/template — Para Todo lo Que No Es HTML

```go
// Plantilla de email, mensaje de Slack, query SQL, etc.
const emailTmpl = `
Hola {{.Name}},

Tu pedido #{{.OrderID}} ha sido {{.Status}}.
Total: ${{printf "%.2f" .Total}}

{{if .TrackingURL}}
Seguimiento: {{.TrackingURL}}
{{else}}
El número de seguimiento estará disponible pronto.
{{end}}

Gracias,
El equipo de {{.Company}}
`

type EmailData struct {
    Name        string
    OrderID     string
    Status      string
    Total       float64
    TrackingURL string
    Company     string
}

func generarEmail(data EmailData) (string, error) {
    tmpl, err := template.New("email").Parse(emailTmpl)
    if err != nil {
        return "", err
    }
    var buf bytes.Buffer
    if err := tmpl.Execute(&buf, data); err != nil {
        return "", err
    }
    return buf.String(), nil
}
```

### html/template — Protección XSS Automática

```go
// A DIFERENCIA de otros lenguajes, html/template escapa AUTOMÁTICAMENTE
// según el contexto (HTML, JavaScript, CSS, URL, atributo). No necesitas
// recordar escapar manualmente.

const pageTmpl = `
<!DOCTYPE html>
<html>
<head><title>{{.Title}}</title></head>
<body>
    <h1>{{.Title}}</h1>
    
    {{range .Items}}
    <div class="product">
        <h2>{{.Name}}</h2>
        <p>{{.Description}}</p>  <!-- Escapado automático: <script> se convierte en &lt;script&gt; -->
        <span class="price">${{printf "%.2f" .Price}}</span>
    </div>
    {{end}}

    {{if .User}}
        <p>Bienvenido, {{.User.Name}}</p>
    {{else}}
        <p><a href="/login">Iniciar sesión</a></p>
    {{end}}
</body>
</html>
`

// Templates embebidos con go:embed (Go 1.16+)
//go:embed templates/*.html
var templateFS embed.FS

func main() {
    // Cargar templates desde sistema de archivos embebido
    tmpl := template.Must(template.ParseFS(templateFS, "templates/*.html"))

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        data := PageData{
            Title: "Mi Tienda",
            Items: loadProducts(),
        }
        tmpl.ExecuteTemplate(w, "page.html", data)
    })
}
```

### Template Composition — Layouts sin Framework

```html
<!-- templates/base.html -->
{{define "base"}}
<!DOCTYPE html>
<html>
<head>
    <title>{{block "title" .}}Mi App{{end}}</title>
    <link rel="stylesheet" href="/static/style.css">
</head>
<body>
    <header>{{template "nav" .}}</header>
    <main>{{template "content" .}}</main>
    <footer>{{template "footer" .}}</footer>
</body>
</html>
{{end}}
```

```html
<!-- templates/products.html -->
{{template "base" .}}

{{define "title"}}Productos - Mi App{{end}}

{{define "content"}}
<h1>Nuestros Productos</h1>
<div class="products">
    {{range .Products}}
        {{template "product-card" .}}
    {{end}}
</div>
{{end}}

{{define "product-card"}}
<div class="card">
    <h2>{{.Name}}</h2>
    <p>${{printf "%.2f" .Price}}</p>
</div>
{{end}}
```

---

## 19.3 Web Frameworks en Go — Cuándo y Cuál Usar

La biblioteca estándar (`net/http`) maneja servidores web robustos. Pero en producción, frameworks y routers ofrecen:

- **Enrutamiento avanzado**: path params, grupos, middlewares.
- **Validación**: binding de requests a structs con validación.
- **Tooling**: hot reload, OpenAPI generation, debugging.

### Comparativa de Frameworks

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│  net/http (stdlib) → Minimalista, sin dependencias               │
│  Go 1.22+: soporta method routing y path params nativos.         │
│  Ideal para: microservicios simples, herramientas CLI, lambdas.  │
│                                                                   │
│  chi (go-chi/chi) → El más idiomático. Parece stdlib extendida.  │
│  Middlewares compatibles con net/http. Rápido, ligero.            │
│  Ideal para: APIs REST, microservicios, equipos que aman stdlib. │
│                                                                   │
│  Gin (gin-gonic/gin) → El más popular. Rápido, validación built-in│
│  No es compatible con net/http (context diferente).               │
│  Ideal para: APIs de alto rendimiento, equipos que valoran DX.   │
│                                                                   │
│  Echo (labstack/echo) → Similar a Gin, más minimalista.          │
│  Ideal para: APIs REST, proyectos que quieren Gin-like sin Gin.  │
│                                                                   │
│  Fiber (gofiber/fiber) → Express.js-style. Muy rápido (fasthttp). │
│  No compatible con net/http. Limitaciones con HTTP/2.            │
│  Ideal para: Migraciones desde Express.js, APIs de baja latencia.│
│                                                                   │
│  gorilla/mux → Legacy. Comunidad lo mantiene pero sin desarrollo │
│  activo desde 2022. Considerar migrar a chi.                     │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### chi — El Framework Idiomático

```go
import (
    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
)

func main() {
    r := chi.NewRouter()

    // Middlewares globales
    r.Use(middleware.Logger)        // Logging de requests
    r.Use(middleware.Recoverer)     // Panic recovery
    r.Use(middleware.Timeout(30 * time.Second))
    r.Use(middleware.RealIP)        // Respetar X-Forwarded-For
    r.Use(middleware.RequestID)     // UUID por request

    // Rutas públicas
    r.Get("/health", healthHandler)
    r.Post("/login", loginHandler)

    // Grupo protegido con autenticación
    r.Group(func(r chi.Router) {
        r.Use(authMiddleware)

        r.Route("/api/v1/orders", func(r chi.Router) {
            r.Get("/", listOrders)          // GET /api/v1/orders
            r.Post("/", createOrder)        // POST /api/v1/orders
            r.Get("/{orderID}", getOrder)   // GET /api/v1/orders/123
            r.Put("/{orderID}", updateOrder)
        })
    })

    http.ListenAndServe(":8080", r)
}
```

### Gin — El Más Popular

```go
import "github.com/gin-gonic/gin"

type CreateOrderRequest struct {
    CustomerID string   `json:"customer_id" binding:"required,uuid"`
    Items      []Item   `json:"items" binding:"required,min=1,dive"`
    CouponCode string   `json:"coupon_code"`
}

type Item struct {
    ProductID string  `json:"product_id" binding:"required"`
    Quantity  int     `json:"quantity" binding:"required,min=1,max=100"`
    Price     float64 `json:"price" binding:"required,min=0"`
}

func main() {
    r := gin.Default() // Incluye Logger y Recovery middlewares

    r.POST("/api/orders", func(c *gin.Context) {
        var req CreateOrderRequest
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(400, gin.H{"error": err.Error()})
            return
        }
        // req YA está validada. Llegar aquí = datos correctos.
        // ... crear pedido ...
        c.JSON(201, gin.H{"order_id": "new-id"})
    })

    r.Run(":8080")
}
```

---

## 19.4 Construyendo tu Propio "Mini-Framework"

No necesitas Gin ni chi para muchos proyectos. Aquí tienes una base sólida sobre stdlib:

```go
// server.go — Servidor HTTP idiomático con Go 1.22+
func main() {
    mux := http.NewServeMux()

    // Go 1.22+: path params nativos con {name}
    mux.HandleFunc("GET /api/v1/orders/{orderID}", getOrder)
    mux.HandleFunc("POST /api/v1/orders", createOrder)
    mux.HandleFunc("GET /health", healthCheck)

    // Encadenar middlewares manualmente
    handler := middlewareChain(mux,
        requestIDMiddleware,
        loggingMiddleware,
        recoveryMiddleware,
        corsMiddleware,
        timeoutMiddleware(30 * time.Second),
    )

    server := &http.Server{
        Addr:         ":8080",
        Handler:      handler,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // Graceful shutdown
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
        <-sigCh

        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        server.Shutdown(ctx)
    }()

    log.Println("Server starting on :8080")
    if err := server.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatal(err)
    }
    log.Println("Server stopped gracefully")
}

// Middleware chain helper
func middlewareChain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

// Reusable middlewares
func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = uuid.New().String()
        }
        w.Header().Set("X-Request-ID", id)
        ctx := context.WithValue(r.Context(), "request_id", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func timeoutMiddleware(timeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), timeout)
            defer cancel()
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

---

> **Reflexión del capítulo**: Go te da todo lo necesario para construir servicios web en su biblioteca estándar. La madurez como arquitecto Go está en saber cuándo mantenerte en stdlib (la mayoría de los casos) y cuándo adoptar un framework como chi o Gin (cuando la productividad del equipo lo justifica). No adoptes un framework porque sí. Adóptalo porque resuelve un problema concreto que la stdlib no resuelve bien.

---

← [Capítulo anterior](18-grpc.md) | [Inicio](README.md) | [Capítulo siguiente →](20-observabilidad.md)
