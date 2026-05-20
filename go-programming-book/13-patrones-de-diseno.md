# Capítulo 13: Patrones de Diseño en Go

Los patrones de diseno son soluciones probadas a problemas recurrentes. En Go, muchos patrones clasicos se adaptan a la filosofia del lenguaje: sin herencia, con interfaces implicitas y composicion. Este capitulo explora los patrones mas utiles en el ecosistema Go.

---

## 13.1 Patrones Creacionales

### Factory (Fabrica)

```go
package main

import "fmt"

type Transporte interface {
    Entregar() string
}

type Camion struct {
    Capacidad int
}

func (c Camion) Entregar() string {
    return fmt.Sprintf("Entregando en camion (capacidad: %d kg)", c.Capacidad)
}

type Barco struct {
    Puerto string
}

func (b Barco) Entregar() string {
    return fmt.Sprintf("Entregando por mar desde %s", b.Puerto)
}

type Avion struct {
    Aerolinea string
}

func (a Avion) Entregar() string {
    return fmt.Sprintf("Entregando por aire con %s", a.Aerolinea)
}

// Factory function: decide que transporte crear segun el tipo
func NuevoTransporte(tipo string) (Transporte, error) {
    switch tipo {
    case "camion":
        return Camion{Capacidad: 1000}, nil
    case "barco":
        return Barco{Puerto: "Valencia"}, nil
    case "avion":
        return Avion{Aerolinea: "DHL"}, nil
    default:
        return nil, fmt.Errorf("tipo de transporte desconocido: %s", tipo)
    }
}

func main() {
    camion, _ := NuevoTransporte("camion")
    barco, _ := NuevoTransporte("barco")

    fmt.Println(camion.Entregar())
    fmt.Println(barco.Entregar())
}
```

### Builder

```go
package main

import (
    "fmt"
    "time"
)

type PeticionHTTP struct {
    Metodo  string
    URL     string
    Headers map[string]string
    Cuerpo  []byte
    Timeout time.Duration
}

type BuilderPeticion struct {
    peticion PeticionHTTP
}

func NuevaPeticion(metodo, url string) *BuilderPeticion {
    return &BuilderPeticion{
        peticion: PeticionHTTP{
            Metodo:  metodo,
            URL:     url,
            Headers: make(map[string]string),
            Timeout: 30 * time.Second,
        },
    }
}

func (b *BuilderPeticion) ConHeader(clave, valor string) *BuilderPeticion {
    b.peticion.Headers[clave] = valor
    return b
}

func (b *BuilderPeticion) ConCuerpo(cuerpo []byte) *BuilderPeticion {
    b.peticion.Cuerpo = cuerpo
    return b
}

func (b *BuilderPeticion) ConTimeout(t time.Duration) *BuilderPeticion {
    b.peticion.Timeout = t
    return b
}

func (b *BuilderPeticion) Construir() PeticionHTTP {
    return b.peticion
}

func main() {
    peticion := NuevaPeticion("POST", "https://api.ejemplo.com/usuarios").
        ConHeader("Content-Type", "application/json").
        ConHeader("Authorization", "Bearer token123").
        ConCuerpo([]byte(`{"nombre": "Andres"}`)).
        ConTimeout(5 * time.Second).
        Construir()

    fmt.Printf("Peticion: %+v\n", peticion)
}
```

### Singleton

```go
package main

import (
    "fmt"
    "sync"
)

type Configuracion struct {
    Puerto    int
    Entorno   string
    Debug     bool
}

var (
    instancia *Configuracion
    unaVez    sync.Once
)

func ObtenerConfiguracion() *Configuracion {
    unaVez.Do(func() {
        fmt.Println("Inicializando configuracion...")
        instancia = &Configuracion{
            Puerto:  8080,
            Entorno: "desarrollo",
            Debug:   true,
        }
    })
    return instancia
}

func main() {
    c1 := ObtenerConfiguracion()
    c2 := ObtenerConfiguracion()

    fmt.Println(c1 == c2)           // true (misma instancia)
    fmt.Println(c1.Puerto)          // 8080
    fmt.Println("Inicializando...") // Solo aparece UNA vez
}
```

### Functional Options

El patron mas idiomatico en Go para configuracion:

