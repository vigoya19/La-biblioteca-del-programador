# Capítulo 6: Interfaces y Polimorfismo

## 6.1 Definicion e Implementacion Implicita

En Go, las interfaces se implementan **implicitamente**. No hay `implements` keyword. Si un tipo tiene todos los metodos de una interface, la implementa automaticamente:

```go
package main

import (
    "fmt"
    "math"
)

// Definir una interface
type Figura interface {
    Area() float64
    Perimetro() float64
}

// Circulo implementa Figura (implicitamente)
type Circulo struct {
    Radio float64
}

func (c Circulo) Area() float64 {
    return math.Pi * c.Radio * c.Radio
}

func (c Circulo) Perimetro() float64 {
    return 2 * math.Pi * c.Radio
}

// Rectangulo tambien implementa Figura
type Rectangulo struct {
    Ancho, Alto float64
}

func (r Rectangulo) Area() float64 {
    return r.Ancho * r.Alto
}

func (r Rectangulo) Perimetro() float64 {
    return 2 * (r.Ancho + r.Alto)
}

// Funcion que acepta cualquier Figura
func imprimirInfo(f Figura) {
    fmt.Printf("Tipo: %T\n", f)
    fmt.Printf("Area: %.2f\n", f.Area())
    fmt.Printf("Perimetro: %.2f\n", f.Perimetro())
    fmt.Println()
}

func main() {
    c := Circulo{Radio: 5}
    r := Rectangulo{Ancho: 4, Alto: 3}

    imprimirInfo(c)
    imprimirInfo(r)

    // Slice de interfaces
    figuras := []Figura{
        Circulo{Radio: 1},
        Rectangulo{Ancho: 2, Alto: 3},
        Circulo{Radio: 10},
    }

    var areaTotal float64
    for _, f := range figuras {
        areaTotal += f.Area()
    }
    fmt.Printf("Area total: %.2f\n", areaTotal)
}
```

### Verificacion en tiempo de compilacion

```go
// Verificar que un tipo implementa una interface (patron comun)
var _ Figura = Circulo{}      // Error de compilacion si no implementa
var _ Figura = (*Circulo)(nil) // Para verificar con receptor de puntero
```

---

## 6.2 Interfaces Comunes de la Libreria Estandar

### fmt.Stringer

```go
type Stringer interface {
    String() string
}

type Persona struct {
    Nombre string
    Edad   int
}

func (p Persona) String() string {
    return fmt.Sprintf("%s (%d anios)", p.Nombre, p.Edad)
}

// Ahora fmt.Println usa String() automaticamente
// fmt.Println(Persona{"Andres", 30}) -> "Andres (30 anios)"
```

### error

```go
type error interface {
    Error() string
}

// Crear errores personalizados
type ErrorValidacion struct {
    Campo   string
    Mensaje string
}

func (e *ErrorValidacion) Error() string {
    return fmt.Sprintf("validacion fallida en %s: %s", e.Campo, e.Mensaje)
}

func validarEdad(edad int) error {
    if edad < 0 || edad > 150 {
        return &ErrorValidacion{
            Campo:   "edad",
            Mensaje: "debe estar entre 0 y 150",
        }
    }
    return nil
}
```

### io.Reader y io.Writer

Las interfaces mas importantes de Go para I/O:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Cientos de tipos las implementan: archivos, conexiones de red, buffers, compresores, etc.

```go
package main

import (
    "fmt"
    "io"
    "os"
    "strings"
)

// Funcion que acepta cualquier Reader
func contarBytes(r io.Reader) (int64, error) {
    return io.Copy(io.Discard, r)
}

// Funcion que escribe en cualquier Writer
func escribirSaludo(w io.Writer, nombre string) {
    fmt.Fprintf(w, "Hola, %s!\n", nombre)
}

func main() {
    // strings.Reader implementa io.Reader
    r := strings.NewReader("Hola mundo, esto es una prueba")
    n, _ := contarBytes(r)
    fmt.Println("Bytes:", n)

    // os.Stdout implementa io.Writer
    escribirSaludo(os.Stdout, "Andres")

    // strings.Builder implementa io.Writer
    var builder strings.Builder
    escribirSaludo(&builder, "Maria")
    fmt.Println(builder.String())
}
```

