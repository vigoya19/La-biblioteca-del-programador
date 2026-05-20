# Capítulo 17: Ejercicios Prácticos

Este capitulo contiene ejercicios organizados por nivel de dificultad para practicar y consolidar los conceptos aprendidos a lo largo del libro. Cada ejercicio incluye una descripcion del problema, pistas y la solucion comentada.

---

## 17.1 Ejercicios Basicos

### Ejercicio 1: Calculadora CLI

Crea un programa de linea de comandos que reciba dos numeros y un operador (`+`, `-`, `*`, `/`) y muestre el resultado. Debe manejar division por cero y operadores invalidos.

```go
// solucion/calculadora/main.go
package main

import (
    "fmt"
    "os"
    "strconv"
)

func main() {
    if len(os.Args) != 4 {
        fmt.Println("Uso: calculadora <num1> <operador> <num2>")
        fmt.Println("Ejemplo: calculadora 10 + 5")
        os.Exit(1)
    }

    a, err := strconv.ParseFloat(os.Args[1], 64)
    if err != nil {
        fmt.Printf("Error: '%s' no es un numero valido\n", os.Args[1])
        os.Exit(1)
    }

    operador := os.Args[2]

    b, err := strconv.ParseFloat(os.Args[3], 64)
    if err != nil {
        fmt.Printf("Error: '%s' no es un numero valido\n", os.Args[3])
        os.Exit(1)
    }

    var resultado float64
    switch operador {
    case "+":
        resultado = a + b
    case "-":
        resultado = a - b
    case "*":
        resultado = a * b
    case "/":
        if b == 0 {
            fmt.Println("Error: division por cero")
            os.Exit(1)
        }
        resultado = a / b
    default:
        fmt.Printf("Error: operador '%s' no valido. Usa +, -, *, /\n", operador)
        os.Exit(1)
    }

    fmt.Printf("%.2f %s %.2f = %.2f\n", a, operador, b, resultado)
}
```

### Ejercicio 2: Contador de Palabras

Escribe una funcion que reciba un string y retorne un `map[string]int` con la frecuencia de cada palabra. Ignora mayusculas/minusculas y signos de puntuacion.

```go
// solucion/contador_palabras/contador.go
package main

import (
    "fmt"
    "strings"
    "unicode"
)

func ContarPalabras(texto string) map[string]int {
    frecuencia := make(map[string]int)

    // Limpiar y separar palabras
    palabras := strings.FieldsFunc(strings.ToLower(texto), func(r rune) bool {
        return !unicode.IsLetter(r) && !unicode.IsNumber(r)
    })

    for _, palabra := range palabras {
        frecuencia[palabra]++
    }

    return frecuencia
}

func main() {
    texto := "Go es genial. Go es rapido. Go, Go, Go!"
    resultado := ContarPalabras(texto)

    for palabra, freq := range resultado {
        fmt.Printf("%s: %d\n", palabra, freq)
    }
    // go: 5
    // es: 2
    // genial: 1
    // rapido: 1
}
```

### Ejercicio 3: Validador de Email

Implementa una funcion `ValidarEmail` que verifique:
- Contiene exactamente un `@`
- Tiene al menos un caracter antes y despues del `@`
- Tiene un dominio con al menos un punto (ej: `dominio.com`)
- No contiene espacios

