# Capítulo 1: Introducción a Go

## 1.1 Historia y Filosofia de Go

Go (tambien conocido como Golang) fue creado en 2007 por **Robert Griesemer**, **Rob Pike** y **Ken Thompson** en Google. Fue anunciado publicamente en 2009 y su version 1.0 fue lanzada en 2012.

### Por que se creo Go?

Los creadores de Go enfrentaban problemas reales en Google:

- **Compilaciones lentas**: los proyectos en C++ tardaban minutos u horas en compilar.
- **Complejidad excesiva**: C++ y Java se habian vuelto demasiado complejos.
- **Concurrencia dificil**: manejar multiples hilos era propenso a errores.
- **Dependencias enredadas**: los sistemas grandes tenian grafos de dependencias inmanejables.

Go fue disenado para resolver estos problemas con una filosofia clara:

> "Less is more" - La simplicidad es una caracteristica, no una limitacion.

### Principios fundamentales de Go

1. **Simplicidad**: Go tiene ~25 palabras reservadas. Si algo se puede hacer de una sola forma, mejor.
2. **Legibilidad**: el codigo se lee mucho mas de lo que se escribe. Go prioriza la claridad.
3. **Compilacion rapida**: un compilador extremadamente rapido, incluso para proyectos grandes.
4. **Concurrencia nativa**: goroutines y channels estan integrados en el lenguaje.
5. **Tipado estatico**: seguridad de tipos sin la verbosidad de Java.
6. **Garbage collection**: manejo automatico de memoria sin la complejidad de C/C++.
7. **Un solo binario**: Go compila a un unico ejecutable sin dependencias externas.
8. **Herramientas incluidas**: formateo, testing, documentacion, profiling, todo viene con el lenguaje.

### Go en la industria

Go es utilizado por empresas como:

- **Docker**: el motor de contenedores esta escrito en Go.
- **Kubernetes**: el orquestador de contenedores mas usado del mundo.
- **Terraform**: infraestructura como codigo.
- **Prometheus**: monitoreo y alertas.
- **CockroachDB**: base de datos distribuida.
- **Uber, Twitch, Dropbox, Netflix**: servicios backend de alta concurrencia.

---

## 1.2 Instalacion y Configuracion del Entorno

### Instalar Go

#### macOS

```bash
# Con Homebrew
brew install go

# O descargando desde https://go.dev/dl/
```

#### Linux

```bash
# Descargar el tarball (ajusta la version)
wget https://go.dev/dl/go1.23.0.linux-amd64.tar.gz

# Extraer en /usr/local
sudo tar -C /usr/local -xzf go1.23.0.linux-amd64.tar.gz

# Agregar al PATH (en ~/.bashrc o ~/.zshrc)
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
```

#### Windows

Descarga el instalador `.msi` desde [go.dev/dl](https://go.dev/dl/) y ejecutalo.

### Verificar la instalacion

```bash
go version
# go version go1.23.0 darwin/arm64

go env GOPATH
# /Users/tu-usuario/go
```

### Estructura del workspace

```
$GOPATH/
  bin/       # Binarios compilados
  pkg/       # Cache de paquetes compilados
  src/       # Codigo fuente (menos relevante con modules)
```

Con **Go Modules** (el estandar actual), puedes trabajar fuera de `$GOPATH`:

```bash
mkdir mi-proyecto
cd mi-proyecto
go mod init github.com/tu-usuario/mi-proyecto
```

### Editores recomendados

- **VS Code** con la extension oficial de Go (gopls)
- **GoLand** (JetBrains) - IDE dedicado para Go
- **Neovim** con gopls via LSP

---

## 1.3 Hola Mundo y Estructura Basica

### Tu primer programa en Go

Crea un archivo `main.go`:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hola, mundo!")
}
```

Ejecutalo:

```bash
go run main.go
# Hola, mundo!
```

O compilalo y ejecutalo:

```bash
go build -o hola main.go
./hola
# Hola, mundo!
```

### Anatomia de un programa Go

```go
// 1. Declaracion de paquete
// Todo archivo Go pertenece a un paquete.
// El paquete "main" es especial: indica un programa ejecutable.
package main