### sort.Interface

```go
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}

type PorEdad []Persona

func (p PorEdad) Len() int           { return len(p) }
func (p PorEdad) Less(i, j int) bool { return p[i].Edad < p[j].Edad }
func (p PorEdad) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

// Uso: sort.Sort(PorEdad(personas))

// Alternativa moderna (Go 1.21+):
// slices.SortFunc(personas, func(a, b Persona) int {
//     return a.Edad - b.Edad
// })
```

### Otros ejemplos de io.Reader/Writer

```go
package main

import (
    "fmt"
    "io"
    "os"
    "strings"
)

func main() {
    // io.LimitReader: limita cuantos bytes leer
    r := strings.NewReader("Hola, Mundo!")
    limitado := io.LimitReader(r, 4)
    buf := make([]byte, 10)
    n, _ := limitado.Read(buf)
    fmt.Println(string(buf[:n])) // "Hola"

    // io.TeeReader: lee y copia simultaneamente
    r2 := strings.NewReader("datos importantes")
    var capturado strings.Builder
    tee := io.TeeReader(r2, &capturado)
    io.ReadAll(tee)
    fmt.Println("Capturado:", capturado.String()) // "datos importantes"

    // io.MultiReader: concatena multiples readers
    r3 := io.MultiReader(
        strings.NewReader("Parte 1 "),
        strings.NewReader("Parte 2 "),
        strings.NewReader("Parte 3"),
    )
    todo, _ := io.ReadAll(r3)
    fmt.Println(string(todo)) // "Parte 1 Parte 2 Parte 3"

    // io.MultiWriter: escribe en multiples writers
    var buf1, buf2 strings.Builder
    w := io.MultiWriter(&buf1, &buf2)
    w.Write([]byte("Hola"))
    fmt.Println(buf1.String(), buf2.String()) // "Hola" "Hola"

    // io.Pipe: conectar reader y writer en memoria
    pipeR, pipeW := io.Pipe()
    go func() {
        defer pipeW.Close()
        pipeW.Write([]byte("mensaje por pipe"))
    }()
    data, _ := io.ReadAll(pipeR)
    fmt.Println(string(data)) // "mensaje por pipe"

    // io.ReadCloser: reader que se puede cerrar
    content, _ := leerYcerrar(strings.NewReader("contenido"))
    fmt.Println(content)
}

func leerYcerrar(r io.Reader) (string, error) {
    data, err := io.ReadAll(r)
    if err != nil {
        return "", err
    }
    if closer, ok := r.(io.Closer); ok {
        closer.Close()
    }
    return string(data), nil
}

// Custom Reader: reporta progreso de lectura
type ProgressReader struct {
    reader io.Reader
    total  int64
    leido  int64
    onProgress func(leido, total int64)
}

func (pr *ProgressReader) Read(p []byte) (int, error) {
    n, err := pr.reader.Read(p)
    pr.leido += int64(n)
    if pr.onProgress != nil {
        pr.onProgress(pr.leido, pr.total)
    }
    return n, err
}
```

### Ejemplo real: procesar upload de archivo

```go
func procesarUpload(r io.Reader, maxSize int64) ([]byte, error) {
    // Limitar tamaño de lectura
    limitado := io.LimitReader(r, maxSize)

    // Leer todo el contenido
    data, err := io.ReadAll(limitado)
    if err != nil {
        return nil, fmt.Errorf("error al leer: %w", err)
    }

    return data, nil
}

func handler(w http.ResponseWriter, r *http.Request) {
    data, err := procesarUpload(r.Body, 10<<20) // 10 MB max
    if err != nil {
        http.Error(w, "Error al procesar archivo", http.StatusBadRequest)
        return
    }
    fmt.Fprintf(w, "Recibidos %d bytes", len(data))
}
```

---

## 6.3 Composicion de Interfaces

Las interfaces se pueden componer embebiendo otras interfaces:

