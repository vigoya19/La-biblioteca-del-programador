# Capítulo 12: Buenas Prácticas y Go Idiomático

Escribir Go idiomatico va mas alla de conocer la sintaxis. Se trata de adoptar las convenciones, patrones y filosofia que hacen que el codigo Go sea reconocible, mantenible y eficiente. Este capitulo recopila las practicas esenciales respaldadas por la comunidad y el equipo de Go.

---

## 12.1 Convenciones de Nombrado

### Nombres de paquetes

```go
// BIEN: corto, minusculas, una palabra, descriptivo
package http
package json
package usuario
package facturacion

// MAL: guiones bajos, mayusculas, genericos
package mi_paquete     // guion bajo
package Utilidades     // mayuscula
package util           // demasiado generico
package common         // sin significado concreto
```

**Reglas:**
- Todo en minusculas, sin guiones bajos ni camelCase.
- Nombre corto y descriptivo (idealmente una palabra).
- El nombre del directorio debe coincidir con el nombre del paquete.
- Evita nombres como `util`, `common`, `misc`, `helpers`.

### Nombres de variables

```go
// BIEN: corto en scopes pequeños, descriptivo en scopes grandes
for i, v := range items {  // i, v son perfectos aqui
    procesar(v)
}

var contadorDeUsuariosActivos int  // Descriptivo para scope global

// CamelCase para multiples palabras
var maximoReintentos int
var tiempoDeEspera time.Duration

// Acronimos en mayuscula consistente
var urlServicio string      // No: urlServicio
var IDUsuario int           // No: idUsuario
var HTTPCliente *http.Client // No: httpCliente
var DBConn *sql.DB          // No: dbConn
```

### Nombres de funciones y metodos

```go
// Exportadas: mayuscula inicial
func NuevoUsuario(nombre string) *Usuario { ... }
func (u *Usuario) ValidarEmail() error { ... }

// No exportadas: minuscula inicial
func validarFormato(email string) bool { ... }
func (r *Repositorio) ejecutarQuery(sql string) error { ... }

// Getters: sin prefijo Get (convencion Go)
func (u *Usuario) Nombre() string { return u.nombre }  // Bien
func (u *Usuario) GetNombre() string { ... }            // Mal (no es Go)

// Setters: con prefijo Set
func (u *Usuario) SetNombre(n string) { u.nombre = n }
```

### Nombres de interfaces

```go
// Interfaces de un metodo: nombre del metodo + "er"
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}

// Tus propias interfaces
type Validator interface {
    Validate() error
}
type Serializer interface {
    Serialize() ([]byte, error)
}

// Interfaces multi-metodo: nombre descriptivo
type RepositorioUsuario interface {
    BuscarPorID(id int) (*Usuario, error)
    Guardar(u *Usuario) error
    Eliminar(id int) error
}
```

---

## 12.2 Organizacion de Codigo

### Estructura de archivos

```go
// usuario/usuario.go
package usuario

// 1. Importaciones (agrupadas: stdlib, externas, internas)
import (
    "errors"
    "fmt"

    "github.com/google/uuid"

    "mi-proyecto/internal/validacion"
)

// 2. Tipos
type Usuario struct {
    ID    uuid.UUID
    Email string
}

type ServicioUsuario struct {
    repo RepositorioUsuario
}

// 3. Interfaces (si se definen aqui)
type RepositorioUsuario interface {
    BuscarPorEmail(email string) (*Usuario, error)
    Guardar(u *Usuario) error
}

// 4. Constructores
func NuevoServicioUsuario(repo RepositorioUsuario) *ServicioUsuario {
    return &ServicioUsuario{repo: repo}
}

// 5. Metodos publicos (exportados)
func (s *ServicioUsuario) Registrar(email string) (*Usuario, error) {
    // ...
}

func (s *ServicioUsuario) Buscar(id uuid.UUID) (*Usuario, error) {
    // ...
}

// 6. Metodos privados (no exportados)
func (s *ServicioUsuario) validarEmail(email string) error {
    // ...
}

// 7. Funciones auxiliares
func normalizarEmail(email string) string {
    // ...
}
```

