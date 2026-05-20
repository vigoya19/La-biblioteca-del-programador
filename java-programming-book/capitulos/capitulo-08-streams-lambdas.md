# Capítulo 8: Expresiones Lambda y la API de Streams

---

## 8.1 Expresiones Lambda

### 8.1.1 ¿Qué es una expresión lambda?

Una **expresión lambda** es una función anónima — un bloque de código que puede ser pasado y ejecutado posteriormente. Las lambdas permiten tratar el comportamiento como un valor, habilitando un estilo de programación más declarativo y conciso.

La evolución histórica en Java ilustra el problema que resuelven:

```java
// Java 1.0 — Clase externa
class FiltroPorPrecio implements Predicate<Producto> {
    public boolean test(Producto p) { return p.getPrecio() > 100; }
}
list.filter(new FiltroPorPrecio());

// Java 1.1 — Clase interna anónima
list.filter(new Predicate<Producto>() {
    @Override
    public boolean test(Producto p) {
        return p.getPrecio() > 100;
    }
});

// Java 8+ — Lambda
list.filter(p -> p.getPrecio() > 100);
```

La lambda elimina todo el "ceremonial": nombre de la clase, nombre del método, tipo de retorno explícito, modificador de acceso. Queda solo la lógica esencial.

Es importante entender que una lambda **no es una clase interna anónima disfrazada**. El compilador las traduce de forma distinta: mientras las clases anónimas generan un archivo `.class` separado, las lambdas usan `invokedynamic` (JSR 292) y se resuelven en tiempo de ejecución mediante `LambdaMetafactory`. Esto las hace más eficientes en memoria y tiempo de carga.

### 8.1.2 Sintaxis de expresiones lambda

La forma general es:

```
(parámetros) -> { cuerpo }
```

Reglas de sintaxis:

- **Múltiples parámetros**: `(a, b) -> a + b`
- **Un solo parámetro**: se pueden omitir los paréntesis: `x -> x * 2`
- **Sin parámetros**: se requieren paréntesis vacíos: `() -> System.out.println("Hola")`
- **Cuerpo con una sola expresión**: se omiten llaves y `return`: `(a, b) -> a + b`
- **Cuerpo con bloque**: se usan llaves y `return` explícito:

```java
(a, b) -> {
    int resultado = a + b;
    return resultado;
};
```

**Variantes adicionales de sintaxis:**

```java
// Con anotaciones de tipo (poco común pero posible)
(@NotNull String a, @NotNull String b) -> a.compareTo(b)

// Con var (Java 11+): inferencia con modificadores de parámetro
(var a, var b) -> a + b

// Lambda que lanza excepción verificada
(String s) -> {
    try {
        return new URI(s);
    } catch (URISyntaxException e) {
        throw new RuntimeException(e);
    }
};

// Lambda multilínea con lógica compleja
(pedido) -> {
    double subtotal = pedido.getItems().stream()
        .mapToDouble(Item::getPrecio).sum();
    double impuesto = subtotal * 0.21;
    return subtotal + impuesto;
};
```

### 8.1.3 Inferencia de tipos

Java infiere los tipos de los parámetros desde el contexto (la interfaz funcional objetivo):

```java
// Tipos declarados explícitamente
Comparator<String> c1 = (String a, String b) -> a.compareTo(b);

// Inferencia completa — lo más común
Comparator<String> c2 = (a, b) -> a.compareTo(b);

// Con una sola línea, sin llaves ni return
List<String> nombres = Arrays.asList("Ana", "Carlos", "Beatriz");
nombres.sort((a, b) -> a.compareToIgnoreCase(b));

// Inferencia en contexto de asignación múltiple
Function<String, Integer> extraerLongitud = s -> s.length();
Predicate<Integer> esPositivo = n -> n > 0;

// Inferencia a través de genéricos anidados
Map<String, Function<Integer, String>> transformadores = new HashMap<>();
transformadores.put("doble", n -> "Doble: " + (n * 2));
```

La inferencia solo funciona cuando el compilador puede determinar inequívocamente la interfaz funcional objetivo. Si hay ambigüedad, se requiere tipo explícito o un cast:

```java
// Ambigüedad — no compila
// var fn = x -> x.length(); // ¿Cuál interfaz funcional?

// Resuelto con cast
Function<String, Integer> fn = x -> x.length();

// Ambigüedad con sobrecarga de métodos
public void procesar(Function<String, Integer> fn) { }
public void procesar(ToIntFunction<String> fn) { }

// Esto no compila por ambigüedad:
// procesar(s -> s.length());

// Solución: especificar el tipo esperado
procesar((Function<String, Integer>) s -> s.length());
```

La inferencia de tipos usa **target typing**: el compilador determina el tipo destino a partir de:
1. El tipo de la variable a la que se asigna
2. El tipo del parámetro del método al que se pasa
3. El tipo de retorno esperado en un `return`
4. El tipo esperado en un operador ternario o cast

### 8.1.4 Variables effectively final

Una lambda puede acceder a variables locales del ámbito que la contiene, pero solo si son **effectively final** (no se reasignan después de su inicialización):

```java
// Correcto — la variable es effectively final
String prefijo = "Sr. ";
List<String> saludados = nombres.stream()
    .map(n -> prefijo + n)  // OK: prefijo no se modifica
    .collect(Collectors.toList());

// También correcto — inicializada una vez, nunca modificada
final int multiplicador = 10;
IntUnaryOperator multiplicar = n -> n * multiplicador;

// Incorrecto — variable modificada
int contador = 0;
// ERROR: contador no es effectively final
// nombres.forEach(n -> System.out.println(n + " " + (++contador)));
```

Esta restricción existe porque las lambdas no crean un nuevo alcance de variables — las capturan por valor desde el stack del método. Modificar la variable local después de la captura crearía condiciones de carrera sutiles en contextos multihilo.

**¿Cómo acumular valores en una lambda?** Si necesitas un contador o acumulador, existen alternativas:

```java
// Alternativa 1: array de un elemento (mutable, pero la referencia es effectively final)
int[] contador = {0};
nombres.forEach(n -> contador[0]++);

// Alternativa 2: AtomicInteger (thread-safe)
AtomicInteger contadorAtomic = new AtomicInteger(0);
nombres.forEach(n -> contadorAtomic.incrementAndGet());

// Alternativa 3: usar reduce/collect en lugar de forEach
long total = nombres.stream().count();
```

### 8.1.5 `this` en expresiones lambda

A diferencia de las clases internas anónimas, las lambdas **no** introducen un nuevo ámbito para `this`. Dentro de una lambda, `this` se refiere al objeto de la clase que la contiene, exactamente como en el método donde se declara:

```java
public class Procesador {
    private String nombre = "Procesador";

    public void ejecutar() {
        // Clase anónima: this -> la instancia anónima
        Runnable r1 = new Runnable() {
            private String nombre = "Anónimo";

            @Override
            public void run() {
                System.out.println(this.nombre); // "Anónimo"
            }
        };

        // Lambda: this -> la instancia de Procesador
        Runnable r2 = () -> System.out.println(this.nombre); // "Procesador"
    }
}
```

Esta diferencia es fundamental al migrar código de clases anónimas a lambdas: no puedes declarar campos en una lambda, ni usar `this` para referirte a la propia lambda.

---

## 8.2 Interfaces Funcionales

Una **interfaz funcional** es aquella que declara exactamente un método abstracto (SAM: Single Abstract Method). Las lambdas y las referencias a métodos proporcionan una implementación en línea de esa interfaz.

### 8.2.1 `@FunctionalInterface`

La anotación `@FunctionalInterface` es opcional pero recomendada: el compilador verificará que la interfaz cumpla el contrato SAM:

```java
@FunctionalInterface
public interface Validacion {
    boolean esValido(String valor);

    // Métodos default y static no cuentan para el SAM
    default void log(String valor) {
        System.out.println("Validando: " + valor);
    }

    static Validacion siempreValido() {
        return v -> true;
    }
}
```

**Reglas de SAM:** una interfaz funcional puede tener:
- Un método abstracto (obligatorio)
- Métodos `default` (ilimitados)
- Métodos `static` (ilimitados)
- Métodos públicos de `Object` (no cuentan): `toString()`, `equals()`, `hashCode()`

Si la interfaz extiende otra interfaz funcional con el mismo método abstracto, sigue siendo funcional:

```java
@FunctionalInterface
interface PredicadoExtendido extends Predicate<String> {
    // Hereda test(String), no añade nuevo método abstracto
    default PredicadoExtendido y(PredicadoExtendido otro) {
        return s -> test(s) && otro.test(s);
    }
}
```

### 8.2.2 El paquete `java.util.function`

Java 8 introdujo un rico conjunto de interfaces funcionales genéricas que cubren la mayoría de los casos de uso, evitando que el programador tenga que crear las suyas propias.

#### `Predicate<T>`: evaluación booleana

Representa una función que toma un valor y devuelve `boolean`.

```java
Predicate<String> esLargo = s -> s.length() > 5;
Predicate<String> empiezaConA = s -> s.startsWith("A");

System.out.println(esLargo.test("Arquitectura")); // true
System.out.println(empiezaConA.test("Arquitectura")); // true

// Combinación: and, or, negate
Predicate<String> largoYEmpiezaConA = esLargo.and(empiezaConA);
Predicate<String> noEmpiezaConA = empiezaConA.negate();
Predicate<String> largoOEmpiezaConA = esLargo.or(empiezaConA);

// Predicate.isEqual(): comparación con equals
Predicate<String> esJava = Predicate.isEqual("Java");
System.out.println(esJava.test("Java"));  // true
System.out.println(esJava.test("Kotlin")); // false

// Predicate.not() (Java 11+): negación más legible
Predicate<String> noEsVacio = Predicate.not(String::isEmpty);
```

**Comparación imperativo vs declarativo:**

```java
// Imperativo: lo que NO queremos
List<String> filtradas = new ArrayList<>();
for (String s : palabras) {
    if (s.length() > 5 && s.startsWith("A")) {
        filtradas.add(s);
    }
}

// Declarativo con Predicate
List<String> filtradas = palabras.stream()
    .filter(esLargo.and(empiezaConA))
    .collect(Collectors.toList());
```

#### `Function<T, R>`: transformación

Representa una función que acepta un argumento de tipo `T` y produce un resultado de tipo `R`.

```java
Function<String, Integer> longitud = s -> s.length();
Function<Integer, String> comoTexto = i -> "Longitud: " + i;

System.out.println(longitud.apply("Lambda"));   // 6

// Composición: andThen y compose
Function<String, Integer> doble = s -> s.length() * 2;
Function<Integer, String> etiqueta = i -> "Resultado=" + i;

Function<String, String> pipeline = doble.andThen(etiqueta);
System.out.println(pipeline.apply("Java"));     // Resultado=8

// compose: aplica el argumento primero, andThen lo aplica después
Function<Integer, Integer> triple = x -> x * 3;
Function<Integer, Integer> cuadrado = x -> x * x;

// compose: primero cuadrado, luego triple → (x²)*3
Function<Integer, Integer> cuadradoLuegoTriple = triple.compose(cuadrado);
System.out.println(cuadradoLuegoTriple.apply(5)); // 75 = (5²)*3

// andThen: primero triple, luego cuadrado → (x*3)²
Function<Integer, Integer> tripleLuegoCuadrado = triple.andThen(cuadrado);
System.out.println(tripleLuegoCuadrado.apply(5)); // 225 = (5*3)²

// Function.identity(): devuelve el mismo valor (útil en Collectors.toMap)
Map<Integer, String> mapa = nombres.stream()
    .collect(Collectors.toMap(String::length, Function.identity(), (a, b) -> a));
```

Especializaciones comunes:

```java
UnaryOperator<String> mayusculas = String::toUpperCase;   // Function<T, T>
IntFunction<String[]> crearArray = String[]::new;          // Toma int, produce R
LongFunction<BigDecimal> desdeLong = BigDecimal::new;
DoubleFunction<String> formatear = d -> String.format("%.2f", d);

// BiFunction: dos argumentos
BiFunction<String, Integer, String> repetir = (s, n) -> s.repeat(n);
System.out.println(repetir.apply("Ho", 3)); // "HoHoHo"
```

#### `Consumer<T>`: consumo

Representa una operación que acepta un argumento y no produce resultado (efecto secundario).

```java
Consumer<String> impresor = s -> System.out.println(" -> " + s);
Consumer<String> registrador = s -> logger.info("Procesando: " + s);

// Encadenamiento
Consumer<String> ambos = impresor.andThen(registrador);
palabras.forEach(ambos);
```

#### `Supplier<T>`: provisión (evaluación perezosa)

Representa un proveedor de resultados. No toma argumentos. Ideal para **evaluación perezosa** (lazy evaluation).

```java
Supplier<Double> aleatorio = Math::random;
System.out.println(aleatorio.get()); // 0.723591...

// Evaluación perezosa: el objeto costoso solo se crea si es necesario
public String obtenerValor(Supplier<String> proveedorCostoso) {
    return cache.getOrDefault("clave", proveedorCostoso.get());
}

// La lambda no se ejecuta si la clave existe en la caché
String valor = obtenerValor(() -> consultaCostosaABaseDeDatos());
```

Comparación con `orElse` vs `orElseGet` en Optional (mismo principio):

```java
// orElse: expansivo — siempre evalúa
String resultado = optional.orElse(consultaCostosa()); // Siempre ejecuta

// orElseGet: perezoso — solo evalúa si es necesario
String resultado = optional.orElseGet(() -> consultaCostosa()); // Ejecuta solo si empty
```

#### `BiFunction<T, U, R>`, `BiConsumer<T, U>`, `BiPredicate<T, U>`

Versiones con dos parámetros de las interfaces anteriores:

```java
// BiFunction: recibe dos argumentos, produce un resultado
BiFunction<Integer, Integer, String> sumaComoTexto = (a, b) -> String.valueOf(a + b);
System.out.println(sumaComoTexto.apply(3, 7)); // "10"

// BiConsumer: recibe dos argumentos, no produce resultado
Map<String, Integer> mapa = new HashMap<>();
BiConsumer<String, Integer> insertar = mapa::put;
insertar.accept("Java", 23);

// BiPredicate: recibe dos argumentos, devuelve boolean
BiPredicate<String, Integer> longitudMayorQue = (s, n) -> s.length() > n;
System.out.println(longitudMayorQue.test("Arquitectura", 5)); // true
```

### 8.2.3 Especializaciones de tipos primitivos

Las interfaces genéricas de `java.util.function` operan con referencias (tipos envoltorios). Para evitar el costo de autoboxing/unboxing, existen especializaciones para `int`, `long` y `double`:

```java
// Sin especialización (boxing) — ineficiente para grandes volúmenes
Function<Integer, Integer> doblar = x -> x * 2;       // autoboxing en cada invocación

// Con especialización (sin boxing)
IntUnaryOperator doblarInt = x -> x * 2;               // opera con int nativos
System.out.println(doblarInt.applyAsInt(7));           // 14

// Otras especializaciones útiles
IntPredicate esPar = n -> n % 2 == 0;
LongFunction<String> formatearFecha = milis ->
    new SimpleDateFormat("yyyy-MM-dd").format(new Date(milis));
DoubleConsumer imprimir = System.out::println;
ToIntFunction<String> extraerLongitud = String::length;
```

**El costo del autoboxing:** un `Integer` ocupa ~16 bytes en heap (cabecera + valor + padding), mientras que un `int` ocupa 4 bytes en stack. Para 10 millones de operaciones, el boxing genera ~160 MB adicionales de objetos temporales que el GC debe recolectar. Siempre prefiere las especializaciones primitivas en streams numéricos.

Tabla resumen de especializaciones principales:

| Genérica | Int | Long | Double |
|----------|-----|------|--------|
| `Predicate<T>` | `IntPredicate` | `LongPredicate` | `DoublePredicate` |
| `Function<T,R>` | `IntFunction<R>` / `ToIntFunction<T>` | `LongFunction<R>` / `ToLongFunction<T>` | `DoubleFunction<R>` / `ToDoubleFunction<T>` |
| `Consumer<T>` | `IntConsumer` | `LongConsumer` | `DoubleConsumer` |
| `Supplier<T>` | `IntSupplier` | `LongSupplier` | `DoubleSupplier` |
| `UnaryOperator<T>` | `IntUnaryOperator` | `LongUnaryOperator` | `DoubleUnaryOperator` |
| `BinaryOperator<T>` | `IntBinaryOperator` | `LongBinaryOperator` | `DoubleBinaryOperator` |

