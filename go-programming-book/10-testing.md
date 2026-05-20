# Capítulo 10: Testing
Go tiene testing integrado en el lenguaje. No necesitas librerias externas para escribir tests unitarios, benchmarks o tests de integracion. La herramienta `go test` y el paquete `testing` son parte de la libreria estandar.
---
## 10.1 Test Unitarios
### Tu primer test
```go
// matematicas.go
package matematicas
func Sumar(a, b int) int {
    return a + b
}
func Dividir(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division por cero")
    }
    return a / b, nil
}
// matematicas_test.go
package matematicas
import "testing"
func TestSumar(t *testing.T) {
    resultado := Sumar(3, 4)
    esperado := 7
    if resultado != esperado {
        t.Errorf("Sumar(3, 4) = %d; esperado %d", resultado, esperado)
    }
}
func TestDividir(t *testing.T) {
    resultado, err := Dividir(10, 2)
    if err != nil {
        t.Fatalf("Dividir(10, 2): error inesperado: %v", err)
    }
    if resultado != 5 {
        t.Errorf("Dividir(10, 2) = %f; esperado 5", resultado)
    }
}
func TestDividirPorCero(t *testing.T) {
    _, err := Dividir(10, 0)
    if err == nil {
        t.Fatal("Dividir(10, 0): esperaba error, no ocurrio")
    }
}
Ejecutar tests
# Todos los tests del paquete actual
go test
# Con salida detallada
go test -v
# Todos los tests del modulo
go test ./...
# Test especifico por nombre
go test -run TestSumar
# Test por patron (regex)
go test -run "TestSumar|TestDividir$"
# Con cobertura
go test -cover ./...
# Generar perfil de cobertura
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out  # Abre en navegador
# Detectar data races
go test -race ./...
# Paralelizar tests (por defecto GOMAXPROCS)
go test -parallel 4 ./...
### Convenciones de archivos de test

```
mi-paquete/
  usuario.go         # Codigo fuente
  usuario_test.go    # Tests (mismo paquete)
  helpers.go
  helpers_test.go
  # Archivos de integracion (build tag)
  usuario_integration_test.go  # //go:build integration
