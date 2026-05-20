# Capítulo 9: Paquetes y Módulos

Go tiene un sistema de paquetes simple pero poderoso. Todo codigo Go pertenece a un paquete, y los modulos agrupan paquetes en unidades versionables. Entender este sistema es fundamental para organizar proyectos correctamente.

---

## 9.1 El Sistema de Paquetes

En Go, cada archivo pertenece a un paquete declarado en la primera linea. Un paquete es una unidad de organizacion, encapsulacion y reutilizacion.

### Estructura basica

```
mi-proyecto/
  go.mod
  main.go           # package main
  usuario/
    usuario.go       # package usuario
    validator.go     # package usuario (mismo paquete, otro archivo)
  database/
    postgres.go      # package database
    mysql.go         # package database
```

### Declaracion de paquete

```go
// main.go - Punto de entrada del programa
package main

import (
    "fmt"
    "mi-proyecto/usuario"
)

func main() {
    u := usuario.Nuevo("Andres", 30)
    fmt.Println(u.Nombre)
}
```

```go
// usuario/usuario.go
package usuario

type Usuario struct {
    Nombre string
    Edad   int
}

// Exportado: empieza con mayuscula
func Nuevo(nombre string, edad int) *Usuario {
    return &Usuario{Nombre: nombre, Edad: edad}
}

// No exportado: empieza con minuscula, solo visible dentro del paquete
func validarEdad(edad int) bool {
    return edad > 0 && edad < 150
}
```

### Visibilidad (exported vs unexported)

```go
package cuentas

import "sync"

// EXPORTADO: accesible desde otros paquetes
type Cuenta struct {
    ID      int       // Exportado
    Titular string    // Exportado
    saldo   float64   // NO exportado (minuscula)
    mu      sync.Mutex // NO exportado
}

// Exportado
func NuevaCuenta(id int, titular string) *Cuenta {
    return &Cuenta{
        ID:      id,
        Titular: titular,
    }
}

// Exportado
func (c *Cuenta) Depositar(monto float64) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.saldo += monto
}

// Exportado: permite leer sin exponer el campo
func (c *Cuenta) Saldo() float64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.saldo
}

// NO exportado: solo para uso interno del paquete
func (c *Cuenta) validarMonto(monto float64) bool {
    return monto > 0
}
```

> **Regla de visibilidad**: mayuscula = exportado (publico), minuscula = no exportado (privado al paquete). No hay `public`/`private` keywords.

### Nombres de paquetes

```go
// BIEN: corto, descriptivo, minuscula, una palabra
package http
package json
package strings
package usuario
package database

// MAL: con guiones bajos, mayusculas, generico
package mi_paquete     // No uses guiones bajos
package util           // Demasiado generico
package Utilidades     // No uses mayusculas
package common         // Evita nombres vacios de significado

// El nombre del directorio debe coincidir con el package
// directorio: "usuarios/" -> package usuarios
// directorio: "database/" -> package database
```

### Multiples archivos en un paquete

Todos los archivos en un directorio pertenecen al mismo paquete y comparten el mismo namespace:

```go
// usuario/entidad.go
package usuario

type Usuario struct {
    ID    int
    Email string
}
```

```go
// usuario/constructor.go
package usuario

import "errors"

func Nuevo(email string) (*Usuario, error) {
    if email == "" {
        return nil, errors.New("email requerido")
    }
    return &Usuario{Email: email}, nil
}
```

```go
// usuario/metodos.go
package usuario

import "strings"

func (u *Usuario) Normalizar() {
    u.Email = strings.ToLower(u.Email)
}

// Todos los archivos ven las mismas funciones y tipos del paquete
```

---

## 9.2 El paquete main

`main` es un paquete especial: indica un programa ejecutable y debe contener la funcion `main()`:

```go
// cmd/servidor/main.go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // Configurar servidor
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Servidor funcionando\n")
    })

    srv := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    // Iniciar servidor en goroutine
    go func() {
        fmt.Println("Servidor escuchando en :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // Esperar senal de interrupcion
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    fmt.Println("\nApagando servidor...")

    // Graceful shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal(err)
    }
    fmt.Println("Servidor apagado correctamente")
}
```

### Estructura tipica de comandos

```
mi-proyecto/
  cmd/
    servidor/
      main.go       # package main, binario servidor
    cliente/
      main.go       # package main, binario cliente
    migrador/
      main.go       # package main, binario migrador
  internal/
    ...
  go.mod
```

