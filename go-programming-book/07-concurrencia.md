# Capítulo 7: Concurrencia

La concurrencia es la caracteristica mas distintiva de Go. No es un añadido ni una libreria: esta integrada en el lenguaje mediante **goroutines** y **channels**.

> "No te comuniques compartiendo memoria; comparte memoria comunicandote." — Proverbio Go

Go usa el modelo **CSP** (Communicating Sequential Processes), donde las goroutines se comunican a traves de channels en lugar de usar locks explicitos.

---

## 7.1 Goroutines

Una goroutine es una funcion que se ejecuta concurrentemente. Es ligera: puedes lanzar miles sin problema.

```go
package main

import (
    "fmt"
    "time"
)

func saludar(nombre string) {
    fmt.Printf("Hola, %s!\n", nombre)
}

func tareaLenta(id int) {
    time.Sleep(500 * time.Millisecond)
    fmt.Printf("Tarea %d completada\n", id)
}

func main() {
    // Lanzar una goroutine con la palabra clave go
    go saludar("Andres")

    // Lanzar varias goroutines
    for i := 1; i <= 5; i++ {
        go tareaLenta(i)
    }

    // Las goroutines se ejecutan concurrentemente.
    // Si main termina, todas las goroutines mueren.
    // Necesitamos esperar para ver los resultados.
    time.Sleep(1 * time.Second)
    fmt.Println("Main terminado")
}
```

### Caracteristicas de las goroutines

- **Ligeras**: una goroutine ocupa ~2KB de stack inicial (un hilo del SO ocupa ~1MB).
- **Stack creciente**: el stack crece y se encoge segun necesidad.
- **Multiplexadas**: multiples goroutines se ejecutan sobre pocos hilos del SO gracias al runtime de Go.
- **No son hilos**: el scheduler de Go las gestiona, no el SO directamente.

```go
// Puedes lanzar miles de goroutines sin problema
func main() {
    contador := 0
    for i := 0; i < 100000; i++ {
        go func() {
            contador++ // CUIDADO: data race (lo veremos mas adelante)
        }()
    }
    // Esto tiene un data race, pero ilustra la ligereza de goroutines
}
```

---

## 7.2 sync.WaitGroup

Para esperar a que un grupo de goroutines termine, usa `sync.WaitGroup`:

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func trabajador(id int, wg *sync.WaitGroup) {
    defer wg.Done() // Marcar como terminado al salir

    fmt.Printf("Trabajador %d comenzando\n", id)
    time.Sleep(time.Duration(id) * 100 * time.Millisecond)
    fmt.Printf("Trabajador %d terminado\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1) // Incrementar contador antes de lanzar
        go trabajador(i, &wg)
    }

    wg.Wait() // Bloquear hasta que el contador llegue a 0
    fmt.Println("Todos los trabajadores terminaron")
}
```

> **Regla de oro**: siempre llama `wg.Add(1)` antes de lanzar la goroutine, no dentro de ella. Si lo haces dentro, puede que `Wait()` termine antes de que `Add()` se ejecute.

---

## 7.3 Channels

Los channels son el mecanismo para que las goroutines se comuniquen y sincronicen:

```go
package main

import "fmt"

func main() {
    // Crear un channel (unbuffered por defecto)
    ch := make(chan int)

    // Enviar y recibir requiere dos goroutines
    go func() {
        ch <- 42 // Enviar: bloquea hasta que alguien reciba
    }()

    valor := <-ch // Recibir: bloquea hasta que alguien envie
    fmt.Println("Recibido:", valor)
}
```

### Channels unbuffered vs buffered

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Unbuffered: sincrono, el envio bloquea hasta que se recibe
    ch1 := make(chan int)

    go func() {
        fmt.Println("Enviando en unbuffered...")
        ch1 <- 1 // Bloquea hasta que main reciba
        fmt.Println("Enviado!")
    }()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Recibido:", <-ch1)

    // Buffered: asincrono hasta llenar el buffer
    ch2 := make(chan string, 3) // Buffer de 3

    ch2 <- "uno"   // No bloquea
    ch2 <- "dos"   // No bloquea
    ch2 <- "tres"  // No bloquea (buffer lleno)
    // ch2 <- "cuatro" // Bloquearia! Buffer lleno, nadie recibe

    fmt.Println(<-ch2) // uno
    fmt.Println(<-ch2) // dos
    fmt.Println(<-ch2) // tres
}
```

### Cerrar channels

