# Capítulo 4: Funciones

## 4.1 Declaracion y Parametros

### Funcion basica

```go
package main

import "fmt"

// Funcion sin parametros ni retorno
func saludar() {
    fmt.Println("Hola!")
}

// Funcion con parametros y retorno
func sumar(a int, b int) int {
    return a + b
}

// Parametros del mismo tipo se pueden abreviar
func multiplicar(a, b int) int {
    return a * b
}

// Parametros con tipos diferentes
func formatear(nombre string, edad int) string {
    return fmt.Sprintf("%s tiene %d anios", nombre, edad)
}

func main() {
    saludar()
    fmt.Println(sumar(3, 4))              // 7
    fmt.Println(multiplicar(3, 4))        // 12
    fmt.Println(formatear("Andres", 30))  // Andres tiene 30 anios
}
```

---

## 4.2 Retorno Multiple

Una de las caracteristicas mas distintivas de Go:

```go
package main

import (
    "errors"
    "fmt"
    "math"
)

// Retorno multiple
func dividir(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division por cero")
    }
    return a / b, nil
}

// Retorno con nombres (named returns)
func estadisticas(numeros []float64) (media, desviacion float64) {
    n := float64(len(numeros))
    if n == 0 {
        return  // "naked return": retorna los zero values
    }

    // Calcular media
    var suma float64
    for _, v := range numeros {
        suma += v
    }
    media = suma / n

    // Calcular desviacion
    var sumaCuadrados float64
    for _, v := range numeros {
        diff := v - media
        sumaCuadrados += diff * diff
    }
    desviacion = math.Sqrt(sumaCuadrados / n)

    return  // Retorna media y desviacion
}

func main() {
    // Patron idiomatico: manejar el error inmediatamente
    resultado, err := dividir(10, 3)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("Resultado: %.2f\n", resultado)

    // Ignorar un valor con _
    solo_resultado, _ := dividir(10, 2)
    fmt.Println(solo_resultado)

    // Named returns
    m, d := estadisticas([]float64{2, 4, 4, 4, 5, 5, 7, 9})
    fmt.Printf("Media: %.2f, Desviacion: %.2f\n", m, d)
}
```

> **Buena practica**: usa named returns solo cuando aclaran el significado. Para funciones cortas, retornos explicitos son mas legibles.

---

## 4.3 Funciones Variadicas

Aceptan un numero variable de argumentos:

```go
package main

import "fmt"

// Los parametros variadicos se reciben como un slice
func sumar(numeros ...int) int {
    total := 0
    for _, n := range numeros {
        total += n
    }
    return total
}

// Pueden combinarse con parametros normales (variadico siempre al final)
func imprimir(prefijo string, valores ...int) {
    fmt.Printf("%s: %v\n", prefijo, valores)
}

func main() {
    fmt.Println(sumar(1, 2, 3))        // 6
    fmt.Println(sumar(1, 2, 3, 4, 5))  // 15
    fmt.Println(sumar())                // 0

    // Pasar un slice como argumento variadico con ...
    numeros := []int{10, 20, 30}
    fmt.Println(sumar(numeros...))      // 60

    imprimir("Resultados", 1, 2, 3)    // Resultados: [1 2 3]
}
```

`fmt.Println` es un ejemplo famoso de funcion variadica:
```go
func Println(a ...any) (n int, err error)
```

---

## 4.4 Funciones Anonimas y Closures

### Funciones anonimas

Funciones sin nombre, usadas como valores:

```go
package main

import "fmt"

func main() {
    // Funcion anonima asignada a una variable
    cuadrado := func(n int) int {
        return n * n
    }
    fmt.Println(cuadrado(5))  // 25

    // Funcion anonima ejecutada inmediatamente (IIFE)
    resultado := func(a, b int) int {
        return a + b
    }(3, 4)
    fmt.Println(resultado)  // 7
}
```

### Closures

Un closure es una funcion que captura variables de su entorno:

```go
package main

import "fmt"

// Retorna una funcion que "recuerda" su estado
func contador() func() int {
    n := 0
    return func() int {
        n++     // n es capturada del scope externo
        return n
    }
}

// Generador de multiplicadores
func multiplicadorPor(factor int) func(int) int {
    return func(n int) int {
        return n * factor  // factor es capturado
    }
}

func main() {
    // Cada llamada a contador() crea un closure independiente
    c1 := contador()
    c2 := contador()

    fmt.Println(c1())  // 1
    fmt.Println(c1())  // 2
    fmt.Println(c1())  // 3
    fmt.Println(c2())  // 1 (independiente de c1)

    doble := multiplicadorPor(2)
    triple := multiplicadorPor(3)
    fmt.Println(doble(5))   // 10
    fmt.Println(triple(5))  // 15
}
```

