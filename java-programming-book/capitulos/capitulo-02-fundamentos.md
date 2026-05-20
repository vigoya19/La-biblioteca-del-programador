# Capítulo 2: Fundamentos del Lenguaje Java

En este capítulo exploraremos los bloques de construcción esenciales de Java: tipos de datos, variables, operadores, estructuras de control y el manejo de cadenas. Profundizaremos en los detalles internos de la JVM, en las trampas más comunes que atrapan incluso a programadores experimentados, y en las técnicas avanzadas que separan a un desarrollador competente de uno excepcional. Dominar estos fundamentos es indispensable antes de avanzar a conceptos más complejos como la programación orientada a objetos.

---

## 2.1 Tipos de Datos Primitivos

Java es un lenguaje **fuertemente tipado**, lo que significa que toda variable debe declararse con un tipo de dato específico. Los tipos de datos en Java se dividen en dos grandes categorías: **primitivos** y **de referencia**.

### 2.1.1 Los Ocho Tipos Primitivos

Java define ocho tipos de datos primitivos, que se almacenan directamente en la pila (stack) y no son objetos. Cada uno tiene un tamaño fijo en todas las plataformas, lo que garantiza la portabilidad del lenguaje.

#### Tipos Enteros

| Tipo    | Tamaño  | Rango                                                | Valor por defecto | Ejemplo            |
|---------|---------|------------------------------------------------------|-------------------|--------------------|
| `byte`  | 8 bits  | -128 a 127                                           | 0                 | `byte edad = 25;`  |
| `short` | 16 bits | -32,768 a 32,767                                     | 0                 | `short anio = 2025;` |
| `int`   | 32 bits | -2,147,483,648 a 2,147,483,647                       | 0                 | `int poblacion = 8_000_000;` |
| `long`  | 64 bits | -9,223,372,036,854,775,808 a 9,223,372,036,854,775,807 | 0L                | `long distancia = 150_000_000_000L;` |

```java
byte nivelBateria = 100;            // 8 bits  — ideal para valores pequeños
short temperatura = -15;            // 16 bits — útil para números medianos
int contador = 42_000;              // 32 bits — el tipo entero más usado
long poblacionMundial = 8_100_000_000L; // 64 bits — se requiere el sufijo L
```

Java permite usar guiones bajos (`_`) para mejorar la legibilidad de números grandes. No afectan el valor almacenado:

```java
int millon = 1_000_000;
long tarjetaCredito = 1234_5678_9012_3456L;
int binario = 0b1010_1100;          // 172 en decimal
int hexadecimal = 0xFF_EC;          // 65516 en decimal
```

#### Tipos de Punto Flotante

| Tipo    | Tamaño  | Rango aproximado                | Precisión     | Valor por defecto | Ejemplo                |
|---------|---------|----------------------------------|---------------|-------------------|------------------------|
| `float` | 32 bits | ±1.4E-45 a ±3.4E+38             | 6-7 dígitos   | 0.0f              | `float precio = 19.99f;` |
| `double`| 64 bits | ±4.9E-324 a ±1.8E+308           | 15 dígitos    | 0.0d              | `double pi = 3.14159265359;` |

```java
float descuento = 12.5f;            // el sufijo f/F es obligatorio
double saldoBancario = 1547.89;     // double es el tipo por defecto para decimales
double notacionCientifica = 2.5e6;  // equivale a 2,500,000.0
double avogadro = 6.02214076e23;    // número de Avogadro
```

#### El Tipo `char`

| Tipo   | Tamaño  | Rango                  | Valor por defecto | Ejemplo             |
|--------|---------|------------------------|-------------------|---------------------|
| `char` | 16 bits | 0 a 65,535 (Unicode)   | '\u0000'          | `char letra = 'A';` |

```java
char inicial = 'J';
char digito = '7';
char simbolo = '@';
char unicode = '\u00F1';            // representa la letra 'ñ'
char corazon = '\u2665';            // ♥
char tabulador = '\t';              // carácter de escape: tabulación
char nuevaLinea = '\n';             // carácter de escape: nueva línea
```

Secuencias de escape más comunes:

| Secuencia | Significado        |
|-----------|--------------------|
| `\n`      | Nueva línea        |
| `\t`      | Tabulación         |
| `\\`      | Barra invertida    |
| `\'`      | Comilla simple     |
| `\"`      | Comilla doble      |
| `\r`      | Retorno de carro   |
| `\b`      | Retroceso          |
| `\f`      | Avance de página   |

#### El Tipo `boolean`

| Tipo      | Tamaño (depende de la JVM) | Valores posibles | Valor por defecto |
|-----------|----------------------------|------------------|-------------------|
| `boolean` | No definido (JVM-dependente) | `true` o `false` | `false`           |

```java
boolean activo = true;
boolean completado = false;
boolean esMayorDeEdad = (edad >= 18);  // resultado de una expresión
```

> **A diferencia de C/C++**, en Java `boolean` no es compatible con valores numéricos. No puedes escribir `if (1)` ni asignar `true` a un entero. Esto elimina una clase entera de bugs.

### 2.1.2 Tabla Resumen de Tipos Primitivos

| Tipo      | Contenido    | Tamaño  | Valor por defecto | Rango / Valores                          |
|-----------|-------------|---------|-------------------|------------------------------------------|
| `byte`    | Entero      | 1 byte  | 0                 | -128 a 127                               |
| `short`   | Entero      | 2 bytes | 0                 | -32,768 a 32,767                         |
| `int`     | Entero      | 4 bytes | 0                 | ±2.1 mil millones                        |
| `long`    | Entero      | 8 bytes | 0L                | ±9.2 trillones                           |
| `float`   | Decimal     | 4 bytes | 0.0f              | Precisión simple IEEE 754                |
| `double`  | Decimal     | 8 bytes | 0.0d              | Precisión doble IEEE 754                 |
| `char`    | Carácter    | 2 bytes | '\u0000'           | 0 a 65,535 (Unicode)                     |
| `boolean` | Lógico      | variable | false             | true / false                             |

### 2.1.3 Punto Flotante IEEE 754 en Profundidad

Esta sección es **crítica**. Entender cómo Java representa números decimales internamente te evitará bugs financieros, errores de comparación y comportamientos inexplicables.

#### Por qué 0.1 + 0.2 != 0.3 en Java

Ejecuta este código:

```java
public class PuntoFlotanteDemo {
    public static void main(String[] args) {
        double a = 0.1;
        double b = 0.2;
        double c = 0.3;

        System.out.println("a + b = " + (a + b));
        System.out.println("c     = " + c);
        System.out.println("a + b == c ? " + ((a + b) == c));
        System.out.println("Diferencia: " + ((a + b) - c));
    }
}
```

La salida es desconcertante:

```
a + b = 0.30000000000000004
c     = 0.3
a + b == c ? false
Diferencia: 5.551115123125783E-17
```

**¿Por qué ocurre esto?** La respuesta está en el estándar IEEE 754.

El número 0.1 en decimal es equivalente a 1/10. En binario, 1/10 es una fracción periódica infinita, algo como `0.0001100110011001100110011...` (periódico). Como `double` tiene solo 64 bits (53 bits de mantisa efectiva), esta fracción debe ser **truncada**, introduciendo un error de redondeo. Lo mismo ocurre con 0.2. Al sumar dos números aproximados, el error se acumula.

Es exactamente el mismo fenómeno por el que 1/3 = 0.333333... en decimal no puede representarse exactamente con un número finito de dígitos.

#### Cómo se almacenan float y double internamente

El estándar IEEE 754 define la representación con tres componentes:

```
┌───────┬──────────────┬─────────────────────────────┐
│ Signo │  Exponente   │          Mantisa             │
│ 1 bit │   8 bits     │         23 bits             │  ← float (32 bits)
│ 1 bit │   11 bits    │         52 bits             │  ← double (64 bits)
└───────┴──────────────┴─────────────────────────────┘
```

- **Bit de signo (S):** 0 = positivo, 1 = negativo.
- **Exponente (E):** Almacenado con un "bias" (sesgo). Para float, bias = 127; para double, bias = 1023. El exponente real es E - bias.
- **Mantisa (M):** Almacena los dígitos significativos del número, con un "bit implícito" (1.) que no se guarda.

La fórmula del valor es: `(-1)^S × (1.M) × 2^(E - bias)`

**Ejemplo: cómo se almacena 0.15625 en float**

```
0.15625 en binario: 0.00101
Normalizado: 1.01 × 2^(-3)
Signo: 0
Exponente real: -3 → E = -3 + 127 = 124 = 01111100
Mantisa: 01 seguido de ceros hasta 23 bits → 01000000000000000000000

Float completo: 0 01111100 01000000000000000000000
Hex: 0x3E200000
```

```java
// Verifiquemos
int bits = Float.floatToIntBits(0.15625f);
System.out.println("0x" + Integer.toHexString(bits)); // 0x3e200000
```

#### Casos especiales: NaN, Infinity, -0.0

El estándar IEEE 754 reserva ciertos patrones de bits para valores especiales:

```java
// Infinity
double posInf = 1.0 / 0.0;
double negInf = -1.0 / 0.0;
System.out.println(posInf);                    // Infinity
System.out.println(negInf);                    // -Infinity
System.out.println(Double.isInfinite(posInf)); // true

// NaN (Not a Number)
double nan = 0.0 / 0.0;
double nan2 = Math.sqrt(-1);
System.out.println(nan);                       // NaN
System.out.println(Double.isNaN(nan));         // true

// Propiedad impactante de NaN: NaN != NaN siempre
System.out.println(nan == nan);                // false
System.out.println(Double.compare(nan, nan));  // 0 (compare sí considera NaN igual a NaN)

// -0.0 vs +0.0
double negZero = -0.0;
double posZero = 0.0;
System.out.println(negZero == posZero);        // true (== dice que son iguales)
System.out.println(1.0 / negZero);             // -Infinity
System.out.println(1.0 / posZero);             // Infinity

// Para distinguir +0.0 de -0.0:
System.out.println(Double.compare(negZero, posZero)); // -1 (son diferentes)
```

Estos valores especiales se propagan en operaciones:

```java
double resultado = 5.0 + Double.NaN;     // NaN
double resultado2 = Double.POSITIVE_INFINITY * 0; // NaN
double resultado3 = Double.POSITIVE_INFINITY + Double.POSITIVE_INFINITY; // Infinity
double resultado4 = Double.POSITIVE_INFINITY - Double.POSITIVE_INFINITY; // NaN
```

#### Errores catastróficos de cancelación

Cuando restas dos números casi iguales, los dígitos significativos se cancelan, dejando solo ruido de redondeo:

```java
public class CancelacionCatastrofica {
    public static void main(String[] args) {
        double a = 1.000000000000001;
        double b = 1.000000000000000;

        double resultado = a - b;
        // Esperado: 0.000000000000001 (1e-15)
        // La cancelación amplifica el error relativo enormemente

        System.out.println("a - b = " + resultado);

        // Ejemplo más dramático: fórmula cuadrática estándar
        // Para ax² + bx + c = 0, x = (-b ± √(b² - 4ac)) / (2a)
        // Cuando b² >> 4ac, hay cancelación catastrófica
        double discriminante = 1e10 * 1e10 - 4 * 1 * 1;  // 1e20 - 4 ≈ 1e20
        double raiz = (-1e10 + Math.sqrt(discriminante)) / 2;
        System.out.println("Raíz con fórmula estándar: " + raiz);
        // Resultado inexacto por cancelación catastrófica en el numerador
    }
}
```

La solución es usar fórmulas numéricamente estables o `BigDecimal`.

#### BigDecimal para dinero y cálculos financieros

**Regla de oro: nunca uses float ni double para dinero.** Usa `BigDecimal`.

```java
import java.math.BigDecimal;
import java.math.RoundingMode;

public class Facturacion {
    public static void main(String[] args) {
        // MAL: double para dinero
        double precioUnitario = 19.99;
        double cantidad = 3;
        double subtotalDouble = precioUnitario * cantidad;
        double impuestoDouble = subtotalDouble * 0.21;  // 21% IVA
        double totalDouble = subtotalDouble + impuestoDouble;
        System.out.println("Total con double: " + totalDouble);
        // Posible: 72.56369999999999 — ¡catastrófico para una factura!

        // BIEN: BigDecimal
        BigDecimal precio = new BigDecimal("19.99");
        BigDecimal cant = new BigDecimal("3");
        BigDecimal tasaIVA = new BigDecimal("0.21");

        BigDecimal subtotal = precio.multiply(cant);
        BigDecimal impuesto = subtotal.multiply(tasaIVA)
            .setScale(2, RoundingMode.HALF_UP);
        BigDecimal total = subtotal.add(impuesto);

        System.out.println("Subtotal: " + subtotal);     // 59.97
        System.out.println("IVA 21%:  " + impuesto);    // 12.59
        System.out.println("TOTAL:    " + total);       // 72.56
    }
}
```

> **Regla crítica:** Crea `BigDecimal` siempre con un `String`, nunca con un `double`. Si escribes `new BigDecimal(0.1)`, el `BigDecimal` capturará el valor inexacto de `double`, propagando el error. Profundizaremos en `BigDecimal` en la sección 2.10.

### 2.1.4 Overflow y Underflow

El overflow (desbordamiento) ocurre cuando una operación produce un resultado que excede el rango del tipo de dato. En Java, el overflow de enteros es **silencioso**: no lanza excepción, simplemente envuelve el valor.

#### Overflow en enteros

```java
int maximo = Integer.MAX_VALUE;   // 2,147,483,647
int desbordado = maximo + 1;      // -2,147,483,648  — ¡overflow silencioso!
System.out.println(desbordado);   // -2147483648

int resultado = 100_000 * 100_000; // esperado: 10,000,000,000
System.out.println(resultado);     // 1,410,065,408 — completamente incorrecto
```

Visualización de lo que ocurre internamente:

```
Integer.MAX_VALUE en binario:  01111111 11111111 11111111 11111111
Integer.MAX_VALUE + 1:          10000000 00000000 00000000 00000000
Ese número en complemento a dos: -2147483648 (Integer.MIN_VALUE)

Integer.MAX_VALUE + 2:          10000000 00000000 00000000 00000001 = -2147483647
```

En esencia, los enteros en Java **dan la vuelta** como un odómetro. Al pasar el máximo, vuelven al mínimo, y viceversa.

#### Casos reales de desastres por overflow

**Ariane 5 (1996):** El cohete Ariane 5 de la Agencia Espacial Europea explotó 37 segundos después del despegue, con una pérdida de 370 millones de dólares. La causa: un valor `double` de 64 bits (aceleración horizontal) se intentó convertir a un entero de 16 bits con signo, causando overflow. El software de navegación, heredado del Ariane 4 (que nunca alcanzaba esas velocidades), lanzó una excepción que se interpretó como datos de vuelo, inclinando el cohete catastróficamente.

**Y2K (año 2000):** Aunque no estrictamente overflow, ilustra el peligro de asumir rangos. Usar 2 dígitos para el año (99 → 00) causó fallos en facturación, intereses bancarios (cálculo de días negativos) y sistemas de control en todo el mundo.

**Gangnam Style en YouTube (2014):** El contador de reproducciones, almacenado en un entero de 32 bits, superó 2,147,483,647. YouTube migró a 64 bits en respuesta.

