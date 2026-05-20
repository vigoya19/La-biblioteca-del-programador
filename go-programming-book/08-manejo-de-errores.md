# Capítulo 8: Manejo de Errores

Go no tiene excepciones. En su lugar, trata los errores como valores que se retornan y se verifican explicitamente. Esta filosofia hace que el flujo de errores sea visible y explicito en el codigo.

> "Errors are values" — Rob Pike

---

## 8.1 El Patron error en Go

El tipo `error` es una interface con un solo metodo:

```go
type error interface {
    Error() string
}
```

Cualquier tipo que implemente `Error() string` es un error. Esto hace que crear y manejar errores sea simple y extensible.

### Retornar y manejar errores

```go
package main

import (
    "errors"
    "fmt"
    "os"
)

func dividir(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division por cero")
    }
    return a / b, nil
}

func leerArchivo(nombre string) (string, error) {
    data, err := os.ReadFile(nombre)
    if err != nil {
        return "", err // Propagar el error hacia arriba
    }
    return string(data), nil
}

func main() {
    // Patron idiomatico: verificar el error inmediatamente
    resultado, err := dividir(10, 0)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Resultado:", resultado)

    // Inline if con scope limitado
    if contenido, err := leerArchivo("datos.txt"); err != nil {
        fmt.Println("Error al leer archivo:", err)
    } else {
        fmt.Println("Contenido:", contenido)
    }
}
```

### Errores como valores vs excepciones

```go
// En lenguajes con excepciones:
// try {
//     resultado = dividir(10, 0)
// } catch (Exception e) {
//     // Manejar error
// }

// En Go: el error es parte explicita del flujo
resultado, err := dividir(10, 0)
if err != nil {
    // El error se maneja aqui, no en otro bloque separado
}
```

**Ventaja del patron error**: el flujo de error es visible. No hay "magia" de saltos de stack como con excepciones. Sabes exactamente por donde pasa el error.

---

## 8.2 Crear Errores

### errors.New y fmt.Errorf

```go
package main

import (
    "errors"
    "fmt"
)

// Error simple con mensaje fijo
var ErrUsuarioNoEncontrado = errors.New("usuario no encontrado")
var ErrPermisoDenegado = errors.New("permiso denegado")

func buscarUsuario(id int) (string, error) {
    if id <= 0 {
        return "", fmt.Errorf("id invalido: %d", id)
    }
    if id > 100 {
        return "", ErrUsuarioNoEncontrado
    }
    return fmt.Sprintf("usuario_%d", id), nil
}

func main() {
    _, err := buscarUsuario(-5)
    if err != nil {
        fmt.Println(err) // id invalido: -5
    }

    _, err = buscarUsuario(200)
    if err != nil {
        fmt.Println(err) // usuario no encontrado
    }
}
```

### Errores personalizados

Crear tus propios tipos de error permite agregar contexto y metadatos:

```go
package main

import (
    "fmt"
    "time"
)

// Error con informacion estructurada
type ErrorValidacion struct {
    Campo   string
    Valor   interface{}
    Mensaje string
    Fecha   time.Time
}

func (e *ErrorValidacion) Error() string {
    return fmt.Sprintf("validacion fallida en '%s' con valor '%v': %s",
        e.Campo, e.Valor, e.Mensaje)
}

// Constructor para errores de validacion
func NuevaErrorValidacion(campo string, valor interface{}, mensaje string) error {
    return &ErrorValidacion{
        Campo:   campo,
        Valor:   valor,
        Mensaje: mensaje,
        Fecha:   time.Now(),
    }
}

// Error con codigo de estado HTTP
type ErrorHTTP struct {
    Codigo  int
    Mensaje string
}

func (e *ErrorHTTP) Error() string {
    return fmt.Sprintf("[%d] %s", e.Codigo, e.Mensaje)
}

func procesarFormulario(edad int, email string) error {
    if edad < 0 || edad > 150 {
        return NuevaErrorValidacion("edad", edad, "debe estar entre 0 y 150")
    }
    if email == "" {
        return NuevaErrorValidacion("email", email, "no puede estar vacio")
    }
    return nil
}

func main() {
    err := procesarFormulario(-5, "")
    if err != nil {
        fmt.Println(err)
        // validacion fallida en 'edad' con valor '-5': debe estar entre 0 y 150
    }
}
```