> **Buena practica**: pon los binarios en `cmd/<nombre>/main.go`, no en la raiz del proyecto. Asi puedes tener multiples comandos en un solo modulo.

---

## 9.3 Inicializacion de Paquetes

### Variables a nivel de paquete

```go
package configuracion

import (
    "encoding/json"
    "os"
)

// Las variables de paquete se inicializan antes de init()
var (
    Puerto    = 8080
    NombreApp = "mi-app"
)

// init() se ejecuta automaticamente al importar el paquete
func init() {
    if p := os.Getenv("PUERTO"); p != "" {
        Puerto = parsearPuerto(p)
    }
}

func parsearPuerto(s string) int {
    // ...
    return 0
}
```

### Orden de inicializacion

Para un programa que importa A, B y C:

1. Se inicializan paquetes importados primero (B, C antes que A)
2. Variables de paquete en orden de declaracion
3. Funciones `init()` en orden de declaracion
4. Finalmente, `main()` del paquete main

```go
package main

import (
    "fmt"
    _ "mi-proyecto/config" // El _ importa solo por efectos secundarios (init)
)

var (
    global = inicializarGlobal() // Se ejecuta antes que init()
)

func inicializarGlobal() string {
    fmt.Println("inicializarGlobal")
    return "lista"
}

func init() {
    fmt.Println("init 1")
}

func init() {
    fmt.Println("init 2")
}

func main() {
    fmt.Println("main")
}

// Orden de salida:
// (inicializaciones de paquetes importados primero)
// inicializarGlobal
// init 1
// init 2
// main
```

### Importar solo por efectos secundarios

```go
// El guion bajo importa el paquete solo para ejecutar su init()
import _ "github.com/lib/pq" // Driver PostgreSQL: se registra en init()

// Util para:
// - Drivers de base de datos
// - Registro de codecs
// - Plugins que se auto-registran
```

---

## 9.4 Go Modules

Los modulos son el sistema de gestion de dependencias de Go. Reemplazaron al antiguo GOPATH.

### Crear un modulo

```bash
# Inicializar un modulo nuevo
go mod init github.com/tu-usuario/mi-proyecto

# Esto crea go.mod
```

```go
// go.mod
module github.com/tu-usuario/mi-proyecto

go 1.23

require (
    github.com/gorilla/mux v1.8.1
    github.com/lib/pq v1.10.9
    golang.org/x/sync v0.7.0
)
```

### El archivo go.mod

```
module github.com/tu-usuario/mi-proyecto

go 1.23

require (
    github.com/google/uuid v1.6.0
    github.com/stretchr/testify v1.9.0
)

// Dependencias indirectas (no importadas directamente)
require (
    github.com/davecgh/go-spew v1.1.1 // indirect
    github.com/pmezard/go-difflib v1.0.0 // indirect
)

// Excluir versiones problematicas
exclude github.com/paquete/roto v1.2.3

// Reemplazar dependencias (desarrollo local, forks)
replace github.com/paquete/original => github.com/tu-fork/original v1.0.0

// Reemplazar con ruta local (desarrollo)
replace github.com/tu-usuario/libreria => ../libreria

// Retract: marcar versiones como no recomendadas
retract v1.0.0 // Contenia un bug critico
retract [v1.5.0, v1.5.5] // Rango de versiones retractadas
```

### go.sum

```bash
# go.sum contiene checksums de todas las dependencias
# Verifica integridad: asegura que el codigo no cambio
# NO se edita manualmente
# Se actualiza automaticamente con:
go mod tidy
go get
go mod download
```

### Semantic Import Versioning

A partir de v2, la ruta del modulo incluye la version mayor:

```
// modulo en v1.x.x
module github.com/usuario/libreria
// import "github.com/usuario/libreria"

// modulo en v2.x.x
module github.com/usuario/libreria/v2
// import "github.com/usuario/libreria/v2"

// modulo en v3.x.x
module github.com/usuario/libreria/v3
// import "github.com/usuario/libreria/v3"
```

```go
// Ejemplo practico
package main

import (
    // v1 del paquete
    "github.com/usuario/libreria"

    // v2 del mismo paquete (son modulos distintos)
    "github.com/usuario/libreria/v2"

    // Puedes usar ambos en el mismo archivo con alias
    libv3 "github.com/usuario/libreria/v3"
)

func main() {
    libreria.FuncionV1()
    v2.FuncionV2()
    libv3.FuncionV3()
}
```