### Closures con goroutines - Cuidado

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    // INCORRECTO: todas las goroutines comparten la misma variable i
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Println(i)  // Probablemente imprime 5,5,5,5,5
        }()
    }
    wg.Wait()

    fmt.Println("---")

    // CORRECTO: pasar i como parametro
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            fmt.Println(n)  // Imprime 0,1,2,3,4 (en cualquier orden)
        }(i)
    }
    wg.Wait()

    // TAMBIEN CORRECTO en Go 1.22+: la variable del loop tiene scope por iteracion
}
```

---

## 4.5 Funciones como Valores y Tipos

### Funciones como parametros

```go
package main

import (
    "fmt"
    "strings"
)

// Recibe una funcion como parametro
func aplicar(numeros []int, operacion func(int) int) []int {
    resultado := make([]int, len(numeros))
    for i, n := range numeros {
        resultado[i] = operacion(n)
    }
    return resultado
}

// Filtrar con una funcion predicado
func filtrar(numeros []int, predicado func(int) bool) []int {
    var resultado []int
    for _, n := range numeros {
        if predicado(n) {
            resultado = append(resultado, n)
        }
    }
    return resultado
}

// Transformar strings con una funcion
func transformar(palabras []string, fn func(string) string) []string {
    resultado := make([]string, len(palabras))
    for i, p := range palabras {
        resultado[i] = fn(p)
    }
    return resultado
}

func main() {
    nums := []int{1, 2, 3, 4, 5}

    dobles := aplicar(nums, func(n int) int { return n * 2 })
    fmt.Println(dobles)  // [2 4 6 8 10]

    pares := filtrar(nums, func(n int) bool { return n%2 == 0 })
    fmt.Println(pares)  // [2 4]

    palabras := []string{"hola", "mundo"}
    mayusculas := transformar(palabras, strings.ToUpper)
    fmt.Println(mayusculas)  // [HOLA MUNDO]
}
```

### Tipos de funcion

```go
package main

import "fmt"

// Definir un tipo de funcion
type Operacion func(a, b int) int
type Validador func(string) bool

func ejecutar(a, b int, op Operacion) int {
    return op(a, b)
}

// Middleware HTTP (ejemplo conceptual)
type Handler func(request string) string
type Middleware func(Handler) Handler

func withLogging(next Handler) Handler {
    return func(request string) string {
        fmt.Println("Log: recibida peticion:", request)
        response := next(request)
        fmt.Println("Log: enviada respuesta:", response)
        return response
    }
}

func main() {
    var suma Operacion = func(a, b int) int { return a + b }
    var resta Operacion = func(a, b int) int { return a - b }

    fmt.Println(ejecutar(10, 3, suma))   // 13
    fmt.Println(ejecutar(10, 3, resta))  // 7

    // Middleware
    handler := func(req string) string {
        return "respuesta para " + req
    }
    handlerConLog := withLogging(handler)
    handlerConLog("/api/users")
}
```

---

## 4.6 init()

La funcion `init()` se ejecuta automaticamente antes de `main()`. Cada archivo puede tener multiples funciones `init()`:

```go
package main

import "fmt"

var configuracion string

func init() {
    // Se ejecuta antes de main
    configuracion = "produccion"
    fmt.Println("init: configuracion cargada")
}

func init() {
    // Puedes tener multiples init en el mismo archivo
    fmt.Println("init: segunda inicializacion")
}

func main() {
    fmt.Println("main: config =", configuracion)
}
// Salida:
// init: configuracion cargada
// init: segunda inicializacion
// main: config = produccion
```

> **Buena practica**: evita `init()` cuando sea posible. Prefiere inicializacion explicita para que el flujo sea mas claro y testeable.

---

## 4.7 Patrones Comunes con Funciones

### Patron Option (funciones que configuran)

```go
type Servidor struct {
    Puerto  int
    Host    string
    Timeout int
}

type OpcionServidor func(*Servidor)

func ConPuerto(puerto int) OpcionServidor {
    return func(s *Servidor) {
        s.Puerto = puerto
    }
}

func ConHost(host string) OpcionServidor {
    return func(s *Servidor) {
        s.Host = host
    }
}

