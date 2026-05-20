# Capítulo 11: Generics

Los generics llegaron a Go en la version 1.18 y fueron el cambio mas esperado en la historia del lenguaje. Permiten escribir funciones y tipos que trabajan con cualquier tipo de dato, manteniendo la seguridad de tipos en tiempo de compilacion.

> "Los generics nos permiten escribir codigo que es a la vez flexible y seguro, sin sacrificar la filosofia de simplicidad de Go."

---

## 11.1 Parametros de Tipo

Un parametro de tipo se declara entre corchetes `[]` antes de los parametros normales:

```go
package main

import "fmt"

// Funcion generica: funciona con cualquier tipo comparable
func Imprimir[T any](valor T) {
    fmt.Println(valor)
}

// Funcion con dos parametros de tipo
func Iguales[T comparable](a, b T) bool {
    return a == b
}

func main() {
    Imprimir(42)          // int
    Imprimir("hola")      // string
    Imprimir(3.14)        // float64
    Imprimir(true)        // bool

    fmt.Println(Iguales(1, 1))           // true
    fmt.Println(Iguales("go", "go"))     // true
    fmt.Println(Iguales(1, 2))           // false
}
```

### Inferencia de tipo

En la mayoria de los casos, Go infiere el parametro de tipo automaticamente:

```go
// Llamada explicita (rara vez necesaria)
Imprimir[int](42)
Imprimir[string]("hola")

// Inferencia automatica (lo mas comun)
Imprimir(42)       // Go infiere int
Imprimir("hola")   // Go infiere string
```

### Multiples parametros de tipo

```go
package main

import "fmt"

// Funcion con dos tipos genericos diferentes
func Mapear[K comparable, V any](m map[K]V) []V {
    valores := make([]V, 0, len(m))
    for _, v := range m {
        valores = append(valores, v)
    }
    return valores
}

// Funcion que combina dos tipos
func Combinar[A, B any](a A, b B) string {
    return fmt.Sprintf("(%v, %v)", a, b)
}

func main() {
    capitales := map[string]string{
        "Mexico":   "CDMX",
        "Espana":   "Madrid",
        "Colombia": "Bogota",
    }
    fmt.Println(Mapear(capitales)) // [CDMX Madrid Bogota]
    fmt.Println(Combinar(42, "hola")) // (42, hola)
}
```

---

## 11.2 Constraints

Los constraints (restricciones) limitan que tipos pueden usarse con un parametro generico:

### Constraints predefinidos

```go
package main

import (
    "fmt"
    "golang.org/x/exp/constraints"
)

// any: acepta cualquier tipo (alias de interface{})
func Identidad[T any](v T) T {
    return v
}

// comparable: tipos que pueden compararse con == y !=
func Contiene[T comparable](slice []T, valor T) bool {
    for _, v := range slice {
        if v == valor {
            return true
        }
    }
    return false
}

// constraints.Ordered: tipos con orden (<, >, <=, >=)
func Minimo[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

func main() {
    fmt.Println(Identidad(42))                      // 42
    fmt.Println(Contiene([]int{1, 2, 3}, 2))       // true
    fmt.Println(Contiene([]string{"a", "b"}, "c"))  // false
    fmt.Println(Minimo(5, 3))                        // 3
    fmt.Println(Minimo(3.14, 2.71))                  // 2.71
    fmt.Println(Minimo("abc", "xyz"))                // abc
}
```

### Constraints principales

| Constraint | Permite | Desde |
|-----------|---------|-------|
| `any` | Cualquier tipo | Go 1.18 |
| `comparable` | Tipos con `==` y `!=` | Go 1.18 |
| `constraints.Ordered` | Tipos con `<<=>>=` | `golang.org/x/exp` |
| `constraints.Integer` | Tipos enteros | `golang.org/x/exp` |
| `constraints.Float` | Tipos flotantes | `golang.org/x/exp` |
| `constraints.Signed` | Enteros con signo | `golang.org/x/exp` |
| `constraints.Unsigned` | Enteros sin signo | `golang.org/x/exp` |

### Crear tus propios constraints