### Principio de proximidad

Agrupa codigo relacionado y mantenlo cerca:

```go
// BIEN: variables, constantes y tipos relacionados juntos
type Config struct {
    Puerto    int
    Host      string
    MaxConex   int
}

var ConfigPorDefecto = Config{
    Puerto:   8080,
    Host:     "localhost",
    MaxConex: 100,
}

// MAL: definiciones dispersas sin orden aparente
var MaxConex = 100
type Config struct { ... }
var puertoPorDefecto = 8080
```

---

## 12.3 Effective Go y Code Review Comments

### Errores comunes en code review

**1. Context como primer parametro**

```go
// BIEN: context siempre es el primer parametro
func Procesar(ctx context.Context, id int) error { ... }

// MAL: context en otra posicion
func Procesar(id int, ctx context.Context) error { ... }
```

**2. No guardar context en structs**

```go
// MAL: context como campo de struct
type Servicio struct {
    ctx context.Context  // No hagas esto
}

// BIEN: pasar context como parametro
type Servicio struct { ... }
func (s *Servicio) Procesar(ctx context.Context) error { ... }
```

**3. Errores como ultimo retorno**

```go
// BIEN: error siempre al final
func Buscar(id int) (*Usuario, error) { ... }

// MAL: error en otra posicion
func Buscar(id int) (error, *Usuario) { ... }
```

**4. Nil slices vs empty slices**

```go
// Para retornos: nil slice es preferible
func BuscarUsuarios() []Usuario {
    return nil  // Bien: el llamante puede hacer range sin problemas
}

// Para campos de struct en JSON: empty slice si quieres [] en vez de null
type Respuesta struct {
    Usuarios []Usuario `json:"usuarios"`
}
resp := Respuesta{
    Usuarios: []Usuario{},  // Serializa como "usuarios": []
}
```

**5. Mutex por valor**

```go
// MAL: Mutex copiado (go vet lo detecta)
type Contador struct {
    mu sync.Mutex
    n  int
}
func (c Contador) Incrementar() { // Receptor por valor: copia el mutex!
    c.mu.Lock()
    c.n++
    c.mu.Unlock()
}

// BIEN: Mutex con receptor de puntero
func (c *Contador) Incrementar() {
    c.mu.Lock()
    c.n++
    c.mu.Unlock()
}
```

**6. Usar defer para cleanup**

```go
// BIEN: defer asegura que el recurso se libera
func LeerArchivo(ruta string) ([]byte, error) {
    f, err := os.Open(ruta)
    if err != nil {
        return nil, err
    }
    defer f.Close()
    return io.ReadAll(f)
}
```

---

## 12.4 Principios SOLID en Go

### S - Single Responsibility (Responsabilidad Unica)

```go
// MAL: una struct hace demasiado
type Reporte struct { ... }
func (r *Reporte) GenerarDatos() { ... }
func (r *Reporte) FormatearPDF() { ... }
func (r *Reporte) EnviarEmail() { ... }
func (r *Reporte) GuardarEnDisco() { ... }

// BIEN: responsabilidades separadas
type GeneradorDatos struct { ... }
func (g *GeneradorDatos) Generar() (*DatosReporte, error) { ... }

type FormateadorPDF struct { ... }
func (f *FormateadorPDF) Formatear(datos *DatosReporte) ([]byte, error) { ... }

type EnviadorEmail struct { ... }
func (e *EnviadorEmail) Enviar(destino string, pdf []byte) error { ... }
```

### O - Open/Closed (Abierto a extension, cerrado a modificacion)