func NuevoServidor(opciones ...OpcionServidor) *Servidor {
    s := &Servidor{
        Puerto:  8080,  // Defaults
        Host:    "localhost",
        Timeout: 30,
    }
    for _, opt := range opciones {
        opt(s)
    }
    return s
}

// Uso:
// s := NuevoServidor(ConPuerto(9090), ConHost("0.0.0.0"))
```

### Patron retry con funcion callback

```go
func conReintentos(intentos int, fn func() error) error {
    var err error
    for i := 0; i < intentos; i++ {
        if err = fn(); err == nil {
            return nil
        }
        fmt.Printf("Intento %d fallido: %v\n", i+1, err)
    }
    return fmt.Errorf("fallaron %d intentos: %w", intentos, err)
}
```

---

## 4.8 Metodos: Funciones con Receptor

Los metodos son funciones asociadas a un tipo mediante un **receptor**. Esta es una de las diferencias clave entre funciones y metodos:

```go
package main

import (
    "fmt"
    "math"
)

type Circulo struct {
    Radio float64
}

// Metodo con receptor de VALOR: recibe una copia, no modifica el original
func (c Circulo) Area() float64 {
    return math.Pi * c.Radio * c.Radio
}

// Metodo con receptor de PUNTERO: puede modificar el original
func (c *Circulo) Escalar(factor float64) {
    c.Radio *= factor
}

// Metodo con receptor de valor en tipo no-struct
type Celsius float64

func (c Celsius) AFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

type Fahrenheit float64

func (f Fahrenheit) ACelsius() Celsius {
    return Celsius((f - 32) * 5 / 9)
}

func main() {
    c := Circulo{Radio: 5}

    // Llamada a metodo: Go automaticamente toma la referencia si es necesario
    fmt.Println("Area:", c.Area())       // 78.54
    c.Escalar(2)                         // Go hace (&c).Escalar(2) automaticamente
    fmt.Println("Radio:", c.Radio)       // 10

    // Metodos en tipos basicos
    temp := Celsius(25)
    fmt.Printf("%.0f°C = %.0f°F\n", temp, temp.AFahrenheit())
}
```

### Receptor de valor vs puntero: reglas

```go
// RECEPTOR DE VALOR (func (t Tipo) Metodo()):
// - El metodo no modifica el receptor
// - El tipo es pequeño (copiar es barato)
// - El tipo es inmutable por diseño
// - Ejemplos: Circulo.Area(), time.Time.Format(), Celsius.AFahrenheit()

// RECEPTOR DE PUNTERO (func (t *Tipo) Metodo()):
// - El metodo MODIFICA el receptor
// - El tipo es grande y copiarlo seria costoso
// - El tipo contiene un sync.Mutex (nunca se debe copiar)
// - Por consistencia: si un metodo usa puntero, todos deberian
// - Ejemplos: Circulo.Escalar(), http.Request.ParseForm(), sql.DB.Ping()
```

### Method sets y consecuencias para interfaces

```go
type Contador struct {
    valor int
}

func (c Contador) Valor() int {  // Receptor de valor
    return c.valor
}

func (c *Contador) Incrementar() {  // Receptor de puntero
    c.valor++
}

func main() {
    var c Contador

    // Ambos metodos son accesibles desde el valor:
    c.Valor()        // OK
    c.Incrementar()  // OK: Go hace (&c).Incrementar()

    // PERO para interfaces:
    var incrementador interface{ Incrementar() }
    // incrementador = c     // ERROR: Contador no implementa Incrementar()
    incrementador = &c       // OK: *Contador si implementa Incrementar()

    var lector interface{ Valor() int }
    lector = c               // OK: Contador implementa Valor()
    lector = &c              // OK: *Contador tambien implementa Valor()
    _ = lector
    _ = incrementador
}
```

> **Regla**: El method set de un tipo `T` incluye solo metodos con receptor `T`. El method set de `*T` incluye metodos con receptor `T` y `*T`. Esto es relevante para interfaces.

### Metodo values vs metodo expressions

```go
package main

import "fmt"

type Calculadora struct{}

func (c Calculadora) Sumar(a, b int) int { return a + b }
func (c Calculadora) Restar(a, b int) int { return a - b }