**Boeing 787 (2015):** Un bug de overflow en un contador interno requería reiniciar los generadores eléctricos cada 248 días (2^31 centésimas de segundo) para evitar que la aeronave perdiera toda la energía eléctrica en vuelo.

#### Cómo detectar overflow: Math.addExact() y familia

Desde Java 8, la clase `Math` ofrece métodos que lanzan `ArithmeticException` en caso de overflow:

```java
// Con detección de overflow
try {
    int seguro = Math.addExact(Integer.MAX_VALUE, 1); // ArithmeticException
} catch (ArithmeticException e) {
    System.out.println("¡Overflow detectado!");
}

try {
    int seguro = Math.multiplyExact(100_000, 100_000); // ArithmeticException
} catch (ArithmeticException e) {
    System.out.println("¡Overflow en multiplicación!");
}

// Métodos disponibles:
// Math.addExact(int,int)       Math.addExact(long,long)
// Math.subtractExact(int,int)  Math.subtractExact(long,long)
// Math.multiplyExact(int,int)  Math.multiplyExact(long,long)
// Math.incrementExact(int)     Math.incrementExact(long)
// Math.decrementExact(int)     Math.decrementExact(long)
// Math.negateExact(int)        Math.negateExact(long)
// Math.toIntExact(long)        Math.multiplyExact(long,int) (Java 9+)
```

> **Recomendación:** Si trabajas con valores que provienen de entrada de usuario, archivos o sistemas externos, usa siempre los métodos `*Exact()` para detectar overflow tempranamente. Un overflow silencioso es mucho más difícil de depurar que una excepción.

#### Underflow en punto flotante

El underflow ocurre cuando un número es tan pequeño que se redondea a 0:

```java
double tiny = Double.MIN_VALUE;   // 4.9E-324 (el menor número positivo representable)
System.out.println(tiny / 2);     // 0.0 — ¡underflow!

// Underflow gradual: números subnormales
// Cuando el exponente alcanza su mínimo, la mantisa empieza a perder bits
// y el número pierde precisión gradualmente hasta convertirse en 0.
```

### 2.1.5 Auto-boxing y Unboxing

El **auto-boxing** es la conversión automática que hace el compilador entre un tipo primitivo y su clase wrapper correspondiente. El **unboxing** es la conversión inversa.

```java
// Auto-boxing: int → Integer
Integer wrapper = 42;          // el compilador genera: Integer.valueOf(42)

// Unboxing: Integer → int
int primitivo = wrapper;       // el compilador genera: wrapper.intValue()

// En colecciones (solo aceptan objetos)
List<Integer> numeros = new ArrayList<>();
numeros.add(100);              // auto-boxing: int → Integer
int valor = numeros.get(0);    // unboxing: Integer → int

// En expresiones
Integer a = 10;
Integer b = 20;
int suma = a + b;              // ambos se unboxean, se suman, resultado primitivo
```

#### El costo oculto del auto-boxing

Cada operación de boxing crea un objeto en el heap (o reutiliza uno del cache). En bucles intensivos, el impacto en rendimiento y memoria puede ser **devastador**:

```java
// TERRIBLE: boxing y unboxing en cada iteración
Long inicio = System.currentTimeMillis();
Long sumaMal = 0L;                  // Long wrapper, no long primitivo
for (long i = 0; i < 10_000_000; i++) {
    sumaMal += i;                   // cada iteración: unbox sumaMal, sumar, boxing al asignar
}
Long fin = System.currentTimeMillis();
System.out.println("Tiempo con Long: " + (fin - inicio) + " ms");
// ~600-800 ms — crea 10 millones de objetos Long en el heap

// EXCELENTE: tipo primitivo
inicio = System.currentTimeMillis();
long sumaBien = 0L;
for (long i = 0; i < 10_000_000; i++) {
    sumaBien += i;
}
fin = System.currentTimeMillis();
System.out.println("Tiempo con long: " + (fin - inicio) + " ms");
// ~5-10 ms — sin objetos, solo aritmética en registros
```

La diferencia de rendimiento puede ser de **60x a 100x**. El Garbage Collector también sufre al tener que recolectar millones de objetos temporales.

#### Integer cache (-128 a 127) y la trampa del ==

Java mantiene un cache de objetos `Integer` para los valores entre -128 y 127. Esto produce comportamientos sorprendentes al comparar con `==`:

```java
Integer a = 100;    // Integer.valueOf(100) → cacheado, mismo objeto
Integer b = 100;    // Integer.valueOf(100) → mismo objeto del cache
System.out.println(a == b);    // true — ¡mismo objeto!

Integer c = 200;    // Integer.valueOf(200) → fuera del cache, nuevo objeto
Integer d = 200;    // Integer.valueOf(200) → otro nuevo objeto
System.out.println(c == d);    // false — ¡objetos diferentes!

// Con byte, short, long: cache de -128 a 127 también
// Con Character: cache de 0 a 127
// Con Boolean: TRUE y FALSE siempre son constantes
Boolean b1 = true;
Boolean b2 = Boolean.valueOf("true");
System.out.println(b1 == b2);  // true — mismo objeto
```

El cache se configura con la propiedad de JVM `-XX:AutoBoxCacheMax=<size>`.

**Regla de oro:** Nunca compares wrappers con `==`. Usa `.equals()` o mejor aún, haz el unboxing explícito y compara primitivos con `==`:

```java
// Peligroso
if (a == b) { ... }               // Compara referencias, no valores

// Correcto
if (a.equals(b)) { ... }          // Compara valores
if (a.intValue() == b.intValue()) // Unboxing explícito
if ((int)a == (int)b)             // Unboxing por cast
```

#### NullPointerException en unboxing

El unboxing de una referencia `null` lanza `NullPointerException`:

```java
Integer valor = null;
int x = valor;                    // NullPointerException en tiempo de ejecución

// Extremadamente común y peligroso en expresiones
Integer a = obtenerValor();       // puede devolver null
int b = a + 10;                   // si a es null → NullPointerException

// También con ternarios
Integer resultado = true ? 42 : null;
int r = resultado;                // NullPointerException — el ternario desencadena unboxing

// La solución: verificar null o usar un valor por defecto
Integer seguro = obtenerValor();
int x = (seguro != null) ? seguro : 0;
// Java 9+: Objects.requireNonNullElse()
int y = java.util.Objects.requireNonNullElse(obtenerValor(), 0);
```

### 2.1.6 Tipos de Referencia (Reference Types)

Los **tipos de referencia** no almacenan el valor directamente, sino una **referencia** (dirección de memoria) que apunta a un objeto en el **heap** (montículo). Todos los tipos que no son primitivos son de referencia: clases, interfaces, arreglos y enumeraciones.

```java
// Tipos primitivos: el valor se guarda en la variable misma
int x = 10;
int y = x;       // y recibe una copia del valor. x e y son independientes.
y = 20;          // x sigue valiendo 10

// Tipos de referencia: la variable guarda una dirección de memoria
String texto = new String("Hola");
String otroTexto = texto;  // otroTexto apunta al MISMO objeto que texto
// Si el objeto fuera mutable, modificar uno afectaría al otro
```

**Diferencias clave entre primitivos y referencias:**

| Característica      | Tipos Primitivos                   | Tipos de Referencia                    |
|---------------------|------------------------------------|----------------------------------------|
| Almacenamiento      | Valor directo en la variable       | Referencia a un objeto en el heap      |
| Memoria             | Stack                              | Heap (el objeto) + Stack (la referencia) |
| Valor por defecto   | Específico del tipo (0, false...)  | `null`                                 |
| Métodos             | No tienen métodos                  | Pueden invocar métodos                 |
| `==`                | Compara valores                    | Compara referencias (direcciones)      |

#### El Valor `null`

`null` es un literal especial que indica que una variable de referencia **no apunta a ningún objeto**. Intentar invocar un método sobre `null` produce la temida excepción `NullPointerException`.

```java
String nombre = null;            // sin problema
int longitud = nombre.length();  // NullPointerException en tiempo de ejecución
```

Siempre verifica si una referencia es `null` antes de usarla:

```java
if (nombre != null) {
    System.out.println(nombre.length());
}
```

---

## 2.2 Variables

### 2.2.1 Declaración e Inicialización de Variables

En Java, toda variable debe declararse con un tipo antes de ser usada.

```java
// Declaración (reserva memoria pero no asigna valor útil)
int edad;

// Inicialización (asigna un valor inicial)
edad = 30;

// Declaración e inicialización en una sola línea
String ciudad = "Bogotá";
double salario = 2500.50;
boolean esEstudiante = true;
```

Puedes declarar varias variables del mismo tipo en una línea:

```java
int a, b, c;                      // tres variables declaradas
int x = 1, y = 2, z = 3;         // declaración con inicialización múltiple
```

### 2.2.2 Convenciones de Nombres

Java tiene convenciones de nomenclatura ampliamente aceptadas:

| Elemento          | Convención         | Ejemplo                         |
|--------------------|--------------------|---------------------------------|
| Variables          | camelCase          | `nombreCompleto`, `totalVentas` |
| Métodos            | camelCase          | `calcularPromedio()`, `getNombre()` |
| Clases             | PascalCase         | `MiClase`, `EstudianteController` |
| Constantes         | UPPER_SNAKE_CASE   | `MAX_VELOCIDAD`, `PI`           |
| Paquetes           | minúsculas         | `com.empresa.proyecto`          |

Los nombres de variables deben ser descriptivos:

```java
// Bien
int cantidadProductos;
String direccionEnvio;

// Mal
int cp;         // ¿qué significa cp?
String str;     // demasiado genérico
```

### 2.2.3 Constantes con `final`

La palabra clave `final` convierte una variable en una **constante**: su valor no puede cambiar después de la inicialización.

```java
final double PI = 3.14159265359;
final int DIAS_SEMANA = 7;
final String NOMBRE_EMPRESA = "TechCorp";

// PI = 3.14;  // Error de compilación: no se puede reasignar una variable final
```

Las constantes de instancia (dentro de una clase pero fuera de métodos) pueden inicializarse en el constructor si se declaran como `final` en blanco:

```java
public class Circulo {
    private final double radio;          // final en blanco

    public Circulo(double radio) {
        this.radio = radio;              // se asigna en el constructor
    }
}
```

### 2.2.4 Inferencia de Tipos con `var` (Java 10+)

A partir de Java 10, puedes usar `var` para declarar variables locales con inferencia de tipos. El compilador deduce el tipo a partir del valor asignado.

```java
var numero = 42;                // el compilador infiere int
var precio = 19.99;             // el compilador infiere double
var nombre = "María";           // el compilador infiere String
var lista = new ArrayList<String>();  // infiere ArrayList<String>
var activo = true;              // el compilador infiere boolean

// Equivalente sin var:
int numero = 42;
double precio = 19.99;
String nombre = "María";
ArrayList<String> lista = new ArrayList<>();
boolean activo = true;
```

**Reglas y limitaciones de `var`:**

1. Solo se usa en **variables locales** (dentro de métodos). No para campos de clase, parámetros o tipos de retorno.
2. Debe **inicializarse** en la misma línea de declaración.
3. No puede inicializarse con `null` (el compilador no puede inferir el tipo).
4. El tipo se fija en la declaración y no puede cambiar después.

```java
// Correcto
var mensaje = "Hola";

// Incorrecto
// var x;                // Error: debe inicializarse
// var y = null;         // Error: no se puede inferir tipo desde null
// class MiClase {
//     var campo = 10;   // Error: var no se permite en campos de clase
// }
```

Cuándo usar `var`: cuando el tipo es obvio por el contexto. Cuándo evitarlo: cuando el tipo no es evidente, ya que reduce la legibilidad.

### 2.2.5 Stack vs Heap: Dónde Vive Cada Variable

Entender dónde residen las variables en memoria es fundamental para comprender el rendimiento, la concurrencia y el comportamiento del Garbage Collector.

**Reglas de ubicación:**

| Tipo de variable               | Ubicación de la variable | Ubicación del valor          |
|--------------------------------|--------------------------|------------------------------|
| Variable local primitiva       | Stack                    | Stack (el valor mismo)       |
| Variable local de referencia   | Stack                    | Stack (la ref.) + Heap (objeto) |
| Parámetro de método primitivo  | Stack                    | Stack                        |
| Parámetro de método referencia | Stack                    | Stack (ref.) + Heap (objeto) |
| Campo de instancia primitivo   | Heap (dentro del objeto) | Heap                         |
| Campo de instancia referencia  | Heap (dentro del objeto) | Heap (ref.) + Heap (objeto)  |
| Campo estático                 | Metaspace                | Metaspace (primitivo) o Heap (objeto) |

```java
public class MemoriaDemo {
    private int campoInstancia = 10;        // vive en el heap, dentro del objeto
    private String nombre = "Java";         // referencia en el heap, String en String Pool (heap)
    private static long contador = 0L;      // vive en Metaspace

    public void metodo(int parametro) {     // parámetro vive en el stack
        int local = parametro * 2;          // variable local vive en el stack
        Object obj = new Object();          // referencia 'obj' en stack, el Object en heap
        String texto = "Hola";             // referencia en stack, "Hola" en String Pool
    }
}
```

#### Escape Analysis: Cómo la JVM Optimiza Poniendo Objetos en el Stack

El **Escape Analysis** es una optimización del compilador JIT (Just-In-Time) de HotSpot. Si el compilador determina que un objeto **nunca escapa** del método donde se crea (no se retorna, no se almacena en un campo), puede aplicar dos optimizaciones:

**1. Stack Allocation (Asignación en el Stack):** El objeto se crea directamente en el stack frame en lugar de en el heap, eliminando la presión sobre el Garbage Collector.

**2. Scalar Replacement (Reemplazo Escalar):** Los campos del objeto se descomponen en variables locales escalares. En vez de crear un objeto `Punto` con campos `x` e `y`, el JIT crea dos variables locales `int x` e `int y` directamente.

```java
// El objeto 'punto' nunca escapa de este método
// El JIT puede eliminar la asignación en heap completamente
public int calcular() {
    Punto punto = new Punto(3, 4);
    // El JIT lo reemplaza por:
    // int x = 3; int y = 4;
    return punto.x * punto.x + punto.y * punto.y;
}
```

**Requisitos para que Escape Analysis funcione:**
- El objeto no se retorna del método.
- El objeto no se almacena en un campo de clase.
- El objeto no se pasa a métodos que puedan retener su referencia.

Para verificar si el Escape Analysis está activo, usa las flags de JVM `-XX:+DoEscapeAnalysis` (activado por defecto desde Java 7).

### 2.2.6 Variables de Instancia vs Locales: Visibilidad entre Hilos

La ubicación de una variable también determina cómo se comporta en entornos concurrentes (múltiples hilos).

| Tipo de variable      | Visibilidad entre hilos                      | ¿Es thread-safe por defecto? |
|-----------------------|----------------------------------------------|------------------------------|
| Variable local        | Cada hilo tiene su propia copia en su stack  | Sí                           |
| Parámetro de método   | Cada hilo tiene su propia copia              | Sí (si el objeto referenciado es inmutable o no compartido) |
| Campo de instancia    | Compartido entre hilos que usan el mismo objeto | No                        |
| Campo estático        | Compartido entre TODOS los hilos             | No                           |