```go
package main

import "fmt"

// Constraint personalizado: interface que lista tipos permitidos
type Numerico interface {
    int | int32 | int64 | float32 | float64
}

// Funcion que solo acepta tipos numericos
func Sumar[T Numerico](a, b T) T {
    return a + b
}

// Constraint con tilde ~ para incluir tipos definidos por el usuario
type Entero interface {
    ~int | ~int32 | ~int64
}

type MiEntero int

func Duplicar[T Entero](v T) T {
    return v * 2
}

func main() {
    fmt.Println(Sumar(3, 4))         // 7 (int)
    fmt.Println(Sumar(3.14, 2.71))   // 5.85 (float64)

    var x MiEntero = 5
    fmt.Println(Duplicar(x))         // 10 (MiEntero)
}
```

### Union de constraints

```go
package main

import "fmt"

// Constraint que combina varios tipos usando |
type Texto interface {
    string | []byte | []rune
}

func Longitud[T Texto](t T) int {
    return len(t)
}

func main() {
    fmt.Println(Longitud("hola"))              // 4
    fmt.Println(Longitud([]byte("mundo")))     // 5
    fmt.Println(Longitud([]rune("🌍!")))       // 2
}
```

---

## 11.3 Funciones y Tipos Genericos

### Funciones genericas utiles

```go
package main

import (
    "fmt"
    "golang.org/x/exp/slices"
)

// Filtrar elementos de un slice
func Filtrar[T any](slice []T, predicado func(T) bool) []T {
    var resultado []T
    for _, v := range slice {
        if predicado(v) {
            resultado = append(resultado, v)
        }
    }
    return resultado
}

// Encontrar el indice de un elemento
func Indice[T comparable](slice []T, valor T) int {
    for i, v := range slice {
        if v == valor {
            return i
        }
    }
    return -1
}

// Aplicar transformacion a cada elemento
func MapearSlice[T, U any](slice []T, fn func(T) U) []U {
    resultado := make([]U, len(slice))
    for i, v := range slice {
        resultado[i] = fn(v)
    }
    return resultado
}

// Reducir slice a un valor
func Reducir[T, U any](slice []T, inicial U, fn func(U, T) U) U {
    resultado := inicial
    for _, v := range slice {
        resultado = fn(resultado, v)
    }
    return resultado
}

func main() {
    numeros := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    pares := Filtrar(numeros, func(n int) bool { return n%2 == 0 })
    fmt.Println("Pares:", pares) // [2 4 6 8 10]

    fmt.Println("Indice de 5:", Indice(numeros, 5)) // 4

    dobles := MapearSlice(numeros, func(n int) int { return n * 2 })
    fmt.Println("Dobles:", dobles) // [2 4 6 8 10 12 14 16 18 20]

    suma := Reducir(numeros, 0, func(acum, n int) int { return acum + n })
    fmt.Println("Suma:", suma) // 55
}
```

### Tipos genericos (structs)

```go
package main

import "fmt"

// Struct generico
type Par[A, B any] struct {
    Primero A
    Segundo B
}

// Pila generica (LIFO)
type Pila[T any] struct {
    elementos []T
}

func (p *Pila[T]) Push(v T) {
    p.elementos = append(p.elementos, v)
}

func (p *Pila[T]) Pop() (T, bool) {
    if len(p.elementos) == 0 {
        var zero T
        return zero, false
    }
    ultimo := p.elementos[len(p.elementos)-1]
    p.elementos = p.elementos[:len(p.elementos)-1]
    return ultimo, true
}

func (p *Pila[T]) Cima() (T, bool) {
    if len(p.elementos) == 0 {
        var zero T
        return zero, false
    }
    return p.elementos[len(p.elementos)-1], true
}

func (p *Pila[T]) Vacia() bool {
    return len(p.elementos) == 0
}

// Cache generica
type Cache[K comparable, V any] struct {
    datos map[K]V
}

func NuevaCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{
        datos: make(map[K]V),
    }
}

func (c *Cache[K, V]) Guardar(clave K, valor V) {
    c.datos[clave] = valor
}

func (c *Cache[K, V]) Obtener(clave K) (V, bool) {
    valor, ok := c.datos[clave]
    return valor, ok
}

func main() {
    p := Par[int, string]{Primero: 1, Segundo: "uno"}
    fmt.Printf("Par: %+v\n", p) // {Primero:1 Segundo:uno}

    var pila Pila[int]
    pila.Push(1)
    pila.Push(2)
    pila.Push(3)

    for !pila.Vacia() {
        v, _ := pila.Pop()
        fmt.Println(v) // 3, 2, 1
    }

    cache := NuevaCache[string, int]()
    cache.Guardar("contador", 42)
    if v, ok := cache.Obtener("contador"); ok {
        fmt.Println("Cache:", v) // 42
    }
}
```