### 8.2.4 Combinando interfaces funcionales

Una de las mayores fortalezas es la composición:

```java
// Pipeline de validación compuesto con Predicates
Predicate<String> noVacio = Predicate.not(String::isEmpty);
Predicate<String> maxLongitud = s -> s.length() <= 100;
Predicate<String> sinEspacios = s -> !s.contains(" ");
Predicate<String> validador = noVacio.and(maxLongitud).and(sinEspacios);

// Pipeline de transformación con Functions
Function<String, String> trim = String::trim;
Function<String, String> normalizar = s -> s.replaceAll("\\s+", " ");
Function<String, String> capitalizar = s ->
    s.substring(0, 1).toUpperCase() + s.substring(1).toLowerCase();
Function<String, String> limpiar = trim.andThen(normalizar).andThen(capitalizar);

System.out.println(limpiar.apply("  jAvA   eS   GeNiAL  ")); // "Java es genial"
```

---

## 8.3 Referencias a Métodos

Las referencias a métodos son una sintaxis más compacta para lambdas cuando la lambda simplemente llama a un método existente. Existen cuatro formas:

### 8.3.1 Referencia a método estático: `Clase::metodoEstatico`

```java
// Lambda
Function<String, Integer> f1 = s -> Integer.parseInt(s);

// Referencia a método
Function<String, Integer> f2 = Integer::parseInt;
```

```java
// Aplicación práctica
List<String> numerosComoTexto = Arrays.asList("1", "2", "3", "4");
List<Integer> numeros = numerosComoTexto.stream()
    .map(Integer::parseInt)
    .collect(Collectors.toList());

// Otros ejemplos comunes
Math::max          // (int,int) -> int
Collections::sort  // (List) -> void
Objects::isNull    // (Object) -> boolean
Double::valueOf    // (String) -> Double
```

### 8.3.2 Referencia a método de instancia (objeto específico): `instancia::metodoInstancia`

```java
// Lambda
String prefijo = "Producto: ";
Function<Producto, String> f1 = p -> prefijo.concat(p.getNombre());

// Referencia a método de instancia específica
Function<Producto, String> f2 = prefijo::concat;
```

```java
// Con System.out — caso muy común
Consumer<String> impresor = System.out::println;
nombres.forEach(System.out::println);

// Con una instancia personalizada
StringBuilder sb = new StringBuilder();
Consumer<String> acumulador = sb::append;
nombres.forEach(acumulador);
System.out.println(sb.toString());
```

### 8.3.3 Referencia a método de instancia (clase, no ligado): `Clase::metodoInstancia`

El primer parámetro de la lambda se convierte en el receptor del método.

```java
// Lambda: el primer argumento (s) es el receptor de toUpperCase()
Function<String, String> f1 = s -> s.toUpperCase();

// Referencia a método no ligado
Function<String, String> f2 = String::toUpperCase;
```

Esto funciona con dos parámetros también:

```java
// Lambda: (a, b) -> a.compareToIgnoreCase(b)
Comparator<String> c1 = (a, b) -> a.compareToIgnoreCase(b);

// Referencia a método no ligado
Comparator<String> c2 = String::compareToIgnoreCase;

// Más ejemplos de referencias no ligadas
BiFunction<String, String, Boolean> mismoPrefijo = String::startsWith;
BiConsumer<List<String>, String> agregar = List::add;  // ¡cuidado con listas inmutables!
```

### 8.3.4 Referencia a constructor: `Clase::new`

```java
// Lambda
Supplier<Producto> f1 = () -> new Producto();
Function<String, Producto> f2 = nombre -> new Producto(nombre);
BiFunction<String, Double, Producto> f3 = (nombre, precio) ->
    new Producto(nombre, precio);

// Referencia a constructor
Supplier<Producto> f4 = Producto::new;
Function<String, Producto> f5 = Producto::new;
BiFunction<String, Double, Producto> f6 = Producto::new;
```

El compilador selecciona el constructor apropiado según la interfaz funcional y el número de parámetros.

**Aplicación con streams:**

```java
List<Producto> productos = nombres.stream()
    .map(Producto::new)
    .collect(Collectors.toList());

// Construir arrays
IntFunction<int[]> crearArray = int[]::new;
int[] array = crearArray.apply(100); // new int[100]
```

### 8.3.5 Regla para elegir

Usa referencia a método **siempre que puedas**. La regla: si la lambda invoca un solo método y los parámetros coinciden en orden y tipo, puedes convertirla en referencia a método:

```java
// Buena candidata: un solo método, parámetros coinciden
lista.forEach(e -> System.out.println(e));   // Lambda
lista.forEach(System.out::println);           // Referencia

// Mala candidata: lambda con operaciones adicionales
lista.forEach(e -> {
    contador++;
    System.out.println(e);
}); // No se puede convertir a referencia a método

// Más ejemplos de buen uso de referencias a métodos
lista.stream().map(String::toUpperCase)                 // en lugar de s -> s.toUpperCase()
lista.stream().filter(Objects::nonNull)                 // en lugar de o -> o != null
lista.stream().mapToInt(Integer::parseInt)              // en lugar de s -> Integer.parseInt(s)
lista.stream().sorted(String::compareToIgnoreCase)      // en lugar de (a,b) -> a.compareToIgnoreCase(b)
```

---

## 8.4 La API de Streams

### 8.4.1 ¿Qué es un Stream?

Un **Stream** en Java **no es una estructura de datos**. Es una **secuencia de elementos que soporta operaciones secuenciales y paralelas para procesar datos**. Es un pipeline declarativo.

Características fundamentales:

| Característica | Explicación |
|---|---|
| **Declarativo** | Describes *qué* quieres, no *cómo* hacerlo |
| **Perezoso (lazy)** | Las operaciones intermedias no se ejecutan hasta que se invoca una operación terminal |
| **Potencialmente infinito** | `Stream.generate()` y `Stream.iterate()` pueden producir streams sin fin |
| **Consumible una sola vez** | Una vez que se aplica una operación terminal, el stream no puede reutilizarse |
| **Paralelizable** | Con `.parallel()` se divide el trabajo en múltiples hilos |
| **No modifica la fuente** | Las operaciones no alteran la colección o fuente original |
| **Sin estado (idealmente)** | Cada elemento se procesa de forma independiente |

**Comparación imperativo vs declarativo:**

```java
// Imperativo — CÓMO
List<Producto> caros = new ArrayList<>();
for (Producto p : productos) {
    if (p.getPrecio() > 500) {
        caros.add(p);
    }
}
Collections.sort(caros, new Comparator<Producto>() {
    public int compare(Producto a, Producto b) {
        return a.getNombre().compareTo(b.getNombre());
    }
});
List<String> nombresCaros = new ArrayList<>();
for (Producto p : caros) {
    nombresCaros.add(p.getNombre().toUpperCase());
}

// Declarativo con Streams — QUÉ
List<String> nombresCaros = productos.stream()
    .filter(p -> p.getPrecio() > 500)
    .sorted(Comparator.comparing(Producto::getNombre))
    .map(p -> p.getNombre().toUpperCase())
    .collect(Collectors.toList());
```

### 8.4.2 Fuentes de Streams

```java
// Desde una colección
Stream<Producto> s1 = productos.stream();
Stream<Producto> s2 = productos.parallelStream();

// Desde un array
Stream<String> s3 = Arrays.stream(arrayDeStrings);
Stream<String> s4 = Stream.of("a", "b", "c");

// Stream infinito — con seed
Stream<Integer> s5 = Stream.iterate(0, n -> n + 2);   // 0, 2, 4, 6, ...
// Java 9+: iterate con condición (hasNext)
Stream<Integer> s6 = Stream.iterate(0, n -> n < 100, n -> n + 2);

// Stream infinito — con Supplier
Stream<Double> s7 = Stream.generate(Math::random);

// Streams primitivos
IntStream s8 = IntStream.range(0, 100);               // 0..99 (exclusivo)
IntStream s9 = IntStream.rangeClosed(1, 100);          // 1..100 (inclusivo)
LongStream s10 = LongStream.of(1L, 2L, 3L);
DoubleStream s11 = DoubleStream.generate(Math::random);

// Desde archivo
try (Stream<String> lineas = Files.lines(Path.of("archivo.txt"))) {
    lineas.filter(l -> !l.isBlank()).forEach(System.out::println);
}

// Desde String (caracteres como int)
IntStream codepoints = "Java".codePoints();

// Concatenación
Stream<String> combinado = Stream.concat(s3, s4);

// Builder
Stream<String> builder = Stream.<String>builder()
    .add("uno").add("dos").add("tres").build();

// Stream vacío (útil como retorno de métodos)
Stream<String> vacio = Stream.empty();

// Desde una expresión regular (Java 9+)
Pattern.compile(" ")
    .splitAsStream("uno dos tres")
    .forEach(System.out::println);
```

### 8.4.3 Streams primitivos

`IntStream`, `LongStream` y `DoubleStream` evitan el boxing y ofrecen métodos especializados:

```java
// Métodos estadísticos solo en streams primitivos
int suma = IntStream.rangeClosed(1, 100).sum();
OptionalDouble promedio = IntStream.rangeClosed(1, 100).average();
OptionalInt maximo = IntStream.of(3, 1, 4, 1, 5, 9, 2).max();
OptionalInt minimo = IntStream.of(3, 1, 4, 1, 5, 9, 2).min();

// SummaryStatistics: todas las estadísticas en una pasada
IntSummaryStatistics stats = IntStream.rangeClosed(1, 100).summaryStatistics();
System.out.println("Count: " + stats.getCount());
System.out.println("Sum: " + stats.getSum());
System.out.println("Min: " + stats.getMin());
System.out.println("Max: " + stats.getMax());
System.out.println("Average: " + stats.getAverage());

// Conversión entre streams
IntStream intStream = productos.stream().mapToInt(Producto::getStock);
Stream<Integer> boxed = intStream.boxed();             // IntStream -> Stream<Integer>
LongStream longs = boxed.mapToLong(Integer::longValue);
DoubleStream doubles = intStream.mapToDouble(i -> i * 0.5);

// mapToObj: de primitivo a referencias
Stream<String> textos = IntStream.range(0, 10)
    .mapToObj(i -> "Número: " + i);
```

---

## 8.5 Operaciones Intermedias (Lazy)

Las operaciones intermedias **no se ejecutan inmediatamente**. Retornan un nuevo Stream y solo se procesan cuando una operación terminal es invocada. Esto permite al stream optimizar el pipeline completo.

### 8.5.1 `filter` — filtrado

```java
List<Producto> disponibles = productos.stream()
    .filter(p -> p.getStock() > 0)
    .collect(Collectors.toList());

// Múltiples filtros encadenados
List<Producto> resultado = productos.stream()
    .filter(p -> p.getStock() > 0)
    .filter(p -> p.getPrecio() > 10)
    .filter(p -> p.getCategoria().equals("Electrónica"))
    .collect(Collectors.toList());
```

### 8.5.2 `map` — transformación uno a uno

Transforma cada elemento en otro (de igual o diferente tipo):

```java
// Extraer nombres
List<String> nombres = productos.stream()
    .map(Producto::getNombre)
    .collect(Collectors.toList());

// Transformar tipo
List<Integer> longitudes = palabras.stream()
    .map(String::length)
    .collect(Collectors.toList());

// Multiples map encadenados
List<String> resultado = productos.stream()
    .map(Producto::getNombre)
    .map(String::toUpperCase)
    .map(s -> "[" + s + "]")
    .collect(Collectors.toList());
```

### 8.5.3 `flatMap` — aplanamiento (uno a muchos)

Transforma cada elemento en un Stream y luego **aplana** todos esos streams en uno solo.

**Diferencia visual entre `map` y `flatMap`:**

```
Entrada:  ["Hola", "Mundo"]

map(p -> p.split(""))
  → Stream<String[]>  →  [["H","o","l","a"], ["M","u","n","d","o"]]
  Dos arrays, no es útil

flatMap(p -> Arrays.stream(p.split("")))
  → Stream<String>    →  ["H","o","l","a","M","u","n","d","o"]
  Un solo stream plano con todos los caracteres
```

Ejemplo práctico:

```java
// Obtener todos los pedidos de todos los clientes
List<Pedido> todosLosPedidos = clientes.stream()
    .flatMap(cliente -> cliente.getPedidos().stream())
    .collect(Collectors.toList());

// Con referencias a métodos
List<Pedido> todosLosPedidos = clientes.stream()
    .map(Cliente::getPedidos)       // Stream<List<Pedido>>
    .flatMap(List::stream)           // Stream<Pedido>
    .collect(Collectors.toList());
```

**Comparación imperativo vs declarativo:**

```java
// Imperativo — bucles anidados
List<Pedido> todos = new ArrayList<>();
for (Cliente c : clientes) {
    for (Pedido p : c.getPedidos()) {
        if (p.getEstado() == Estado.PENDIENTE) {
            todos.add(p);
        }
    }
}

// Declarativo
List<Pedido> todos = clientes.stream()
    .flatMap(c -> c.getPedidos().stream())
    .filter(p -> p.getEstado() == Estado.PENDIENTE)
    .collect(Collectors.toList());
```

### 8.5.4 `distinct` — elementos únicos

```java
List<String> categoriasUnicas = productos.stream()
    .map(Producto::getCategoria)
    .distinct()
    .collect(Collectors.toList());

// distinct() usa equals() para determinar unicidad
List<Producto> productosUnicos = productos.stream()
    .distinct()
    .collect(Collectors.toList());
```

**Atención:** `distinct()` es una operación **stateful** — necesita mantener un `HashSet` interno de elementos ya vistos. En streams paralelos, cada hilo mantiene su propio conjunto y se combinan al final, lo que tiene un costo de memoria adicional.

### 8.5.5 `sorted` — ordenamiento

```java
// Orden natural (requiere que T implemente Comparable)
productos.stream()
    .sorted()
    .forEach(System.out::println);

// Con Comparator
productos.stream()
    .sorted(Comparator.comparing(Producto::getPrecio).reversed())
    .forEach(System.out::println);

// Ordenamiento encadenado
productos.stream()
    .sorted(Comparator.comparing(Producto::getCategoria)
                      .thenComparing(Producto::getPrecio))
    .collect(Collectors.toList());

// Comparator con null handling
productos.stream()
    .sorted(Comparator.nullsLast(Comparator.comparing(Producto::getNombre)))
    .collect(Collectors.toList());
```

`sorted()` es también una operación **stateful**: necesita ver todos los elementos antes de producir el primer resultado. No puede emitir ningún elemento hasta haber visto toda la entrada.

### 8.5.6 `peek` — inspección (debugging)

```java
productos.stream()
    .filter(p -> p.getPrecio() > 100)
    .peek(p -> System.out.println("Después de filter: " + p))
    .map(Producto::getNombre)
    .peek(n -> System.out.println("Después de map: " + n))
    .collect(Collectors.toList());
```

**Importante**: `peek` está diseñado para depuración. No usarlo para lógica de negocio — su ejecución no está garantizada con streams paralelos y `findFirst()` puede omitir elementos. Si una operación de cortocircuito termina antes de procesar todos los elementos, `peek` solo se ejecutará para los elementos realmente procesados.

### 8.5.7 `limit` y `skip` — paginación

```java
// Top 5 más caros
productos.stream()
    .sorted(Comparator.comparing(Producto::getPrecio).reversed())
    .limit(5)
    .forEach(System.out::println);

// Omitir los primeros 3
productos.stream()
    .skip(3)
    .forEach(System.out::println);

// Paginación (página 2, tamaño 10)
int pagina = 2;
int tamano = 10;
productos.stream()
    .skip((pagina - 1) * tamano)
    .limit(tamano)
    .collect(Collectors.toList());

// Optimización: limit antes de sorted (reduce elementos a ordenar)
productos.stream()
    .limit(10)        // solo los primeros 10
    .sorted()         // ordenar 10 elementos es barato
    .collect(Collectors.toList());
```