```go
// solucion/validador_email/validador.go
package main

import (
    "fmt"
    "strings"
)

func ValidarEmail(email string) error {
    email = strings.TrimSpace(email)

    if email == "" {
        return fmt.Errorf("email no puede estar vacio")
    }

    if strings.Contains(email, " ") {
        return fmt.Errorf("email no puede contener espacios")
    }

    partes := strings.Split(email, "@")
    if len(partes) != 2 {
        return fmt.Errorf("email debe contener exactamente un @")
    }

    usuario := partes[0]
    dominio := partes[1]

    if usuario == "" {
        return fmt.Errorf("email debe tener un usuario antes del @")
    }

    if dominio == "" {
        return fmt.Errorf("email debe tener un dominio despues del @")
    }

    if !strings.Contains(dominio, ".") {
        return fmt.Errorf("dominio debe contener al menos un punto")
    }

    if strings.HasPrefix(dominio, ".") || strings.HasSuffix(dominio, ".") {
        return fmt.Errorf("dominio no puede empezar o terminar con punto")
    }

    return nil
}

func main() {
    emails := []string{
        "usuario@ejemplo.com",
        "invalido",
        "@sinusuario.com",
        "sin@",
        "usuario@dominio",
        "usuario @ejemplo.com",
        "",
    }

    for _, email := range emails {
        if err := ValidarEmail(email); err != nil {
            fmt.Printf("'%s': %v\n", email, err)
        } else {
            fmt.Printf("'%s': VALIDO\n", email)
        }
    }
}
```

---

## 17.2 Ejercicios Intermedios

### Ejercicio 4: Servidor de Tareas REST

Construye una API REST para gestionar una lista de tareas con los siguientes endpoints:

| Metodo | Ruta | Descripcion |
|--------|------|-------------|
| `GET` | `/tareas` | Listar todas las tareas |
| `POST` | `/tareas` | Crear una tarea |
| `GET` | `/tareas/{id}` | Obtener una tarea |
| `PUT` | `/tareas/{id}` | Actualizar una tarea |
| `DELETE` | `/tareas/{id}` | Eliminar una tarea |

Cada tarea debe tener: `id`, `titulo`, `descripcion`, `completada`, `fecha_creacion`.