---

## 8.3 Wrapping de Errores

Desde Go 1.13, puedes "envolver" errores para agregar contexto sin perder el error original:

### fmt.Errorf con %w

```go
package main

import (
    "database/sql"
    "errors"
    "fmt"
)

var ErrNoEncontrado = errors.New("registro no encontrado")

func consultarUsuario(db *sql.DB, id int) (string, error) {
    var nombre string
    err := db.QueryRow("SELECT nombre FROM usuarios WHERE id = ?", id).Scan(&nombre)
    if errors.Is(err, sql.ErrNoRows) {
        return "", fmt.Errorf("consultarUsuario(%d): %w", id, ErrNoEncontrado)
    }
    if err != nil {
        return "", fmt.Errorf("consultarUsuario(%d): %w", id, err)
    }
    return nombre, nil
}

func obtenerPerfil(db *sql.DB, id int) error {
    nombre, err := consultarUsuario(db, id)
    if err != nil {
        return fmt.Errorf("obtenerPerfil: %w", err)
    }
    fmt.Println("Perfil de:", nombre)
    return nil
}

func main() {
    // Simulacion del flujo de errores:
    err := obtenerPerfil(nil, 999)
    if err != nil {
        fmt.Println("Error completo:", err)
        // Error completo: obtenerPerfil: consultarUsuario(999): registro no encontrado
    }
}
```

### errors.Is

`errors.Is` recorre la cadena de errores envueltos buscando coincidencia:

```go
package main

import (
    "errors"
    "fmt"
    "os"
)

func abrirArchivo(ruta string) error {
    _, err := os.Open(ruta)
    if err != nil {
        return fmt.Errorf("abrirArchivo(%s): %w", ruta, err)
    }
    return nil
}

func main() {
    err := abrirArchivo("/ruta/inexistente")

    // Verificar si es ErrNotExist en cualquier nivel de wrapping
    if errors.Is(err, os.ErrNotExist) {
        fmt.Println("El archivo no existe")
    }

    // Verificar si es ErrPermission
    if errors.Is(err, os.ErrPermission) {
        fmt.Println("Permiso denegado")
    }

    // Is funciona a traves de la cadena de wrapping
}
```

### errors.As

`errors.As` busca un tipo especifico de error en la cadena de wrapping:

```go
package main

import (
    "errors"
    "fmt"
)

type ErrorRed struct {
    Codigo int
    Host   string
    Causa  error
}

func (e *ErrorRed) Error() string {
    return fmt.Sprintf("error de red [%d] en %s: %v", e.Codigo, e.Host, e.Causa)
}

func conectar(host string) error {
    return &ErrorRed{
        Codigo: 503,
        Host:   host,
        Causa:  fmt.Errorf("timeout de conexion"),
    }
}

func hacerPeticion() error {
    err := conectar("api.ejemplo.com")
    if err != nil {
        return fmt.Errorf("hacerPeticion: %w", err)
    }
    return nil
}

func main() {
    err := hacerPeticion()

    // Extraer ErrorRed si existe en la cadena
    var errRed *ErrorRed
    if errors.As(err, &errRed) {
        fmt.Printf("Error de red - Codigo: %d, Host: %s\n",
            errRed.Codigo, errRed.Host)
        // Error de red - Codigo: 503, Host: api.ejemplo.com
    }
}
```

### La diferencia clave entre Is y As

```go
// errors.Is: compara por valor (sentinel errors)
if errors.Is(err, sql.ErrNoRows) {
    // El error ES ErrNoRows (o lo envuelve)
}

// errors.As: busca por tipo
var pathError *os.PathError
if errors.As(err, &pathError) {
    // err ES un *os.PathError (o lo envuelve)
    fmt.Println("Path:", pathError.Path)
}
```

---

## 8.4 Sentinel Errors

Los sentinel errors son errores predefinidos que actuan como valores centinela:

