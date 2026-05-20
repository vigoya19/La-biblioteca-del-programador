# Capítulo 4: Genéricos y el Collection Framework

---

## 4.1 Introducción a Genéricos

Los genéricos (*generics*) se introdujeron en Java 5 para proporcionar **seguridad de tipos en tiempo de compilación** y eliminar la necesidad de *casts* explícitos. Antes de los genéricos, las colecciones almacenaban `Object`, lo que obligaba al programador a hacer *casting* manual y exponía el código a errores en tiempo de ejecución.

### 4.1.1 El problema de los tipos crudos (*raw types*)

Sin genéricos, cualquier tipo de objeto podía añadirse a una colección:

```java
// Antes de Java 5: tipos crudos (raw types)
List lista = new ArrayList();
lista.add("Hola");
lista.add(42);             // Permitido, sin error de compilación
lista.add(new Date());     // Permitido, sin error de compilación

// Al recuperar, se requiere casting explícito
String texto = (String) lista.get(0);  // OK
String error = (String) lista.get(1);  // ClassCastException en tiempo de ejecución
```

La línea `String error = (String) lista.get(1)` compila sin problemas, pero lanza una `ClassCastException` en tiempo de ejecución porque el elemento en la posición 1 es un `Integer`, no un `String`. Este es precisamente el problema que los genéricos resuelven.

### 4.1.2 Clases genéricas

Una clase genérica declara uno o más parámetros de tipo entre corchetes angulares `<>`:

```java
// Declaración de una clase genérica simple
public class Caja<T> {
    private T contenido;

    public void guardar(T contenido) {
        this.contenido = contenido;
    }

    public T obtener() {
        return contenido;
    }
}

// Uso de la clase genérica
Caja<String> cajaTexto = new Caja<>();
cajaTexto.guardar("Hola mundo");
String texto = cajaTexto.obtener();    // No necesita casting

Caja<Integer> cajaNumero = new Caja<>();
cajaNumero.guardar(42);
// cajaNumero.guardar("texto");        // Error de compilación: tipo incompatible
```

La seguridad de tipos se aplica en tiempo de compilación: el compilador rechaza cualquier intento de asignar un tipo incorrecto.

**Clase genérica con múltiples parámetros de tipo:**

```java
public class Par<K, V> {
    private K clave;
    private V valor;

    public Par(K clave, V valor) {
        this.clave = clave;
        this.valor = valor;
    }

    public K getClave() { return clave; }
    public V getValor() { return valor; }
}

Par<String, Integer> par = new Par<>("edad", 30);
```

### 4.1.3 Métodos genéricos

Un método puede declarar sus propios parámetros de tipo, independientemente de si la clase es genérica o no:

```java
public class Util {

    // Método genérico: el parámetro <T> se declara antes del tipo de retorno
    public static <T> T obtenerPrimero(List<T> lista) {
        if (lista == null || lista.isEmpty()) {
            return null;
        }
        return lista.get(0);
    }

    // Método genérico que intercambia elementos de un array
    public static <T> void intercambiar(T[] array, int i, int j) {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }

    // Método genérico que imprime cualquier tipo de array
    public static <T> void imprimirArray(T[] array) {
        for (T elemento : array) {
            System.out.print(elemento + " ");
        }
        System.out.println();
    }
}

// Uso
List<String> nombres = List.of("Ana", "Luis", "Carlos");
String primero = Util.obtenerPrimero(nombres);   // Inferencia de tipo automática

Integer[] numeros = {1, 2, 3, 4, 5};
Util.intercambiar(numeros, 0, 4);
Util.imprimirArray(numeros);                     // 5 2 3 4 1
```

### 4.1.4 Parámetros de tipo acotados (*Bounded Type Parameters*)

Es posible restringir qué tipos pueden usarse como argumento de tipo mediante la palabra clave `extends`:

```java
// Solo acepta tipos que sean Number o subclases de Number
public class Calculadora<T extends Number> {
    private T numero;

    public Calculadora(T numero) {
        this.numero = numero;
    }

    public double obtenerDoble() {
        return numero.doubleValue();  // doubleValue() está disponible porque T extiende Number
    }
}

Calculadora<Integer> calcInt = new Calculadora<>(10);      // OK: Integer extiende Number
Calculadora<Double> calcDouble = new Calculadora<>(3.14);  // OK: Double extiende Number
// Calculadora<String> calcStr = new Calculadora<>("abc"); // Error de compilación
```

**Acotamiento múltiple:**

```java
// T debe implementar Comparable y ser subclase de Number
public class NumeroOrdenado<T extends Number & Comparable<T>> {
    private T valor;

    public NumeroOrdenado(T valor) {
        this.valor = valor;
    }
}
```

**Nota importante sobre acotamiento múltiple:** Si se usa una clase y una o más interfaces, la clase debe aparecer primero:

```java
class MiClase { }
interface MiInterfaz1 { }
interface MiInterfaz2 { }

// Correcto: clase primero, luego interfaces
class Contenedor<T extends MiClase & MiInterfaz1 & MiInterfaz2> { }

// Incorrecto: no compila
// class Contenedor<T extends MiInterfaz1 & MiClase> { }
```

### 4.1.5 Inferencia de tipos y el operador diamante (`<>`)

Desde Java 7, el operador diamante (`<>`) permite que el compilador infiera los argumentos de tipo del contexto, eliminando la redundancia:

```java
// Antes de Java 7: redundante
List<String> lista = new ArrayList<String>();
Map<String, List<Integer>> mapa = new HashMap<String, List<Integer>>();

// Con diamante (Java 7+): el compilador infiere los tipos
List<String> lista = new ArrayList<>();
Map<String, List<Integer>> mapa = new HashMap<>();
```

**El diamante y las clases anónimas (Java 9+):**

```java
// Java 9+ permite el diamante en clases anónimas
Callable<String> tarea = new Callable<>() {
    @Override
    public String call() throws Exception {
        return "Hola";
    }
};
```

### 4.1.6 Wildcards (Comodines)

Los wildcards (`?`) representan un tipo desconocido y se usan para hacer el código genérico más flexible.

**Unbounded wildcard `<?>`:**

Se usa cuando el código funciona con cualquier tipo y no necesita conocerlo:

```java
// Método que imprime cualquier lista, sin importar su tipo
public static void imprimirLista(List<?> lista) {
    for (Object elemento : lista) {
        System.out.println(elemento);
    }
}

// Se puede usar con listas de cualquier tipo
List<String> nombres = List.of("Ana", "Luis");
List<Integer> edades = List.of(25, 30);
imprimirLista(nombres);  // Funciona
imprimirLista(edades);   // Funciona
```

**Upper bounded wildcard `<? extends T>`:**

Acepta `T` o cualquier subtipo de `T`. Es útil para **leer** datos de una estructura:

```java
// Método que calcula la suma de una lista de números
public static double sumar(List<? extends Number> numeros) {
    double suma = 0.0;
    for (Number n : numeros) {
        suma += n.doubleValue();  // Se puede LEER como Number
    }
    return suma;
}

List<Integer> enteros = List.of(1, 2, 3, 4, 5);
List<Double> decimales = List.of(1.5, 2.5, 3.5);

System.out.println(sumar(enteros));    // 15.0
System.out.println(sumar(decimales));  // 7.5
```

**Nota importante:** Con `<? extends T>` **no se puede escribir** en la colección, porque el tipo exacto es desconocido. El compilador no puede garantizar la seguridad de tipos en la escritura:

```java
List<? extends Number> lista = new ArrayList<Integer>();
// lista.add(10);              // Error de compilación: no se puede añadir
// lista.add(new Integer(5));  // Error de compilación
// lista.add(new Double(5.0)); // Error de compilación
Number n = lista.get(0);       // Solo se permite lectura
```

**Lower bounded wildcard `<? super T>`:**

Acepta `T` o cualquier supertipo de `T`. Es útil para **escribir** datos en una estructura:

```java
// Método que añade números a una lista
public static void agregarNumeros(List<? super Integer> lista) {
    for (int i = 1; i <= 5; i++) {
        lista.add(i);  // Se puede ESCRIBIR Integer
    }
}

List<Number> numeros = new ArrayList<>();
List<Object> objetos = new ArrayList<>();

agregarNumeros(numeros);  // OK: Number es supertipo de Integer
agregarNumeros(objetos);  // OK: Object es supertipo de Integer
```

**Nota importante:** Con `<? super T>` se puede escribir, pero **la lectura devuelve `Object`** porque el tipo exacto es desconocido hacia arriba:

```java
List<? super Integer> lista = new ArrayList<Number>();
lista.add(42);               // Se puede escribir
Object obj = lista.get(0);   // Lectura devuelve Object, no Integer
```

### 4.1.7 Principio PECS (Producer Extends, Consumer Super)

El principio PECS es una regla mnemotécnica para decidir entre `<? extends T>` y `<? super T>`:

- **Producer Extends**: Si la estructura **produce** valores que se van a leer, usa `<? extends T>`. (El código obtiene valores *de* la estructura).
- **Consumer Super**: Si la estructura **consume** valores que se van a escribir, usa `<? super T>`. (El código pone valores *en* la estructura).

```java
// PECS en acción: copy(destino, origen)
//   - origen produce elementos → <? extends T>
//   - destino consume elementos → <? super T>
public static <T> void copiar(List<? super T> destino, List<? extends T> origen) {
    for (T elemento : origen) {         // origen PRODUCE elementos
        destino.add(elemento);          // destino CONSUME elementos
    }
}

List<Number> numeros = new ArrayList<>();
List<Integer> enteros = List.of(1, 2, 3);
copiar(numeros, enteros);  // Copia Integer a Number: OK
```

**Tabla de decisión PECS:**

| Escenario | Wildcard | Significado |
|-----------|----------|-------------|
| Solo leer datos | `<? extends T>` | Acepta T y subtipos de T |
| Solo escribir datos | `<? super T>` | Acepta T y supertipos de T |
| Leer y escribir | `<T>` (sin wildcard) | Tipo exacto T |
| No importa el tipo | `<?>` | Cualquier tipo |

### 4.1.8 Wildcards avanzados

**Captura de wildcards:**

La "captura de wildcards" es una situación que ocurre cuando el compilador no puede inferir el tipo concreto detrás de un wildcard. Esto genera errores de compilación confusos que se resuelven con un *helper method* privado:

```java
// Este método NO compila: el compilador no puede determinar
// que el elemento que sale de la lista es del mismo tipo que
// el elemento que entra
public static void intercambiarErroneo(List<?> lista, int i, int j) {
    // lista.set(i, lista.set(j, lista.get(i))); // ERROR
}

// Solución: helper method genérico privado que "captura" el wildcard
public static void intercambiar(List<?> lista, int i, int j) {
    intercambiarHelper(lista, i, j);
}

// El compilador captura el tipo concreto ? y lo asigna a T
private static <T> void intercambiarHelper(List<T> lista, int i, int j) {
    T temp = lista.get(i);
    lista.set(i, lista.get(j));
    lista.set(j, temp);
}
```

**Wildcards anidados:**

Los wildcards pueden anidarse para expresar restricciones complejas. Aunque es un patrón avanzado y poco común, es importante conocerlo:

```java
// Una lista de listas de números: acepta List<List<Integer>>, List<List<Double>>, etc.
public static void procesarListaDeListas(List<? extends List<? extends Number>> lista) {
    for (List<? extends Number> sublista : lista) {
        for (Number n : sublista) {
            System.out.println(n);
        }
    }
}

// Uso
List<List<Integer>> listasEnteros = new ArrayList<>();
listasEnteros.add(List.of(1, 2, 3));
listasEnteros.add(List.of(4, 5, 6));
procesarListaDeListas(listasEnteros); // OK

List<List<Double>> listasDecimales = new ArrayList<>();
listasDecimales.add(List.of(1.1, 2.2));
procesarListaDeListas(listasDecimales); // OK
```

**Diferencia entre `List<?>` y `List<Object>`:**

Muchos principiantes confunden estos tipos. Son fundamentalmente diferentes:

```java
public static void demo() {
    List<Object> objetos = new ArrayList<>();
    List<String> strings = List.of("a", "b");
    List<?> wildcard = new ArrayList<>();

    objetos.add("Hola");       // OK: String es Object
    objetos.add(42);           // OK: Integer es Object

    // objetos = strings;      // ERROR: List<Object> NO es supertipo de List<String>
    wildcard = strings;        // OK: List<?> acepta cualquier List parametrizado

    // wildcard.add("Hola");   // ERROR: no se puede escribir en List<?>
    Object o = wildcard.get(0); // OK: get devuelve Object
}
```
### 4.1.9 Type Erasure (Borrado de tipos) — en profundidad

Los genéricos en Java se implementan mediante **borrado de tipos** (*type erasure*). Esta fue una decisión de diseño deliberada para mantener compatibilidad hacia atrás con el código pre-Java 5. A diferencia de C# (que usa *reified generics*), en Java los genéricos son puramente una construcción de tiempo de compilación.

#### Cómo funciona el borrado de tipos

El compilador aplica las siguientes transformaciones durante la compilación:

1. **Reemplazo de parámetros de tipo por su límite superior:**
   - Si el parámetro no tiene límite (`<T>`), se reemplaza por `Object`.
   - Si tiene límite (`<T extends Number>`), se reemplaza por `Number`.

2. **Inserción de casts:** Se insertan *casts* en los puntos donde se recuperan valores para garantizar la seguridad de tipos.

3. **Generación de bridge methods:** Se crean métodos puente para preservar el polimorfismo.

```java
// Código fuente
public class Caja<T> {
    private T contenido;
    public void guardar(T contenido) { this.contenido = contenido; }
    public T obtener() { return contenido; }
}

Caja<String> caja = new Caja<>();
caja.guardar("Hola");
String texto = caja.obtener();

// Después del borrado de tipos (bytecode equivalente)
public class Caja {
    private Object contenido;
    public void guardar(Object contenido) { this.contenido = contenido; }
    public Object obtener() { return contenido; }
}

Caja caja = new Caja();
caja.guardar("Hola");
String texto = (String) caja.obtener(); // El compilador inserta este cast
```