```go
// solucion/api_tareas/main.go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "strconv"
    "sync"
    "time"
)

type Tarea struct {
    ID          int       `json:"id"`
    Titulo      string    `json:"titulo"`
    Descripcion string    `json:"descripcion"`
    Completada  bool      `json:"completada"`
    FechaCreacion time.Time `json:"fecha_creacion"`
}

type RepositorioTareas struct {
    mu     sync.RWMutex
    tareas map[int]*Tarea
    nextID int
}

func NuevoRepositorio() *RepositorioTareas {
    return &RepositorioTareas{
        tareas: make(map[int]*Tarea),
        nextID: 1,
    }
}

func (r *RepositorioTareas) Listar() []*Tarea {
    r.mu.RLock()
    defer r.mu.RUnlock()

    resultado := make([]*Tarea, 0, len(r.tareas))
    for _, t := range r.tareas {
        resultado = append(resultado, t)
    }
    return resultado
}

func (r *RepositorioTareas) Crear(titulo, desc string) *Tarea {
    r.mu.Lock()
    defer r.mu.Unlock()

    t := &Tarea{
        ID:          r.nextID,
        Titulo:      titulo,
        Descripcion: desc,
        Completada:  false,
        FechaCreacion: time.Now(),
    }
    r.tareas[r.nextID] = t
    r.nextID++
    return t
}

func (r *RepositorioTareas) Obtener(id int) (*Tarea, bool) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    t, ok := r.tareas[id]
    return t, ok
}

func (r *RepositorioTareas) Actualizar(id int, titulo, desc string, completada *bool) (*Tarea, bool) {
    r.mu.Lock()
    defer r.mu.Unlock()

    t, ok := r.tareas[id]
    if !ok {
        return nil, false
    }

    if titulo != "" {
        t.Titulo = titulo
    }
    if desc != "" {
        t.Descripcion = desc
    }
    if completada != nil {
        t.Completada = *completada
    }

    return t, true
}

func (r *RepositorioTareas) Eliminar(id int) bool {
    r.mu.Lock()
    defer r.mu.Unlock()

    _, ok := r.tareas[id]
    if ok {
        delete(r.tareas, id)
    }
    return ok
}

type TareaHandler struct {
    repo *RepositorioTareas
}

func (h *TareaHandler) Listar(w http.ResponseWriter, r *http.Request) {
    responderJSON(w, http.StatusOK, h.repo.Listar())
}

func (h *TareaHandler) Crear(w http.ResponseWriter, r *http.Request) {
    var entrada struct {
        Titulo      string `json:"titulo"`
        Descripcion string `json:"descripcion"`
    }

    if err := json.NewDecoder(r.Body).Decode(&entrada); err != nil {
        http.Error(w, "JSON invalido", http.StatusBadRequest)
        return
    }

    if entrada.Titulo == "" {
        http.Error(w, "titulo es requerido", http.StatusBadRequest)
        return
    }

    tarea := h.repo.Crear(entrada.Titulo, entrada.Descripcion)
    responderJSON(w, http.StatusCreated, tarea)
}

func (h *TareaHandler) Obtener(w http.ResponseWriter, r *http.Request) {
    id, _ := strconv.Atoi(r.PathValue("id"))
    t, ok := h.repo.Obtener(id)
    if !ok {
        http.Error(w, "tarea no encontrada", http.StatusNotFound)
        return
    }
    responderJSON(w, http.StatusOK, t)
}

func (h *TareaHandler) Actualizar(w http.ResponseWriter, r *http.Request) {
    id, _ := strconv.Atoi(r.PathValue("id"))

    var entrada struct {
        Titulo      string `json:"titulo"`
        Descripcion string `json:"descripcion"`
        Completada  *bool  `json:"completada"`
    }

    if err := json.NewDecoder(r.Body).Decode(&entrada); err != nil {
        http.Error(w, "JSON invalido", http.StatusBadRequest)
        return
    }

    t, ok := h.repo.Actualizar(id, entrada.Titulo, entrada.Descripcion, entrada.Completada)
    if !ok {
        http.Error(w, "tarea no encontrada", http.StatusNotFound)
        return
    }
    responderJSON(w, http.StatusOK, t)
}

func (h *TareaHandler) Eliminar(w http.ResponseWriter, r *http.Request) {
    id, _ := strconv.Atoi(r.PathValue("id"))
    if !h.repo.Eliminar(id) {
        http.Error(w, "tarea no encontrada", http.StatusNotFound)
        return
    }
    w.WriteHeader(http.StatusNoContent)
}

func responderJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if data != nil {
        json.NewEncoder(w).Encode(data)
    }
}

func main() {
    repo := NuevoRepositorio()
    h := &TareaHandler{repo: repo}

    mux := http.NewServeMux()
    mux.HandleFunc("GET /tareas", h.Listar)
    mux.HandleFunc("POST /tareas", h.Crear)
    mux.HandleFunc("GET /tareas/{id}", h.Obtener)
    mux.HandleFunc("PUT /tareas/{id}", h.Actualizar)
    mux.HandleFunc("DELETE /tareas/{id}", h.Eliminar)

    log.Println("API de tareas en :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Ejercicio 5: Cache con TTL

Implementa una cache generica con tiempo de expiracion (TTL) usando goroutines y channels.

```go
// solucion/cache_ttl/cache.go
package main

import (
    "fmt"
    "sync"
    "time"
)

type entrada[V any] struct {
    valor   V
    expira time.Time
}

type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]entrada[V]
    ttl   time.Duration
    done  chan struct{}
}

func NuevaCache[K comparable, V any](ttl time.Duration) *Cache[K, V] {
    c := &Cache[K, V]{
        items: make(map[K]entrada[V]),
        ttl:   ttl,
        done:  make(chan struct{}),
    }

    go c.limpiarExpirados()

    return c
}

func (c *Cache[K, V]) Guardar(clave K, valor V) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.items[clave] = entrada[V]{
        valor:   valor,
        expira: time.Now().Add(c.ttl),
    }
}

func (c *Cache[K, V]) Obtener(clave K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    entry, ok := c.items[clave]
    if !ok {
        var zero V
        return zero, false
    }

    if time.Now().After(entry.expira) {
        var zero V
        return zero, false
    }

    return entry.valor, true
}

func (c *Cache[K, V]) Eliminar(clave K) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, clave)
}

func (c *Cache[K, V]) Tamano() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return len(c.items)
}