```go
package main

import (
    "errors"
    "fmt"
)

// Sentinel errors (exportados, documentados)
var (
    ErrUsuarioNoEncontrado = errors.New("usuario no encontrado")
    ErrCuentaBloqueada     = errors.New("cuenta bloqueada")
    ErrFondosInsuficientes = errors.New("fondos insuficientes")
    ErrLimiteExcedido      = errors.New("limite de operaciones excedido")
)

type Cuenta struct {
    ID      int
    Saldo   float64
    Activa  bool
}

func transferir(origen, destino *Cuenta, monto float64) error {
    if !origen.Activa {
        return fmt.Errorf("transferir: %w", ErrCuentaBloqueada)
    }
    if origen.Saldo < monto {
        return fmt.Errorf("transferir: %w", ErrFondosInsuficientes)
    }
    origen.Saldo -= monto
    destino.Saldo += monto
    return nil
}

func main() {
    c1 := &Cuenta{ID: 1, Saldo: 100, Activa: true}
    c2 := &Cuenta{ID: 2, Saldo: 0, Activa: true}

    err := transferir(c1, c2, 200)
    switch {
    case errors.Is(err, ErrFondosInsuficientes):
        fmt.Println("No tienes suficiente saldo")
    case errors.Is(err, ErrCuentaBloqueada):
        fmt.Println("Tu cuenta esta bloqueada")
    case err != nil:
        fmt.Println("Error desconocido:", err)
    default:
        fmt.Println("Transferencia exitosa")
    }
}
```

### Cuando usar sentinel errors

```go
// BIEN: para condiciones esperadas que el llamante debe manejar
if errors.Is(err, io.EOF) {
    // Fin de archivo: condicion normal, no es un "error"
}

if errors.Is(err, sql.ErrNoRows) {
    // Sin resultados: caso esperado en consultas
}

// MAL: para cada posible error
var ErrDivisionPorCero = errors.New("division por cero")    // Usa error inline
var ErrStringVacio = errors.New("string vacio")             // Usa error inline
var ErrNumeroNegativo = errors.New("numero negativo")       // Demasiado granular

// Para errores puntuales que no necesitan ser verificados por el llamante,
// usa fmt.Errorf o errors.New directamente
```

---

## 8.5 Estrategias de Manejo de Errores

### Estrategia 1: Propagacion con contexto

Agrega contexto en cada nivel sin perder el error original:

```go
func capa3() error {
    return errors.New("error en base de datos")
}

func capa2() error {
    err := capa3()
    if err != nil {
        return fmt.Errorf("capa2: %w", err)
    }
    return nil
}

func capa1() error {
    err := capa2()
    if err != nil {
        return fmt.Errorf("capa1: %w", err)
    }
    return nil
}

// Resultado: capa1: capa2: error en base de datos
```

### Estrategia 2: Retry con backoff

```go
package main

import (
    "errors"
    "fmt"
    "time"
)

func operacionFalible() error {
    return errors.New("fallo temporal")
}

func conReintentos(intentos int, espera time.Duration, fn func() error) error {
    var err error
    for i := 0; i < intentos; i++ {
        err = fn()
        if err == nil {
            return nil
        }
        fmt.Printf("Intento %d fallido: %v\n", i+1, err)
        time.Sleep(espera)
        espera *= 2 // Exponential backoff
    }
    return fmt.Errorf("fallaron %d intentos: %w", intentos, err)
}

func main() {
    err := conReintentos(3, 100*time.Millisecond, operacionFalible)
    if err != nil {
        fmt.Println("Operacion fallida definitivamente:", err)
    }
}
```

### Estrategia 3: Loggear y continuar

```go
package main

import (
    "fmt"
    "log"
    "os"
)

func procesarArchivos(nombres []string) {
    for _, nombre := range nombres {
        if err := procesarArchivo(nombre); err != nil {
            log.Printf("AVISO: omitiendo %s: %v", nombre, err)
            continue // Continuar con el siguiente
        }
        fmt.Printf("%s procesado correctamente\n", nombre)
    }
}

func procesarArchivo(nombre string) error {
    data, err := os.ReadFile(nombre)
    if err != nil {
        return fmt.Errorf("no se pudo leer %s: %w", nombre, err)
    }
    fmt.Println(string(data))
    return nil
}
```

### Estrategia 4: Errores fatales (solo para irrecuperables)

```go
package main

import (
    "fmt"
    "log"
    "os"
)

func main() {
    // Solo usar log.Fatal/Fatalf para errores irrecuperables en main/init
    config, err := cargarConfiguracion("config.yaml")
    if err != nil {
        log.Fatalf("Error fatal: no se pudo cargar configuracion: %v", err)
    }
    fmt.Printf("Servidor iniciado en puerto %d\n", config.Puerto)
}

func cargarConfiguracion(ruta string) (*Config, error) {
    data, err := os.ReadFile(ruta)
    if err != nil {
        return nil, fmt.Errorf("cargar config: %w", err)
    }
    // ... parsear config ...
    return &Config{Puerto: 8080}, nil
}

type Config struct {
    Puerto int
}
```