#### Qué información se pierde y qué se conserva

| Se conserva en tiempo de ejecución | Se pierde en tiempo de ejecución |
|------------------------------------|----------------------------------|
| La clase `Caja` (raw type) | `Caja<String>` vs `Caja<Integer>`: son la misma clase |
| Tipos acotados en declaración de clase | Parámetros de tipo de variables locales |
| Tipos genéricos en campos, métodos y superclases (vía reflection) | Información de instancias individuales |

#### Verificación del borrado de tipos

```java
List<String> listaStrings = new ArrayList<>();
List<Integer> listaIntegers = new ArrayList<>();

System.out.println(listaStrings.getClass() == listaIntegers.getClass()); // true
System.out.println(listaStrings.getClass()); // class java.util.ArrayList
```

#### Cómo el borrado afecta a la reflection

Aunque los parámetros de tipo de las instancias se borran, la información genérica de la **declaración** de clases, campos y métodos sí se conserva en los archivos `.class` (vía la propiedad `Signature` en el bytecode) y es accesible mediante `java.lang.reflect`:

```java
import java.lang.reflect.*;
import java.util.*;

public class AnalisisTypeErasure {
    private List<String> listaStrings;
    private Map<Integer, String> mapa;

    public List<String> procesar(List<Integer> entrada) {
        return List.of("a", "b");
    }

    public static void main(String[] args) throws Exception {
        Class<?> clazz = AnalisisTypeErasure.class;

        System.out.println("=== Campos genéricos ===");
        for (Field field : clazz.getDeclaredFields()) {
            Type tipoGenerico = field.getGenericType();
            if (tipoGenerico instanceof ParameterizedType) {
                ParameterizedType pt = (ParameterizedType) tipoGenerico;
                System.out.println("Campo: " + field.getName());
                System.out.println("  Tipo raw: " + pt.getRawType());
                System.out.println("  Argumentos de tipo: " +
                    Arrays.toString(pt.getActualTypeArguments()));
            }
        }

        System.out.println("\n=== Métodos genéricos ===");
        for (Method method : clazz.getDeclaredMethods()) {
            if (method.getName().equals("procesar")) {
                System.out.println("Retorno genérico: " + method.getGenericReturnType());
                for (Type tp : method.getGenericParameterTypes()) {
                    System.out.println("Parámetro genérico: " + tp);
                }
            }
        }
    }
}
```

**Salida esperada:**

```
=== Campos genéricos ===
Campo: listaStrings
  Tipo raw: interface java.util.List
  Argumentos de tipo: [class java.lang.String]
Campo: mapa
  Tipo raw: interface java.util.Map
  Argumentos de tipo: [class java.lang.Integer, class java.lang.String]

=== Métodos genéricos ===
Retorno genérico: java.util.List<java.lang.String>
Parámetro genérico: java.util.List<java.lang.Integer>
```

**¡Cuidado!** La información genérica está disponible a nivel de **declaración de clase/método**, pero NO a nivel de **instancia individual**:

```java
List<String> miLista = new ArrayList<>();
// En tiempo de ejecución solo es ArrayList...
// No hay forma de saber que era List<String> desde la instancia misma
```

### 4.1.10 Bridge Methods

Los *bridge methods* (métodos puente) son métodos sintéticos que el compilador genera automáticamente para mantener el polimorfismo correcto después del borrado de tipos.

**Problema que resuelven:**

Imagina una clase que implementa `Comparable` de forma genérica:

```java
// Código fuente
public class Persona implements Comparable<Persona> {
    private String nombre;

    @Override
    public int compareTo(Persona otra) {
        return this.nombre.compareTo(otra.nombre);
    }
}

// Después del borrado, Comparable<Persona> se convierte en Comparable.
// La interfaz Comparable declara: public int compareTo(Object o)
// Pero Persona tiene: public int compareTo(Persona otra)
// ¡Las firmas no coinciden! Se necesita un bridge method.
```

**Lo que el compilador genera realmente:**

```java
// Bytecode generado (conceptualmente):
public class Persona implements Comparable {
    // Método original que escribiste
    public int compareTo(Persona otra) {
        return this.nombre.compareTo(otra.nombre);
    }

    // BRIDGE METHOD generado automáticamente
    public int compareTo(Object otra) {
        return this.compareTo((Persona) otra);
    }
}
```

**Cómo ver los bridge methods:**

```java
for (Method m : Persona.class.getDeclaredMethods()) {
    System.out.println(m.getName() + " -> " + Arrays.toString(m.getParameterTypes()));
    System.out.println("  Es bridge: " + m.isBridge());
    System.out.println("  Es sintético: " + m.isSynthetic());
}
// Salida:
// compareTo -> [class Persona]
//   Es bridge: false       ← el que escribiste
//   Es sintético: false
// compareTo -> [class Object]
//   Es bridge: true        ← generado automáticamente
//   Es sintético: true
```

**Problemas que pueden causar los bridge methods:**

1. **Reflection confusa:** Puedes encontrar métodos duplicados al inspeccionar con reflection.
2. **Errores en frameworks:** Spring o Hibernate que inspeccionan métodos vía reflection deben manejar bridge methods (filtrando con `!method.isBridge()`).
3. **Anotaciones duplicadas:** A veces el bridge method hereda anotaciones del método original.

### 4.1.11 Heap Pollution

*Heap pollution* es una situación en la que una variable de tipo parametrizado referencia un objeto cuyo tipo no coincide con el parámetro de tipo declarado. Ocurre típicamente al mezclar tipos crudos (*raw types*) con tipos genéricos, o con varargs genéricos.

**Ejemplo clásico de heap pollution:**

```java
List<String> listaStrings = new ArrayList<>();
listaStrings.add("Hola");

List raw = listaStrings;  // unchecked warning
raw.add(42);  // Heap pollution: la lista "de Strings" ahora contiene un Integer

// String s = listaStrings.get(1); // ClassCastException en tiempo de ejecución
System.out.println("Tamaño: " + listaStrings.size()); // 2 — sin error aún
```

**Heap pollution con varargs genéricos:**

```java
// PELIGROSO: mezcla varargs con parámetros de tipo
public static <T> void agregarALista(List<T> lista, T... elementos) {
    for (T elem : elementos) {
        lista.add(elem);
    }
}

public static void main(String[] args) {
    List<String> lista = new ArrayList<>();
    agregarALista(lista, "A", "B", "C"); // OK

    Object[] datos = {42, 99};
    // agregarALista(lista, (String[]) datos); // ClassCastException — el cast falla
}
```

#### @SafeVarargs

La anotación `@SafeVarargs` (Java 7+) suprime los *unchecked warnings* en métodos que usan varargs genéricos. **Solo debe usarse cuando el método realmente es seguro**:

1. El método no almacena nada en el array varargs (solo lee).
2. El método no expone el array varargs a código externo.

```java
// SEGURO: solo copia elementos a una lista
@SafeVarargs
public static <T> List<T> crearLista(T... elementos) {
    List<T> resultado = new ArrayList<>();
    for (T e : elementos) resultado.add(e);
    return resultado;
}

// PELIGROSO: modifica el array varargs
@SuppressWarnings("unchecked") // NUNCA uses @SafeVarargs aquí
public static <T> void metodoPeligroso(T... elementos) {
    elementos[0] = null; // Potencial heap pollution
}
```

**Reglas de @SafeVarargs:**
- Solo en métodos `static`, `final`, o `private` (desde Java 9).
- El método no debe modificar el contenido del array varargs.
- El método no debe exponer el array varargs fuera de sí mismo.

### 4.1.12 Genéricos y Arrays

Una de las restricciones más confusas de los genéricos en Java es que **no puedes crear arrays de tipos genéricos**. Esta limitación deriva del type erasure y de cómo los arrays manejan los tipos en tiempo de ejecución.

**Arrays vs Genéricos: una comparación fundamental**

| Característica | Arrays | Genéricos |
|----------------|--------|-----------|
| Verificación de tipos | En tiempo de ejecución | En tiempo de compilación |
| Covarianza | `String[]` es subtipo de `Object[]` | `List<String>` NO es subtipo de `List<Object>` |
| Type erasure | No aplica (tipos reificados) | Sí (tipos borrados) |
| Runtime type check | `array instanceof String[]` | No disponible para parámetros de tipo |

**Por qué no puedes crear `new T[10]`:**

```java
// NO compila:
// public class Caja<T> {
//     // private T[] elementos = new T[10]; // ERROR
// }

// Explicación: Si new T[10] compilara, después del type erasure sería new Object[10].
// Una Caja<String> tendría un Object[] interno, pero getter cast a String[].
// ClassCastException garantizada.
```

**Por qué no puedes crear `new List<String>[10]`:**

```java
// List<String>[] array = new List<String>[10]; // ERROR: generic array creation

// Si fuera permitido:
// Object[] objetos = array;  // OK porque array es Object[]
// objetos[0] = new ArrayList<Integer>(); // Heap pollution
// String s = array[0].get(0); // ClassCastException — es Integer, no String
```

**Soluciones y workarounds:**

```java
// Estrategia 1: Usar colecciones en vez de arrays
List<List<String>> listaDeListas = new ArrayList<>();

// Estrategia 2: Crear array via reflection
@SuppressWarnings("unchecked")
public static <T> T[] crearArrayGenerico(Class<T> tipo, int tamano) {
    return (T[]) java.lang.reflect.Array.newInstance(tipo, tamano);
}

// Estrategia 3: Pasar el array desde fuera (patrón receptor)
public static <T> T[] toArray(Collection<T> col, T[] array) {
    return col.toArray(array);
}

// Estrategia 4: Usar Object[] internamente (como hace ArrayList)
public class ListaGenerica<T> {
    private Object[] elementos;
    private int size;

    public ListaGenerica() { elementos = new Object[10]; }

    @SuppressWarnings("unchecked")
    public T get(int i) { return (T) elementos[i]; }

    public void add(T elemento) { elementos[size++] = elemento; }
}
```

### 4.1.13 Ejemplos reales de genéricos

**Ejemplo 1: Sistema de eventos genérico**

```java
interface Evento { String descripcion(); }

class EventoUsuarioCreado implements Evento {
    private final String nombre;
    public EventoUsuarioCreado(String nombre) { this.nombre = nombre; }
    @Override public String descripcion() { return "Usuario creado: " + nombre; }
    public String getNombre() { return nombre; }
}

class EventoPagoProcesado implements Evento {
    private final double monto;
    public EventoPagoProcesado(double monto) { this.monto = monto; }
    @Override public String descripcion() { return "Pago procesado: $" + monto; }
    public double getMonto() { return monto; }
}

public class SistemaEventos<E extends Evento> {
    private final Map<Class<? extends E>, List<Consumer<? extends E>>> manejadores = new HashMap<>();

    public <T extends E> void registrar(Class<T> tipo, Consumer<T> manejador) {
        manejadores.computeIfAbsent(tipo, k -> new ArrayList<>()).add(manejador);
    }

    @SuppressWarnings("unchecked")
    public <T extends E> void enviar(T evento) {
        manejadores.getOrDefault(evento.getClass(), List.of())
                   .forEach(m -> ((Consumer<T>) m).accept(evento));
    }
}
```

**Ejemplo 2: Repositorio CRUD genérico**

```java
interface Entidad<I> { I getId(); }

class Producto implements Entidad<Long> {
    private Long id;
    private String nombre;
    private double precio;
    public Producto(Long id, String nombre, double precio) {
        this.id = id; this.nombre = nombre; this.precio = precio;
    }
    @Override public Long getId() { return id; }
    public String getNombre() { return nombre; }
    public double getPrecio() { return precio; }
}

class Usuario implements Entidad<String> {
    private String email, nombre;
    public Usuario(String email, String nombre) {
        this.email = email; this.nombre = nombre;
    }
    @Override public String getId() { return email; }
    public String getNombre() { return nombre; }
}

public class Repositorio<T extends Entidad<I>, I> {
    private final Map<I, T> almacen = new HashMap<>();

    public void guardar(T e) { almacen.put(e.getId(), e); }
    public Optional<T> buscarPorId(I id) { return Optional.ofNullable(almacen.get(id)); }
    public List<T> buscarTodas() { return new ArrayList<>(almacen.values()); }
    public List<T> filtrar(Predicate<T> cond) {
        return almacen.values().stream().filter(cond).collect(Collectors.toList());
    }
    public void eliminar(I id) { almacen.remove(id); }

    public static void main(String[] args) {
        Repositorio<Producto, Long> repoProd = new Repositorio<>();
        repoProd.guardar(new Producto(1L, "Laptop", 999.99));
        repoProd.guardar(new Producto(2L, "Mouse", 29.99));

        List<Producto> caros = repoProd.filtrar(p -> p.getPrecio() > 50);
        System.out.println("Productos caros: " + caros);
        // [Producto{id=1, nombre='Laptop', precio=999.99}]

        Repositorio<Usuario, String> repoUsr = new Repositorio<>();
        repoUsr.guardar(new Usuario("ana@email.com", "Ana"));
        System.out.println(repoUsr.buscarPorId("ana@email.com").orElse(null));
    }
}
```

---

## 4.2 El Collection Framework

El *Collection Framework* proporciona una arquitectura unificada para almacenar y manipular grupos de objetos.

### 4.2.1 Jerarquía de interfaces

```
                           Iterable
                              |
                         Collection
                        /    |     \
                      List  Set   Queue
                       |     |      |
                    [varias implementaciones]

                           Map
                        /   |   \
                  HashMap  SortedMap  ConcurrentMap
                             |
                         NavigableMap
```

**Interfaces principales:**

