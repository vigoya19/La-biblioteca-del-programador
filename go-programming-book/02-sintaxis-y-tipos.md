# Capítulo 2: Sintaxis y Tipos de Datos

## 2.1 Variables y Constantes

### Declaracion de variables

Go ofrece varias formas de declarar variables:

```go
package main

import "fmt"

func main() {
    // 1. Declaracion con var y tipo explicito
    var nombre string = "Andres"

    // 2. Declaracion con var e inferencia de tipo
    var edad = 30

    // 3. Declaracion corta con := (solo dentro de funciones)
    ciudad := "Madrid"

    // 4. Declaracion sin valor inicial (zero value)
    var activo bool    // false
    var contador int   // 0
    var saludo string  // "" (cadena vacia)

    // 5. Declaracion multiple
    var (
        x int    = 10
        y int    = 20
        z string = "hola"
    )

    // 6. Declaracion multiple en una linea
    a, b, c := 1, 2, "tres"

    fmt.Println(nombre, edad, ciudad, activo, contador, saludo)
    fmt.Println(x, y, z)
    fmt.Println(a, b, c)
}
```

### Zero values

En Go, toda variable tiene un valor por defecto llamado **zero value**:

> [!NOTE]
> 📦 **La Caja Vacía vs. La Caja no Inicializada (El Misterio de `nil`)**
> 
> Al comenzar en Go, verás la palabra reservada `nil` repetidamente. Piensa en `nil` como **"esta estructura aún no ha sido creada"** o **"esta variable no apunta a ningún lugar real en la memoria"**:
> - **Zero Value tradicional (`int = 0`, `string = ""`):** Tienes una taza vacía en tu mesa. La taza existe física y concretamente, pero no contiene café. Puedes usarla o modificarla en cualquier momento.
> - **Zero Value compuesto (`slice = nil`, `map = nil`):** ¡No tienes taza! Solo tienes un boleto de papel que dice "Aquí irá una taza en el futuro". Si intentas verter café en el boleto de papel (es decir, intentar escribir o guardar datos en un `map` que es `nil`), tu programa se romperá de inmediato ("panic").
> 
> En Go, las estructuras dinámicas avanzadas como slices, maps y channels inician como `nil` (el boleto sin la taza). Aprenderemos a fabricar las tazas reales usando la instrucción especial `make()` en los siguientes capítulos.

| Tipo | Zero Value |
|------|-----------|
| `int`, `float64` | `0` |
| `string` | `""` |
| `bool` | `false` |
| `pointer` | `nil` |
| `slice`, `map`, `channel`, `function`, `interface` | `nil` |

```go
var i int       // 0
var f float64   // 0.0
var s string    // ""
var b bool      // false
var p *int      // nil
var sl []int    // nil (pero len(sl) == 0, se puede usar append)
```

### Constantes

Las constantes se definen con `const` y se evaluan en tiempo de compilacion:

```go
const Pi = 3.14159
const (
    StatusOK    = 200
    StatusError = 500
)

// iota: generador de constantes incrementales
const (
    Lunes = iota + 1  // 1
    Martes             // 2
    Miercoles          // 3
    Jueves             // 4
    Viernes            // 5
    Sabado             // 6
    Domingo            // 7
)

// iota con bit shifting (muy comun para flags)
const (
    Leer    = 1 << iota  // 1  (001)
    Escribir              // 2  (010)
    Ejecutar              // 4  (100)
)

// Se pueden combinar: Leer | Escribir = 3 (011)

> [!TIP]
> 🎛️ **El Tablero de Interruptores de Luz (Bit Flags con iota)**
> 
> Cuando veas código como `1 << iota` o combinaciones como `Leer | Escribir`, no te asustes con las matemáticas binarias. Imagina que tienes un **tablero con 3 interruptores de luz alineados en la pared**:
> - **Interruptor 1 (Leer)**: `1 << 0` (el primer interruptor está encendido: representa `001` en binario, valor decimal = 1).
> - **Interruptor 2 (Escribir)**: `1 << 1` (el segundo interruptor está encendido: representa `010` en binario, valor decimal = 2).
> - **Interruptor 3 (Ejecutar)**: `1 << 2` (el tercer interruptor está encendido: representa `100` en binario, valor decimal = 4).
> 
> **¿Por qué combinarlos con el operador OR (`|`)?**
> Si quieres darle a un usuario permisos de **Leer** Y **Escribir** a la vez, simplemente activas los dos primeros interruptores en el tablero. En Go, haces `Leer | Escribir`, lo que da como resultado `011` en binario (valor decimal = 3).
> 
> Gracias a este truco de bajo nivel, puedes guardar múltiples opciones de configuración independientes en una sola variable numérica ultra-ligera, simplemente encendiendo y apagando interruptores individuales en memoria.
```