```go
package main

import (
    "fmt"
    "time"
)

type Servidor struct {
    Puerto     int
    Timeout    time.Duration
    MaxConex    int
    TLS        bool
    Middleware []func()
}

type OpcionServidor func(*Servidor)

func ConPuerto(p int) OpcionServidor {
    return func(s *Servidor) {
        s.Puerto = p
    }
}

func ConTimeout(t time.Duration) OpcionServidor {
    return func(s *Servidor) {
        s.Timeout = t
    }
}

func ConTLS() OpcionServidor {
    return func(s *Servidor) {
        s.TLS = true
    }
}

func ConMaxConexiones(n int) OpcionServidor {
    return func(s *Servidor) {
        s.MaxConex = n
    }
}

func NuevoServidor(opciones ...OpcionServidor) *Servidor {
    // Valores por defecto
    s := &Servidor{
        Puerto:  8080,
        Timeout: 30 * time.Second,
        MaxConex: 100,
    }

    // Aplicar opciones
    for _, opt := range opciones {
        opt(s)
    }

    return s
}

func main() {
    s := NuevoServidor(
        ConPuerto(9090),
        ConTLS(),
        ConMaxConexiones(1000),
    )
    fmt.Printf("Servidor: puerto=%d, tls=%t, maxconex=%d\n",
        s.Puerto, s.TLS, s.MaxConex)
}
```

---

## 13.2 Patrones Estructurales

### Adapter

Adapta una interface a otra que el cliente espera:

```go
package main

import "fmt"

// Interface que espera nuestro sistema
type ProcesadorPago interface {
    Pagar(monto float64, moneda string) error
}

// Sistema legacy que ya existe (API de terceros)
type PayPalSDK struct{}

func (p *PayPalSDK) RealizarCobro(cantidad float64, codigoMoneda string) error {
    fmt.Printf("PayPal: cobrando %.2f %s\n", cantidad, codigoMoneda)
    return nil
}

// Adapter: traduce ProcesadorPago -> PayPalSDK
type PayPalAdapter struct {
    sdk *PayPalSDK
}

func (a *PayPalAdapter) Pagar(monto float64, moneda string) error {
    return a.sdk.RealizarCobro(monto, moneda)
}

// Otro sistema legacy
type StripeAPI struct{}

func (s *StripeAPI) Charge(amount int64, currency string) error {
    fmt.Printf("Stripe: cobrando %d %s\n", amount, currency)
    return nil
}

type StripeAdapter struct {
    api *StripeAPI
}

func (a *StripeAdapter) Pagar(monto float64, moneda string) error {
    // Stripe usa centavos
    centavos := int64(monto * 100)
    return a.api.Charge(centavos, moneda)
}

// Orquestador que usa la interface comun
type Tienda struct {
    pagos ProcesadorPago
}

func (t *Tienda) Comprar(monto float64) {
    if err := t.pagos.Pagar(monto, "EUR"); err != nil {
        fmt.Println("Error en pago:", err)
    }
}

func main() {
    tienda1 := Tienda{pagos: &PayPalAdapter{sdk: &PayPalSDK{}}}
    tienda1.Comprar(99.99)

    tienda2 := Tienda{pagos: &StripeAdapter{api: &StripeAPI{}}}
    tienda2.Comprar(49.50)
}
```

### Decorator

Agrega comportamiento a objetos sin modificar su estructura:

```go
package main

import (
    "fmt"
    "log"
    "time"
)

// Interface base
type Procesador interface {
    Procesar(datos string) (string, error)
}

// Implementacion concreta
type ProcesadorBase struct{}

func (p *ProcesadorBase) Procesar(datos string) (string, error) {
    return fmt.Sprintf("procesado: %s", datos), nil
}

// Decorator: agrega logging
type ProcesadorConLog struct {
    base Procesador
}

func (p *ProcesadorConLog) Procesar(datos string) (string, error) {
    log.Printf("INICIO: procesando '%s'", datos)
    resultado, err := p.base.Procesar(datos)
    if err != nil {
        log.Printf("ERROR: %v", err)
    } else {
        log.Printf("FIN: resultado '%s'", resultado)
    }
    return resultado, err
}

// Decorator: agrega medicion de tiempo
type ProcesadorConMetricas struct {
    base Procesador
}

func (p *ProcesadorConMetricas) Procesar(datos string) (string, error) {
    inicio := time.Now()
    resultado, err := p.base.Procesar(datos)
    fmt.Printf("Metrica: procesamiento en %v\n", time.Since(inicio))
    return resultado, err
}

// Decorator: agrega validacion
type ProcesadorConValidacion struct {
    base Procesador
}

func (p *ProcesadorConValidacion) Procesar(datos string) (string, error) {
    if datos == "" {
        return "", fmt.Errorf("datos vacios no permitidos")
    }
    return p.base.Procesar(datos)
}

func main() {
    // Apilar decorators
    procesador := &ProcesadorConMetricas{
        base: &ProcesadorConLog{
            base: &ProcesadorConValidacion{
                base: &ProcesadorBase{},
            },
        },
    }

    resultado, _ := procesador.Procesar("Hola, mundo!")
    fmt.Println(resultado)
}
```