> **Regla**: `log.Fatal` y `os.Exit` solo en `main()` o `init()`. Nunca en librerias.

### Estrategia 5: Defer para manejo de errores en cleanup

```go
package main

import (
    "fmt"
    "io"
    "os"
)

func copiarArchivo(origen, destino string) (err error) {
    src, err := os.Open(origen)
    if err != nil {
        return fmt.Errorf("abrir origen: %w", err)
    }
    defer src.Close()

    dst, err := os.Create(destino)
    if err != nil {
        return fmt.Errorf("crear destino: %w", err)
    }
    defer func() {
        cerrarErr := dst.Close()
        if err == nil {
            err = cerrarErr // Propagar error de cierre si no hubo error antes
        }
    }()

    if _, err = io.Copy(dst, src); err != nil {
        return fmt.Errorf("copiar datos: %w", err)
    }
    return nil
}
```

---

## 8.6 Manejo de Errores por Capas

En aplicaciones reales, el manejo de errores varia segun la capa:

```go
package main

import (
    "database/sql"
    "errors"
    "fmt"
    "net/http"
)

// ===== CAPA DE DOMINIO =====
// Los errores de dominio son semanticos, no tecnicos
var (
    ErrUsuarioNoExiste    = errors.New("el usuario no existe")
    ErrEmailYaRegistrado  = errors.New("el email ya esta registrado")
    ErrContrasenaDebil    = errors.New("la contrasena no cumple los requisitos")
)

type Usuario struct {
    ID    int
    Email string
}

type RepositorioUsuario interface {
    ObtenerPorID(id int) (*Usuario, error)
    ObtenerPorEmail(email string) (*Usuario, error)
    Guardar(u *Usuario) error
}

// ===== CAPA DE INFRAESTRUCTURA =====
// Traduce errores tecnicos a errores de dominio
type RepositorioPostgres struct {
    db *sql.DB
}

func (r *RepositorioPostgres) ObtenerPorID(id int) (*Usuario, error) {
    u := &Usuario{}
    err := r.db.QueryRow("SELECT id, email FROM usuarios WHERE id = $1", id).
        Scan(&u.ID, &u.Email)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrUsuarioNoExiste // Traducir a error de dominio
    }
    if err != nil {
        return nil, fmt.Errorf("error en BD al obtener usuario %d: %w", id, err)
    }
    return u, nil
}

func (r *RepositorioPostgres) Guardar(u *Usuario) error {
    _, err := r.db.Exec("INSERT INTO usuarios (email) VALUES ($1)", u.Email)
    if err != nil {
        // Verificar constraint de unicidad (depende del driver)
        if esErrorDuplicado(err) {
            return ErrEmailYaRegistrado
        }
        return fmt.Errorf("error en BD al guardar usuario: %w", err)
    }
    return nil
}

func esErrorDuplicado(err error) bool {
    // Implementacion especifica del driver de BD
    return false
}

// ===== CAPA DE APLICACION =====
// Orquesta casos de uso y agrega contexto de negocio
type ServicioUsuario struct {
    repo RepositorioUsuario
}

func (s *ServicioUsuario) Registrar(email, contrasena string) error {
    if len(contrasena) < 8 {
        return ErrContrasenaDebil
    }

    _, err := s.repo.ObtenerPorEmail(email)
    if err == nil {
        return ErrEmailYaRegistrado
    }
    if !errors.Is(err, ErrUsuarioNoExiste) {
        return fmt.Errorf("error al verificar email: %w", err)
    }

    u := &Usuario{Email: email}
    return s.repo.Guardar(u)
}

// ===== CAPA DE TRANSPORTE (HTTP) =====
// Convierte errores de dominio a codigos HTTP
func (s *ServicioUsuario) RegistrarHandler(w http.ResponseWriter, r *http.Request) {
    email := r.FormValue("email")
    contrasena := r.FormValue("password")

    err := s.Registrar(email, contrasena)
    if err == nil {
        w.WriteHeader(http.StatusCreated)
        return
    }

    // Mapear errores de dominio a HTTP
    switch {
    case errors.Is(err, ErrEmailYaRegistrado):
        http.Error(w, err.Error(), http.StatusConflict) // 409
    case errors.Is(err, ErrContrasenaDebil):
        http.Error(w, err.Error(), http.StatusBadRequest) // 400
    default:
        // No exponer errores internos al cliente
        fmt.Printf("ERROR INTERNO: %v\n", err)
        http.Error(w, "error interno del servidor", http.StatusInternalServerError) // 500
    }
}
```