```java
public class VisibilidadDemo {
    private int contadorCompartido = 0;            // NO thread-safe
    private static int contadorGlobal = 0;         // NO thread-safe

    public void metodoConVariableLocal() {
        int local = 0;                             // thread-safe: cada hilo tiene su copia
        local++;
        // local nunca será afectado por otros hilos
    }

    public void metodoPeligroso() {
        contadorCompartido++;                      // ¡condición de carrera! NO es atómico
        // contadorCompartido++ se traduce en tres instrucciones:
        // 1. leer valor actual
        // 2. incrementarlo
        // 3. escribirlo de vuelta
        // Entre los pasos 1 y 3, otro hilo puede haber modificado el valor
    }
}
```

---

## 2.3 Operadores

### 2.3.1 Operadores Aritméticos

| Operador | Descripción         | Ejemplo       |
|----------|---------------------|---------------|
| `+`      | Suma                | `a + b`       |
| `-`      | Resta               | `a - b`       |
| `*`      | Multiplicación      | `a * b`       |
| `/`      | División            | `a / b`       |
| `%`      | Módulo (resto)      | `a % b`       |

```java
int suma = 10 + 3;         // 13
int resta = 10 - 3;        // 7
int producto = 10 * 3;     // 30
int cociente = 10 / 3;     // 3  (¡cuidado! división entera)
int resto = 10 % 3;        // 1

// División entre double conserva decimales
double divisionReal = 10.0 / 3.0;  // 3.3333...
double divisionMixta = 10 / 3.0;   // 3.3333...  (promoción automática a double)
```

**Error común: truncamiento en división entera:**

```java
int resultado = 5 / 2;            // resultado = 2,  NO 2.5
double resultadoMalo = 5 / 2;     // 2.0 (la división entera ya ocurrió antes de asignar)
double resultadoBueno = 5.0 / 2;  // 2.5
```

### 2.3.2 Operadores Relacionales

Devuelven un valor `boolean` (`true` o `false`).

| Operador | Descripción           | Ejemplo    |
|----------|-----------------------|------------|
| `==`     | Igual a               | `a == b`   |
| `!=`     | Distinto de           | `a != b`   |
| `>`      | Mayor que             | `a > b`    |
| `<`      | Menor que             | `a < b`    |
| `>=`     | Mayor o igual que     | `a >= b`   |
| `<=`     | Menor o igual que     | `a <= b`   |

```java
int a = 10, b = 20;

boolean igual       = (a == b);   // false
boolean distinto    = (a != b);   // true
boolean mayor       = (a > b);    // false
boolean menor       = (a < b);    // true
boolean mayorOIgual = (a >= 10);  // true
```

> **Atención:** Usa `==` para comparar tipos primitivos. Para comparar objetos (como `String`), usa `.equals()`. Ver sección 2.7.

### 2.3.3 Operadores Lógicos

Se usan para combinar expresiones booleanas.

| Operador | Descripción        | Ejemplo                    |
|----------|--------------------|----------------------------|
| `&&`     | AND lógico (corto-circuito) | `a && b`            |
| `||`     | OR lógico (corto-circuito)  | `a || b`            |
| `!`      | NOT lógico         | `!a`                       |

El comportamiento de **corto-circuito** significa que la evaluación se detiene tan pronto como el resultado es definitivo:

```java
// Corto-circuito con &&
// Si la primera condición es false, la segunda NUNCA se evalúa
if (x != 0 && (y / x) > 10) {
    // Si x es 0, no se evalúa (y / x), evitando división por cero
}

// Corto-circuito con ||
// Si la primera condición es true, la segunda NUNCA se evalúa
if (nombre == null || nombre.isEmpty()) {
    // Si nombre es null, no se llama a isEmpty(), evitando NullPointerException
}
```

Existen también versiones **sin corto-circuito** (`&` y `|`) que evalúan ambas condiciones siempre, pero son menos comunes:

```java
boolean andSinCorto = (x != 0) & (y / x > 10);   // evalúa ambas, riesgo de división por cero
boolean orSinCorto  = validarA() | validarB();    // llama ambos métodos siempre
```

### 2.3.4 Operadores de Asignación

| Operador | Equivalencia       |
|----------|--------------------|
| `=`      | `a = b`            |
| `+=`     | `a = a + b`        |
| `-=`     | `a = a - b`        |
| `*=`     | `a = a * b`        |
| `/=`     | `a = a / b`        |
| `%=`     | `a = a % b`        |
| `&=`     | `a = a & b`        |
| `|=`     | `a = a | b`        |
| `^=`     | `a = a ^ b`        |
| `<<=`    | `a = a << b`       |
| `>>=`    | `a = a >> b`       |
| `>>>=`   | `a = a >>> b`      |

```java
int total = 100;
total += 50;      // total = 150
total -= 30;      // total = 120
total *= 2;       // total = 240
total /= 6;       // total = 40
total %= 7;       // total = 5  (40 % 7 = 5)
```

### 2.3.5 Operadores de Incremento y Decremento

| Operador | Descripción               | Ejemplo |
|----------|---------------------------|---------|
| `++`     | Incrementa en 1           | `a++` o `++a` |
| `--`     | Decrementa en 1           | `a--` o `--a` |

La diferencia entre **prefijo** y **sufijo** es el momento de la evaluación:

```java
int a = 5;
int b = ++a;   // Pre-incremento: primero incrementa a (6), luego asigna a b (6)
// Ahora: a = 6, b = 6

int c = 5;
int d = c++;   // Post-incremento: primero asigna c a d (5), luego incrementa c (6)
// Ahora: c = 6, d = 5
```

**Buena práctica:** Evita usar incremento/decremento dentro de expresiones complejas. Prefiere líneas separadas para mantener la claridad.

```java
// Confuso
int resultado = ++x * y-- + z;

// Claro
x++;
resultado = x * y + z;
y--;
```

### 2.3.6 Operador Ternario (`? :`)

Es el único operador que toma tres operandos. Es una forma compacta de escribir un `if-else`:

```
condicion ? valorSiVerdadero : valorSiFalso
```

```java
int edad = 20;
String categoria = (edad >= 18) ? "Adulto" : "Menor de edad";

int numero = -5;
int absoluto = (numero >= 0) ? numero : -numero;   // 5

// También se puede usar en expresiones más complejas
String estado = (puntaje >= 90) ? "Excelente"
              : (puntaje >= 75) ? "Aprobado"
              : "Reprobado";
```

**Cuidado con el ternario y auto-boxing/unboxing:**

```java
// Si uno de los operandos es primitivo y el otro es wrapper, ocurre unboxing
// Si el wrapper es null → NullPointerException
Integer valor = null;
int resultado = (condicion) ? valor : 0;  // NullPointerException si condicion es true
// El compilador unboxea 'valor' a int porque el otro operando (0) es int
```

### 2.3.7 Operadores Bitwise (a Nivel de Bits)

Operan sobre la representación binaria de los valores enteros. Se usan principalmente en programación de bajo nivel, gráficos, criptografía y flags de configuración.

| Operador | Descripción           |
|----------|-----------------------|
| `&`      | AND bit a bit         |
| `|`      | OR bit a bit          |
| `^`      | XOR bit a bit         |
| `~`      | NOT bit a bit (complemento) |
| `<<`     | Desplazamiento a la izquierda |
| `>>`     | Desplazamiento a la derecha (con signo) |
| `>>>`    | Desplazamiento a la derecha (sin signo) |

```java
int a = 5;   // 0101 en binario
int b = 3;   // 0011 en binario

int andBit = a & b;   // 0001 = 1
int orBit  = a | b;   // 0111 = 7
int xorBit = a ^ b;   // 0110 = 6
int notBit = ~a;      // complemento a dos de 5 en 32 bits: 1111...1010 = -6

// Desplazamientos
int izquierda  = a << 2;   // 0101 -> 010100 = 20  (multiplica por 2^2 = 4)
int derecha    = a >> 1;   // 0101 -> 0010 = 2     (divide por 2, con signo)
int derechaSin = -8 >>> 2; // desplazamiento sin signo (rellena con ceros a la izq.)

// Aplicación: flags de permisos
int LECTURA   = 1 << 0;   // 0001 = 1
int ESCRITURA = 1 << 1;   // 0010 = 2
int EJECUCION = 1 << 2;   // 0100 = 4
int ELIMINAR  = 1 << 3;   // 1000 = 8

int permisos = LECTURA | ESCRITURA;           // 0011 = 3
boolean puedeLeer    = (permisos & LECTURA) != 0;     // true
boolean puedeEjecutar = (permisos & EJECUCION) != 0;  // false
```

### 2.3.8 Tabla de Precedencia de Operadores

Los operadores con mayor precedencia se evalúan primero. Cuando hay empate, la asociatividad decide el orden (generalmente de izquierda a derecha).

| Precedencia | Operadores                                                   | Asociatividad  |
|-------------|--------------------------------------------------------------|----------------|
| 1 (más alta)| `()` (paréntesis), `[]` (acceso a arreglos), `.` (acceso a miembro) | Izq. a Der. |
| 2           | `++`, `--`, `+` (unario), `-` (unario), `!`, `~`, `(type)` (cast) | Der. a Izq. |
| 3           | `*`, `/`, `%`                                                | Izq. a Der.    |
| 4           | `+`, `-`                                                     | Izq. a Der.    |
| 5           | `<<`, `>>`, `>>>`                                            | Izq. a Der.    |
| 6           | `<`, `>`, `<=`, `>=`, `instanceof`                           | Izq. a Der.    |
| 7           | `==`, `!=`                                                   | Izq. a Der.    |
| 8           | `&`                                                          | Izq. a Der.    |
| 9           | `^`                                                          | Izq. a Der.    |
| 10          | `|`                                                          | Izq. a Der.    |
| 11          | `&&`                                                         | Izq. a Der.    |
| 12          | `||`                                                         | Izq. a Der.    |
| 13          | `? :`                                                        | Der. a Izq.    |
| 14 (más baja)| `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `^=`, `|=`, `<<=`, `>>=`, `>>>=` | Der. a Izq. |

#### Bugs reales causados por precedencia de operadores

**Bug 1: Mezclar desplazamiento con suma**

```java
int resultado = 1 << 2 + 1;   // ¿1 << (2 + 1) = 8, o (1 << 2) + 1 = 5?
System.out.println(resultado); // 8 — << tiene MENOR precedencia que +
// 2 + 1 se evalúa primero → 3, luego 1 << 3 = 8

// La intención probablemente era (1 << 2) + 1 = 5
int correcto = (1 << 2) + 1;
```

**Bug 2: Ternario con concatenación de strings**

```java
String resultado = "Valor: " + (condicion) ? "A" : "B";
// El + tiene mayor precedencia que ? :
// Se evalúa como: ("Valor: " + (condicion)) ? "A" : "B"
// "Valor: " + (condicion) produce un String, y String no es boolean → error de compilación

// CORRECTO:
String resultado = "Valor: " + ((condicion) ? "A" : "B");
```

> **Regla de oro:** Ante la duda, usa paréntesis. Hacen el código más legible y evitan errores. Ningún programador debería tener que memorizar la tabla de precedencia para entender tu código.

### 2.3.9 `instanceof` en Profundidad

El operador `instanceof` verifica si un objeto es una instancia de una clase, subclase o interfaz. Retorna `boolean`.

```java
Object obj = "Hola Mundo";

boolean esString = obj instanceof String;          // true
boolean esInteger = obj instanceof Integer;        // false
boolean esObject = obj instanceof Object;          // true (todo es Object)
boolean esComparable = obj instanceof Comparable;  // true (String implementa Comparable)
```

**instanceof con null:**

```java
String str = null;
boolean esNullString = str instanceof String;      // false — null nunca es instanceof nada
// No lanza NullPointerException, lo cual es útil para validación

// Patrón común: verificar null y tipo en una sola línea
if (obj instanceof String) {
    // obj no es null y además es String
    String s = (String) obj;
    // ...
}
```

**instanceof con herencia e interfaces:**

```java
class Animal {}
class Perro extends Animal {}
class Gato extends Animal {}

Object mascota = new Perro();

System.out.println(mascota instanceof Perro);     // true
System.out.println(mascota instanceof Animal);    // true (subclase de Animal)
System.out.println(mascota instanceof Gato);      // false
System.out.println(mascota instanceof Object);    // true
System.out.println(mascota instanceof Comparable); // false (Perro no implementa Comparable)
```

### 2.3.10 Pattern Matching para `instanceof` (Java 16+)

Desde Java 16, `instanceof` puede declarar una **variable de patrón** (pattern variable) que se crea y asigna si la verificación es exitosa, eliminando el cast posterior:

```java
// ANTES (Java 15-): verificación + cast separados
if (obj instanceof String) {
    String s = (String) obj;   // cast redundante y verboso
    System.out.println(s.length());
}

// AHORA (Java 16+): pattern matching
if (obj instanceof String s) {    // s se declara y asigna automáticamente
    System.out.println(s.length());
    System.out.println(s.toUpperCase());
}
// s está en scope SOLO dentro del bloque if
```

**Alcance (scope) de la variable de patrón:**

```java
// La variable de patrón está en scope donde el compilador puede garantizar
// que la verificación fue exitosa

if (!(obj instanceof String s)) {
    // s NO está disponible aquí (la verificación falló)
    return;
}
// s SÍ está disponible aquí (si llegamos, instanceof fue true)
System.out.println(s.length());

// En expresiones booleanas compuestas con &&
if (obj instanceof String s && s.length() > 5) {
    // s está disponible en el segundo operando porque && es corto-circuito
    // y s.length() solo se evalúa si instanceof fue true
    System.out.println(s);
}

// Con || NO funciona — no hay garantía de que instanceof haya sido true
// if (obj instanceof String s || s.length() > 5) {  // Error de compilación
//     ...
// }
```

### 2.3.11 Errores Comunes con Operadores

**1. Comparar objetos con `==`**

```java
String a = new String("hola");
String b = new String("hola");
System.out.println(a == b);       // false (compara referencias, no contenido)
System.out.println(a.equals(b));  // true  (compara contenido, que es lo correcto)
```

**2. División entera inesperada (ya vista en 2.3.1).**

**3. Overflow silencioso (ya visto en 2.1.4).**
## 2.4 Conversiones de Tipos (Casting)

### 2.4.1 Conversión Implícita (Widening)

Java convierte automáticamente un tipo más pequeño a uno más grande cuando no hay riesgo de pérdida de datos.

```
byte → short → int → long → float → double
            ↑
           char
```

```java
int entero = 100;
long largo = entero;        // int → long: automático, sin pérdida
float flotante = largo;     // long → float: automático, pero posible pérdida de precisión
double doble = flotante;    // float → double: automático

byte b = 50;
short s = b;                // byte → short: automático
int i = s;                  // short → int: automático

// En expresiones, los tipos pequeños se promocionan a int automáticamente
byte sumaBytes = b + b;     // Error: b + b se promociona a int, no cabe en byte
int sumaCorrecta = b + b;   // Correcto
```

### 2.4.2 Conversión Explícita (Narrowing / Casting)

Cuando conviertes un tipo grande a uno más pequeño, debes hacer un **cast** explícito. Asumes el riesgo de pérdida de datos.

```java
double grande = 9.78;
int pequeno = (int) grande;      // pequeno = 9 (la parte decimal se TRUNCA, no redondea)
System.out.println(pequeno);     // 9