### 8.5.8 `takeWhile` y `dropWhile` (Java 9+)

```java
// takeWhile: toma elementos mientras el predicado sea verdadero,
// luego descarta el resto (corta en la primera falsedad)
List<Integer> numeros = Arrays.asList(1, 2, 3, 5, 7, 8, 9, 2, 1);
List<Integer> menoresDeSeis = numeros.stream()
    .takeWhile(n -> n < 6)     // -> [1, 2, 3, 5]
    .collect(Collectors.toList());

// dropWhile: descarta elementos mientras el predicado sea verdadero,
// luego conserva el resto
List<Integer> desdeSeis = numeros.stream()
    .dropWhile(n -> n < 6)     // -> [7, 8, 9, 2, 1]
    .collect(Collectors.toList());
```

Diferencia clave frente a `filter`: `takeWhile`/`dropWhile` dependen del **orden** y dejan de evaluar en la primera falsedad. `filter` evalúa todos los elementos.

```java
// Ejemplo práctico: procesar registros hasta encontrar uno inválido
List<Transaccion> transacciones = repo.findAllOrderedByFecha();
List<Transaccion> validas = transacciones.stream()
    .takeWhile(Transaccion::esValida)
    .collect(Collectors.toList());
// Procesa solo hasta la primera transacción inválida
```

---

## 8.6 Operaciones Terminales

Las operaciones terminales **activan la ejecución del pipeline** y producen un resultado o un efecto secundario. Después de una operación terminal, **el stream se consume y no puede reutilizarse**.

```java
Stream<String> stream = nombres.stream();
long count = stream.count();
// ERROR: stream already consumed
// stream.forEach(System.out::println);
```

### 8.6.1 `forEach` — iteración

```java
productos.stream()
    .filter(p -> p.getPrecio() > 100)
    .forEach(System.out::println);
```

Nota: `forEach` en streams está diseñado para efectos secundarios. Para operaciones que requieran orden garantizado en paralelo, usar `forEachOrdered`.

### 8.6.2 `collect` — recolección

La operación más versátil. Convierte el stream en una colección, mapa, string o cualquier estructura mediante `Collectors`:

```java
List<String> nombres = productos.stream()
    .map(Producto::getNombre)
    .collect(Collectors.toList());

Set<String> categorias = productos.stream()
    .map(Producto::getCategoria)
    .collect(Collectors.toSet());
```

### 8.6.3 `toList` (Java 16+)

Atajo más conciso:

```java
List<String> nombres = productos.stream()
    .map(Producto::getNombre)
    .toList(); // Lista inmutable, más conciso que collect(Collectors.toList())
```

**Diferencia**: `Collectors.toList()` produce una lista modificable; `Stream.toList()` produce una lista **inmutable**. Si necesitas modificar la lista después, usa `collect(Collectors.toList())` o `collect(Collectors.toCollection(ArrayList::new))`.

### 8.6.4 `reduce` — reducción

Combina los elementos del stream en un solo resultado mediante una función asociativa:

```java
// Suma de precios (con valor inicial)
double total = productos.stream()
    .map(Producto::getPrecio)
    .reduce(0.0, (acum, precio) -> acum + precio);

// Con referencia a método
double total = productos.stream()
    .map(Producto::getPrecio)
    .reduce(0.0, Double::sum);

// Sin valor inicial (devuelve Optional)
Optional<Producto> masCaro = productos.stream()
    .reduce((a, b) -> a.getPrecio() > b.getPrecio() ? a : b);

// Reducción con tres argumentos: identidad, acumulador, combinador
// (el combinador se usa en streams paralelos)
int sumaDeLongitudes = palabras.parallelStream()
    .reduce(0,
            (acum, s) -> acum + s.length(),    // acumulador por hilo
            (a, b) -> a + b);                   // combinador de resultados

// IMPORTANTE: la identidad debe ser realmente un valor neutro
// 0 para suma, 1 para multiplicación, "" para concatenación, etc.
// Si no se cumple: identidad + elemento == elemento
// los resultados en paralelo serán incorrectos.

// Ejemplo de identidad correcta para concatenación:
String concatenado = palabras.parallelStream()
    .reduce("",
            (a, b) -> a + b,
            (a, b) -> a + b);

// Equivalente a: palabras.collect(Collectors.joining())
```

### 8.6.5 Búsqueda y coincidencia

```java
// count — total de elementos
long cantidad = productos.stream().count();

// min y max — requieren Comparator
Optional<Producto> masBarato = productos.stream()
    .min(Comparator.comparing(Producto::getPrecio));
Optional<Producto> masCaro = productos.stream()
    .max(Comparator.comparing(Producto::getPrecio));

// findFirst — primer elemento (útil con streams ordenados y filtrados)
Optional<Producto> primero = productos.stream()
    .filter(p -> p.getStock() > 0)
    .findFirst();

// findAny — cualquier elemento (permite optimización en paralelo)
Optional<Producto> cualquiera = productos.parallelStream()
    .filter(p -> p.getStock() > 0)
    .findAny();

// anyMatch — ¿alguno cumple?
boolean hayAgotados = productos.stream()
    .anyMatch(p -> p.getStock() == 0);

// allMatch — ¿todos cumplen?
boolean todosDisponibles = productos.stream()
    .allMatch(p -> p.getStock() > 0);

// noneMatch — ¿ninguno cumple?
boolean ningunoCaducado = productos.stream()
    .noneMatch(p -> p.getFechaCaducidad().isBefore(LocalDate.now()));

// Ejemplo combinado: encontrar el primer producto caro disponible
Optional<Producto> primerCaroDisponible = productos.stream()
    .filter(p -> p.getPrecio() > 500)
    .filter(p -> p.getStock() > 0)
    .findFirst();
```

**Comparaciones imperativo vs declarativo:**

```java
// Imperativo: ¿existe algún producto con precio > 1000?
boolean existe = false;
for (Producto p : productos) {
    if (p.getPrecio() > 1000) {
        existe = true;
        break; // necesario para no seguir iterando
    }
}

// Declarativo: más legible y el stream es cortocircuitable
boolean existe = productos.stream()
    .anyMatch(p -> p.getPrecio() > 1000);

// Imperativo: encontrar el primer producto en stock
Producto primero = null;
for (Producto p : productos) {
    if (p.getStock() > 0) {
        primero = p;
        break;
    }
}

// Declarativo
Optional<Producto> primero = productos.stream()
    .filter(p -> p.getStock() > 0)
    .findFirst();
```

### 8.6.6 `toArray` — conversión a array

```java
// Array de Object (no recomendado — pierde tipo)
Object[] array1 = productos.stream().toArray();

// Array tipado con referencia a constructor
Producto[] array2 = productos.stream().toArray(Producto[]::new);

// Para streams primitivos: toArray() devuelve array primitivo directamente
int[] array3 = IntStream.range(0, 100).toArray();
```

---

## 8.7 Optional

### 8.7.1 El problema de null

```java
// Pesadilla de null checks anidados
public String getNombreCiudad(Cliente c) {
    if (c != null) {
        Direccion d = c.getDireccion();
        if (d != null) {
            Ciudad ciudad = d.getCiudad();
            if (ciudad != null) {
                return ciudad.getNombre();
            }
        }
    }
    return "Desconocida";
}
```

`Optional<T>` es un contenedor que puede o no contener un valor no-nulo, forzando al programador a manejar explícitamente la ausencia.

### 8.7.2 Creación de Optionals

```java
// Optional con valor (si es null, lanza NullPointerException)
Optional<String> presente = Optional.of("Java");

// Optional que acepta null
Optional<String> quizaVacio = Optional.ofNullable(valorQuePodriaSerNull);

// Optional vacío
Optional<String> vacio = Optional.empty();
```

### 8.7.3 Extracción y manejo del valor

```java
// isPresent + get — evítalo, es igual que null check
if (opt.isPresent()) {
    System.out.println(opt.get());
}

// isEmpty (Java 11+) — lo opuesto a isPresent
if (opt.isEmpty()) {
    System.out.println("No hay valor");
}

// Mejor: ifPresent — ejecuta solo si hay valor
opt.ifPresent(valor -> System.out.println("Encontrado: " + valor));
opt.ifPresent(System.out::println);

// orElse — valor por defecto (se evalúa siempre, incluso si el Optional tiene valor)
String nombre = opt.orElse("Desconocido");

// orElseGet — valor por defecto perezoso (solo se evalúa si el Optional está vacío)
String nombre = opt.orElseGet(() -> consultaCostosa());

// orElseThrow — lanza excepción si está vacío
String nombre = opt.orElseThrow(() -> new NoSuchElementException("No encontrado"));

// Java 10+: orElseThrow sin argumento lanza NoSuchElementException
String nombre = opt.orElseThrow();
```

### 8.7.4 Transformaciones sobre Optional

```java
// map: transforma el valor si existe
Optional<String> nombreMayus = optProducto
    .map(Producto::getNombre)
    .map(String::toUpperCase);

// flatMap: evita Optional<Optional<T>> anidados
public Optional<Seguro> getSeguro(Optional<Coche> coche) {
    return coche.flatMap(Coche::getSeguro); // getSeguro ya devuelve Optional<Seguro>
}

// filter: Optional vacío si falla el predicado
Optional<Producto> caroYDisponible = optProducto
    .filter(p -> p.getPrecio() > 500)
    .filter(p -> p.getStock() > 0);
```

### 8.7.5 Optional como valor de retorno (no como campo)

```java
// CORRECTO: retorno de método
public Optional<Producto> buscarPorId(String id) {
    return Optional.ofNullable(repositorio.findById(id));
}

// INCORRECTO: Optional como campo de clase
public class Producto {
    // NO: Optional no fue diseñado para ser campo
    // private Optional<String> descripcion;

    // CORRECTO: campo normal, retornar Optional en el getter
    private String descripcion;

    public Optional<String> getDescripcion() {
        return Optional.ofNullable(descripcion);
    }
}
```

**Encadenamiento elegante** — reescribiendo el ejemplo del inicio con `Optional.map`/`flatMap`:

```java
public String getNombreCiudad(Cliente c) {
    return Optional.ofNullable(c)
        .map(Cliente::getDireccion)
        .map(Direccion::getCiudad)
        .map(Ciudad::getNombre)
        .orElse("Desconocida");
}
```

### 8.7.6 Operaciones de cortocircuito

`findFirst`, `findAny`, `min`, `max` y `reduce` (sin valor inicial) devuelven `Optional`:

```java
OptionalDouble promedio = IntStream.rangeClosed(1, 100).average();
promedio.ifPresent(System.out::println); // 50.5

OptionalInt maximo = IntStream.rangeClosed(1, 100).max();
maximo.ifPresentOrElse(
    n -> System.out.println("Máximo: " + n),
    () -> System.out.println("Stream vacío")
);
```

---

## 8.8 Collectors

`Collectors` es una clase de utilidad con implementaciones predefinidas de la interfaz `Collector`. Permiten transformar streams en colecciones, mapas, strings, y realizar agregaciones complejas.

### 8.8.1 Colecciones básicas

```java
List<String> lista = stream.collect(Collectors.toList());
Set<String> conjunto = stream.collect(Collectors.toSet());

// Colecciones específicas
ArrayList<String> arrayList = stream.collect(Collectors.toCollection(ArrayList::new));
TreeSet<String> treeSet = stream.collect(Collectors.toCollection(TreeSet::new));
ConcurrentLinkedDeque<String> concurrente = stream.collect(
    Collectors.toCollection(ConcurrentLinkedDeque::new));
```

### 8.8.2 `toMap` — construir Mapas

```java
// Mapa simple: clave = id, valor = objeto
Map<String, Producto> mapaPorId = productos.stream()
    .collect(Collectors.toMap(
        Producto::getId,     // key mapper
        p -> p               // value mapper
    ));

// Mapa con función de resolución de conflictos (claves duplicadas)
Map<String, Producto> ultimoPorCategoria = productos.stream()
    .collect(Collectors.toMap(
        Producto::getCategoria,
        Function.identity(),                // valor = el propio producto
        (existente, nuevo) -> nuevo         // ante conflicto, conservar el nuevo
    ));

// Mapa con tipo de implementación específica
Map<String, Producto> linkedMap = productos.stream()
    .collect(Collectors.toMap(
        Producto::getId,
        Function.identity(),
        (a, b) -> a,
        LinkedHashMap::new
    ));

// toUnmodifiableMap (Java 10+): mapa inmutable
Map<String, Producto> inmutable = productos.stream()
    .collect(Collectors.toUnmodifiableMap(
        Producto::getId, Function.identity()
    ));
```

### 8.8.3 Agregaciones numéricas

```java
// Conteo
long total = productos.stream().collect(Collectors.counting());

// Suma
int stockTotal = productos.stream()
    .collect(Collectors.summingInt(Producto::getStock));
double precioTotal = productos.stream()
    .collect(Collectors.summingDouble(Producto::getPrecio));

// Promedio
double precioPromedio = productos.stream()
    .collect(Collectors.averagingDouble(Producto::getPrecio));

// Estadísticas completas
IntSummaryStatistics stats = productos.stream()
    .collect(Collectors.summarizingInt(Producto::getStock));
System.out.println("Count: " + stats.getCount());
System.out.println("Sum: " + stats.getSum());
System.out.println("Min: " + stats.getMin());
System.out.println("Max: " + stats.getMax());
System.out.println("Avg: " + stats.getAverage());
```

### 8.8.4 `joining` — concatenación de strings

```java
String concatenado = productos.stream()
    .map(Producto::getNombre)
    .collect(Collectors.joining(", "));

// Con prefijo y sufijo
String json = productos.stream()
    .map(p -> "\"" + p.getNombre() + "\"")
    .collect(Collectors.joining(", ", "[", "]"));
// Resultado: ["Manzana", "Pera", "Uva"]
```

### 8.8.5 `maxBy` y `minBy`

```java
Optional<Producto> masCaro = productos.stream()
    .collect(Collectors.maxBy(Comparator.comparing(Producto::getPrecio)));
```

### 8.8.6 `groupingBy` — agrupación

La operación más poderosa de `Collectors`. Equivale a `GROUP BY` en SQL.

**Agrupación simple (clave → lista de valores):**

```java
// Agrupar productos por categoría
Map<String, List<Producto>> porCategoria = productos.stream()
    .collect(Collectors.groupingBy(Producto::getCategoria));

// Resultado:
// {"Electrónica": [laptop, tablet],
//  "Alimentos":  [manzana, pan],
//  "Ropa":       [camisa, zapatos]}
```

**Agrupación con downstream collector:**

```java
// Contar productos por categoría
Map<String, Long> conteoPorCategoria = productos.stream()
    .collect(Collectors.groupingBy(
        Producto::getCategoria,
        Collectors.counting()
    ));
// Resultado: {"Electrónica": 2, "Alimentos": 2, "Ropa": 2}

// Suma de stock por categoría
Map<String, Integer> stockPorCategoria = productos.stream()
    .collect(Collectors.groupingBy(
        Producto::getCategoria,
        Collectors.summingInt(Producto::getStock)
    ));

// Producto más caro de cada categoría
Map<String, Optional<Producto>> masCaroPorCategoria = productos.stream()
    .collect(Collectors.groupingBy(
        Producto::getCategoria,
        Collectors.maxBy(Comparator.comparing(Producto::getPrecio))
    ));

// Mapear dentro del grupo: solo nombres, ordenados
Map<String, List<String>> nombresPorCategoria = productos.stream()
    .collect(Collectors.groupingBy(
        Producto::getCategoria,
        Collectors.mapping(Producto::getNombre, Collectors.toList())
    ));
```

**Agrupación con map supplier (tipo de mapa específico):**

```java
TreeMap<String, List<Producto>> ordenadoPorCategoria = productos.stream()
    .collect(Collectors.groupingBy(
        Producto::getCategoria,
        TreeMap::new,
        Collectors.toList()
    ));
```

**Agrupación multinivel:**