---

## 8.7 Errores y Goroutines

El manejo de errores en goroutines requiere atencion especial:

```go
package main

import (
    "errors"
    "fmt"
    "sync"
)

// Patron: channel de errores
func procesarConcurrente(items []string) error {
    var wg sync.WaitGroup
    errores := make(chan error, len(items))
    resultados := make(chan string, len(items))

    for _, item := range items {
        wg.Add(1)
        go func(item string) {
            defer wg.Done()
            resultado, err := procesarItem(item)
            if err != nil {
                errores <- err
                return
            }
            resultados <- resultado
        }(item)
    }

    wg.Wait()
    close(errores)
    close(resultados)

    // Recolectar errores
    var errs []error
    for err := range errores {
        errs = append(errs, err)
    }

    if len(errs) > 0 {
        return fmt.Errorf("errores en procesamiento concurrente: %v", errs)
    }

    for r := range resultados {
        fmt.Println("Resultado:", r)
    }
    return nil
}

func procesarItem(item string) (string, error) {
    if item == "" {
        return "", errors.New("item vacio")
    }
    return "procesado_" + item, nil
}
```

---

## 8.8 Buenas Practicas

1. **Siempre verifica los errores**

```go
// MAL: ignorar errores
data, _ := os.ReadFile("config.yaml")

// BIEN: manejar el error
data, err := os.ReadFile("config.yaml")
if err != nil {
    return fmt.Errorf("no se pudo leer config: %w", err)
}
```

2. **Agrega contexto al propagar**

```go
// MAL: perder contexto
func obtenerUsuario(id int) (*Usuario, error) {
    u, err := db.Query(...)
    if err != nil {
        return nil, err // El llamante no sabe que paso
    }
    return u, nil
}

// BIEN: agregar contexto
func obtenerUsuario(id int) (*Usuario, error) {
    u, err := db.Query(...)
    if err != nil {
        return nil, fmt.Errorf("obtenerUsuario(%d): %w", id, err)
    }
    return u, nil
}
```

3. **No repitas "error" en el mensaje**

```go
// MAL: redundante
return errors.New("error: archivo no encontrado")
return fmt.Errorf("error al abrir archivo: %w", err)

// BIEN: describe que fallo, no que es un error
return errors.New("archivo no encontrado")
return fmt.Errorf("abrir archivo: %w", err)
```

4. **Los mensajes de error en minuscula**

```go
// MAL: mayuscula inicial
return errors.New("Usuario no encontrado")

// BIEN: minuscula (se concatenan con otros mensajes)
return errors.New("usuario no encontrado")
```

5. **No uses panic para errores normales**

```go
// MAL: panic para errores esperados
if err != nil {
    panic(err) // NUNCA hagas esto
}

// BIEN: retorna el error
if err != nil {
    return fmt.Errorf("operacion fallida: %w", err)
}
```

---

## Resumen del Capítulo

- Los errores en Go son valores que se retornan y verifican explicitamente.
- `errors.Is` busca errores por valor en la cadena de wrapping.
- `errors.As` busca errores por tipo en la cadena de wrapping.
- `fmt.Errorf` con `%w` envuelve errores agregando contexto.
- Los sentinel errors son errores predefinidos para condiciones esperadas.
- Los errores personalizados permiten agregar metadatos estructurados.
- Cada capa de la aplicacion maneja errores de forma diferente: dominio, infraestructura, transporte.
- Para errores en goroutines, usa channels dedicados o `errgroup`.
- Los mensajes de error se escriben en minuscula, sin "error:" al inicio, y con contexto claro.

En el siguiente capitulo exploraremos el sistema de paquetes y modulos de Go.

---

← [Capítulo anterior](07-concurrencia.md) | [Inicio](README.md) | [Capítulo siguiente →](09-paquetes-modulos.md)