```go
// Usando interfaces para extension sin modificacion
type Descuento interface {
    Aplicar(precio float64) float64
}

type CalculadoraPrecio struct {
    descuentos []Descuento
}

// Podemos agregar nuevos descuentos sin modificar CalculadoraPrecio
type DescuentoPorcentaje struct {
    Porcentaje float64
}
func (d DescuentoPorcentaje) Aplicar(precio float64) float64 {
    return precio * (1 - d.Porcentaje/100)
}

type DescuentoFijo struct {
    Monto float64
}
func (d DescuentoFijo) Aplicar(precio float64) float64 {
    if precio > d.Monto {
        return precio - d.Monto
    }
    return 0
}
```

### L - Liskov Substitution

En Go con implementacion implicita de interfaces, LSP se respeta automaticamente si la interface es coherente:

```go
type Figura interface {
    Area() float64
}

// Circulo y Rectangulo implementan Figura
// Cualquier codigo que use Figura debe funcionar igual con ambos
func ImprimirArea(f Figura) {
    fmt.Printf("Area: %.2f\n", f.Area())
}
```

### I - Interface Segregation

```go
// MAL: interface demasiado grande
type RepositorioCompleto interface {
    Buscar(id int) (*Entidad, error)
    Listar() ([]*Entidad, error)
    Guardar(e *Entidad) error
    Actualizar(e *Entidad) error
    Eliminar(id int) error
    Respaldar() error       // No todos los repos necesitan esto
    Restaurar() error       // No todos los repos necesitan esto
}

// BIEN: interfaces pequeñas y enfocadas
type Buscador interface {
    Buscar(id int) (*Entidad, error)
}

type Escritor interface {
    Guardar(e *Entidad) error
    Eliminar(id int) error
}

type Repositorio interface {
    Buscador
    Escritor
}
```

### D - Dependency Inversion

```go
// MAL: dependencia directa en implementacion concreta
type Servicio struct {
    db *sql.DB  // Depende de implementacion concreta
}

// BIEN: depende de abstraccion (interface)
type RepositorioUsuario interface {
    Guardar(u *Usuario) error
    Buscar(id int) (*Usuario, error)
}

type Servicio struct {
    repo RepositorioUsuario  // Depende de interface
}

// Ahora podemos pasar cualquier implementacion
repoPostgres := NuevoRepositorioPostgres(db)
repoMemoria := NuevoRepositorioMemoria()
repoMock := NuevoRepositorioMock()

servicio1 := Servicio{repo: repoPostgres}
servicio2 := Servicio{repo: repoMemoria}
servicio3 := Servicio{repo: repoMock}
```

---

## 12.5 Manejo de Dependencias

### Dependency Injection sin frameworks

Go no necesita frameworks de DI. Las dependencias se inyectan manualmente:

```go
package main

import (
    "database/sql"
    "log"
    "net/http"
)

func main() {
    // Construir dependencias
    db, err := sql.Open("postgres", "...")
    if err != nil {
        log.Fatal(err)
    }

    repo := NuevoRepositorioPostgres(db)
    servicio := NuevoServicioUsuario(repo)
    handler := NuevoUsuarioHandler(servicio)

    // Configurar rutas
    mux := http.NewServeMux()
    mux.HandleFunc("/usuarios", handler.Crear)
    mux.HandleFunc("/usuarios/", handler.Obtener)

    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Constructor sencillo con opciones

```go
type Servidor struct {
    Puerto    int
    Timeout   time.Duration
    MaxConex   int
    Logger    Logger
}

type OpcionServidor func(*Servidor)

func ConPuerto(p int) OpcionServidor {
    return func(s *Servidor) { s.Puerto = p }
}

func ConTimeout(t time.Duration) OpcionServidor {
    return func(s *Servidor) { s.Timeout = t }
}

func ConLogger(l Logger) OpcionServidor {
    return func(s *Servidor) { s.Logger = l }
}

func NuevoServidor(opciones ...OpcionServidor) *Servidor {
    s := &Servidor{
        Puerto:  8080,
        Timeout: 30 * time.Second,
        MaxConex: 100,
    }
    for _, opt := range opciones {
        opt(s)
    }
    return s
}