```go
// Interfaces pequenas de la stdlib
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// Interfaces compuestas
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// Tu propia composicion
type Repositorio interface {
    Lector
    Escritor
}

type Lector interface {
    Buscar(id string) (*Entidad, error)
    Listar() ([]*Entidad, error)
}

type Escritor interface {
    Guardar(e *Entidad) error
    Eliminar(id string) error
}
```

> **Principio**: "Accept interfaces, return structs" (acepta interfaces, retorna structs concretos).

> **Proverbio Go**: "The bigger the interface, the weaker the abstraction" - interfaces pequenas son mas utiles.

---

## 6.4 Type Assertions y Type Switches

### Type assertion

Extrae el valor concreto de una interface:

```go
package main

import "fmt"

func main() {
    var i interface{} = "hola"

    // Type assertion (puede causar panic si falla)
    s := i.(string)
    fmt.Println(s)  // hola

    // Type assertion segura (comma ok)
    s, ok := i.(string)
    if ok {
        fmt.Println("Es string:", s)
    }

    n, ok := i.(int)
    if !ok {
        fmt.Println("No es int, n =", n)  // n = 0 (zero value)
    }
}
```

### Type switch

```go
func procesar(dato interface{}) string {
    switch v := dato.(type) {
    case string:
        return "String de longitud " + fmt.Sprint(len(v))
    case int:
        return fmt.Sprintf("Entero: %d", v)
    case bool:
        if v {
            return "Verdadero"
        }
        return "Falso"
    case []int:
        return fmt.Sprintf("Slice con %d elementos", len(v))
    case nil:
        return "Nil"
    default:
        return fmt.Sprintf("Tipo no manejado: %T", v)
    }
}
```

### Ejemplo practico: validar interfaces opcionales

```go
type Logger interface {
    Log(msg string)
}

type LoggerConNivel interface {
    Logger
    LogConNivel(nivel, msg string)
}

func registrar(l Logger, msg string) {
    // Verificar si el logger tambien soporta niveles
    if ln, ok := l.(LoggerConNivel); ok {
        ln.LogConNivel("INFO", msg)
    } else {
        l.Log(msg)
    }
}
```

---

## 6.5 Interface Vacia (any)

`interface{}` (o su alias `any` desde Go 1.18) puede contener cualquier valor:

```go
package main

import "fmt"

func main() {
    // any puede contener cualquier tipo
    var datos []any
    datos = append(datos, 42)
    datos = append(datos, "hola")
    datos = append(datos, true)
    datos = append(datos, []int{1, 2, 3})

    for _, d := range datos {
        fmt.Printf("Tipo: %T, Valor: %v\n", d, d)
    }

    // Ejemplo: map generico (como JSON)
    config := map[string]any{
        "puerto":    8080,
        "debug":     true,
        "nombre":    "mi-app",
        "servicios": []string{"auth", "api"},
    }

    // Para usar el valor, necesitas type assertion
    if puerto, ok := config["puerto"].(int); ok {
        fmt.Println("Puerto:", puerto)
    }
}
```

> **Buena practica**: evita `any` siempre que sea posible. Usa tipos concretos o interfaces especificas. Con generics (Cap. 11) hay aun menos razones para usar `any`.

---

## 6.6 Interfaces y nil

Un concepto sutil pero importante:

```go
package main

import "fmt"

type MiError struct {
    Mensaje string
}

func (e *MiError) Error() string {
    return e.Mensaje
}

func obtenerError(falla bool) error {
    var err *MiError  // nil de tipo *MiError

    if falla {
        err = &MiError{Mensaje: "algo fallo"}
    }

    return err  // CUIDADO: retorna interface no-nil con valor nil!
}

func main() {
    err := obtenerError(false)

    // Esto NUNCA sera true, aunque err.(*MiError) sea nil
    if err != nil {
        fmt.Println("Error:", err)  // Se ejecuta! panic al llamar Error()
    }

    // La interface tiene (tipo=*MiError, valor=nil) != nil
}

// SOLUCION: retorna nil explicitamente
func obtenerErrorCorrecto(falla bool) error {
    if falla {
        return &MiError{Mensaje: "algo fallo"}
    }
    return nil  // Retorna nil directamente
}
```