| Interfaz | Descripción | Duplicados | Orden | Claves/Valores |
|----------|-------------|------------|-------|----------------|
| `Collection` | Raíz de la jerarquía | Depende | Depende | Solo valores |
| `List` | Secuencia ordenada, acceso por índice | Sí | Mantiene inserción | Solo valores |
| `Set` | Conjunto sin duplicados | No | Depende | Solo valores |
| `Queue` | Cola FIFO | Sí | FIFO o prioridad | Solo valores |
| `Deque` | Cola de doble extremo | Sí | Ambos extremos | Solo valores |
| `Map` | Pares clave-valor | Claves: No | Depende | Claves y valores |
| `SortedSet` | Set ordenado | No | Orden natural/Comparator | Solo valores |
| `SortedMap` | Map ordenado por claves | Claves: No | Orden natural/Comparator | Claves y valores |

### 4.2.2 Implementaciones más comunes

**List:**
| Implementación | Estructura interna | Características |
|----------------|-------------------|-----------------|
| `ArrayList` | Array redimensionable | Acceso aleatorio O(1) |
| `LinkedList` | Lista doblemente enlazada | Inserción/eliminación O(1) en extremos |
| `Vector` | Array redimensionable sincronizado | Legado. Preferir `ArrayList` |

**Set:**
| Implementación | Estructura interna | Características |
|----------------|-------------------|-----------------|
| `HashSet` | HashMap interno | O(1) promedio. Sin orden |
| `LinkedHashSet` | HashMap + lista enlazada | O(1). Preserva orden de inserción |
| `TreeSet` | Árbol rojo-negro | O(log n). Ordenado |

**Map:**
| Implementación | Estructura interna | Características |
|----------------|-------------------|-----------------|
| `HashMap` | Tabla hash | O(1) promedio. Sin orden |
| `LinkedHashMap` | Tabla hash + lista enlazada | O(1). Preserva orden de inserción |
| `TreeMap` | Árbol rojo-negro | O(log n). Ordenado por claves |
| `Hashtable` | Tabla hash sincronizada | Legado. Preferir `ConcurrentHashMap` |

### 4.2.3 Interfaces vs Implementaciones

```java
// Buena práctica: declarar con la interfaz, instanciar con la implementación
List<String> lista = new ArrayList<>();
Set<Integer> conjunto = new HashSet<>();
Map<String, Integer> mapa = new HashMap<>();
// Facilita cambiar de implementación sin modificar el resto del código
```

---

## 4.3 List

### 4.3.1 ArrayList — en profundidad

`ArrayList` usa un **array redimensionable** interno. Cuando el array se llena, se crea uno nuevo +50% de capacidad y se copian los elementos.

#### Cómo crece internamente

```java
// Código real del JDK (simplificado)
public class ArrayList<E> {
    transient Object[] elementData;
    private int size;
    private static final int DEFAULT_CAPACITY = 10;

    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA; // Array vacío lazy
    }

    public boolean add(E e) {
        if (size == elementData.length) elementData = grow();
        elementData[size++] = e;
        return true;
    }

    private Object[] grow() {
        int old = elementData.length;
        // Si es el primer add: usa DEFAULT_CAPACITY (10)
        // Si no: oldCapacity + oldCapacity/2 = old * 1.5
        int newCap = (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            ? Math.max(DEFAULT_CAPACITY, size + 1)
            : old + (old >> 1);
        return elementData = Arrays.copyOf(elementData, newCap);
    }
}
```

**Secuencia de crecimiento:**

| Operación | elementData.length | size | Nota |
|-----------|-------------------|------|------|
| `new ArrayList<>()` | 0 | 0 | Array perezoso |
| `add(A)` | 10 | 1 | Primera inserción expande a 10 |
| 9 adds más (B-J) | 10 | 10 | Sin crecer |
| `add(K)` | 15 | 11 | Crece a 15 (+50%) |
| `add(P)` | 22 | 16 | Crece a 22 |
| `add(W)` | 33 | 23 | Crece a 33 |

Con 1.000.000 de elementos: **~20 redimensionamientos** y **~1.5 millones de copias** de elementos.

#### Optimización: especificar capacidad inicial

```java
// INEFICIENTE: múltiples redimensionamientos
List<String> lista = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) lista.add("Elemento " + i);

// EFICIENTE: un solo array del tamaño correcto
List<String> lista = new ArrayList<>(1_000_000);
for (int i = 0; i < 1_000_000; i++) lista.add("Elemento " + i);
// ~30% más rápido
```

#### trimToSize(): liberar memoria no utilizada

```java
List<String> lista = new ArrayList<>(10_000);
for (int i = 0; i < 10; i++) lista.add("Item " + i);
// Capacidad interna: 10.000, solo 10 elementos usados

((ArrayList<String>) lista).trimToSize();
// Libera 9.990 espacios de array
```

#### Rendimiento: add(E) vs add(int, E)

```
add("E") al final → O(1) amortizado:
  [A, B, C, D, E, _, _] — solo una asignación

add("X", 0) al principio → O(n):
  [X, A, B, C, D, _, _] — desplaza todos los elementos una posición
```

**Resultados JMH en hardware moderno:**

| N | addAlFinal (ns) | addAlPrincipio (ns) | getPorIndice (ns) | contains (ns) |
|---|-----------------|---------------------|-------------------|---------------|
| 1,000 | ~25 | ~600 | ~3 | ~250 |
| 10,000 | ~28 | ~5,800 | ~3 | ~2,500 |
| 100,000 | ~35 | ~58,000 | ~3 | ~25,000 |

**Conclusión:** add(E) al final es entre 1000x-2000x más rápido que add(0,E) para 100K elementos. get() es siempre ~3 ns — acceso directo a memoria.

#### CPU cache y localidad de referencia

```
ArrayList — contiguo en memoria:
┌────────────────────────────────────────────────┐
│ [A] [B] [C] [D] [E] [F] [G] [H] [I] [J] [K]  │
└────────────────────────────────────────────────┘
→ La CPU precarga cache lines completas (64 bytes)
→ Iteración secuencial: ~0.5 ms para 1M elementos

LinkedList — disperso en heap:
┌──────┐     ┌──────┐     ┌──────┐
│ prev │     │ prev │     │ prev │
│ "A"  │     │ "B"  │     │ "C"  │
│ next─┼────>│ next─┼────>│ next─┼───> ...
└──────┘     └──────┘     └──────┘
→ Cada salto causa cache miss (RAM ~100x más lenta que L1)
→ Iteración secuencial: ~5,100 ms para 1M elementos
```

### 4.3.2 LinkedList — en profundidad

`LinkedList` implementa una **lista doblemente enlazada** con nodos que contienen referencias next/prev.

#### Estructura interna de nodos

```java
public class LinkedList<E> {
    transient Node<E> first;
    transient Node<E> last;
    transient int size;

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element; this.next = next; this.prev = prev;
        }
    }
}
```

**Diagrama de nodos:**

```
LinkedList con 4 elementos: "A", "B", "C", "D"

     first                                          last
       │                                             │
       ▼                                             ▼
   ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
   │ prev=null│     │ prev ───┼─────┼── prev   │     │ prev ───┼───┐
   │ item="A"│     │ item="B"│     │ item="C"│     │ item="D"│   │
   │ next ────┼────>│ next ───┼────>│ next ───┼────>│ next=null│  │
   └─────────┘     └─────────┘     └─────────┘     └─────────┘   │
       ▲              │  ▲             │  ▲                        │
       └──────────────┘  └─────────────┘  └────────────────────────┘

Overhead por nodo (JVM 64-bit compressed OOPs):
┌──────────────────────────┐
│ Cabecera: 12 bytes       │
│ prev: 4 bytes            │
│ item: 4 bytes            │
│ next: 4 bytes            │
├──────────────────────────┤
│ Total: 24 bytes por nodo │
└──────────────────────────┘

ArrayList: 4 bytes por elemento (solo referencia)
LinkedList: 24 bytes por elemento → 6x más memoria en overhead
```

**Diagrama de operaciones:**

```
INSERCIÓN addFirst("X") — O(1):
  PASO 1: new Node(null, "X", first)
  PASO 2: first.prev = X
  PASO 3: first = X
  → Solo 3 asignaciones, sin desplazar elementos

ELIMINACIÓN removeFirst() — O(1):
  PASO 1: first = first.next
  PASO 2: first.prev = null
  → Solo 2 asignaciones

ACCESO get(2) — O(n):
  Si index < size/2 → recorre desde first
  Si index >= size/2 → recorre desde last
```

#### Por qué casi SIEMPRE debes usar ArrayList

```java
// RAZÓN 1: get(int) es catastrófico
// ArrayList.get(500K): ~3 ns
// LinkedList.get(500K): ~1,200,000 ns → 400,000x más lento

// RAZÓN 2: add(E) — LinkedList ni siquiera gana
// ArrayList.add(): 1 asignación + ocasional resize (~25 ns am.)
// LinkedList.add(): crear Node + 3 asignaciones (~50 ns)

// RAZÓN 3: add(int, E) en medio
// ArrayList: desplaza referencias (memcpy rápido de CPU)
// LinkedList: navega punteros con cache misses → 25x más lento

// RAZÓN 4: Memoria
// ArrayList: ~4 bytes/elemento (referencia en array)
// LinkedList: ~24 bytes/elemento (objeto Node) → 6x más

// RAZÓN 5: Garbage Collector
// ArrayList: 1 objeto array → GC rápido
// LinkedList: N objetos Node → GC pauses más largas
```

**Los únicos casos donde LinkedList gana:**

- Cola/pila con MUCHAS inserciones al principio (aunque ArrayDeque suele ser mejor).
- Eliminación frecuente durante iteración con Iterator.remove().
- REGLA PRÁCTICA: Si no sabes cuál usar, usa ArrayList. Solo considera LinkedList si un benchmark lo demuestra.

### 4.3.3 Vector y Stack (Legado)

```java
// Vector: ArrayList sincronizado (legado)
Vector<String> v = new Vector<>();

// Stack: pila LIFO, extiende Vector (legado)
Stack<Integer> pila = new Stack<>();
pila.push(1); pila.push(2);
int tope = pila.pop(); // 2

// Alternativas modernas:
// En lugar de Vector → ArrayList o Collections.synchronizedList()
// En lugar de Stack → ArrayDeque
Deque<Integer> pilaModerna = new ArrayDeque<>();
pilaModerna.push(1); pilaModerna.push(2);
int valor = pilaModerna.pop(); // 2
```

### 4.3.4 Métodos comunes de List

```java
List<String> lista = new ArrayList<>();

// Añadir
lista.add("Java");                    // Al final
lista.add(0, "Python");               // En índice
lista.addAll(List.of("C", "Rust"));   // Colección

// Acceso y modificación
String s = lista.get(0);              // Por índice
lista.set(0, "Kotlin");               // Reemplazar

// Eliminación
lista.remove(1);                      // Por índice
lista.remove("Rust");                 // Por valor
lista.removeIf(x -> x.length() < 2);  // Por predicado (Java 8+)

// Búsqueda
int idx = lista.indexOf("Java");      // -1 si no existe
boolean existe = lista.contains("Kotlin");

// Sublista (vista, modifica la original)
List<Integer> nums = new ArrayList<>(List.of(1,2,3,4,5));
List<Integer> sub = nums.subList(1, 4); // [2,3,4]
sub.set(0, 20); // nums ahora es [1,20,3,4,5]

// Ordenación
nums.sort(null);                          // Orden natural
nums.sort(Collections.reverseOrder());    // Orden descendente
```

### 4.3.5 Creación de listas inmutables

```java
// List.of() — inmutable (Java 9+)
List<String> inmutable = List.of("A", "B", "C");
// inmutable.add("D"); → UnsupportedOperationException

// Arrays.asList() — tamaño fijo, respaldado por array
String[] arr = {"X", "Y", "Z"};
List<String> fija = Arrays.asList(arr);
fija.set(0, "Nuevo");  // OK, modifica el array subyacente
// fija.add("W");      // UnsupportedOperationException
```

### 4.3.6 Iteración sobre listas

```java
// for-each (recomendado para recorrido simple)
for (String s : lista) { ... }

// Iterator (permite eliminar)
Iterator<String> it = lista.iterator();
while (it.hasNext()) {
    if (it.next().equals("X")) it.remove();
}

// forEach con lambda (Java 8+)
lista.forEach(System.out::println);
```
---

## 4.4 Set

La interfaz `Set` modela el concepto matemático de **conjunto**: no permite duplicados. La igualdad se determina mediante `equals()` y `hashCode()`.

### 4.4.1 HashSet

`HashSet` internamente usa un `HashMap` donde las claves son los elementos y los valores son un objeto dummy `PRESENT`.

```java
Set<String> paises = new HashSet<>();
paises.add("México");
paises.add("Argentina");
paises.add("México");        // No se añade: ya existe
System.out.println(paises.size()); // 2
```

Características: O(1) promedio, sin orden garantizado, permite un `null`.

### 4.4.2 LinkedHashSet

Extiende `HashSet` con una lista doblemente enlazada que preserva el **orden de inserción**.

```java
Set<String> ordenado = new LinkedHashSet<>();
ordenado.add("Tercero");
ordenado.add("Primero");
ordenado.add("Segundo");
// Itera: Tercero, Primero, Segundo (orden de inserción)
```

### 4.4.3 TreeSet

Implementa `SortedSet`/`NavigableSet`. Mantiene elementos **ordenados** según `Comparable` o `Comparator`.

```java
Set<String> alfabetico = new TreeSet<>();
alfabetico.add("Zorro");
alfabetico.add("Águila");
System.out.println(alfabetico); // [Zorro, Águila]

// Con Comparator por longitud
Set<String> porLongitud = new TreeSet<>(
    (a, b) -> {
        int cmp = Integer.compare(a.length(), b.length());
        return cmp != 0 ? cmp : a.compareTo(b);
    });
```

Características: O(log n) para add/remove/contains, no permite `null`, métodos de rango: `first()`, `last()`, `headSet()`, `tailSet()`, `subSet()`.

### 4.4.4 Operaciones de conjunto