### Regla importante: las variables no usadas son un error

```go
func main() {
    x := 5
    // Si no usas x, el programa NO compila:
    // "x declared but not used"
}
```

Esto es intencional: Go te obliga a mantener el codigo limpio.

---

## 2.2 Tipos Basicos

### Tipos numericos enteros

```go
// Con signo
var a int8   = 127                  // -128 a 127
var b int16  = 32767                // -32768 a 32767
var c int32  = 2147483647           // -2^31 a 2^31-1
var d int64  = 9223372036854775807  // -2^63 a 2^63-1
var e int    = 42                   // 32 o 64 bits segun la plataforma

// Sin signo
var f uint8  = 255                  // 0 a 255
var g uint16 = 65535                // 0 a 65535
var h uint32 = 4294967295           // 0 a 2^32-1
var i uint64 = 18446744073709551615 // 0 a 2^64-1
var j uint   = 42                   // 32 o 64 bits segun la plataforma

// Alias
var k byte = 255  // alias de uint8
var l rune = 'A'  // alias de int32, representa un code point Unicode
```

### Tipos numericos de punto flotante

```go
var f1 float32 = 3.14       // Precision simple (~7 digitos)
var f2 float64 = 3.14159265 // Precision doble (~15 digitos) - preferido

// Tipos complejos
var c1 complex64  = 1 + 2i
var c2 complex128 = 1 + 2i
```

> **Buena practica**: usa `int` para enteros y `float64` para decimales, a menos que tengas una razon especifica para usar otro tipo.

### Strings

Los strings en Go son secuencias inmutables de bytes, tipicamente codificados en UTF-8:

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    // Strings con comillas dobles (interpretan secuencias de escape)
    s1 := "Hola\nMundo"

    // Strings literales con backticks (raw strings, sin escape)
    s2 := `Hola
Mundo
    Esto conserva el formato`

    // Strings son inmutables
    s := "Hola"
    // s[0] = 'h'  // ERROR: no se puede modificar

    // Longitud
    fmt.Println(len(s))  // 4 (bytes, no caracteres)

    // Longitud en caracteres (runes)
    emoji := "Hola 🌍"
    fmt.Println(len(emoji))                    // 9 (bytes)
    fmt.Println(len([]rune(emoji)))            // 6 (caracteres)

    // Concatenacion
    saludo := "Hola" + " " + "Mundo"

    // Operaciones comunes con strings
    fmt.Println(strings.ToUpper("hola"))           // HOLA
    fmt.Println(strings.ToLower("HOLA"))           // hola
    fmt.Println(strings.Contains("Hola Mundo", "Mundo")) // true
    fmt.Println(strings.Replace("aaa", "a", "b", 2))     // bba
    fmt.Println(strings.Split("a,b,c", ","))              // [a b c]
    fmt.Println(strings.TrimSpace("  hola  "))            // hola
    fmt.Println(strings.HasPrefix("Hola", "Ho"))          // true
    fmt.Println(strings.HasSuffix("Hola", "la"))          // true

    // Para construir strings eficientemente usa strings.Builder
    var builder strings.Builder
    for i := 0; i < 100; i++ {
        builder.WriteString("x")
    }
    resultado := builder.String()

    fmt.Println(s1, s2, saludo, resultado)
}
```

### Bytes y Runes

```go
package main

import "fmt"