func (c *Cache[K, V]) limpiarExpirados() {
    ticker := time.NewTicker(c.ttl / 2)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            c.mu.Lock()
            ahora := time.Now()
            for k, v := range c.items {
                if ahora.After(v.expira) {
                    delete(c.items, k)
                }
            }
            c.mu.Unlock()
        case <-c.done:
            return
        }
    }
}

func (c *Cache[K, V]) Cerrar() {
    close(c.done)
}

func main() {
    cache := NuevaCache[string, string](2 * time.Second)
    defer cache.Cerrar()

    cache.Guardar("usuario:1", "Andres")
    cache.Guardar("usuario:2", "Maria")

    if v, ok := cache.Obtener("usuario:1"); ok {
        fmt.Println("usuario:1 =", v) // Andres
    }

    fmt.Println("Esperando que expiren...")
    time.Sleep(3 * time.Second)

    if _, ok := cache.Obtener("usuario:1"); !ok {
        fmt.Println("usuario:1 expiro")
    }

    fmt.Printf("Tamaño cache: %d\n", cache.Tamano()) // 0
}
```

### Ejercicio 6: Worker Pool concurrente

Implementa un pool de workers que procese trabajos de un channel de entrada y envie resultados a un channel de salida.

```go
// solucion/worker_pool/pool.go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Trabajo struct {
    ID    int
    Datos string
}

type Resultado struct {
    TrabajoID int
    Output    string
    Error     error
}

type WorkerPool struct {
    numWorkers  int
    trabajos    chan Trabajo
    resultados  chan Resultado
    wg          sync.WaitGroup
}

func NuevoWorkerPool(numWorkers, bufferSize int) *WorkerPool {
    return &WorkerPool{
        numWorkers: numWorkers,
        trabajos:   make(chan Trabajo, bufferSize),
        resultados: make(chan Resultado, bufferSize),
    }
}

func (wp *WorkerPool) Iniciar(procesador func(Trabajo) Resultado) {
    for i := 0; i < wp.numWorkers; i++ {
        wp.wg.Add(1)
        go func(workerID int) {
            defer wp.wg.Done()
            for trabajo := range wp.trabajos {
                fmt.Printf("[Worker %d] Procesando trabajo %d\n", workerID, trabajo.ID)
                resultado := procesador(trabajo)
                wp.resultados <- resultado
            }
        }(i)
    }
}

func (wp *WorkerPool) EnviarTrabajo(t Trabajo) {
    wp.trabajos <- t
}

func (wp *WorkerPool) Resultados() <-chan Resultado {
    return wp.resultados
}

func (wp *WorkerPool) Esperar() {
    close(wp.trabajos)
    wp.wg.Wait()
    close(wp.resultados)
}

func main() {
    pool := NuevoWorkerPool(3, 10)

    procesador := func(t Trabajo) Resultado {
        // Simular procesamiento
        time.Sleep(200 * time.Millisecond)
        return Resultado{
            TrabajoID: t.ID,
            Output:    fmt.Sprintf("procesado: %s", t.Datos),
        }
    }

    pool.Iniciar(procesador)

    // Enviar trabajos en otra goroutine
    go func() {
        for i := 1; i <= 10; i++ {
            pool.EnviarTrabajo(Trabajo{
                ID:    i,
                Datos: fmt.Sprintf("datos-%d", i),
            })
        }
        pool.Esperar()
    }()

    // Recibir resultados
    for resultado := range pool.Resultados() {
        if resultado.Error != nil {
            fmt.Printf("Error en trabajo %d: %v\n", resultado.TrabajoID, resultado.Error)
        } else {
            fmt.Printf("Resultado %d: %s\n", resultado.TrabajoID, resultado.Output)
        }
    }
}
```

---

## 17.3 Ejercicios Avanzados

### Ejercicio 7: Chat en Tiempo Real con WebSockets

Construye un servidor de chat que permita a multiples clientes conectarse y enviar mensajes en tiempo real.

```go
// solucion/chat/main.go
package main