// Uso:
srv := NuevoServidor(
    ConPuerto(9090),
    ConTimeout(60*time.Second),
)
```

---

## 12.6 Linters y Herramientas de Calidad

### golangci-lint

El linter mas completo para Go. Agrupa docenas de linters en una sola herramienta:

```bash
# Instalar
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# Ejecutar
golangci-lint run ./...

# Con configuracion especifica
golangci-lint run -c .golangci.yml

# Arreglar issues automaticamente
golangci-lint run --fix
```

### Archivo de configuracion .golangci.yml

```yaml
linters:
  enable:
    - errcheck      # Verificar errores no manejados
    - gosimple      # Simplificar codigo
    - govet         # Errores comunes (go vet)
    - ineffassign   # Asignaciones no usadas
    - staticcheck   # Analisis estatico avanzado
    - unused        # Codigo no usado
    - gofmt         # Verificar formato
    - goimports     # Verificar imports
    - revive        # Reemplazo de golint
    - misspell      # Errores ortograficos
    - bodyclose     # Cerrar cuerpos HTTP
    - noctx         # http.Request sin context

linters-settings:
  revive:
    rules:
      - name: exported
        severity: warning
      - name: blank-imports
      - name: context-as-argument
      - name: error-return
      - name: error-naming
      - name: error-strings
  gofmt:
    simplify: true

issues:
  exclude-use-default: false
  max-issues-per-linter: 0
  max-same-issues: 0
```

### Otras herramientas esenciales

```bash
# goimports: formatear y organizar imports automaticamente
go install golang.org/x/tools/cmd/goimports@latest
goimports -w .

# gofumpt: formateador mas estricto que gofmt
go install mvdan.cc/gofumpt@latest
gofumpt -l -w .

# staticcheck: analisis estatico avanzado
go install honnef.co/go/tools/cmd/staticcheck@latest
staticcheck ./...

# errcheck: detecta errores ignorados
go install github.com/kisielk/errcheck@latest
errcheck ./...

# govulncheck: escaneo de vulnerabilidades
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

---

## 12.7 Manejo de Errores Idiomatico

```go
// Patron standard: retornar error como ultimo valor
func Dividir(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division por cero")
    }
    return a / b, nil
}

// Siempre manejar el error inmediatamente
resultado, err := Dividir(10, 0)
if err != nil {
    log.Printf("Error al dividir: %v", err)
    return
}

// Agregar contexto al propagar
func ProcesarArchivo(ruta string) error {
    data, err := os.ReadFile(ruta)
    if err != nil {
        return fmt.Errorf("ProcesarArchivo(%s): %w", ruta, err)
    }
    // ...
    return nil
}

// Sentinels para errores esperados
var ErrNoEncontrado = errors.New("recurso no encontrado")

// Usar errors.Is para verificar errores envueltos
if errors.Is(err, ErrNoEncontrado) {
    // Manejar caso especifico
}
```

---

## 12.8 Concurrente Idiomatico

```go
// Preferir channels para comunicacion entre goroutines
func procesarItems(items []string) []Resultado {
    ch := make(chan Resultado, len(items))
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)
        go func(item string) {
            defer wg.Done()
            ch <- procesarItem(item)
        }(item)
    }

    go func() {
        wg.Wait()
        close(ch)
    }()

    var resultados []Resultado
    for r := range ch {
        resultados = append(resultados, r)
    }
    return resultados
}

// Usar Mutex para proteger estado compartido (cuando channels no aplican)
type CacheSeguro struct {
    mu    sync.RWMutex
    items map[string]string
}

func (c *CacheSeguro) Obtener(k string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[k]
    return v, ok
}
```

---

## 12.9 Testing Best Practices

### Table-Driven Tests (el patron mas importante)