### Facade (Fachada)

Provee una interface simplificada a un sistema complejo:

```go
package main

import (
    "fmt"
)

// Subsistema complejo: varios componentes
type Autenticador struct{}
func (a *Autenticador) VerificarToken(token string) error {
    fmt.Println("Verificando token...")
    return nil
}

type BaseDeDatos struct{}
func (b *BaseDeDatos) BuscarUsuario(id string) (string, error) {
    fmt.Println("Buscando usuario en BD...")
    return "Andres", nil
}

type Cache struct{}
func (c *Cache) Obtener(clave string) (string, bool) {
    fmt.Println("Buscando en cache...")
    return "", false
}
func (c *Cache) Guardar(clave, valor string) {
    fmt.Println("Guardando en cache...")
}

type Logger struct{}
func (l *Logger) Registrar(accion string) {
    fmt.Printf("Log: %s\n", accion)
}

// Facade: simplifica el acceso al subsistema
type ServicioPerfil struct {
    auth  *Autenticador
    db    *BaseDeDatos
    cache *Cache
    log   *Logger
}

func NuevoServicioPerfil() *ServicioPerfil {
    return &ServicioPerfil{
        auth:  &Autenticador{},
        db:    &BaseDeDatos{},
        cache: &Cache{},
        log:   &Logger{},
    }
}

func (s *ServicioPerfil) ObtenerPerfil(token, usuarioID string) (string, error) {
    s.log.Registrar("obtener_perfil: inicio")

    if err := s.auth.VerificarToken(token); err != nil {
        return "", fmt.Errorf("autenticacion fallida: %w", err)
    }

    // Intentar cache primero
    if nombre, ok := s.cache.Obtener(usuarioID); ok {
        return nombre, nil
    }

    nombre, err := s.db.BuscarUsuario(usuarioID)
    if err != nil {
        return "", fmt.Errorf("usuario no encontrado: %w", err)
    }

    s.cache.Guardar(usuarioID, nombre)
    s.log.Registrar("obtener_perfil: exito")
    return nombre, nil
}

func main() {
    servicio := NuevoServicioPerfil()
    nombre, _ := servicio.ObtenerPerfil("token123", "user456")
    fmt.Printf("Perfil: %s\n", nombre)
}
```

---

## 13.3 Patrones de Comportamiento

### Strategy

Permite seleccionar un algoritmo en tiempo de ejecucion:

```go
package main

import "fmt"

// Estrategia de descuento
type EstrategiaDescuento interface {
    Calcular(precio float64) float64
}

type SinDescuento struct{}
func (d SinDescuento) Calcular(precio float64) float64 {
    return precio
}

type DescuentoPorcentaje struct {
    Porcentaje float64
}
func (d DescuentoPorcentaje) Calcular(precio float64) float64 {
    return precio * (1 - d.Porcentaje/100)
}

type DescuentoFijo struct {
    Monto float64
}
func (d DescuentoFijo) Calcular(precio float64) float64 {
    resultado := precio - d.Monto
    if resultado < 0 {
        return 0
    }
    return resultado
}

// Contexto que usa la estrategia
type Carrito struct {
    items      []float64
    descuento  EstrategiaDescuento
}

func (c *Carrito) SetEstrategiaDescuento(e EstrategiaDescuento) {
    c.descuento = e
}

func (c *Carrito) AgregarItem(precio float64) {
    c.items = append(c.items, precio)
}

func (c *Carrito) Total() float64 {
    var suma float64
    for _, item := range c.items {
        suma += item
    }
    return c.descuento.Calcular(suma)
}

func main() {
    carrito := &Carrito{descuento: SinDescuento{}}
    carrito.AgregarItem(100)
    carrito.AgregarItem(50)

    fmt.Printf("Sin descuento: %.2f\n", carrito.Total()) // 150.00

    carrito.SetEstrategiaDescuento(DescuentoPorcentaje{Porcentaje: 20})
    fmt.Printf("20%% descuento: %.2f\n", carrito.Total()) // 120.00

    carrito.SetEstrategiaDescuento(DescuentoFijo{Monto: 30})
    fmt.Printf("30 de descuento: %.2f\n", carrito.Total()) // 120.00
}
```

### Observer

Notifica a multiples suscriptores cuando ocurre un evento:

```go
package main

import "fmt"

type Observador interface {
    Actualizar(evento string, datos interface{})
}

type Notificador struct {
    observadores map[string][]Observador
}

func NuevoNotificador() *Notificador {
    return &Notificador{
        observadores: make(map[string][]Observador),
    }
}

func (n *Notificador) Suscribir(evento string, o Observador) {
    n.observadores[evento] = append(n.observadores[evento], o)
}

func (n *Notificador) Emitir(evento string, datos interface{}) {
    for _, o := range n.observadores[evento] {
        o.Actualizar(evento, datos)
    }
}

// Observadores concretos
type LoggerEventos struct {
    Nombre string
}
func (l LoggerEventos) Actualizar(evento string, datos interface{}) {
    fmt.Printf("[%s] Evento %s: %v\n", l.Nombre, evento, datos)
}

type EnviadorEmail struct {
    Destino string
}
func (e EnviadorEmail) Actualizar(evento string, datos interface{}) {
    fmt.Printf("Enviando email a %s: evento %s, datos %v\n", e.Destino, evento, datos)
}

type MetricaService struct{}
func (m MetricaService) Actualizar(evento string, datos interface{}) {
    fmt.Printf("Metrica: %s contador++\n", evento)
}

func main() {
    notificador := NuevoNotificador()

    notificador.Suscribir("usuario.creado", LoggerEventos{"AppLogger"})
    notificador.Suscribir("usuario.creado", EnviadorEmail{"admin@ejemplo.com"})
    notificador.Suscribir("usuario.creado", MetricaService{})

    notificador.Suscribir("usuario.eliminado", LoggerEventos{"AppLogger"})
    notificador.Suscribir("usuario.eliminado", MetricaService{})

    notificador.Emitir("usuario.creado", map[string]string{"id": "123", "email": "a@test.com"})
    fmt.Println("---")
    notificador.Emitir("usuario.eliminado", map[string]string{"id": "456"})
}
```

### Command

Encapsula una peticion como un objeto:

```go
package main

import "fmt"

type Comando interface {
    Ejecutar() error
    Deshacer() error
}

// Comandos concretos
type CrearUsuarioComando struct {
    repo     *RepositorioMemoria
    usuario  Usuario
}

func (c *CrearUsuarioComando) Ejecutar() error {
    return c.repo.Guardar(c.usuario)
}

func (c *CrearUsuarioComando) Deshacer() error {
    return c.repo.Eliminar(c.usuario.ID)
}

type ActualizarUsuarioComando struct {
    repo     *RepositorioMemoria
    id       int
    nuevo    Usuario
    anterior Usuario
}

func (c *ActualizarUsuarioComando) Ejecutar() error {
    u, err := c.repo.Obtener(c.id)
    if err != nil {
        return err
    }
    c.anterior = *u
    return c.repo.Actualizar(c.nuevo)
}

func (c *ActualizarUsuarioComando) Deshacer() error {
    return c.repo.Actualizar(c.anterior)
}

// Invocador que ejecuta y mantiene historial
type Invocador struct {
    historial []Comando
}

func (i *Invocador) Ejecutar(c Comando) error {
    if err := c.Ejecutar(); err != nil {
        return err
    }
    i.historial = append(i.historial, c)
    return nil
}

func (i *Invocador) DeshacerUltimo() error {
    if len(i.historial) == 0 {
        return fmt.Errorf("no hay comandos para deshacer")
    }
    ultimo := i.historial[len(i.historial)-1]
    i.historial = i.historial[:len(i.historial)-1]
    return ultimo.Deshacer()
}

// Repositorio simple para el ejemplo
type Usuario struct {
    ID     int
    Nombre string
}

type RepositorioMemoria struct {
    usuarios map[int]Usuario
}

func NuevoRepositorioMemoria() *RepositorioMemoria {
    return &RepositorioMemoria{usuarios: make(map[int]Usuario)}
}

func (r *RepositorioMemoria) Guardar(u Usuario) error {
    r.usuarios[u.ID] = u
    return nil
}

func (r *RepositorioMemoria) Obtener(id int) (*Usuario, error) {
    u, ok := r.usuarios[id]
    if !ok {
        return nil, fmt.Errorf("usuario %d no encontrado", id)
    }
    return &u, nil
}

func (r *RepositorioMemoria) Actualizar(u Usuario) error {
    r.usuarios[u.ID] = u
    return nil
}

func (r *RepositorioMemoria) Eliminar(id int) error {
    delete(r.usuarios, id)
    return nil
}

func main() {
    repo := NuevoRepositorioMemoria()
    invocador := &Invocador{}

    crear := &CrearUsuarioComando{
        repo:    repo,
        usuario: Usuario{ID: 1, Nombre: "Andres"},
    }

    invocador.Ejecutar(crear)
    fmt.Printf("Usuarios: %v\n", repo.usuarios)

    invocador.DeshacerUltimo()
    fmt.Printf("Despues de deshacer: %v\n", repo.usuarios)
}
```