```java
// Agrupar por categoría y dentro de cada categoría, por disponibilidad
Map<String, Map<Boolean, List<Producto>>> porCategoriaYDisponibilidad =
    productos.stream()
        .collect(Collectors.groupingBy(
            Producto::getCategoria,
            Collectors.groupingBy(p -> p.getStock() > 0)
        ));
```

### 8.8.7 `partitioningBy` — partición booleana

Caso especial de `groupingBy` que divide en dos grupos (true/false):

```java
Map<Boolean, List<Producto>> disponibleVsAgotado = productos.stream()
    .collect(Collectors.partitioningBy(p -> p.getStock() > 0));

List<Producto> disponibles = disponibleVsAgotado.get(true);
List<Producto> agotados = disponibleVsAgotado.get(false);

// Con downstream collector
Map<Boolean, Long> conteoDisponibles = productos.stream()
    .collect(Collectors.partitioningBy(
        p -> p.getStock() > 0,
        Collectors.counting()
    ));
// Resultado: {true=8, false=3}

// Partición con transformación: sumar precios de disponibles vs agotados
Map<Boolean, Double> precioPorDisponibilidad = productos.stream()
    .collect(Collectors.partitioningBy(
        p -> p.getStock() > 0,
        Collectors.summingDouble(Producto::getPrecio)
    ));
```

### 8.8.8 `mapping` — mapeo dentro del collector

```java
// Equivalente a: stream.map(...).collect(groupingBy(...))
Map<String, List<String>> nombresPorCategoria = productos.stream()
    .collect(Collectors.groupingBy(
        Producto::getCategoria,
        Collectors.mapping(Producto::getNombre, Collectors.toList())
    ));

// Set en lugar de List para evitar duplicados
Map<String, Set<String>> nombresUnicosPorCategoria = productos.stream()
    .collect(Collectors.groupingBy(
        Producto::getCategoria,
        Collectors.mapping(Producto::getNombre, Collectors.toSet())
    ));
```

### 8.8.9 `collectingAndThen` — post-procesamiento

Aplica una transformación después de recolectar:

```java
// Recolectar en una lista y luego hacerla inmutable
List<String> inmutables = productos.stream()
    .map(Producto::getNombre)
    .collect(Collectors.collectingAndThen(
        Collectors.toList(),
        Collections::unmodifiableList
    ));

// El producto más caro por categoría sin Optional
Map<String, Producto> masCaroPorCategoria = productos.stream()
    .collect(Collectors.groupingBy(
        Producto::getCategoria,
        Collectors.collectingAndThen(
            Collectors.maxBy(Comparator.comparing(Producto::getPrecio)),
            Optional::get // seguro porque ningún grupo está vacío
        )
    ));
```

### 8.8.10 `teeing` (Java 12+)

Ejecuta dos collectors independientes sobre el mismo stream y combina los resultados:

```java
// Obtener el más barato y el más caro en una sola pasada
record RangoPrecios(Producto masBarato, Producto masCaro) {}

RangoPrecios rango = productos.stream()
    .collect(Collectors.teeing(
        Collectors.minBy(Comparator.comparing(Producto::getPrecio)),
        Collectors.maxBy(Comparator.comparing(Producto::getPrecio)),
        (min, max) -> new RangoPrecios(min.orElse(null), max.orElse(null))
    ));

// Conteo y suma en una sola pasada
record Estadisticas(long total, int stock) {}

Estadisticas stats = productos.stream()
    .collect(Collectors.teeing(
        Collectors.counting(),
        Collectors.summingInt(Producto::getStock),
        Estadisticas::new
    ));

// Diferencia entre máximo y mínimo con teeing
double rangoPrecios = productos.stream()
    .collect(Collectors.teeing(
        Collectors.summarizingDouble(Producto::getPrecio),
        Collectors.counting(),
        (summary, count) -> count > 0 ? summary.getMax() - summary.getMin() : 0
    ));
```

### 8.8.11 Collector personalizado

Implementar la interfaz `Collector<T, A, R>` donde `T` es el tipo de entrada, `A` el acumulador mutable y `R` el resultado:

```java
// Collector que une strings con un separador (similar a joining)
public class UnirConSeparador implements Collector<String, StringBuilder, String> {
    private final String separador;

    public UnirConSeparador(String separador) {
        this.separador = separador;
    }

    @Override
    public Supplier<StringBuilder> supplier() {
        return StringBuilder::new;
    }

    @Override
    public BiConsumer<StringBuilder, String> accumulator() {
        return (sb, s) -> {
            if (sb.length() > 0) sb.append(separador);
            sb.append(s);
        };
    }

    @Override
    public BinaryOperator<StringBuilder> combiner() {
        return (sb1, sb2) -> sb1.append(separador).append(sb2);
    }

    @Override
    public Function<StringBuilder, String> finisher() {
        return StringBuilder::toString;
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.emptySet();
    }
}

// Uso
String resultado = stream.collect(new UnirConSeparador(" | "));
```
---

## 8.9 Cómo Funcionan los Streams Internamente

Para dominar los Streams en Java no basta con saber usarlos — hay que entender qué ocurre por debajo. Esta sección explora los mecanismos internos que hacen posible la evaluación perezosa, el paralelismo y las optimizaciones del pipeline.

### 8.9.1 El pipeline de operaciones: source → intermediate ops → terminal op

Todo Stream en Java sigue una arquitectura de pipeline con tres fases bien diferenciadas:

```
Fase 1: Source (Fuente)          Fase 2: Intermediate Ops        Fase 3: Terminal Op
┌───────────────────┐          ┌──────────────────────┐        ┌──────────────────┐
│ Collection.stream │  ──────→ │ filter → map → sorted│ ─────→ │ collect / reduce │
│ IntStream.range   │          │ (no se ejecutan aún) │        │ (dispara todo)   │
│ Stream.of(...)    │          │ Construyen el plan   │        │ Ejecuta y consume │
└───────────────────┘          └──────────────────────┘        └──────────────────┘
```

Cada stream se compone internamente de **stages** (etapas) encadenadas como una lista enlazada. Cada operación intermedia crea un nuevo `AbstractPipeline` que referencia a su stage anterior. Solo cuando se invoca la operación terminal, el pipeline completo se "materializa" y comienza a empujar elementos desde la fuente a través de todas las etapas.

Esta estructura permite fusionar operaciones (loop fusion): en lugar de iterar la colección una vez por `filter`, otra por `map` y otra por `collect`, el pipeline procesa **cada elemento de principio a fin antes de pasar al siguiente**:

```java
// Este pipeline:
productos.stream()
    .filter(p -> p.getPrecio() > 100)
    .map(Producto::getNombre)
    .collect(Collectors.toList());

// NO se ejecuta así:
//   1. Filtrar toda la lista → lista intermedia de 500 elementos
//   2. Mapear toda la lista intermedia → otra lista de 500 strings
//   3. Recolectar

// SÍ se ejecuta así (fusión de bucles):
//   para cada producto:
//     si precio > 100:
//       String nombre = producto.getNombre()
//       listaResultado.add(nombre)
```

Esta fusión de bucles ahorra memoria y mejora la localidad de caché de la CPU significativamente.

### 8.9.2 Spliterator: qué es y cómo divide colecciones para paralelismo

`Spliterator<T>` es el motor del paralelismo en Streams. El nombre viene de "Splittable Iterator": un iterador que puede dividirse a sí mismo para procesamiento paralelo.

```java
public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);   // consumir un elemento
    Spliterator<T> trySplit();                         // dividir en dos spliterators
    long estimateSize();                               // tamaño estimado
    int characteristics();                             // metadatos para optimizaciones
    default void forEachRemaining(Consumer<? super T> action); // consumir todos los restantes
}
```

El algoritmo de división funciona recursivamente:
1. El `Spliterator` raíz se divide con `trySplit()` en dos mitades
2. Cada mitad se divide de nuevo con `trySplit()`
3. Cuando ya no se puede dividir más, cada fragmento es procesado por un hilo del `ForkJoinPool`

**Cómo se divide un ArrayList** (implementación `ArrayListSpliterator`): simplemente divide el array por índices. `trySplit()` parte el rango `[origen, fin)` a la mitad. Esto es O(1) y extremadamente eficiente.

**Cómo se divide un HashSet** (implementación `HashMap.KeySpliterator`): divide los buckets del hash table subyacente. Es menos eficiente porque los buckets pueden tener tamaños desiguales.

**Cómo se divide un LinkedList**: NO tiene buena división. La implementación recorre los nodos secuencialmente hasta encontrar el punto medio, lo que hace que `trySplit()` sea O(n). Esto convierte una LinkedList en pésima fuente para `parallelStream()`.

### 8.9.3 Características de Spliterator

Las características son banderas (bitmask) que informan al framework de Streams sobre las propiedades de los datos, permitiendo optimizaciones agresivas:

| Característica | Significado |
|---|---|
| `ORDERED` | Los elementos tienen un orden definido (listas, arrays, sorted sets) |
| `DISTINCT` | No hay duplicados (`Set`, `distinct()`) |
| `SORTED` | Los elementos siguen un orden (`TreeSet`, `sorted()`) |
| `SIZED` | El tamaño exacto se conoce (`estimateSize()` devuelve valor exacto) |
| `SUBSIZED` | Los spliterators hijos también tendrán SIZED |
| `NONNULL` | No hay elementos null |
| `IMMUTABLE` | La fuente no puede modificarse estructuralmente durante la iteración |
| `CONCURRENT` | La fuente puede modificarse concurrentemente sin problemas |

El framework usa estas características para eliminar operaciones redundantes. Por ejemplo:
- Si el spliterator ya es `SORTED`, una operación `sorted()` posterior es un no-op
- Si ya es `DISTINCT`, `distinct()` no hace nada
- Si es `SIZED`, el stream puede pre-alocar arrays al tamaño exacto en `toArray()`

**Inspeccionar las características de una colección:**

```java
Spliterator<String> spliterator = lista.spliterator();
int caracteristicas = spliterator.characteristics();

System.out.println("ORDERED:   " + spliterator.hasCharacteristics(Spliterator.ORDERED));
System.out.println("SIZED:     " + spliterator.hasCharacteristics(Spliterator.SIZED));
System.out.println("SUBSIZED:  " + spliterator.hasCharacteristics(Spliterator.SUBSIZED));
System.out.println("DISTINCT:  " + spliterator.hasCharacteristics(Spliterator.DISTINCT));
System.out.println("SORTED:    " + spliterator.hasCharacteristics(Spliterator.SORTED));
System.out.println("NONNULL:   " + spliterator.hasCharacteristics(Spliterator.NONNULL));
System.out.println("IMMUTABLE: " + spliterator.hasCharacteristics(Spliterator.IMMUTABLE));
System.out.println("CONCURRENT:" + spliterator.hasCharacteristics(Spliterator.CONCURRENT));
System.out.println("Tamaño:    " + spliterator.estimateSize());

// ArrayList:  ORDERED, SIZED, SUBSIZED           → excelente para paralelo
// HashSet:    SIZED, DISTINCT                     → bueno para paralelo
// TreeSet:    ORDERED, SORTED, SIZED, DISTINCT    → excelente para paralelo
// LinkedList: ORDERED, SIZED, SUBSIZED            → pésimo: trySplit es O(n)
```

### 8.9.4 trySplit() y forEachRemaining() en acción

```java
public class DemoSpliterator {
    public static void main(String[] args) {
        List<Integer> numeros = IntStream.rangeClosed(1, 20)
            .boxed().collect(Collectors.toList());

        System.out.println("=== División recursiva de un ArrayList ===");
        dividirYMostrar(numeros.spliterator(), 0);
    }

    private static void dividirYMostrar(Spliterator<Integer> spliterator, int nivel) {
        String indent = "  ".repeat(nivel);

        Spliterator<Integer> mitad = spliterator.trySplit();
        if (mitad != null) {
            System.out.println(indent + "División nivel " + nivel
                + ": izquierda=" + mitad.estimateSize()
                + ", derecha=" + spliterator.estimateSize());
            dividirYMostrar(mitad, nivel + 1);
            dividirYMostrar(spliterator, nivel + 1);
        } else {
            System.out.print(indent + "Hoja: [");
            spliterator.forEachRemaining(n -> System.out.print(n + " "));
            System.out.println("]");
        }
    }
}

// Salida típica:
// División nivel 0: izquierda=10, derecha=10
//   División nivel 1: izquierda=5, derecha=5
//     Hoja: [1 2 3 4 5 ]
//     Hoja: [6 7 8 9 10 ]
//   División nivel 1: izquierda=5, derecha=5
//     Hoja: [11 12 13 14 15 ]
//     Hoja: [16 17 18 19 20 ]
```

### 8.9.5 Crear un Spliterator propio: Fibonacci infinito

Una de las formas más poderosas de entender Spliterator es implementando uno. Vamos a crear un Spliterator que genera la secuencia de Fibonacci bajo demanda:

```java
public class FibonacciSpliterator implements Spliterator<BigInteger> {
    private BigInteger a = BigInteger.ZERO;
    private BigInteger b = BigInteger.ONE;
    private final long limite;
    private long generados = 0;

    public FibonacciSpliterator(long limite) {
        this.limite = limite;
    }

    @Override
    public boolean tryAdvance(Consumer<? super BigInteger> action) {
        if (generados >= limite) return false;

        BigInteger actual = a;
        a = b;
        b = actual.add(b);
        generados++;
        action.accept(actual);
        return true;
    }

    @Override
    public Spliterator<BigInteger> trySplit() {
        long resto = limite - generados;
        if (resto < 10) return null; // no dividir trozos muy pequeños

        // Calcular la división: saltar hasta la mitad
        long mitad = generados + resto / 2;

        // Crear un nuevo spliterator hijo que empieza desde el principio
        FibonacciSpliterator hijo = new FibonacciSpliterator(mitad);

        return hijo;
    }

    @Override
    public long estimateSize() {
        return limite - generados;
    }

    @Override
    public int characteristics() {
        return ORDERED | NONNULL | IMMUTABLE | SIZED;
    }

    // Uso del Spliterator personalizado
    public static void main(String[] args) {
        Stream<BigInteger> fibonacci = StreamSupport.stream(
            new FibonacciSpliterator(50), false);

        fibonacci.forEach(f -> System.out.print(f + " "));
        // 0 1 1 2 3 5 8 13 21 34 55 89 ...
    }
}
```

### 8.9.6 Evaluación perezosa (lazy evaluation): demostración con peek()

La evaluación perezosa es uno de los conceptos más importantes de Streams. Las operaciones intermedias **no se ejecutan** hasta que se invoca una operación terminal. Esto se demuestra fácilmente con `peek()`:

```java
public class DemostracionLazy {
    public static void main(String[] args) {
        System.out.println("=== Sin operación terminal: nada se ejecuta ===");
        Stream.of("A", "B", "C", "D", "E")
            .peek(s -> System.out.println("PEEK filter: " + s))
            .filter(s -> {
                System.out.println("  FILTER: " + s);
                return s.compareTo("C") < 0;
            })
            .peek(s -> System.out.println("  PEEK después filter: " + s))
            .map(s -> {
                System.out.println("    MAP: " + s);
                return "[" + s + "]";
            });
        System.out.println("El stream se definió pero NADA se ejecutó todavía.\n");

        System.out.println("=== Con operación terminal: todo se ejecuta ===");
        List<String> resultado = Stream.of("A", "B", "C", "D", "E")
            .peek(s -> System.out.println("PEEK filter: " + s))
            .filter(s -> {
                System.out.println("  FILTER: " + s);
                return s.compareTo("C") < 0;
            })
            .peek(s -> System.out.println("  PEEK después filter: " + s))
            .map(s -> {
                System.out.println("    MAP: " + s);
                return "[" + s + "]";
            })
            .collect(Collectors.toList());

        System.out.println("\nResultado: " + resultado);
    }
}

// SALIDA:
//
// === Sin operación terminal: nada se ejecuta ===
// El stream se definió pero NADA se ejecutó todavía.
//
// === Con operación terminal: todo se ejecuta ===
// PEEK filter: A
//   FILTER: A
//   PEEK después filter: A
//     MAP: A
// PEEK filter: B
//   FILTER: B
//   PEEK después filter: B
//     MAP: B
// PEEK filter: C
//   FILTER: C
// PEEK filter: D
//   FILTER: D
// PEEK filter: E
//   FILTER: E
// Resultado: [[A], [B]]
```