```go
func TestValidarEmail(t *testing.T) {
    casos := []struct {
        nombre    string
        email     string
        esperaErr bool
    }{
        {"email valido", "usuario@ejemplo.com", false},
        {"sin arroba", "usuario", true},
        {"vacio", "", true},
        {"sin dominio", "usuario@", true},
        {"dominio sin punto", "usuario@dominio", true},
        {"subdominios", "usuario@sub.dominio.com", false},
    }

    for _, c := range casos {
        t.Run(c.nombre, func(t *testing.T) {
            err := ValidarEmail(c.email)
            if (err != nil) != c.esperaErr {
                t.Errorf("ValidarEmail(%q) error = %v, esperaErr = %v",
                    c.email, err, c.esperaErr)
            }
        })
    }
}
```

### t.Helper() en funciones auxiliares

```go
// Sin t.Helper(): el error apunta al helper, no al test
func verificarError(t *testing.T, err error) {
    if err != nil {
        t.Fatalf("error inesperado: %v", err) // Linea del helper
    }
}

// Con t.Helper(): el error apunta al test que llama al helper
func verificarError(t *testing.T, err error) {
    t.Helper()  // <- Esto es clave
    if err != nil {
        t.Fatalf("error inesperado: %v", err) // Linea del test!
    }
}

func TestCrearUsuario(t *testing.T) {
    u, err := CrearUsuario("Andres", "a@test.com")
    verificarError(t, err) // Si falla, apunta AQUI, no al helper
}
```

### t.Parallel() y sub-tests

```go
func TestOperaciones(t *testing.T) {
    // Sub-tests que pueden ejecutarse en paralelo
    t.Run("Suma", func(t *testing.T) {
        t.Parallel() // Este sub-test corre en paralelo con otros
        if Sumar(2, 3) != 5 {
            t.Error("suma incorrecta")
        }
    })

    t.Run("Resta", func(t *testing.T) {
        t.Parallel()
        if Restar(5, 3) != 2 {
            t.Error("resta incorrecta")
        }
    })

    t.Run("Division", func(t *testing.T) {
        t.Run("normal", func(t *testing.T) {
            _, err := Dividir(10, 2)
            if err != nil {
                t.Error(err)
            }
        })
        t.Run("por cero", func(t *testing.T) {
            _, err := Dividir(10, 0)
            if err == nil {
                t.Error("esperaba error")
            }
        })
    })
}
```

### t.Cleanup vs defer

```go
func TestConRecursos(t *testing.T) {
    // t.Cleanup: ideal para recursos de test (se ejecuta al terminar el test)
    dir := t.TempDir() // Creado y eliminado automaticamente

    db, err := sql.Open("postgres", dsn)
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() {
        db.Close() // Se ejecuta al final del test, incluso si falla
    })

    // defer: se ejecuta al salir de la funcion
    // Usa defer para recursos de la funcion, t.Cleanup para recursos del test
}
```

### Convencion testdata/

```
mi-paquete/
  usuario.go
  usuario_test.go
  testdata/
    usuario_valido.json
    usuario_invalido.json
    golden_output.txt
```

```go
func TestConDatosExternos(t *testing.T) {
    data, err := os.ReadFile("testdata/usuario_valido.json")
    if err != nil {
        t.Fatal(err)
    }
    // Usar data en el test...
}
```

---

## 12.10 Project Layout y Diseno de Paquetes

### Estructura de directorios estandar

```
mi-proyecto/
├── cmd/                  # Puntos de entrada (uno por binario)
│   ├── servidor/main.go
│   ├── trabajador/main.go
│   └── cli/main.go
├── internal/             # Codigo privado al modulo
│   ├── dominio/          # Reglas de negocio
│   ├── aplicacion/       # Casos de uso
│   └── infraestructura/  # Adaptadores externos
├── pkg/                  # Codigo reutilizable PUBLICO (debatible: omite si es API interna)
├── api/                  # Especificaciones (OpenAPI, proto)
├── scripts/              # Scripts de build, CI, migracion
├── configs/              # Archivos de configuracion
├── testdata/             # Datos para tests
├── go.mod
├── go.sum
└── Makefile              # Tareas comunes
```

### El debate pkg/