```go
package main

import "fmt"

func productor(ch chan<- int) {
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch) // Cerrar el channel cuando terminamos
}

func main() {
    ch := make(chan int)
    go productor(ch)

    // Leer hasta que el channel se cierre
    for valor := range ch {
        fmt.Println("Recibido:", valor)
    }
    // Cuando ch se cierra, range termina automaticamente

    // Verificar si un channel esta cerrado
    valor, ok := <-ch
    if !ok {
        fmt.Println("Channel cerrado, valor:", valor) // zero value
    }
}
```

### Canales unidireccionales

```go
package main

import "fmt"

// Solo enviar
func enviar(ch chan<- int, valor int) {
    ch <- valor
    // <-ch // Error de compilacion: no se puede recibir
}

// Solo recibir
func recibir(ch <-chan int) int {
    return <-ch
    // ch <- 5 // Error de compilacion: no se puede enviar
}

func main() {
    ch := make(chan int, 1)
    enviar(ch, 42)
    fmt.Println(recibir(ch)) // 42
    // Go convierte bidireccional a unidireccional automaticamente
}
```

---

## 7.4 Select

`select` permite esperar en multiples operaciones de channel simultaneamente. Es como un switch pero para channels:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "uno"
    }()

    go func() {
        time.Sleep(500 * time.Millisecond)
        ch2 <- "dos"
    }()

    // select espera al primer channel listo
    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-ch1:
            fmt.Println("Recibido de ch1:", msg1)
        case msg2 := <-ch2:
            fmt.Println("Recibido de ch2:", msg2)
        }
    }
}
```

### Timeout con select

```go
func esperarConTimeout(ch <-chan string) {
    select {
    case msg := <-ch:
        fmt.Println("Recibido:", msg)
    case <-time.After(2 * time.Second):
        fmt.Println("Timeout: no se recibio nada en 2 segundos")
    }
}
```

### Ticker: ejecutar periodicamente

```go
func periodicamente() {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()

    hecho := make(chan bool)

    go func() {
        time.Sleep(2 * time.Second)
        hecho <- true
    }()

    for {
        select {
        case <-hecho:
            fmt.Println("Terminado!")
            return
        case t := <-ticker.C:
            fmt.Println("Tick en", t.Format("15:04:05.000"))
        }
    }
}
```

### Select con default (no bloqueante)

```go
func intentarEnviar(ch chan<- string, msg string) {
    select {
    case ch <- msg:
        fmt.Println("Enviado:", msg)
    default:
        fmt.Println("Channel lleno, descartando:", msg)
    }
}

func main() {
    ch := make(chan string, 2)
    ch <- "a"
    ch <- "b"
    intentarEnviar(ch, "c") // Channel lleno, descartando: c
}
```

> **Cuidado**: un `select` con `default` en un bucle infinito puede consumir CPU. Si necesitas esperar, no uses `default`.

---

## 7.5 Patrones de Concurrencia

### Fan-out / Fan-in

```go
package main

import (
    "fmt"
    "sync"
)

// Fan-out: distribuir trabajo entre multiples goroutines
func fanOut(trabajos []int, numTrabajadores int) <-chan int {
    ch := make(chan int)
    var wg sync.WaitGroup

    for i := 0; i < numTrabajadores; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for _, t := range trabajos {
                ch <- t * 2 // Procesar trabajo
            }
        }(i)
    }

    // Fan-in: reunir resultados
    go func() {
        wg.Wait()
        close(ch)
    }()

    return ch
}

// NOTA: este ejemplo es simplificado. En la practica se usa un channel
// de entrada para los trabajos y otro de salida para los resultados.
```

### Pipeline

```go
package main

import "fmt"