### Metodos genericos

Los metodos no pueden tener parametros de tipo adicionales, pero si pueden usar los parametros de tipo del struct:

```go
package main

import "fmt"

type Conjunto[T comparable] struct {
    elementos map[T]struct{}
}

func NuevoConjunto[T comparable]() *Conjunto[T] {
    return &Conjunto[T]{
        elementos: make(map[T]struct{}),
    }
}

func (c *Conjunto[T]) Agregar(v T) {
    c.elementos[v] = struct{}{}
}

func (c *Conjunto[T]) Contiene(v T) bool {
    _, ok := c.elementos[v]
    return ok
}

func (c *Conjunto[T]) Eliminar(v T) {
    delete(c.elementos, v)
}

func (c *Conjunto[T]) Tamano() int {
    return len(c.elementos)
}

func main() {
    conjunto := NuevoConjunto[string]()
    conjunto.Agregar("Go")
    conjunto.Agregar("Rust")
    conjunto.Agregar("Go") // Duplicado, ignorado

    fmt.Println(conjunto.Contiene("Go"))   // true
    fmt.Println(conjunto.Tamano())          // 2
}
```

---

## 11.4 Cuando Usar y Cuando No Usar Generics

### Cuando USAR generics

**1. Estructuras de datos reutilizables**

```go
// BIEN: estructuras genericas como slice, map, set, stack
type Cola[T any] struct { ... }
type Arbol[T comparable] struct { ... }
```

**2. Funciones que operan sobre slices/maps de forma generica**

```go
// BIEN: operaciones independientes del tipo
func Filtrar[T any](s []T, fn func(T) bool) []T { ... }
func Contiene[T comparable](s []T, v T) bool { ... }
```

**3. Cuando el tipo no importa para la logica**

```go
// BIEN: la logica no depende del tipo concreto
func Coalesce[T any](valores ...T) T {
    for _, v := range valores {
        var zero T
        if v != zero {
            return v
        }
    }
    var zero T
    return zero
}
```

### Cuando NO USAR generics

**1. Cuando una interface es suficiente**

```go
// MAL: generics innecesario
func ImprimirGenerico[T fmt.Stringer](v T) {
    fmt.Println(v.String())
}

// BIEN: usa la interface directamente
func Imprimir(v fmt.Stringer) {
    fmt.Println(v.String())
}
```

**2. Cuando solo tienes uno o dos tipos concretos**

```go
// MAL: sobre-ingenieria
func ProcesarNumeros[T int | float64](valores []T) float64 {
    var suma float64
    for _, v := range valores {
        suma += float64(v)
    }
    return suma
}

// BIEN: dos funciones concretas, mas simple
func ProcesarEnteros(valores []int) float64 { ... }
func ProcesarFlotantes(valores []float64) float64 { ... }
```

**3. Cuando complica la lectura sin beneficio real**

```go
// MAL: generics que no aportan reutilizacion real
func Saludar[T ~string](nombre T) string {
    return "Hola, " + string(nombre)
}

// BIEN: funcion simple con string
func Saludar(nombre string) string {
    return "Hola, " + nombre
}
```

**4. Cuando genera demasiada abstraccion**

```go
// MAL: demasiadas capas de abstraccion generica
type RepositorioGenerico[T any, ID comparable, Q any] interface {
    BuscarPorID(id ID) (T, error)
    Buscar(query Q) ([]T, error)
    Guardar(entidad T) error
    Eliminar(id ID) error
}

// BIEN: interfaces especificas para cada dominio
type RepositorioUsuario interface {
    BuscarPorID(id int) (*Usuario, error)
    BuscarPorEmail(email string) (*Usuario, error)
    Guardar(u *Usuario) error
}
```