---

## 13.4 Patrones Especificos de Go

### Table-Driven Tests (Ver Capitulo 10 para detalle completo)

```go
func TestValidarEmail(t *testing.T) {
    casos := []struct {
        nombre    string
        email     string
        esperaErr bool
    }{
        {"valido", "a@b.com", false},
        {"sin arroba", "usuario", true},
        {"vacio", "", true},
    }
    for _, c := range casos {
        t.Run(c.nombre, func(t *testing.T) {
            err := ValidarEmail(c.email)
            if (err != nil) != c.esperaErr {
                t.Errorf("ValidarEmail(%q) error = %v", c.email, err)
            }
        })
    }
}
```

### Middleware Chain (para HTTP)

```go
package main

import (
    "log"
    "net/http"
    "time"
)

type Middleware func(http.Handler) http.Handler

func Encadenar(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func MiddlewareLogging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        inicio := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(inicio))
    })
}

func MiddlewareRecovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                http.Error(w, "Internal Server Error", 500)
                log.Printf("PANIC: %v", err)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func main() {
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hola, mundo!"))
    })

    http.Handle("/", Encadenar(handler,
        MiddlewareLogging,
        MiddlewareRecovery,
    ))

    http.ListenAndServe(":8080", nil)
}
```

### Circuit Breaker

```go
package main

import (
    "errors"
    "fmt"
    "sync"
    "time"
)

type Estado int

const (
    Cerrado Estado = iota
    Abierto
    SemiAbierto
)

type CircuitBreaker struct {
    mu            sync.Mutex
    estado        Estado
    fallos        int
    maxFallos     int
    timeout       time.Duration
    ultimoFallo   time.Time
}

func NuevoCircuitBreaker(maxFallos int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        estado:    Cerrado,
        maxFallos: maxFallos,
        timeout:   timeout,
    }
}

func (cb *CircuitBreaker) Ejecutar(fn func() error) error {
    cb.mu.Lock()
    if cb.estado == Abierto {
        if time.Since(cb.ultimoFallo) > cb.timeout {
            cb.estado = SemiAbierto
            cb.mu.Unlock()
        } else {
            cb.mu.Unlock()
            return errors.New("circuito abierto")
        }
    } else {
        cb.mu.Unlock()
    }

    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.fallos++
        cb.ultimoFallo = time.Now()
        if cb.fallos >= cb.maxFallos {
            cb.estado = Abierto
        }
        return err
    }

    cb.fallos = 0
    cb.estado = Cerrado
    return nil
}

func main() {
    cb := NuevoCircuitBreaker(3, 2*time.Second)
    intentosFallidos := 0

    for i := 0; i < 10; i++ {
        err := cb.Ejecutar(func() error {
            if intentosFallidos < 5 {
                intentosFallidos++
                return fmt.Errorf("fallo %d", intentosFallidos)
            }
            return nil
        })
        if err != nil {
            fmt.Printf("Intento %d: ERROR - %v\n", i+1, err)
        } else {
            fmt.Printf("Intento %d: EXITO\n", i+1)
        }
        time.Sleep(100 * time.Millisecond)
    }
}
```

---

## Resumen del Capítulo

- **Factory** crea objetos sin exponer la logica de creacion al cliente.
- **Builder** construye objetos complejos paso a paso con una API fluida.
- **Singleton** garantiza una unica instancia usando `sync.Once`.
- **Functional Options** es el patron idiomatico de Go para configuracion.
- **Adapter** traduce interfaces para integrar sistemas incompatibles.
- **Decorator** agrega comportamiento a objetos envolviendolos en capas.
- **Facade** simplifica el acceso a sistemas complejos.
- **Strategy** permite intercambiar algoritmos en tiempo de ejecucion via interfaces.
- **Observer** notifica a multiples suscriptores de eventos.
- **Command** encapsula operaciones como objetos (util para undo/redo).
- **Patrones especificos de Go** incluyen Table-Driven Tests, Middleware Chain y Circuit Breaker.

En el siguiente capitulo exploraremos la arquitectura hexagonal aplicada a Go.