// Etapa 1: genera numeros
func generar(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Etapa 2: eleva al cuadrado
func cuadrado(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Etapa 3: filtra pares
func filtrarPares(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
        close(out)
    }()
    return out
}

func main() {
    // Pipeline: generar -> cuadrado -> filtrarPares
    entrada := generar(1, 2, 3, 4, 5, 6, 7)
    cuadrados := cuadrado(entrada)
    pares := filtrarPares(cuadrados)

    for resultado := range pares {
        fmt.Println(resultado) // 4, 16, 36
    }
}
```

### Worker Pool

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func workerPool(numTrabajadores int, trabajos <-chan int, resultados chan<- int) {
    var wg sync.WaitGroup

    for i := 0; i < numTrabajadores; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for trabajo := range trabajos {
                fmt.Printf("Trabajador %d procesando %d\n", id, trabajo)
                time.Sleep(100 * time.Millisecond) // Simular trabajo
                resultados <- trabajo * 2
            }
        }(i)
    }

    go func() {
        wg.Wait()
        close(resultados)
    }()
}

func main() {
    trabajos := make(chan int, 10)
    resultados := make(chan int, 10)

    // Lanzar 3 workers
    go workerPool(3, trabajos, resultados)

    // Enviar 10 trabajos
    go func() {
        for i := 1; i <= 10; i++ {
            trabajos <- i
        }
        close(trabajos)
    }()

    // Recibir resultados
    for r := range resultados {
        fmt.Println("Resultado:", r)
    }
}
```

---

## 7.6 Sincronizacion con Mutex

A veces los channels no son la mejor solucion. Cuando varias goroutines comparten datos, necesitas proteger el acceso:

```go
package main

import (
    "fmt"
    "sync"
)

type ContadorSeguro struct {
    mu    sync.Mutex
    valor int
}

func (c *ContadorSeguro) Incrementar() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.valor++
}

func (c *ContadorSeguro) Valor() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.valor
}

func main() {
    var wg sync.WaitGroup
    contador := &ContadorSeguro{}

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            contador.Incrementar()
        }()
    }

    wg.Wait()
    fmt.Println("Contador:", contador.Valor()) // 1000
}
```

### sync.RWMutex

Permite multiples lectores simultaneos, pero solo un escritor:

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Obtener(clave string) (string, bool) {
    c.mu.RLock() // Multiples lectores pueden entrar
    defer c.mu.RUnlock()
    valor, ok := c.data[clave]
    return valor, ok
}

func (c *Cache) Guardar(clave, valor string) {
    c.mu.Lock() // Solo un escritor a la vez
    defer c.mu.Unlock()
    c.data[clave] = valor
}
```

### sync.Once

Garantiza que una funcion se ejecute exactamente una vez:

```go
var (
    once     sync.Once
    instancia *Singleton
)

type Singleton struct {
    Valor string
}

func ObtenerInstancia() *Singleton {
    once.Do(func() {
        instancia = &Singleton{Valor: "inicializado"}
        fmt.Println("Singleton inicializado")
    })
    return instancia
}
```

### sync/atomic

Para operaciones atomicas simples en tipos basicos (mas rapido que Mutex):

```go
import "sync/atomic"

var contador int64

// Operaciones atomicas
atomic.AddInt64(&contador, 1)        // Incrementar
atomic.StoreInt64(&contador, 42)     // Guardar
valor := atomic.LoadInt64(&contador)  // Leer
cambiado := atomic.CompareAndSwapInt64(&contador, 42, 100) // CAS
```

---

## 7.7 Context

El paquete `context` permite propagar cancelacion, deadlines y valores a traves de llamadas:

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func operacion(ctx context.Context, id int) error {
    select {
    case <-time.After(time.Duration(id) * 200 * time.Millisecond):
        fmt.Printf("Operacion %d completada\n", id)
        return nil
    case <-ctx.Done():
        fmt.Printf("Operacion %d cancelada: %v\n", id, ctx.Err())
        return ctx.Err()
    }
}

func main() {
    // Context con timeout
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()

    for i := 1; i <= 5; i++ {
        go operacion(ctx, i)
    }

    time.Sleep(1 * time.Second)
}
```

### Tipos de context

```go
// Context raiz (nunca se cancela, no tiene deadline)
ctx := context.Background()

// Context para peticiones entrantes (como Background pero para servidores)
ctx := context.TODO()

// Context con cancelacion manual
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // Siempre llama a cancel para liberar recursos

// Context con deadline absoluto
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// Context con timeout relativo
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

// Context con valores (usar con moderacion)
type clave string // Evita colisiones usando tipo propio
ctx = context.WithValue(ctx, clave("usuarioID"), 42)
usuarioID := ctx.Value(clave("usuarioID")).(int)
```

### Ejemplo practico: servidor HTTP con cancelacion

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // Simular procesamiento largo respetando cancelacion
    resultado, err := procesar(ctx)
    if err != nil {
        http.Error(w, "Request cancelado", http.StatusRequestTimeout)
        return
    }

    fmt.Fprintf(w, "Resultado: %s", resultado)
}