### Regla practica

> Escribe primero el codigo sin generics. Si encuentras que estas repitiendo la misma logica para multiples tipos con codigo identico, considera usar generics. Si no hay repeticion, no los necesitas.

---

## 11.5 Generics en la Practica

### Funciones utilitarias comunes

```go
package main

import (
    "fmt"
    "golang.org/x/exp/maps"
    "golang.org/x/exp/slices"
)

func main() {
    // slices.Clone: copia profunda de un slice
    original := []int{1, 2, 3}
    copia := slices.Clone(original)
    copia[0] = 99
    fmt.Println(original) // [1 2 3] (sin cambios)

    // slices.Sort: ordenar
    nums := []int{3, 1, 4, 1, 5, 9}
    slices.Sort(nums)
    fmt.Println(nums) // [1 1 3 4 5 9]

    // slices.SortFunc: ordenar con comparador personalizado
    palabras := []string{"manzana", "pera", "banana", "uva"}
    slices.SortFunc(palabras, func(a, b string) int {
        return len(a) - len(b)
    })
    fmt.Println(palabras) // [uva pera banana manzana]

    // slices.SortStableFunc: orden estable
    // slices.BinarySearch: busqueda binaria
    // slices.Compact: eliminar duplicados consecutivos
    // slices.Max/Min: maximo y minimo

    // maps.Clone: copiar mapa
    m := map[string]int{"a": 1, "b": 2}
    copiaMap := maps.Clone(m)

    // maps.Keys / maps.Values
    claves := maps.Keys(m)
    valores := maps.Values(m)
    fmt.Println(claves, valores)
    fmt.Println(copiaMap)
}
```

---

## 11.6 Type Sets: El Modelo Mental de Constraints

En Go, un constraint de interfaz define un **conjunto de tipos** (type set). Entender este modelo unifica todo el sistema de generics:

```go
// Un constraint es un CONJUNTO de tipos permitidos

// Conjunto de tipos por union:
// int | float64  => {int, float64}
type Numerico interface {
    int | float64
}

// Conjunto de tipos con tilde ~:
// ~int => {int, MiEntero, ID, cualquier tipo cuyo underlying type sea int}
type Entero interface {
    ~int
}

// Conjunto de tipos con metodos:
// {todos los tipos que tengan String() string}
type Stringer interface {
    String() string
}

// Conjunto VACIO (ningun tipo lo satisface):
// interface{ int; float64 }  => interseccion vacia
```

### Visualizacion de type sets

```
┌─────────────────────────────────────────┐
│ any = TODOS los tipos                   │
│  ┌────────────────────────────────────┐ │
│  │ comparable                         │ │
│  │  ┌──────────────────────────────┐  │ │
│  │  │ constraints.Ordered          │  │ │
│  │  │  ┌────────────────────┐      │  │ │
│  │  │  │ ~int | ~int8 | ... │      │  │ │
│  │  │  └────────────────────┘      │  │ │
│  │  └──────────────────────────────┘  │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

```go
package main

import "fmt"

// Constraint con type set explicito (union de tipos)
type CodigoPostal interface {
    ~string | ~int
}

// Funcion que acepta cualquier tipo en el type set
func FormatearCodigoPostal[T CodigoPostal](cp T) string {
    return fmt.Sprintf("CP: %v", cp)
}

type CPSpain string  // underlying type = string
type CPMexico int    // underlying type = int

func main() {
    fmt.Println(FormatearCodigoPostal(CPSpain("28001")))  // CP: 28001
    fmt.Println(FormatearCodigoPostal(CPMexico(6600)))     // CP: 6600
    // FormatearCodigoPostal(true)  // ERROR: bool no esta en el type set
}
```

---

## 11.7 Rendimiento y Generacion de Codigo

Go usa **GC Shape Stenciling** (no monomorfizacion completa como Rust/C++):

```go
// Estos dos usos comparten el mismo codigo generado (misma "shape")
Filtrar[int]([]int{1, 2, 3}, ...)
Filtrar[int64]([]int64{1, 2, 3}, ...)
// int e int64 tienen la misma forma (mismo tamaño, misma alineacion)