---

## 6.7 Diseno con Interfaces

### Interfaces del consumidor, no del productor

```go
// INCORRECTO: definir la interface junto al tipo que la implementa
// paquete database
type Database interface {  // Demasiado especifica, nadie la necesita asi
    Query(sql string) ([]Row, error)
    Execute(sql string) error
    Close() error
}

type PostgresDB struct { ... }
func (db *PostgresDB) Query(sql string) ([]Row, error) { ... }
func (db *PostgresDB) Execute(sql string) error { ... }
func (db *PostgresDB) Close() error { ... }

// CORRECTO: definir la interface donde se consume
// paquete servicio
type Querier interface {  // Solo los metodos que necesito
    Query(sql string) ([]Row, error)
}

func ObtenerUsuarios(db Querier) ([]Usuario, error) {
    rows, err := db.Query("SELECT * FROM usuarios")
    // ...
}
```

### Interfaces pequenas y enfocadas

```go
// Bien: interfaces de 1-3 metodos
type Enviador interface {
    Enviar(destino, mensaje string) error
}

type Almacen interface {
    Guardar(clave string, valor []byte) error
    Obtener(clave string) ([]byte, error)
}

// Mal: interface "god object"
type ServicioCompleto interface {
    Enviar(destino, mensaje string) error
    Guardar(clave string, valor []byte) error
    Obtener(clave string) ([]byte, error)
    Eliminar(clave string) error
    Listar() ([]string, error)
    Conectar() error
    Desconectar() error
    // ... 20 metodos mas
}
```

---

## 6.8 http.Handler: La Interface Web de Go

`http.Handler` es la interface mas importante del desarrollo web en Go. Tiene un solo metodo:

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

### Implementar Handler

```go
package main

import (
    "fmt"
    "net/http"
)

// Struct que implementa Handler
type SaludoHandler struct {
    NombreApp string
}

func (h SaludoHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Bienvenido a %s!\n", h.NombreApp)
}

// Funcion convertida a Handler
func miHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hola desde funcion\n")
}

func main() {
    // Opcion 1: Handler como struct
    http.Handle("/app", SaludoHandler{NombreApp: "Mi API"})

    // Opcion 2: HandlerFunc convierte una funcion en Handler
    http.Handle("/func", http.HandlerFunc(miHandler))

    // Opcion 3: Funcion anonima con HandleFunc (atajo)
    http.HandleFunc("/anon", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hola desde anonima\n")
    })

    http.ListenAndServe(":8080", nil)
}
```

### Middleware con http.Handler

```go
// Middleware: funcion que envuelve un Handler
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "No autorizado", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// Encadenar middlewares
func Encadenar(h http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func main() {
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Recurso protegido\n")
    })

    http.Handle("/api/", Encadenar(handler, LoggingMiddleware, AuthMiddleware))
    http.ListenAndServe(":8080", nil)
}
```

---

## 6.9 Anti-Patrones con Interfaces

### Anti-patron 1: Interfaces prematuras

```go
// MAL: definir interface antes de tener 2+ implementaciones
type RepositorioUsuario interface {
    Guardar(u *Usuario) error
    Buscar(id int) (*Usuario, error)
    Eliminar(id int) error
}

type PostgresRepo struct { db *sql.DB }
// Solo existe UNA implementacion... ¿para que la interface?

// BIEN: empieza con el tipo concreto
type PostgresRepo struct { db *sql.DB }
func (r *PostgresRepo) Guardar(u *Usuario) error { ... }
func (r *PostgresRepo) Buscar(id int) (*Usuario, error) { ... }

// Agrega la interface SOLO cuando tienes una segunda implementacion
// (ej: mock para tests, cache layer, otra BD)
```

### Anti-patron 2: Interfaces demasiado grandes