```go
// La intencion original de pkg/: codigo que otras personas PUEDEN importar
// Muchos proyectos Go exitosos NO usan pkg/ (CockroachDB, Moby)
// Si todo tu codigo es interno, no necesitas pkg/

// REGLA: Si no sabes si poner algo en pkg/, ponlo en internal/ primero.
// Es mas facil mover codigo de internal/ a pkg/ que al reves.

// Pkg/ tiene sentido cuando:
// - Tu modulo es una libreria publica con sub-paquetes reutilizables
// - Tienes utilidades que otros proyectos importan

// Pkg/ NO tiene sentido cuando:
// - Todo es interno a tu aplicacion
// - No tienes intencion de que otros importen tu codigo
```

### Paquetes planos vs jerarquicos

```go
// PLANO: pocos paquetes con muchas responsabilidades
// Bien para proyectos pequeños (<10 archivos)
mi-app/
  main.go
  handler.go
  database.go
  models.go

// JERARQUICO: muchos paquetes, cada uno con una responsabilidad
// Bien para proyectos medianos/grandes
mi-app/
  internal/
    usuario/usuario.go
    orden/orden.go
    pago/pago.go
```

### Cohesion de paquetes

```go
// BIEN: paquete cohesionado (todo sobre usuarios)
package usuario
// usuario.go, repositorio.go, servicio.go, errores.go

// MAL: paquete por tipo de componente (baja cohesion)
package models    // Solo structs
package handlers  // Solo HTTP handlers
package repos     // Solo acceso a datos
// Estos paquetes siempre se importan juntos => deberian ser uno solo
```

---

## 12.11 Anti-Patrones de Nombrado de Paquetes

```go
// NUNCA uses estos nombres de paquete:
package util      // Demasiado generico, se convierte en un cajon de sastre
package common    // Igual que util, sin significado
package base      // ¿Base de que?
package helper    // Las funciones deberian tener un hogar con significado
package misc      // Miscellaneous = "no supe donde poner esto"
package global    // Las variables globales son un anti-patron

// QUE HACER en su lugar:
// Si tienes una funcion ParseDate(), preguntate: ¿a que dominio pertenece?
// Si es sobre fechas, crea un paquete "fecha" o "calendario"
// Si es sobre formato, tal vez va en "formato" o junto al codigo que la usa

// Ejemplo: en vez de util/strings.go
// Crea paquetes con proposito:
package texto      // Para manipulacion de texto
package validacion // Para validadores
package formato    // Para formateo de datos

// Nombres que roban identificadores utiles
package url   // Ahora no puedes tener variable "url"
package time  // Conflicto con la stdlib
package error // Conflicto con el tipo error
```

---

## 12.12 Concurrencia: Patrones y Errores Comunes

### Channel ownership: el creador cierra

```go
// REGLA: Quien crea el channel, lo cierra
// El consumidor solo lee y maneja el cierre via range o comma-ok

func productor() <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)  // EL CREADOR CIERRA
        for i := 0; i < 10; i++ {
            ch <- i
        }
    }()
    return ch
}

func consumidor(ch <-chan int) {
    for v := range ch {  // Consumidor: solo lee
        fmt.Println(v)
    }
    // El consumidor NUNCA cierra el channel
}
```

### errgroup: multiples goroutines con errores

```go
import "golang.org/x/sync/errgroup"

func procesarConcurrente(ctx context.Context, items []string) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, item := range items {
        item := item // Capturar para closure
        g.Go(func() error {
            return procesarItem(ctx, item)
        })
    }

    // Espera a todas y retorna el primer error
    return g.Wait()
}
```

### Semaforo con buffered channel

```go
// Limitar concurrencia a N goroutines simultaneas
func procesarConLimite(items []string, maxConcurrent int) {
    semaforo := make(chan struct{}, maxConcurrent)
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)
        go func(item string) {
            defer wg.Done()
            semaforo <- struct{}{}        // Adquirir
            defer func() { <-semaforo }() // Liberar
            procesarItem(item)
        }(item)
    }
    wg.Wait()
}
```