```java
Set<Integer> a = new HashSet<>(Set.of(1, 2, 3, 4));
Set<Integer> b = new HashSet<>(Set.of(3, 4, 5, 6));

Set<Integer> union = new HashSet<>(a); union.addAll(b);       // [1,2,3,4,5,6]
Set<Integer> inter = new HashSet<>(a);  inter.retainAll(b);   // [3,4]
Set<Integer> diff  = new HashSet<>(a);  diff.removeAll(b);    // [1,2]
```

### 4.4.5 Creación de sets inmutables

```java
Set<String> inmutable = Set.of("Rojo", "Verde", "Azul");
Set<Integer> copia = Set.copyOf(coleccionExistente); // Java 10+
```

---

## 4.5 Map

La interfaz `Map` representa pares **clave-valor**. Las claves son únicas.

### 4.5.1 HashMap

```java
Map<String, Integer> edades = new HashMap<>();
edades.put("Ana", 25);
edades.put("Luis", 30);
edades.put("Ana", 26);              // Actualiza el valor de "Ana"
int edad = edades.get("Ana");       // 26
Integer edadX = edades.get("X");    // null (no existe)
boolean existe = edades.containsKey("Ana"); // true
edades.remove("Carlos");
```

### 4.5.2 LinkedHashMap

Mantiene el **orden de inserción** (o de acceso) mediante una lista doblemente enlazada.

```java
Map<String, String> capitales = new LinkedHashMap<>();
capitales.put("Francia", "París");
capitales.put("Japón", "Tokio");
// Itera en orden de inserción: Francia → Japón
```

### 4.5.3 TreeMap

Implementa `SortedMap`/`NavigableMap`. Claves **ordenadas**.

```java
Map<String, Double> precios = new TreeMap<>();
precios.put("Computadora", 1200.99);
precios.put("Monitor", 350.50);
// Itera por orden alfabético de claves
```

### 4.5.4 Comparativa de Map

| Característica | HashMap | LinkedHashMap | TreeMap |
|----------------|---------|---------------|---------|
| Estructura | Tabla hash | Tabla hash + lista | Árbol rojo-negro |
| Orden | Ninguno | Inserción/acceso | Natural/Comparator |
| Complejidad | O(1) | O(1) | O(log n) |
| Claves null | Sí (una) | Sí (una) | No |
| Uso memoria | Menor | Mayor | Moderado |

### 4.5.5 Métodos comunes de Map

```java
Map<String, Integer> map = new HashMap<>();

// Inserción y actualización
map.put("A", 100);
map.putIfAbsent("A", 200);           // No cambia, ya existe
map.putIfAbsent("B", 50);            // Inserta nueva

// Recuperación segura
int v = map.getOrDefault("X", 0);    // 0 si no existe

// Actualización condicional
map.replace("A", 90);                // Reemplaza si existe
map.replace("A", 90, 95);            // Solo si valor actual es 90

// computeIfAbsent / computeIfPresent
map.computeIfAbsent("C", k -> 0);    // Añade con valor 0
map.computeIfPresent("A", (k, v) -> v + 10); // 110

// merge
map.merge("A", 50, Integer::sum);    // 160 = 110 + 50
map.merge("Nuevo", 75, Integer::sum);// Inserta 75
```

### 4.5.6 Iteración sobre Map

```java
// entrySet() — RECOMENDADO
for (Map.Entry<String, Integer> e : map.entrySet()) {
    System.out.println(e.getKey() + ": " + e.getValue());
}

// forEach con lambda (Java 8+)
map.forEach((k, v) -> System.out.println(k + " -> " + v));
```

### 4.5.7 Creación de mapas inmutables

```java
Map<String, String> inmutable = Map.of("L", "Monday", "M", "Tuesday");

Map<Integer, String> estaciones = Map.ofEntries(
    Map.entry(1, "Primavera"),
    Map.entry(2, "Verano")
);

Map<String, Integer> copia = Map.copyOf(inventario); // Java 10+
```

---

## 4.6 Funcionamiento interno de HashSet y HashMap (Sección Crítica)

### 4.6.1 Arquitectura de la tabla hash

HashMap es un array de "buckets". El índice del bucket se calcula a partir del `hashCode()` de la clave.

```
table = [0] [1] [2] [3] [4] [5] [6] [7] [8] [9] [10] [11] [12] [13] [14] [15]
          │       │                        │
          ▼       ▼                        ▼
        null  Node("A",1)              Node("Z",26)
                  │
                  ▼
              Node("B",2) → Node("C",3)
```

### 4.6.2 Cálculo del índice: hashCode() → bucket

```java
// Paso 1: hashCode de la clave
int h = key.hashCode();

// Paso 2: Función de dispersión — XOR con bits superiores
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// Esto asegura que los bits altos del hashCode influyan en el índice.

// Paso 3: índice del bucket (table.length es potencia de 2)
int index = hash & (table.length - 1);
// hash & 15 = hash % 16 (módulo con AND es ~10x más rápido en CPU)
```

**Diagrama del proceso:**

```
CLAVE: "Hola" → hashCode() = 2256382 (0x00226FDE)

   hashCode:  00000000 00100010 01101111 11011110
   >>> 16:    00000000 00000000 00000000 00100010
   XOR:      ─────────────────────────────────────
   hash:      00000000 00100010 01101111 11111100 = 0x00226FFC

   table.length - 1 = 15 = 0b01111
   hash & 15:  00000000 00100010 01101111 11111100
             & 00000000 00000000 00000000 00001111
             ─────────────────────────────────────
   index:      00000000 00000000 00000000 00001100 = 12

   RESULTADO: Bucket 12
```

### 4.6.3 Manejo de colisiones

**Colisión:** Dos claves diferentes → mismo bucket.

Estrategia: **Separate Chaining** con dos fases desde Java 8:

```
FASE 1: Lista enlazada (pocos elementos)
┌──────┐    ┌──────┐    ┌──────┐
│Node A├───>│Node B├───>│Node C│ → null
└──────┘    └──────┘    └──────┘
Búsqueda: O(n)

FASE 2: Árbol rojo-negro (> 8 elementos y tabla > 64)
         [K5]
        /     \
     [K2]    [K7]
     /  \    /  \
  [K1] [K3][K6] [K8]
          \
          [K4]
Búsqueda: O(log n)
```

### 4.6.4 Treeify Thresholds

```java
// Código real del JDK:
static final int TREEIFY_THRESHOLD = 8;    // > 8 → árbol
static final int UNTREEIFY_THRESHOLD = 6;  // <= 6 → lista
static final int MIN_TREEIFY_CAPACITY = 64; // tabla >= 64 para treeificar
```

**¿Por qué 8?** Bajo una función hash ideal (Poisson λ≈0.5), la probabilidad de que un bucket tenga k elementos:

| k | Probabilidad |
|---|-------------|
| 0 | 0.60653 |
| 1 | 0.30327 |
| 8 | 0.00000006 ← prácticamente nunca, pero posible en ataques DoS |

**Protección HashDoS:** Java 8 introdujo la treeificación como mitigación. Java 7 añadió hash seed aleatorio para `String.hashCode()`.

### 4.6.5 Load Factor: qué es 0.75 y por qué

```
load factor = size / table.length

Ejemplo: table.length=16, load factor=0.75
  Umbral = 16 * 0.75 = 12
  Cuando size=13 → tabla se duplica a 32 → todos los elementos se redistribuyen
```

**Análisis de trade-off:**

| Load Factor | Ventaja | Desventaja |
|-------------|---------|------------|
| 0.5 | Menos colisiones | Más memoria, resize frecuente |
| **0.75** | **Balance óptimo** | — |
| 1.0 | Máximo ahorro de memoria | Muchas colisiones (+30% tiempo búsqueda) |

```java
// Personalizar load factor:
new HashMap<>(16, 0.5f); // Prioriza velocidad
new HashMap<>(1_000_000, 0.9f); // Prioriza memoria
```

### 4.6.6 Cómo escribir un buen hashCode()

**Principios:**

1. **Consistente con equals():** `a.equals(b)` → `a.hashCode() == b.hashCode()` (OBLIGATORIO)
2. **Distribución uniforme** en el espacio de `int` (2³² valores)
3. **Rápido de calcular** (se llama en cada `put`/`get`)
4. **Usar números primos** (31 es tradicional, optimizado como `(i << 5) - i`)

**Fórmula recomendada (Effective Java, Joshua Bloch):**

```java
@Override
public int hashCode() {
    int result = 17;
    result = 31 * result + nombre.hashCode();
    result = 31 * result + Integer.hashCode(edad);
    result = 31 * result + Double.hashCode(salario);
    result = 31 * result + Objects.hashCode(departamento); // nullable
    return result;
}

// Alternativa: Objects.hash() (auto-boxing → ligeramente más lento)
@Override
public int hashCode() {
    return Objects.hash(nombre, edad, email);
}
```

### 4.6.7 Ejemplo de hashCode() roto: todos devuelven 42

```java
public class ClaveMala {
    private String valor;
    @Override public boolean equals(Object o) { /* correcto */ return ...; }

    @Override public int hashCode() {
        return 42; // ← TODAS las instancias caen en el mismo bucket
    }
}
```

**Consecuencias:**

```
HashMap con 1M elementos con ClaveMala:

table = [0]...[10]→1→2→3→4→5→...→999,999 ... [15]

  put() con buen hashCode(): ~80 ms
  put() con hashCode=42:    ~45,000 ms → 560x más lento

  get() con buen hashCode(): ~50 ns
  get() con hashCode=42:    ~5,000 ns → 100x más lento

HashMap se convierte en O(n) — una lista enlazada gloriosa.
Java 8 lo mitiga treeificando a los 8 elementos, pero sigue siendo muy lento.
```

### 4.6.8 Análisis de rendimiento con JMH

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Thread)
public class HashMapBenchmark {

    static class ClaveBuena {
        String valor;
        @Override public int hashCode() { return valor.hashCode(); } // Distribución natural
    }

    static class ClaveMala {
        String valor;
        @Override public int hashCode() { return 42; } // ¡Todos el mismo bucket!
    }

    @Param({"1000", "10000", "50000"})
    public int N;

    @Benchmark
    public void put_buenHashCode() {
        Map<ClaveBuena, String> map = new HashMap<>();
        for (var c : clavesBuenas) map.put(c, "v");
    }

    @Benchmark
    public void put_malHashCode() {
        Map<ClaveMala, String> map = new HashMap<>();
        for (var c : clavesMalas) map.put(c, "v");
    }
}
```

Resultados típicos:

| Operación | N=1,000 | N=10,000 | N=50,000 |
|-----------|---------|----------|----------|
| put buenHashCode | 80 µs | 900 µs | 5,200 µs |
| put malHashCode | 120 µs | 3,800 µs | 78,000 µs |
| get buenHashCode | 45 µs | 500 µs | 2,800 µs |
| get malHashCode | 95 µs | 2,100 µs | 48,000 µs |

Degradación con N=50,000: put ~15x más lento, get ~17x más lento.

### 4.6.9 Resumen práctico

```java
// REGLAS DE ORO:
// 1. SIEMPRE implementa hashCode() y equals() juntos
// 2. hashCode() consistente con equals()
// 3. Usa objetos INMUTABLES como claves

// Ejemplo de CORRUPCIÓN por clave mutable:
class ClaveMutable {
    String valor;
    @Override public int hashCode() { return valor.hashCode(); }
    public void setValor(String v) { this.valor = v; }
}

Map<ClaveMutable, String> map = new HashMap<>();
ClaveMutable k = new ClaveMutable("original");
map.put(k, "valor");       // Bucket calculado con hashCode("original")
k.setValor("modificado");  // ¡hashCode cambia!
map.get(k);                // null — busca en bucket equivocado
// El HashMap está CORROMPIDO.
```

---

## 4.7 TreeSet y TreeMap en profundidad

### 4.7.1 El árbol rojo-negro

`TreeMap`/`TreeSet` implementan un **árbol binario de búsqueda auto-balanceado rojo-negro**.

**Propiedades:**
1. Cada nodo es **rojo** o **negro**.
2. La raíz es **negra**.
3. Hojas (NIL) son **negras**.
4. Si un nodo es rojo, ambos hijos son **negros**.
5. Todo camino raíz→hoja tiene el **mismo número de nodos negros**.

### 4.7.2 Visualización ASCII

```
ÁRBOL ROJO-NEGRO: 8, 3, 10, 1, 6, 14, 4, 7, 13

  ● = negro, ○ = rojo

                ┌───────────┐
                │    8 (●)  │  ← Raíz siempre negra
                └─────┬─────┘
              ┌───────┴───────┐
     ┌───────────┐     ┌───────────┐
     │    3 (○)  │     │   10 (●)  │
     └─────┬─────┘     └─────┬─────┘
        ┌──┴──┐           ┌──┴──┐
   ┌───────────┐  ┌───────────┐  ┌───────────┐
   │    1 (●)  │  │    6 (●)  │  │   14 (○)  │
   └───────────┘  └─────┬─────┘  └─────┬─────┘
                      ┌──┴──┐         ┌──┴──┐
               ┌───────────┐┌───────────┐┌───────────┐
               │    4 (○)  ││   7 (○)   ││   13 (●)  │
               └───────────┘└───────────┘└───────────┘

Propiedad 5: altura negra = 3 en todos los caminos ✓
```

### 4.7.3 Rotaciones y balanceo

```
ROTACIÓN DERECHA:          ROTACIÓN IZQUIERDA:
      x                         x
     / \                       / \
    y   C    →               A   y    →
   / \                           / \
  A   B                         B   C
      y                         y
     / \                       / \
    A   x                     x   C
       / \                   / \
      B   C                 A   B
```

**Ejemplo de inserción con corrección:**

```
Insertar 5, 3, 1:

   [5(●)]          [5(●)]       [5(●)]         [3(●)]
                     /           /              /    \
                  [3(○)]       [3(○)]        [1(○)] [5(○)]
                                /
                             [1(○)] ← VIOLACIÓN: dos rojos consecutivos
                       → rotación derecha en 5 + recoloreo → balanceado
