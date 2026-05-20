# Capítulo 15: Temas Avanzados

Este capitulo cubre herramientas y tecnicas avanzadas de Go que, aunque no se usan a diario, son fundamentales para ciertos escenarios: reflection para codigo generico en runtime, integracion con C, compilacion condicional, profiling y optimizacion.

---

## 15.1 Reflection

El paquete `reflect` permite examinar y manipular tipos en tiempo de ejecucion. Es poderoso pero debe usarse con moderacion: es mas lento, menos seguro y mas dificil de leer que el codigo tipado estaticamente.

### Examinar tipos

```go
package main

import (
    "fmt"
    "reflect"
)

type Persona struct {
    Nombre string `json:"nombre" validate:"required"`
    Edad   int    `json:"edad"`
}

func main() {
    p := Persona{Nombre: "Andres", Edad: 30}
    t := reflect.TypeOf(p)
    v := reflect.ValueOf(p)

    fmt.Println("Tipo:", t.Name())        // Persona
    fmt.Println("Kind:", t.Kind())         // struct
    fmt.Println("Num campos:", t.NumField()) // 2

    for i := 0; i < t.NumField(); i++ {
        campo := t.Field(i)
        valor := v.Field(i)
        tag := campo.Tag

        fmt.Printf("  Campo %d: %s (%s) = %v\n", i+1, campo.Name, campo.Type, valor)
        fmt.Printf("    Tag json: %q\n", tag.Get("json"))
    }
}
```

### Crear y modificar valores

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    // Crear un slice via reflection
    tipoSlice := reflect.TypeOf([]int{})
    slice := reflect.MakeSlice(tipoSlice, 0, 3)

    slice = reflect.Append(slice, reflect.ValueOf(1))
    slice = reflect.Append(slice, reflect.ValueOf(2))
    slice = reflect.Append(slice, reflect.ValueOf(3))

    fmt.Println("Slice:", slice.Interface()) // [1 2 3]

    // Modificar un valor a traves de reflection
    x := 42
    v := reflect.ValueOf(&x).Elem() // Necesitamos un puntero para modificar
    if v.CanSet() {
        v.SetInt(100)
    }
    fmt.Println("x:", x) // 100
}
```

### Llamar metodos por reflection

```go
package main

import (
    "fmt"
    "reflect"
)

type Calculadora struct{}

func (c Calculadora) Sumar(a, b int) int {
    return a + b
}

func (c Calculadora) Restar(a, b int) int {
    return a - b
}

func main() {
    calc := Calculadora{}
    v := reflect.ValueOf(calc)

    // Llamar Sumar
    metodoSumar := v.MethodByName("Sumar")
    args := []reflect.Value{
        reflect.ValueOf(10),
        reflect.ValueOf(3),
    }
    resultados := metodoSumar.Call(args)
    fmt.Println("Sumar:", resultados[0].Int()) // 13

    // Llamar Restar
    metodoRestar := v.MethodByName("Restar")
    resultados = metodoRestar.Call(args)
    fmt.Println("Restar:", resultados[0].Int()) // 7
}
```

### Casos de uso legitimos

```go
// 1. Serializacion/deserializacion (encoding/json usa reflection)
// 2. ORMs y mapeo de BD
// 3. Inyeccion de dependencias
// 4. Validacion con tags de struct
// 5. Generacion de codigo

// Ejemplo: validador generico con tags
func ValidarStruct(s interface{}) error {
    v := reflect.ValueOf(s)
    t := v.Type()

    for i := 0; i < t.NumField(); i++ {
        campo := t.Field(i)
        valor := v.Field(i)

        if tag := campo.Tag.Get("validate"); tag == "required" {
            if valor.IsZero() {
                return fmt.Errorf("campo %s es requerido", campo.Name)
            }
        }
    }
    return nil
}
```

---

## 15.2 unsafe

El paquete `unsafe` permite eludir el sistema de tipos de Go. Su nombre lo dice todo: **no es seguro**. Usalo solo cuando sea absolutamente necesario.

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    // Tamaño y alineacion
    var x int32 = 42
    fmt.Println("Tamaño:", unsafe.Sizeof(x))      // 4 bytes
    fmt.Println("Alineacion:", unsafe.Alignof(x))  // 4

    // Convertir entre punteros (MUY peligroso)
    var f float64 = 3.14
    ptr := unsafe.Pointer(&f)
    intPtr := (*int64)(ptr) // Interpretar float64 como int64
    fmt.Println("Bits de 3.14:", *intPtr) // Representacion binaria
}
```