```go
// MAL: interface "dios" que hace todo
type ServicioCompleto interface {
    CrearUsuario(u *Usuario) error
    ActualizarUsuario(u *Usuario) error
    EliminarUsuario(id int) error
    CrearOrden(o *Orden) error
    ProcesarPago(p *Pago) error
    EnviarEmail(destino, asunto, cuerpo string) error
    GenerarReporte(tipo string) ([]byte, error)
}

// BIEN: interfaces pequeñas y segregadas
type CreadorUsuario interface {
    CrearUsuario(u *Usuario) error
}

type ProcesadorPagos interface {
    ProcesarPago(p *Pago) error
}

// El consumidor define SOLO los metodos que realmente necesita
```

### Anti-patron 3: Retornar interfaces

```go
// MAL: el constructor retorna una interface
func NuevoRepositorio(dsn string) RepositorioUsuario {
    return &PostgresRepo{dsn: dsn}
}
// El llamante no puede acceder a metodos especificos de PostgresRepo
// sin hacer type assertion

// BIEN: retorna el tipo concreto
func NuevoRepositorio(dsn string) *PostgresRepo {
    return &PostgresRepo{dsn: dsn}
}
// El llamante puede usar la interface si quiere:
// var repo RepositorioUsuario = NuevoRepositorio(dsn)

// Excepcion: cuando el constructor devuelve diferentes tipos
// segun condiciones, ahi si es valido retornar interface
func NuevoRepositorio(dsn string) RepositorioUsuario {
    if strings.Contains(dsn, "postgres") {
        return &PostgresRepo{dsn: dsn}
    }
    return &MemoriaRepo{}
}
```

---

## 6.10 Interfaces vs Generics: Cuando Usar Cada Uno

Esta es una decision diaria en Go moderno:

```go
// CASO 1: Usa INTERFACE cuando el comportamiento varia
// pero los tipos de entrada/salida son los mismos

type Escritor interface {
    Write(p []byte) (n int, err error)
}
// Todos los escritores reciben []byte y retornan (int, error)

// CASO 2: Usa GENERICS cuando el tipo de datos varia
// pero la operacion es la misma

func Filtrar[T any](slice []T, fn func(T) bool) []T {
    var resultado []T
    for _, v := range slice {
        if fn(v) {
            resultado = append(resultado, v)
        }
    }
    return resultado
}
// La operacion es identica, solo cambia el tipo

// CASO 3: INTERFACE para polimorfismo en runtime
type Descuento interface {
    Aplicar(precio float64) float64
}
// Puedes tener N implementaciones y elegir en runtime

// CASO 4: GENERICS para seguridad de tipos en tiempo de compilacion
func Ordenar[T constraints.Ordered](s []T) { ... }
// Sin generics, necesitarias reflection o codigo repetido
```

### Tabla de decision

| Situacion | Usar | Por que |
|-----------|------|---------|
| Comportamiento diferente, mismos tipos | Interface | Polimorfismo clasico |
| Misma operacion, tipos diferentes | Generics | Evita repeticion |
| Necesitas almacenar tipos mixtos en un slice | Interface | `[]Descuento` |
| Necesitas operaciones matematicas genericas | Generics | `Sumar[T Numerico](a, b T) T` |
| El tipo define el comportamiento | Interface | `io.Reader` |
| El comportamiento define el tipo | Generics | `Pila[T any]` |

---

## Resumen del Capítulo

- Las interfaces en Go se implementan implicitamente (duck typing estatico).
- Interfaces pequenas (1-3 metodos) son mas utiles y componibles.
- `io.Reader` y `io.Writer` son las interfaces mas importantes del ecosistema, con muchos wrappers utiles (`LimitReader`, `TeeReader`, `MultiReader`, `Pipe`).
- `http.Handler` es la interface del desarrollo web: un solo metodo, maxima flexibilidad con middlewares.
- Type assertions y type switches permiten trabajar con tipos concretos.
- Cuidado con interfaces nil: una interface con tipo pero valor nil no es igual a nil.
- Define interfaces donde se consumen, no donde se producen.
- Evita interfaces prematuras (espera 2+ implementaciones), interfaces gigantes (segrega) y retornar interfaces desde constructores.
- Usa interfaces para polimorfismo en runtime, generics para operaciones identicas con tipos diferentes.
- "Accept interfaces, return structs".