long largo = 130;
byte pequenoByte = (byte) largo;  // 130 no cabe en byte (-128 a 127)
System.out.println(pequenoByte);  // -126 (overflow inesperado)
```

> El casting de punto flotante a entero **trunca** (no redondea). Para redondear, usa `Math.round()`.

```java
double valor = 7.8;
int truncado = (int) valor;        // 7
int redondeado = (int) Math.round(valor);  // 8
```

**Pérdida de datos al hacer narrowing:**

```java
int grande = 1_000_000;
short corto = (short) grande;  // -16960  — los bits de mayor orden se pierden
```

### 2.4.3 Pérdida de Precisión en Conversiones de Punto Flotante

Esta es una de las trampas más sutiles y peligrosas de Java. El hecho de que `float` tenga 32 bits y `int` también 32 bits no significa que todos los `int` quepan en un `float`.

#### int → float: la pérdida de precisión

Un `float` usa 23 bits de mantisa (más 1 bit implícito = 24 bits efectivos). Un `int` usa 32 bits para representar valores exactos. Cuando conviertes un `int` grande a `float`, los bits menos significativos pueden perderse:

```java
int grande = 16777217;            // 2^24 + 1
float f = grande;
int recuperado = (int) f;

System.out.println("Original:  " + grande);      // 16777217
System.out.println("Float:     " + f);           // 1.6777216E7 = 16777216
System.out.println("Recuperado: " + recuperado); // 16777216
System.out.println("¿Iguales?: " + (grande == recuperado)); // false — ¡perdimos el 1!
```

**Explicación:** 2^24 = 16777216. Con 24 bits de precisión en `float`, el siguiente número representable es 16777218. El número 16777217 simplemente no existe en el universo `float` — se redondea al más cercano, que es 16777216.

```java
// Demostración sistemática
for (int i = 16777210; i <= 16777220; i++) {
    float f = i;
    System.out.printf("int %d → float %.0f → (int) %d%n", i, f, (int) f);
}
// Observa cómo varios enteros colapsan en el mismo float
```

#### long → float y long → double

Un `long` tiene 64 bits de precisión. Convertirlo a `double` (53 bits de mantisa efectiva) también pierde precisión para valores grandes:

```java
long grande = 9_007_199_254_740_993L;     // 2^53 + 1
double d = grande;
long recuperado = (long) d;

System.out.println("Original:  " + grande);       // 9007199254740993
System.out.println("Double:    " + d);            // 9.007199254740992E15
System.out.println("Recuperado: " + recuperado);  // 9007199254740992
System.out.println("¿Iguales?: " + (grande == recuperado)); // false
```

#### double → float: pérdida de precisión y rango

```java
double valorDouble = 1.23456789012345678;
float valorFloat = (float) valorDouble;

System.out.println("double: " + valorDouble);  // 1.2345678901234567 (15 dígitos)
System.out.println("float:  " + valorFloat);   // 1.2345679 (7 dígitos — el resto se pierde)
```

La conversión explícita `double → float` trunca los bits extra. Si el valor double es mayor que `Float.MAX_VALUE`, el resultado es `Float.POSITIVE_INFINITY`. Si está entre `Float.MIN_VALUE` y `Float.MAX_VALUE`, se redondea al float más cercano (según el modo de redondeo IEEE 754, por defecto round-to-nearest-even).

### 2.4.4 Conversión con Strings

#### De tipo primitivo a String

```java
// Método 1: String.valueOf() — funciona con todos los tipos
String sEntero = String.valueOf(100);
String sDouble = String.valueOf(3.1416);
String sBoolean = String.valueOf(true);
String sChar = String.valueOf('A');

// Método 2: Concatenación con una cadena vacía
String usandoConcatenacion = 100 + "";

// Método 3: Métodos de las clases Wrapper
String usandoWrapper = Integer.toString(100);
String hex = Integer.toHexString(255);          // "ff"
String bin = Integer.toBinaryString(42);        // "101010"
String oct = Integer.toOctalString(64);         // "100"
```

#### De String a tipo primitivo

```java
// Cada tipo primitivo tiene su clase Wrapper con método parseXxx()
int entero = Integer.parseInt("42");
long largo = Long.parseLong("3000000000");
double doble = Double.parseDouble("3.1416");
float flotante = Float.parseFloat("2.5");
boolean booleano = Boolean.parseBoolean("true");
byte byteVal = Byte.parseByte("127");
short shortVal = Short.parseShort("30000");
```

**Convertir String a número en diferentes bases:**

```java
int binario = Integer.parseInt("101010", 2);     // 42 (base 2)
int octal = Integer.parseInt("77", 8);           // 63 (base 8)
int hex = Integer.parseInt("FF", 16);            // 255 (base 16)
// La base puede ser de 2 a 36
```

**Manejo seguro de NumberFormatException:**

```java
public static int parsearSeguro(String texto, int valorPorDefecto) {
    try {
        return Integer.parseInt(texto.trim());
    } catch (NumberFormatException e) {
        return valorPorDefecto;
    }
}
```

---

## 2.5 Estructuras de Control Condicionales

Las estructuras de control determinan el **flujo de ejecución** del programa tomando decisiones basadas en condiciones.

### 2.5.1 `if`, `if-else`, `if-else if-else`

```java
// if simple
int temperatura = 30;
if (temperatura > 25) {
    System.out.println("Hace calor");
}

// if-else
int nota = 65;
if (nota >= 70) {
    System.out.println("Aprobado");
} else {
    System.out.println("Reprobado");
}

// if-else if-else (escalera)
int calificacion = 85;
if (calificacion >= 90) {
    System.out.println("Sobresaliente (A)");
} else if (calificacion >= 80) {
    System.out.println("Notable (B)");
} else if (calificacion >= 70) {
    System.out.println("Bien (C)");
} else if (calificacion >= 60) {
    System.out.println("Suficiente (D)");
} else {
    System.out.println("Insuficiente (F)");
}
// Salida: Notable (B)
```

**Notas importantes:**

- Las llaves `{ }` son opcionales si el bloque tiene una sola sentencia, pero se recomienda usarlas siempre para evitar errores.

```java
// Peligroso sin llaves: solo la primera línea depende del if
if (condicion)
    hacerAlgo();
    hacerOtraCosa();  // esta línea se ejecuta SIEMPRE, independientemente de la condición

// Seguro con llaves
if (condicion) {
    hacerAlgo();
    hacerOtraCosa();
}
```

- El orden de las condiciones importa: se evalúan en secuencia y gana la primera que sea `true`.

### 2.5.2 `switch` Tradicional

Útil cuando se compara una misma variable contra múltiples valores constantes.

```java
int diaDeLaSemana = 3;
String nombreDia;

switch (diaDeLaSemana) {
    case 1:
        nombreDia = "Lunes";
        break;
    case 2:
        nombreDia = "Martes";
        break;
    case 3:
        nombreDia = "Miércoles";
        break;
    case 4:
        nombreDia = "Jueves";
        break;
    case 5:
        nombreDia = "Viernes";
        break;
    case 6:
    case 7:
        nombreDia = "Fin de semana";  // dos casos que comparten código (fall-through)
        break;
    default:
        nombreDia = "Día inválido";   // se ejecuta si ningún case coincide
        break;
}

System.out.println(nombreDia);  // Miércoles
```

**Tipos permitidos en `switch` tradicional:** `byte`, `short`, `int`, `char`, sus wrappers (`Byte`, `Short`, `Integer`, `Character`), `String` (desde Java 7) y `enum`.

> **El `break` es crucial.** Si lo omites, la ejecución "cae" al siguiente `case` (fall-through), lo que puede ser deseado (como en el caso 6 y 7 del ejemplo) o un error difícil de detectar.

### 2.5.3 `switch` Mejorado (Arrow Syntax, Java 14+)

A partir de Java 14, el `switch` puede usar la sintaxis de flecha (`->`) que elimina la necesidad de `break` y hace el código más conciso.

```java
int diaDeLaSemana = 3;
String nombreDia = switch (diaDeLaSemana) {
    case 1 -> "Lunes";
    case 2 -> "Martes";
    case 3 -> "Miércoles";
    case 4 -> "Jueves";
    case 5 -> "Viernes";
    case 6, 7 -> "Fin de semana";     // múltiples valores separados por coma
    default -> "Día inválido";
};

System.out.println(nombreDia);  // Miércoles
```

**Ventajas del switch con flecha:**
- No necesita `break` (no hay fall-through accidental).
- Cada `case` puede tener una sola línea o un bloque `{ ... }`.
- Múltiples valores se agrupan con coma.
- El `switch` puede devolver un valor directamente (switch como expresión).

### 2.5.4 `switch` como Expresión con `yield` (Java 14+)

Cuando necesitas un bloque de código más complejo dentro de un case, usas `yield` para devolver el valor:

```java
int mes = 8;
int dias = switch (mes) {
    case 1, 3, 5, 7, 8, 10, 12 -> 31;
    case 4, 6, 9, 11 -> 30;
    case 2 -> {
        int anio = 2025;
        boolean esBisiesto = (anio % 4 == 0 && anio % 100 != 0) || (anio % 400 == 0);
        yield esBisiesto ? 29 : 28;   // yield devuelve el valor de este case
    }
    default -> throw new IllegalArgumentException("Mes inválido: " + mes);
};

System.out.println("Días: " + dias);  // Días: 31
```

**Ejemplo avanzado: procesador de comandos con switch expression:**

```java
public class ProcesadorComandos {
    record Comando(String nombre, String[] args) {}

    public static String ejecutar(Comando cmd) {
        return switch (cmd.nombre().toLowerCase()) {
            case "crear" -> {
                if (cmd.args().length < 2) {
                    yield "Error: crear requiere nombre y tipo";
                }
                yield "Creado: " + cmd.args()[0] + " (" + cmd.args()[1] + ")";
            }
            case "listar" -> {
                var builder = new StringBuilder("Elementos:\n");
                for (String item : cmd.args()) {
                    builder.append("  - ").append(item).append("\n");
                }
                yield builder.toString();
            }
            case "ayuda" -> "Comandos: crear, listar, borrar, ayuda";
            case "borrar" -> "Borrado: " + (cmd.args().length > 0 ? cmd.args()[0] : "todo");
            default -> "Comando desconocido: " + cmd.nombre();
        };
    }
}
```

### 2.5.5 Pattern Matching en `switch` (Java 17+ Preview, Java 21 Estable)

El pattern matching permite verificar el **tipo de un objeto** directamente en el `switch`, eliminando la necesidad de `instanceof` + cast.

```java
// Java 21+
Object valor = 42;

String resultado = switch (valor) {
    case Integer i -> "Es un entero: " + i;
    case String s  -> "Es una cadena de longitud " + s.length();
    case Double d  -> "Es un double: " + d;
    case null      -> "Es null";
    default        -> "Tipo desconocido";
};

System.out.println(resultado);  // Es un entero: 42
```

**Ejemplo avanzado: procesador de mensajes con pattern matching switch:**

```java
// Java 21+
sealed interface Mensaje permits MensajeTexto, MensajeBinario, Ping {}
record MensajeTexto(String contenido) implements Mensaje {}
record MensajeBinario(byte[] datos) implements Mensaje {}
record Ping(long timestamp) implements Mensaje {}

String procesar(Mensaje msg) {
    return switch (msg) {
        case MensajeTexto(var texto) when texto.length() < 1000
            -> "Texto corto: " + texto;
        case MensajeTexto(var texto)
            -> "Texto largo (" + texto.length() + " chars)";
        case MensajeBinario(var datos) when datos.length == 0
            -> "Binario vacío";
        case MensajeBinario(var datos)
            -> "Binario: " + datos.length + " bytes";
        case Ping(long ts)
            -> "Ping con latencia: " + (System.currentTimeMillis() - ts) + "ms";
    };
}
```

> El pattern matching para switch es una característica que evolucionó desde preview en Java 17 hasta estable en Java 21. Las cláusulas `when` permiten añadir condiciones adicionales (guarded patterns).

---

## 2.6 Estructuras de Control Repetitivas (Bucles)

Los bucles permiten ejecutar un bloque de código múltiples veces.

### 2.6.1 Bucle `for` Tradicional

```java
// Estructura: for (inicialización; condición; actualización) { cuerpo }

// Contar del 1 al 5
for (int i = 1; i <= 5; i++) {
    System.out.println("Iteración: " + i);
}
// Salida: Iteración: 1, Iteración: 2, ..., Iteración: 5

// Recorrer un arreglo con índice
int[] numeros = {10, 20, 30, 40, 50};
for (int i = 0; i < numeros.length; i++) {
    System.out.printf("numeros[%d] = %d%n", i, numeros[i]);
}
```

Las tres partes del `for` son opcionales (pero los punto y coma son obligatorios):

```java
// inicialización fuera
int i = 0;
for ( ; i < 10; i++) { ... }

// condición omitida → bucle infinito
for (int j = 0; ; j++) {
    if (j >= 10) break;
}

// for infinito (equivale a while(true))
for ( ; ; ) {
    // se necesita un break para salir
}
```

### 2.6.2 Bucle `for-each` (Enhanced For Loop)

Itera sobre todos los elementos de un arreglo o una colección sin necesidad de índice.

```java
int[] numeros = {10, 20, 30, 40, 50};

for (int num : numeros) {
    System.out.println(num);
}
// Salida: 10, 20, 30, 40, 50

// Con Strings
String[] frutas = {"Manzana", "Pera", "Uva"};
for (String fruta : frutas) {
    System.out.println(fruta);
}

// Con var (Java 10+)
for (var num : numeros) {
    System.out.println(num);
}
```

**Limitaciones del for-each:**
- Solo recorre **hacia adelante**, de principio a fin.
- No accede al índice del elemento actual.
- No puede modificar el arreglo/colección que está recorriendo (puede lanzar `ConcurrentModificationException`).

### 2.6.3 for-each vs for indexado: Rendimiento en ArrayList vs LinkedList

No todos los bucles son iguales. La elección del tipo de bucle puede tener un impacto dramático en el rendimiento dependiendo de la estructura de datos subyacente.

#### ArrayList: acceso aleatorio O(1)

```java
// ArrayList usa un arreglo interno → get(i) es O(1)
ArrayList<Integer> lista = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    lista.add(i);
}

// for indexado en ArrayList: BUENO
long inicio = System.nanoTime();
long suma = 0;
for (int i = 0; i < lista.size(); i++) {
    suma += lista.get(i);           // O(1) — acceso directo al arreglo
}
long fin = System.nanoTime();
System.out.println("ArrayList for-i: " + (fin - inicio) / 1_000_000 + " ms");

// for-each en ArrayList: IGUAL DE BUENO (el compilador lo convierte a for-i)
inicio = System.nanoTime();
suma = 0;
for (int valor : lista) {
    suma += valor;                  // El compilador genera código equivalente a for-i
}
fin = System.nanoTime();
System.out.println("ArrayList for-each: " + (fin - inicio) / 1_000_000 + " ms");
```

#### LinkedList: acceso secuencial O(n) por índice

```java
// LinkedList usa nodos encadenados → get(i) recorre desde el inicio: O(n)
LinkedList<Integer> linkedList = new LinkedList<>();
for (int i = 0; i < 50_000; i++) {  // 50k es suficiente para ver la diferencia
    linkedList.add(i);
}