### Cuando usar unsafe (y cuando no)

```go
// BIEN: optimizaciones de muy bajo nivel
// - Convertir []byte a string sin copia (cuando sabes que no mutara)
func BytesAString(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}

// MAL: para evitar el sistema de tipos
// - Acceder a campos privados de otros paquetes
// - Convertir tipos incompatibles
// - Hacer aritmetica de punteros sin necesidad

// Regla: si puedes resolverlo sin unsafe, hazlo sin unsafe.
```

---

## 15.3 CGo

CGo permite llamar codigo C desde Go y viceversa. Util cuando necesitas integrar librerias C existentes.

```go
package main

/*
#include <stdio.h>
#include <stdlib.h>

void saludar(char* nombre) {
    printf("Hola desde C, %s!\n", nombre);
}

int sumar(int a, int b) {
    return a + b;
}
*/
import "C"
import (
    "fmt"
    "unsafe"
)

func main() {
    // Llamar funcion C
    nombre := C.CString("Andres")
    defer C.free(unsafe.Pointer(nombre))
    C.saludar(nombre)

    resultado := C.sumar(3, 4)
    fmt.Println("Suma desde C:", resultado) // 7
}
```

### Pasar datos complejos entre Go y C

```go
package main

/*
#include <stdlib.h>

typedef struct {
    int id;
    char* nombre;
} Persona;

Persona* crearPersona(int id, char* nombre) {
    Persona* p = (Persona*)malloc(sizeof(Persona));
    p->id = id;
    p->nombre = nombre;
    return p;
}
*/
import "C"
import (
    "fmt"
    "unsafe"
)

func main() {
    nombre := C.CString("Andres")
    defer C.free(unsafe.Pointer(nombre))

    persona := C.crearPersona(1, nombre)
    defer C.free(unsafe.Pointer(persona))

    fmt.Printf("Persona C: id=%d, nombre=%s\n", persona.id, C.GoString(persona.nombre))
}
```

### Consideraciones de CGo

- **Costo**: cada llamada CGo tiene overhead (~40-100ns). No para bucles calientes.
- **Compilacion cruzada**: CGo complica el cross-compiling.
- **Portabilidad**: pierdes la portabilidad pura de Go.
- **Alternativa**: considera reimplementar en Go puro si es factible.

---

## 15.4 Build Tags

Los build tags controlan que archivos se incluyen en la compilacion segun condiciones:

```go
// archivo_windows.go
//go:build windows

package main

import "fmt"

func SistemaOperativo() string {
    return "Windows"
}

func separadorRuta() string {
    return "\\"
}
```

```go
// archivo_unix.go
//go:build linux || darwin

package main

import "fmt"

func SistemaOperativo() string {
    return "Unix/Linux"
}

func separadorRuta() string {
    return "/"
}
```

```go
// main.go
package main

import "fmt"

func main() {
    fmt.Println("SO:", SistemaOperativo())
    fmt.Println("Separador:", separadorRuta())
}
```

### Compilar con build tags

```bash
# Compilar para el SO actual (selecciona automaticamente)
go build

# Compilar especificando tags
go build -tags=integration

# Compilar para otro SO (cross-compile)
GOOS=linux GOARCH=amd64 go build
GOOS=windows GOARCH=amd64 go build
```

### Tags comunes

```go
//go:build integration       // Tests de integracion
//go:build !race             // Excluir cuando -race esta activo
//go:build linux && amd64    // Combinacion de SO y arquitectura
//go:build go1.18            // Version minima de Go
//go:build ignore            // Ignorar completamente el archivo
```

---

## 15.5 Embedding de Archivos (embed)

Desde Go 1.16, puedes embeber archivos directamente en el binario:

```go
package main

import (
    "embed"
    "fmt"
    "net/http"
)

//go:embed static/*
var staticFiles embed.FS

//go:embed templates/*.html
var templates embed.FS

//go:embed config.yaml
var configBytes []byte

//go:embed version.txt
var version string

func main() {
    // Servir archivos estaticos
    http.Handle("/static/", http.FileServer(http.FS(staticFiles)))

    // Leer configuracion embebida
    fmt.Println("Config:", string(configBytes))
    fmt.Println("Version:", version)

    // Leer archivo especifico
    contenido, _ := templates.ReadFile("templates/index.html")
    fmt.Println("Template:", string(contenido))

    // Listar archivos embebidos
    entries, _ := templates.ReadDir("templates")
    for _, e := range entries {
        fmt.Println("  -", e.Name())
    }

    http.ListenAndServe(":8080", nil)
}
```