func main() {
    s := "Hola 🌍"

    // Iterar por bytes
    for i := 0; i < len(s); i++ {
        fmt.Printf("byte[%d] = %d\n", i, s[i])
    }

    // Iterar por runes (caracteres Unicode) - FORMA CORRECTA
    for i, r := range s {
        fmt.Printf("rune[%d] = %c (U+%04X)\n", i, r, r)
    }

    // Conversion entre string, []byte y []rune
    bytes := []byte(s)
    runes := []rune(s)
    fmt.Println(bytes)           // [72 111 108 97 32 240 159 140 141]
    fmt.Println(runes)           // [72 111 108 97 32 127757]
    fmt.Println(string(bytes))   // Hola 🌍
    fmt.Println(string(runes))   // Hola 🌍
}
```

### Bool

```go
var verdadero bool = true
var falso bool = false

// Operadores logicos
fmt.Println(true && false)  // false (AND)
fmt.Println(true || false)  // true  (OR)
fmt.Println(!true)          // false (NOT)
```

---

## 2.3 Operadores

### Operadores aritmeticos

```go
a, b := 10, 3

fmt.Println(a + b)   // 13  (suma)
fmt.Println(a - b)   // 7   (resta)
fmt.Println(a * b)   // 30  (multiplicacion)
fmt.Println(a / b)   // 3   (division entera)
fmt.Println(a % b)   // 1   (modulo)

// Con float64
x, y := 10.0, 3.0
fmt.Println(x / y)   // 3.3333333333333335
```

### Operadores de comparacion

```go
fmt.Println(5 == 5)   // true
fmt.Println(5 != 3)   // true
fmt.Println(5 > 3)    // true
fmt.Println(5 < 3)    // false
fmt.Println(5 >= 5)   // true
fmt.Println(5 <= 3)   // false
```

### Operadores de bits

```go
a, b := 0b1010, 0b1100  // 10 y 12

fmt.Printf("%04b\n", a & b)   // 1000 (AND)
fmt.Printf("%04b\n", a | b)   // 1110 (OR)
fmt.Printf("%04b\n", a ^ b)   // 0110 (XOR)
fmt.Printf("%04b\n", a &^ b)  // 0010 (AND NOT)
fmt.Printf("%04b\n", a << 1)  // 10100 (shift left)
fmt.Printf("%04b\n", a >> 1)  // 0101 (shift right)
```

### Operadores de asignacion

```go
x := 10
x += 5   // x = x + 5  -> 15
x -= 3   // x = x - 3  -> 12
x *= 2   // x = x * 2  -> 24
x /= 4   // x = x / 4  -> 6
x %= 4   // x = x % 4  -> 2
x++      // x = x + 1  -> 3 (es una sentencia, no una expresion)
x--      // x = x - 1  -> 2
```

> **Nota**: en Go, `x++` y `x--` son sentencias, no expresiones. No puedes hacer `y := x++`.

---

## 2.4 Conversion de Tipos

Go no tiene conversiones implicitas. Toda conversion debe ser explicita:

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Conversiones numericas
    var i int = 42
    var f float64 = float64(i)
    var u uint = uint(i)
    fmt.Println(i, f, u)

    // CUIDADO: perdida de precision
    var big float64 = 1.9
    var truncated int = int(big)
    fmt.Println(truncated)  // 1 (trunca, no redondea)

    // String a numero
    s := "42"
    num, err := strconv.Atoi(s)  // String to Int
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println(num)  // 42

    // Numero a string
    str := strconv.Itoa(42)  // Int to String
    fmt.Println(str)          // "42"

    // Conversiones mas especificas
    f64, _ := strconv.ParseFloat("3.14", 64)
    b, _ := strconv.ParseBool("true")
    i64, _ := strconv.ParseInt("FF", 16, 64)  // Hexadecimal
    fmt.Println(f64, b, i64)  // 3.14 true 255

    // Formateo con fmt.Sprintf
    texto := fmt.Sprintf("Tengo %d anios y mido %.2f metros", 30, 1.75)
    fmt.Println(texto)  // Tengo 30 anios y mido 1.75 metros

    // int a string con string() - CUIDADO
    fmt.Println(string(65))  // "A" (convierte a rune, NO al string "65")
}
```

---

## 2.5 Punteros

Los punteros almacenan la direccion de memoria de una variable:

```go
package main

import "fmt"

func main() {
    x := 42
    p := &x    // p es un puntero a x (tipo *int)

    fmt.Println(x)   // 42        (valor)
    fmt.Println(p)   // 0xc000... (direccion de memoria)
    fmt.Println(*p)  // 42        (desreferenciar: obtener el valor)

    // Modificar a traves del puntero
    *p = 100
    fmt.Println(x)  // 100 (x cambio porque modificamos via puntero)

    // Puntero nil
    var ptr *int
    fmt.Println(ptr)        // <nil>
    // fmt.Println(*ptr)    // PANIC: nil pointer dereference

    // new() crea un puntero a un zero value
    p2 := new(int)
    fmt.Println(*p2)  // 0
    *p2 = 7
    fmt.Println(*p2)  // 7
}

// Punteros como parametros (paso por referencia)
func duplicar(n *int) {
    *n *= 2
}

func ejemplo() {
    x := 5
    duplicar(&x)
    fmt.Println(x)  // 10
}

// Sin puntero (paso por valor - se copia)
func duplicarCopia(n int) {
    n *= 2  // Solo modifica la copia local
}

func ejemploCopia() {
    x := 5
    duplicarCopia(x)
    fmt.Println(x)  // 5 (no cambio)
}
```

### Cuando usar punteros

```go
// 1. Cuando necesitas modificar el valor original
func incrementar(contador *int) {
    *contador++
}

// 2. Para evitar copiar structs grandes
type UsuarioGrande struct {
    // Muchos campos...
    Nombre   string
    Datos    [1000]byte
}

func procesar(u *UsuarioGrande) {
    // Trabaja con el original, no una copia
}

// 3. Para indicar ausencia de valor (nil)
func buscar(id int) *Usuario {
    // Retorna nil si no se encuentra
    return nil
}
```

### Diferencias con C/C++

- **No hay aritmetica de punteros** en Go (no puedes hacer `p++` para avanzar en memoria).
- El garbage collector maneja la memoria automaticamente.
- Es seguro retornar punteros a variables locales (Go detecta el "escape" y las mueve al heap).

```go
// Esto es SEGURO en Go (seria peligroso en C)
func crearNumero() *int {
    x := 42
    return &x  // Go mueve x al heap automaticamente
}
```

---

## 2.6 Formateo con fmt

La funcion `fmt.Printf` usa verbos de formato:

```go
package main

import "fmt"

func main() {
    // Verbos generales
    fmt.Printf("%v\n", 42)          // 42          (valor por defecto)
    fmt.Printf("%+v\n", struct{X int}{1})  // {X:1}  (con nombres de campo)
    fmt.Printf("%#v\n", "hola")     // "hola"       (sintaxis Go)
    fmt.Printf("%T\n", 42)          // int           (tipo)
    fmt.Printf("%%\n")              // %             (literal)

    // Enteros
    fmt.Printf("%d\n", 42)          // 42            (decimal)
    fmt.Printf("%b\n", 42)          // 101010        (binario)
    fmt.Printf("%o\n", 42)          // 52            (octal)
    fmt.Printf("%x\n", 42)          // 2a            (hexadecimal)
    fmt.Printf("%05d\n", 42)        // 00042         (con padding)

    // Flotantes
    fmt.Printf("%f\n", 3.14)        // 3.140000
    fmt.Printf("%.2f\n", 3.14)      // 3.14
    fmt.Printf("%e\n", 100000.0)    // 1.000000e+05 (notacion cientifica)

    // Strings
    fmt.Printf("%s\n", "hola")      // hola
    fmt.Printf("%q\n", "hola")      // "hola"       (con comillas)
    fmt.Printf("%10s\n", "hola")    //       hola   (ancho minimo)
    fmt.Printf("%-10s|\n", "hola")  // hola      |  (alineado a la izquierda)

    // Punteros
    x := 42
    fmt.Printf("%p\n", &x)          // 0xc000012088

    // Bool
    fmt.Printf("%t\n", true)        // true
}
```

---

## 2.7 Named Types (Definiciones de Tipo)

Go permite crear tipos con nombre a partir de tipos existentes. Esto es util para dar significado semantico y agregar metodos:

```go
package main

import "fmt"

// Definicion de tipo: nuevo tipo basado en uno existente
type Celsius float64
type Fahrenheit float64
type UsuarioID int
type Email string

// NO confundir con alias de tipo (Go 1.9+)
type MiEntero = int  // Alias: MiEntero ES int (intercambiables)

func main() {
    var temp Celsius = 25.5
    var id UsuarioID = 12345
    var correo Email = "usuario@ejemplo.com"

    fmt.Println(temp, id, correo)

    // Con alias, son exactamente el mismo tipo:
    var x MiEntero = 42
    var y int = x  // OK: son el mismo tipo con alias
    fmt.Println(y)

    // Sin alias, son tipos distintos:
    var c Celsius = 30
    var f Fahrenheit = 86
    // c = f  // ERROR: tipos diferentes
    // c = 30  // ERROR: 30 es int, no Celsius
    c = Celsius(30)  // OK: conversion explicita
    _ = f
}
```

### Cuando usar named types

```go
// 1. Semantica de dominio: evita confusiones
type UsuarioID int
type OrdenID int

func BuscarUsuario(id UsuarioID) (*Usuario, error) { ... }
func BuscarOrden(id OrdenID) (*Orden, error) { ... }

// Sin named types, podrias pasar un ID de orden a buscar usuario
// Con named types, el compilador lo rechaza

// 2. Agregar metodos a tipos basicos
type Distancia float64

func (d Distancia) Kilometros() float64 { return float64(d) }
func (d Distancia) Metros() float64     { return float64(d) * 1000 }
func (d Distancia) Millas() float64     { return float64(d) * 0.621371 }

// 3. Slice con metodos
type ListaNombres []string

func (l ListaNombres) Contiene(nombre string) bool {
    for _, n := range l {
        if n == nombre {
            return true
        }
    }
    return false
}
```

---

## 2.8 Tipos Compuestos (Vista Previa)

Ademas de los tipos basicos, Go incluye tipos compuestos que se estudian en detalle en el Capitulo 5:

| Tipo | Descripcion | Ejemplo |
|------|-------------|---------|
| **Array** | Coleccion de tamaño fijo | `[5]int{1,2,3,4,5}` |
| **Slice** | Vista dinamica sobre arrays | `[]int{1,2,3}` |
| **Map** | Tabla hash / diccionario | `map[string]int{"a": 1}` |
| **Struct** | Agrupacion de campos | `struct{X int; Y string}` |
| **Channel** | Comunicacion entre goroutines | `make(chan int)` |
| **Funcion** | Tipo funcion | `func(int) string` |

```go
package main

import "fmt"

func main() {
    // Array: tamaño fijo, copia por valor
    arr := [3]int{10, 20, 30}
    fmt.Println(arr[0], len(arr))  // 10, 3

    // Slice: tamaño dinamico, referencia a array subyacente
    slice := []int{1, 2, 3}
    slice = append(slice, 4)
    fmt.Println(slice, len(slice))  // [1 2 3 4], 4

    // Map: clave-valor
    edades := map[string]int{
        "Ana": 25,
        "Luis": 30,
    }
    edad, existe := edades["Ana"]  // Comma-ok idiom
    fmt.Println(edad, existe)       // 25 true

    // Struct: campos con nombre
    type Punto struct {
        X, Y int
    }
    p := Punto{X: 10, Y: 20}
    fmt.Println(p.X, p.Y)  // 10 20
}
```

> **Nota**: `nil slice` vs `empty slice` tienen diferencias practicas. Un slice nil (`var s []int`) serializa a `null` en JSON, mientras que uno vacio (`s := []int{}`) serializa a `[]`. Ambos tienen `len(s) == 0`, pero solo el nil es igual a `nil`. Un map nil no permite escritura (panic), solo lectura.

---

## Resumen del Capítulo

- Go tiene tipos basicos claros: `int`, `float64`, `string`, `bool`, `byte`, `rune`.
- Toda variable tiene un **zero value** por defecto.
- Los tipos con nombre (`type Celsius float64`) dan semantica de dominio y permiten metodos.
- No hay conversiones implicitas; toda conversion es explicita.
- Los punteros permiten trabajar con referencias, pero sin aritmetica de punteros.
- `iota` es una herramienta poderosa para generar constantes.
- El paquete `fmt` proporciona formateo flexible con Printf.
- Arrays, slices, maps, structs y channels son tipos compuestos (ver Capitulo 5).

---

← [Capítulo anterior](01-introduccion.md) | [Inicio](README.md) | [Capítulo siguiente →](03-estructuras-de-control.md)
