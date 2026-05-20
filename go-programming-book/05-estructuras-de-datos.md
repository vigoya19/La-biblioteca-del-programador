# Capítulo 5: Estructuras de Datos

## 5.1 Arrays

Los arrays en Go tienen tamano fijo y son valores (se copian al asignar):

```go
package main

import "fmt"

func main() {
    // Declaracion
    var numeros [5]int                          // [0 0 0 0 0]
    letras := [3]string{"a", "b", "c"}         // [a b c]
    auto := [...]int{1, 2, 3, 4}               // [...] cuenta automaticamente -> [4]int
    disperso := [10]int{0: 1, 5: 50, 9: 100}   // Indices especificos

    // Acceso
    numeros[0] = 42
    fmt.Println(numeros[0])  // 42
    fmt.Println(len(letras)) // 3

    // Los arrays son valores, se copian
    a := [3]int{1, 2, 3}
    b := a        // b es una COPIA de a
    b[0] = 99
    fmt.Println(a) // [1 2 3] (no cambio)
    fmt.Println(b) // [99 2 3]

    // Comparacion (solo si son del mismo tipo y tamano)
    fmt.Println([3]int{1, 2, 3} == [3]int{1, 2, 3}) // true

    // Iterar
    for i, v := range auto {
        fmt.Printf("auto[%d] = %d\n", i, v)
    }

    fmt.Println(disperso)
}
```

> **En la practica**: los arrays se usan poco. Los **slices** son mucho mas comunes y flexibles.

---

## 5.2 Slices

Los slices son la estructura de datos mas usada en Go. Son vistas dinamicas sobre arrays:

```go
package main

import "fmt"

func main() {
    // Crear slices
    s1 := []int{1, 2, 3, 4, 5}          // Literal
    s2 := make([]int, 5)                  // Con make: len=5, cap=5
    s3 := make([]int, 0, 10)             // len=0, cap=10
    var s4 []int                          // nil slice (len=0, cap=0)

    fmt.Println(s1, s2, s3, s4)
    fmt.Println(s4 == nil) // true

    // Longitud y capacidad
    fmt.Println(len(s1), cap(s1))  // 5 5

    // Acceso y modificacion
    s1[0] = 10
    fmt.Println(s1)  // [10 2 3 4 5]

    // Slicing (crear sub-slices)
    sub := s1[1:3]     // [2 3]    (indices 1 y 2)
    desde := s1[2:]    // [3 4 5]  (desde indice 2)
    hasta := s1[:3]    // [10 2 3] (hasta indice 3, sin incluir)
    copia := s1[:]     // [10 2 3 4 5] (todo)

    fmt.Println(sub, desde, hasta, copia)

    // CUIDADO: los sub-slices comparten el array subyacente
    sub[0] = 99
    fmt.Println(s1)  // [10 99 3 4 5] - s1 tambien cambio!
}
```

### append

```go
package main

import "fmt"

func main() {
    s := []int{1, 2, 3}

    // Agregar elementos
    s = append(s, 4)
    s = append(s, 5, 6, 7)
    fmt.Println(s)  // [1 2 3 4 5 6 7]

    // Agregar otro slice
    extra := []int{8, 9, 10}
    s = append(s, extra...)
    fmt.Println(s)  // [1 2 3 4 5 6 7 8 9 10]

    // IMPORTANTE: append puede reasignar el array subyacente
    // Siempre usa s = append(s, ...) para capturar el nuevo slice
}
```

### Copiar slices

```go
package main

import "fmt"

func main() {
    original := []int{1, 2, 3, 4, 5}

    // copy() - copia independiente
    copia := make([]int, len(original))
    copy(copia, original)
    copia[0] = 99
    fmt.Println(original) // [1 2 3 4 5] (no afectado)
    fmt.Println(copia)    // [99 2 3 4 5]

    // Tambien puedes usar slices.Clone (Go 1.21+)
    // copia2 := slices.Clone(original)
}
```

### Eliminar elementos

```go
package main

import "fmt"

func main() {
    s := []int{1, 2, 3, 4, 5}

    // Eliminar elemento en indice i (manteniendo orden)
    i := 2
    s = append(s[:i], s[i+1:]...)
    fmt.Println(s)  // [1 2 4 5]

    // Eliminar sin mantener orden (mas eficiente)
    s2 := []int{1, 2, 3, 4, 5}
    j := 2
    s2[j] = s2[len(s2)-1]   // Reemplazar con el ultimo
    s2 = s2[:len(s2)-1]      // Reducir tamano
    fmt.Println(s2)  // [1 2 5 4]

    // Con slices.Delete (Go 1.21+)
    // s = slices.Delete(s, i, i+1)
}
```

### Slices como parametros

```go
// Los slices se pasan por referencia (al header, no al contenido)
func duplicarValores(s []int) {
    for i := range s {
        s[i] *= 2
    }
    // Modifica el slice original
}

// Para funciones que pueden cambiar el tamano, retorna el nuevo slice
func agregarSiPositivo(s []int, n int) []int {
    if n > 0 {
        s = append(s, n)
    }
    return s
}
```