Observa cómo cada elemento se procesa de principio a fin antes de pasar al siguiente (loop fusion). Y cómo `C`, `D` y `E` nunca llegan al `peek` después del filter porque no pasan el filtro.

### 8.9.7 Corto-circuito: cómo findFirst() y anyMatch() terminan antes

Las operaciones de corto-circuito permiten que el pipeline termine sin procesar todos los elementos. Son más eficientes porque evitan trabajo innecesario:

```java
public class DemostracionCortoCircuito {
    public static void main(String[] args) {
        List<Integer> numeros = Arrays.asList(3, 8, 2, 5, 9, 1, 4, 7, 6);

        System.out.println("=== findFirst: se detiene en el primer match ===");
        Optional<Integer> primero = numeros.stream()
            .peek(n -> System.out.println("Procesando: " + n))
            .filter(n -> {
                System.out.println("  Filter " + n + " > 5? " + (n > 5));
                return n > 5;
            })
            .findFirst();
        System.out.println("Resultado: " + primero.orElse(-1));
        // Solo se procesan: 3 (no pasa), 8 (pasa, fin)

        System.out.println("\n=== anyMatch: se detiene al encontrar verdadero ===");
        boolean existe = numeros.stream()
            .peek(n -> System.out.println("Verificando: " + n))
            .anyMatch(n -> {
                System.out.println("  ¿" + n + " > 5? " + (n > 5));
                return n > 5;
            });
        System.out.println("¿Alguno > 5? " + existe);
        // Solo se procesan: 3 (false), 8 (true, fin)

        System.out.println("\n=== limit con corto-circuito ===");
        List<Integer> primeros3 = numeros.stream()
            .peek(n -> System.out.println("Procesando: " + n))
            .filter(n -> n > 2)
            .limit(3)
            .collect(Collectors.toList());
        System.out.println("Resultado: " + primeros3);
        // Se procesan: 3(vale), 8(vale), 2(no vale), 5(vale, ya tenemos 3 → fin)
    }
}
```

Las operaciones de cortocircuito son:
- `findFirst()`, `findAny()`
- `anyMatch()`, `allMatch()`, `noneMatch()`
- `limit()`, `takeWhile()`

`allMatch()` cortocircuita en la **primera falsedad** (si encuentra un elemento que no cumple, devuelve `false` inmediatamente). `noneMatch()` cortocircuita en la **primera verdad** (si encuentra un elemento que sí cumple, devuelve `false` inmediatamente).

### 8.9.8 Stateless vs Stateful Operations

Las operaciones de Stream se clasifican en dos categorías según necesiten o no mantener estado:

**Operaciones sin estado (stateless):** cada elemento se procesa de forma independiente. Son ideales para paralelismo.

- `filter` — evaluar una condición
- `map` — transformar un elemento
- `flatMap` — expandir un elemento en un sub-stream
- `peek` — inspeccionar un elemento
- `takeWhile` — condición que solo depende del elemento actual

**Operaciones con estado (stateful):** necesitan ver todos los elementos (o un subconjunto significativo) antes de producir resultados.

- `sorted` — necesita ver todos los elementos para ordenar
- `distinct` — necesita mantener un `HashSet` de elementos vistos
- `skip` — necesita contar cuántos elementos ha omitido
- `limit` — necesita contar cuántos elementos ha emitido

```java
// Demostración: sorted() necesita todo antes de emitir
System.out.println("=== sorted: bloquea hasta ver todo ===");
Stream.of(3, 1, 4, 1, 5, 9, 2, 6)
    .peek(n -> System.out.println("Antes de sorted: " + n))
    .sorted()  // ¡necesita ver TODO antes de pasar el primer elemento!
    .peek(n -> System.out.println("  Después de sorted: " + n))
    .collect(Collectors.toList());

// Salida:
// Antes de sorted: 3
// Antes de sorted: 1
// Antes de sorted: 4
// Antes de sorted: 1
// Antes de sorted: 5
// Antes de sorted: 9
// Antes de sorted: 2
// Antes de sorted: 6     ← todos los "Antes" aparecen primero
//   Después de sorted: 1
//   Después de sorted: 1
//   Después de sorted: 2
// ...                     ← solo entonces aparecen los "Después"
```

Las operaciones stateful son problemáticas en paralelo porque cada hilo debe mantener su propio estado parcial y luego combinarlo, lo que añade overhead. En particular, `sorted()` paralelo requiere una fase de merge similar a mergesort distribuido.

---

## 8.10 Streams Paralelos en Profundidad

### 8.10.1 Creación de streams paralelos

```java
// Desde una colección
Stream<Producto> paralelo1 = productos.parallelStream();

// Convertir stream secuencial a paralelo
Stream<Producto> paralelo2 = productos.stream().parallel();

// Convertir paralelo a secuencial
Stream<Producto> secuencial = paralelo2.sequential();
```

### 8.10.2 ForkJoinPool común: todos los parallelStream() comparten el mismo pool

Por debajo, los streams paralelos utilizan el **ForkJoinPool común** de la JVM (`ForkJoinPool.commonPool()`). Este pool tiene por defecto `Runtime.getRuntime().availableProcessors() - 1` hilos (típicamente 7 en un CPU de 8 núcleos, porque el hilo principal también participa en el trabajo).

```java
// Verificar el paralelismo del pool común
System.out.println("Paralelismo: " +
    ForkJoinPool.commonPool().getParallelism());
System.out.println("Núcleos disponibles: " +
    Runtime.getRuntime().availableProcessors());
```

**Problema crítico:** Todos los `parallelStream()` de tu aplicación comparten este pool. Si tienes una aplicación web donde cada petición usa `parallelStream()`, todas compiten por los mismos hilos. Peor aún: si una operación es I/O-bloqueante, puede monopolizar los hilos del pool común y degradar toda la aplicación.

### 8.10.3 Cómo configurar el paralelismo

Hay varias formas de controlar el número de hilos del pool común:

```java
// Opción 1: propiedad del sistema (antes de cualquier uso de parallelStream)
System.setProperty(
    "java.util.concurrent.ForkJoinPool.common.parallelism", "8");

// Opción 2: argumento JVM
// java -Djava.util.concurrent.ForkJoinPool.common.parallelism=8 MiApp

// Opción 3: ForkJoinPool personalizado (para aislar operaciones)
public class StreamParaleloAislado {
    public static void main(String[] args) throws Exception {
        ForkJoinPool poolPersonalizado = new ForkJoinPool(4);

        long resultado = poolPersonalizado.submit(() ->
            IntStream.rangeClosed(1, 10_000_000)
                .parallel()
                .filter(n -> n % 2 == 0)
                .count()
        ).get();

        System.out.println("Resultado con pool propio: " + resultado);
        poolPersonalizado.shutdown();
    }
}
```

**Recomendación:** En aplicaciones empresariales (Spring Boot, Jakarta EE), evita usar `parallelStream()` dentro del pool de peticiones HTTP. Si necesitas paralelismo, usa un `ForkJoinPool` aislado o `CompletableFuture` con `ExecutorService` dedicado.

### 8.10.4 Cuándo NO usar parallelStream()

**1. Datasets pequeños (sobrecarga de coordinación > beneficio)**

Crear y coordinar hilos tiene un costo fijo. Para menos de ~10,000 elementos, el overhead de dividir el trabajo y fusionar resultados suele superar cualquier ganancia.

```java
// Pésima idea: parallelStream con 10 elementos
List<String> lista = Arrays.asList("a", "b", "c", "d", "e");
lista.parallelStream().map(String::toUpperCase).collect(Collectors.toList());
// El overhead de coordinación es mayor que el trabajo real
```

**2. Operaciones I/O-bound (bloquean los hilos del pool)**

```java
// PELIGROSO: I/O en parallelStream agota el pool común
productos.parallelStream().forEach(p -> {
    // Esto bloquea un hilo del ForkJoinPool común
    guardarEnBaseDeDatos(p); // operación de red/BD bloqueante
});
// Efecto: todos los hilos del pool quedan bloqueados,
// paralizando otras operaciones parallelStream de la aplicación

// Alternativa: usar ExecutorService dedicado para I/O
ExecutorService ioPool = Executors.newFixedThreadPool(10);
List<CompletableFuture<Void>> futures = productos.stream()
    .map(p -> CompletableFuture.runAsync(
        () -> guardarEnBaseDeDatos(p), ioPool))
    .collect(Collectors.toList());
CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new)).join();
ioPool.shutdown();
```

**3. Operaciones con estado compartido (race conditions)**

```java
// GRAVE: condición de carrera — resultados no deterministas
List<Integer> acumulador = new ArrayList<>(); // NO thread-safe
IntStream.range(0, 10000)
    .parallel()
    .forEach(acumulador::add); // ArrayList.add no es atómico
System.out.println(acumulador.size()); // ¿10000? Depende de la suerte

// CORRECTO: usar collect con collector thread-safe
List<Integer> seguro = IntStream.range(0, 10000)
    .parallel()
    .boxed()
    .collect(Collectors.toList()); // Thread-safe garantizado
```

**4. Ordenamiento necesario (forEach vs forEachOrdered)**

```java
// forEach en paralelo: orden impredecible
System.out.println("=== forEach en paralelo ===");
IntStream.range(0, 20)
    .parallel()
    .forEach(n -> System.out.print(n + " "));
// Posible salida: 9 10 11 12 13 14 15 16 17 18 19 0 1 2 3 4 5 6 7 8

// forEachOrdered en paralelo: mantiene orden pero pierde paralelismo
System.out.println("\n=== forEachOrdered en paralelo ===");
IntStream.range(0, 20)
    .parallel()
    .forEachOrdered(n -> System.out.print(n + " "));
// Salida: 0 1 2 3 ... 19 (siempre en orden, pero encola los resultados)
```

### 8.10.5 Benchmark: sumar 100 millones de números con stream secuencial vs paralelo

```java
public class BenchmarkSuma {
    public static void main(String[] args) {
        final int N = 100_000_000;

        // Crear array de 100 millones de números
        long[] numeros = new long[N];
        Arrays.parallelSetAll(numeros, i -> i + 1); // 1, 2, 3, ..., 100M

        // Calentamiento JVM (warmup)
        for (int i = 0; i < 5; i++) {
            sumarSecuencial(numeros);
            sumarParalelo(numeros);
        }

        // Benchmark: secuencial
        long inicio = System.nanoTime();
        long sumaSecuencial = sumarSecuencial(numeros);
        long fin = System.nanoTime();
        double tiempoSec = (fin - inicio) / 1_000_000_000.0;

        // Benchmark: paralelo
        inicio = System.nanoTime();
        long sumaParalelo = sumarParalelo(numeros);
        fin = System.nanoTime();
        double tiempoPar = (fin - inicio) / 1_000_000_000.0;

        System.out.println("Suma de 1 a " + N);
        System.out.println("Resultado esperado: " + (N * (N + 1L) / 2));
        System.out.println();
        System.out.println("Stream secuencial: " + sumaSecuencial
            + " en " + String.format("%.3f", tiempoSec) + " segundos");
        System.out.println("Stream paralelo:   " + sumaParalelo
            + " en " + String.format("%.3f", tiempoPar) + " segundos");
        System.out.println("Speedup: " + String.format("%.2fx", tiempoSec / tiempoPar));
    }

    private static long sumarSecuencial(long[] numeros) {
        return Arrays.stream(numeros).sum();
    }

    private static long sumarParalelo(long[] numeros) {
        return Arrays.stream(numeros).parallel().sum();
    }
}

// Salida típica (8 núcleos):
// Suma de 1 a 100000000
// Resultado esperado: 5000000050000000
//
// Stream secuencial: 5000000050000000 en 0.087 segundos
// Stream paralelo:   5000000050000000 en 0.028 segundos
// Speedup: 3.11x
```

### 8.10.6 Ejemplo extendido: conteo de primos con medición

```java
public class ComparacionRendimientoPrimos {
    public static void main(String[] args) {
        int N = 10_000_000;
        List<Integer> numeros = new ArrayList<>(N);
        for (int i = 0; i < N; i++) {
            numeros.add(i);
        }

        // Secuencial
        long inicio = System.nanoTime();
        long totalSecuencial = numeros.stream()
            .filter(ComparacionRendimientoPrimos::esPrimo)
            .count();
        long fin = System.nanoTime();
        System.out.println("Secuencial: " + totalSecuencial + " primos en "
            + (fin - inicio) / 1_000_000 + " ms");

        // Paralelo
        inicio = System.nanoTime();
        long totalParalelo = numeros.parallelStream()
            .filter(ComparacionRendimientoPrimos::esPrimo)
            .count();
        fin = System.nanoTime();
        System.out.println("Paralelo:   " + totalParalelo + " primos en "
            + (fin - inicio) / 1_000_000 + " ms");
    }

    private static boolean esPrimo(int n) {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        for (int i = 3; i <= Math.sqrt(n); i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }
}

// Salida típica (8 núcleos):
// Secuencial: 664579 primos en 12453 ms
// Paralelo:   664579 primos en 1987 ms  (~6x más rápido)
```

### 8.10.7 Reglas de oro para streams paralelos

1. **Medir, medir, medir**. No asumas que `parallelStream()` es más rápido.
2. **Sin estado mutable compartido**. Usa `collect` con `Collector` apropiado.
3. **Evita operaciones que dependan del orden** (salvo que uses `forEachOrdered`).
4. **Cuidado con `findFirst` en paralelo**: puede forzar ordenamiento y perder beneficios.
5. **Usa `findAny`** en paralelo si el orden no importa.
6. **Evita streams paralelos dentro de pools de aplicaciones web** (JEE/Spring): ya tienen su propio pool de hilos y competirían con el ForkJoinPool común.
7. **Elige bien la fuente**: `ArrayList` y arrays se paralelizan bien. `LinkedList` y `Files.lines()` no.
8. **Operaciones con estado caro**: `sorted()` y `distinct()` en paralelo tienen overhead adicional de combinación.
9. **Tamaño mínimo**: no uses `parallelStream()` con menos de 10,000 elementos.
10. **Operaciones no bloqueantes**: nunca hagas I/O dentro de operaciones de stream paralelo.

---

## 8.11 Trampas Comunes con Streams

Incluso programadores experimentados caen en estas trampas. Esta sección recopila los errores más frecuentes y sus soluciones.

### 8.11.1 Consumir un stream más de una vez: IllegalStateException

Un stream solo puede consumirse una vez. Después de la operación terminal, se considera "operated upon" o "closed":

```java
// ERROR: IllegalStateException
Stream<String> stream = nombres.stream();
long cantidad = stream.count();                            // OK: primera terminal
List<String> lista = stream.collect(Collectors.toList());  // ERROR: stream ya consumido

// Solución: re-crear el stream desde la fuente
Stream<String> stream1 = nombres.stream();
long cantidad = stream1.count();

Stream<String> stream2 = nombres.stream();
List<String> lista = stream2.collect(Collectors.toList());

// O mejor: recolectar una vez y reutilizar la colección
List<String> lista = nombres.stream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.toList());
long cantidad = lista.size();          // Reutilizar la lista
lista.forEach(System.out::println);    // Reutilizar la lista
```

### 8.11.2 Modificar la fuente durante el stream: ConcurrentModificationException

Nunca modifiques la colección fuente mientras un stream está en ejecución:

```java
// ERROR: ConcurrentModificationException
List<String> lista = new ArrayList<>(Arrays.asList("A", "B", "C"));
lista.stream().forEach(s -> {
    if (s.equals("B")) {
        lista.add("D"); // ¡Modifica la fuente durante la iteración!
    }
});

// Solución: recolectar primero, modificar después
List<String> lista = new ArrayList<>(Arrays.asList("A", "B", "C"));
List<String> resultado = lista.stream()
    .flatMap(s -> s.equals("B") ? Stream.of(s, "D") : Stream.of(s))
    .collect(Collectors.toList());
```