// Estos NO comparten codigo (different shapes)
Filtrar[int]([]int{1, 2, 3}, ...)
Filtrar[string]([]string{"a"}, ...)
// int y string tienen formas diferentes
```

### Cuando generics causa boxing (allocaciones en heap)

```go
// Boxing: convertir un valor concreto a interface{}
// Esto causa allocacion en heap (mas lento)

// MAL: boxing innecesario
func ImprimirTodo[T any](valores []T) {
    for _, v := range valores {
        fmt.Println(v)  // v se convierte a interface{} -> allocacion
    }
}

// MEJOR: pasar la funcion de formateo
func ImprimirConFormato[T any](valores []T, format func(T) string) {
    for _, v := range valores {
        fmt.Println(format(v))  // Sin boxing adicional
    }
}
```

### Benchmark: generics vs no generics

```go
package main

import "testing"

// Version generica
func SumaGenerica[T Numerico](a, b T) T {
    return a + b
}

// Version concreta
func SumaInt(a, b int) int {
    return a + b
}

func BenchmarkSumaGenerica(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = SumaGenerica(10, 20)
    }
}

func BenchmarkSumaConcreta(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = SumaInt(10, 20)
    }
}
// Resultado tipico: la version generica es ~5% mas lenta
// Para funciones que hacen trabajo real (I/O, calculos), la diferencia es negligible.
```

> **Regla**: No evites generics por rendimiento a menos que un benchmark lo demuestre. Para la mayoria del codigo, la diferencia es minima. El costo real es la complejidad del codigo, no la velocidad.

---

## 11.8 Limitaciones de la Inferencia de Tipos

La inferencia de tipos en Go generics NO es tipo Hindley-Milner. Tiene limitaciones concretas que encontraras en el dia a dia:

```go
// LIMITACION 1: No infiere de valores de retorno
func Identidad[T any](v T) T { return v }

x := Identidad(42)     // OK: infiere int del argumento

// Pero no funciona si el tipo viene del contexto:
var y float64 = Identidad(42)  // ERROR: infiere int, no float64
var y float64 = Identidad[float64](42)  // OK: tipo explicito

// LIMITACION 2: Metodos genericos no pueden tener type params adicionales
type Pila[T any] struct { ... }

// func (p *Pila[T]) Mapear[U any](fn func(T) U) *Pila[U] { ... }
// ERROR: los metodos no pueden introducir nuevos type params

// Solucion: funcion independiente
func MapearPila[T, U any](p *Pila[T], fn func(T) U) *Pila[U] { ... }

// LIMITACION 3: No se pueden usar constraints como tipos normales
type Numerico interface { int | float64 }

var x Numerico = 42  // ERROR: constraint solo para generics
// Debes usar la interface como constraint, no como tipo

// LIMITACION 4: Dificultad con tipos sin nombre (struct literals)
func Comparar[T comparable](a, b T) bool { return a == b }

// Comparar(struct{X int}{1}, struct{X int}{1})  // ERROR: inferencia compleja
type Punto struct{ X int }
Comparar(Punto{1}, Punto{1})  // OK: tipo con nombre
```

---

## 11.9 El Constraint comparable y sus Matices

`comparable` parece simple pero tiene sutilezas importantes:

```go
// comparable permite == y !=
// PERO no todos los tipos comparables son seguros en runtime:

type Cualquier interface{}  // interface vacia

func EsIgual[T comparable](a, b T) bool {
    return a == b
}

func main() {
    // OK: tipos concretos comparables
    EsIgual(1, 2)                    // OK
    EsIgual("hola", "mundo")         // OK
    EsIgual([3]int{1,2,3}, [3]int{1,2,3})  // OK: arrays son comparables

    // PELIGROSO: interfaces son "comparable" pero pueden panic en runtime
    var a Cualquier = 42
    var b Cualquier = "hola"
    // EsIgual(a, b)  // PANIC en runtime: comparing uncomparable types
    // Las interfaces son comparables si sus valores dinamicos lo son

    // NO comparable: slices, maps, funciones
    // EsIgual([]int{1}, []int{1})  // ERROR: []int no es comparable
}