---

## 9.5 Gestion de Dependencias

### Agregar dependencias

```bash
# Agregar una dependencia (ultima version)
go get github.com/gorilla/mux

# Agregar version especifica
go get github.com/gorilla/mux@v1.8.1

# Agregar commit especifico
go get github.com/gorilla/mux@abc1234

# Agregar rama
go get github.com/gorilla/mux@main

# Actualizar a la ultima version
go get -u github.com/gorilla/mux

# Actualizar todas las dependencias
go get -u ./...
```

### Remover dependencias no usadas

```bash
# Limpiar go.mod: remueve dependencias no importadas, agrega las faltantes
go mod tidy

# Verificar que todo este consistente
go mod verify
```

### Vendor

```bash
# Crear directorio vendor/ con copias locales de dependencias
go mod vendor

# Compilar usando vendor (ignora el cache de modulos)
go build -mod=vendor

# Util para:
# - CI/CD sin acceso a internet
# - Garantizar que las dependencias no cambian
# - Auditoria de codigo de dependencias
```

### Analizar dependencias

```bash
# Ver el grafo de dependencias
go mod graph

# Ver por que un modulo esta en tus dependencias
go mod why github.com/paquete/indirecto

# Listar modulos y versiones
go list -m all

# Ver modulos con actualizaciones disponibles
go list -m -u all

# Ver versiones disponibles de un modulo
go list -m -versions github.com/paquete
```

---

## 9.6 Paquetes Internos

El directorio `internal/` es especial: solo los paquetes dentro del arbol del modulo pueden importarlo:

```
mi-proyecto/
  go.mod (module github.com/usuario/mi-proyecto)
  internal/
    database/
      postgres.go   // Solo importable por mi-proyecto/...
    auth/
      tokens.go      // Solo importable por mi-proyecto/...
  publico/
    api/
      handler.go     // Puede importar internal/database
  cmd/
    servidor/
      main.go        // Puede importar internal/...
```

```go
// internal/database/postgres.go
package database

// Este paquete solo es visible dentro de github.com/usuario/mi-proyecto
// Nadie externo puede importarlo
type Cliente struct {
    // ...
}

func NuevoCliente(dsn string) *Cliente {
    // ...
    return &Cliente{}
}
```

```go
// publico/api/handler.go
package api

import "github.com/usuario/mi-proyecto/internal/database" // OK: mismo modulo

// cmd/main.go tambien puede importarlo
// Pero otro modulo NO puede hacer:
// import "github.com/usuario/mi-proyecto/internal/database" // ERROR
```

> **Buena practica**: usa `internal/` para codigo que no quieres exponer como API publica. Es la unica forma de encapsulacion real entre paquetes en Go.

---

## 9.7 Organizacion de Proyectos

### Estructura recomendada para proyectos medianos/grandes

```
mi-proyecto/
  cmd/
    servidor/main.go         # Punto de entrada
    trabajador/main.go       # Otro binario
  internal/
    dominio/
      usuario.go             # Entidades y reglas de negocio
      orden.go
    aplicacion/
      servicio_usuario.go    # Casos de uso
      puertos.go             # Interfaces (puertos)
    infraestructura/
      postgres/
        repositorio_usuario.go
      redis/
        cache.go
      http/
        handler_usuario.go
        middleware.go
  pkg/
    paginacion/
      paginacion.go          # Utilidades reutilizables
    errores/
      errores.go
  api/
    openapi.yaml              # Especificacion de API
  configs/
    config.yaml
  scripts/
    migracion.sh
  go.mod
  go.sum
  Makefile
```

### Estructura para librerias

```
mi-libreria/
  usuario.go          # API principal al nivel raiz
  usuario_test.go
  opciones.go         # Functional options
  internal/
    parser/
      parser.go       # Detalles de implementacion ocultos
  go.mod
  README.md
```

### Estructura para proyectos pequeños

```
mi-api/
  main.go
  handler.go
  usuario.go
  database.go
  go.mod
  go.sum
```

> **Consejo**: no sobre-ingenieria la estructura al inicio. Deja que la complejidad dicte la organizacion. Es mas facil mover codigo que mantener una estructura prematura que no encaja.