// for indexado en LinkedList: FATAL — O(n²)
inicio = System.nanoTime();
suma = 0;
for (int i = 0; i < linkedList.size(); i++) {
    suma += linkedList.get(i);      // O(n) — cada get(i) recorre desde el inicio
}
fin = System.nanoTime();
System.out.println("LinkedList for-i: " + (fin - inicio) / 1_000_000 + " ms");
// Puede tardar varios segundos para 50k elementos

// for-each en LinkedList: EXCELENTE — O(n)
inicio = System.nanoTime();
suma = 0;
for (int valor : linkedList) {
    suma += valor;                  // Usa el iterador, recorrido secuencial O(n)
}
fin = System.nanoTime();
System.out.println("LinkedList for-each: " + (fin - inicio) / 1_000_000 + " ms");
// Milisegundos — el iterador mantiene un puntero al nodo actual
```

**Tabla resumen del rendimiento:**

| Estructura   | for indexado (get(i))        | for-each                        |
|-------------|------------------------------|---------------------------------|
| `ArrayList`  | O(n) — bueno                 | O(n) — bueno (mismo código generado) |
| `LinkedList` | O(n²) — **catastrófico**     | O(n) — bueno (usa iterador)     |
| Array (`[]`)  | O(n) — bueno                 | O(n) — bueno                    |
| `HashSet`    | No aplica (sin índice)       | O(n) — bueno                    |

**Regla de oro:** Usa `for-each` a menos que necesites explícitamente el índice. El compilador genera el código más eficiente para la estructura de datos en uso.

### 2.6.4 Bucle `while`

Evalúa la condición **antes** de cada iteración. El cuerpo puede no ejecutarse nunca si la condición es falsa inicialmente.

```java
int contador = 0;
while (contador < 5) {
    System.out.println("Contador: " + contador);
    contador++;
}
// Salida: Contador: 0, 1, 2, 3, 4
```

Ejemplo típico: leer entrada del usuario hasta que ingrese un valor válido.

```java
import java.util.Scanner;

Scanner scanner = new Scanner(System.in);
int numero;

while (true) {
    System.out.print("Ingrese un número positivo: ");
    numero = scanner.nextInt();
    if (numero > 0) {
        break;
    }
    System.out.println("Número inválido. Intente de nuevo.");
}
System.out.println("Gracias. Número: " + numero);
```

### 2.6.5 Bucle `do-while`

Evalúa la condición **después** de cada iteración. Garantiza que el cuerpo se ejecuta **al menos una vez**.

```java
int numero;
do {
    System.out.print("Adivina un número entre 1 y 10: ");
    numero = scanner.nextInt();
} while (numero < 1 || numero > 10);

System.out.println("Número válido: " + numero);
```

**Diferencia clave entre `while` y `do-while`:**

```java
// while: si la condición es falsa, no se ejecuta nunca
int x = 0;
while (x > 10) {
    System.out.println("Esto NUNCA se imprime");
}

// do-while: se ejecuta al menos una vez, aunque la condición sea falsa
int y = 0;
do {
    System.out.println("Esto se imprime UNA VEZ");
} while (y > 10);
```

### 2.6.6 `break` y `continue`

#### `break` — Sale del bucle inmediatamente

```java
// Buscar un valor y dejar de iterar cuando se encuentra
int[] datos = {4, 7, 2, 9, 1, 6};
int buscado = 9;
boolean encontrado = false;

for (int n : datos) {
    if (n == buscado) {
        encontrado = true;
        break;   // salir del bucle, no seguir iterando inútilmente
    }
}

System.out.println(encontrado ? "Encontrado" : "No encontrado");
```

#### `continue` — Salta a la siguiente iteración

```java
// Imprimir solo números pares
for (int i = 1; i <= 10; i++) {
    if (i % 2 != 0) {
        continue;  // saltar números impares
    }
    System.out.print(i + " ");  // 2 4 6 8 10
}
```

### 2.6.7 Etiquetas y `break`/`continue` Etiquetados

Cuando tienes bucles anidados, puedes usar etiquetas para especificar cuál bucle quieres romper o continuar.

```java
busqueda:
for (int i = 0; i < 5; i++) {
    for (int j = 0; j < 5; j++) {
        if (j == 3) {
            break busqueda;    // sale de AMBOS bucles, no solo del interno
        }
        System.out.println("(" + i + ", " + j + ")");
    }
}
// Imprime pares hasta que j llega a 3 en la primera fila
```

```java
procesarDatos:
for (int[] fila : matriz) {
    for (int valor : fila) {
        if (valor < 0) {
            continue procesarDatos;  // pasa a la siguiente fila
        }
        // procesar valor...
    }
}
```

> Las etiquetas son una herramienta poderosa pero deben usarse con moderación. Si tu lógica requiere muchas etiquetas, considera refactorizar a métodos separados.

### 2.6.8 Bucles Infinitos y Cómo Evitarlos

Un bucle infinito ocurre cuando la condición nunca se vuelve falsa o no hay un mecanismo de salida.

```java
// Bucle infinito deliberado (útil en servidores, juegos, UIs)
while (true) {
    // procesar...
    if (condicionDeSalida) {
        break;  // salida explícita
    }
}

// Error común: olvidar actualizar la variable de control
int contador = 0;
while (contador < 10) {
    System.out.println(contador);
    // olvidé escribir contador++  →  bucle infinito
}

// Error común: modificar la variable en dirección equivocada
for (int i = 10; i > 0; i++) {   // i++ debería ser i--
    System.out.println(i);        // bucle infinito
}

// Error común: punto y coma extra
for (int i = 0; i < 10; i++) ;  // el ; cierra el for, el bloque siguiente no es parte del bucle
{
    System.out.println("Esto solo se ejecuta UNA vez, no 10");
}
```

**Buenas prácticas para evitar bucles infinitos accidentales:**
- Revisa que la variable de control se actualiza correctamente en cada iteración.
- Asegúrate de que la condición de parada es alcanzable.
- Prefiere `for-each` cuando recorres colecciones completas.
- Usa el `for` tradicional cuando necesitas índice y sabes cuántas iteraciones harás.
- Usa `while` cuando la cantidad de iteraciones sea desconocida de antemano.

---

## 2.7 La Clase `String`

En Java, `String` es una **clase** del paquete `java.lang`, no un tipo primitivo. Representa una secuencia inmutable de caracteres Unicode. Esta sección te llevará desde lo básico hasta los rincones más profundos del manejo de cadenas en Java.

### 2.7.1 Creación de Strings

Hay dos formas de crear un objeto `String`:

```java
// Forma 1: Mediante un literal (recomendada y más común)
String saludo = "Hola Mundo";

// Forma 2: Mediante el operador new (crea un nuevo objeto)
String nombre = new String("María");
```

**¿Cuál es la diferencia?** Los strings creados con literales se almacenan en el **String Pool**, una zona especial del heap que reutiliza instancias idénticas. Los strings creados con `new` siempre generan un objeto nuevo fuera del pool.

```java
String a = "Java";
String b = "Java";
String c = new String("Java");

System.out.println(a == b);       // true  — mismo objeto en el String Pool
System.out.println(a == c);       // false — c es un objeto distinto
System.out.println(a.equals(c));  // true  — mismo contenido
```

### 2.7.2 Inmutabilidad de `String`

Un objeto `String` **no puede modificarse** después de ser creado. Cualquier operación que "modifique" un string en realidad crea un nuevo objeto.

```java
String texto = "Hola";
texto.toUpperCase();              // Crea un NUEVO string "HOLA"...
System.out.println(texto);        // Imprime "Hola" — el original NO cambió

String nuevo = texto.toUpperCase();
System.out.println(nuevo);        // Imprime "HOLA" — el nuevo objeto sí se capturó
```

Esta inmutabilidad tiene ventajas importantes:
- **Seguridad:** Los strings no pueden ser alterados inesperadamente por otros hilos o código malicioso.
- **Hashing consistente:** El `hashCode()` de un string se calcula una sola vez y se cachea (campo `hash` interno con valor por defecto 0).
- **String Pool:** Permite la reutilización de instancias, ahorrando memoria.
- **Candidato natural para claves de Map:** Su inmutabilidad garantiza que el hash no cambia.

### 2.7.3 Métodos Comunes de `String`

```java
String str = "  ¡Bienvenido a Java!  ";
String vacia = "";

// Longitud
int len = str.length();                 // 22 (incluye espacios)

// Extraer carácter
char c = str.charAt(3);                 // 'B' (índice basado en 0)

// Substring
String sub1 = str.substring(4);         // "Bienvenido a Java!  "
String sub2 = str.substring(4, 14);     // "Bienvenido" (índice final exclusivo)

// Búsqueda
int idx1 = str.indexOf("Java");          // 17
int idx2 = str.indexOf('e');             // 6 (primera ocurrencia)
int idx3 = str.lastIndexOf('a');         // 18 (última ocurrencia)

// Comparación
String a = "Hola";
String b = "hola";
boolean iguales = a.equals(b);            // false (sensible a mayúsculas)
boolean igualesIgnore = a.equalsIgnoreCase(b);  // true (ignora mayúsculas)

// Mayúsculas / minúsculas
String mayus = str.toUpperCase();        // "  ¡BIENVENIDO A JAVA!  "
String minus = str.toLowerCase();        // "  ¡bienvenido a java!  "

// Eliminar espacios al inicio y final
String limpio = str.trim();              // "¡Bienvenido a Java!"
// Java 11+: strip() también elimina espacios Unicode
String strip = str.strip();              // "¡Bienvenido a Java!"
String stripLeading = str.stripLeading(); // Elimina solo al inicio
String stripTrailing = str.stripTrailing(); // Elimina solo al final

// Reemplazo
String reemplazado = str.replace('a', 'X');   // reemplaza caracter
String reemplazado2 = str.replace("Java", "Kotlin");  // reemplaza subcadena
String reemplazadoRegex = str.replaceAll("[aeiou]", "*"); // reemplaza con regex

// División
String datos = "manzana,pera,uva,mango";
String[] partes = datos.split(",");      // ["manzana", "pera", "uva", "mango"]

String textoMulti = "uno dos  tres";
String[] palabras = textoMulti.split("\\s+"); // split por uno o más espacios

// Verificar contenido
boolean contiene = str.contains("Java");     // true
boolean empieza = str.startsWith("  ¡B");    // true
boolean termina = str.endsWith("!  ");       // true
boolean esVacia = vacia.isEmpty();           // true
// Java 11+
boolean esBlanco = "   ".isBlank();          // true (vacío o solo espacios)

// Formateo
String formato = String.format("Hola %s, tienes %d años", "Ana", 25);
// "Hola Ana, tienes 25 años"

// valueOf (convierte cualquier tipo a String)
String numStr = String.valueOf(123);         // "123"
String boolStr = String.valueOf(true);       // "true"
String charStr = String.valueOf('X');        // "X"
```

**Ejemplo práctico: validador de correo electrónico:**

```java
public class ValidadorCorreo {
    public static boolean esCorreoValido(String correo) {
        if (correo == null || correo.isBlank()) {
            return false;
        }

        String limpio = correo.trim().toLowerCase();

        if (!limpio.contains("@")) {
            return false;
        }

        String[] partes = limpio.split("@");
        if (partes.length != 2) {
            return false;
        }

        String local = partes[0];
        String dominio = partes[1];

        if (local.isEmpty() || dominio.isEmpty()) {
            return false;
        }

        return dominio.contains(".")
            && !dominio.startsWith(".")
            && !dominio.endsWith(".");
    }
}
```

### 2.7.4 Concatenación de Strings

```java
// Operador +
String nombre = "Carlos";
String apellido = "García";
String completo = nombre + " " + apellido;  // "Carlos García"

// Concatenar con otros tipos (conversión automática)
String mensaje = "Edad: " + 30;             // "Edad: 30"
String info = "Promedio: " + 8.5;           // "Promedio: 8.5"
```

#### Qué hace realmente el compilador con el operador +

Antes de Java 9, el compilador traducía las concatenaciones con `+` a código con `StringBuilder`:

```java
// Código que escribes:
String resultado = "Hola " + nombre + ", tienes " + edad + " años";

// Lo que generaba el compilador (pre-Java 9):
// String resultado = new StringBuilder()
//     .append("Hola ")
//     .append(nombre)
//     .append(", tienes ")
//     .append(edad)
//     .append(" años")
//     .toString();
```

A partir de Java 9, el compilador usa **`invokedynamic`** y **`StringConcatFactory`** (JEP 280) para generar estrategias de concatenación más eficientes, que pueden usar `StringBuilder` o crear el `String` directamente si el tamaño final se puede predecir.

**Conclusión:** Usar `+` para concatenaciones simples (pocas, fuera de bucles) está bien y el compilador lo optimiza. Para bucles, usa `StringBuilder` explícitamente.

### 2.7.5 `StringBuilder` y `StringBuffer`

Ambas clases representan secuencias **mutables** de caracteres, diseñadas para construir strings de manera eficiente.

| Característica  | `StringBuilder`                  | `StringBuffer`                   |
|-----------------|----------------------------------|----------------------------------|
| Hilos (threads) | No sincronizado (no thread-safe) | Sincronizado (thread-safe)       |
| Rendimiento     | Más rápido                       | Más lento (overhead de sincronización) |
| Uso recomendado | Código de un solo hilo           | Código con múltiples hilos       |
| Versión Java    | Java 5+                          | Java 1.0+                        |

```java
// Ejemplo con StringBuilder (la opción preferida en la mayoría de casos)
StringBuilder sb = new StringBuilder();

sb.append("Hola");
sb.append(", ");
sb.append("mundo");
sb.append("!");
String resultado = sb.toString();   // "Hola, mundo!"

// Métodos encadenados (chaining)
String texto = new StringBuilder()
    .append("Línea 1\n")
    .append("Línea 2\n")
    .append("Línea 3")
    .toString();

// Otros métodos útiles de StringBuilder
StringBuilder builder = new StringBuilder("Java");

builder.insert(0, "Lenguaje ");     // "Lenguaje Java"
builder.delete(0, 9);               // "Java" (elimina "Lenguaje ")
builder.replace(0, 2, "Ja");        // "Java" (reemplaza caracteres en rango)
builder.reverse();                   // "avaJ"
builder.setCharAt(0, 'J');          // modifica un carácter en una posición
builder.setLength(0);               // limpia el StringBuilder
builder.capacity();                  // 16 (capacidad inicial por defecto)
builder.ensureCapacity(1000);       // garantiza capacidad mínima sin redimensionar
```

#### Benchmark de rendimiento: String vs StringBuilder

```java
public class StringBenchmark {
    public static void main(String[] args) {
        final int ITERACIONES = 100_000;

        // Test con String + (catastrófico para ITERACIONES grandes)
        long inicioStr = System.currentTimeMillis();
        String s = "";
        for (int i = 0; i < ITERACIONES; i++) {
            s += "a";
        }
        long finStr = System.currentTimeMillis();
        System.out.println("String +: " + (finStr - inicioStr)
            + " ms (longitud: " + s.length() + ")");

        // Test con StringBuilder
        long inicioSb = System.currentTimeMillis();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < ITERACIONES; i++) {
            sb.append("a");
        }
        String resultado = sb.toString();
        long finSb = System.currentTimeMillis();
        System.out.println("StringBuilder: " + (finSb - inicioSb)
            + " ms (longitud: " + resultado.length() + ")");

