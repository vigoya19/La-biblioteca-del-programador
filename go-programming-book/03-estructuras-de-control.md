# Capítulo 3: Estructuras de Control

## 3.1 Condicionales

### if / else

```go
package main

import "fmt"

func main() {
    edad := 25

    // if basico
    if edad >= 18 {
        fmt.Println("Eres mayor de edad")
    }

    // if / else
    if edad >= 18 {
        fmt.Println("Mayor de edad")
    } else {
        fmt.Println("Menor de edad")
    }

    // if / else if / else
    if edad < 13 {
        fmt.Println("Nino")
    } else if edad < 18 {
        fmt.Println("Adolescente")
    } else if edad < 65 {
        fmt.Println("Adulto")
    } else {
        fmt.Println("Adulto mayor")
    }
}
```

### if con declaracion inicial

Una caracteristica muy usada en Go: puedes declarar una variable en el `if` que solo existe en ese scope:

```go
package main

import (
    "fmt"
    "os"
    "strconv"
)

func main() {
    // La variable err solo existe dentro del if/else
    if err := verificar(); err != nil {
        fmt.Println("Error:", err)
        return
    }

    // Patron muy comun: abrir archivo
    if archivo, err := os.Open("datos.txt"); err != nil {
        fmt.Println("Error al abrir:", err)
    } else {
        defer archivo.Close()
        fmt.Println("Archivo abierto:", archivo.Name())
    }

    // Conversion con verificacion
    if num, err := strconv.Atoi("42"); err == nil {
        fmt.Println("Numero:", num)
    }
}

func verificar() error {
    return nil
}
```

### switch

El `switch` en Go es mas poderoso que en otros lenguajes. **No necesita `break`** (se sale automaticamente):

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    // Switch basico
    dia := time.Now().Weekday()
    switch dia {
    case time.Saturday, time.Sunday:  // Multiples valores
        fmt.Println("Fin de semana!")
    case time.Monday:
        fmt.Println("Lunes...")
    default:
        fmt.Println("Dia de semana")
    }

    // Switch con inicializacion
    switch os := runtime.GOOS; os {
    case "darwin":
        fmt.Println("Estas en macOS")
    case "linux":
        fmt.Println("Estas en Linux")
    case "windows":
        fmt.Println("Estas en Windows")
    default:
        fmt.Println("Otro OS:", os)
    }

    // Switch sin condicion (reemplaza if/else if largos)
    hora := time.Now().Hour()
    switch {
    case hora < 6:
        fmt.Println("Madrugada")
    case hora < 12:
        fmt.Println("Manana")
    case hora < 18:
        fmt.Println("Tarde")
    default:
        fmt.Println("Noche")
    }

    // fallthrough: continuar al siguiente case (raro en Go)
    x := 1
    switch x {
    case 1:
        fmt.Println("Uno")
        fallthrough
    case 2:
        fmt.Println("Dos (tambien se ejecuta)")
    case 3:
        fmt.Println("Tres (NO se ejecuta)")
    }
    // Imprime: Uno, Dos (tambien se ejecuta)
}
```

### Type switch

Permite actuar segun el tipo de una interface:

> [!NOTE]
> ### 🔍 El Clasificador de Objetos Desconocidos (Type Switch)
> 
> Imagina que tienes una caja negra con un agujero. Alguien mete la mano y coloca un objeto sorpresa dentro de tu mano (`interface{}`). No puedes ver el objeto, pero tu programa debe reaccionar según lo que sea:
> - Si es un **bloque de madera** (un tipo `int`), quieres medirlo.
> - Si es una **tarjeta de felicitación** (un tipo `string`), quieres leerla.
> - Si es una **moneda** (un tipo `bool`), quieres ver si es cara o cruz.
> 
> En Go, un **Type Switch** (`switch v := i.(type)`) es como pasar ese objeto sorpresa por un escáner inteligente:
> 1. El escáner detecta qué material exacto es en una milésima de segundo.
> 2. Te entrega el objeto desempaquetado en una nueva variable `v` con su tipo real (ej. si era un entero, `v` será un número real con todas sus propiedades matemáticas y aritméticas).
> 
> Esto nos permite trabajar de forma ultra-segura con datos de origen desconocido, sin arriesgarnos a que el programa se rompa por intentar sumar un texto con un número.

```go
func describir(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("Entero: %d", v)
    case string:
        return fmt.Sprintf("String: %s", v)
    case bool:
        return fmt.Sprintf("Bool: %t", v)
    case []int:
        return fmt.Sprintf("Slice de enteros con %d elementos", len(v))
    default:
        return fmt.Sprintf("Tipo desconocido: %T", v)
    }
}
```

---

## 3.2 Bucles

### for - El unico bucle en Go

Go solo tiene `for`. No existe `while` ni `do-while`, pero `for` puede comportarse como todos ellos:

```go
package main