func main() {
    calc := Calculadora{}

    // Method value: vincula el receptor y crea una funcion
    sumar := calc.Sumar
    fmt.Println(sumar(5, 3))  // 8

    // Method expression: usa el tipo para crear una funcion
    // El receptor se convierte en el primer parametro
    restar := Calculadora.Restar
    fmt.Println(restar(calc, 10, 4))  // 6
}
```

---

## 4.9 Patron: Decorator de Funciones

Envolver funciones para agregar comportamiento transversal:

```go
package main

import (
    "fmt"
    "time"
)

type Operacion func(a, b int) int

// Decorator que mide el tiempo de ejecucion
func ConMedicion(fn Operacion) Operacion {
    return func(a, b int) int {
        inicio := time.Now()
        resultado := fn(a, b)
        fmt.Printf("Tiempo: %v\n", time.Since(inicio))
        return resultado
    }
}

// Decorator que agrega logging
func ConLogging(nombre string, fn Operacion) Operacion {
    return func(a, b int) int {
        fmt.Printf("Llamando %s(%d, %d)...\n", nombre, a, b)
        resultado := fn(a, b)
        fmt.Printf("Resultado: %d\n", resultado)
        return resultado
    }
}

// Decorator que valida entrada
func ConValidacion(fn Operacion) Operacion {
    return func(a, b int) int {
        if a < 0 || b < 0 {
            panic("solo se permiten numeros positivos")
        }
        return fn(a, b)
    }
}

func main() {
    var sumar Operacion = func(a, b int) int { return a + b }

    // Apilar decorators
    sumarDecorada := ConMedicion(
        ConLogging("sumar",
            ConValidacion(sumar),
        ),
    )

    fmt.Println(sumarDecorada(5, 3))
}
```

---

## 4.10 Recursion y Tail Call

Go soporta recursion pero NO optimiza tail calls:

```go
package main

import "fmt"

// Recursion clasica: factorial
func Factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * Factorial(n-1)
}

// Recursion con manejo de errores
func FactorialSeguro(n int) (int, error) {
    if n < 0 {
        return 0, fmt.Errorf("factorial no definido para negativos: %d", n)
    }
    if n > 20 { // Previene overflow de int64
        return 0, fmt.Errorf("factorial demasiado grande: %d", n)
    }
    return factorialRec(n), nil
}

func factorialRec(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorialRec(n-1)
}

// Arbol binario: recursion natural
type Nodo struct {
    Valor int
    Izq   *Nodo
    Der   *Nodo
}

func (n *Nodo) Suma() int {
    if n == nil {
        return 0
    }
    return n.Valor + n.Izq.Suma() + n.Der.Suma()
}

func (n *Nodo) Altura() int {
    if n == nil {
        return 0
    }
    izq := n.Izq.Altura()
    der := n.Der.Altura()
    if izq > der {
        return izq + 1
    }
    return der + 1
}

func main() {
    fmt.Println(Factorial(5))  // 120

    if r, err := FactorialSeguro(-1); err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println(r)
    }

    arbol := &Nodo{
        Valor: 1,
        Izq:   &Nodo{Valor: 2},
        Der:   &Nodo{Valor: 3, Izq: &Nodo{Valor: 4}},
    }
    fmt.Println("Suma:", arbol.Suma())     // 10
    fmt.Println("Altura:", arbol.Altura()) // 3
}
```

> **Atencion**: Go no tiene Tail Call Optimization (TCO). Recursiones profundas pueden causar stack overflow. Para iteraciones profundas, prefiere bucles `for` o transforma la recursion en iteracion manual con un slice/pila.

---

## Resumen del Capítulo

- Go soporta retorno multiple, fundamental para el patron `(resultado, error)`.
- Las funciones variadicas aceptan un numero variable de argumentos.
- Los closures capturan variables de su entorno (cuidado con goroutines).
- Las funciones son valores de primera clase: pueden pasarse como parametros y retornarse.
- Los tipos de funcion (`type Handler func(...)`) mejoran la legibilidad.
- El patron Functional Options aprovecha closures para configuracion flexible.
- Los **metodos** son funciones con receptor. Usa puntero para modificacion, valor para solo lectura.
- El method set de `T` incluye metodos con receptor `T`; el de `*T` incluye ambos.
- Los decorators de funciones agregan comportamiento sin modificar la funcion original.
- Go soporta recursion pero sin optimizacion de tail calls; cuidado con recursiones profundas.

---

← [Capítulo anterior](03-estructuras-de-control.md) | [Inicio](README.md) | [Capítulo siguiente →](05-estructuras-de-datos.md)