### Combinar con build tags

```go
// config_dev.go
//go:build !production

package main

import _ "embed"

//go:embed config.dev.yaml
var configBytes []byte
```

```go
// config_prod.go
//go:build production

package main

import _ "embed"

//go:embed config.prod.yaml
var configBytes []byte
```

---

## 15.6 Profiling y Optimizacion

### CPU Profiling

```go
package main

import (
    "fmt"
    "os"
    "runtime/pprof"
)

func main() {
    // Iniciar CPU profile
    f, _ := os.Create("cpu.prof")
    defer f.Close()
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    // Codigo a perfilar
    for i := 0; i < 1000000; i++ {
        fmt.Sprintf("numero %d", i)
    }
}
```

```bash
# Analizar perfil de CPU
go tool pprof cpu.prof

# Comandos dentro de pprof:
# (pprof) top         # Top funciones por tiempo
# (pprof) list func   # Ver codigo fuente con tiempos
# (pprof) web         # Grafico de llamadas (necesita graphviz)
```

### Memory Profiling

```go
package main

import (
    "os"
    "runtime"
    "runtime/pprof"
)

func main() {
    // Codigo que asigna memoria...

    // Escribir perfil de memoria
    f, _ := os.Create("mem.prof")
    defer f.Close()
    runtime.GC() // Forzar GC antes del perfil
    pprof.WriteHeapProfile(f)
}
```

```bash
# Analizar perfil de memoria
go tool pprof mem.prof

# Ver allocaciones por linea
go tool pprof -alloc_objects mem.prof
```

### Goroutine Profiling

```go
package main

import (
    "net/http"
    _ "net/http/pprof" // Importar para habilitar profiling HTTP
    "runtime"
    "time"
)

func main() {
    // Iniciar servidor de profiling en puerto separado
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // Tu aplicacion...
    for i := 0; i < 100; i++ {
        go func(id int) {
            time.Sleep(time.Duration(id) * time.Second)
        }(i)
    }

    time.Sleep(10 * time.Second)
}
```

```bash
# Acceder a perfiles via HTTP
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Ver en navegador
open http://localhost:6060/debug/pprof/
```

### Benchmarking para optimizacion

```go
package main

import "testing"

// Comparar dos implementaciones
func BenchmarkSprintf(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = fmt.Sprintf("hola %s %d", "mundo", 42)
    }
}

func BenchmarkStringsBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        sb.WriteString("hola ")
        sb.WriteString("mundo ")
        sb.WriteString(strconv.Itoa(42))
        _ = sb.String()
    }
}
```

```bash
# Comparar benchmarks
go test -bench=. -benchmem -count=10 > nuevo.txt
# benchstat viejo.txt nuevo.txt
```

### Consejos de optimizacion

```go
// 1. Pre-alocar slices cuando sabes el tamaño
// MAL: append repetido causa reallocaciones
var resultado []string
for _, item := range items {
    resultado = append(resultado, procesar(item))
}

// BIEN: pre-alocar
resultado := make([]string, 0, len(items))
for _, item := range items {
    resultado = append(resultado, procesar(item))
}

// 2. Usar strings.Builder en vez de concatenacion (+)
var sb strings.Builder
for _, s := range palabras {
    sb.WriteString(s)
}
resultado := sb.String()

// 3. Usar sync.Pool para objetos reutilizables
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

buf := bufferPool.Get().(*bytes.Buffer)
buf.Reset()
defer bufferPool.Put(buf)

// 4. Evitar reflection en codigo caliente
// 5. Usar type assertions en vez de reflection cuando sea posible
// 6. Preferir pasar punteros a structs grandes
```

---

## 15.7 Fuzzing (testing.F)

El fuzzing nativo de Go (desde 1.18) genera entradas aleatorias para encontrar bugs que los tests tradicionales no cubren:

```go
package main

import (
    "testing"
    "unicode/utf8"
)

func ValidarNombre(nombre string) error {
    if nombre == "" {
        return fmt.Errorf("nombre vacio")
    }
    if !utf8.ValidString(nombre) {
        return fmt.Errorf("nombre no es UTF-8 valido")
    }
    if len(nombre) > 100 {
        return fmt.Errorf("nombre demasiado largo")
    }
    return nil
}

// Fuzz test: Go genera entradas aleatorias para encontrar bugs
func FuzzValidarNombre(f *testing.F) {
    // Seeds: casos iniciales para guiar la generacion
    f.Add("Andres")
    f.Add("")
    f.Add("a")
    f.Add(string(make([]byte, 200))) // Muy largo

    f.Fuzz(func(t *testing.T, nombre string) {
        err := ValidarNombre(nombre)
        // Si hay panic, el fuzzer lo detecta y reporta
        // No deberia panic para ninguna entrada
        _ = err
    })
}

// Ejemplo: fuzzing de un parser
func FuzzParserJSON(f *testing.F) {
    f.Add(`{"nombre": "Andres"}`)
    f.Add(`{}`)
    f.Add(`invalido`)

    f.Fuzz(func(t *testing.T, input string) {
        var resultado map[string]interface{}
        // json.Unmarshal nunca deberia panic, solo retornar error
        err := json.Unmarshal([]byte(input), &resultado)
        if err != nil {
            return // Error esperado con entradas invalidas
        }
        // Verificar que el resultado es consistente
        if resultado == nil && input != "null" {
            t.Logf("resultado nil para input no-null: %q", input)
        }
    })
}
```

```bash
# Ejecutar fuzz tests (por tiempo ilimitado hasta encontrar bug)
go test -fuzz=FuzzValidarNombre

# Por tiempo especifico
go test -fuzz=FuzzValidarNombre -fuzztime=30s

# Con carpeta de corpus (guarda entradas interesantes)
go test -fuzz=. -fuzztime=1m

# El corpus se guarda en testdata/fuzz/FuzzValidarNombre/
# Commitea el corpus al repo para que CI tambien fuzzee
```

### Fuzzing en CI

```bash
# En CI, ejecuta fuzzing por tiempo limitado
go test -fuzz=. -fuzztime=30s ./...

# Si el fuzzer encuentra un bug, genera un archivo en testdata/fuzz/
# con la entrada que causo el fallo. El test falla hasta que corrijas el bug.
```

---

## 15.8 Escape Analysis

El compilador de Go decide si una variable vive en el stack o en el heap. Entenderlo ayuda a escribir codigo mas eficiente:

```go
package main

import "fmt"

// Caso 1: NO escapa (stack)
func sumaLocal() int {
    x := 42          // Vive en el stack
    y := 10          // Vive en el stack
    return x + y     // Retorna copia, no referencia
}

// Caso 2: ESCAPA al heap (retorna puntero)
func nuevoUsuario() *Usuario {
    u := Usuario{Nombre: "Andres"}  // u ESCAPA al heap
    return &u  // La referencia sobrevive al return
}

// Caso 3: ESCAPA al heap (interface boxing)
func imprimir(v interface{}) {
    fmt.Println(v)  // v se convierte en interface{} -> heap allocation
}

// Caso 4: ESCAPA al heap (closure captura)
func contador() func() int {
    n := 0            // n ESCAPA al heap
    return func() int {
        n++
        return n      // El closure mantiene viva a n
    }
}

// Caso 5: NO escapa (slice conocido en compilacion)
func sumarSlice() int {
    s := make([]int, 10)  // Puede vivir en stack si el tamaño es constante
    return s[0] + s[1]
}
```

### Como inspeccionar el escape analysis

```bash
# Ver decisiones de escape
go build -gcflags="-m" ./...

# Ejemplo de salida:
# ./main.go:5:2: moved to heap: u
# ./main.go:8:9: ... argument does not escape
# ./main.go:12:13: n escapes to heap

# Flag mas detallada
go build -gcflags="-m -m" ./...

# En tests
go test -gcflags="-m" ./...
```

### Patrones que causan escapes innecesarios

```go
// MAL: interface boxing innecesario
func Procesar(valores []int) {
    for _, v := range valores {
        fmt.Println(v)  // Cada v escapa al heap via interface{}
    }
}

// BIEN: usa funciones tipadas si es posible
func Procesar(valores []int) {
    for _, v := range valores {
        fmt.Println(strconv.Itoa(v))  // string no escapa igual
    }
}

// MAL: retornar puntero a variable de loop
func crearUsuarios(nombres []string) []*Usuario {
    var usuarios []*Usuario
    for _, nombre := range nombres {
        u := Usuario{Nombre: nombre}
        usuarios = append(usuarios, &u)  // Cada u escapa
    }
    return usuarios
}

// BIEN (Go 1.22+): la variable del loop tiene scope por iteracion
// En Go <1.22, necesitabas u := u al inicio del loop
```