import "fmt"

func main() {
    // 1. for clasico (como C)
    for i := 0; i < 5; i++ {
        fmt.Println(i)  // 0, 1, 2, 3, 4
    }

    // 2. for como while
    contador := 0
    for contador < 5 {
        fmt.Println(contador)
        contador++
    }

    // 3. for infinito
    intentos := 0
    for {
        intentos++
        if intentos >= 3 {
            break  // Salir del bucle
        }
        fmt.Println("Intento", intentos)
    }

    // 4. continue: saltar a la siguiente iteracion
    for i := 0; i < 10; i++ {
        if i%2 == 0 {
            continue  // Saltar los pares
        }
        fmt.Println(i)  // 1, 3, 5, 7, 9
    }
}
```

### for range

Itera sobre colecciones (slices, maps, strings, channels):

```go
package main

import "fmt"

func main() {
    // Range sobre slice
    frutas := []string{"manzana", "banana", "cereza"}
    for i, fruta := range frutas {
        fmt.Printf("[%d] %s\n", i, fruta)
    }

    // Solo indices
    for i := range frutas {
        fmt.Println(i)  // 0, 1, 2
    }

    // Solo valores (ignorar indice con _)
    for _, fruta := range frutas {
        fmt.Println(fruta)
    }

    // Range sobre map
    capitales := map[string]string{
        "Mexico":   "CDMX",
        "Espana":   "Madrid",
        "Colombia": "Bogota",
    }
    for pais, capital := range capitales {
        fmt.Printf("%s -> %s\n", pais, capital)
    }
    // NOTA: el orden de iteracion en maps NO esta garantizado

    // Range sobre string (itera por runes, no bytes)
    for i, ch := range "Hola 🌍" {
        fmt.Printf("posicion %d: %c\n", i, ch)
    }

    // Range sobre canal
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3
    close(ch)
    for valor := range ch {
        fmt.Println(valor)  // 1, 2, 3
    }

    // Range con numero entero (Go 1.22+)
    for i := range 5 {
        fmt.Println(i)  // 0, 1, 2, 3, 4
    }
}
```

### Labels y break/continue con etiquetas

Para bucles anidados, puedes usar etiquetas:

```go
package main

import "fmt"

func main() {
    // break con label: sale del bucle externo
    externo:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if i == 1 && j == 1 {
                break externo  // Sale de ambos bucles
            }
            fmt.Printf("(%d, %d) ", i, j)
        }
    }
    // Imprime: (0, 0) (0, 1) (0, 2) (1, 0)

    fmt.Println()

    // continue con label: salta a la siguiente iteracion del bucle externo
    filas:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if j == 1 {
                continue filas  // Salta al siguiente i
            }
            fmt.Printf("(%d, %d) ", i, j)
        }
    }
    // Imprime: (0, 0) (1, 0) (2, 0)
}
```

---

## 3.3 Defer, Panic y Recover

### defer

`defer` pospone la ejecucion de una funcion hasta que la funcion que la contiene retorne. Los defers se ejecutan en orden **LIFO** (ultimo en entrar, primero en salir):

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // Uso basico
    fmt.Println("Inicio")
    defer fmt.Println("Esto se ejecuta al final")
    fmt.Println("Medio")
    // Imprime: Inicio, Medio, Esto se ejecuta al final

    // Orden LIFO
    defer fmt.Println("Primero en defer, ultimo en ejecutar")
    defer fmt.Println("Segundo en defer, penultimo en ejecutar")
    defer fmt.Println("Ultimo en defer, primero en ejecutar")

    // Caso de uso principal: cerrar recursos
    leerArchivo()
}

func leerArchivo() {
    archivo, err := os.Open("datos.txt")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer archivo.Close()  // Se cierra al salir de la funcion, pase lo que pase

    // ... trabajar con el archivo ...
    // No importa si hay un return anticipado o un panic,
    // defer garantiza que el archivo se cierra.
}
```

### Cuidado con defer en bucles

```go
// MALO: acumula muchos defers, los archivos no se cierran hasta que termine la funcion
func procesarArchivos(nombres []string) {
    for _, nombre := range nombres {
        f, err := os.Open(nombre)
        if err != nil {
            continue
        }
        defer f.Close()  // No se cierra hasta que termine procesarArchivos!
    }
}

// BUENO: extraer a una funcion separada
func procesarArchivos2(nombres []string) {
    for _, nombre := range nombres {
        procesarUnArchivo(nombre)
    }
}

func procesarUnArchivo(nombre string) {
    f, err := os.Open(nombre)
    if err != nil {
        return
    }
    defer f.Close()  // Se cierra al terminar esta funcion (cada iteracion)
    // ... procesar ...
}
```