```

### 4.7.4 El costo real de mantener el orden

```
Insertar 1M elementos y luego iterar ordenado:

┌──────────────────────┬───────────┬──────────┬──────────┐
│                      │ HashMap   │ TreeMap  │ HashMap  │
│                      │ (sin orden)│ (orden) │ + sort   │
├──────────────────────┼───────────┼──────────┼──────────┤
│ Insertar 1M          │ ~80 ms    │ ~380 ms  │ ~80 ms   │
│ Iterar ordenado      │ -         │ ~45 ms   │ -        │
│ Insertar + sort      │ ~555 ms   │ -        │ ~555 ms  │
│ Búsqueda individual  │ ~50 ns    │ ~150 ns  │ ~50 ns   │
│ Memoria/entrada      │ ~40 bytes │ ~48 bytes│ ~40 bytes│
└──────────────────────┴───────────┴──────────┴──────────┘

CONCLUSIÓN: TreeMap ~5x más lento insertando, ~3x más lento buscando.
Usa TreeMap cuando necesites operaciones de rango (headMap, tailMap, subMap)
o consultas de menor/mayor constantemente.
```
---

## 4.8 Iteración

### 4.8.1 La interfaz Iterator

| Método | Descripción |
|--------|-------------|
| `boolean hasNext()` | `true` si hay más elementos |
| `E next()` | Retorna el siguiente elemento |
| `void remove()` | Elimina el último elemento retornado |

```java
Iterator<String> it = lista.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("X")) it.remove(); // Eliminación segura
}
```

**Nunca** llames `next()` sin verificar `hasNext()` → `NoSuchElementException`.

### 4.8.2 ListIterator

Extiende `Iterator`: recorrido bidireccional, modificación de elementos.

```java
ListIterator<String> it = lista.listIterator();
while (it.hasNext()) {
    int idx = it.nextIndex();
    String s = it.next();
    it.set("Reemplazo"); // Modifica el elemento actual
    it.add("Nuevo");     // Inserta después del cursor
}
while (it.hasPrevious()) {
    String s = it.previous(); // Hacia atrás
}
```

### 4.8.3 Fail-Fast vs Fail-Safe

**Fail-Fast** (`ArrayList`, `HashSet`, `HashMap`): lanzan `ConcurrentModificationException` si la colección se modifica fuera del iterador. Detectan cambios mediante un contador `modCount`.

```java
for (String s : lista) {
    lista.remove(s); // ConcurrentModificationException
}
```

**Fail-Safe** (`ConcurrentHashMap`, `CopyOnWriteArrayList`): iteran sobre copia/snapshot, no lanzan excepción.

### 4.8.4 Soluciones a ConcurrentModificationException

```java
// Solución 1: Iterator.remove()
Iterator<String> it = lista.iterator();
while (it.hasNext()) {
    if (it.next().startsWith("C")) it.remove();
}

// Solución 2: removeIf() (Java 8+)
lista.removeIf(s -> s.startsWith("C"));

// Solución 3: Recorrer hacia atrás por índice
for (int i = lista.size() - 1; i >= 0; i--) {
    if (lista.get(i).startsWith("C")) lista.remove(i);
}

// Solución 4: Stream con filter
List<String> filtrada = lista.stream()
    .filter(s -> !s.startsWith("C"))
    .collect(Collectors.toList());
```

---

## 4.9 Ordenación

### 4.9.1 Comparable

Define el **orden natural**. Método: `int compareTo(T o)`.

| Valor | Significado |
|-------|-------------|
| `< 0` | `this` es menor que `o` |
| `0` | Iguales |
| `> 0` | `this` es mayor que `o` |

```java
class Estudiante implements Comparable<Estudiante> {
    private String nombre;
    private double promedio;

    @Override
    public int compareTo(Estudiante otro) {
        return Double.compare(otro.promedio, this.promedio); // Descendente
    }
}
```

**Reglas:**
1. **Reflexivo:** `x.compareTo(y)` signo opuesto a `y.compareTo(x)`.
2. **Transitivo:** `x>y && y>z` → `x>z`.
3. **Consistente con equals:** `compareTo==0` → `equals` para colecciones ordenadas.

### 4.9.2 Comparator

Define un **orden externo**: múltiples criterios sin modificar la clase.

```java
Comparator<Estudiante> porNombre = (e1, e2) ->
    e1.getNombre().compareTo(e2.getNombre());

Comparator<Estudiante> porPromedio =
    Comparator.comparingDouble(Estudiante::getPromedio);
```

### 4.9.3 Métodos de fábrica (Java 8+)

```java
// comparing() — ordena por un campo
Comparator<Estudiante> porNombre = Comparator.comparing(Estudiante::getNombre);

// thenComparing() — desempate
Comparator<Estudiante> porPromedioLuegoNombre = Comparator
    .comparingDouble(Estudiante::getPromedio)
    .thenComparing(Estudiante::getNombre);

// reversed() — orden inverso
Comparator<Estudiante> porNombreDesc = Comparator
    .comparing(Estudiante::getNombre).reversed();

// reversed() correcto dentro de thenComparing:
productos.sort(Comparator
    .comparing(Producto::getCategoria)
    .thenComparing(Comparator.comparingDouble(Producto::getPrecio).reversed())
);

// Manejo de null
Comparator<String> nullPrimero = Comparator.nullsFirst(Comparator.naturalOrder());
Comparator<String> nullUltimo = Comparator.nullsLast(String.CASE_INSENSITIVE_ORDER);
```

### 4.9.4 Métodos de ordenación

```java
Collections.sort(lista);                           // Orden natural
Collections.sort(lista, Collections.reverseOrder()); // Descendente
lista.sort(null);                                   // Orden natural (Java 8+)
lista.sort(Comparator.comparing(...));              // Con comparator (Java 8+)
```

**Comparativa: Comparable vs Comparator**

| | Comparable (`java.lang`) | Comparator (`java.util`) |
|---|--------------------------|---------------------------|
| Propósito | Orden natural del objeto | Orden externo / múltiples |
| Método | `compareTo(T o)` | `compare(T o1, T o2)` |
| Dónde se define | Dentro de la clase | Fuera de la clase |
| Criterios | Uno solo | Ilimitados |
| Encadenamiento | No | `.thenComparing()` |

### 4.9.5 Ejemplo práctico completo

```java
class Empleado implements Comparable<Empleado> {
    private int id;
    private String nombre;
    private String departamento;
    private double salario;

    // Orden natural: por ID
    @Override public int compareTo(Empleado otro) {
        return Integer.compare(this.id, otro.id);
    }

    // Orden por nombre ascendente
    static Comparator<Empleado> POR_NOMBRE =
        Comparator.comparing(Empleado::getNombre);

    // Orden por departamento, luego salario descendente
    static Comparator<Empleado> POR_DEPTO_SALARIO = Comparator
        .comparing(Empleado::getDepartamento)
        .thenComparing(Comparator.comparingDouble(Empleado::getSalario).reversed());

    public static void main(String[] args) {
        List<Empleado> emps = new ArrayList<>(List.of(
            new Empleado(103, "Carlos", "Ventas", 45000),
            new Empleado(101, "Ana", "IT", 60000),
            new Empleado(105, "Beatriz", "IT", 65000)
        ));

        // Orden natural
        emps.sort(null);

        // Orden por nombre
        emps.sort(POR_NOMBRE);

        // Orden por departamento + salario descendente
        emps.sort(POR_DEPTO_SALARIO);

        // Top 3 salarios más altos
        emps.stream()
            .sorted(Comparator.comparingDouble(Empleado::getSalario).reversed())
            .limit(3)
            .forEach(System.out::println);

        // TreeSet con orden personalizado
        Set<Empleado> tree = new TreeSet<>(Comparator.comparing(Empleado::getNombre));
        tree.addAll(emps);
    }
}
```

---

## 4.10 Estructuras de datos avanzadas con colecciones Java

### 4.10.1 LRU Cache con LinkedHashMap

`LinkedHashMap` con `accessOrder=true` es perfecto para implementar una caché LRU (Least Recently Used):

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacidadMaxima;

    public LRUCache(int capacidadMaxima) {
        // accessOrder=true: ordena por acceso, no por inserción
        super(capacidadMaxima, 0.75f, true);
        this.capacidadMaxima = capacidadMaxima;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacidadMaxima;
    }

    public static void main(String[] args) {
        LRUCache<Integer, String> cache = new LRUCache<>(3);

        cache.put(1, "A");
        cache.put(2, "B");
        cache.put(3, "C");
        System.out.println(cache); // {1=A, 2=B, 3=C}

        cache.get(1);              // Accede a 1 → se vuelve el más reciente
        cache.put(4, "D");         // Expulsa al menos reciente (2)
        System.out.println(cache); // {3=C, 1=A, 4=D}

        cache.put(5, "E");         // Expulsa a 3
        System.out.println(cache); // {1=A, 4=D, 5=E}

        System.out.println(cache.containsKey(2)); // false — fue expulsado
        System.out.println(cache.get(1));          // "A" — aún está
    }
}
```

### 4.10.2 PriorityQueue — Heap binario

`PriorityQueue` implementa un **heap binario** (min-heap por defecto). Los elementos se ordenan según orden natural o `Comparator`.

```java
// Min-heap (por defecto): el menor elemento tiene prioridad
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5);
minHeap.offer(1);
minHeap.offer(8);
minHeap.offer(3);

System.out.println(minHeap.peek()); // 1 — el menor
while (!minHeap.isEmpty()) {
    System.out.print(minHeap.poll() + " "); // 1 3 5 8
}

// Max-heap con Comparator.reverseOrder()
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(5);
maxHeap.offer(1);
System.out.println(maxHeap.peek()); // 5 — el mayor
```

**Casos de uso:**

```java
// 1. Planificación de tareas (task scheduler)
record Tarea(String nombre, int prioridad) {}
PriorityQueue<Tarea> scheduler = new PriorityQueue<>(
    Comparator.comparingInt(Tarea::prioridad).reversed()
);
scheduler.offer(new Tarea("Procesar pago", 10));
scheduler.offer(new Tarea("Enviar email", 5));
scheduler.offer(new Tarea("Generar reporte", 8));
while (!scheduler.isEmpty()) {
    System.out.println("Ejecutando: " + scheduler.poll());
}
// Ejecutando: Tarea[Procesar pago, 10]
// Ejecutando: Tarea[Generar reporte, 8]
// Ejecutando: Tarea[Enviar email, 5]

// 2. Top-K elementos más frecuentes
Map<String, Integer> frecuencias = new HashMap<>();
// ... llenar frecuencias ...
PriorityQueue<String> topK = new PriorityQueue<>(
    Comparator.comparingInt(frecuencias::get)
);
for (String palabra : frecuencias.keySet()) {
    topK.offer(palabra);
    if (topK.size() > 3) topK.poll(); // Mantener solo top 3
}
```

**Complejidad:** offer/poll O(log n), peek O(1), remove O(n).

### 4.10.3 ArrayDeque — el reemplazo de Stack y LinkedList

`ArrayDeque` es más rápido que `Stack` y `LinkedList` para pilas y colas. Usa un array circular internamente, sin overhead de nodos.

```java
// Como pila (LIFO) — más rápido que java.util.Stack
Deque<String> pila = new ArrayDeque<>();
pila.push("Primero");   // Añade al principio
pila.push("Segundo");
pila.push("Tercero");

while (!pila.isEmpty()) {
    System.out.println(pila.pop()); // Tercero, Segundo, Primero
}

// Como cola (FIFO) — más rápido que LinkedList
Deque<String> cola = new ArrayDeque<>();
cola.offer("A");         // Añade al final
cola.offer("B");
cola.offer("C");

System.out.println(cola.poll()); // A — FIFO
System.out.println(cola.peek()); // B

// Como deque (doble extremo)
Deque<String> deque = new ArrayDeque<>();
deque.addFirst("Inicio");
deque.addLast("Final");
deque.offerFirst("NuevoInicio"); // Inserta sin lanzar excepción si falla
deque.offerLast("NuevoFinal");
```

**¿Por qué ArrayDeque es más rápido?**

```
ArrayDeque: array circular, sin nodos, sin punteros, sin GC overhead
  Operaciones O(1) amortizado con factor constante muy bajo

LinkedList: nodos con next/prev, allocaciones individuales, pointer chasing
  Operaciones O(1) con factor constante alto

Comparativa JMH (push/pop 1M operaciones):
  ArrayDeque: ~5 ms
  Stack:      ~8 ms   (sincronización innecesaria)
  LinkedList: ~12 ms  (creación de nodos)
```

### 4.10.4 EnumSet y EnumMap — optimizados con bit vectors

`EnumSet` y `EnumMap` están **extremadamente optimizados** para enums. Internamente usan **vectores de bits** (bit vectors), lo que los hace órdenes de magnitud más rápidos que `HashSet`/`HashMap`.

```java
enum Dia { LUNES, MARTES, MIERCOLES, JUEVES, VIERNES, SABADO, DOMINGO }

// EnumSet — implementado como bit vector
// Cada bit representa un enum: LUNES=bit0, MARTES=bit1, etc.
// 7 días → cabe en un solo long (64 bits)

EnumSet<Dia> finde = EnumSet.of(Dia.SABADO, Dia.DOMINGO);
// Internamente: 0b1100000 (solo 7 bits usados de un long)

EnumSet<Dia> laborables = EnumSet.range(Dia.LUNES, Dia.VIERNES);
// 0b0011111

EnumSet<Dia> todos = EnumSet.allOf(Dia.class);
EnumSet<Dia> ninguno = EnumSet.noneOf(Dia.class);

// Operaciones de conjunto extremadamente rápidas (operaciones bitwise)
EnumSet<Dia> copia = EnumSet.copyOf(findfe);
copia.add(Dia.VIERNES);
copia.remove(Dia.DOMINGO);
// contains() es una operación AND bit a bit: O(1) CONSTANTE REAL (~1 ns)

// EnumMap — implementado como array directo
// ordinal() del enum se usa como índice del array: acceso O(1) sin hashing
EnumMap<Dia, String> horario = new EnumMap<>(Dia.class);
horario.put(Dia.LUNES, "9:00-18:00");
horario.put(Dia.VIERNES, "9:00-15:00");

String horarioLunes = horario.get(Dia.LUNES); // Acceso directo por índice
```