func procesar(ctx context.Context) (string, error) {
    select {
    case <-time.After(3 * time.Second):
        return "procesado con exito", nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

> **Buena practica**: `context` debe ser el primer parametro de toda funcion que lo use. No lo guardes en structs; pasalo explicitamente.

---

## 7.8 errgroup — Paralelismo con Manejo de Errores

`golang.org/x/sync/errgroup` es la herramienta que necesitas cuando lanzas N goroutines y cualquiera puede fallar. Sin errgroup, manejar errores en goroutines requiere channels manuales propensos a bugs.

### El Problema que Resuelve

```go
// ❌ SIN errgroup: propenso a errores y verboso
func procesarItems(items []Item) error {
    var wg sync.WaitGroup
    errCh := make(chan error, len(items))

    for _, item := range items {
        wg.Add(1)
        go func(item Item) {
            defer wg.Done()
            if err := procesar(item); err != nil {
                errCh <- err
            }
        }(item)
    }

    wg.Wait()
    close(errCh)

    for err := range errCh {
        return err // Solo devuelve el PRIMER error, pierde los demás
    }
    return nil
}
```

```go
// ✅ CON errgroup: limpio, seguro, con cancelación automática
import "golang.org/x/sync/errgroup"

func procesarItems(items []Item) error {
    g := new(errgroup.Group)

    for _, item := range items {
        item := item // Capturar variable del loop (Go <1.22)
        g.Go(func() error {
            return procesar(item)
        })
    }

    return g.Wait() // Espera todas. Si alguna falla, devuelve el primer error
}
```

### errgroup con Contexto — Cancelación en Cascada

```go
// Si una goroutine falla, el contexto se cancela.
// Las demás goroutines deben detectar la cancelación y detenerse.
func fetchMultiSource(ctx context.Context, urls []string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)
    resultados := make([]string, len(urls))

    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            // Crear request CON el contexto del grupo
            req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
            if err != nil {
                return err
            }
            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                return err // Cancela las demás goroutines
            }
            defer resp.Body.Close()

            body, err := io.ReadAll(resp.Body)
            if err != nil {
                return err
            }
            // ¿Contexto cancelado mientras leíamos? No guardar resultado
            if ctx.Err() != nil {
                return ctx.Err()
            }
            resultados[i] = string(body)
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, fmt.Errorf("fetch multi-source: %w", err)
    }
    return resultados, nil
}
```

---

## 7.9 Patrón Semáforo — Paralelismo Limitado

Lanzar 10,000 goroutines simultáneas para llamar a una API externa es una receta para el desastre: rate limiting, connection pool exhaustion, memory spikes. El semáforo limita cuántas goroutines se ejecutan en paralelo.

### Implementación con Channel Buffered

```go
// Un channel buffered es un semáforo natural en Go
// Capacidad = número máximo de operaciones concurrentes
func scrapeURLs(urls []string, maxConcurrent int) []string {
    sem := make(chan struct{}, maxConcurrent) // Semáforo
    var wg sync.WaitGroup
    resultados := make([]string, 0, len(urls))
    var mu sync.Mutex

    for _, url := range urls {
        wg.Add(1)
        go func(url string) {
            defer wg.Done()
            sem <- struct{}{}        // Adquirir slot (bloquea si lleno)
            defer func() { <-sem }() // Liberar slot

            data := fetchURL(url)
            mu.Lock()
            resultados = append(resultados, data)
            mu.Unlock()
        }(url)
    }
    wg.Wait()
    return resultados
}
```

### Semáforo con Peso (Weighted) — Concurrencia por Unidad de Trabajo

```go
import "golang.org/x/sync/semaphore"

// Semáforo ponderado: cada operación consume N unidades
// No es lo mismo procesar un archivo de 1KB que uno de 1GB
func procesarArchivosConLimite(archivos []Archivo) error {
    // Máximo 100MB de procesamiento concurrente
    sem := semaphore.NewWeighted(100 * 1024 * 1024)
    ctx := context.Background()

    for _, arch := range archivos {
        peso := arch.Tamaño
        if err := sem.Acquire(ctx, peso); err != nil {
            return err
        }

        go func(a Archivo) {
            defer sem.Release(a.Tamaño)
            procesarArchivo(a)
        }(arch)
    }

    // Esperar a que todos los archivos terminen (adquiriendo todo)
    return sem.Acquire(ctx, 100*1024*1024)
}
```

---

## 7.10 Done Channel — Señal de Terminación

El patrón más simple y poderoso para propagar cancelación sin context:

```go
// Done channel: el productor cierra el channel para señalar "terminé"
// TODOS los consumidores reciben la señal inmediatamente (close broadcast)
func buscarEnMultiplesFuentes(termino string) <-chan string {
    resultados := make(chan string)
    done := make(chan struct{}) // Señal de terminación

    fuentes := []string{"google", "bing", "duckduckgo"}
    for _, fuente := range fuentes {
        go func(f string) {
            select {
            case <-done:
                return // Alguien ya encontró el resultado
            default:
                resultado, err := buscar(termino, f)
                if err == nil {
                    select {
                    case resultados <- resultado:
                        close(done) // ¡Señal broadcast! Todos los demás se detienen
                    case <-done:
                        // El resultado llegó tarde, otro ya encontró
                    }
                }
            }
        }(fuente)
    }
    return resultados
}

func main() {
    resultados := buscarEnMultiplesFuentes("golang concurrencia")
    primero := <-resultados
    fmt.Println("Primer resultado:", primero)
}
```

---

## 7.11 Pipeline con Cancelación — Fin a Fin

Los pipelines del capítulo 7.6 funcionan con channels. Pero no manejan cancelación. Aquí está la versión producción:

```go
// Pipeline con cancelación via context
// Si el consumidor cancela, toda la cadena se limpia

func generar(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return // Consumidor canceló, dejar de generar
            }
        }
    }()
    return out
}

func cuadrado(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func filtrarPares(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n%2 == 0 {
                select {
                case out <- n:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}

func main() {
    // El consumidor puede cancelar en cualquier momento
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    naturales := generar(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    cuadrados := cuadrado(ctx, naturales)
    filtrados := filtrarPares(ctx, cuadrados)

    for n := range filtrados {
        fmt.Println(n)
        if n > 20 {
            cancel() // Cancelar pipeline completo
            break
        }
    }
    // Todas las goroutines del pipeline terminan limpiamente
}
```

---

## 7.12 Goroutine Leaks — El Asesino Silencioso

Una goroutine bloqueada para siempre en un channel es una fuga de memoria. No crashea. No se ve en tests normales. Pero en producción, después de 30 días, tu servicio usa 8 GB de RAM y nadie sabe por qué.

### Fuga #1: Channel sin Receptor

```go
// ❌ FUGA: La goroutine se bloquea para siempre
func fugaIngenuo() {
    ch := make(chan int)
    go func() {
        ch <- 42 // Nadie lee este channel NUNCA. Goroutine bloqueada para siempre.
        fmt.Println("Nunca se imprime")
    }()
    // main termina... pero la goroutine NO. Sigue viva, bloqueada, consumiendo memoria.
}
```

```go
// ✅ Solución: Context con timeout o done channel
func sinFuga(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case ch <- 42:
        case <-ctx.Done():
            return // Contexto cancelado, no enviar
        }
    }()
}
```

### Fuga #2: Channel sin Emisor

```go
// ❌ FUGA: El receptor espera para siempre
func fugaEsperando(ch <-chan int) {
    go func() {
        // Si ch nunca recibe datos y nunca se cierra...
        val := <-ch // Bloqueado para siempre
        fmt.Println(val)
    }()
    // Nadie escribe a ch. Goroutine fugada.
}
```

```go
// ✅ Solución: Usar select con timeout o context
func sinFuga(ctx context.Context, ch <-chan int) {
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-ctx.Done():
            return
        }
    }()
}
```

### Fuga #3: Loop con Channel sin Close

```go
// ❌ FUGA: for-range sobre channel que nunca se cierra
func fugaRange(ch <-chan int) {
    go func() {
        for val := range ch { // Si ch nunca se cierra...
            fmt.Println(val)
        }
        // Este código NUNCA se ejecuta. Goroutine fugada.
    }()
}
```

```go
// ✅ Regla de oro: El PRODUCTOR cierra el channel.
//    El CONSUMIDOR usa context para protegerse.
func productor(ch chan<- int, done <-chan struct{}) {
    defer close(ch) // Productor siempre cierra su channel
    for i := 0; i < 100; i++ {
        select {
        case ch <- i:
        case <-done:
            return
        }
    }
}
```

### Cómo Detectar Goroutine Leaks

```go
// Añadir esto a main() o TestMain() en desarrollo
func init() {
    go func() {
        for {
            time.Sleep(10 * time.Second)
            n := runtime.NumGoroutine()
            if n > 100 { // Umbral de alarma
                buf := make([]byte, 1<<16)
                runtime.Stack(buf, true)
                log.Printf("⚠️ %d goroutines activas:\n%s", n, buf)
            }
        }
    }()
}

// En producción: pprof
// go tool pprof http://localhost:6060/debug/pprof/goroutine
```

---

## 7.13 Patrones Avanzados de Concurrencia

### Patrón Or-Done — Multiplexar Señales de Terminación

Combina múltiples señales de terminación en una sola. "Termina si el contexto se cancela O si el done channel se cierra."

```go
func orDone(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case <-done:
                return
            case val, ok := <-in:
                if !ok {
                    return // Channel de entrada cerrado
                }
                select {
                case out <- val:
                case <-done:
                    return
                }
            }
        }
    }()
    return out
}

// Versión multi-done: devuelve un channel que se cierra
// cuando CUALQUIERA de los done channels se cierra.
func or(channels ...<-chan struct{}) <-chan struct{} {
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan struct{})
    go func() {
        defer close(orDone)
        switch len(channels) {
        case 2:
            select {
            case <-channels[0]:
            case <-channels[1]:
            }
        default:
            select {
            case <-channels[0]:
            case <-channels[1]:
            case <-channels[2]:
            case <-or(append(channels[3:], orDone)...):
            }
        }
    }()
    return orDone
}
```

### Patrón Tee — Bifurcar un Channel

Envía los mismos datos a dos channels independientes.

```go
func tee(done <-chan struct{}, in <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    go func() {
        defer close(out1)
        defer close(out2)
        for val := range orDone(done, in) {
            var out1, out2 = out1, out2 // Shadow para no bloquear
            for i := 0; i < 2; i++ {
                select {
                case out1 <- val:
                    out1 = nil // Ya entregado, no intentar de nuevo
                case out2 <- val:
                    out2 = nil
                }
            }
        }
    }()
    return out1, out2
}
```

### Patrón Bridge — Aplanar un Channel de Channels

Convierte `<-chan <-chan int` en `<-chan int`.

```go
func bridge(done <-chan struct{}, chanStream <-chan <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            var stream <-chan int
            select {
            case maybeStream, ok := <-chanStream:
                if !ok {
                    return
                }
                stream = maybeStream
            case <-done:
                return
            }
            for val := range orDone(done, stream) {
                select {
                case out <- val:
                case <-done:
                    return
                }
            }
        }
    }()
    return out
}
```

### Patrón Rate Limiter con Buffered Channel

```go
// Token bucket implementado con un channel y un ticker
type RateLimiter struct {
    tokens chan struct{}
}

func NewRateLimiter(rate int, burst int) *RateLimiter {
    rl := &RateLimiter{
        tokens: make(chan struct{}, burst),
    }
    // Llenar inicialmente con burst tokens
    for i := 0; i < burst; i++ {
        rl.tokens <- struct{}{}
    }
    // Reponer tokens a la tasa especificada
    go func() {
        ticker := time.NewTicker(time.Second / time.Duration(rate))
        defer ticker.Stop()
        for range ticker.C {
            select {
            case rl.tokens <- struct{}{}:
            default:
                // Bucket lleno, descartar token
            }
        }
    }()
    return rl
}

func (rl *RateLimiter) Wait(ctx context.Context) error {
    select {
    case <-rl.tokens:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

// Uso: limita a 10 req/s con ráfagas de hasta 5
func main() {
    limiter := NewRateLimiter(10, 5)
    for _, url := range urls {
        limiter.Wait(context.Background())
        go fetchURL(url)
    }
}
```

---

## 7.14 Deteccion de Data Races

El race detector de Go es una herramienta indispensable:

```bash
go run -race main.go
go test -race ./...
go build -race -o mi-app .
```

Ejemplo de data race:

```go
func main() {
    contador := 0
    for i := 0; i < 1000; i++ {
        go func() {
            contador++ // DATA RACE: lectura y escritura concurrente sin sincronizacion
        }()
    }
}

// Ejecutar con: go run -race main.go
// El race detector mostrara el conflicto
```

> **Regla de oro**: si escribes codigo concurrente, siempre ejecuta los tests con `-race`.

---

## Resumen del Capítulo

- Las goroutines son ligeras y eficientes. Se lanzan con `go`.
- `sync.WaitGroup` permite esperar a que un grupo de goroutines termine.
- Los channels son el mecanismo de comunicacion entre goroutines. Pueden ser buffered o unbuffered.
- `select` permite esperar en multiples operaciones de channel.
- `sync.Mutex` y `sync.RWMutex` protegen el acceso a datos compartidos.
- `sync.Once` garantiza ejecucion unica.
- El paquete `context` maneja cancelacion, deadlines y propagacion de valores.
- Ejecuta siempre con `-race` para detectar data races.

En el siguiente capitulo exploraremos el manejo de errores en profundidad.