### 8.11.3 Variables capturadas no effectively final

Las lambdas solo pueden usar variables locales que sean `final` o effectively final:

```java
// ERROR: compilación — la variable debe ser effectively final
int suma = 0;
nombres.forEach(n -> suma += n.length()); // suma no es effectively final

// Solución 1: usar reduce
int suma = nombres.stream().mapToInt(String::length).sum();

// Solución 2: AtomicInteger (para casos donde realmente necesitas mutación)
AtomicInteger suma = new AtomicInteger(0);
nombres.forEach(n -> suma.addAndGet(n.length()));

// Solución 3: array de 1 elemento (hack, no recomendado)
int[] suma = {0};
nombres.forEach(n -> suma[0] += n.length());
```

### 8.11.4 Uso excesivo de collect() cuando reduce() es más eficiente

`collect()` está pensado para acumular en una estructura mutable. `reduce()` es para combinar valores inmutables. Usar `collect` para operaciones de reducción es ineficiente:

```java
// INEFICIENTE: collect con StringBuilder para concatenar
String resultado = palabras.stream()
    .collect(StringBuilder::new,
             StringBuilder::append,
             StringBuilder::append)
    .toString();

// MEJOR: usar reduce para concatenación simple
String resultado = palabras.stream()
    .reduce("", (a, b) -> a + b);

// MEJOR AÚN: usar Collectors.joining() que optimiza internamente
String resultado = palabras.stream()
    .collect(Collectors.joining());

// INEFICIENTE: collect para sumar enteros (crea un objeto int[] mutable)
int[] suma = productos.stream()
    .collect(() -> new int[1],
             (arr, p) -> arr[0] += p.getStock(),
             (a, b) -> a[0] += b[0]);
int total = suma[0];

// CORRECTO: reduce o mapToInt().sum()
int total = productos.stream().mapToInt(Producto::getStock).sum();
```

### 8.11.5 flatMap anidado excesivo

Múltiples `flatMap` encadenados hacen el código ilegible y difícil de depurar:

```java
// CÓDIGO ILEGIBLE: tres niveles de flatMap
List<String> resultado = clientes.stream()
    .flatMap(c -> c.getPedidos().stream())
    .flatMap(p -> p.getItems().stream())
    .flatMap(i -> i.getEtiquetas().stream())
    .distinct()
    .collect(Collectors.toList());

// REFACTOR: extraer métodos con nombres descriptivos
List<String> resultado = clientes.stream()
    .flatMap(Cliente::pedidos)
    .flatMap(Pedido::items)
    .flatMap(Item::etiquetas)
    .distinct()
    .collect(Collectors.toList());

// En las clases, definir métodos helper:
class Cliente {
    public Stream<Pedido> pedidos() { return pedidos.stream(); }
}
class Pedido {
    public Stream<Item> items() { return items.stream(); }
}
class Item {
    public Stream<String> etiquetas() { return etiquetas.stream(); }
}
```

### 8.11.6 forEach con efectos secundarios en Streams

`forEach` es una operación terminal diseñada para efectos secundarios. Sin embargo, abusar de ella rompe el paradigma funcional y causa problemas:

```java
// MALA PRÁCTICA: forEach para lógica que debería ser declarativa
List<Producto> caros = new ArrayList<>();
productos.stream()
    .filter(p -> p.getPrecio() > 100)
    .forEach(p -> caros.add(p)); // efecto secundario

// CORRECTO: collect
List<Producto> caros = productos.stream()
    .filter(p -> p.getPrecio() > 100)
    .collect(Collectors.toList());

// ACEPTABLE: forEach para efectos secundarios legítimos (logging, enviar eventos)
productos.stream()
    .filter(p -> p.getStock() == 0)
    .forEach(p -> log.warn("Producto agotado: " + p.getNombre()));
```

### 8.11.7 Debugging de Streams: cómo usar peek() para inspeccionar el pipeline

`peek()` es la herramienta principal para depurar pipelines. Úsalo para entender qué está pasando en cada etapa:

```java
public class DebugPipeline {
    public static void main(String[] args) {
        List<Producto> productos = crearProductos();

        // Debugging paso a paso con peek
        Map<String, Long> resultado = productos.stream()
            .peek(p -> System.out.println("[INICIO] " + p.getNombre()))
            .filter(p -> {
                boolean pasa = p.getPrecio() > 50;
                System.out.println("  [FILTER precio>50] "
                    + p.getNombre() + " -> " + pasa);
                return pasa;
            })
            .peek(p -> System.out.println("  [DESPUÉS FILTER] " + p.getNombre()))
            .filter(p -> p.getStock() > 0)
            .peek(p -> System.out.println("  [DESPUÉS STOCK] " + p.getNombre()))
            .collect(Collectors.groupingBy(
                Producto::getCategoria,
                Collectors.counting()
            ));

        System.out.println("\nResultado final: " + resultado);
    }

    private static List<Producto> crearProductos() {
        return Arrays.asList(
            new Producto("Laptop", "Electrónica", 999.99, 5),
            new Producto("Manzana", "Alimentos", 1.50, 0),
            new Producto("Tablet", "Electrónica", 299.99, 12),
            new Producto("Pan", "Alimentos", 2.00, 50),
            new Producto("Mouse", "Electrónica", 25.00, 30)
        );
    }
}

// Técnica avanzada: peek con contador para seguir el flujo
AtomicInteger contador = new AtomicInteger(1);
productos.stream()
    .peek(p -> System.out.printf("[%d] Procesando: %s%n",
        contador.getAndIncrement(), p.getNombre()))
    // ... resto del pipeline
```

### 8.11.8 Confundir Stream.iterate Java 8 con Java 9+

```java
// Java 8: Stream.iterate(seed, unaryOperator) — infinito, solo truncable con limit()
Stream<Integer> java8 = Stream.iterate(0, n -> n + 2);
java8.limit(10).forEach(System.out::println); // 0, 2, 4, 6, ..., 18

// Java 9+: Stream.iterate(seed, predicate, unaryOperator) — finito con condición
Stream<Integer> java9 = Stream.iterate(0, n -> n < 20, n -> n + 2);
java9.forEach(System.out::println); // 0, 2, 4, ..., 18 (termina automáticamente)

// Error común: usar Java 8 iterate sin limit → bucle infinito
// Stream.iterate(0, n -> n + 1).collect(Collectors.toList()); // ¡Nunca termina!
```

### 8.11.9 Operaciones stateful que dependen del orden de ejecución

```java
// ERROR: resultado impredecible — skip depende del orden de los elementos
Set<Integer> numeros = new HashSet<>(Arrays.asList(3, 1, 4, 1, 5));
List<Integer> resultado = numeros.stream()
    .skip(2)    // ¿Qué elementos se saltan? ¡Depende del orden del HashSet!
    .limit(3)
    .collect(Collectors.toList());
System.out.println(resultado); // Impredecible entre ejecuciones

// CORRECTO: usar una fuente ordenada si skip/limit importa
List<Integer> numeros = new ArrayList<>(Arrays.asList(3, 1, 4, 1, 5));
List<Integer> resultado = numeros.stream()
    .sorted()   // o usar el orden natural de la lista
    .skip(2)
    .limit(3)
    .collect(Collectors.toList());
System.out.println(resultado); // [3, 4, 5] — predecible
```

---

## 8.12 Optional Avanzado

### 8.12.1 Optional como mónada: map, flatMap, filter

`Optional` es una mónada: un contenedor que soporta encadenamiento de operaciones mediante `map`, `flatMap` y `filter`. Esto permite escribir pipelines de transformación condicional sin un solo `if (x != null)`:

```java
// Sin Optional: pirámide del infierno
public String getCodigoPostalFormateado(Pedido pedido) {
    if (pedido != null) {
        Cliente cliente = pedido.getCliente();
        if (cliente != null) {
            Direccion direccion = cliente.getDireccion();
            if (direccion != null) {
                CodigoPostal cp = direccion.getCodigoPostal();
                if (cp != null) {
                    return cp.getCodigo().trim().toUpperCase();
                }
            }
        }
    }
    return "SIN CP";
}

// Con mónada Optional: cadena plana
public String getCodigoPostalFormateado(Pedido pedido) {
    return Optional.ofNullable(pedido)
        .map(Pedido::getCliente)
        .map(Cliente::getDireccion)
        .map(Direccion::getCodigoPostal)
        .map(CodigoPostal::getCodigo)
        .map(String::trim)
        .map(String::toUpperCase)
        .orElse("SIN CP");
}
```

**Diferencia clave entre map y flatMap en Optional:**

```java
// map: cuando la función devuelve un valor normal (no Optional)
Optional<Cliente> cliente = Optional.of(pedido)
    .map(Pedido::getCliente); // Pedido -> Cliente

// flatMap: cuando la función ya devuelve Optional (evita Optional<Optional<T>>)
Optional<Cliente> cliente = buscarPedido(id)    // Optional<Pedido>
    .flatMap(Pedido::getClienteOpt);             // Pedido -> Optional<Cliente>

// Sin flatMap tendríamos: Optional<Optional<Cliente>> ← engorroso
```

### 8.12.2 Optional vs null: cuándo usar Optional como retorno

El debate tiene pautas claras respaldadas por los diseñadores de Java:

**Usa Optional como tipo de retorno cuando:**
- El método puede legítimamente no encontrar un resultado
- Quieres forzar al llamador a manejar la ausencia
- Es parte de una API pública

```java
// CORRECTO: retornar Optional para "puede no existir"
public Optional<Usuario> buscarPorEmail(String email) {
    return Optional.ofNullable(baseDeDatos.findByEmail(email));
}

public Optional<Configuracion> getConfiguracion(String clave) {
    return Optional.ofNullable(mapaConfiguraciones.get(clave));
}

// INCORRECTO: retornar Optional para colecciones
// NO: Optional<List<Producto>> — usa Collections.emptyList()
// CORRECTO:
public List<Producto> buscarPorCategoria(String cat) {
    List<Producto> resultado = repositorio.findByCategoria(cat);
    return resultado != null ? resultado : Collections.emptyList();
}
```

**Reglas generales:**
- Retorna `Optional` en vez de `null` para valores escalares que pueden faltar
- NO retornes `Optional<List<T>>`, `Optional<Set<T>>`, etc. — usa colecciones vacías
- NO uses `Optional` como parámetro de método — fuerza al llamador a envolver valores
- En getters de beans simples, es aceptable pero controvertido

### 8.12.3 Optional como campo: casi NUNCA

```java
// MUY MALA PRÁCTICA: Optional como campo
public class Producto {
    private Optional<String> descripcion; // NO HAGAS ESTO

    // Problemas:
    // 1. Optional no es Serializable (rompe serialización)
    // 2. Ocupa más memoria (el objeto Optional + el valor)
    // 3. No fue diseñado para esto (lo dice Brian Goetz, arquitecto de Java)
    // 4. Hibernate/JPA no lo soportan
}

// CORRECTO: campo normal + getter que retorna Optional
public class Producto {
    private String descripcion; // campo normal, nullable

    public Optional<String> getDescripcion() {
        return Optional.ofNullable(descripcion);
    }

    // Si necesitas el setter, recibe el valor normal
    public void setDescripcion(String descripcion) {
        this.descripcion = descripcion;
    }
}
```

### 8.12.4 orElseGet() vs orElse(): diferencia de evaluación perezosa

Esta es una de las trampas más comunes incluso entre programadores con experiencia:

```java
// PROBLEMA: orElse() evalúa su argumento SIEMPRE, incluso si Optional tiene valor
public String getNombre() {
    return optionalNombre.orElse(generarNombrePorDefecto());
    // generadoPorDefecto() se ejecuta AUNQUE optionalNombre tenga valor
    // Si la generación es costosa (consulta BD, llamada API), estás desperdiciando recursos
}

// CORRECTO: orElseGet() solo evalúa si Optional está vacío
public String getNombre() {
    return optionalNombre.orElseGet(() -> generarNombrePorDefecto());
    // La lambda solo se ejecuta si optionalNombre está vacío
}

// Demostración visual:
public class OrElseVsOrElseGet {
    public static void main(String[] args) {
        Optional<String> lleno = Optional.of("Java");

        System.out.println("=== orElse: evalúa SIEMPRE ===");
        String r1 = lleno.orElse(costoso("orElse"));
        System.out.println("Resultado: " + r1);

        System.out.println("\n=== orElseGet: evalúa SOLO SI ES NECESARIO ===");
        String r2 = lleno.orElseGet(() -> costoso("orElseGet"));
        System.out.println("Resultado: " + r2);
    }

    private static String costoso(String tipo) {
        System.out.println("  >>> EJECUTANDO operación costosa para " + tipo);
        return "Valor por defecto";
    }
}

// SALIDA:
// === orElse: evalúa SIEMPRE ===
//   >>> EJECUTANDO operación costosa para orElse    ← ¡Se ejecutó!
// Resultado: Java
//
// === orElseGet: evalúa SOLO SI ES NECESARIO ===
// Resultado: Java                                     ← No se ejecutó la lambda
```

**Regla mnemotécnica:** `orElse(T valor)` = siempre hambriento (eager). `orElseGet(Supplier<T>)` = perezoso (lazy). Usa `orElse` solo para constantes (`orElse("Desconocido")`). Para cualquier cosa que requiera computación, usa `orElseGet`.

### 8.12.5 ifPresentOrElse() (Java 9+)

Combina dos acciones en una: qué hacer si hay valor y qué hacer si no:

```java
// Java 8: dos comprobaciones separadas
if (optional.isPresent()) {
    System.out.println("Encontrado: " + optional.get());
} else {
    System.out.println("No encontrado");
}

// Java 9+: ifPresentOrElse — una sola llamada
optional.ifPresentOrElse(
    valor -> System.out.println("Encontrado: " + valor),
    () -> System.out.println("No encontrado")
);

// Ejemplo práctico: cargar de caché o de base de datos
public Producto obtenerProducto(String id) {
    Optional<Producto> cacheado = cache.get(id);
    cacheado.ifPresentOrElse(
        p -> log.info("Cache hit: " + id),     // éxito
        () -> log.warn("Cache miss: " + id)     // fallo
    );
    return cacheado.orElseGet(() -> {
        Producto p = baseDatos.buscar(id);
        cache.put(id, p);
        return p;
    });
}
```

### 8.12.6 or() (Java 9+)

Permite encadenar múltiples fuentes alternativas de Optional:

```java
// Java 8: encadenamiento manual
public Optional<Producto> buscarProducto(String id) {
    Optional<Producto> resultado = buscarEnCache(id);
    if (!resultado.isPresent()) {
        resultado = buscarEnBDPrincipal(id);
    }
    if (!resultado.isPresent()) {
        resultado = buscarEnBDRespaldo(id);
    }
    return resultado;
}

// Java 9+: or() encadenado — mucho más elegante
public Optional<Producto> buscarProducto(String id) {
    return buscarEnCache(id)
        .or(() -> buscarEnBDPrincipal(id))
        .or(() -> buscarEnBDRespaldo(id));
}

// Ejemplo con múltiples repositorios
public Optional<Usuario> autenticar(String email, String password) {
    return buscarEnRepositorioLocal(email)
        .or(() -> buscarEnRepositorioRemoto(email))
        .or(() -> buscarEnLDAP(email))
        .filter(u -> u.passwordCoincide(password));
}

// or() con Supplier perezoso — solo consulta si es necesario
public Optional<Configuracion> getConfig(String clave) {
    return Optional.ofNullable(configuraciones.get(clave))
        .or(() -> Optional.ofNullable(valoresPorDefecto.get(clave)))
        .or(() -> Optional.ofNullable(calcularConfigDinamica(clave)));
}
```

### 8.12.7 stream() (Java 9+)

Convierte un Optional en un Stream de 0 o 1 elementos. Esto permite integrar Optional en pipelines de Stream de forma nativa:

```java
// Java 8: convertir Optional a Stream manualmente
List<Producto> productosConDescuento = ids.stream()
    .map(repositorio::buscarPorId)       // Stream<Optional<Producto>>
    .filter(Optional::isPresent)
    .map(Optional::get)
    .filter(p -> p.tieneDescuento())
    .collect(Collectors.toList());

// Java 9+: flatMap con Optional.stream()
List<Producto> productosConDescuento = ids.stream()
    .map(repositorio::buscarPorId)       // Stream<Optional<Producto>>
    .flatMap(Optional::stream)           // Stream<Producto> (0 o 1 elementos)
    .filter(p -> p.tieneDescuento())
    .collect(Collectors.toList());

// Otro ejemplo: procesar todos los valores presentes
List<Optional<String>> opcionales = Arrays.asList(
    Optional.of("A"), Optional.empty(), Optional.of("B"));
List<String> valores = opcionales.stream()
    .flatMap(Optional::stream)
    .collect(Collectors.toList());
System.out.println(valores); // [A, B]
```

---

## 8.13 Collectors Personalizados y Avanzados

### 8.13.1 Crear un Collector propio: supplier, accumulator, combiner, finisher

La interfaz `Collector<T, A, R>` requiere implementar cuatro componentes:

| Componente | Función | Tipo |
|---|---|---|
| `supplier()` | Crea el contenedor mutable | `Supplier<A>` |
| `accumulator()` | Añade un elemento al contenedor | `BiConsumer<A, T>` |
| `combiner()` | Fusiona dos contenedores (paralelo) | `BinaryOperator<A>` |
| `finisher()` | Transforma A en R (resultado final) | `Function<A, R>` |

```java
// Collector que calcula estadísticas (min, max, avg, count) en una sola pasada
public class EstadisticasCollector
    implements Collector<Double, EstadisticasCollector.Acumulador,
                          EstadisticasCollector.Resultado> {

    @Override
    public Supplier<Acumulador> supplier() {
        return Acumulador::new;
    }

    @Override
    public BiConsumer<Acumulador, Double> accumulator() {
        return (acc, valor) -> {
            acc.count++;
            acc.sum += valor;
            acc.sumCuadrados += valor * valor;
            if (valor < acc.min) acc.min = valor;
            if (valor > acc.max) acc.max = valor;
        };
    }

    @Override
    public BinaryOperator<Acumulador> combiner() {
        return (a, b) -> {
            Acumulador combinado = new Acumulador();
            combinado.count = a.count + b.count;
            combinado.sum = a.sum + b.sum;
            combinado.sumCuadrados = a.sumCuadrados + b.sumCuadrados;
            combinado.min = Math.min(a.min, b.min);
            combinado.max = Math.max(a.max, b.max);
            return combinado;
        };
    }

    @Override
    public Function<Acumulador, Resultado> finisher() {
        return acc -> new Resultado(
            acc.count,
            acc.min,
            acc.max,
            acc.sum / acc.count,
            Math.sqrt(acc.sumCuadrados / acc.count -
                     (acc.sum / acc.count) * (acc.sum / acc.count))
        );
    }

    @Override
    public Set<Characteristics> characteristics() {
        return EnumSet.of(Characteristics.UNORDERED,
                         Characteristics.CONCURRENT);
    }

    static class Acumulador {
        long count = 0;
        double sum = 0.0;
        double sumCuadrados = 0.0;
        double min = Double.POSITIVE_INFINITY;
        double max = Double.NEGATIVE_INFINITY;
    }

    public record Resultado(
        long count,
        double min,
        double max,
        double media,
        double desviacionEstandar
    ) {
        @Override
        public String toString() {
            return String.format(
                "n=%d, min=%.2f, max=%.2f, μ=%.2f, σ=%.2f",
                count, min, max, media, desviacionEstandar);
        }
    }

    // Uso
    public static void main(String[] args) {
        List<Double> datos = Arrays.asList(
            10.5, 12.3, 9.8, 15.1, 11.2, 13.7, 10.0, 14.5);

        Resultado stats = datos.stream()
            .collect(new EstadisticasCollector());

        System.out.println(stats);
        // n=8, min=9.80, max=15.10, μ=12.14, σ=1.92
    }
}
```

### 8.13.2 collectingAndThen: modificar el resultado después de recolectar

```java
// Envolver resultado en objeto inmutable
List<String> inmutables = productos.stream()
    .map(Producto::getNombre)
    .collect(Collectors.collectingAndThen(
        Collectors.toList(),
        Collections::unmodifiableList
    ));

// Obtener resultado de grupo sin Optional
Map<String, Producto> masCaroPorCategoria = productos.stream()
    .collect(Collectors.groupingBy(
        Producto::getCategoria,
        Collectors.collectingAndThen(
            Collectors.maxBy(Comparator.comparing(Producto::getPrecio)),
            opt -> opt.orElseThrow(() ->
                new IllegalStateException("Categoría sin productos"))
        )
    ));

// Transformar Map tras groupingBy
Map<String, Integer> totalesPorCategoria = productos.stream()
    .collect(Collectors.collectingAndThen(
        Collectors.groupingBy(
            Producto::getCategoria,
            Collectors.summingInt(Producto::getStock)
        ),
        map -> {
            int total = map.values().stream().mapToInt(Integer::intValue).sum();
            map.put("TOTAL", total);
            return map;
        }
    ));
```

### 8.13.3 groupingByConcurrent: agrupación thread-safe para streams paralelos

```java
// groupingBy normal NO es thread-safe para streams paralelos
// groupingByConcurrent SÍ lo es (usa ConcurrentHashMap internamente)

// groupingByConcurrent: para streams paralelos
ConcurrentMap<String, List<Producto>> porCategoriaParalelo =
    productos.parallelStream()
        .collect(Collectors.groupingByConcurrent(
            Producto::getCategoria
        ));

// Con downstream collector
ConcurrentMap<String, Long> conteoParalelo =
    productos.parallelStream()
        .collect(Collectors.groupingByConcurrent(
            Producto::getCategoria,
            Collectors.counting()
        ));

// Diferencia clave: groupingByConcurrent NO preserva el orden de los grupos.
// Si necesitas orden (TreeMap), usa groupingBy normal con stream secuencial.
```

### 8.13.4 flatMapping() y filtering() collectors (Java 9+)

Estos collectors trabajan dentro de un downstream collector, como `groupingBy`. Permiten filtrar o expandir dentro del grupo:

```java
// flatMapping: expande cada elemento del grupo en un sub-stream
// Ejemplo: agrupar pedidos por cliente y obtener todas las categorías de productos
Map<Cliente, Set<String>> categoriasPorCliente = pedidos.stream()
    .collect(Collectors.groupingBy(
        Pedido::getCliente,
        Collectors.flatMapping(
            p -> p.getItems().stream().map(Item::getCategoria),
            Collectors.toSet()
        )
    ));

// filtering: filtra elementos dentro del grupo
// Ejemplo: agrupar por departamento y contar solo empleados con salario > 50000
Map<String, Long> empleadosBienPagadosPorDept = empleados.stream()
    .collect(Collectors.groupingBy(
        Empleado::getDepartamento,
        Collectors.filtering(
            e -> e.getSalario() > 50000,
            Collectors.counting()
        )
    ));

// Combinación: flatMapping + filtering en groupingBy
Map<Cliente, Set<String>> etiquetasCarasPorCliente = pedidos.stream()
    .collect(Collectors.groupingBy(
        Pedido::getCliente,
        Collectors.flatMapping(
            p -> p.getItems().stream()
                .filter(i -> i.getPrecio() > 100)
                .map(Item::getEtiquetas)
                .flatMap(Set::stream),
            Collectors.toSet()
        )
    ));
```

### 8.13.5 Ejemplo completo: agrupar pedidos por cliente y sumar totales

```java
public class ReporteVentas {
    public record Cliente(String nombre, String ciudad) {}
    public record Item(String producto, double precio, int cantidad) {
        public double subtotal() { return precio * cantidad; }
    }
    public record Pedido(String id, Cliente cliente, List<Item> items) {
        public double total() {
            return items.stream().mapToDouble(Item::subtotal).sum();
        }
    }
    public record ResumenCliente(
        String cliente, String ciudad, long numPedidos,
        double totalGastado, double pedidoMaximo
    ) {}

    public static void main(String[] args) {
        Cliente ana = new Cliente("Ana García", "Madrid");
        Cliente carlos = new Cliente("Carlos López", "Barcelona");
        Cliente beatriz = new Cliente("Beatriz Ruiz", "Madrid");

        List<Pedido> pedidos = Arrays.asList(
            new Pedido("P001", ana, Arrays.asList(
                new Item("Laptop", 800, 1),
                new Item("Mouse", 25, 2))),
            new Pedido("P002", carlos, Arrays.asList(
                new Item("Monitor", 300, 1),
                new Item("Teclado", 60, 1))),
            new Pedido("P003", ana, Arrays.asList(
                new Item("Tablet", 400, 1))),
            new Pedido("P004", beatriz, Arrays.asList(
                new Item("Monitor", 300, 2),
                new Item("Webcam", 80, 1))),
            new Pedido("P005", carlos, Arrays.asList(
                new Item("Auriculares", 90, 1)))
        );

        // Agrupar por cliente y calcular métricas en UNA SOLA PASADA
        Map<Cliente, ResumenCliente> reporte = new HashMap<>();
        pedidos.stream()
            .collect(Collectors.groupingBy(Pedido::cliente))
            .forEach((cliente, pedidosCliente) -> {
                long num = pedidosCliente.size();
                double total = pedidosCliente.stream()
                    .mapToDouble(Pedido::total).sum();
                double maxPedido = pedidosCliente.stream()
                    .mapToDouble(Pedido::total).max().orElse(0.0);
                reporte.put(cliente, new ResumenCliente(
                    cliente.nombre(), cliente.ciudad(),
                    num, total, maxPedido
                ));
            });

        reporte.forEach((cliente, resumen) ->
            System.out.printf("%s (%s): %d pedidos, total=%.2f€, máx=%.2f€%n",
                resumen.cliente(), resumen.ciudad(),
                resumen.numPedidos(), resumen.totalGastado(),
                resumen.pedidoMaximo()));
    }
}

// Salida:
// Ana García (Madrid): 2 pedidos, total=1250.00€, máx=850.00€
// Carlos López (Barcelona): 2 pedidos, total=450.00€, máx=360.00€
// Beatriz Ruiz (Madrid): 1 pedidos, total=680.00€, máx=680.00€
```

---

## 8.14 Streams Infinitos

Los streams infinitos son una herramienta poderosa para modelar secuencias conceptualmente ilimitadas: números naturales, secuencias matemáticas, generación de datos aleatorios, flujos de sensores. Son posibles gracias a la evaluación perezosa.

### 8.14.1 Stream.iterate() (Java 9+): generar secuencias infinitas con condición

```java
// Java 8: iterate sin condición → infinito, necesita limit()
Stream<Integer> pares = Stream.iterate(0, n -> n + 2);
pares.limit(10).forEach(System.out::println);

// Java 9+: iterate con predicado (hasNext)
// (semilla, predicado, función) → finito pero definido como infinito
Stream<Integer> menoresDe100 = Stream.iterate(0, n -> n < 100, n -> n + 2);
menoresDe100.forEach(System.out::println);

// Ejemplos prácticos de iterate
// Días desde hoy durante 30 días
Stream<LocalDate> dias = Stream.iterate(
    LocalDate.now(),
    d -> d.isBefore(LocalDate.now().plusDays(30)),
    d -> d.plusDays(1)
);

// Progresión geométrica: potencias de 2 menores de 10000
Stream<Integer> potenciasDe2 = Stream.iterate(
    1, n -> n < 10000, n -> n * 2);
potenciasDe2.forEach(n -> System.out.print(n + " "));
// 1 2 4 8 16 32 64 128 256 512 1024 2048 4096 8192
```

### 8.14.2 Stream.generate(): suplidor infinito

`generate()` produce un stream infinito a partir de un `Supplier<T>`. A diferencia de `iterate()`, los elementos no dependen del anterior:

```java
// Números aleatorios (infinito)
Stream<Double> aleatorios = Stream.generate(Math::random);

// UUIDs (infinito)
Stream<UUID> uuids = Stream.generate(UUID::randomUUID);

// Objetos desde un pool
AtomicInteger contador = new AtomicInteger(1);
Stream<String> etiquetas = Stream.generate(() ->
    "item-" + contador.getAndIncrement());

// Tomar solo los primeros 5
etiquetas.limit(5).forEach(System.out::println);
// item-1
// item-2
// item-3
// item-4
// item-5
```

### 8.14.3 Generar primos bajo demanda

```java
public class GeneradorPrimos {
    public static void main(String[] args) {
        // Stream infinito de números primos
        Stream<Integer> primos = Stream.iterate(
            2, GeneradorPrimos::siguientePrimo);

        // Obtener los primeros 20 primos
        primos.limit(20).forEach(p -> System.out.print(p + " "));
        // 2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71
    }

    private static int siguientePrimo(int actual) {
        int candidato = actual + 1;
        while (!esPrimo(candidato)) {
            candidato++;
        }
        return candidato;
    }

    private static boolean esPrimo(int n) {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        for (int i = 3; i * i <= n; i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }
}
```

### 8.14.4 Secuencia de Fibonacci con generate() e iterate()

```java
public class FibonacciStreams {
    public static void main(String[] args) {
        // Método 1: Stream.iterate con array de estado
        Stream.iterate(
            new long[]{0, 1},
            fib -> new long[]{fib[1], fib[0] + fib[1]}
        )
        .limit(20)
        .forEach(fib -> System.out.print(fib[0] + " "));
        System.out.println();
        // 0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987 1597 2584 4181

        // Método 2: Stream.iterate (Java 9+) con clase contenedora
        record Par(long a, long b) {
            Par siguiente() { return new Par(b, a + b); }
        }

        Stream.iterate(
            new Par(0, 1),
            p -> true,  // infinito
            Par::siguiente
        )
        .limit(15)
        .map(Par::a)
        .forEach(n -> System.out.print(n + " "));
        System.out.println();
        // 0 1 1 2 3 5 8 13 21 34 55 89 144 233 377

        // Método 3: Stream.generate con estado mutable (cuidado en paralelo)
        long[] fib = {0, 1};
        Stream.generate(() -> {
            long actual = fib[0];
            long siguiente = fib[1];
            fib[0] = siguiente;
            fib[1] = actual + siguiente;
            return actual;
        })
        .limit(10)
        .forEach(n -> System.out.print(n + " "));
        // 0 1 1 2 3 5 8 13 21 34
    }
}
```

### 8.14.5 takeWhile() (Java 9+): truncar streams infinitos

`takeWhile()` es fundamental para trabajar con streams infinitos de forma segura:

```java
// Sin takeWhile: riesgo de bucle infinito
// Stream.iterate(0, n -> n + 1).collect(Collectors.toList()); // ¡NUNCA TERMINA!

// Con takeWhile: seguro
List<Integer> primeros100 = Stream.iterate(0, n -> n + 1)
    .takeWhile(n -> n < 100)
    .collect(Collectors.toList());

// takeWhile con streams de datos ordenados
// Procesar transacciones hasta encontrar un error
Stream<Double> lecturasSensor = Stream.generate(() -> Math.random() * 100);
List<Double> lecturasEstables = lecturasSensor
    .takeWhile(v -> v > 10.0)  // detenerse al primer valor anómalo
    .collect(Collectors.toList());

// Ejemplo: sumar números hasta que la suma supere un umbral
int[] suma = {0};
int[] contador = {0};
Stream.iterate(1, n -> n + 1)
    .takeWhile(n -> {
        suma[0] += n;
        contador[0]++;
        return suma[0] < 1000;
    })
    .forEach(n -> {}); // operación terminal vacía
System.out.println("Se necesitan " + contador[0] +
    " números para superar 1000. Suma=" + suma[0]);
```

### 8.14.6 dropWhile() (Java 9+)

Descarta elementos mientras se cumple una condición, luego procesa el resto:

```java
// Ignorar líneas de comentario al inicio de un archivo
try (Stream<String> lineas = Files.lines(Path.of("config.ini"))) {
    List<String> lineasReales = lineas
        .dropWhile(l -> l.startsWith("#") || l.isBlank())
        .collect(Collectors.toList());
}

// Procesar un log desde una fecha específica (asumiendo líneas ordenadas)
List<String> log = Arrays.asList(
    "[2024-01-01] Inicio",
    "[2024-01-02] Operación A",
    "[2024-06-15] Operación B",
    "[2024-12-31] Fin");

List<String> desdeJunio = log.stream()
    .dropWhile(l -> l.compareTo("[2024-06-01]") < 0)
    .collect(Collectors.toList());
System.out.println(desdeJunio);
// [[2024-06-15] Operación B, [2024-12-31] Fin]
```
---

## 8.15 Rendimiento: Streams vs Bucles Tradicionales

Una de las preguntas más frecuentes: ¿son los streams más lentos que los bucles `for` tradicionales? La respuesta no es binaria: depende del caso de uso, el tamaño de los datos y si hay autoboxing involucrado.

### 8.15.1 Benchmark: sum, filter-map-reduce

Primero, un benchmark comparativo sin framework (para entender las diferencias):

```java
public class BenchmarkStreamVsBucle {
    private static final int N = 10_000_000;
    private static final int WARMUP = 10;
    private static final int ITERACIONES = 20;

    public static void main(String[] args) {
        int[] datos = new int[N];
        Arrays.setAll(datos, i -> i);

        // Warmup JVM
        for (int i = 0; i < WARMUP; i++) {
            bucleFor(datos);
            streamSecuencial(datos);
            streamIntStream(datos);
        }

        System.gc();

        // Benchmark
        medir("Bucle for clásico   ", () -> bucleFor(datos));
        medir("Stream<Integer>     ", () -> streamSecuencial(datos));
        medir("IntStream (primitivo)", () -> streamIntStream(datos));
    }

    private static long bucleFor(int[] datos) {
        long suma = 0;
        for (int i = 0; i < datos.length; i++) {
            int n = datos[i];
            if (n % 2 == 0) {
                suma += n * n;
            }
        }
        return suma;
    }

    private static long streamSecuencial(int[] datos) {
        return Arrays.stream(datos)
            .boxed()
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> (long) n * n)
            .sum();
    }

    private static long streamIntStream(int[] datos) {
        return Arrays.stream(datos)
            .filter(n -> n % 2 == 0)
            .mapToLong(n -> (long) n * n)
            .sum();
    }

    private static void medir(String etiqueta, Supplier<Long> fn) {
        long total = 0;
        long minTiempo = Long.MAX_VALUE;
        for (int i = 0; i < ITERACIONES; i++) {
            long inicio = System.nanoTime();
            long resultado = fn.get();
            long fin = System.nanoTime();
            total += resultado; // evitar que el JIT elimine código muerto
            minTiempo = Math.min(minTiempo, fin - inicio);
        }
        System.out.printf("%s: %.2f ms (mejor de %d iteraciones)%n",
            etiqueta, minTiempo / 1_000_000.0, ITERACIONES);
    }
}

// Resultados típicos:
// Bucle for clásico   : 15.23 ms
// Stream<Integer>     : 82.45 ms  ← ¡autoboxing penaliza 5x!
// IntStream (primitivo): 18.12 ms  ← prácticamente igual al bucle for
```

### 8.15.2 Autoboxing en streams: por qué IntStream/DoubleStream/LongStream son más rápidos

El autoboxing es la conversión implícita entre tipos primitivos (`int`) y sus envoltorios (`Integer`). Los streams genéricos (`Stream<T>`) fuerzan boxing en cada operación:

```java
// MAL: Stream<Integer> — boxing en cada paso
Stream<Integer> numeros = IntStream.range(0, N).boxed();   // cada int → Integer
numeros.filter(n -> n % 2 == 0)                             // desempaquetar para %
    .map(n -> n * n)                                         // desempaquetar, multiplicar, empaquetar
    .collect(Collectors.toList());                           // empaquetar para lista

// BIEN: IntStream — cero boxing
IntStream.range(0, N)
    .filter(n -> n % 2 == 0)                                 // int nativo
    .map(n -> n * n)                                          // int nativo
    .toArray();                                               // int[] nativo
```

**Impacto medido del boxing:**

| Operación | Stream<Tipo> | Stream primitivo | Bucle for | Penalización boxing |
|---|---|---|---|---|
| Sumar 10M ints | 82 ms | 18 ms | 15 ms | ~5x más lento |
| Filtrar y contar | 95 ms | 22 ms | 19 ms | ~4.5x más lento |
| Map y suma | 110 ms | 25 ms | 21 ms | ~4x más lento |

**Conclusión:** `Stream<T>` con tipos envoltorios es 4-6x más lento que `IntStream`/`LongStream`/`DoubleStream`. Para cualquier operación numérica significativa, **siempre usa streams primitivos**.

### 8.15.3 El costo de crear lambdas y streams vs un bucle for

```java
// ¿Cuánto cuesta crear un stream?
// Infraestructura básica de un stream:
//   - Objeto AbstractPipeline (~80 bytes en heap)
//   - Uno por cada operación intermedia
//   - Lambdas capturadas (invokedynamic, generadas una vez)
//   - Spliterator desde la fuente

// Para 10 elementos: el overhead domina — bucle for es mejor
// Para 10,000 elementos: similar rendimiento
// Para 1,000,000+ elementos: streams pueden ser más rápidos (mejor uso de caché)
```

### 8.15.4 Conclusión: streams vs bucles

```
Rendimiento comparativo:

Para datasets pequeños (<1000 elementos):
  Bucle for > Stream (overhead de infraestructura domina)

Para datasets medianos (1000-100000 elementos):
  Bucle for ≈ Stream (rendimiento comparable)

Para datasets grandes (>100000 elementos):
  Stream primitivo ≈ Bucle for (loop fusion y localidad de caché ayudan)
  Stream paralelo >> Bucle for (en CPU-bound con buena fuente)

Para legibilidad y mantenibilidad:
  Stream >> Bucle for (código declarativo, menos errores)
  
Recomendación práctica:
  1. Escribe primero con Streams (más legible)
  2. Si el rendimiento es crítico, mide con JMH o profiler
  3. Solo refactoriza a bucles for si el benchmark lo justifica
  4. Siempre usa streams primitivos para operaciones numéricas
```

---

## 8.16 Streams y Manejo de Excepciones

### 8.16.1 El problema: las checked exceptions no encajan en lambdas

El mayor punto de fricción entre Streams y el sistema de excepciones de Java: las interfaces funcionales (`Function`, `Predicate`, `Consumer`) **no declaran excepciones verificadas**. Esto obliga a capturarlas dentro de la lambda:

```java
// PROBLEMA: IOException en Files.lines — ya manejado por el stream
// Pero si tu lambda lanza checked exception...

// ESTO NO COMPILA:
// productos.stream()
//     .map(p -> new URI(p.getUrl())) // URISyntaxException es checked
//     .collect(Collectors.toList());

// Solución 1: try-catch dentro de la lambda (verbose, rompe el flujo)
productos.stream()
    .map(p -> {
        try {
            return new URI(p.getUrl());
        } catch (URISyntaxException e) {
            throw new RuntimeException(e);
        }
    })
    .collect(Collectors.toList());
```

### 8.16.2 Solución: función wrapper reutilizable

```java
// Paso 1: definir una interfaz funcional que permita checked exceptions
@FunctionalInterface
public interface ThrowingFunction<T, R, E extends Exception> {
    R apply(T t) throws E;
}

// Paso 2: método helper que envuelve la checked en runtime
public static <T, R> Function<T, R> wrap(ThrowingFunction<T, R, Exception> fn) {
    return t -> {
        try {
            return fn.apply(t);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    };
}

// Paso 3: uso limpio
List<URI> uris = productos.stream()
    .map(wrap(p -> new URI(p.getUrl())))
    .collect(Collectors.toList());
```

**Versión genérica completa:**

```java
// Interfaz funcional con checked exception
@FunctionalInterface
public interface ThrowingFunction<T, R, E extends Exception> {
    R apply(T t) throws E;

    static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R, Exception> fn) {
        return t -> {
            try {
                return fn.apply(t);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}

// Interfaz para Consumer con checked exception
@FunctionalInterface
public interface ThrowingConsumer<T, E extends Exception> {
    void accept(T t) throws E;

    static <T> Consumer<T> unchecked(ThrowingConsumer<T, Exception> fn) {
        return t -> {
            try {
                fn.accept(t);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}

// Uso limpio
productos.stream()
    .map(ThrowingFunction.unchecked(p -> new URI(p.getUrl())))
    .forEach(ThrowingConsumer.unchecked(uri -> Files.writeString(Path.of(uri))));
```

### 8.16.3 Solución alternativa: @SneakyThrows (Lombok)

```java
import lombok.SneakyThrows;

// Lombok oculta la checked exception en el bytecode
public class ConLombok {
    @SneakyThrows
    private static URI convertir(String s) {
        return new URI(s);
    }

    public List<URI> procesar(List<Producto> productos) {
        return productos.stream()
            .map(p -> convertir(p.getUrl()))
            .collect(Collectors.toList());
    }
}

// Precaución: @SneakyThrows viola el contrato de checked exceptions
// El llamador no sabrá que puede lanzar URISyntaxException
// Usar solo en contextos internos, no en APIs públicas
```

### 8.16.4 Solución alternativa: librería Vavr (Try monádico)

```java
import io.vavr.control.Try;

// Vavr: Try es una mónada que contiene éxito o excepción
List<URI> uris = productos.stream()
    .map(p -> Try.of(() -> new URI(p.getUrl())))
    .filter(Try::isSuccess)
    .map(Try::get)
    .collect(Collectors.toList());

// O con flatMap para manejar el error sin RuntimeException
List<String> resultados = productos.stream()
    .map(p -> Try.of(() -> new URI(p.getUrl())))
    .map(tryUri -> tryUri
        .map(URI::toString)
        .getOrElse("URL inválida"))
    .collect(Collectors.toList());
```

### 8.16.5 Recomendación práctica

Para equipos sin Lombok ni Vavr, la opción más pragmática es definir `ThrowingFunction` y `ThrowingConsumer` en una clase de utilidad interna del proyecto. Son ~10 líneas de código y evitan `try-catch` por todo el pipeline.

---

## 8.17 Resumen del Pipeline

Ejemplo completo que combina lambdas, interfaces funcionales, streams y collectors en un escenario realista:

```java
public class ReporteInventario {

    public record ResumenCategoria(
        String categoria,
        long totalProductos,
        double precioPromedio,
        int stockTotal,
        Producto masCaro
    ) {}

    public Map<String, ResumenCategoria> generarReporte(List<Producto> productos) {
        return productos.stream()
            .filter(p -> p.getPrecio() > 0)
            .collect(Collectors.groupingBy(
                Producto::getCategoria,
                TreeMap::new,
                Collectors.collectingAndThen(
                    Collectors.toList(),
                    lista -> {
                        long total = lista.size();
                        double precioProm = lista.stream()
                            .mapToDouble(Producto::getPrecio).average().orElse(0.0);
                        int stockTotal = lista.stream()
                            .mapToInt(Producto::getStock).sum();
                        Producto masCaro = lista.stream()
                            .max(Comparator.comparing(Producto::getPrecio)).orElse(null);
                        return new ResumenCategoria(
                            lista.get(0).getCategoria(),
                            total, precioProm, stockTotal, masCaro
                        );
                    }
                )
            ));
    }
}
```

---

## 8.18 Ejercicios Propuestos

1. **Filtrado y mapeo**: Dada una lista de `Empleado` (nombre, salario, departamento), obtén los nombres en mayúsculas de los empleados que ganan más de 50,000 ordenados alfabéticamente.

2. **Agrupación**: Agrupa los empleados por departamento y calcula el salario promedio por departamento.

3. **Partición**: Divide una lista de estudiantes (nombre, nota) en aprobados (nota >= 6) y suspendidos.

4. **Reducción personalizada**: Implementa un reducer que encuentre el empleado con mayor salario en cada departamento usando `groupingBy` con `reducing`.

5. **Optional**: Refactoriza el siguiente código con Optional para eliminar los null checks:
   ```java
   public String getCodigoPostal(Cliente c) {
       if (c != null && c.getDireccion() != null
           && c.getDireccion().getCodigoPostal() != null) {
           return c.getDireccion().getCodigoPostal();
       }
       return "00000";
   }
   ```

6. **flatMap**: Dada una lista de `Curso` (cada curso tiene una lista de `Estudiante`), obtén la lista plana de todos los estudiantes cuyo nombre empieza con "A".

7. **Streams paralelos**: Mide la diferencia de rendimiento entre `stream()` y `parallelStream()` para calcular la suma de los cuadrados de los primeros 10 millones de números.

8. **Collector personalizado**: Implementa un `Collector` que agrupe strings por su primera letra produciendo un `Map<Character, Set<String>>`.

9. **Spliterator propio**: Crea un `Spliterator` que genere números triangulares (1, 3, 6, 10, 15, ...) y exponlos como Stream.

10. **Manejo de excepciones**: Implementa una función `ThrowingFunction` y úsala para procesar un stream de URLs que puedan ser inválidas, capturando los errores sin `try-catch` en cada lambda.

---

## 8.19 Resumen Visual del Pipeline de Stream

```
             Operaciones Intermedias (lazy)                     Operación Terminal
             ─────────────────────────────                     ─────────────────
            ┌──────┐   ┌─────┐   ┌──────┐   ┌──────────┐
   fuente → │filter│ → │ map │ → │sorted│ → │takeWhile │ → ... → collect/forEach/reduce → resultado
            └──────┘   └─────┘   └──────┘   └──────────┘
                ↑                     ↑              ↑
           Stateless            Stateful      Corto-circuito

Características:
  • No se ejecutan hasta la terminal (evaluación perezosa)
  • Loop fusion: cada elemento recorre todo el pipeline
  • El stream se consume una sola vez (IllegalStateException si se reutiliza)
  • La fuente original no se modifica (inmutabilidad)
  • Paralelizable con .parallel() y .parallelStream()
  • Spliterator gobierna la división del trabajo en paralelo
```

---

**Conceptos clave de este capítulo:**

- Las **lambdas** permiten pasar comportamiento como parámetro, eliminando el código repetitivo de clases anónimas. Usan `invokedynamic` para máxima eficiencia.
- Las **interfaces funcionales** de `java.util.function` cubren la mayoría de casos: `Predicate`, `Function`, `Consumer`, `Supplier` y especializaciones primitivas.
- Las **referencias a métodos** (`String::length`, `System.out::println`) hacen el código aún más conciso. Siempre que puedas, prefiérelas sobre lambdas equivalentes.
- Los **Streams** son pipelines declarativos: una fuente, cero o más operaciones intermedias perezosas, y una operación terminal que dispara la ejecución.
- El **Spliterator** es el motor oculto del paralelismo: divide colecciones para su procesamiento concurrente usando `trySplit()`.
- La **evaluación perezosa** y el **corto-circuito** permiten que los streams sean eficientes, procesando solo los elementos necesarios.
- Las operaciones **stateful** (`sorted`, `distinct`) deben usarse con moderación: bloquean el pipeline y penalizan el paralelismo.
- **`Optional`** es una mónada que reemplaza los null checks con un API funcional. Nuevas características en Java 9+ (`ifPresentOrElse`, `or`, `stream`) amplían su utilidad.
- **`Collectors`** ofrece recolecciones predefinidas para listas, conjuntos, mapas, agrupaciones, particiones y estadísticas. `groupingBy` es equivalente a `GROUP BY` de SQL.
- Los **streams paralelos** aceleran operaciones CPU-intensivas sobre grandes volúmenes, pero comparten el `ForkJoinPool` común y requieren cuidado con el estado compartido.
- Los **streams primitivos** (`IntStream`, `LongStream`, `DoubleStream`) eliminan el autoboxing y son 4-5x más rápidos que `Stream<Integer>`.
- Las **excepciones verificadas** no encajan en lambdas. Usa wrappers (`ThrowingFunction`), `@SneakyThrows` o librerías como Vavr para manejarlas limpiamente.
- Los **streams infinitos** (`Stream.iterate`, `Stream.generate`) combinados con `takeWhile`/`limit` modelan secuencias ilimitadas de forma segura.