**Benchmark: EnumSet vs HashSet para enums:**

```
Operación contains() con 5 elementos:
  EnumSet: ~1 ns   (operación AND bit a bit)
  HashSet: ~35 ns  (hashing + búsqueda en bucket)

Operación add() para 5 elementos:
  EnumSet: ~3 ns   (operación OR bit a bit)
  HashSet: ~50 ns  (hashing + inserción)

EnumSet es ~35x más rápido que HashSet para enums.
```

### 4.10.5 WeakHashMap — cachés con referencias débiles

`WeakHashMap` usa **referencias débiles** (`WeakReference`) para las claves. Si una clave ya no tiene referencias fuertes en el código, el GC puede recolectarla y la entrada se elimina automáticamente del mapa.

```java
// Ejemplo: caché que no impide que los objetos sean recolectados
WeakHashMap<ClavePesada, DatosCalculados> cache = new WeakHashMap<>();

ClavePesada clave = new ClavePesada("config.xml");
DatosCalculados datos = calcularCostoso(clave);
cache.put(clave, datos);

// Mientras 'clave' tenga referencias fuertes, la entrada persiste
System.out.println(cache.get(clave)); // DatosCalculados{...}

// Si 'clave' se vuelve inalcanzable...
clave = null;
System.gc(); // Sugerir (no garantizar) GC

// Tras el GC, la entrada desaparece automáticamente
// cache.size() probablemente será 0
```

**Caso de uso real: caché de metadatos:**

```java
public class CacheDebil<K, V> {
    private final WeakHashMap<K, V> cache = new WeakHashMap<>();

    public V obtener(K clave, Function<K, V> calculadora) {
        return cache.computeIfAbsent(clave, calculadora);
    }

    public int tamanio() {
        return cache.size();
    }
}

// Útil para cachés donde la clave puede ser eliminada cuando
// ya no se necesita en otra parte del programa
```

**Comparación con otros tipos de caché:**

| Tipo | Mecanismo | Cuándo se elimina |
|------|-----------|-------------------|
| `WeakHashMap` | WeakReference en claves | GC cuando clave es inalcanzable |
| `SoftReference` | Referencia suave | GC cuando memoria escasa |
| LRU Cache | `LinkedHashMap.removeEldestEntry()` | Cuando tamaño > máximo |
| TTL Cache | Timer / ScheduledExecutor | Después de tiempo de expiración |

---

## 4.11 Comparación de rendimiento de colecciones

### 4.11.1 Tabla comparativa completa

Resultados de benchmarks JMH en JDK 21, Apple M1, con warmup adecuado. Tiempos en nanosegundos por operación individual (N=100,000 elementos):

| Operación | ArrayList | LinkedList | HashSet | TreeSet | HashMap | TreeMap |
|-----------|-----------|------------|---------|---------|---------|---------|
| **add** | ~25 ns | ~50 ns | ~60 ns | ~180 ns | ~55 ns | ~175 ns |
| **get/contains** | ~3 ns (índice) | ~60 µs (índice) | ~30 ns | ~140 ns | ~30 ns | ~135 ns |
| **remove** | ~3 µs (medio) | ~60 µs (medio) | ~30 ns | ~140 ns | ~30 ns | ~135 ns |
| **iterate (1M)** | ~500 µs | ~5 ms | ~3 ms | ~5 ms | ~3.5 ms | ~6 ms |
| **memory/elem** | 4 bytes | 24 bytes | ~40 bytes | ~48 bytes | ~40 bytes | ~48 bytes |

Notas:
- `ArrayList.get(int)`: ~3 ns (acceso directo a array)
- `LinkedList.get(int)`: ~60 µs para N=100K (navegación de punteros)
- `HashSet.add()`: incluye el costo de `hashCode()` + inserción en bucket
- `TreeSet.add()`: incluye balanceo del árbol rojo-negro

### 4.11.2 Cuándo usar cada estructura

```
                    ¿Necesitas pares clave-valor?
                               │
                    ┌──────────┴──────────┐
                    SI                    NO
                    │                     │
         ¿Necesitas orden?         ¿Necesitas orden?
         ┌────┴────┐              ┌────┴────┐
         SI        NO             SI        NO
         │         │              │         │
     TreeMap   HashMap        TreeSet   ¿Duplicados?
                                        ┌───┴───┐
                                        SI      NO
                                        │       │
                                  ArrayList  HashSet
                                  (por defecto)

EXCEPCIONES:
- ¿Muchas inserciones al principio? → ArrayDeque
- ¿Solo enums? → EnumSet/EnumMap
- ¿Caché con límite? → LinkedHashMap access-order
- ¿Cola de prioridad? → PriorityQueue
- ¿Claves que pueden ser GC? → WeakHashMap
```

### 4.11.3 Reglas de oro

1. **La elección por defecto es ArrayList para listas, HashMap para mapas.**
2. **Nunca uses LinkedList a menos que un benchmark lo justifique.**
3. **Nunca uses Vector o Stack (legado). Usa ArrayList y ArrayDeque.**
4. **Especifica la capacidad inicial si conoces el tamaño aproximado.**
5. **Implementa hashCode() y equals() correctamente para usar HashMap/HashSet.**
6. **Usa objetos inmutables como claves de HashMap/TreeMap.**
7. **Para enums, usa siempre EnumSet y EnumMap.**
8. **Prefiere LinkedHashMap a HashMap cuando necesites orden de inserción.**
9. **Usa TreeMap/TreeSet solo cuando necesites orden garantizado en todo momento.**
10. **No uses `Collections.unmodifiable*` para crear colecciones inmutables; usa `List.of()`, `Set.of()`, `Map.of()`.**

---

## 4.12 Collections.unmodifiable* vs List.of() vs List.copyOf()

### 4.12.1 Tres formas de crear colecciones "que no se pueden modificar"

| Método | Comportamiento | Vista o copia | Acepta null | Java |
|--------|---------------|---------------|-------------|------|
| `Collections.unmodifiableList(list)` | Vista no modificable | **Vista** (cambios en original afectan la vista) | Sí | 1.2 |
| `List.of(e1, e2, ...)` | Colección inmutable | **Copia** (independiente) | No | 9+ |
| `List.copyOf(collection)` | Copia inmutable | **Copia** (independiente) | No | 10+ |

### 4.12.2 Diferencias críticas

```java
// 1. Collections.unmodifiableList() — VISTA, no copia
List<String> original = new ArrayList<>();
original.add("A");
original.add("B");

List<String> vista = Collections.unmodifiableList(original);
System.out.println(vista); // [A, B]

original.add("C");
System.out.println(vista); // [A, B, C] — ¡la vista refleja el cambio!
// La "inmutabilidad" es solo a través de la vista, no del original.

// 2. List.of() — COPIA inmutable real
List<String> inmutable = List.of("A", "B", "C");
// inmutable.add("D");     // UnsupportedOperationException
// inmutable.set(0, "Z");  // UnsupportedOperationException
// inmutable.contains(null); // NullPointerException — no acepta null en absoluto

// 3. List.copyOf() — COPIA inmutable (Java 10+)
List<String> original2 = new ArrayList<>(List.of("X", "Y", "Z"));
List<String> copia = List.copyOf(original2);

original2.add("W"); // Modificamos el original
System.out.println(copia);      // [X, Y, Z] — LA COPIA NO SE VE AFECTADA
System.out.println(original2);  // [X, Y, Z, W]
```

### 4.12.3 Rendimiento y casos de uso

```java
// CUÁNDO USAR CADA UNA:

// 1. Collections.unmodifiable*: cuando necesitas EXPONER una colección
//    interna sin dar acceso de modificación, pero MANTENIENDO actualización.
//    Ejemplo: API pública de una clase que mantiene datos mutables.
public class Usuario {
    private final List<String> permisos = new ArrayList<>();

    // Expone vista actualizada sin permitir modificaciones externas
    public List<String> getPermisos() {
        return Collections.unmodifiableList(permisos);
    }

    public void agregarPermiso(String permiso) {
        permisos.add(permiso);
        // Quienes tengan referencia a getPermisos() verán el cambio
    }
}

// 2. List.of() / Set.of() / Map.of(): para crear colecciones CONSTANTES
//    que nunca cambiarán. Más compacto en memoria que unmodifiableList.
//    Ejemplo: constantes, configuraciones, valores por defecto.
public class Config {
    public static final List<String> ROLES_PERMITIDOS =
        List.of("ADMIN", "USER", "GUEST");

    public static final Set<String> PAISES_HABILITADOS =
        Set.of("MX", "AR", "CL", "CO", "PE");

    public static final Map<String, Integer> PUERTOS_DEFECTO =
        Map.of("HTTP", 80, "HTTPS", 443, "SSH", 22);
}

// 3. List.copyOf(): para CREAR UNA COPIA DEFENSIVA inmutable de una
//    colección existente. No retiene referencia a la original.
public class Reporte {
    private final List<String> resultados;

    public Reporte(List<String> resultados) {
        // Copia defensiva: si el llamador modifica su lista,
        // no afecta nuestro estado interno.
        this.resultados = List.copyOf(resultados);
    }

    public List<String> getResultados() {
        return resultados; // Ya es inmutable, seguro retornar directamente
    }
}
```

### 4.12.4 Tabla de decisión

| Escenario | Recomendación |
|-----------|---------------|
| Constante conocida en tiempo de compilación | `List.of()`, `Set.of()`, `Map.of()` |
| Copia defensiva de colección recibida | `List.copyOf(col)`, `Set.copyOf(col)` |
| Exponer colección interna que sigue mutando | `Collections.unmodifiableList(lista)` |
| API que debe aceptar null | `Collections.unmodifiable*` (las `of()`/`copyOf()` no aceptan null) |
| Máximo rendimiento en memoria | `List.of()` (implementaciones optimizadas para pocos elementos) |
| Java 8 o anterior | `Collections.unmodifiable*` (no hay `of()`/`copyOf()`) |

---

---

## 4.13 La clase Collections y algoritmos utilitarios

La clase `java.util.Collections` proporciona algoritmos polimórficos estáticos para trabajar con colecciones.

### 4.13.1 Ordenación y búsqueda

```java
List<Integer> nums = new ArrayList<>(List.of(3, 1, 4, 1, 5, 9, 2, 6));

// Ordenación
Collections.sort(nums);                  // [1, 1, 2, 3, 4, 5, 6, 9]
Collections.reverse(nums);               // Invierte el orden actual
Collections.shuffle(nums);               // Orden aleatorio
Collections.sort(nums, Comparator.reverseOrder()); // Descendente

// Búsqueda binaria (requiere lista ordenada)
Collections.sort(nums);                  // Debe estar ordenada primero
int idx = Collections.binarySearch(nums, 5);       // Devuelve índice; -(insertion point)-1 si no existe
System.out.println("5 está en índice: " + idx);     // 5
System.out.println(Collections.binarySearch(nums, 7)); // -8 (estaría en índice 7)

// Relleno
Collections.fill(nums, 0);               // Todos los elementos = 0
```

### 4.13.2 Valores extremos y frecuencia

```java
List<Integer> nums = List.of(3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5);

int min = Collections.min(nums);         // 1
int max = Collections.max(nums);         // 9

// Con Comparator personalizado para objetos
List<String> palabras = List.of("Java", "Python", "Rust", "Go");
String masCorta = Collections.min(palabras,
    Comparator.comparingInt(String::length)); // "Go"
String masLarga = Collections.max(palabras,
    Comparator.comparingInt(String::length)); // "Python"

// Frecuencia de un elemento
int frecuencia = Collections.frequency(nums, 5); // 3 (aparece 3 veces)

// Elementos disjuntos (sin elementos en común)
boolean disjuntos = Collections.disjoint(
    Set.of(1, 2, 3), Set.of(4, 5, 6)); // true

boolean noDisjuntos = Collections.disjoint(
    Set.of(1, 2, 3), Set.of(3, 4, 5)); // false (comparten 3)
```

### 4.13.3 Wrappers sincronizados

```java
// Convertir una colección no sincronizada en thread-safe
List<String> listaSync = Collections.synchronizedList(new ArrayList<>());
Set<Integer> setSync = Collections.synchronizedSet(new HashSet<>());
Map<String, Integer> mapSync = Collections.synchronizedMap(new HashMap<>());
SortedSet<String> sortedSync = Collections.synchronizedSortedSet(new TreeSet<>());
SortedMap<String, Integer> sortedMapSync =
    Collections.synchronizedSortedMap(new TreeMap<>());

// ¡IMPORTANTE! La iteración sobre colecciones sincronizadas
// DEBE hacerse dentro de un bloque synchronized:
synchronized (listaSync) {
    for (String s : listaSync) {
        System.out.println(s);
    }
}
// El for-each fuera de synchronized lanza ConcurrentModificationException
// si otro hilo modifica la lista durante la iteración.
```

### 4.13.4 Wrappers no modificables

```java
// Vistas no modificables (no copian, solo envuelven)
List<String> lista = Collections.unmodifiableList(original);
Set<Integer> set = Collections.unmodifiableSet(originalSet);
Map<String, Integer> map = Collections.unmodifiableMap(originalMap);
SortedSet<String> sorted = Collections.unmodifiableSortedSet(originalSortedSet);
SortedMap<String, Integer> sortedMap = Collections.unmodifiableSortedMap(originalSortedMap);

// Colecciones vacías inmutables (reutilizan singleton)
List<String> vacia = Collections.emptyList();       // Siempre la misma instancia
Set<Integer> vacio = Collections.emptySet();
Map<String, String> vacioMap = Collections.emptyMap();
Iterator<String> vacioIt = Collections.emptyIterator();

// Colecciones singleton
List<String> singletonList = Collections.singletonList("Unico");
Set<Integer> singletonSet = Collections.singleton(42);
Map<String, Integer> singletonMap = Collections.singletonMap("clave", 100);
```