// 2. Imports
// Se importan paquetes que necesitamos usar.
// "fmt" es el paquete de formateo e impresion.
import "fmt"

// 3. Funcion main
// Es el punto de entrada del programa.
// Solo puede existir en el paquete "main".
func main() {
    fmt.Println("Hola, mundo!")
}
```

### Reglas fundamentales

```go
package main

import (
    "fmt"    // Import agrupado con parentesis
    "math"
)

// Las funciones y variables exportadas empiezan con Mayuscula
func Sumar(a, b int) int {
    return a + b
}

// Las que empiezan con minuscula son privadas al paquete
func restar(a, b int) int {
    return a - b
}

func main() {
    resultado := Sumar(3, 4)  // := es declaracion corta
    fmt.Println(resultado)     // 7
    fmt.Println(math.Pi)       // 3.141592653589793
}
```

**Reglas importantes:**
- Las llaves de apertura `{` van en la **misma linea** (obligatorio, no es estilo).
- Las variables declaradas deben usarse (error de compilacion si no).
- Los imports deben usarse (error de compilacion si no).
- No hay punto y coma al final de las lineas.

---

## 1.4 Herramientas del Ecosistema

Go viene con un conjunto de herramientas poderosas integradas:

### go run

Compila y ejecuta en un solo paso (ideal para desarrollo):

```bash
go run main.go
go run .                    # Ejecuta todo el paquete main del directorio
go run ./cmd/servidor       # Ejecuta un paquete especifico
```

### go build

Compila y genera un binario ejecutable:

```bash
go build -o mi-app .
go build ./...              # Compila todos los paquetes

# Cross-compilation (compilar para otra plataforma)
GOOS=linux GOARCH=amd64 go build -o mi-app-linux .
GOOS=windows GOARCH=amd64 go build -o mi-app.exe .
```

### go fmt

Formatea el codigo automaticamente. **Todos** los proyectos Go usan el mismo formato:

```bash
go fmt ./...               # Formatea todos los archivos
gofmt -s -w .              # Version mas avanzada con simplificacion
```

> En Go no hay debates sobre estilos de formateo. `go fmt` es la ley.

### go vet

Analiza el codigo buscando errores comunes que el compilador no detecta:

```bash
go vet ./...
```

Detecta cosas como:
- Argumentos incorrectos en `fmt.Printf`
- Valores no usados en comparaciones
- Copias accidentales de locks

### go test

Ejecuta tests:

```bash
go test ./...                    # Todos los tests
go test -v ./...                 # Con detalle
go test -cover ./...             # Con cobertura
go test -bench=. ./...           # Con benchmarks
go test -race ./...              # Detectar data races
```

### go mod

Manejo de dependencias:

```bash
go mod init github.com/user/proyecto    # Inicializar modulo
go mod tidy                              # Limpiar dependencias
go mod download                          # Descargar dependencias
go mod vendor                            # Crear directorio vendor
```

### go doc

Documentacion desde la terminal:

```bash
go doc fmt                  # Documentacion del paquete fmt
go doc fmt.Println          # Documentacion de una funcion
go doc -all fmt             # Documentacion completa
```

### go generate

Ejecuta comandos de generacion de codigo definidos en comentarios:

```go
//go:generate stringer -type=Color
```

```bash
go generate ./...
```

### Otras herramientas utiles

```bash
go install golang.org/x/tools/cmd/goimports@latest    # Import automatico
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest  # Linter completo
```

---

## Resumen del Capítulo

- Go fue creado en Google para resolver problemas reales de escala.
- Su filosofia es la simplicidad, legibilidad y productividad.
- El ecosistema de herramientas es completo y consistente.
- Todo programa Go tiene: `package`, `import`, y una funcion `main` como punto de entrada.
- Las convenciones se refuerzan con herramientas (`go fmt`, `go vet`).

En el siguiente capitulo exploraremos la sintaxis y los tipos de datos en detalle.