---

## 15.9 Structured Logging (log/slog)

Desde Go 1.21, `log/slog` ofrece logging estructurado nativo:

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    // Logger JSON (produccion)
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))

    // Logger texto (desarrollo)
    devLogger := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
    }))

    // Pares clave-valor
    logger.Info("servidor iniciado",
        slog.Int("puerto", 8080),
        slog.String("entorno", "produccion"),
        slog.Bool("debug", false),
    )

    // Grupos anidados
    logger.Info("usuario creado",
        slog.Group("usuario",
            slog.Int("id", 123),
            slog.String("email", "usuario@ejemplo.com"),
        ),
        slog.Duration("tiempo_procesamiento", 150*time.Millisecond),
    )

    // Logs con nivel
    devLogger.Debug("consulta SQL", slog.String("query", "SELECT * FROM ..."))
    devLogger.Warn("reintento de conexion", slog.Int("intento", 3))
    devLogger.Error("fallo al guardar", slog.Any("error", err))

    // Logger por defecto (usa el global)
    slog.Info("mensaje simple")
    slog.LogAttrs(context.Background(), slog.LevelInfo, "con atributos tipados",
        slog.Int("contador", 42),
    )
}
```

### Integrar slog con contexto

```go
// Agregar atributos al contexto
func ProcesarSolicitud(ctx context.Context, solicitudID string) {
    logger := slog.With(
        slog.String("solicitud_id", solicitudID),
        slog.String("servicio", "api-usuarios"),
    )

    logger.Info("iniciando procesamiento")

    usuario, err := buscarUsuario(ctx, "123")
    if err != nil {
        logger.Error("error al buscar usuario",
            slog.String("usuario_id", "123"),
            slog.Any("error", err),
        )
        return
    }

    logger.Info("usuario encontrado",
        slog.String("nombre", usuario.Nombre),
    )
}
```

---

## 15.10 GC Tuning y Gestion de Memoria

### Variables de entorno para GC

```bash
# GOGC: porcentaje de crecimiento del heap que dispara GC
# Valor por defecto: 100 (GC corre cuando el heap duplica su tamaño)
GOGC=50 ./mi-app     # GC mas frecuente, menos memoria, mas CPU
GOGC=200 ./mi-app    # GC menos frecuente, mas memoria, menos CPU
GOGC=off ./mi-app    # Deshabilitar GC (solo para benchmarks)

# GOMEMLIMIT (Go 1.19+): limite suave de memoria total
GOMEMLIMIT=512MiB ./mi-app  # GC se vuelve mas agresivo cerca del limite

# GODEBUG: trazas de GC
GODEBUG=gctrace=1 ./mi-app

# Salida tipica de gctrace:
# gc 1 @0.012s 2%: 0.50+1.2+0.10 ms clock, 2.0+0/1.0/0+0.40 ms cpu,
# 4->4->2 MB, 5 MB goal, 4 P
# Interpretacion: GC #1, a los 0.012s, heap: 4MB antes -> 4MB durante -> 2MB despues
```

### sync.Pool: patrones correctos

```go
package main

import (
    "bytes"
    "sync"
)

// Pool de buffers reutilizables
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func procesarDatos(datos []byte) string {
    // Obtener buffer del pool
    buf := bufferPool.Get().(*bytes.Buffer)

    // IMPORTANTE: resetear antes de usar
    buf.Reset()

    // Devolver al pool al terminar (incluso si hay panic)
    defer bufferPool.Put(buf)

    buf.Write(datos)
    buf.WriteString(" - procesado")

    return buf.String()
}

// Pool por tamaño (evita fragmentacion)
var pools = map[int]*sync.Pool{
    1024:  {New: func() interface{} { return make([]byte, 1024) }},
    4096:  {New: func() interface{} { return make([]byte, 4096) }},
    16384: {New: func() interface{} { return make([]byte, 16384) }},
}

func obtenerBuffer(tamano int) []byte {
    // Encontrar el pool del tamaño mas cercano
    for s, p := range pools {
        if tamano <= s {
            return p.Get().([]byte)
        }
    }
    return make([]byte, tamano) // Muy grande, alocar directamente
}
```

### Memory Ballast (obsoleto desde Go 1.19)

```go
// ANTES (Go <1.19): memory ballast para reducir frecuencia de GC
// func main() {
//     ballast := make([]byte, 100<<20) // 100MB
//     runtime.KeepAlive(ballast)
// }