### 4.13.5 Rotación, intercambio e inversión

```java
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4, 5));

// Rotar: desplazar elementos circularmente
Collections.rotate(nums, 2);
System.out.println(nums); // [4, 5, 1, 2, 3]

Collections.rotate(nums, -1);
System.out.println(nums); // [5, 1, 2, 3, 4]

// Intercambiar dos elementos
Collections.swap(nums, 0, 4);
System.out.println(nums); // [4, 1, 2, 3, 5]

// Invertir
Collections.reverse(nums);
System.out.println(nums); // [5, 3, 2, 1, 4]
```

### 4.13.6 nCopies: crear lista con N copias del mismo elemento

```java
// Crea una lista inmutable con 10 copias del mismo objeto
List<String> diezVecesHola = Collections.nCopies(10, "Hola");
System.out.println(diezVecesHola.size()); // 10

// ¡Cuidado con objetos mutables! Todas las "copias" son el MISMO objeto
List<StringBuilder> builders = Collections.nCopies(3, new StringBuilder("Inicio"));
builders.get(0).append(" modificado");
System.out.println(builders.get(1)); // "Inicio modificado" ← ¡MISMO OBJETO!

// Útil para inicializar listas
List<Boolean> flags = new ArrayList<>(Collections.nCopies(100, false));
// Lista de 100 elementos, todos false
```

### 4.13.7 addAll: añadir múltiples elementos de una vez

```java
List<Integer> nums = new ArrayList<>();

// Añadir varios elementos a una colección
Collections.addAll(nums, 1, 2, 3, 4, 5);

// Equivalente pero más eficiente que:
// for (int i : new int[]{1,2,3,4,5}) nums.add(i);

// También funciona con Set
Set<String> colores = new HashSet<>();
Collections.addAll(colores, "rojo", "verde", "azul", "rojo"); // solo 3 elementos
```

---

## 4.14 NavigableSet y NavigableMap

### 4.14.1 NavigableSet (TreeSet)

`NavigableSet` extiende `SortedSet` con métodos de navegación:

```java
NavigableSet<Integer> set = new TreeSet<>();
set.addAll(List.of(10, 20, 30, 40, 50, 60, 70, 80, 90, 100));

// --- Métodos de navegación ---
Integer menor   = set.first();        // 10
Integer mayor   = set.last();         // 100

Integer menorQue50  = set.lower(50);  // 40  (estrictamente menor)
Integer menorOIgual  = set.floor(50); // 50  (menor o igual)
Integer mayorOIgual  = set.ceiling(50); // 50 (mayor o igual)
Integer mayorQue50  = set.higher(50); // 60  (estrictamente mayor)

// Al buscar valores que no existen:
Integer lower65 = set.lower(65);      // 60 (el mayor elemento < 65)
Integer floor65 = set.floor(65);      // 60 (el mayor elemento <= 65)
Integer ceiling65 = set.ceiling(65);  // 70 (el menor elemento >= 65)
Integer higher65 = set.higher(65);    // 70 (el menor elemento > 65)

// Para valores fuera de rango:
Integer lower5 = set.lower(5);        // null (no hay elemento < 5)
Integer higher110 = set.higher(110);  // null (no hay elemento > 110)

// --- Operaciones de eliminación navegables ---
Integer primero = set.pollFirst();    // 10 — elimina y devuelve el menor
Integer ultimo = set.pollLast();      // 100 — elimina y devuelve el mayor

// --- Subconjuntos (vistas respaldadas) ---
SortedSet<Integer> headSet = set.headSet(50);     // [20, 30, 40]
SortedSet<Integer> tailSet = set.tailSet(70);     // [70, 80, 90]
SortedSet<Integer> subSet  = set.subSet(30, 70);  // [30, 40, 50, 60]

// headSet con inclusividad (NavigableSet)
NavigableSet<Integer> headSetInc = set.headSet(50, true);  // [20, 30, 40, 50]
NavigableSet<Integer> headSetExc = set.headSet(50, false); // [20, 30, 40]

// --- Iteración en orden descendente ---
NavigableSet<Integer> descendente = set.descendingSet();
for (Integer n : descendente) {
    System.out.print(n + " "); // 90 80 70 60 50 40 30 20
}

Iterator<Integer> itDesc = set.descendingIterator();
```

### 4.14.2 NavigableMap (TreeMap)

```java
NavigableMap<String, Integer> scores = new TreeMap<>();
scores.put("Ana", 85);
scores.put("Carlos", 92);
scores.put("Beatriz", 78);
scores.put("Diana", 95);
scores.put("Ernesto", 88);

// --- Métodos de navegación ---
Map.Entry<String, Integer> primera  = scores.firstEntry(); // Ana=85
Map.Entry<String, Integer> ultima   = scores.lastEntry();  // Ernesto=88

Map.Entry<String, Integer> menorQueCarlos = scores.lowerEntry("Carlos"); // Beatriz=78
Map.Entry<String, Integer> menorOIgual    = scores.floorEntry("Carlos"); // Carlos=92
Map.Entry<String, Integer> mayorOIgual    = scores.ceilingEntry("Daniel");  // Diana=95
Map.Entry<String, Integer> mayorQue   = scores.higherEntry("Carlos");  // Diana=95

// --- Versiones con solo la clave ---
String claveMenor = scores.lowerKey("Carlos");     // "Beatriz"
String claveMayor = scores.higherKey("Carlos");    // "Diana"

// --- Operaciones de eliminación ---
Map.Entry<String, Integer> eliminadoPrimero = scores.pollFirstEntry();
Map.Entry<String, Integer> eliminadoUltimo  = scores.pollLastEntry();

// --- Submapas (vistas) ---
NavigableMap<String, Integer> headMap = scores.headMap("Carlos", true);
NavigableMap<String, Integer> tailMap = scores.tailMap("Carlos", true);
NavigableMap<String, Integer> subMap  = scores.subMap("Beatriz", true, "Diana", true);

// --- Orden descendente ---
NavigableMap<String, Integer> descendente = scores.descendingMap();
NavigableSet<String> clavesDesc = scores.descendingKeySet();
```

---

## 4.15 Deque: uso avanzado de colas de doble extremo

### 4.15.1 La interfaz Deque

`Deque` (Double Ended Queue, pronunciado "deck") permite insertar y eliminar elementos en ambos extremos. Puede funcionar como pila LIFO, cola FIFO, o cola de doble extremo.

```
OPERACIONES DE DEQUE:

        addFirst() / offerFirst()              addLast() / offerLast()
        push()                                 add() / offer()
        ┌───┐                                  ┌───┐
        ▼   │                                  │   ▼
   ┌───────────────────────────────────────────────────┐
   │   elemento → [ ... elementos internos ... ]        │
   └───────────────────────────────────────────────────┘
        │   ▲                                  ▲   │
        └───┘                                  └───┘
        removeFirst() / pollFirst()            removeLast() / pollLast()
        pop() / remove()
```

### 4.15.2 Métodos de Deque

| Operación | Lanza excepción | Retorna valor especial (null/false) |
|-----------|-----------------|-------------------------------------|
| **Insertar al principio** | `addFirst(e)` | `offerFirst(e)` |
| **Insertar al final** | `addLast(e)` | `offerLast(e)` |
| **Eliminar del principio** | `removeFirst()` | `pollFirst()` |
| **Eliminar del final** | `removeLast()` | `pollLast()` |
| **Examinar principio** | `getFirst()` | `peekFirst()` |
| **Examinar final** | `getLast()` | `peekLast()` |

### 4.15.3 Deque como pila (LIFO)

```java
Deque<Integer> pila = new ArrayDeque<>();

// push/pop/peek (equivalentes Deque)
pila.push(1);           // addFirst
pila.push(2);
pila.push(3);
System.out.println(pila); // [3, 2, 1]

int cima = pila.peek();     // 3 — peekFirst
int valor = pila.pop();     // 3 — removeFirst
System.out.println(valor);  // 3

// Alternativa con nombres Deque explícitos
pila.addFirst(4);
pila.removeFirst(); // equivalente a pop()
```

### 4.15.4 Deque como cola (FIFO)

```java
Deque<String> cola = new ArrayDeque<>();

// offer/poll/peek
cola.offer("Primero");     // addLast
cola.offer("Segundo");
cola.offer("Tercero");
System.out.println(cola);  // [Primero, Segundo, Tercero]

String frente = cola.peek();   // "Primero" — peekFirst
String atendido = cola.poll(); // "Primero" — pollFirst
System.out.println(atendido);  // "Primero"

// Alternativa con nombres Deque
cola.addLast("Cuarto");
cola.removeFirst(); // equivalente a poll()
```

### 4.15.5 ArrayDeque vs LinkedList como Deque

```
Benchmark JMH — 1M operaciones push/pop:

┌──────────────────┬──────────┬────────────────┬──────────┐
│ Operación         │ ArrayDeque│ LinkedList    │ Ratio    │
├──────────────────┼──────────┼────────────────┼──────────┤
│ addFirst (push)  │ ~4 ms    │ ~10 ms         │ 2.5x     │
│ addLast (offer)  │ ~3 ms    │ ~8 ms          │ 2.7x     │
│ removeFirst (pop)│ ~3 ms    │ ~5 ms          │ 1.7x     │
│ removeLast       │ ~3 ms    │ ~9 ms          │ 3.0x     │
│ Memoria/elem     │ ~4 bytes │ ~24 bytes      │ 6x       │
└──────────────────┴──────────┴────────────────┴──────────┘

ArrayDeque es siempre mejor como pila o cola. LinkedList solo si necesitas
eliminar desde el medio durante la iteración (con Iterator.remove()).
```

### 4.15.6 Caso de uso real: deshacer/rehacer (undo/redo)

```java
public class EditorTexto {
    private final Deque<String> deshacer = new ArrayDeque<>();
    private final Deque<String> rehacer = new ArrayDeque<>();
    private String textoActual = "";

    public void escribir(String nuevoTexto) {
        deshacer.push(textoActual);  // Guardar estado actual
        textoActual = nuevoTexto;
        rehacer.clear();             // Nueva acción invalida rehacer
    }

    public boolean deshacer() {
        if (deshacer.isEmpty()) return false;
        rehacer.push(textoActual);
        textoActual = deshacer.pop();
        return true;
    }

    public boolean rehacer() {
        if (rehacer.isEmpty()) return false;
        deshacer.push(textoActual);
        textoActual = rehacer.pop();
        return true;
    }

    public String getTexto() { return textoActual; }

    public static void main(String[] args) {
        EditorTexto editor = new EditorTexto();
        editor.escribir("Hola");
        editor.escribir("Hola Mundo");
        editor.escribir("Hola Mundo!");

        editor.deshacer();
        System.out.println(editor.getTexto()); // "Hola Mundo"

        editor.deshacer();
        System.out.println(editor.getTexto()); // "Hola"

        editor.rehacer();
        System.out.println(editor.getTexto()); // "Hola Mundo"
    }
}
```

---

## 4.16 Conversión entre colecciones, arrays y streams

### 4.16.1 Colecciones a arrays y viceversa

```java
// List → Array
List<String> lista = List.of("A", "B", "C");
String[] array = lista.toArray(new String[0]);
// Pasar array de tamaño 0 es más eficiente desde Java 6+
// (el método crea uno nuevo del tamaño correcto internamente)

String[] arrayPreDimensionado = new String[lista.size()];
lista.toArray(arrayPreDimensionado); // También funciona

// Array → List (tamaño fijo, respaldado por el array)
String[] datos = {"X", "Y", "Z"};
List<String> desdeArray = Arrays.asList(datos);
// ¡Cuidado! Es una vista: modificar la lista modifica el array

// Array → List (independiente, copia real)
List<String> copiaReal = new ArrayList<>(Arrays.asList(datos));
// o con List.of()
List<String> inmutable = List.of(datos);

// Colección → Set
List<String> conDuplicados = List.of("A", "B", "A", "C", "B");
Set<String> sinDuplicados = new HashSet<>(conDuplicados); // [A, B, C]
Set<String> sinDupOrdenado = new LinkedHashSet<>(conDuplicados); // [A, B, C]
Set<String> sinDupOrdenadoNat = new TreeSet<>(conDuplicados); // [A, B, C]

// Set → List
Set<Integer> conjunto = Set.of(1, 2, 3, 4, 5);
List<Integer> desdeSet = new ArrayList<>(conjunto);
```

### 4.16.2 Streams y colecciones

```java
List<String> nombres = List.of("Ana", "Luis", "Carlos", "Beatriz", "Diana");

// Colección → Stream → List (filtrar y transformar)
List<String> filtrados = nombres.stream()
    .filter(n -> n.length() > 3)
    .collect(Collectors.toList()); // [Luis, Carlos, Beatriz, Diana]

// Colección → Stream → Set
Set<String> setNombres = nombres.stream()
    .filter(n -> n.startsWith("A") || n.startsWith("C"))
    .collect(Collectors.toSet());

// Colección → Stream → Map (agrupar)
Map<Character, List<String>> porInicial = nombres.stream()
    .collect(Collectors.groupingBy(n -> n.charAt(0)));
// {A=[Ana], B=[Beatriz], C=[Carlos], D=[Diana], L=[Luis]}

// Colección → Stream → Array
String[] largo4 = nombres.stream()
    .filter(n -> n.length() == 4)
    .toArray(String[]::new); // [Luis]

// Stream → Colección inmutable (Java 16+)
List<String> inmutable = nombres.stream()
    .filter(n -> n.length() > 3)
    .toList(); // Java 16+

// Paralelismo
List<String> mayusculas = nombres.parallelStream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

### 4.16.3 Iterables, Iterator y Spliterator

```java
// De Iterator a Stream
Iterator<String> it = List.of("A", "B", "C").iterator();
Spliterator<String> spliterator = Spliterators.spliteratorUnknownSize(it, 0);
Stream<String> stream = StreamSupport.stream(spliterator, false);
stream.forEach(System.out::println);