### defer captura valores

> [!TIP]
> ### 📸 La Foto Instantánea vs. El Mensajero al Final (Captura de defer)
> 
> Cuando usas la palabra clave `defer` para posponer una tarea en Go, debes tener cuidado con **cuándo** lee Go el valor de tus variables. Imagina estas dos situaciones:
> 
> 1. **La Foto Instantánea (`defer fmt.Println(x)`)**: Cuando Go lee esta línea, toma una foto instantánea del valor de `x` en ese preciso momento (digamos, `x = 0`). Guarda la foto en un cajón y, cuando la función termina, la imprime. Aunque cambies `x = 42` más abajo en tu código, la foto ya fue tomada con `0`.
> 2. **El Mensajero al Final (`defer func() { fmt.Println(x) }()`)**: Aquí no tomas una foto de `x`. En su lugar, dejas una nota que dice: *"Cuando termine la función, ve a buscar la variable `x` e imprime lo que sea que tenga en ese instante"*. Al finalizar la función, el mensajero va a revisar `x`, encuentra `42` y lo imprime.
> 
> Usa la **foto instantánea** para la mayoría de los casos simples, y usa el **mensajero (closure)** cuando necesites que el valor se evalúe dinámicamente justo al final de la función.

```go
func main() {
    x := 0
    defer fmt.Println("defer con valor:", x)  // Captura x=0 en el momento del defer
    x = 42
    fmt.Println("x =", x)
    // Imprime:
    // x = 42
    // defer con valor: 0

    // Para capturar el valor actual, usa un closure:
    y := 0
    defer func() {
        fmt.Println("defer con closure:", y)  // Captura la variable, no el valor
    }()
    y = 42
    // Imprime: defer con closure: 42
}
```

### panic

`panic` detiene la ejecucion normal de una funcion. Es similar a una excepcion no manejada, pero en Go se usa solo para errores irrecuperables:

```go
package main

import "fmt"

func main() {
    fmt.Println("Inicio")
    hacerAlgoPeligroso()
    fmt.Println("Esto nunca se ejecuta")
}

func hacerAlgoPeligroso() {
    defer fmt.Println("Defer: esto SI se ejecuta incluso con panic")
    fmt.Println("Antes del panic")
    panic("algo salio terriblemente mal")
    fmt.Println("Despues del panic - NUNCA se ejecuta")
}
// Salida:
// Inicio
// Antes del panic
// Defer: esto SI se ejecuta incluso con panic
// panic: algo salio terriblemente mal
// [stack trace...]
```

> **Regla de oro**: NO uses `panic` para manejar errores normales. Usa el patron `error`. Solo usa `panic` para situaciones verdaderamente irrecuperables (estado corrupto, invariantes rotas).

### recover

`recover` captura un `panic` y permite que el programa continue. Solo funciona dentro de una funcion `defer`:

```go
package main

import "fmt"

func main() {
    fmt.Println("Inicio")
    resultado := operacionSegura()
    fmt.Println("Resultado:", resultado)
    fmt.Println("El programa continua normalmente")
}

func operacionSegura() (resultado string) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recuperado de panic:", r)
            resultado = "valor por defecto"
        }
    }()

    // Esto causaria un panic
    operacionPeligrosa()

    return "exito"
}

func operacionPeligrosa() {
    panic("error catastrofico!")
}
```

### Patron comun: proteger goroutines

```go
func trabajadorSeguro(id int) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Goroutine %d recuperada de: %v\n", id, r)
        }
    }()

    // Si esta goroutine hace panic, no tumba todo el programa
    hacerTrabajo(id)
}

func hacerTrabajo(id int) {
    if id == 3 {
        panic("el trabajador 3 fallo")
    }
    fmt.Printf("Trabajador %d completado\n", id)
}
```

---

## Resumen del Capítulo

- Go solo tiene `for` como bucle, pero cubre todos los casos: clasico, while, infinito.
- `for range` itera sobre slices, maps, strings y channels.
- `switch` no necesita `break` y puede usarse sin condicion.
- `defer` garantiza limpieza de recursos (LIFO).
- `panic`/`recover` son para errores irrecuperables, no para control de flujo normal.
- El patron `if err != nil` es la forma idiomatica de manejar errores (detalle en capitulo 8).

---

← [Capítulo anterior](02-sintaxis-y-tipos.md) | [Inicio](README.md) | [Capítulo siguiente →](04-funciones.md)