import (
    "fmt"
    "log"
    "net/http"
    "sync"

    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

type Cliente struct {
    conn *websocket.Conn
    send chan []byte
}

type Hub struct {
    clientes    map[*Cliente]bool
    broadcast   chan []byte
    registrar   chan *Cliente
    desregistrar chan *Cliente
    mu          sync.RWMutex
}

func NuevoHub() *Hub {
    return &Hub{
        clientes:    make(map[*Cliente]bool),
        broadcast:   make(chan []byte),
        registrar:   make(chan *Cliente),
        desregistrar: make(chan *Cliente),
    }
}

func (h *Hub) Ejecutar() {
    for {
        select {
        case cliente := <-h.registrar:
            h.mu.Lock()
            h.clientes[cliente] = true
            h.mu.Unlock()
            log.Printf("Cliente conectado. Total: %d", len(h.clientes))

        case cliente := <-h.desregistrar:
            h.mu.Lock()
            if _, ok := h.clientes[cliente]; ok {
                delete(h.clientes, cliente)
                close(cliente.send)
            }
            h.mu.Unlock()
            log.Printf("Cliente desconectado. Total: %d", len(h.clientes))

        case mensaje := <-h.broadcast:
            h.mu.RLock()
            for cliente := range h.clientes {
                select {
                case cliente.send <- mensaje:
                default:
                    close(cliente.send)
                    delete(h.clientes, cliente)
                }
            }
            h.mu.RUnlock()
        }
    }
}

func (c *Cliente) leer(hub *Hub) {
    defer func() {
        hub.desregistrar <- c
        c.conn.Close()
    }()

    for {
        _, mensaje, err := c.conn.ReadMessage()
        if err != nil {
            break
        }
        hub.broadcast <- mensaje
    }
}

func (c *Cliente) escribir() {
    defer c.conn.Close()

    for mensaje := range c.send {
        if err := c.conn.WriteMessage(websocket.TextMessage, mensaje); err != nil {
            break
        }
    }
}

func manejarWebSocket(hub *Hub, w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println("Error upgrade:", err)
        return
    }

    cliente := &Cliente{
        conn: conn,
        send: make(chan []byte, 256),
    }

    hub.registrar <- cliente

    go cliente.escribir()
    go cliente.leer(hub)
}

func main() {
    hub := NuevoHub()
    go hub.Ejecutar()

    http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
        manejarWebSocket(hub, w, r)
    })

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, htmlChat)
    })

    log.Println("Chat server en :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

const htmlChat = `<!DOCTYPE html>
<html>
<head><title>Chat Go</title></head>
<body>
    <h1>Chat en Tiempo Real</h1>
    <div id="mensajes" style="height:300px;overflow:auto;border:1px solid #ccc;padding:10px;"></div>
    <input id="entrada" type="text" placeholder="Escribe un mensaje..." style="width:80%;">
    <button onclick="enviar()">Enviar</button>
    <script>
        const ws = new WebSocket("ws://localhost:8080/ws");
        ws.onmessage = (e) => {
            const div = document.getElementById("mensajes");
            div.innerHTML += "<p>" + e.data + "</p>";
            div.scrollTop = div.scrollHeight;
        };
        function enviar() {
            const input = document.getElementById("entrada");
            ws.send(input.value);
            input.value = "";
        }
        document.getElementById("entrada").addEventListener("keypress", (e) => {
            if (e.key === "Enter") enviar();
        });
    </script>
</body>
</html>`
```

### Ejercicio 8: Rate Limiter con Token Bucket

Implementa un rate limiter usando el algoritmo Token Bucket.

```go
// solucion/rate_limiter/limiter.go
package main

import (
    "fmt"
    "net/http"
    "sync"
    "time"
)

type TokenBucket struct {
    capacidad   int
    tokens      int
    tasa        time.Duration // Tiempo entre tokens
    ultimaVez  time.Time
    mu          sync.Mutex
}

func NuevoTokenBucket(capacidad int, tasa time.Duration) *TokenBucket {
    return &TokenBucket{
        capacidad:  capacidad,
        tokens:     capacidad,
        tasa:       tasa,
        ultimaVez: time.Now(),
    }
}

func (tb *TokenBucket) Permitir() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    ahora := time.Now()
    transcurrido := ahora.Sub(tb.ultimaVez)
    tokensNuevos := int(transcurrido / tb.tasa)

    if tokensNuevos > 0 {
        tb.tokens += tokensNuevos
        if tb.tokens > tb.capacidad {
            tb.tokens = tb.capacidad
        }
        tb.ultimaVez = ahora
    }

    if tb.tokens > 0 {
        tb.tokens--
        return true
    }

    return false
}