### Nil slice vs empty slice

```go
var nilSlice []int            // nil, len=0, cap=0
emptySlice := []int{}         // no nil, len=0, cap=0
emptySlice2 := make([]int, 0) // no nil, len=0, cap=0

// Ambos funcionan igual en la practica
fmt.Println(len(nilSlice) == len(emptySlice))  // true
// Pero:
fmt.Println(nilSlice == nil)    // true
fmt.Println(emptySlice == nil)  // false

// En JSON:
// nilSlice     -> null
// emptySlice   -> []
```

---

## 5.3 Maps

Los maps son tablas hash (diccionarios):

```go
package main

import "fmt"

func main() {
    // Crear maps
    m1 := map[string]int{
        "uno":  1,
        "dos":  2,
        "tres": 3,
    }

    m2 := make(map[string]int)    // Map vacio
    var m3 map[string]int         // nil map (NO se puede escribir!)

    fmt.Println(m1, m2, m3)

    // Insertar / Modificar
    m2["cuatro"] = 4
    m2["cinco"] = 5

    // Leer
    valor := m1["dos"]
    fmt.Println(valor)  // 2

    // Leer clave inexistente retorna zero value
    fmt.Println(m1["inexistente"])  // 0

    // Verificar si una clave existe (comma ok idiom)
    valor, existe := m1["dos"]
    if existe {
        fmt.Println("Encontrado:", valor)
    }

    if _, existe := m1["seis"]; !existe {
        fmt.Println("No existe 'seis'")
    }

    // Eliminar
    delete(m1, "uno")
    fmt.Println(m1)  // map[dos:2 tres:3]

    // Iterar (orden NO garantizado)
    for clave, valor := range m1 {
        fmt.Printf("%s = %d\n", clave, valor)
    }

    // Longitud
    fmt.Println(len(m1))  // 2
}
```

### Maps como conjuntos (sets)

Go no tiene un tipo `set`, pero se puede simular con maps:

```go
package main

import "fmt"

func main() {
    // Set de strings
    visitados := make(map[string]bool)

    visitados["Madrid"] = true
    visitados["Paris"] = true
    visitados["Tokio"] = true

    // Verificar pertenencia
    if visitados["Madrid"] {
        fmt.Println("Ya visitaste Madrid")
    }

    // Alternativa mas eficiente en memoria: map[string]struct{}
    set := make(map[string]struct{})
    set["a"] = struct{}{}
    set["b"] = struct{}{}

    if _, ok := set["a"]; ok {
        fmt.Println("'a' esta en el set")
    }
}
```

### Maps y concurrencia

```go
// Los maps NO son seguros para uso concurrente
// Para concurrencia, usa sync.Map o protege con sync.Mutex

import "sync"

var (
    mu    sync.Mutex
    cache = make(map[string]string)
)

func obtener(clave string) string {
    mu.Lock()
    defer mu.Unlock()
    return cache[clave]
}

func guardar(clave, valor string) {
    mu.Lock()
    defer mu.Unlock()
    cache[clave] = valor
}
```

---

## 5.4 Structs

Los structs son tipos compuestos que agrupan campos:

```go
package main

import "fmt"

// Definir un struct
type Persona struct {
    Nombre string
    Edad   int
    Email  string
}

// Struct con campos privados (minuscula)
type usuario struct {
    nombre    string
    password  string  // privado al paquete
    Activo    bool    // exportado
}

func main() {
    // Crear structs
    p1 := Persona{
        Nombre: "Andres",
        Edad:   30,
        Email:  "andres@email.com",
    }

    p2 := Persona{"Maria", 25, "maria@email.com"} // Orden posicional (fragil, evitar)

    p3 := Persona{Nombre: "Carlos"}  // Campos no especificados = zero value

    var p4 Persona  // Todos los campos en zero value

    fmt.Println(p1, p2, p3, p4)

    // Acceder a campos
    fmt.Println(p1.Nombre)  // Andres
    p1.Edad = 31

    // Punteros a structs
    pp := &p1
    fmt.Println(pp.Nombre)   // Andres (Go desreferencia automaticamente)
    fmt.Println((*pp).Nombre) // Equivalente pero innecesario

    // Constructor (funcion convencional, no hay constructores nativos)
    p5 := NuevaPersona("Laura", 28, "laura@email.com")
    fmt.Println(p5)
}

// Constructor idiomatico
func NuevaPersona(nombre string, edad int, email string) *Persona {
    return &Persona{
        Nombre: nombre,
        Edad:   edad,
        Email:  email,
    }
}
```

### Composicion (embedding)

Go no tiene herencia, pero tiene composicion mediante embedding:

```go
package main

import "fmt"

type Direccion struct {
    Calle  string
    Ciudad string
    Pais   string
}

type Empleado struct {
    Nombre    string
    Direccion              // Embedded (sin nombre de campo)
    Salario   float64
}

type Gerente struct {
    Empleado               // Embedded
    Departamento string
    Reportes     int
}

func main() {
    e := Empleado{
        Nombre: "Andres",
        Direccion: Direccion{
            Calle:  "Calle Principal 123",
            Ciudad: "Madrid",
            Pais:   "Espana",
        },
        Salario: 50000,
    }

    // Acceso directo a campos embebidos (promovidos)
    fmt.Println(e.Ciudad)           // Madrid (acceso directo)
    fmt.Println(e.Direccion.Ciudad) // Madrid (acceso explicito)

    g := Gerente{
        Empleado: Empleado{
            Nombre: "Laura",
            Direccion: Direccion{Ciudad: "Barcelona"},
            Salario: 80000,
        },
        Departamento: "Ingenieria",
        Reportes:     10,
    }

    // Acceso a traves de multiples niveles
    fmt.Println(g.Nombre)        // Laura
    fmt.Println(g.Ciudad)        // Barcelona
    fmt.Println(g.Departamento)  // Ingenieria
}
```

### Tags de struct

Los tags son metadatos que se adjuntan a los campos. Son muy usados para JSON, bases de datos, validacion, etc:

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Usuario struct {
    ID        int    `json:"id"`
    Nombre    string `json:"nombre"`
    Email     string `json:"email,omitempty"`
    Password  string `json:"-"`                    // Nunca se serializa
    Edad      int    `json:"edad" validate:"min=0,max=150"`
}

func main() {
    u := Usuario{
        ID:       1,
        Nombre:   "Andres",
        Email:    "",
        Password: "secreto",
        Edad:     30,
    }

    // Serializar a JSON
    data, _ := json.Marshal(u)
    fmt.Println(string(data))
    // {"id":1,"nombre":"Andres","edad":30}
    // Email omitido (omitempty + vacio), Password excluido (-)

    // Deserializar de JSON
    jsonStr := `{"id":2,"nombre":"Maria","email":"maria@email.com","edad":25}`
    var u2 Usuario
    json.Unmarshal([]byte(jsonStr), &u2)
    fmt.Printf("%+v\n", u2)
}
```

---

## 5.5 Metodos en Structs

Los metodos son funciones asociadas a un tipo:

```go
package main

import (
    "fmt"
    "math"
)

type Circulo struct {
    Radio float64
}

// Metodo con receptor de valor (no modifica el original)
func (c Circulo) Area() float64 {
    return math.Pi * c.Radio * c.Radio
}

func (c Circulo) Perimetro() float64 {
    return 2 * math.Pi * c.Radio
}

// Metodo con receptor de puntero (puede modificar el original)
func (c *Circulo) Escalar(factor float64) {
    c.Radio *= factor
}

func (c Circulo) String() string {
    return fmt.Sprintf("Circulo{radio=%.2f}", c.Radio)
}

type Rectangulo struct {
    Ancho, Alto float64
}

func (r Rectangulo) Area() float64 {
    return r.Ancho * r.Alto
}

func main() {
    c := Circulo{Radio: 5}
    fmt.Println(c.Area())       // 78.53981633974483
    fmt.Println(c.Perimetro())  // 31.41592653589793

    c.Escalar(2)
    fmt.Println(c.Radio)  // 10
    fmt.Println(c)         // Circulo{radio=10.00} (usa String())

    r := Rectangulo{Ancho: 4, Alto: 3}
    fmt.Println(r.Area())  // 12
}
```

### Cuando usar receptor de valor vs puntero

```go
// Receptor de VALOR: func (t Tipo) Metodo()
// - El struct es pequeno (pocos campos, sin slices/maps)
// - El metodo NO modifica el receptor
// - Quieres que sea inmutable

// Receptor de PUNTERO: func (t *Tipo) Metodo()
// - El struct es grande
// - El metodo MODIFICA el receptor
// - El struct contiene campos que no se deben copiar (sync.Mutex)
// - Quieres consistencia (si un metodo usa puntero, todos deberian)
```

> **Regla practica**: si dudas, usa receptor de puntero. Es mas eficiente y consistente.

### Metodos en tipos no-struct

Puedes definir metodos en cualquier tipo que definas:

```go
type Celsius float64
type Fahrenheit float64

func (c Celsius) AFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (f Fahrenheit) ACelsius() Celsius {
    return Celsius((f - 32) * 5 / 9)
}

type ListaEnteros []int

func (l ListaEnteros) Suma() int {
    total := 0
    for _, v := range l {
        total += v
    }
    return total
}

func (l ListaEnteros) Promedio() float64 {
    if len(l) == 0 {
        return 0
    }
    return float64(l.Suma()) / float64(len(l))
}
```

---

## Resumen del Capítulo

- **Arrays**: tamano fijo, son valores (se copian). Se usan poco directamente.
- **Slices**: dinamicos, referencia a un array subyacente. La estructura mas usada.
- **Maps**: tablas hash, no son seguros para concurrencia. Utiles como sets.
- **Structs**: tipos compuestos. Composicion via embedding, no herencia.
- **Metodos**: funciones con receptor. Valor para lectura, puntero para modificacion.
- **Tags**: metadatos en structs para JSON, DB, validacion, etc.