```
10.2 Tests de Tabla (Table-Driven Tests)
El patron mas comun en Go. Cada caso de prueba es una entrada en un slice:
package calculadora
import "testing"
func Sumar(a, b int) int {
    return a + b
}
func TestSumar(t *testing.T) {
    // Cada caso: nombre, entradas, esperado
    casos := []struct {
        nombre   string
        a, b     int
        esperado int
    }{
        {"numeros positivos", 3, 4, 7},
        {"cero mas positivo", 0, 5, 5},
        {"negativo mas positivo", -3, 5, 2},
        {"dos negativos", -3, -4, -7},
        {"cero mas cero", 0, 0, 0},
        {"numero grande", 1000000, 2000000, 3000000},
    }
    for _, c := range casos {
        t.Run(c.nombre, func(t *testing.T) {
            resultado := Sumar(c.a, c.b)
            if resultado != c.esperado {
                t.Errorf("Sumar(%d, %d) = %d; esperado %d",
                    c.a, c.b, resultado, c.esperado)
            }
        })
    }
}
Test de tabla con errores
func TestValidarEmail(t *testing.T) {
    casos := []struct {
        nombre    string
        email     string
        esperaErr bool
        mensaje   string // Mensaje esperado en el error
    }{
        {
            nombre:    "email valido",
            email:     "usuario@ejemplo.com",
            esperaErr: false,
        },
        {
            nombre:    "sin arroba",
            email:     "usuario",
            esperaErr: true,
            mensaje:   "el email debe contener @",
        },
        {
            nombre:    "vacio",
            email:     "",
            esperaErr: true,
            mensaje:   "email no puede estar vacio",
        },
        {
            nombre:    "sin dominio",
            email:     "usuario@",
            esperaErr: true,
        },
        {
            nombre:    "subdominios",
            email:     "usuario@sub.dominio.com",
            esperaErr: false,
        },
    }
    for _, c := range casos {
        t.Run(c.nombre, func(t *testing.T) {
            err := ValidarEmail(c.email)
            if c.esperaErr && err == nil {
                t.Errorf("ValidarEmail(%q): esperaba error, no ocurrio", c.email)
            }
            if !c.esperaErr && err != nil {
                t.Errorf("ValidarEmail(%q): error inesperado: %v", c.email, err)
            }
        })
    }
}
Test de tabla con setup complejo
func TestServicioUsuario(t *testing.T) {
    // Setup compartido
    repo := NuevoRepositorioMemoria()
    svc := NuevoServicioUsuario(repo)
    casos := []struct {
        nombre   string
        entrada  CrearUsuarioInput
        esperado *Usuario
        errEsperado error
    }{
        {
            nombre:  "crear usuario valido",
            entrada: CrearUsuarioInput{Nombre: "Andres", Email: "a@test.com"},
            esperado: &Usuario{ID: 1, Nombre: "Andres", Email: "a@test.com"},
        },
        {
            nombre:   "email duplicado",
            entrada:  CrearUsuarioInput{Nombre: "Otro", Email: "a@test.com"},
            errEsperado: ErrEmailDuplicado,
        },
        {
            nombre:   "nombre vacio",
            entrada:  CrearUsuarioInput{Nombre: "", Email: "b@test.com"},
            errEsperado: ErrNombreRequerido,
        },
    }
    for _, c := range casos {
        t.Run(c.nombre, func(t *testing.T) {
            u, err := svc.Crear(c.entrada)
            if c.errEsperado != nil {
                if !errors.Is(err, c.errEsperado) {
                    t.Errorf("error esperado %v, obtenido %v", c.errEsperado, err)
                }
                return
            }
            if err != nil {
                t.Fatalf("error inesperado: %v", err)
            }
            if u.Nombre != c.esperado.Nombre || u.Email != c.esperado.Email {
                t.Errorf("usuario = %+v; esperado %+v", u, c.esperado)
            }
        })
    }
}
10.3 Helpers y t.Helper()
Marca tus funciones auxiliares con t.Helper() para que los errores apunten al test, no al helper:
package usuario
import (
    "errors"
    "testing"
)
func TestCrearUsuario(t *testing.T) {
    u, err := CrearUsuario("Andres", "a@test.com")
    verificarSinError(t, err)
    verificarIgual(t, "nombre", "Andres", u.Nombre)
    verificarIgual(t, "email", "a@test.com", u.Email)
}
// Helper: verifica que no haya error. t.Helper() hace que
// los mensajes de error apunten a la linea del test que llama,
// no a esta funcion.
func verificarSinError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("error inesperado: %v", err)
    }
}
func verificarIgual(t *testing.T, campo string, esperado, obtenido string) {
    t.Helper()
    if obtenido != esperado {
        t.Errorf("%s: esperado %q, obtenido %q", campo, esperado, obtenido)
    }
}
func verificarErrorEs(t *testing.T, esperado, obtenido error) {
    t.Helper()
    if !errors.Is(obtenido, esperado) {
        t.Fatalf("error esperado %v, obtenido %v", esperado, obtenido)
    }
}
10.4 Sub-tests con t.Run()
t.Run() crea sub-tests que pueden ejecutarse individualmente y en paralelo:
func TestCalculadora(t *testing.T) {
    t.Run("Suma", func(t *testing.T) {
        t.Parallel() // Ejecutar en paralelo con otros sub-tests
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
    t.Run("Multiplicacion", func(t *testing.T) {
        t.Parallel()
        if Multiplicar(4, 3) != 12 {
            t.Error("multiplicacion incorrecta")
        }
    })
    t.Run("Division", func(t *testing.T) {
        t.Run("normal", func(t *testing.T) {
            r, err := Dividir(10, 2)
            if err != nil || r != 5 {
                t.Error("division normal incorrecta")
            }
        })
        t.Run("por cero", func(t *testing.T) {
            _, err := Dividir(10, 0)
            if err == nil {
                t.Error("esperaba error en division por cero")
            }
        })
    })
}
# Ejecutar un sub-test especifico
go test -run TestCalculadora/Suma
# Ejecutar grupo de sub-tests
go test -run TestCalculadora/Division
10.5 TestMain
TestMain permite setup y teardown global para todos los tests de un paquete:
package database
import (
    "os"
    "testing"
)
var testDB *Database
func TestMain(m *testing.M) {
    // Setup: inicializar recursos
    db, err := Inicializar("postgres://localhost/testdb?sslmode=disable")
    if err != nil {
        fmt.Fprintf(os.Stderr, "no se pudo conectar a BD: %v\n", err)
        os.Exit(1)
    }
    testDB = db
    // Ejecutar migraciones
    if err := testDB.Migrar(); err != nil {
        fmt.Fprintf(os.Stderr, "error en migracion: %v\n", err)
        os.Exit(1)
    }
    // Ejecutar todos los tests
    codigo := m.Run()
    // Teardown: limpiar
    testDB.Cerrar()
    os.Exit(codigo)
}
func TestCrearUsuario(t *testing.T) {
    // Usar testDB aqui
    u, err := testDB.CrearUsuario("Andres")
    if err != nil {
        t.Fatal(err)
    }
    if u.Nombre != "Andres" {
        t.Errorf("nombre incorrecto: %s", u.Nombre)
    }
}
10.6 Fakes, Stubs y Mocks
Go prefiere interfaces para testing. No necesitas librerias de mocking (aunque existen).
Fake: implementacion simplificada
package usuario
import (
    "errors"
    "sync"
)
// Interface que queremos testear
type RepositorioUsuario interface {
    Obtener(id int) (*Usuario, error)
    Guardar(u *Usuario) error
}
// Servicio que usa la interface
type ServicioUsuario struct {
    repo RepositorioUsuario
}
func (s *ServicioUsuario) ObtenerNombre(id int) (string, error) {
    u, err := s.repo.Obtener(id)
    if err != nil {
        return "", fmt.Errorf("ObtenerNombre: %w", err)
    }
    return u.Nombre, nil
}
// Fake: repositorio en memoria para tests
type RepositorioMemoria struct {
    mu       sync.RWMutex
    usuarios map[int]*Usuario
    secuencia int
}
func NuevoRepositorioMemoria() *RepositorioMemoria {
    return &RepositorioMemoria{
        usuarios: make(map[int]*Usuario),
    }
}
func (r *RepositorioMemoria) Obtener(id int) (*Usuario, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    u, ok := r.usuarios[id]
    if !ok {
        return nil, ErrUsuarioNoEncontrado
    }
    return u, nil
}
func (r *RepositorioMemoria) Guardar(u *Usuario) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.secuencia++
    u.ID = r.secuencia
    r.usuarios[u.ID] = u
    return nil
}
// Test usando el fake
func TestObtenerNombre(t *testing.T) {
    repo := NuevoRepositorioMemoria()
    repo.Guardar(&Usuario{Nombre: "Andres"})
    svc := &ServicioUsuario{repo: repo}
    nombre, err := svc.ObtenerNombre(1)
    if err != nil {
        t.Fatal(err)
    }
    if nombre != "Andres" {
        t.Errorf("esperado Andres, obtenido %s", nombre)
    }
}
Stub: respuestas predefinidas
// Stub que retorna valores fijos
type RepositorioStub struct {
    UsuarioAceptar *Usuario
    ErrorARetornar error
    GuardadoRecibido *Usuario
}
func (r *RepositorioStub) Obtener(id int) (*Usuario, error) {
    return r.UsuarioAceptar, r.ErrorARetornar
}
func (r *RepositorioStub) Guardar(u *Usuario) error {
    r.GuardadoRecibido = u
    return r.ErrorARetornar
}
func TestServicioConStub(t *testing.T) {
    t.Run("usuario encontrado", func(t *testing.T) {
        stub := &RepositorioStub{
            UsuarioAceptar: &Usuario{Nombre: "Andres"},
        }
        svc := &ServicioUsuario{repo: stub}
        nombre, err := svc.ObtenerNombre(1)
        if err != nil {
            t.Fatal(err)
        }
        if nombre != "Andres" {
            t.Errorf("esperado Andres, obtenido %s", nombre)
        }
    })
    t.Run("usuario no encontrado", func(t *testing.T) {
        stub := &RepositorioStub{
            ErrorARetornar: ErrUsuarioNoEncontrado,
        }
        svc := &ServicioUsuario{repo: stub}
        _, err := svc.ObtenerNombre(99)
        if !errors.Is(err, ErrUsuarioNoEncontrado) {
            t.Errorf("esperado ErrUsuarioNoEncontrado, obtenido %v", err)
        }
    })
}
Spy: registra llamadas
type RepositorioSpy struct {
    ObtenerLlamadas []int
    Devolver        *Usuario
}
func (r *RepositorioSpy) Obtener(id int) (*Usuario, error) {
    r.ObtenerLlamadas = append(r.ObtenerLlamadas, id)
    if r.Devolver == nil {
        return nil, ErrUsuarioNoEncontrado
    }
    return r.Devolver, nil
}
func TestServicioLlamaRepositorio(t *testing.T) {
    spy := &RepositorioSpy{
        Devolver: &Usuario{Nombre: "Andres"},
    }
    svc := &ServicioUsuario{repo: spy}
    svc.ObtenerNombre(1)
    svc.ObtenerNombre(2)
    svc.ObtenerNombre(1)
    if len(spy.ObtenerLlamadas) != 3 {
        t.Errorf("esperaba 3 llamadas, hubo %d", len(spy.ObtenerLlamadas))
    }
    if spy.ObtenerLlamadas[0] != 1 || spy.ObtenerLlamadas[1] != 2 {
        t.Errorf("llamadas incorrectas: %v", spy.ObtenerLlamadas)
    }
}
10.7 Benchmarks
Los benchmarks miden el rendimiento de funciones:
package calculadora
import (
    "testing"
)
func Sumar(a, b int) int {
    return a + b
}
// Benchmark basico
func BenchmarkSumar(b *testing.B) {
    // b.N lo ajusta el framework para mediciones estables
    for i := 0; i < b.N; i++ {
        Sumar(10, 20)
    }
}
// Benchmark con setup (el timer se pausa durante el setup)
func BenchmarkSumarConSetup(b *testing.B) {
    // Setup costoso
    datos := generarDatosGrandes()
    b.ResetTimer() // Reiniciar timer despues del setup
    for i := 0; i < b.N; i++ {
        procesarDatos(datos)
    }
}
// Benchmark comparando alternativas
func BenchmarkConcatenarStrings(b *testing.B) {
    b.Run("fmt.Sprintf", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = fmt.Sprintf("%s %s %d", "hola", "mundo", 42)
        }
    })
    b.Run("strings.Builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var sb strings.Builder
            sb.WriteString("hola")
            sb.WriteString(" ")
            sb.WriteString("mundo")
            sb.WriteString(" ")
            sb.WriteString(strconv.Itoa(42))
            _ = sb.String()
        }
    })
    b.Run("concatenacion simple", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = "hola" + " " + "mundo" + " " + strconv.Itoa(42)
        }
    })
}
Ejecutar benchmarks
# Todos los benchmarks
go test -bench=. ./...
# Benchmark especifico
go test -bench=BenchmarkSumar
# Con memoria
go test -bench=. -benchmem
# Multiples iteraciones (5 veces, para estabilidad)
go test -bench=. -count=5
# Por tiempo minimo (por defecto 1 segundo)
go test -bench=. -benchtime=5s
# Perfil de CPU durante benchmark
go test -bench=. -cpuprofile=cpu.out
go tool pprof cpu.out
# Perfil de memoria
go test -bench=. -memprofile=mem.out
go tool pprof mem.out
# Comparar benchmarks
go test -bench=. -count=10 > nuevo.txt
# Antes: go test -bench=. -count=10 > viejo.txt
# benchstat viejo.txt nuevo.txt
Tabla de benchmarks
func BenchmarkJSONUnmarshal(b *testing.B) {
    tamanos := []int{100, 1000, 10000, 100000}
    for _, tam := range tamanos {
        b.Run(fmt.Sprintf("tamano_%d", tam), func(b *testing.B) {
            data := generarJSON(tam)
            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                var resultado MiStruct
                json.Unmarshal(data, &resultado)
            }
        })
    }
}
10.8 Fuzzing (Go 1.18+)
El fuzzing genera entradas aleatorias para encontrar bugs:
package usuario
import (
    "testing"
    "unicode/utf8"
)
func ValidarNombre(nombre string) error {
    if nombre == "" {
        return errors.New("nombre vacio")
    }
    if !utf8.ValidString(nombre) {
        return errors.New("nombre no es UTF-8 valido")
    }
    if len(nombre) > 100 {
        return errors.New("nombre demasiado largo")
    }
    return nil
}
// Fuzz test: Go genera entradas aleatorias
func FuzzValidarNombre(f *testing.F) {
    // Seeds: casos iniciales para guiar la generacion
    f.Add("Andres")
    f.Add("")
    f.Add("a")
    f.Add(string(make([]byte, 200))) // Muy largo
    f.Fuzz(func(t *testing.T, nombre string) {
        err := ValidarNombre(nombre)
        // No deberia panic para ninguna entrada
        // Si hay panic, el fuzzer lo detecta
        _ = err
    })
}
# Ejecutar fuzz tests (por tiempo limitado, por defecto)
go test -fuzz=FuzzValidarNombre
# Por tiempo especifico
go test -fuzz=FuzzValidarNombre -fuzztime=30s
# Con carpeta de corpus (guarda entradas que encontraron bugs)
go test -fuzz=. -fuzztime=1m
10.9 Tests de Integracion
Build tags para separar tests
// usuario_integration_test.go
//go:build integration
package usuario
import (
    "os"
    "testing"
)
func TestIntegracionCrearUsuario(t *testing.T) {
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" {
        t.Skip("TEST_DATABASE_URL no configurada, omitiendo test de integracion")
    }
    db, err := ConectarBD(dsn)
    if err != nil {
        t.Fatal(err)
    }
    defer db.Close()
    repo := NuevoRepositorioPostgres(db)
    svc := NuevoServicioUsuario(repo)
    u, err := svc.Crear(CrearUsuarioInput{Nombre: "Andres", Email: "test@test.com"})
    if err != nil {
        t.Fatal(err)
    }
    if u.ID == 0 {
        t.Error("esperaba ID > 0")
    }
}
# Solo tests unitarios (sin build tag integration)
go test ./...
# Solo tests de integracion
go test -tags=integration ./...
# Ejecutar ambos
go test -tags=integration ./...
Short mode: omitir tests lentos
func TestExportacionMasiva(t *testing.T) {
    if testing.Short() {
        t.Skip("omitido en modo corto")
    }
    // Test que tarda mucho...
}
# Omitir tests largos
go test -short ./...
10.10 Ejemplos como Tests
Los ejemplos de documentacion son tests ejecutables:
package matematicas
import "fmt"
func Sumar(a, b int) int {
    return a + b
}
// ExampleSumar se ejecuta como test y aparece en la documentacion
func ExampleSumar() {
    fmt.Println(Sumar(3, 4))
    // Output:
    // 7
}
// ExampleSumar_segundo muestra otro caso
func ExampleSumar_segundo() {
    fmt.Println(Sumar(-1, 1))
    // Output:
    // 0
}
// Example con desorden (orden no determinista)
func Example_conDesorden() {
    for _, v := range []int{1, 2, 3} {
        fmt.Println(v)
    }
    // Unordered output:
    // 1
    // 2
    // 3
}
# Los ejemplos se ejecutan como tests
go test -v
# Tambien aparecen en documentacion
go doc matematicas Sumar
10.11 Cobertura de Codigo
# Generar perfil de cobertura
go test -coverprofile=coverage.out ./...
# Ver cobertura por funcion
go tool cover -func=coverage.out
# Ver cobertura en HTML
go tool cover -html=coverage.out
# Cobertura en CI (umbral no nativo, usa herramienta externa)
go test -cover ./... | grep coverage
10.12 Buenas Practicas de Testing
1. Paquete de test: _test
// usuario_test.go
package usuario_test // Paquete externo, solo ve API publica
import (
    "testing"
    "mi-proyecto/usuario"
)
func TestNuevo(t *testing.T) {
    u := usuario.Nuevo("Andres", "a@test.com")
    // Solo puede llamar funciones exportadas
    // No puede acceder a validarEmail() (no exportada)
}
// VENTAJA: pruebas como usuario real de la API
// DESVENTAJA: no puedes probar funciones no exportadas
2. Nombres descriptivos
// BIEN: describe que se prueba y el escenario
func TestServicioUsuario_Crear_CuandoEmailDuplicado_RetornaError(t *testing.T) {}
// ACEPTABLE: con guiones bajos para separar
func TestServicioUsuario_Crear_EmailDuplicado(t *testing.T) {}
// ACEPTABLE: tabla con nombres de caso
func TestCrearUsuario(t *testing.T) {
    t.Run("email duplicado", func(t *testing.T) {})
    t.Run("nombre vacio", func(t *testing.T) {})
}
3. No testees librerias externas
// MAL: testear que la BD funciona
func TestPostgresInsert(t *testing.T) {
    rows, _ := db.Query("INSERT INTO ...")
    // Esto testea PostgreSQL, no tu codigo
}
// BIEN: testear TU codigo asumiendo que la BD funciona
func TestRepositorio_Guardar(t *testing.T) {
    repo := NuevoRepositorioMemoria() // O fake
    err := repo.Guardar(&Usuario{Nombre: "Andres"})
    if err != nil {
        t.Fatal(err)
    }
}
4. Un aserto por test como regla general
// BIEN: un concepto por test
func TestEmail_ConFormatoValido_RetornaNil(t *testing.T) {}
func TestEmail_Vacio_RetornaError(t *testing.T) {}
func TestEmail_SinArroba_RetornaError(t *testing.T) {}
// Para tests de tabla, cada caso prueba un concepto
5. Evita logica compleja en tests
// MAL: tests con bucles, condiciones, calculos
func TestComplejo(t *testing.T) {
    for i := 0; i < len(datos); i++ {
        if i%2 == 0 {
            // ...
        } else if datos[i].activo {
            // ...
        }
    }
}
// BIEN: test lineal y predecible
func TestSimple(t *testing.T) {
    entrada := 5
    esperado := 25
    resultado := Cuadrado(entrada)
    if resultado != esperado {
        t.Errorf("Cuadrado(%d) = %d; esperado %d", entrada, resultado, esperado)
    }
}
## Resumen del Capítulo
- 
go test ejecuta tests con el paquete estandar testing.
- 
Los tests de tabla son el patron idiomatico por excelencia.
- 
t.Run() crea sub-tests que pueden ejecutarse en paralelo.
- 
Fakes, stubs y spies evitan dependencias externas en tests.
- 
Los benchmarks miden rendimiento con testing.B.
- 
El fuzzing genera entradas aleatorias para encontrar bugs.
- 
Build tags separan tests unitarios de integracion.
- 
testing.Short() permite omitir tests lentos.
- 
Los ejemplos son tests ejecutables que ademas documentan.
- 
Testea comportamiento, no implementacion. Manten los tests simples.
En el siguiente capitulo exploraremos Generics en Go.

---

← [Capítulo anterior](09-paquetes-modulos.md) | [Inicio](README.md) | [Capítulo siguiente →](11-generics.md)