---

## 9.8 Importaciones y Ciclos

### Reglas de importacion

```go
// 1. No puedes tener dependencias circulares
// Si paquete A importa B, B no puede importar A

// 2. Los paths de import son relativos al modulo
import (
    "github.com/usuario/mi-proyecto/usuario"    // Absoluto (recomendado)
)

// 3. No uses import relativos (go.mod los desaconseja)
// import "../usuario"  // MAL

// 4. Agrupar imports
import (
    // Biblioteca estandar
    "fmt"
    "net/http"

    // Dependencias externas
    "github.com/gorilla/mux"

    // Paquetes del proyecto
    "github.com/usuario/mi-proyecto/usuario"
)
```

### Romper dependencias circulares

```go
// PROBLEMA: A -> B -> A (ciclo)

// SOLUCION 1: Extraer una interface

// tipos/tipos.go
package tipos

type Notificador interface {
    Notificar(mensaje string) error
}

// usuario/usuario.go
package usuario

import "mi-proyecto/tipos"

type Servicio struct {
    notificador tipos.Notificador // Depende de interface, no de implementacion
}

// notificacion/email.go
package notificacion

import "mi-proyecto/tipos"

// Implementa tipos.Notificador
type Email struct { }

func (e *Email) Notificar(mensaje string) error {
    // ...
    return nil
}

// Se resolvio el ciclo:
// usuario -> tipos (interface)
// notificacion -> tipos (implementa interface)
// main ensambla: usuario.Servicio{notificador: &notificacion.Email{}}

// SOLUCION 2: Fusionar paquetes
// Si A y B estan muy acoplados, tal vez deberian ser un solo paquete

// SOLUCION 3: Paquete compartido
// Extraer el codigo comun a un tercer paquete C que ambos importan
```

---

## 9.9 Publicar un Modulo

### Preparar para publicar

```bash
# 1. Asegurar que go.mod esta limpio
go mod tidy

# 2. Ejecutar tests
go test ./...

# 3. Verificar documentacion
go doc -all .

# 4. Formatear
go fmt ./...
go vet ./...
```

### Versionado

```bash
# Crear tag de version
git tag v1.0.0
git push origin v1.0.0

# Version v2 y superiores requieren cambiar el module path
# go.mod:
# module github.com/usuario/libreria/v2
```

### Convenciones de documentacion

```go
// Package math provee constantes y funciones matematicas basicas.
//
// El paquete incluye funciones trigonometricas, logaritmicas y
// de redondeo. Para operaciones con grandes numeros, ver math/big.
package math

// Suma retorna la suma de dos enteros.
//
// Deprecated: Usa SumaAvanzada en su lugar.
func Suma(a, b int) int {
    return a + b
}
```

```bash
# Generar documentacion local
go run golang.org/x/tools/cmd/godoc@latest -http=:6060
# Luego abrir http://localhost:6060
```

---

## 9.10 Workspaces (Go 1.18+)

Los workspaces permiten trabajar con multiples modulos locales simultaneamente:

```
// go.work en el directorio padre
go 1.23

use (
    ./servidor
    ./libreria-compartida
    ./cliente
)
```

```bash
# Inicializar workspace
go work init ./servidor ./libreria-compartida

# Agregar modulo al workspace
go work use ./otro-modulo

# Sincronizar dependencias entre modulos
go work sync
```

```go
// Beneficio: editas la libreria-compartida y el servidor
// ve los cambios inmediatamente sin necesidad de replace
```

> **Nota**: `go.work` no se commitea normalmente. Es para desarrollo local.

---

## Resumen del Capítulo

- Todo codigo Go pertenece a un paquete. La visibilidad se controla con mayuscula/minuscula.
- `main` es el paquete especial para ejecutables, con funcion `main()`.
- `init()` se ejecuta automaticamente al importar. Usar con moderacion.
- Los modulos (`go.mod`) gestionan dependencias y versionado.
- `go mod tidy` limpia dependencias; `go get` las agrega/actualiza.
- El directorio `internal/` impide importaciones externas al modulo.
- El versionado semantico con `/v2`, `/v3` se refleja en el module path.
- Evita dependencias circulares usando interfaces o fusionando paquetes.
- Los workspaces facilitan el desarrollo multi-modulo local.

En el siguiente capitulo exploraremos testing en Go.
