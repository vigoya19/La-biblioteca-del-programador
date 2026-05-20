# Go: Guía Completa de Programación

## Índice General

1. [Capítulo 1: Introducción a Go](01-introduccion.md)
   - Historia y filosofia de Go
   - Instalacion y configuracion del entorno
   - Hola Mundo y estructura basica de un programa
   - Herramientas del ecosistema (go build, go run, go fmt, go vet)

2. [Capítulo 2: Sintaxis y Tipos de Datos](02-sintaxis-y-tipos.md)
   - Variables y constantes
   - Tipos basicos (int, float, string, bool, byte, rune)
   - Operadores
   - Conversion de tipos
   - Punteros

3. [Capítulo 3: Estructuras de Control](03-estructuras-de-control.md)
   - Condicionales (if, else, switch)
   - Bucles (for, range)
   - Defer, panic y recover

4. [Capítulo 4: Funciones](04-funciones.md)
   - Declaracion y parametros
   - Retorno multiple
   - Funciones variadicas
   - Funciones anonimas y closures
   - Funciones como valores y tipos

5. [Capítulo 5: Estructuras de Datos](05-estructuras-de-datos.md)
   - Arrays y slices
   - Maps
   - Structs
   - Métodos en structs

6. [Capítulo 6: Interfaces y Polimorfismo](06-interfaces.md)
   - Definición e implementación implícita
   - Interfaces comunes (Stringer, Reader, Writer, Error)
   - Composicion de interfaces
   - Type assertions y type switches
   - Interface vacia (any)

7. [Capítulo 7: Concurrencia](07-concurrencia.md)
   - Goroutines
   - Channels (buffered y unbuffered)
   - Select
   - Patrones de concurrencia
   - sync.WaitGroup, sync.Mutex, sync.Once
   - Context

8. [Capítulo 8: Manejo de Errores](08-manejo-de-errores.md)
   - El patron error en Go
   - Errores personalizados
   - Wrapping de errores (errors.Is, errors.As, fmt.Errorf con %w)
   - Sentinel errors
   - Estrategias de manejo de errores

9. [Capítulo 9: Paquetes y Módulos](09-paquetes-modulos.md)
   - Sistema de paquetes
   - Go modules (go.mod, go.sum)
   - Versionado semantico
   - Paquetes internos
   - Publicar un modulo

10. [Capítulo 10: Testing](10-testing.md)
    - Tests unitarios
    - Tests de tabla (table-driven tests)
    - Benchmarks
    - Tests de integracion
    - Mocks y fakes
    - Cobertura de codigo

11. [Capítulo 11: Generics](11-generics.md)
    - Parametros de tipo
    - Constraints
    - Funciones y tipos genericos
    - Cuando usar y cuando no usar generics

12. [Capítulo 12: Buenas Prácticas y Go Idiomático](12-buenas-practicas.md)
    - Convenciones de nombrado
    - Organizacion de codigo
    - Effective Go y Code Review Comments
    - Principios SOLID en Go
    - Manejo de dependencias
    - Linters y herramientas de calidad

13. [Capítulo 13: Patrones de Diseño en Go](13-patrones-de-diseno.md)
    - Patrones creacionales (Factory, Builder, Singleton, Options)
    - Patrones estructurales (Adapter, Decorator, Facade)
    - Patrones de comportamiento (Strategy, Observer, Command)
    - Patrones especificos de Go (Functional Options, Table-Driven)

14. [Capítulo 14: Arquitectura Hexagonal en Go](14-arquitectura-hexagonal.md)
    - Principios de la arquitectura hexagonal
    - Puertos y adaptadores
    - Dominio, aplicacion e infraestructura
    - Implementacion paso a paso
    - Proyecto completo de ejemplo

15. [Capítulo 15: Temas Avanzados](15-temas-avanzados.md)
    - Reflection
    - unsafe package
    - CGo
    - Build tags
    - Embedding de archivos (embed)
    - Profiling y optimizacion

16. [Capítulo 16: Desarrollo Web y APIs](16-desarrollo-web.md)
    - net/http (stdlib)
    - Middlewares
    - APIs REST
    - Bases de datos (database/sql)
    - Migraciones

17. [Capítulo 17: Ejercicios Prácticos](17-ejercicios.md)
    - Ejercicios por nivel (básico, intermedio, avanzado)
    - Proyectos integradores
    - Soluciones comentadas

18. [Capítulo 18: gRPC y Protocol Buffers en Go](18-grpc.md)
    - Setup completo (protoc + plugins)
    - Servicios unarios y streaming (4 tipos)
    - Interceptors (logging, autenticación)
    - Manejo de errores con status codes
    - Cuándo usar gRPC vs REST en Go

19. [Capítulo 19: HTTP Client, Templating y Web Frameworks](19-http-client-web.md)
    - HTTP Client con timeouts y connection pooling
    - Retry con backoff exponencial
    - Templating (html/template, text/template, composición)
    - Comparativa de frameworks (stdlib, chi, Gin, Echo, Fiber)
    - Construyendo tu propio mini-framework sobre stdlib

20. [Capítulo 20: Observabilidad en Go](20-observabilidad.md)
    - Structured logging con log/slog (Go 1.21+)
    - Métricas con Prometheus (counter, gauge, histogram)
    - Distributed tracing con OpenTelemetry
    - Stack completo: slog + Prometheus + OTel en main.go