// AHORA (Go 1.19+): usa GOMEMLIMIT directamente
// export GOMEMLIMIT=200MiB
```

---

## 15.11 Primitivas Avanzadas de Concurrencia

### sync.Cond: esperar condiciones

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Cola struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []string
}

func NuevaCola() *Cola {
    q := &Cola{}
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Cola) Encolar(item string) {
    q.mu.Lock()
    q.items = append(q.items, item)
    q.mu.Unlock()
    q.cond.Signal() // Despertar a UN consumidor
    // q.cond.Broadcast() // Despertar a TODOS los consumidores
}

func (q *Cola) Desencolar() string {
    q.mu.Lock()
    defer q.mu.Unlock()

    for len(q.items) == 0 {
        q.cond.Wait() // Esperar hasta que haya items
    }

    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```

### singleflight: deduplicar peticiones concurrentes

```go
package main

import (
    "fmt"
    "sync"
    "time"

    "golang.org/x/sync/singleflight"
)

var grupo singleflight.Group

func obtenerDatosCostosos(clave string) (interface{}, error) {
    // Si multiples goroutines llaman con la misma clave,
    // solo una ejecuta la funcion; las demas reciben el mismo resultado
    return grupo.Do(clave, func() (interface{}, error) {
        fmt.Println("Ejecutando consulta costosa para:", clave)
        time.Sleep(1 * time.Second) // Simular trabajo
        return fmt.Sprintf("datos_%s", clave), nil
    })
}

func main() {
    var wg sync.WaitGroup

    // 10 goroutines piden la misma clave
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            resultado, _ := obtenerDatosCostosos("usuarios")
            fmt.Println(resultado)
        }()
    }
    wg.Wait()
    // Solo UNA ejecucion de la funcion costosa
}
```

### sync.Map: cuando usarlo

```go
// sync.Map es util SOLO en casos especificos:
// 1. La clave se escribe una vez y se lee muchas veces
// 2. Multiples goroutines leen/escriben claves distintas

var cache sync.Map

func main() {
    // Guardar
    cache.Store("clave1", "valor1")
    cache.Store("clave2", 42)

    // Leer
    if v, ok := cache.Load("clave1"); ok {
        fmt.Println(v)
    }

    // Leer o guardar (atomico)
    v, loaded := cache.LoadOrStore("clave3", "valor3")
    fmt.Println(v, loaded)

    // Eliminar
    cache.Delete("clave1")

    // Iterar y operar
    cache.Range(func(clave, valor interface{}) bool {
        fmt.Printf("%v: %v\n", clave, valor)
        return true // Continuar iterando
    })
}

// En la mayoria de los casos, map + sync.RWMutex es mejor
// Solo usa sync.Map si tienes benchmarks que lo justifican
```

---

## Resumen del Capítulo

- `reflect` permite examinar y manipular tipos en runtime. Usalo con moderacion.
- `unsafe` elude el sistema de tipos. Solo para casos extremos y muy justificados.
- CGo integra codigo C en Go, con costo de rendimiento y portabilidad.
- Build tags (`//go:build`) controlan inclusion de archivos por SO, arquitectura o tags personalizados.
- `embed` incrusta archivos en el binario desde Go 1.16.
- Profiling con `pprof` ayuda a identificar cuellos de botella de CPU, memoria y goroutines.
- Optimiza solo despues de medir: pre-aloca slices, usa `strings.Builder`, evita reflection en bucles calientes.
- **Fuzzing** (`testing.F`) genera entradas aleatorias para encontrar bugs. Commitea el corpus a CI.
- **Escape analysis** (`-gcflags="-m"`) muestra que variables van al heap. Interface boxing y closures son causas comunes.
- **`log/slog`** (Go 1.21+) es el estandar para logging estructurado con JSON, niveles y atributos.
- **GOGC** y **GOMEMLIMIT** controlan la frecuencia del GC. `sync.Pool` reusa objetos y reduce presion.
- **`sync.Cond`** para esperar condiciones, **`singleflight`** para deduplicar y **`sync.Map`** para casos especificos.

En el siguiente capitulo exploraremos el desarrollo web y APIs con Go.

---

← [Capítulo anterior](14-arquitectura-hexagonal.md) | [Inicio](README.md) | [Capítulo siguiente →](16-desarrollo-web.md)