### Goroutine leak prevention

```go
// MAL: goroutine puede quedarse bloqueada para siempre
func lanzarOperacion() {
    ch := make(chan string)
    go func() {
        resultado := operacionLarga()  // Podria tardar horas
        ch <- resultado  // Bloqueado si nadie lee
    }()

    select {
    case r := <-ch:
        fmt.Println(r)
    case <-time.After(5 * time.Second):
        fmt.Println("Timeout")
        return  // GOROUTINE LEAK: la goroutine sigue viva esperando enviar
    }
}

// BIEN: usar context para cancelar
func lanzarOperacion(ctx context.Context) {
    ch := make(chan string, 1) // Buffer de 1 para envio no bloqueante
    go func() {
        resultado := operacionLarga()
        select {
        case ch <- resultado:
        case <-ctx.Done():
            return // Cancelada, no se bloquea
        }
    }()

    select {
    case r := <-ch:
        fmt.Println(r)
    case <-ctx.Done():
        fmt.Println("Cancelado")
    }
}
```

---

## 12.13 Code Review Checklist

Usa esta lista en cada code review para detectar problemas comunes:

| # | Verificacion | Busca |
|---|--------------|-------|
| 1 | **Errores manejados** | Ningun `_` ignorando errores; `err != nil` siempre verificado |
| 2 | **Context propagado** | `ctx` como primer parametro; no guardado en structs |
| 3 | **Defer para cleanup** | Archivos, locks, conexiones cerradas con `defer` |
| 4 | **Mutex por puntero** | `sync.Mutex` nunca copiado (receptor por valor) |
| 5 | **Nil slice preferido** | `return nil` en vez de `return []T{}` para errores |
| 6 | **Interfaces pequeñas** | 1-3 metodos; segregadas si son mas grandes |
| 7 | **Constructores retornan structs** | No interfaces (salvo multiples implementaciones) |
| 8 | **Nombres de paquete validos** | Sin `util`, `common`, `helper`; todo en minusculas |
| 9 | **Acronimos consistentes** | `ID`, `URL`, `HTTP` en mayuscula |
| 10 | **Sin log.Fatal en librerias** | Solo en `main()` o `init()` |
| 11 | **Errores con contexto** | `fmt.Errorf("...: %w", err)` al propagar |
| 12 | **Mensajes de error en minuscula** | Sin puntuacion final ni "error:" |
| 13 | **Getters sin Get** | `Nombre()` no `GetNombre()` |
| 14 | **Channel ownership** | El creador cierra el channel |
| 15 | **Goroutine leaks** | Toda goroutine tiene una forma de terminar |

---

## Resumen del Capítulo

- Los paquetes usan nombres cortos, en minusculas, sin guiones bajos.
- Las variables y funciones siguen camelCase; acronimos en mayuscula consistente (`ID`, `URL`, `HTTP`).
- Los getters no usan prefijo `Get`. Los setters si usan `Set`.
- Las interfaces de un metodo se nombran con el sufijo `-er` (`Reader`, `Writer`).
- `context.Context` es siempre el primer parametro; nunca se guarda en structs.
- Los errores son siempre el ultimo valor de retorno.
- Aplica SOLID: interfaces pequeñas, single responsibility, dependency inversion.
- Inyecta dependencias manualmente; Go no necesita frameworks de DI.
- Usa `golangci-lint` en CI para mantener calidad de codigo.
- Formatea automaticamente con `gofmt`/`gofumpt`.
- Table-driven tests con `t.Run` y `t.Helper()` son el estandar.
- `internal/` impide importaciones externas; `cmd/` organiza binarios.
- El creador del channel lo cierra; usa `errgroup` para goroutines con errores.
- Evita paquetes `util`, `common`, `helper`; prefiere nombres con significado.
- Usa la checklist de code review para detectar problemas comunes sistematicamente.

---

← [Capítulo anterior](11-generics.md) | [Inicio](README.md) | [Capítulo siguiente →](13-patrones-de-diseno.md)