// Rate limiter por IP
type RateLimiter struct {
    buckets map[string]*TokenBucket
    mu      sync.Mutex
    limite  int
    ventana time.Duration
}

func NuevoRateLimiter(limite int, ventana time.Duration) *RateLimiter {
    rl := &RateLimiter{
        buckets: make(map[string]*TokenBucket),
        limite:  limite,
        ventana: ventana,
    }

    // Limpiar buckets viejos periodicamente
    go func() {
        for {
            time.Sleep(ventana)
            rl.mu.Lock()
            for ip, bucket := range rl.buckets {
                bucket.mu.Lock()
                if bucket.tokens == bucket.capacidad {
                    delete(rl.buckets, ip)
                }
                bucket.mu.Unlock()
            }
            rl.mu.Unlock()
        }
    }()

    return rl
}

func (rl *RateLimiter) Permitir(ip string) bool {
    rl.mu.Lock()
    bucket, ok := rl.buckets[ip]
    if !ok {
        bucket = NuevoTokenBucket(rl.limite, rl.ventana/time.Duration(rl.limite))
        rl.buckets[ip] = bucket
    }
    rl.mu.Unlock()

    return bucket.Permitir()
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr

        if !rl.Permitir(ip) {
            w.Header().Set("Retry-After", "1")
            http.Error(w, "demasiadas peticiones", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}

func main() {
    limiter := NuevoRateLimiter(10, time.Second)

    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "OK")
    })

    handler := limiter.Middleware(mux)

    fmt.Println("Servidor con rate limiter en :8080")
    http.ListenAndServe(":8080", handler)
}
```

---

## 17.4 Proyectos Integradores

### Proyecto 1: Microservicio de Usuarios

Construye un microservicio completo con arquitectura hexagonal que incluya:

- Registro y autenticacion de usuarios (JWT)
- CRUD de perfiles
- Persistencia en PostgreSQL
- Tests unitarios y de integracion
- Migraciones de base de datos
- Middleware de logging, recovery y rate limiting
- Configuracion via variables de entorno
- Graceful shutdown
- Dockerfile para despliegue

### Proyecto 2: Acortador de URLs

Construye un servicio para acortar URLs con:

- API REST para crear y redirigir URLs cortas
- Cache en Redis para URLs frecuentes
- Estadisticas de visitas
- Panel de administracion basico
- Soporte para URLs personalizadas
- Limpieza automatica de URLs expiradas

### Proyecto 3: Sistema de Procesamiento de Eventos

Construye un sistema que procese eventos en tiempo real:

- Productor que genera eventos (simulando sensores o usuarios)
- Message broker con channels
- Workers que procesan eventos en paralelo
- Agregacion de metricas (promedios, maximos, contadores)
- API para consultar metricas en tiempo real
- Almacenamiento historico en base de datos

---

## Resumen del Capítulo

- Los ejercicios basicos refuerzan sintaxis, slices, maps y manejo de errores.
- Los ejercicios intermedios introducen APIs REST, concurrencia y estructuras genericas.
- Los ejercicios avanzados combinan WebSockets, rate limiting y patrones de diseno.
- Los proyectos integradores consolidan todos los conceptos del libro en aplicaciones completas.
- La practica constante es la clave para dominar Go. Intenta resolver cada ejercicio antes de mirar la solucion.

---

Este es el final del libro. Has recorrido un camino completo desde los fundamentos de Go hasta temas avanzados como concurrencia, arquitectura hexagonal y desarrollo web. El siguiente paso es construir tus propios proyectos y contribuir a la comunidad Go. ¡Feliz programacion!