        // Test con StringBuilder con capacidad pre-inicializada
        long inicioSbPre = System.currentTimeMillis();
        StringBuilder sbPre = new StringBuilder(ITERACIONES);
        for (int i = 0; i < ITERACIONES; i++) {
            sbPre.append("a");
        }
        resultado = sbPre.toString();
        long finSbPre = System.currentTimeMillis();
        System.out.println("StringBuilder (pre-dimensionado): "
            + (finSbPre - inicioSbPre) + " ms");
    }
}
// Resultados típicos con 100,000 iteraciones:
// String +:             ~800-2000 ms (~100,000 objetos temporales creados)
// StringBuilder:         ~1-3 ms
// StringBuilder (pre):   ~1 ms
```

### 2.7.6 El String Pool en Profundidad

El **String Pool** (piscina de strings, también llamado String Intern Pool) es una tabla hash especial dentro del heap de la JVM donde se almacenan los strings literales para ser reutilizados.

#### Cómo funciona realmente

1. Cuando la JVM encuentra un **literal** de string (ej. `"Hola"`), busca ese string en el String Pool.
2. Si existe, devuelve la referencia al objeto existente.
3. Si no existe, crea el string, lo inserta en el pool y devuelve la referencia.

```java
String s1 = "Java";                   // va al String Pool
String s2 = "Java";                   // reutiliza el mismo objeto del pool
String s3 = new String("Java");       // nuevo objeto FUERA del pool
String s4 = s3.intern();             // busca "Java" en el pool... lo encuentra, devuelve el del pool

System.out.println(s1 == s2);         // true  — mismo objeto en el pool
System.out.println(s1 == s3);         // false — objetos distintos
System.out.println(s1 == s4);         // true  — s4 apunta al objeto del pool
System.out.println(s3 == s4);         // false — s3 sigue apuntando al objeto fuera del pool
```

#### Configuración del String Pool: -XX:StringTableSize

El String Pool se implementa como una tabla hash con buckets. Su tamaño afecta directamente al rendimiento de `intern()` y a la probabilidad de colisiones.

```bash
# Configurar el tamaño del String Pool (debe ser un número primo o potencia de 2)
java -XX:StringTableSize=1000003 MiApp

# Ver el tamaño actual y estadísticas (JVM en modo debug)
java -XX:+PrintStringTableStatistics MiApp
```

**Valores por defecto:** El tamaño mínimo es 60013. La JVM puede hacer resize automático en algunas implementaciones. Para aplicaciones que internan muchos strings (procesamiento de XML/JSON masivo), aumentar este valor reduce colisiones y mejora el rendimiento.

#### Impacto en rendimiento en parsing de JSON/XML

Al parsear archivos JSON o XML grandes, muchos strings se repiten (etiquetas, nombres de campos). Sin internamiento, cada ocurrencia genera un nuevo objeto String, duplicando memoria. Con internamiento, solo existe una copia:

```java
// Simulación de parsing sin intern
String generarEtiqueta(String nombre) {
    return new String(nombre.toCharArray());  // siempre nuevo objeto
}

// Simulación de parsing con intern
String generarEtiquetaOptimizada(String nombre) {
    return nombre.intern();  // reutiliza si ya existe en el pool
}
```

**Precaución:** No internes strings indiscriminadamente. `intern()` tiene un costo (búsqueda en tabla hash) y los strings internados nunca se liberan mientras la clase que los referencia siga cargada. Úsalo solo cuando la reutilización de strings sea significativa y el número de strings distintos no sea excesivo.

### 2.7.7 Compact Strings (Java 9+)

**JEP 254: Compact Strings** fue una de las mejoras de rendimiento más importantes en Java 9.

#### Antes de Java 9

Internamente, `String` se almacenaba como un `char[]`. Cada `char` ocupa **2 bytes** (16 bits, UTF-16). Si tu aplicación trabajaba predominantemente con caracteres Latin-1 (ASCII, ISO-8859-1, la mayoría de lenguajes occidentales), estabas desperdiciando 1 byte por carácter.

```
"Hola Mundo" → char[10] → 10 caracteres × 2 bytes = 20 bytes solo para los datos
```

#### A partir de Java 9

`String` se almacena como un `byte[]` más un campo `coder` que indica la codificación:

| Campo         | Descripción                                           |
|---------------|-------------------------------------------------------|
| `byte[] value` | Los caracteres codificados                            |
| `byte coder`   | `LATIN1 = 0` (1 byte/carácter) o `UTF16 = 1` (2 bytes/carácter) |
| `int hash`     | Hash cacheado (heredado de antes)                     |

Si todos los caracteres del string caben en Latin-1 (U+0000 a U+00FF), Java usa 1 byte por carácter, reduciendo el uso de memoria a la mitad para strings occidentales.

```
"Hola Mundo" → byte[10] + coder=LATIN1 → 10 bytes de datos
```

**Impacto práctico:** Las aplicaciones que manejan grandes volúmenes de texto occidental (logs, JSON, HTML, código fuente) vieron reducciones de memoria de heap del orden de 10-15% y mejoras en el rendimiento del Garbage Collector al haber menos presión de memoria.

### 2.7.8 Text Blocks (Java 15+)

**JEP 378: Text Blocks** introdujo una sintaxis para strings multilínea que conserva el formato, elimina la necesidad de escape y hace que el código que contiene JSON, SQL, HTML o XML sea legible.

#### Antes de Text Blocks

```java
// Pesadilla de escapes y concatenación
String json = "{\n" +
              "  \"nombre\": \"María\",\n" +
              "  \"edad\": 28,\n" +
              "  \"hobbies\": [\"leer\", \"música\"]\n" +
              "}";
```

#### Con Text Blocks

```java
String json = """
    {
      "nombre": "María",
      "edad": 28,
      "hobbies": ["leer", "música"]
    }
    """;