// De Iterable a Stream
Iterable<String> iterable = () -> List.of("X", "Y", "Z").iterator();
StreamSupport.stream(iterable.spliterator(), false)
    .forEach(System.out::println);

// Spliterator para procesamiento paralelo (divide y vencerás)
List<Integer> numeros = IntStream.range(0, 1_000_000).boxed().toList();
Spliterator<Integer> spl = numeros.spliterator();

// trySplit() divide el Spliterator para procesamiento paralelo
Spliterator<Integer> mitad = spl.trySplit();
// spl contiene ~500K elementos, mitad contiene los otros ~500K
System.out.println("Mitad 1 estimado: " + spl.estimateSize());   // ~500,000
System.out.println("Mitad 2 estimado: " + mitad.estimateSize());  // ~500,000
```

---

## 4.17 Patrones y antipatrones con colecciones

### 4.17.1 Patrones recomendados

```java
// PATRÓN 1: Copia defensiva en constructores
public class Historial {
    private final List<String> eventos;

    public Historial(List<String> eventos) {
        this.eventos = List.copyOf(eventos); // Inmutable, independiente
    }

    public List<String> getEventos() {
        return eventos; // Ya es inmutable, se puede retornar directamente
    }
}

// PATRÓN 2: capacity planning
public List<Usuario> cargarUsuarios() {
    int cuenta = repositorio.contar();
    List<Usuario> usuarios = new ArrayList<>(cuenta); // Sin resizes
    // ... cargar
    return usuarios;
}

// PATRÓN 3: computeIfAbsent para inicialización perezosa
Map<String, List<String>> grupos = new HashMap<>();
grupos.computeIfAbsent("admins", k -> new ArrayList<>()).add("root");
// Más limpio y eficiente que:
// if (!grupos.containsKey("admins")) grupos.put("admins", new ArrayList<>());
// grupos.get("admins").add("root");

// PATRÓN 4: removeIf en vez de Iterator
lista.removeIf(s -> s == null || s.isBlank());
// Más legible y eficiente que:
// Iterator<String> it = lista.iterator();
// while (it.hasNext()) { if (it.next().isBlank()) it.remove(); }

// PATRÓN 5: EnumSet para flags de opciones
public class Archivo {
    public enum Permiso { LECTURA, ESCRITURA, EJECUCION }

    private final EnumSet<Permiso> permisos;
    public Archivo(EnumSet<Permiso> permisos) {
        this.permisos = EnumSet.copyOf(permisos);
    }
    public boolean puedeLeer() { return permisos.contains(Permiso.LECTURA); }
    public boolean puedeEscribir() { return permisos.contains(Permiso.ESCRITURA); }
}
```

### 4.17.2 Antipatrones comunes

```java
// ANTIPATRÓN 1: Usar LinkedList sin benchmark que lo justifique
// MAL: LinkedList<String> list = new LinkedList<>();
// BIEN: List<String> list = new ArrayList<>();

// ANTIPATRÓN 2: No implementar hashCode() al usar HashMap/HashSet
// MAL:
class PersonaSinHash {
    String nombre;
    @Override public boolean equals(Object o) { /* solo equals */ }
    // SIN hashCode → degradación O(n) en HashMap
}

// ANTIPATRÓN 3: Claves mutables en HashMap
// MAL:
class ClaveConSetter {
    String valor;
    public void setValor(String v) { this.valor = v; }
}

// ANTIPATRÓN 4: Crear ArrayList sin capacidad inicial cuando se conoce el tamaño
// MAL: List<String> list = new ArrayList<>(); for (int i=0; i<1_000_000; i++) list.add(..)
// BIEN: List<String> list = new ArrayList<>(1_000_000);

// ANTIPATRÓN 5: get() en loop sobre LinkedList
// MAL:
LinkedList<String> list = new LinkedList<>();
for (int i = 0; i < list.size(); i++) {
    String s = list.get(i); // O(n²)!
}
// BIEN: for-each o Iterator

// ANTIPATRÓN 6: Modificar colección durante for-each
// MAL:
for (String s : lista) { if (cond) lista.remove(s); } // ConcurrentModificationException
// BIEN: lista.removeIf(s -> cond);

// ANTIPATRÓN 7: Usar Vector o Stack en código nuevo
// MAL:
Stack<String> pila = new Stack<>();
// BIEN: Deque<String> pila = new ArrayDeque<>();

// ANTIPATRÓN 8: hashCode() que no es consistente con equals()
class MalaImplementacion {
    int id;
    @Override public boolean equals(Object o) {
        return o instanceof MalaImplementacion m && m.id == this.id;
    }
    @Override public int hashCode() {
        return (int) (Math.random() * 1000); // ¡ALEATORIO! Rompe el contrato
    }
}
```

---

## 4.18 Apéndice: Tabla completa de complejidades

### 4.18.1 Todas las implementaciones principales

```
COMPLEJIDAD TEMPORAL DE OPERACIONES POR IMPLEMENTACIÓN

┌──────────────────────┬──────────┬────────────┬─────────┬─────────┬──────────┬──────────┐
│ Operación             │ ArrayList│ LinkedList │ HashSet │ TreeSet │ HashMap  │ TreeMap  │
├──────────────────────┼──────────┼────────────┼─────────┼─────────┼──────────┼──────────┤
│ add(E)                │ O(1)*    │ O(1)       │ O(1)    │ O(log n)│ O(1)     │ O(log n) │
│ add(int, E)           │ O(n)     │ O(n)       │ N/A     │ N/A     │ N/A      │ N/A      │
│ get(int)              │ O(1)     │ O(n)       │ N/A     │ N/A     │ N/A      │ N/A      │
│ set(int, E)           │ O(1)     │ O(n)       │ N/A     │ N/A     │ N/A      │ N/A      │
│ remove(int)           │ O(n)     │ O(n)       │ N/A     │ N/A     │ N/A      │ N/A      │
│ remove(Object)        │ O(n)     │ O(n)       │ O(1)    │ O(log n)│ O(1)     │ O(log n) │
│ contains(Object)      │ O(n)     │ O(n)       │ O(1)    │ O(log n)│ O(1)**   │ O(log n) │
│ indexOf(Object)       │ O(n)     │ O(n)       │ N/A     │ N/A     │ N/A      │ N/A      │
│ iterator().remove()   │ O(n)     │ O(1)       │ O(1)    │ O(log n)│ O(1)     │ O(log n) │
│ listIterator().add()  │ O(n)     │ O(n)       │ N/A     │ N/A     │ N/A      │ N/A      │
├──────────────────────┼──────────┼────────────┼─────────┼─────────┼──────────┼──────────┤
│ put(K,V)              │ N/A      │ N/A        │ N/A     │ N/A     │ O(1)     │ O(log n) │
│ get(K)                │ N/A      │ N/A        │ N/A     │ N/A     │ O(1)     │ O(log n) │
│ containsKey(K)        │ N/A      │ N/A        │ N/A     │ N/A     │ O(1)     │ O(log n) │
│ containsValue(V)      │ N/A      │ N/A        │ N/A     │ N/A     │ O(n)     │ O(n)     │
├──────────────────────┼──────────┼────────────┼─────────┼─────────┼──────────┼──────────┤
│ first()/last()        │ O(1)     │ O(1)       │ N/A     │ O(log n)│ N/A      │ O(log n) │
│ lower()/higher()      │ N/A      │ N/A        │ N/A     │ O(log n)│ N/A      │ O(log n) │
│ headSet()/tailSet()   │ N/A      │ N/A        │ N/A     │ O(log n)│ N/A      │ O(log n) │
├──────────────────────┼──────────┼────────────┼─────────┼─────────┼──────────┼──────────┤
│ sort()                │ O(n log n)│ O(n log n)│ N/A     │ N/A     │ N/A      │ N/A      │
├──────────────────────┴──────────┴────────────┴─────────┴─────────┴──────────┴──────────┤
│ * Amortizado, O(n) en el peor caso cuando requiere redimensionamiento del array         │
│ ** O(1) para containsKey, O(n) para containsValue                                       │
└─────────────────────────────────────────────────────────────────────────────────────────┘


COMPLEJIDAD ESPACIAL POR IMPLEMENTACIÓN

┌─────────────────────┬───────────────────────────────────────────────┐
│ Implementación       │ Memoria por elemento (aproximada)             │
├─────────────────────┼───────────────────────────────────────────────┤
│ ArrayList            │ 4 bytes (referencia en array contiguo)        │
│ LinkedList           │ 24 bytes (objeto Node: 3 referencias)         │
│ HashSet/HashMap      │ ~40 bytes (Node: hash+key+value+next)         │
│ LinkedHashSet/Map    │ ~48 bytes (Node + referencias before/after)   │
│ TreeSet/TreeMap      │ ~48 bytes (Entry: key+value+left+right+parent+color) │
│ ArrayDeque           │ 4 bytes (referencia en array circular)        │
│ PriorityQueue        │ 4 bytes (referencia en array del heap)        │
│ EnumSet              │ ~8 bytes (1-2 longs para bit vectors)         │
│ EnumMap              │ ~4 bytes por entrada (array indexado)         │
└─────────────────────┴───────────────────────────────────────────────┘
```

### 4.18.2 Complejidad de operaciones de Collections

```
┌──────────────────────────────┬───────────────────┐
│ Método                        │ Complejidad       │
├──────────────────────────────┼───────────────────┤
│ Collections.sort(list)        │ O(n log n)        │
│ Collections.binarySearch()    │ O(log n)          │
│ Collections.reverse()         │ O(n)              │
│ Collections.shuffle()         │ O(n)              │
│ Collections.fill()            │ O(n)              │
│ Collections.copy(dest, orig)  │ O(n)              │
│ Collections.min() / max()     │ O(n)              │
│ Collections.frequency()       │ O(n)              │
│ Collections.rotate(list, d)   │ O(n)              │
│ Collections.swap(list, i, j)  │ O(1)              │
│ Collections.indexOfSubList()  │ O(n * m)          │
│ Collections.disjoint()        │ O(n * m)          │
│ Collections.nCopies(n, obj)   │ O(1) (vista lazy)│
│ Collections.singletonList()   │ O(1)              │
└──────────────────────────────┴───────────────────┘
```

---

## 4.19 Guía de selección rápida

```
                    RESUMEN DE DECISIÓN

Tengo que almacenar...
  │
  ├── Valores únicos (sin duplicados)
  │   ├── Sin orden → HashSet
  │   ├── Orden de inserción → LinkedHashSet
  │   └── Orden natural/comparator → TreeSet
  │
  ├── Secuencia ordenada (con duplicados)
  │   ├── Muchas lecturas por índice → ArrayList
  │   ├── Muchas inserciones al principio → ArrayDeque
  │   └── Solo colas/pilas → ArrayDeque
  │
  ├── Pares clave-valor
  │   ├── Sin orden → HashMap
  │   ├── Orden de inserción/acceso → LinkedHashMap
  │   ├── Orden por claves → TreeMap
  │   └── Solo enums → EnumMap
  │
  ├── Procesamiento por prioridad → PriorityQueue
  │
  └── Caché con límite de tamaño → LinkedHashMap (access-order)
      Caché con referencias débiles → WeakHashMap
```

## Resumen del capítulo (completo)

- Los **genéricos** proporcionan seguridad de tipos en tiempo de compilación. El **type erasure** borra los parámetros de tipo, generando **bridge methods** para mantener el polimorfismo. La información genérica de declaraciones se conserva vía reflection.
- El **principio PECS** guía wildcards: `<? extends T>` para lectura, `<? super T>` para escritura. Los wildcards pueden anidarse y capturarse con helper methods.
- **Heap pollution** ocurre al mezclar raw types con genéricos o con varargs. `@SafeVarargs` suprime warnings en métodos seguros.
- **No se pueden crear arrays de tipos genéricos** (`new T[10]`, `new List<String>[10]`). Usa `Object[]` interno como hace ArrayList, o reflection.
- `ArrayList`: array redimensionable (+50% al crecer), localidad de referencia CPU, ~3 ns para get(). **Especifica capacidad inicial** para evitar redimensionamientos.
- `LinkedList`: nodos con next/prev, 6x más memoria, O(n) para get(), cache misses constantes. **Casi nunca la uses.**
- `HashMap` internamente: `hashCode() → hash() → index = hash & (n-1)`. Colisiones → lista enlazada → árbol rojo-negro (>8 elementos). Load factor 0.75 es el balance óptimo. **hashCode() roto = rendimiento O(n)**.
- `TreeMap`/`TreeSet`: árbol rojo-negro auto-balanceado, O(log n). ~5x más lento que HashMap. Ideal para operaciones de rango y consultas de menor/mayor.
- **Estructuras avanzadas:** `LinkedHashMap` access-order para LRU Cache, `PriorityQueue` como heap binario, `ArrayDeque` más rápido que Stack/LinkedList, `EnumSet`/`EnumMap` con bit vectors (~35x más rápido), `WeakHashMap` para cachés con referencias débiles.
- **Elección por defecto:** ArrayList para listas, HashMap para mapas, ArrayDeque para pilas/colas.
- **Inmutabilidad:** `List.of()`/`Set.of()`/`Map.of()` para colecciones inmutables (Java 9+). `List.copyOf()` para copia defensiva (Java 10+). `Collections.unmodifiable*()` crea vistas, no copias.

---

← [Capítulo anterior](capitulo-03-poo.md) | [Inicio](README.md) | [Capítulo siguiente →](capitulo-05-excepciones.md)