// Map keys: MAS restrictivo que comparable
type Conjunto[T comparable] map[T]struct{}

type ContieneSlice struct {
    Datos []int  // Slice no es comparable
}

// Conjunto[ContieneSlice]{}  // ERROR: struct con slice no es usable como key
```

---

## 11.10 Testing de Codigo Generico

Testear codigo generico requiere cubrir multiples tipos concretos:

```go
package main

import "testing"

func TestFiltrar(t *testing.T) {
    // Testear con diferentes tipos concretos
    t.Run("int", func(t *testing.T) {
        entrada := []int{1, 2, 3, 4, 5, 6}
        resultado := Filtrar(entrada, func(n int) bool { return n%2 == 0 })
        esperado := []int{2, 4, 6}
        if !slices.Equal(resultado, esperado) {
            t.Errorf("Filtrar(int) = %v; esperado %v", resultado, esperado)
        }
    })

    t.Run("string", func(t *testing.T) {
        entrada := []string{"go", "rust", "c", "python"}
        resultado := Filtrar(entrada, func(s string) bool { return len(s) > 2 })
        esperado := []string{"rust", "python"}
        if !slices.Equal(resultado, esperado) {
            t.Errorf("Filtrar(string) = %v; esperado %v", resultado, esperado)
        }
    })

    t.Run("slice vacio", func(t *testing.T) {
        resultado := Filtrar([]int{}, func(n int) bool { return true })
        if len(resultado) != 0 {
            t.Errorf("Filtrar vacio debe ser vacio, obtenido %v", resultado)
        }
    })

    t.Run("todos filtrados", func(t *testing.T) {
        entrada := []int{1, 3, 5, 7}
        resultado := Filtrar(entrada, func(n int) bool { return n%2 == 0 })
        if len(resultado) != 0 {
            t.Errorf("esperado slice vacio, obtenido %v", resultado)
        }
    })

    t.Run("ninguno filtrado", func(t *testing.T) {
        entrada := []int{2, 4, 6}
        resultado := Filtrar(entrada, func(n int) bool { return n%2 == 0 })
        if !slices.Equal(resultado, entrada) {
            t.Errorf("esperado %v, obtenido %v", entrada, resultado)
        }
    })
}

// Test helper para codigo generico
func verificarIguales[T comparable](t *testing.T, nombre string, obtenido, esperado T) {
    t.Helper()
    if obtenido != esperado {
        t.Errorf("%s: obtenido %v, esperado %v", nombre, obtenido, esperado)
    }
}

func TestVerificarIguales(t *testing.T) {
    verificarIguales(t, "int", 42, 42)
    verificarIguales(t, "string", "go", "go")
}
```

---

## Resumen del Capítulo

- Los generics permiten escribir codigo reutilizable con seguridad de tipos.
- Los parametros de tipo van entre `[]` antes de los parametros normales.
- `any` acepta cualquier tipo, `comparable` requiere `==` y `!=` (con matices).
- Los constraints definen **type sets**: conjuntos de tipos permitidos.
- La tilde `~` en un constraint incluye tipos definidos por el usuario basados en ese tipo.
- Tipos genericos como `Pila[T]` y `Cache[K, V]` son estructuras de datos flexibles.
- Go usa GC Shape Stenciling: mismo shape comparte codigo; shapes distintos generan codigo separado.
- La inferencia de tipos tiene limitaciones: no infiere de retornos, metodos no pueden agregar type params.
- `comparable` permite tipos que podrian panic en runtime (interfaces con valores dinamicos incomparables).
- Usa generics cuando tengas codigo repetido identico para multiples tipos.
- Evita generics cuando una interface es suficiente o complican la lectura.
- `golang.org/x/exp/slices` y `maps` ofrecen utilidades genericas listas para usar.

En el siguiente capitulo exploraremos las buenas practicas y el Go idiomatico.