```

**Reglas de formateo de Text Blocks:**

1. Los delimitadores `"""` de apertura y cierre deben estar en líneas separadas (o el contenido empieza en la línea siguiente).
2. El **espacio en blanco incidental** (indentación común a todas las líneas) se elimina automáticamente.
3. El contenido se determina por la línea con el menor margen izquierdo.
4. Las líneas terminan con `\n`.

```java
// La indentación común se determina por la línea más a la izquierda
String html = """
        <html>
            <body>
                <h1>Título</h1>
            </body>
        </html>
        """;
// Resultado:
// <html>
//     <body>
//         <h1>Título</h1>
//     </body>
// </html>
```

**Secuencias de escape en Text Blocks:**

| Secuencia | Significado                                                       |
|-----------|-------------------------------------------------------------------|
| `\n`      | Nueva línea (explícita)                                           |
| `\t`      | Tabulación                                                        |
| `\\`      | Barra invertida                                                   |
| `\"`      | Comilla doble                                                     |
| `\s`      | Espacio (previene que se elimine el espacio al final de línea)    |
| `\<eol>`  | Escape de continuación de línea (el salto de línea no se incluye) |

```java
// \s fuerza un espacio que de otro modo se perdería
String poema = """
    Rosas son rojas,\s
    violetas son azules\s
    """;
// "Rosas son rojas, violetas son azules "

// \<eol> une líneas (útil para código SQL legible sin saltos)
String sql = """
    SELECT id, nombre, email \
    FROM usuarios \
    WHERE activo = true \
    ORDER BY nombre
    """;
// "SELECT id, nombre, email FROM usuarios WHERE activo = true ORDER BY nombre"
```

#### Ejemplos prácticos de Text Blocks

**SQL:**

```java
String query = """
    SELECT u.id, u.nombre, COUNT(p.id) as total_pedidos
    FROM usuarios u
    LEFT JOIN pedidos p ON u.id = p.usuario_id
    WHERE u.fecha_registro > ?
      AND u.activo = true
    GROUP BY u.id, u.nombre
    ORDER BY total_pedidos DESC
    LIMIT ?
    """;
```

**HTML:**

```java
String pagina = """
    <!DOCTYPE html>
    <html lang="es">
    <head>
        <meta charset="UTF-8">
        <title>%s</title>
    </head>
    <body>
        <h1>%s</h1>
        <p>%s</p>
    </body>
    </html>
    """.formatted(titulo, encabezado, contenido);
```

**JSON:**

```java
String config = """
    {
      "app": {
          "name": "MiApp",
          "version": "2.0.0",
          "puerto": 8080
      },
      "database": {
          "url": "jdbc:postgresql://localhost:5432/midb",
          "usuariosMax": 50
      }
    }
    """;
```

### 2.7.9 `String.indent()` y `String.transform()` (Java 12+)

#### indent(int n) — Java 12

Añade `n` espacios al inicio de cada línea (o elimina si n es negativo, hasta donde sea posible):

```java
String original = "primera línea\nsegunda línea\ntercera línea";

String indentado = original.indent(4);
// "    primera línea\n    segunda línea\n    tercera línea"

// Indentación negativa: elimina espacios
String muyIndentado = "        línea\n        otra";
String menosIndentado = muyIndentado.indent(-4);
// "    línea\n    otra"

// Si n es negativo y excede los espacios disponibles, no elimina caracteres no-espacio
String conPocoEspacio = "  texto";
String sinIndent = conPocoEspacio.indent(-10);
// "texto" — solo elimina los 2 espacios que había
```

#### transform(Function<String, R>) — Java 12

Aplica una función de transformación al string:

```java
String resultado = "  hola  ".transform(String::strip)
                              .transform(String::toUpperCase);
// "HOLA"

// Ejemplo: parsear y formatear en una línea
int numero = "42".transform(Integer::parseInt);  // 42

// Ejemplo: encriptar un string
String encriptado = "secreto".transform(
    s -> Base64.getEncoder().encodeToString(s.getBytes()));
```

### 2.7.10 Expresiones Regulares Básicas

Java ofrece soporte completo para expresiones regulares a través del paquete `java.util.regex`.

#### String.matches() — verificación simple

```java
String email = "usuario@dominio.com";
boolean esEmailValido = email.matches("^[\\w.-]+@[\\w.-]+\\.\\w{2,}$");
System.out.println(esEmailValido); // true

String telefono = "123-456-7890";
boolean esTelefono = telefono.matches("\\d{3}-\\d{3}-\\d{4}");
System.out.println(esTelefono);   // true
```

> **Nota:** `matches()` verifica que **todo** el string coincida con el patrón. Si solo quieres verificar si el patrón aparece en algún lugar, usa `find()` con `Matcher`.

#### Pattern y Matcher — control completo

```java
import java.util.regex.Pattern;
import java.util.regex.Matcher;

String texto = "Contacto: ana@mail.com, luis@empresa.org";
String regex = "(\\w+)@(\\w+\\.\\w{2,})";

Pattern pattern = Pattern.compile(regex);
Matcher matcher = pattern.matcher(texto);

while (matcher.find()) {
    String emailCompleto = matcher.group(0);
    String usuario = matcher.group(1);
    String dominio = matcher.group(2);
    System.out.printf("Email: %s | Usuario: %s | Dominio: %s | Pos: %d-%d%n",
        emailCompleto, usuario, dominio, matcher.start(), matcher.end());
}
// Email: ana@mail.com | Usuario: ana | Dominio: mail.com | Pos: 11-23
// Email: luis@empresa.org | Usuario: luis | Dominio: empresa.org | Pos: 25-41
```

#### Métodos clave de Pattern y Matcher

```java
Pattern pattern = Pattern.compile("\\d+");

// find() — busca la siguiente coincidencia
Matcher m = pattern.matcher("Hay 42 manzanas y 7 peras");
while (m.find()) {
    System.out.println("Encontrado: " + m.group() + " en posición " + m.start());
}

// matches() — ¿el string completo coincide?
System.out.println(Pattern.matches("\\d+", "12345"));   // true
System.out.println(Pattern.matches("\\d+", "12a45"));   // false

// replaceAll() — reemplaza todas las coincidencias
String reemplazado = pattern.matcher("foo123bar456").replaceAll("NUM");
// "fooNUMbarNUM"

// split() — divide el string usando la regex como delimitador
String[] partes = Pattern.compile(",\\s*")
    .split("rojo, verde, azul, amarillo");
// ["rojo", "verde", "azul", "amarillo"]
```

#### Patrones comunes de regex útiles

| Propósito            | Patrón                                   |
|----------------------|------------------------------------------|
| Email                | `[\\w.-]+@[\\w.-]+\\.\\w{2,}`           |
| URL                  | `https?://[\\w./?=&%-]+`                 |
| Fecha dd/mm/aaaa     | `\\d{2}/\\d{2}/\\d{4}`                   |
| Solo letras          | `[a-zA-ZáéíóúüñÁÉÍÓÚÜÑ ]+`              |
| Solo números         | `\\d+`                                   |
| Números decimales    | `\\d+(\\.\\d+)?`                          |
| Código postal español| `\\d{5}`                                 |
| Varios espacios      | `\\s+`                                   |

#### Flags de compilación de Pattern

```java
// CASE_INSENSITIVE — ignora mayúsculas/minúsculas
Pattern p1 = Pattern.compile("java", Pattern.CASE_INSENSITIVE);
System.out.println(p1.matcher("Java").matches());  // true

// MULTILINE — ^ y $ coinciden con inicio/fin de cada línea
Pattern p2 = Pattern.compile("^//.*$", Pattern.MULTILINE);
Matcher m2 = p2.matcher("código;\n// comentario\nmás código;\n// otro");
while (m2.find()) {
    System.out.println(m2.group());  // // comentario, // otro
}

// DOTALL — . coincide también con saltos de línea
Pattern p3 = Pattern.compile("<!--.*?-->", Pattern.DOTALL);
// Permite capturar comentarios HTML que ocupan múltiples líneas

// Combinar flags con |
Pattern p4 = Pattern.compile("patron",
    Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);
```

### 2.7.11 Comparación de String: == vs equals()

Este es uno de los errores más comunes en Java. Repitámoslo para que quede claro:

```java
String a = "hola";
String b = "hola";
String c = new String("hola");

a == b       // true  — comparación de REFERENCIA. Ambos apuntan al mismo objeto del pool.
a == c       // false — comparación de REFERENCIA. Son objetos diferentes en memoria.
a.equals(b)  // true  — comparación de CONTENIDO. Las secuencias de caracteres son iguales.
a.equals(c)  // true  — comparación de CONTENIDO. Las secuencias de caracteres son iguales.
```

**Regla de oro para Strings:** Siempre usa `equals()` para comparar el contenido de strings. Usa `==` solo si necesitas verificar si dos referencias apuntan exactamente al mismo objeto.

**Invertir la comparación para evitar NullPointerException:**

```java
String entrada = obtenerDesdeFuenteExterna(); // podría ser null

// Peligroso: NullPointerException si entrada es null
if (entrada.equals("admin")) { ... }

// Seguro: "admin" nunca es null
if ("admin".equals(entrada)) { ... }
```
## 2.8 Clases Wrapper

Las **clases wrapper** (envoltorio) convierten cada tipo primitivo en un objeto. Son esenciales para usar primitivos en colecciones (`List<int>` no existe, debes usar `List<Integer>`) y proporcionan métodos de utilidad.

### 2.8.1 Correspondencia Primitivo ↔ Wrapper

| Primitivo | Wrapper      | Tamaño |
|-----------|-------------|--------|
| `byte`    | `Byte`      | 8 bits |
| `short`   | `Short`     | 16 bits |
| `int`     | `Integer`   | 32 bits |
| `long`    | `Long`      | 64 bits |
| `float`   | `Float`     | 32 bits |
| `double`  | `Double`    | 64 bits |
| `char`    | `Character` | 16 bits |
| `boolean` | `Boolean`   | ~1 bit |

Todas las clases wrapper son **inmutables** y `final` (no se pueden extender).

### 2.8.2 Constantes MAX_VALUE y MIN_VALUE

Cada wrapper expone las constantes del tipo:

```java
int maxInt = Integer.MAX_VALUE;       // 2,147,483,647
int minInt = Integer.MIN_VALUE;       // -2,147,483,648
long maxLong = Long.MAX_VALUE;        // 9,223,372,036,854,775,807
double maxDouble = Double.MAX_VALUE;  // 1.7976931348623157E308
double minPosDouble = Double.MIN_VALUE; // 4.9E-324 (el menor positivo, no el más negativo)

// Constantes de punto flotante especiales
double posInf = Double.POSITIVE_INFINITY;
double negInf = Double.NEGATIVE_INFINITY;
double nan = Double.NaN;

// Tamaños en bits y bytes
int bitsInt = Integer.SIZE;         // 32
int bytesInt = Integer.BYTES;       // 4
```

### 2.8.3 Métodos de Conversión (Parsing)

```java
// String → primitivo (parseXxx)
int i = Integer.parseInt("42");
long l = Long.parseLong("10000000000");
double d = Double.parseDouble("3.1416");
float f = Float.parseFloat("2.5f");
boolean b = Boolean.parseBoolean("true");   // "true" → true, cualquier otra cosa → false
byte by = Byte.parseByte("127");
short s = Short.parseShort("32000");

// String → primitivo con base
int bin = Integer.parseInt("1010", 2);    // 10
int oct = Integer.parseInt("77", 8);      // 63
int hex = Integer.parseInt("FF", 16);     // 255

// String → wrapper (valueOf) — reutiliza el cache cuando aplica
Integer wi = Integer.valueOf("42");        // usa el cache (-128 a 127)
Integer wi2 = Integer.valueOf(42);         // también desde primitivo
Double wd = Double.valueOf("3.14");

// De base numérica a String
String binario = Integer.toBinaryString(42);    // "101010"
String octal = Integer.toOctalString(64);       // "100"
String hexadecimal = Integer.toHexString(255);  // "ff"
String hexMayus = Integer.toHexString(255).toUpperCase(); // "FF"

// Parsear números con formato: Integer.decode()
Integer decodificado = Integer.decode("0xFF");  // 255
Integer decodificado2 = Integer.decode("010");  // 8 (octal por el 0 inicial)
Integer decodificado3 = Integer.decode("#FF");  // 255
```

### 2.8.4 Métodos de Comparación y Aritméticos

```java
// Comparación segura (sin riesgo de NullPointerException si los valores son primitivos)
int cmp = Integer.compare(5, 10);     // -1 (negativo si a < b)
int cmp2 = Long.compare(10L, 10L);    // 0 (cero si iguales)
int cmp3 = Double.compare(3.0, 2.0);  // 1 (positivo si a > b)

// Para wrappers potencialmente null, hay que manejar el null:
Integer a = null;
Integer b = 5;
// a.compareTo(b);  // NullPointerException
int cmpSeguro = Integer.compare(
    a != null ? a : 0,
    b != null ? b : 0
);

// Métodos aritméticos (Java 8+)
int suma = Integer.sum(3, 7);         // 10
int max = Integer.max(3, 7);          // 7
int min = Integer.min(3, 7);          // 3
long maxLong = Long.max(100L, 200L);  // 200
double minDouble = Double.min(1.5, 2.5); // 1.5
```

### 2.8.5 Conversión entre Tipos

```java
// Los wrappers ofrecen métodos xxxValue() para convertir entre tipos
Integer entero = 100;
int i = entero.intValue();            // 100
long l = entero.longValue();          // 100L
float f = entero.floatValue();        // 100.0f
double d = entero.doubleValue();      // 100.0
byte b = entero.byteValue();          // 100
short s = entero.shortValue();        // 100

// Double → int (con truncamiento)
Double dValor = 9.99;
int iTruncado = dValor.intValue();    // 9
```

### 2.8.6 Boolean: Cuidado con parseBoolean()

```java
// parseBoolean devuelve true SOLO para "true" (ignorando mayúsculas)
System.out.println(Boolean.parseBoolean("true"));    // true
System.out.println(Boolean.parseBoolean("TRUE"));    // true
System.out.println(Boolean.parseBoolean("false"));   // false
System.out.println(Boolean.parseBoolean("yes"));     // false — ¡no lanza excepción!
System.out.println(Boolean.parseBoolean("1"));       // false
System.out.println(Boolean.parseBoolean(null));      // false — ¡no NullPointerException!
```

### 2.8.7 Character: Métodos de Clasificación

```java
char c1 = 'A';
char c2 = '7';
char c3 = ' ';
char c4 = 'ñ';

System.out.println(Character.isLetter(c1));       // true
System.out.println(Character.isDigit(c2));        // true
System.out.println(Character.isWhitespace(c3));   // true
System.out.println(Character.isUpperCase(c1));    // true
System.out.println(Character.isLowerCase('b'));   // true
System.out.println(Character.isLetterOrDigit(c4)); // true
System.out.println(Character.isJavaIdentifierStart('_')); // true
System.out.println(Character.isJavaIdentifierPart('$'));  // true

// Conversión
System.out.println(Character.toLowerCase('A'));   // 'a'
System.out.println(Character.toUpperCase('b'));   // 'B'
System.out.println(Character.getNumericValue('7')); // 7
System.out.println(Character.toString('X'));      // "X"
```

---

## 2.9 Math API

La clase `java.lang.Math` proporciona métodos estáticos para operaciones matemáticas comunes.

### 2.9.1 Math vs StrictMath

Java ofrece dos clases con los mismos métodos:

| Clase         | Precisión                                                                 | Rendimiento |
|---------------|---------------------------------------------------------------------------|-------------|
| `Math`        | Puede usar instrucciones nativas de la CPU (más rápido, pero puede variar entre plataformas) | Mayor       |
| `StrictMath`  | Resultados idénticos bit a bit en todas las plataformas (implementación pura Java) | Menor       |

En aplicaciones típicas, `Math` es la elección correcta. Usa `StrictMath` solo si necesitas resultados **exactamente** reproducibles entre plataformas (simulaciones científicas, criptografía, juegos multiplayer con determinismo).

```java
// Math: puede delegar en instrucciones de hardware (más rápido)
double resultadoMath = Math.sin(Math.PI / 4);

// StrictMath: garantiza el mismo resultado en cualquier JVM
double resultadoStrict = StrictMath.sin(StrictMath.PI / 4);
```

> Desde Java 17, muchos métodos de `Math` y `StrictMath` convergen en la misma implementación nativa, reduciendo la diferencia práctica.

### 2.9.2 Constantes Matemáticas

```java
double pi = Math.PI;               // 3.141592653589793
double e = Math.E;                 // 2.718281828459045
double tau = Math.TAU;             // 6.283185307179586 (Java 19+)
```

### 2.9.3 Métodos Clave

**Funciones trigonométricas (en radianes):**

```java
double sin30 = Math.sin(Math.toRadians(30));  // 0.5
double cos60 = Math.cos(Math.PI / 3);         // 0.5
double tan45 = Math.tan(Math.PI / 4);         // 1.0

// Inversas
double angulo = Math.asin(1.0);           // π/2
double angulo2 = Math.atan2(1.0, 1.0);    // π/4 (atan2(y, x) da el ángulo del punto)

// Conversión grados ↔ radianes
double radianes = Math.toRadians(180);    // 3.141592653589793
double grados = Math.toDegrees(Math.PI);  // 180.0
```

**Potencias y logaritmos:**

```java
double cuadrado = Math.pow(2, 10);      // 1024.0
double raiz = Math.sqrt(144);           // 12.0
double raizCubica = Math.cbrt(27);      // 3.0 (raíz cúbica)
double logNatural = Math.log(Math.E);   // 1.0 (logaritmo natural, base e)
double logBase10 = Math.log10(1000);    // 3.0
double exp = Math.exp(2);              // e² ≈ 7.389
double hipotenusa = Math.hypot(3, 4);   // 5.0 (sqrt(x² + y²))
```

**Redondeo y valor absoluto:**

```java
double valor = -7.8;

double absoluto = Math.abs(valor);      // 7.8
int absolutoInt = Math.abs(-42);        // 42
// CUIDADO: Math.abs(Integer.MIN_VALUE) == Integer.MIN_VALUE (overflow silencioso)

double techo = Math.ceil(7.3);          // 8.0 (redondea hacia arriba)
double piso = Math.floor(7.8);          // 7.0 (redondea hacia abajo)
long redondeo = Math.round(7.5);        // 8 (redondeo clásico, half-up)
double truncado = Math.rint(7.5);       // 8.0 (redondeo "banker's rounding": half-even)

// Máximo y mínimo
int max = Math.max(10, 20);             // 20
double min = Math.min(3.14, 2.71);      // 2.71
```

**Diferencia entre round(), ceil(), floor() y rint():**

| Método      | -2.5  | -2.1  | 2.1   | 2.5   | 2.9   |
|-------------|-------|-------|-------|-------|-------|
| `ceil()`    | -2.0  | -2.0  | 3.0   | 3.0   | 3.0   |
| `floor()`   | -3.0  | -3.0  | 2.0   | 2.0   | 2.0   |
| `round()`   | -2    | -2    | 2     | 3     | 3     |
| `rint()`    | -2.0  | -2.0  | 2.0   | 2.0   | 3.0   |

### 2.9.4 Generación de Números Aleatorios

#### Math.random()

Devuelve un double pseudo-aleatorio en el rango [0.0, 1.0). Internamente usa una instancia de `java.util.Random` sincronizada.

```java
double aleatorio = Math.random();                        // [0.0, 1.0)
int dado = (int)(Math.random() * 6) + 1;                 // [1, 6]
int entre10y20 = (int)(Math.random() * 11) + 10;         // [10, 20]
double entre25y50 = 25 + Math.random() * (50 - 25);      // [25.0, 50.0)
```

#### ThreadLocalRandom (Java 7+) — Preferido para concurrencia

`Math.random()` usa un `Random` compartido con sincronización interna, lo que causa contención en entornos multi-hilo. `ThreadLocalRandom` mantiene una instancia independiente por hilo, eliminando la contención:

```java
import java.util.concurrent.ThreadLocalRandom;

// En cualquier hilo (sin necesidad de crear instancias)
int aleatorio = ThreadLocalRandom.current().nextInt();       // cualquier int
int entre1y100 = ThreadLocalRandom.current().nextInt(1, 101); // [1, 100]
long aleatorioLong = ThreadLocalRandom.current().nextLong();
double entre0y1 = ThreadLocalRandom.current().nextDouble();   // [0.0, 1.0)
boolean coin = ThreadLocalRandom.current().nextBoolean();     // true o false

// Generar un número con distribución gaussiana
double gauss = ThreadLocalRandom.current().nextGaussian();
```

**Comparación de rendimiento:** En entornos de alto rendimiento con múltiples hilos, `ThreadLocalRandom` es hasta **10x más rápido** que `Math.random()`.

#### java.util.Random y SecureRandom

```java
// Random con semilla explícita (resultados reproducibles)
Random rng = new Random(12345L);
int n1 = rng.nextInt(100);        // [0, 99]

// SecureRandom — para criptografía, tokens, contraseñas
import java.security.SecureRandom;

SecureRandom secure = new SecureRandom();  // semilla desde fuente de entropía del SO
byte[] bytes = new byte[16];
secure.nextBytes(bytes);                  // 16 bytes aleatorios criptográficamente seguros
String token = Base64.getEncoder().encodeToString(bytes);
```

### 2.9.5 Otros Métodos Útiles

```java
// signum: devuelve -1.0, 0.0, o 1.0 según el signo
double signo = Math.signum(-42.5);  // -1.0

// copySign: copia el signo de un número a otro
double copiado = Math.copySign(3.14, -1.0);    // -3.14

// nextUp / nextDown: siguiente/anterior valor representable en punto flotante
double siguiente = Math.nextUp(1.0);    // 1.0000000000000002
double anterior = Math.nextDown(1.0);   // 0.9999999999999999

// ulp: "unit in the last place" — distancia al siguiente valor representable
double ulp1 = Math.ulp(1.0);            // 2.220446049250313E-16
double ulpGrande = Math.ulp(1_000_000.0); // 1.1641532182693481E-10

// IEEEremainder: resto según IEEE 754
double restoIEEE = Math.IEEEremainder(10.0, 3.0);  // 1.0

// fma: fused multiply-add — (a * b) + c con un solo redondeo (más preciso)
double resultado = Math.fma(2.0, 3.0, 4.0);  // 10.0 (calculado con precisión extendida)

// multiplyExact y familia — lanzan ArithmeticException si overflow
int seguro = Math.multiplyExact(100_000, 100_000);  // ArithmeticException

// floorDiv y floorMod — división y módulo con redondeo hacia abajo (no hacia cero)
int floorDiv = Math.floorDiv(-7, 3);    // -3 (división normal -7/3 = -2)
int floorMod = Math.floorMod(-7, 3);    // 2  (módulo normal -7%3 = -1)
// floorMod siempre devuelve un resultado no negativo cuando el divisor es positivo
```

---

## 2.10 BigDecimal y BigInteger

### 2.10.1 ¿Por qué BigDecimal?

Como vimos en la sección 2.1.3, `float` y `double` no pueden representar valores decimales exactos. Para cálculos financieros, fiscales, monetarios o cualquier escenario que requiera precisión decimal exacta, `BigDecimal` es **obligatorio**.

### 2.10.2 Creación de BigDecimal

**Regla #1:** Usa el constructor con `String`. **Nunca** uses el constructor con `double`.

```java
import java.math.BigDecimal;
import java.math.RoundingMode;

// CORRECTO: desde String
BigDecimal precio = new BigDecimal("19.99");
BigDecimal tasa = new BigDecimal("0.21");

// FATAL: desde double (arrastra el error de precisión)
BigDecimal malo = new BigDecimal(0.1);
System.out.println(malo);
// 0.1000000000000000055511151231257827021181583404541015625
// ¡Este es el verdadero valor de 0.1 en double, heredado por BigDecimal!

// Alternativas seguras:
BigDecimal desdeString = new BigDecimal("0.1");
BigDecimal desdeValueOf = BigDecimal.valueOf(0.1);     // usa Double.toString() internamente
System.out.println(desdeString);   // 0.1
System.out.println(desdeValueOf);  // 0.1
```

> `BigDecimal.valueOf(double)` es seguro porque internamente llama a `Double.toString(double)`, que produce una representación decimal humana del double, no la representación binaria exacta.

### 2.10.3 Operaciones Aritméticas

```java
BigDecimal a = new BigDecimal("10.50");
BigDecimal b = new BigDecimal("3.25");

BigDecimal suma = a.add(b);             // 13.75
BigDecimal resta = a.subtract(b);       // 7.25
BigDecimal multiplicacion = a.multiply(b); // 34.1250
BigDecimal division = a.divide(b, 2, RoundingMode.HALF_UP); // 3.23

// La división sin RoundingMode lanza ArithmeticException si no es exacta
// ArithmeticException: Non-terminating decimal expansion
// BigDecimal division = a.divide(b);  // PELIGRO

// Operaciones con enteros largos
BigDecimal potencia = a.pow(3);        // a^3
BigDecimal absoluto = new BigDecimal("-5.5").abs();  // 5.5
BigDecimal negado = a.negate();        // -10.50
```

### 2.10.4 Redondeo (RoundingMode)

```java
BigDecimal valor = new BigDecimal("2.345");

// HALF_UP: redondeo clásico (>= 0.5 sube, < 0.5 baja)
BigDecimal halfUp = valor.setScale(2, RoundingMode.HALF_UP);    // 2.35

// HALF_DOWN: (<= 0.5 baja, > 0.5 sube)
BigDecimal halfDown = valor.setScale(2, RoundingMode.HALF_DOWN); // 2.34

// HALF_EVEN: "banker's rounding" (redondea al par más cercano)
BigDecimal halfEven = new BigDecimal("2.345")
    .setScale(2, RoundingMode.HALF_EVEN);  // 2.34 (porque 4 es par)
BigDecimal halfEven2 = new BigDecimal("2.355")
    .setScale(2, RoundingMode.HALF_EVEN);  // 2.36 (porque 6 es par)
// HALF_EVEN es el modo preferido en finanzas porque no tiene sesgo estadístico

// UP: siempre redondea hacia arriba (lejos de cero)
BigDecimal up = valor.setScale(2, RoundingMode.UP);              // 2.35

// DOWN: siempre trunca (hacia cero)
BigDecimal down = valor.setScale(2, RoundingMode.DOWN);          // 2.34

// CEILING: hacia el infinito positivo
BigDecimal ceil = new BigDecimal("-2.345")
    .setScale(2, RoundingMode.CEILING);  // -2.34

// FLOOR: hacia el infinito negativo
BigDecimal floor = new BigDecimal("-2.345")
    .setScale(2, RoundingMode.FLOOR);    // -2.35
```

### 2.10.5 Comparación

**Regla #2:** Usa `compareTo()`, **NO** `equals()`. `equals()` considera la escala; `compareTo()` compara valores numéricos.

```java
BigDecimal a = new BigDecimal("2.00");
BigDecimal b = new BigDecimal("2.0");

System.out.println(a.equals(b));       // false — escalas diferentes (2 vs 1)
System.out.println(a.compareTo(b));    // 0 — numéricamente iguales

// compareTo devuelve:
// -1 (o negativo) si a < b
//  0              si a == b
// +1 (o positivo) si a > b

// Para igualdad numérica con tolerancia de escala:
if (a.compareTo(b) == 0) {
    System.out.println("Numéricamente iguales");
}
```

### 2.10.6 Escala y Precisión

```java
BigDecimal valor = new BigDecimal("123.4500");

// Escala: número de dígitos a la derecha del punto decimal
int escala = valor.scale();            // 4

// Precisión: número total de dígitos significativos
int precision = valor.precision();     // 6 (1,2,3,4,5,0)

// Quitar ceros finales
BigDecimal strip = valor.stripTrailingZeros(); // 123.45 (escala pasa a 2)
System.out.println(strip.scale());     // 2

// Mover el punto decimal
BigDecimal desplazado = valor.movePointLeft(2);  // 1.234500
BigDecimal desplazadoDer = valor.movePointRight(2); // 12345.00
```

### 2.10.7 BigDecimal: Ejemplo Completo de Calculadora Financiera

```java
import java.math.BigDecimal;
import java.math.RoundingMode;

public class CalculadoraPrestamo {

    public static BigDecimal calcularCuotaMensual(
            BigDecimal capital, BigDecimal tasaAnual, int plazosMeses) {

        BigDecimal tasaMensual = tasaAnual
            .divide(new BigDecimal("12"), 10, RoundingMode.HALF_UP)
            .divide(new BigDecimal("100"), 10, RoundingMode.HALF_UP);

        BigDecimal unoMasTasa = BigDecimal.ONE.add(tasaMensual);
        BigDecimal factor = unoMasTasa.pow(plazosMeses);

        BigDecimal numerador = tasaMensual.multiply(factor);
        BigDecimal denominador = factor.subtract(BigDecimal.ONE);

        return capital
            .multiply(numerador)
            .divide(denominador, 2, RoundingMode.HALF_UP);
    }

    public static void main(String[] args) {
        BigDecimal capital = new BigDecimal("150000");
        BigDecimal tasaAnual = new BigDecimal("5.5");
        int plazos = 240; // 20 años

        BigDecimal cuota = calcularCuotaMensual(capital, tasaAnual, plazos);

        BigDecimal totalPagado = cuota.multiply(new BigDecimal(plazos));
        BigDecimal interesesTotales = totalPagado.subtract(capital);

        System.out.println("Capital solicitado:   " + capital + " EUR");
        System.out.println("Tasa anual:           " + tasaAnual + "%");
        System.out.println("Plazo:                " + plazos + " meses");
        System.out.println("-----------------------------------");
        System.out.println("Cuota mensual:        " + cuota + " EUR");
        System.out.println("Total a pagar:        " + totalPagado + " EUR");
        System.out.println("Intereses totales:    " + interesesTotales + " EUR");
    }
}
// Salida típica:
// Capital solicitado:   150000 EUR
// Tasa anual:           5.5%
// Plazo:                240 meses
// -----------------------------------
// Cuota mensual:        1033.53 EUR
// Total a pagar:        248047.20 EUR
// Intereses totales:    98047.20 EUR
```

### 2.10.8 BigInteger

Para enteros de precisión arbitraria (más allá de `long`):

```java
import java.math.BigInteger;

BigInteger a = new BigInteger("12345678901234567890");
BigInteger b = new BigInteger("98765432109876543210");

BigInteger suma = a.add(b);
BigInteger producto = a.multiply(b);
BigInteger potencia = a.pow(10);           // a^10

// Métodos útiles
BigInteger mcd = a.gcd(b);                // máximo común divisor
boolean esProbablePrimo = a.isProbablePrime(10); // test de primalidad
BigInteger modPow = a.modPow(b, new BigInteger("1000")); // a^b mod m
BigInteger modInverse = a.modInverse(b);  // inverso modular

// Constantes
BigInteger cero = BigInteger.ZERO;
BigInteger uno = BigInteger.ONE;
BigInteger diez = BigInteger.TEN;

// Conversión
int i = a.intValue();                      // trunca si no cabe (no recomendado)
long l = a.longValueExact();              // ArithmeticException si no cabe
String hex = a.toString(16);              // representación en base 16
```

---

## 2.11 Scope de Variables (Alcance)

El **scope** (alcance) de una variable define la región del programa donde esa variable es accesible. Java tiene reglas estrictas de scope.

### 2.11.1 Block Scope (Alcance de Bloque)

Una variable declarada dentro de un bloque `{ }` solo existe dentro de ese bloque.

```java
public void metodo() {
    int x = 10;            // scope: desde aquí hasta el final del método

    if (x > 5) {
        int y = 20;        // scope: solo dentro de este bloque if
        System.out.println(x);   // OK, x es visible
        System.out.println(y);   // OK, y está en scope
    }

    System.out.println(x);    // OK
    // System.out.println(y); // Error de compilación: y fuera de scope
}
```

### 2.11.2 Method Scope (Alcance de Método)

Los parámetros de un método y las variables locales declaradas en su cuerpo existen desde su declaración hasta el final del método.

```java
public int sumar(int a, int b) {  // a y b: scope de parámetros
    int resultado = a + b;        // resultado: variable local, scope de método
    return resultado;
}
```

Los parámetros no pueden redefinirse dentro del método:

```java
public void repetir(int a) {
    // int a = 5;  // Error de compilación: a ya está definido como parámetro
}
```

### 2.11.3 Class Scope (Alcance de Clase)

Los campos (atributos) de una clase son accesibles desde cualquier método de la clase.

```java
public class Persona {
    private String nombre;       // scope de clase (campo de instancia)
    private static int contador; // scope de clase (campo estático)

    public void setNombre(String n) {
        nombre = n;   // accede al campo de clase
    }

    public String getNombre() {
        return nombre;
    }
}
```

### 2.11.4 Shadowing (Sombreamiento de Variables)

Ocurre cuando una variable en un scope interno tiene el mismo nombre que una variable en un scope externo. La variable interna "sombrea" a la externa.

```java
public class ShadowingDemo {
    private int valor = 100;       // campo de clase

    public void mostrar() {
        System.out.println(valor); // 100 — usa el campo de clase

        int valor = 200;           // variable local con mismo nombre → shadowing
        System.out.println(valor); // 200 — usa la variable local

        // Para acceder al campo sombreado, usa this
        System.out.println(this.valor); // 100 — accede explícitamente al campo
    }

    public void otroEjemplo(int valor) {  // el parámetro sombrea el campo
        System.out.println(valor);        // imprime el parámetro
        System.out.println(this.valor);   // imprime el campo (100)
    }
}
```

**Buena práctica:** Evita el shadowing. Usa nombres distintos para variables en distintos scopes, o califica con `this` cuando sea necesario.

---

## Ejercicios del Capítulo

1. Declara variables de cada uno de los 8 tipos primitivos, inicialízalas y muéstralas por consola.

2. Escribe un programa que demuestre por qué `0.1 + 0.2 != 0.3` en Java y explica la causa.

3. Escribe un programa que reciba dos números por teclado y muestre su suma, resta, multiplicación, división (con decimales) y módulo.

4. Demuestra el overflow de enteros: suma 1 a `Integer.MAX_VALUE` y muestra el resultado. Luego usa `Math.addExact()` y captura la excepción.

5. Compara el rendimiento de usar `Long` wrapper vs `long` primitivo en un bucle de 10 millones de iteraciones.

6. Escribe un programa que determine si un año es bisiesto usando operadores lógicos.

7. Convierte un `double` a `int` mediante casting y observa qué sucede con la parte decimal. Luego haz la conversión usando `Math.round()` y compara.

8. Demuestra la pérdida de precisión de `int` a `float` usando el valor `16777217`.

9. Crea un `switch` mejorado que reciba un número del 1 al 12 y devuelva el nombre del mes en español.

10. Escribe un `switch` como expresión con `yield` que reciba un mes y devuelva el número de días, considerando años bisiestos para febrero.

11. Escribe un bucle `for` que imprima los primeros 20 números de la serie de Fibonacci.

12. Compara la velocidad de recorrer una `LinkedList` de 50,000 elementos usando `for` indexado vs `for-each`. Explica la diferencia.

13. Crea un Text Block con una consulta SQL multilínea y formatea sus parámetros con `.formatted()`.

14. Escribe un método que reciba una cadena y devuelva `true` si es un palíndromo, ignorando mayúsculas y espacios.

15. Compara la velocidad de concatenar 50,000 caracteres usando `+` en un bucle versus usar `StringBuilder` (pre-dimensionado y sin pre-dimensionar).

16. Usa `Pattern` y `Matcher` para extraer todas las direcciones de correo electrónico de un texto.

17. Implementa una calculadora financiera con `BigDecimal` que calcule el precio final de un producto aplicando IVA y un descuento. Usa `RoundingMode.HALF_UP` para redondear a 2 decimales.

18. Compara el rendimiento de `Math.random()` vs `ThreadLocalRandom` generando 10 millones de números aleatorios desde 10 hilos simultáneos.

---

## Resumen del Capítulo

- Java tiene **8 tipos primitivos**: `byte`, `short`, `int`, `long`, `float`, `double`, `char`, `boolean`. Cada uno tiene tamaño fijo y valor por defecto.
- Los tipos de punto flotante (`float`, `double`) siguen el estándar **IEEE 754**, lo que implica que no pueden representar todos los decimales exactamente (ej. 0.1). Para dinero, usa **`BigDecimal`**.
- El **overflow** de enteros es silencioso en Java (no lanza excepción). Usa `Math.addExact()`, `Math.multiplyExact()` para detectarlo.
- El **auto-boxing/unboxing** convierte automáticamente entre primitivos y wrappers, pero tiene costo de rendimiento. Nunca compares wrappers con `==`; usa `.equals()`.
- Las **variables locales** viven en el stack. Los **objetos** viven en el heap. El **Escape Analysis** del JIT puede optimizar poniendo objetos en el stack.
- Los **operadores lógicos** `&&` y `||` usan corto-circuito para evitar evaluaciones innecesarias.
- **`instanceof`** verifica el tipo de un objeto. Desde Java 16, el **pattern matching** (`instanceof String s`) elimina el cast manual.
- La **precedencia** determina el orden de evaluación. Ante la duda, usa paréntesis.
- Las **conversiones implícitas** (widening) son automáticas. Las **explícitas** (narrowing/casting) pueden perder datos. Un `float` de 32 bits no puede representar todos los `int` de 32 bits.
- Las **estructuras condicionales** (`if-else`, `switch`) controlan el flujo. Java 14+ ofrece `switch` con flechas, `switch` como expresión con `yield`, y desde Java 21, pattern matching en `switch`.
- Las **estructuras repetitivas** incluyen `for`, `for-each`, `while`, `do-while`. Usa `for-each` siempre que puedas; es más limpio y el compilador lo optimiza. En `LinkedList`, `for` indexado es O(n²) — catastrófico.
- **`String`** es inmutable. El **String Pool** reutiliza literales. Desde Java 9, **Compact Strings** ahorran memoria usando `byte[]` para texto Latin-1. Los **Text Blocks** (Java 15+) simplifican strings multilínea.
- Para construir strings en bucles, usa **`StringBuilder`** (no sincronizado, rápido) o `StringBuffer` (sincronizado, para multi-hilo). El compilador usa `invokedynamic` y `StringConcatFactory` para optimizar `+` desde Java 9.
- Las **clases Wrapper** (`Integer`, `Double`, etc.) encapsulan primitivos y ofrecen métodos de conversión, comparación y constantes como `MAX_VALUE`.
- La clase **`Math`** ofrece funciones trigonométricas, potencias, redondeo y generación de aleatorios. Prefiere **`ThreadLocalRandom`** sobre `Math.random()` en entornos multi-hilo.
- **`BigDecimal`** es para aritmética decimal exacta. Créalo siempre desde `String`. Usa `compareTo()` para comparar, no `equals()`. Especifica `RoundingMode` al dividir.
- **`BigInteger`** ofrece enteros de precisión arbitraria para criptografía y matemáticas avanzadas.
- El **scope** determina dónde es visible una variable: bloque, método o clase. El shadowing ocurre cuando una variable interna oculta una externa.

---

← [Capítulo anterior](capitulo-01-introduccion.md) | [Inicio](README.md) | [Capítulo siguiente →](capitulo-03-poo.md)
