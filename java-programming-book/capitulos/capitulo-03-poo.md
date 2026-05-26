# Capítulo 3: Programación Orientada a Objetos en Java

---

## 3.1 Clases y Objetos

### 3.1.1 Definiciones fundamentales

Una **clase** es un plano, molde o plantilla (_blueprint_) que define la estructura y el comportamiento que tendrán los objetos creados a partir de ella. Un **objeto** es una instancia concreta de una clase: ocupa memoria, tiene un estado propio y puede ejecutar los comportamientos definidos por su clase.

Pensemos en una analogía: la clase `Coche` es el plano de ingeniería que describe qué atributos tiene todo coche (marca, modelo, color, velocidad) y qué puede hacer (acelerar, frenar). Un objeto sería un coche físico concreto: "un Toyota Corolla rojo, parado", que es una instancia de esa clase.

| Concepto   | Definición                                              | Ejemplo                         |
|------------|--------------------------------------------------------|---------------------------------|
| Clase      | Plantilla que define atributos y métodos               | `Coche` (el plano)              |
| Objeto     | Instancia concreta de una clase, con estado propio     | `miCoche` (el coche físico)     |
| Atributo   | Variable que almacena el estado del objeto             | `color`, `velocidad`            |
| Método     | Función que define el comportamiento del objeto         | `acelerar()`, `frenar()`        |

### 3.1.2 El porqué de la POO: modelando el mundo real

La Programación Orientada a Objetos no surgió como un capricho académico. Nació de una necesidad real: los programas crecían hasta hacerse inmanejables con el paradigma procedural, donde los datos y las funciones que los manipulan viven separados. La POO propuso un cambio radical: **agrupar datos y comportamiento bajo una misma entidad**, el objeto.

#### El mapeo objeto-relacional mental

Cuando modelamos un dominio, establecemos un paralelismo entre conceptos del mundo real y clases de nuestro programa:

```
Mundo real                    Modelo en Java
───────────                   ──────────────
Un bibliotecario              Clase Bibliotecario
Un libro físico               Objeto de tipo Libro
Prestar un libro              Método prestarLibro()
El catálogo de la biblioteca  Colección<Libro>
```

Este mapeo no es automático ni trivial. Para no perderte entre tanta jerga de arquitectura, divide tus clases de objetos en tres conceptos cotidianos muy sencillos:

> [!NOTE]
> ### 📦 Pasaportes, Billetes y Cajas de Herramientas
> 
> 1. **Entidades (El Pasaporte)**: Es un objeto que tiene una identidad única. Tu pasaporte físico tiene un número único impreso. Si le cambias la foto o la dirección, sigue siendo *tu* pasaporte. Dos pasaportes con los mismos datos personales pero diferente número de pasaporte representan a dos personas distintas. En programación, un `Cliente` o un `Pedido` son entidades.
> 2. **Objetos de Valor / Value Objects (El Billete de $20 USD)**: A estos objetos no les importa su identidad individual, sino el valor de sus propiedades. Si tienes un billete de $20 en tu cartera y lo cambias por otro billete de $20 diferente, sigues teniendo exactamente la misma cantidad de dinero. No te importa el número de serie del billete. En Java, cosas como un color (`Color`), una coordenada (`Coordenada`) o el dinero (`Dinero`) son Value Objects: si sus atributos son iguales, son el mismo objeto.
> 3. **Agregados (La Caja de Herramientas)**: Es un grupo de objetos relacionados que se manipulan como una sola unidad. Imagina una caja de herramientas cerrada. No puedes sacar el martillo ni el destornillador sin abrir la caja completa. La caja es la raíz del agregado (por ejemplo, el `Pedido` es el agregado y las `Líneas de Pedido` son las herramientas internas; solo modificas las líneas a través del pedido principal).

En el modelado tradicional y el diseño táctico de Domain-Driven Design (DDD), esto se clasifica en:
- **Entidades**: objetos con identidad propia que persisten en el tiempo (un `Cliente`, un `Pedido`).
- **Objetos de valor** (_value objects_): inmutables, definidos por sus atributos, no por identidad (un `Dinero`, una `Dirección`, una `Coordenada`).
- **Agregados**: grupos de objetos tratados como una unidad (`Pedido` contiene `LineaPedido`).
- **Servicios**: operaciones que no pertenecen naturalmente a ninguna entidad (`ProcesadorDePagos`, `NotificadorDeEmail`).
- **Repositorios**: abstraen el almacenamiento (`RepositorioDeLibros`).

#### Ejemplo: modelando un sistema de biblioteca

Imaginemos que debemos construir software para una biblioteca universitaria. El proceso de modelado comienza identificando los sustantivos en la descripción del negocio:

> "Un **estudiante** puede **tomar prestados** **libros** de la **biblioteca**. Cada **préstamo** tiene una **fecha de inicio** y una **fecha límite de devolución**. Si un libro no se devuelve a tiempo, se genera una **multa**."

De aquí extraemos candidatos a clases: `Estudiante`, `Libro`, `Biblioteca`, `Prestamo`, `Multa`. Los verbos ("tomar prestados", "devolver", "generar") se convierten en métodos.

```java
public class Libro {
    private final String isbn;
    private final String titulo;
    private final String autor;
    private boolean prestado;

    public Libro(String isbn, String titulo, String autor) {
        this.isbn = isbn;
        this.titulo = titulo;
        this.autor = autor;
        this.prestado = false;
    }

    public void marcarComoPrestado() {
        if (prestado) {
            throw new IllegalStateException("El libro ya está prestado");
        }
        this.prestado = true;
    }

    public void marcarComoDevuelto() {
        if (!prestado) {
            throw new IllegalStateException("El libro no estaba prestado");
        }
        this.prestado = false;
    }

    public boolean estaDisponible() {
        return !prestado;
    }

    public String getIsbn() { return isbn; }
    public String getTitulo() { return titulo; }
    public String getAutor() { return autor; }
}

public class Estudiante {
    private final String matricula;
    private final String nombre;
    private final List<Prestamo> historialPrestamos;

    public Estudiante(String matricula, String nombre) {
        this.matricula = matricula;
        this.nombre = nombre;
        this.historialPrestamos = new ArrayList<>();
    }

    public boolean puedeTomarPrestado() {
        long prestamosActivos = historialPrestamos.stream()
            .filter(p -> !p.estaDevuelto())
            .count();
        return prestamosActivos < 5;
    }

    public void registrarPrestamo(Prestamo prestamo) {
        historialPrestamos.add(prestamo);
    }

    public String getMatricula() { return matricula; }
    public String getNombre() { return nombre; }
}

public class Prestamo {
    private final Libro libro;
    private final Estudiante estudiante;
    private final LocalDate fechaInicio;
    private final LocalDate fechaLimite;
    private LocalDate fechaDevolucion;

    public Prestamo(Libro libro, Estudiante estudiante, LocalDate fechaInicio) {
        this.libro = libro;
        this.estudiante = estudiante;
        this.fechaInicio = fechaInicio;
        this.fechaLimite = fechaInicio.plusDays(14);
    }

    public boolean estaVencido() {
        return fechaDevolucion == null && LocalDate.now().isAfter(fechaLimite);
    }

    public void devolver() {
        if (fechaDevolucion != null) {
            throw new IllegalStateException("El libro ya fue devuelto");
        }
        this.fechaDevolucion = LocalDate.now();
        libro.marcarComoDevuelto();
    }

    public double calcularMulta() {
        if (!estaVencido()) return 0;
        long diasRetraso = ChronoUnit.DAYS.between(fechaLimite,
            fechaDevolucion != null ? fechaDevolucion : LocalDate.now());
        return diasRetraso * 2.50;
    }

    public boolean estaDevuelto() {
        return fechaDevolucion != null;
    }

    public Libro getLibro() { return libro; }
    public Estudiante getEstudiante() { return estudiante; }
    public LocalDate getFechaLimite() { return fechaLimite; }
}

public class Multa {
    private final Estudiante estudiante;
    private final double monto;
    private boolean pagada;

    public Multa(Estudiante estudiante, double monto) {
        this.estudiante = estudiante;
        this.monto = monto;
        this.pagada = false;
    }

    public void pagar() { this.pagada = true; }
    public boolean estaPagada() { return pagada; }
    public double getMonto() { return monto; }
}

public class Biblioteca {
    private final Map<String, Libro> catalogo;
    private final List<Prestamo> prestamosActivos;

    public Biblioteca() {
        this.catalogo = new HashMap<>();
        this.prestamosActivos = new ArrayList<>();
    }

    public void agregarLibro(Libro libro) {
        catalogo.put(libro.getIsbn(), libro);
    }

    public Prestamo prestarLibro(String isbn, Estudiante estudiante) {
        Libro libro = catalogo.get(isbn);
        if (libro == null) throw new IllegalArgumentException("Libro no encontrado");
        if (!libro.estaDisponible()) throw new IllegalStateException("Libro prestado");
        if (!estudiante.puedeTomarPrestado()) throw new IllegalStateException("Límite excedido");

        libro.marcarComoPrestado();
        Prestamo prestamo = new Prestamo(libro, estudiante, LocalDate.now());
        estudiante.registrarPrestamo(prestamo);
        prestamosActivos.add(prestamo);
        return prestamo;
    }
}
```

#### Ejemplo: modelando una tienda online

```java
// Value objects: definidos por sus atributos, inmutables
public record Dinero(BigDecimal cantidad, String moneda) {
    public Dinero {
        Objects.requireNonNull(cantidad);
        Objects.requireNonNull(moneda);
    }
    public Dinero sumar(Dinero otro) {
        if (!moneda.equals(otro.moneda)) throw new IllegalArgumentException("Monedas distintas");
        return new Dinero(cantidad.add(otro.cantidad), moneda);
    }
}

// Entidad con identidad propia
public class Producto {
    private final String sku;
    private String nombre;
    private Dinero precio;

    public Producto(String sku, String nombre, Dinero precio) {
        this.sku = Objects.requireNonNull(sku);
        this.nombre = nombre;
        this.precio = precio;
    }

    public String getSku() { return sku; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Producto p)) return false;
        return sku.equals(p.sku);
    }

    @Override
    public int hashCode() { return sku.hashCode(); }
}

// Agregado: raíz que agrupa entidades relacionadas
public class Pedido {
    private final String numeroPedido;
    private final Cliente cliente;
    private final List<LineaPedido> lineas;
    private EstadoPedido estado;

    public Pedido(String numero, Cliente cliente) {
        this.numeroPedido = numero;
        this.cliente = cliente;
        this.lineas = new ArrayList<>();
        this.estado = EstadoPedido.PENDIENTE;
    }

    public void añadirProducto(Producto p, int cantidad) {
        if (estado != EstadoPedido.PENDIENTE) throw new IllegalStateException("Pedido cerrado");
        lineas.add(new LineaPedido(p, cantidad));
    }

    public Dinero calcularTotal() {
        return lineas.stream()
            .map(LineaPedido::calcularSubtotal)
            .reduce(new Dinero(BigDecimal.ZERO, "EUR"), Dinero::sumar);
    }

    public void confirmar() {
        if (lineas.isEmpty()) throw new IllegalStateException("Pedido vacío");
        this.estado = EstadoPedido.CONFIRMADO;
    }
}

// Servicio de dominio (sin estado, opera sobre entidades)
public class ServicioPagos {
    public boolean procesarPago(Pedido pedido, MetodoPago metodo) {
        Dinero total = pedido.calcularTotal();
        return metodo.cobrar(total);
    }
}
```

#### POO como paradigma vs Java como lenguaje

Es crucial distinguir entre la POO como paradigma y Java como lenguaje:

| POO (paradigma)                    | Java (lenguaje)                         |
|------------------------------------|----------------------------------------|
| Concepto abstracto, independiente  | Implementación concreta del paradigma  |
| Encapsulamiento, herencia, polimorfismo | Modificadores `private`, `extends`, _vtables_ |
| Principios de diseño (SOLID, GRASP)| Reglas sintácticas y semánticas        |
| Se puede hacer POO en C, Python... | También soporta paradigma funcional (desde Java 8) |

Un programador puede escribir Java procedural (todo `static`, sin objetos de dominio) o Java verdaderamente orientado a objetos. La sintaxis no garantiza buen diseño. La POO bien aplicada produce código que refleja el dominio del problema, facilitando el mantenimiento y la evolución.

### 3.1.3 Sintaxis para declarar una clase

La sintaxis mínima para declarar una clase en Java es:

```java
[modificador] class NombreDeLaClase {
    // atributos (campos)
    // constructores
    // métodos
}
```

**Ejemplo — Clase `Persona`:**

```java
public class Persona {
    // Atributos (estado)
    String nombre;
    int edad;

    // Métodos (comportamiento)
    void saludar() {
        System.out.println("Hola, mi nombre es " + nombre
            + " y tengo " + edad + " años.");
    }
}
```

**Ejemplo — Clase `Coche`:**

```java
public class Coche {
    String marca;
    String modelo;
    int velocidad;   // velocidad en km/h

    void acelerar(int incremento) {
        velocidad = velocidad + incremento;
        System.out.println("El coche acelera a " + velocidad + " km/h");
    }

    void frenar(int decremento) {
        velocidad = velocidad - decremento;
        if (velocidad < 0) velocidad = 0;
        System.out.println("El coche frena a " + velocidad + " km/h");
    }

    void mostrarEstado() {
        System.out.println(marca + " " + modelo
            + " va a " + velocidad + " km/h");
    }
}
```

### 3.1.4 Creación de objetos con `new`

Los objetos se crean con el operador `new`, que reserva memoria en el _heap_, invoca al constructor y devuelve una referencia al objeto recién creado.

```java
public class Principal {
    public static void main(String[] args) {
        // Declaración de variable + creación del objeto
        Persona p1 = new Persona();
        Persona p2 = new Persona();

        // Asignación de valores a los atributos
        p1.nombre = "Ana";
        p1.edad = 28;

        p2.nombre = "Carlos";
        p2.edad = 34;

        // Invocación de métodos con el operador punto
        p1.saludar();   // Hola, mi nombre es Ana y tengo 28 años.
        p2.saludar();   // Hola, mi nombre es Carlos y tengo 34 años.
    }
}
```

### 3.1.5 El operador punto (`.`)

El operador punto permite acceder a los miembros (atributos y métodos) de un objeto:

```java
Coche miCoche = new Coche();
miCoche.marca = "Toyota";       // acceso a atributo
miCoche.modelo = "Corolla";     // acceso a atributo
miCoche.acelerar(40);           // invocación de método
miCoche.mostrarEstado();        // invocación de método
```

### 3.1.6 Estado y comportamiento

- El **estado** de un objeto está determinado por los valores de sus atributos en un momento dado. Dos objetos de la misma clase pueden tener estados completamente diferentes.
- El **comportamiento** está definido por los métodos que la clase expone.

```
┌──────────────────────────────────┐
│  Clase: CuentaBancaria           │
├──────────────────────────────────┤
│  Atributos (estado):             │
│    - titular: String             │
│    - saldo: double               │
├──────────────────────────────────┤
│  Métodos (comportamiento):       │
│    - depositar(cantidad)         │
│    - retirar(cantidad)           │
│    - consultarSaldo()            │
└──────────────────────────────────┘
```

```java
public class CuentaBancaria {
    String titular;
    double saldo;

    void depositar(double cantidad) {
        saldo += cantidad;
    }

    void retirar(double cantidad) {
        if (cantidad <= saldo) {
            saldo -= cantidad;
        } else {
            System.out.println("Saldo insuficiente");
        }
    }

    double consultarSaldo() {
        return saldo;
    }
}
```

### 3.1.7 Anatomía de un objeto en memoria (heap y stack)

Cada vez que creas un objeto con `new`, la JVM reserva memoria en el _heap_. La variable que lo referencia vive en el _stack_ (si es local) o como parte del objeto contenedor. Entender esta distinción es esencial para razonar sobre el paso de parámetros, la recolección de basura y el rendimiento.

```
Stack                          Heap
┌─────────────────┐           ┌─────────────────────────┐
│ main() frame    │           │                         │
│  p1 ───────────────────────►│ Persona                 │
│                 │           │  nombre = "Ana"         │
│  p2 ───────────────────────►│  edad = 28              │
│                 │           ├─────────────────────────┤
│                 │           │ Persona                 │
│                 │           │  nombre = "Carlos"      │
│                 │           │  edad = 34              │
└─────────────────┘           └─────────────────────────┘
```

Los Strings "Ana" y "Carlos" están a su vez en el _string pool_ o en el _heap_. Cada objeto `Persona` tiene su propia copia de `nombre` (una referencia) y `edad` (un `int`).

---

## 3.2 Atributos

### 3.2.1 Variables de instancia (campos)

Los atributos declarados directamente en el cuerpo de la clase (fuera de métodos) son **variables de instancia**. Cada objeto tiene su propia copia de estas variables.

```java
public class Estudiante {
    // Variables de instancia: cada Estudiante tiene su propia copia
    String matricula;
    String nombre;
    double promedio;
}
```

### 3.2.2 Valores por defecto

En Java, las variables de instancia reciben valores por defecto si no se inicializan explícitamente. Esto no ocurre con las variables locales.

| Tipo          | Valor por defecto |
|---------------|-------------------|
| `byte`        | `0`               |
| `short`       | `0`               |
| `int`         | `0`               |
| `long`        | `0L`              |
| `float`       | `0.0f`            |
| `double`      | `0.0d`             |
| `char`        | `'\u0000'`        |
| `boolean`     | `false`           |
| Referencias   | `null`            |

```java
public class DemoValores {
    int entero;         // 0
    double decimal;     // 0.0
    boolean bandera;    // false
    String texto;       // null
    char caracter;      // '\u0000'

    void mostrar() {
        System.out.println("int: " + entero);
        System.out.println("double: " + decimal);
        System.out.println("boolean: " + bandera);
        System.out.println("String: " + texto);
        System.out.println("char: [" + caracter + "]");
    }
}
```

### 3.2.3 Variables de clase (`static`)

Un atributo declarado con `static` pertenece a la clase, no a cada instancia. Solo existe una copia compartida por todos los objetos.

```java
public class Contador {
    int id;
    static int totalObjetos = 0;

    public Contador() {
        totalObjetos++;
        id = totalObjetos;
    }

    void mostrar() {
        System.out.println("ID: " + id + ", Total: " + totalObjetos);
    }
}

public class DemoStatic {
    public static void main(String[] args) {
        Contador c1 = new Contador();  // id=1, total=1
        Contador c2 = new Contador();  // id=2, total=2
        Contador c3 = new Contador();  // id=3, total=3

        System.out.println(Contador.totalObjetos);  // 3

        c1.mostrar();  // ID: 1, Total: 3
        c2.mostrar();  // ID: 2, Total: 3
    }
}
```

| Variable de instancia                              | Variable de clase (`static`)                              |
|---------------------------------------------------|----------------------------------------------------------|
| Una copia por cada objeto                          | Una única copia compartida                                |
| Se accede con `objeto.variable`                    | Se accede con `Clase.variable` (recomendado)              |
| Se inicializa al crear el objeto                   | Se inicializa al cargar la clase                          |
| Representa el estado de un objeto concreto         | Representa información de toda la clase                   |

### 3.2.4 Atributos `final`

Un campo declarado `final` debe ser inicializado exactamente una vez (en la declaración, en un bloque de inicialización o en el constructor) y no puede ser reasignado después.

```java
public class Configuracion {
    private final String entorno;
    private final int maxConexiones;
    public static final double VERSION = 2.0;  // constante de clase

    public Configuracion(String entorno, int maxConexiones) {
        this.entorno = entorno;
        this.maxConexiones = maxConexiones;
    }
}
```

**Importante**: `final` en una referencia impide que la referencia apunte a otro objeto, pero **no** impide modificar el objeto referenciado:

```java
private final List<String> nombres = new ArrayList<>();

public void inicializar() {
    // nombres = new ArrayList<>();  // ERROR: no se puede reasignar
    nombres.add("Ana");              // OK: el contenido del objeto sí es mutable
}
```

### 3.2.5 Visibilidad entre hilos: `volatile` y `AtomicReference`

Cuando varios hilos acceden a un mismo atributo, la JVM puede mantener copias locales del valor en cachés de CPU o en registros. Esto significa que un hilo puede no ver los cambios realizados por otro. El modificador `volatile` resuelve este problema.

### `volatile`

Un campo `volatile` garantiza que:
1. Las lecturas siempre ven el valor más reciente escrito por cualquier hilo (se lee directamente de memoria principal).
2. Las escrituras son inmediatamente visibles para todos los hilos.
3. Se establece una relación _happens-before_: todo lo que un hilo escribió antes de escribir en un campo volatile es visible para cualquier hilo que lea ese campo después.

```java
public class TaskRunner {
    private volatile boolean running = true;

    public void ejecutar() {
        while (running) {
            // procesar tareas...
        }
        // Sin volatile, el compilador JIT podría optimizar
        // el bucle y no ver nunca el cambio de running = false
    }

    public void detener() {
        running = false;  // visible inmediatamente para todos los hilos
    }
}
```

#### Cómo la JVM reordena instrucciones

La JVM y la CPU pueden reordenar instrucciones por razones de rendimiento, siempre que no alteren la semántica del programa desde la perspectiva de un **único hilo**. En entornos concurrentes, este reordenamiento puede causar comportamientos catastróficos.

```java
// Ejemplo del patrón Double-Checked Locking ROTO (sin volatile)
class SingletonRoto {
    private static SingletonRoto instance;  // falta volatile
    private String datos;

    private SingletonRoto() {
        this.datos = cargarDesdeBaseDeDatos();  // operación costosa
    }

    public static SingletonRoto getInstance() {
        if (instance == null) {                       // (1)
            synchronized (SingletonRoto.class) {
                if (instance == null) {               // (2)
                    instance = new SingletonRoto();    // (3) PROBLEMA
                }
            }
        }
        return instance;                               // (4)
    }

    private static String cargarDesdeBaseDeDatos() {
        return "datos cargados...";
    }
}
```

En la línea (3), `new SingletonRoto()` implica tres pasos:
1. Reservar memoria para el objeto.
2. Ejecutar el constructor (inicializar `datos`).
3. Asignar la referencia a `instance`.

La JVM puede reordenar 2 y 3: un hilo puede ver `instance` no-null **antes** de que el constructor termine, obteniendo un objeto con `datos = null`. La solución es declarar `instance` como `volatile`:

```java
private static volatile SingletonRoto instance;
```

### `AtomicReference`

`volatile` solo garantiza visibilidad, no atomicidad. Para operaciones de lectura-escritura compuestas (como _compare-and-swap_), necesitas las clases atómicas del paquete `java.util.concurrent.atomic`.

`AtomicReference<V>` permite actualizar referencias de forma atómica sin usar `synchronized`, usando instrucciones CAS del procesador de forma eficiente.

```java
import java.util.concurrent.atomic.AtomicReference;

public class CacheConcurrente {
    private final AtomicReference<Map<String, Object>> cache =
        new AtomicReference<>(new ConcurrentHashMap<>());

    public void put(String clave, Object valor) {
        Map<String, Object> nuevo = new ConcurrentHashMap<>(cache.get());
        nuevo.put(clave, valor);
        cache.set(nuevo);  // reemplazo atómico de todo el mapa
    }

    public Object get(String clave) {
        return cache.get().get(clave);
    }

    // Actualización atómica con compareAndSet
    public boolean reemplazarSiCoincide(
            String clave, Object esperado, Object nuevoValor) {
        Map<String, Object> actual;
        do {
            actual = cache.get();
            if (!esperado.equals(actual.get(clave))) return false;
            Map<String, Object> copia = new ConcurrentHashMap<>(actual);
            copia.put(clave, nuevoValor);
        } while (!cache.compareAndSet(actual, copia));
        return true;
    }
}
```

#### El costo de la sincronización

No todo debe ser `volatile` ni `synchronized`. Cada mecanismo tiene un costo:

| Mecanismo         | Costo aproximado | Cuándo usarlo                              |
|--------------------|-------------------|--------------------------------------------|
| Sin sincronización | ~1 ns            | Datos confinados a un hilo                 |
| `volatile`         | ~5-10 ns         | Una sola variable de control (flags)       |
| `AtomicReference`  | ~20-50 ns        | Operaciones CAS sobre referencias          |
| `synchronized`     | ~100-500 ns      | Secciones críticas con múltiples variables |
| `ReentrantLock`    | ~100-400 ns      | Cuando necesitas tryLock, fairness, etc.   |

La regla práctica: empieza sin sincronización. Si hay contención, mide primero y luego decide si `volatile` o atómicos bastan. Usa `synchronized` o `Lock` solo cuando varias variables deben modificarse como una unidad atómica.

---

## 3.3 Métodos

### 3.3.1 Estructura de un método

```java
[modificador] tipoRetorno nombreMetodo([tipoParam param1, tipoParam param2, ...]) {
    // cuerpo del método
    [return valor;]
}
```

- **`tipoRetorno`**: el tipo de dato que devuelve el método (`int`, `String`, etc.) o `void` si no devuelve nada.
- **`nombreMetodo`**: por convención en _camelCase_.
- **Parámetros**: lista de tipo y nombre separados por comas.
- **`return`**: devuelve el control y opcionalmente un valor al código que invocó el método.

### 3.3.2 Retorno de valores y `void`

```java
public class Calculadora {

    // Método que retorna un valor
    int sumar(int a, int b) {
        return a + b;
    }

    // Método con return en varios puntos
    int valorAbsoluto(int n) {
        if (n < 0) {
            return -n;
        }
        return n;
    }

    // Método void: no retorna valor
    void imprimirResultado(int resultado) {
        System.out.println("Resultado: " + resultado);
    }
}
```

### 3.3.3 Paso de parámetros: por valor (deep dive)

En Java **siempre se pasa por valor**. No existe el paso por referencia. La distinción clave es qué es lo que se copia:

- **Primitivos**: se copia el valor del primitivo (un `int`, `double`, `boolean`, etc.).
- **Referencias**: se copia la referencia (el puntero al objeto), no el objeto en sí.

#### Primitivos: se copia el valor

El método recibe una copia del valor; modificar el parámetro no afecta a la variable original.

```
Antes de llamar:       Dentro del método:
┌───────────┐          ┌───────────┐
│ x = 10    │  copia   │ n = 10    │  →  n = 20 (solo cambia la copia)
└───────────┘          └───────────┘

Después de llamar:
┌───────────┐
│ x = 10    │  ← no se modifica
└───────────┘
```

```java
public class DemoPasoPrimitivo {
    void modificar(int n) {
        n = 20;   // solo modifica la copia local
    }

    public static void main(String[] args) {
        DemoPasoPrimitivo demo = new DemoPasoPrimitivo();
        int x = 10;
        demo.modificar(x);
        System.out.println(x);  // sigue siendo 10
    }
}
```

#### Referencias: se copia la referencia (no el objeto)

El método recibe una copia de la referencia, que apunta al mismo objeto. Puede **modificar el estado del objeto**, pero no puede hacer que la referencia original apunte a otro objeto.

```
Antes:                    Dentro del método:
p ──────► [nombre="Ana"]  ref ──────► [nombre="Ana"]
                                       ref.nombre = "Luis"  // modifica el objeto compartido
                                       ref = new Persona()  // solo cambia la copia de la referencia

Después:
p ──────► [nombre="Luis"]  ← el objeto sí se modificó
```

```java
public class DemoPasoReferencia {
    void modificarObjeto(Persona p) {
        p.nombre = "Luis";   // modifica el objeto original
    }

    void reasignarReferencia(Persona p) {
        p = new Persona();   // solo cambia la copia local de la referencia
        p.nombre = "María";
    }

    public static void main(String[] args) {
        DemoPasoReferencia demo = new DemoPasoReferencia();
        Persona persona = new Persona();
        persona.nombre = "Ana";

        demo.modificarObjeto(persona);
        System.out.println(persona.nombre);  // "Luis" — se modificó

        demo.reasignarReferencia(persona);
        System.out.println(persona.nombre);  // "Luis" — no cambió
    }
}
```

#### Caso especial: Arrays y paso por valor

Los arrays en Java son objetos, por lo que se aplica la misma regla: el método recibe una copia de la referencia al array. Puedes modificar los elementos, pero no reasignar la variable del llamador:

```java
public class DemoPasoArray {

    void modificarElementos(int[] arr) {
        arr[0] = 999;         // MODIFICA el array original
        arr[1] = 888;
    }

    void reasignarArray(int[] arr) {
        arr = new int[]{100, 200, 300};  // NO afecta al array original
        System.out.println("Dentro: arr[0] = " + arr[0]);  // 100
    }

    void redimensionarArray(int[] arr) {
        // Esto NO funciona: la referencia original sigue apuntando al array viejo
        int[] nuevo = new int[10];
        System.arraycopy(arr, 0, nuevo, 0, arr.length);
        arr = nuevo;  // solo cambia la copia local
    }

    public static void main(String[] args) {
        DemoPasoArray demo = new DemoPasoArray();
        int[] numeros = {1, 2, 3, 4, 5};

        demo.modificarElementos(numeros);
        System.out.println("numeros[0] = " + numeros[0]);  // 999
        System.out.println("numeros[1] = " + numeros[1]);  // 888

        demo.reasignarArray(numeros);
        System.out.println("numeros[0] = " + numeros[0]);  // sigue 999

        demo.redimensionarArray(numeros);
        System.out.println("numeros.length = " + numeros.length);  // sigue 5
    }
}
```

Este comportamiento es causa frecuente de bugs. Si necesitas que un método "devuelva" un array modificado, la opción correcta es retornarlo:

```java
public int[] añadirElemento(int[] original, int nuevoElemento) {
    int[] resultado = new int[original.length + 1];
    System.arraycopy(original, 0, resultado, 0, original.length);
    resultado[original.length] = nuevoElemento;
    return resultado;
}

// Uso:
numeros = añadirElemento(numeros, 42);  // reasignación explícita
```

### 3.3.4 Cómo funciona realmente en la JVM: bytecodes de invocación

Cuando compilas una llamada a un método, el compilador de Java genera instrucciones _bytecode_ que la JVM ejecutará. Existen cinco instrucciones distintas para invocar métodos, cada una con una semántica y rendimiento diferentes:

| Instrucción      | Uso                                                            | Resolución |
|------------------|----------------------------------------------------------------|-------------|
| `invokevirtual`  | Métodos de instancia normales (polimórficos)                   | Runtime (vtable) |
| `invokeinterface`| Métodos declarados en interfaces                               | Runtime (itable) |
| `invokestatic`   | Métodos `static`                                               | Compilación |
| `invokespecial`  | Constructores, métodos `private`, `super.metodo()`             | Compilación |
| `invokedynamic`  | Lambdas, referencias a métodos, _string concatenation_ (Java 9+) | Runtime (bootstrap) |

#### `invokevirtual` — el caballo de batalla del polimorfismo

Para un método normal de instancia, el compilador emite `invokevirtual`. La JVM resuelve la llamada usando la **tabla de métodos virtuales** (_vtable_) del tipo real del objeto:

```java
Animal a = new Perro();
a.hacerSonido();  // compila a: invokevirtual Animal.hacerSonido:()V
```

El bytecode referencia a `Animal.hacerSonido`, pero en tiempo de ejecución la JVM consulta la vtable del objeto real (`Perro`) y ejecuta `Perro.hacerSonido()`. Este mecanismo es la base del polimorfismo dinámico.

Cada clase tiene su propia vtable. La vtable de `Perro` contiene entradas para los métodos de `Object`, `Animal` y los propios de `Perro`. Cuando `Perro` sobrescribe `hacerSonido()`, la entrada correspondiente en su vtable apunta al código de `Perro.hacerSonido()` en lugar del de `Animal`.

```
vtable de Animal:                    vtable de Perro:
┌──────────────────────┐            ┌──────────────────────┐
│ toString() -> Object │            │ toString() -> Object │
│ equals()   -> Object │            │ equals()   -> Object │
│ hashCode() -> Object │            │ hashCode() -> Object │
│ comer()    -> Animal │            │ comer()    -> Animal │
│ dormir()   -> Animal │            │ dormir()   -> Animal │
│ hacerSonido() -> Animal │         │ hacerSonido() -> Perro │ ← sobrescrito
└──────────────────────┘            │ ladrar()   -> Perro │
                                    └──────────────────────┘
```

#### `invokeinterface` — polimorfismo con interfaces

Cuando invocas un método a través de una referencia de tipo interfaz, se usa `invokeinterface`. La resolución es ligeramente más costosa que `invokevirtual`, porque la JVM debe buscar en la **itable** (interface table) del objeto, que mapea interfaces a implementaciones:

```java
Volador v = new Pato("Donald");
v.volar();  // compila a: invokeinterface Volador.volar:()V
```

La JVM busca la interfaz `Volador` en la itable del objeto `Pato` y obtiene el puntero al método `Pato.volar()`. Esta indirección adicional explica por qué las llamadas a través de interfaces pueden ser imperceptiblemente más lentas que a través de clases (aunque en la práctica las optimizaciones del JIT lo mitigan casi por completo).

#### `invokestatic` — métodos de clase

Los métodos `static` se resuelven en tiempo de compilación. La JVM no necesita buscar en ninguna tabla virtual:

```java
Math.max(10, 20);  // compila a: invokestatic Math.max:(II)I
```

Al no tener despacho dinámico, los métodos estáticos son los más rápidos de invocar.

#### `invokespecial` — sin polimorfismo

Se usa para constructores (`<init>`), métodos `private` y llamadas a `super.metodo()`. No hay despacho virtual porque el método a ejecutar se conoce en compilación:

```java
public Perro(String nombre, int edad, String raza) {
    super(nombre, edad);    // invokespecial Animal.<init>:(String;I)V
    this.raza = raza;
}
```

#### `invokedynamic` — la revolución de Java 7+

Introducido para soportar lenguajes dinámicos sobre la JVM, `invokedynamic` delega la resolución del método a un **bootstrap method** definido por el compilador. En Java moderno, se usa masivamente para:

- **Lambdas**: en lugar de generar clases anónimas, el compilador usa `invokedynamic` y una _factory_ (`LambdaMetafactory`) que genera la implementación en tiempo de ejecución.
- **String concatenation** (Java 9+): `"a" + "b"` ya no compila a `StringBuilder`, sino a `invokedynamic` con un bootstrap que elige la estrategia óptima de concatenación.

```java
// Este código...
Runnable r = () -> System.out.println("Hola");

// ...compila aproximadamente a:
// invokedynamic BootstrapMethod #0  run:()Ljava/lang/Runnable;
// Donde el bootstrap method es LambdaMetafactory.metafactory()
// que genera dinámicamente una clase que implementa Runnable
```

La ventaja es que el compilador JIT puede optimizar e incluso _inline_ lambdas de formas que serían imposibles con clases anónimas tradicionales, resultando en un rendimiento comparable o superior al código imperativo equivalente.

### 3.3.5 Sobrecarga de métodos (_overloading_)

Varios métodos con el mismo nombre pero **diferente firma** (número, tipo u orden de parámetros). El compilador decide cuál llamar en tiempo de compilación según los argumentos pasados.

```java
public class Impresora {

    void imprimir(int numero) {
        System.out.println("Entero: " + numero);
    }

    void imprimir(double numero) {
        System.out.println("Decimal: " + numero);
    }

    void imprimir(String texto) {
        System.out.println("Texto: " + texto);
    }

    void imprimir(String texto, int veces) {
        for (int i = 0; i < veces; i++) {
            System.out.println(texto);
        }
    }

    // No compila si solo cambia el tipo de retorno:
    // int imprimir(String texto) { ... }   // ERROR
}

// Uso:
Impresora imp = new Impresora();
imp.imprimir(42);              // Entero: 42
imp.imprimir(3.1416);          // Decimal: 3.1416
imp.imprimir("Hola");          // Texto: Hola
imp.imprimir("Java", 3);       // Imprime Java tres veces
```

**Reglas:** el tipo de retorno no forma parte de la firma para sobrecarga. Dos métodos que solo difieran en el tipo de retorno generan error de compilación.

### 3.3.6 Argumentos de longitud variable (_varargs_)

Sintaxis: `tipo... nombre`. Permite pasar cero o más argumentos del tipo especificado. El parámetro vararg se trata como un array dentro del método. Debe ser el **último** parámetro.

```java
public class VarargsDemo {

    int sumar(int... numeros) {
        int suma = 0;
        for (int n : numeros) {
            suma += n;
        }
        return suma;
    }

    void imprimirConPrefijo(String prefijo, String... palabras) {
        for (String p : palabras) {
            System.out.println(prefijo + " " + p);
        }
    }

    public static void main(String[] args) {
        VarargsDemo vd = new VarargsDemo();

        System.out.println(vd.sumar());           // 0
        System.out.println(vd.sumar(1, 2));       // 3
        System.out.println(vd.sumar(1, 2, 3, 4)); // 10

        // También acepta un array explícito
        int[] arr = {5, 10, 15};
        System.out.println(vd.sumar(arr));        // 30

        vd.imprimirConPrefijo(">>", "Java", "Python", "Kotlin");
    }
}
```

### 3.3.7 Métodos estáticos (`static`)

Pertenecen a la clase, no a las instancias. Se invocan con `Clase.metodo()`. No pueden acceder a miembros de instancia directamente.

```java
public class Matematicas {
    public static final double PI = 3.141592653589793;

    public static int maximo(int a, int b) {
        return a > b ? a : b;
    }

    public static double areaCirculo(double radio) {
        return PI * radio * radio;
    }
}

// Uso:
double area = Matematicas.areaCirculo(5.0);
int mayor = Matematicas.maximo(10, 20);
```

---

## 3.4 Constructores

### 3.4.1 ¿Qué es un constructor?

Un constructor es un bloque de código especial que se invoca al crear un objeto con `new`. Inicializa el estado del objeto. **No tiene tipo de retorno** (ni siquiera `void`) y su nombre coincide exactamente con el nombre de la clase.

### 3.4.2 Constructor por defecto

Si no se declara ningún constructor, el compilador proporciona uno sin parámetros (constructor por defecto) que inicializa los atributos a sus valores por defecto.

```java
public class Producto {
    String nombre;   // null
    double precio;   // 0.0
    int stock;       // 0
}

// En main:
Producto p = new Producto();  // usa el constructor por defecto
System.out.println(p.nombre); // null
```

**Importante:** el compilador solo genera el constructor por defecto si no hay ningún constructor explícito. Si defines al menos uno, el constructor por defecto desaparece.

### 3.4.3 Constructores parametrizados

Permiten inicializar los atributos con valores proporcionados en el momento de la creación.

```java
public class Producto {
    String nombre;
    double precio;
    int stock;

    public Producto(String nombre, double precio, int stock) {
        this.nombre = nombre;
        this.precio = precio;
        this.stock = stock;
    }
}

// Uso:
Producto p1 = new Producto("Laptop", 999.99, 10);
Producto p2 = new Producto("Mouse", 25.50, 50);
```

### 3.4.4 Sobrecarga de constructores

Una clase puede tener múltiples constructores con diferentes parámetros, siguiendo las mismas reglas de sobrecarga que los métodos.

```java
public class Empleado {
    String nombre;
    String departamento;
    double salario;

    public Empleado(String nombre, String departamento, double salario) {
        this.nombre = nombre;
        this.departamento = departamento;
        this.salario = salario;
    }

    public Empleado(String nombre, String departamento) {
        this(nombre, departamento, 30000.0);
    }

    public Empleado(String nombre) {
        this(nombre, "Sin asignar");
    }

    void mostrar() {
        System.out.println(nombre + " | " + departamento + " | " + salario);
    }
}

// Uso:
Empleado e1 = new Empleado("Ana", "TI", 45000);
Empleado e2 = new Empleado("Carlos", "Ventas");
Empleado e3 = new Empleado("Luis");
e1.mostrar();  // Ana | TI | 45000.0
e2.mostrar();  // Carlos | Ventas | 30000.0
e3.mostrar();  // Luis | Sin asignar | 30000.0
```

### 3.4.5 `this()` — Llamada entre constructores

La palabra clave `this()` invoca a otro constructor de la **misma** clase. Debe ser la **primera instrucción** del constructor y solo puede usarse dentro de constructores.

```
Cadena de llamadas:

new Empleado("Luis")
    └─► this("Luis", "Sin asignar")
            └─► this("Luis", "Sin asignar", 30000.0)
                    └─► asigna nombre, departamento, salario
```

```java
public class Libro {
    String titulo;
    String autor;
    String isbn;
    int año;

    public Libro(String titulo, String autor, String isbn, int año) {
        this.titulo = titulo;
        this.autor = autor;
        this.isbn = isbn;
        this.año = año;
    }

    public Libro(String titulo, String autor) {
        this(titulo, autor, "N/A", 2025);
    }

    public Libro() {
        this("Sin título", "Anónimo");
    }
}
```

### 3.4.6 Bloques de inicialización

#### Bloque de inicialización de instancia

Se ejecuta cada vez que se crea un objeto, antes del constructor.

```java
public class Configuracion {
    String entorno;
    String version;

    {
        entorno = "desarrollo";
        version = "1.0.0";
        System.out.println("Bloque de inicialización de instancia ejecutado");
    }

    public Configuracion() {
        System.out.println("Constructor ejecutado");
    }
}
// Salida al hacer new Configuracion():
// Bloque de inicialización de instancia ejecutado
// Constructor ejecutado
```

#### Bloque de inicialización estático

Se ejecuta una sola vez, cuando la clase se carga en memoria por primera vez.

```java
public class BaseDatos {
    static String url;
    static String usuario;
    static int maxConexiones;

    static {
        url = "jdbc:mysql://localhost:3306/mi_db";
        usuario = "admin";
        maxConexiones = 100;
        System.out.println("Clase BaseDatos cargada e inicializada");
    }

    public BaseDatos() {
        System.out.println("Nueva instancia creada");
    }
}

// Uso:
BaseDatos db1 = new BaseDatos(); // imprime el mensaje estático + el de instancia
BaseDatos db2 = new BaseDatos(); // solo imprime el mensaje de instancia
```

**Orden de ejecución al crear un objeto:**

1. Bloques estáticos (una sola vez, al cargar la clase)
2. Inicialización de campos de instancia y bloques de instancia (en orden de aparición)
3. Cuerpo del constructor

### 3.4.7 El patrón Builder vs constructor telescópico vs setters

Cuando una clase tiene muchos parámetros de construcción (especialmente opcionales), surgen tres aproximaciones:

#### Constructor telescópico (problema)

```java
public class Pizza {
    private final String tamaño;
    private final boolean queso;
    private final boolean pepperoni;
    private final boolean champiñones;
    private final boolean aceitunas;

    public Pizza(String tamaño) {
        this(tamaño, true, false, false, false);
    }

    public Pizza(String tamaño, boolean queso) {
        this(tamaño, queso, false, false, false);
    }

    public Pizza(String tamaño, boolean queso, boolean pepperoni) {
        this(tamaño, queso, pepperoni, false, false);
    }

    public Pizza(String tamaño, boolean queso, boolean pepperoni,
                 boolean champiñones, boolean aceitunas) {
        this.tamaño = tamaño;
        this.queso = queso;
        this.pepperoni = pepperoni;
        this.champiñones = champiñones;
        this.aceitunas = aceitunas;
    }
}

// Uso: ¿qué significa cada true? ¡Ilegible!
Pizza p = new Pizza("grande", true, false, true, true);
```

El constructor telescópico escala mal: con N parámetros booleanos necesitas 2^N constructores. Es propenso a errores y poco legible.

#### JavaBeans con setters (problema)

```java
Pizza p = new Pizza();
p.setTamaño("grande");
p.setQueso(true);
p.setChampiñones(true);
// ups, ¿estará la pizza en estado inconsistente entre llamadas?
```

Con setters, el objeto puede estar en estado inconsistente durante la construcción. Además, impide la inmutabilidad.

#### Builder manual (solución recomendada)

```java
public class Pizza {
    private final String tamaño;
    private final boolean queso;
    private final boolean pepperoni;
    private final boolean champiñones;
    private final boolean aceitunas;

    private Pizza(Builder builder) {
        this.tamaño = builder.tamaño;
        this.queso = builder.queso;
        this.pepperoni = builder.pepperoni;
        this.champiñones = builder.champiñones;
        this.aceitunas = builder.aceitunas;
    }

    public static class Builder {
        private final String tamaño;  // requerido
        private boolean queso = false;
        private boolean pepperoni = false;
        private boolean champiñones = false;
        private boolean aceitunas = false;

        public Builder(String tamaño) {
            this.tamaño = Objects.requireNonNull(tamaño);
        }

        public Builder conQueso()     { this.queso = true; return this; }
        public Builder conPepperoni()  { this.pepperoni = true; return this; }
        public Builder conChampiñones(){ this.champiñones = true; return this; }
        public Builder conAceitunas()  { this.aceitunas = true; return this; }

        public Pizza build() {
            return new Pizza(this);
        }
    }
}

// Uso: legible, fluido, inmutable
Pizza pizza = new Pizza.Builder("grande")
    .conQueso()
    .conPepperoni()
    .conAceitunas()
    .build();
```

#### Builder con Lombok

Lombok reduce el boilerplate a una sola anotación:

```java
import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
public class Pizza {
    private final String tamaño;
    @Builder.Default private boolean queso = false;
    @Builder.Default private boolean pepperoni = false;
    @Builder.Default private boolean champiñones = false;
    @Builder.Default private boolean aceitunas = false;
}

// Uso generado automáticamente:
Pizza p = Pizza.builder()
    .tamaño("grande")
    .queso(true)
    .pepperoni(true)
    .build();
```

#### ¿Cuándo usar cada uno?

| Escenario                              | Usar...                |
|----------------------------------------|------------------------|
| Pocos parámetros (1-4)                 | Constructor simple     |
| Parámetros obligatorios + opcionales   | Builder                |
| Objetos que deben ser inmutables       | Builder o constructor  |
| Muchos booleanos / opciones            | Builder                |
| Frameworks que requieren setter (JPA)  | JavaBean con setters   |
| DTOs simples (todos campos requeridos) | Record                 |

---

## 3.5 Diseño de clases efectivo

Escribir clases que funcionen es fácil. Escribir clases que sean mantenibles, extensibles y comprensibles es un arte que requiere aplicar principios de diseño probados.

### 3.5.1 Cohesión alta, acoplamiento bajo

**Cohesión** mide cuán relacionadas están las responsabilidades dentro de una clase. **Acoplamiento** mide cuánto depende una clase de otras.

El objetivo universal: **alta cohesión + bajo acoplamiento**.

| Alta cohesión (bueno)                        | Baja cohesión (malo)                           |
|----------------------------------------------|------------------------------------------------|
| Cada clase tiene una responsabilidad clara   | Dioses objeto: 8000 líneas que hacen de todo  |
| Los métodos usan la mayoría de los campos    | Métodos que solo usan 1 de 15 campos           |
| El nombre describe perfectamente su propósito| Nombre vago: `Utils`, `Manager`, `Helper`      |

```java
// MAL: baja cohesión — esta clase hace demasiado
class GestorFacturas {
    void crearFactura() { /* ... */ }
    void enviarEmail() { /* ... */ }
    void guardarEnBaseDeDatos() { /* ... */ }
    void generarPDF() { /* ... */ }
    void calcularImpuestos() { /* ... */ }
    void conectarConPasarelaDePago() { /* ... */ }
}

// BIEN: alta cohesión — responsabilidades separadas
class Factura { /* datos de la factura */ }
class ServicioFacturacion {
    Factura crear(Pedido pedido) { /* ... */ }
}
class GeneradorPDF { byte[] generar(Factura factura) { /* ... */ } }
class EnviadorEmail { void enviar(String destinatario, byte[] adjunto) { /* ... */ } }
class PasarelaPago { void procesar(Factura factura) { /* ... */ } }
```

### 3.5.2 Tell, Don't Ask (Di, no preguntes)

Este principio dicta que en lugar de pedir datos a un objeto para tomar decisiones sobre él, debemos **decirle al objeto que haga algo** y dejar que él decida internamente cómo hacerlo.

```java
// MAL: preguntar y luego decidir (rompe el encapsulamiento)
if (cuenta.getSaldo() >= cantidad) {
    cuenta.setSaldo(cuenta.getSaldo() - cantidad);
    System.out.println("Retiro exitoso");
} else {
    System.out.println("Saldo insuficiente");
}

// BIEN: decirle a la cuenta que se encargue
cuenta.retirar(cantidad);
```

Otro ejemplo con una colección encapsulada:

```java
// MAL: exponer la lista y dejar que otros la manipulen
class Pedido {
    private List<LineaPedido> lineas;

    public List<LineaPedido> getLineas() {
        return lineas;  // expone la lista mutable
    }
}

// En otra clase:
pedido.getLineas().add(nuevaLinea);      // modifica desde fuera
pedido.getLineas().clear();              // ¡desastre!

// BIEN: Tell Don't Ask
class Pedido {
    private final List<LineaPedido> lineas = new ArrayList<>();

    public void añadirLinea(Producto p, int cantidad) {
        lineas.add(new LineaPedido(p, cantidad));
    }

    public List<LineaPedido> getLineas() {
        return Collections.unmodifiableList(lineas);  // vista inmutable
    }
}
```

### 3.5.3 Law of Demeter (Principio del menor conocimiento)

La Ley de Demeter establece que un método solo debe invocar métodos de:

1. El propio objeto (`this`)
2. Los parámetros recibidos
3. Objetos creados dentro del método
4. Componentes directos del objeto (sus campos)

En la práctica: **no encadenes llamadas** como `a.getB().getC().doSomething()`. Esto crea un acoplamiento a toda la cadena de tipos intermedios.

```java
// MAL: violación de Demeter — acoplado a estructura interna
class ServicioEnvio {
    void enviar(Pedido pedido) {
        String ciudad = pedido.getCliente().getDireccion().getCiudad();
        String codigoPostal = pedido.getCliente().getDireccion().getCodigoPostal();
    }
}

// BIEN: el pedido sabe lo necesario para el envío
class Pedido {
    private Cliente cliente;

    public DireccionEnvio getDireccionEnvio() {
        return cliente.getDireccionEnvio();  // encapsula la navegación
    }
}

class ServicioEnvio {
    void enviar(Pedido pedido) {
        DireccionEnvio dir = pedido.getDireccionEnvio();
    }
}
```

El problema no es estético, es de mantenimiento. Si la estructura interna de `Cliente` cambia (por ejemplo, `getDireccion()` pasa a llamarse `getDireccionPrincipal()`), cada encadenamiento en el código debe actualizarse. Con Demeter, solo cambias un método en `Pedido`.

Ejemplos adicionales:

```java
// Encadenamiento excesivo: frágil y difícil de testear
String zip = orden.getUsuario().getPerfil().getDireccion().getCodigoPostal().toUpperCase();

// Refactor: cada nivel expone solo lo necesario
String zip = orden.getCodigoPostalEnvio();

// Otro caso clásico
// MAL:
factura.getCliente().getHistorial().getUltimaCompra().getFecha();
// BIEN:
factura.getFechaUltimaCompraCliente();
```

### 3.5.4 Clases pequeñas y enfocadas

El Principio de Responsabilidad Única (SRP, la S de SOLID) dice que una clase debe tener un único motivo para cambiar. En términos prácticos: si una clase tiene más de ~200-300 líneas, probablemente hace demasiado.

**Señales de que una clase debería dividirse:**
- Tiene secciones claramente separadas con comentarios tipo `// --- SECCIÓN X ---`
- Algunos métodos no usan ningún campo de instancia (deberían ser estáticos o moverse a otra clase)
- La clase depende de demasiadas clases externas (más de 5-7 imports de dominio suele ser sospechoso)
- Nombre con "y" u "o": `GestorDePedidosYFacturas`, `ValidadorOCalculadora`

```java
// ANTES: clase con demasiadas responsabilidades (~500 líneas)
class ProcesadorPedido {
    boolean validar(Pedido p) { /* ... */ }
    double calcularImpuestos(Pedido p) { /* ... */ }
    boolean procesarPago(Pago pago) { /* ... */ }
    void actualizarInventario(Pedido p) { /* ... */ }
    void enviarConfirmacion(Pedido p) { /* ... */ }
    String generarEtiqueta(Pedido p) { /* ... */ }
}

// DESPUÉS: responsabilidades separadas — cada clase tiene un motivo para cambiar
class ValidadorPedido { boolean validar(Pedido p) { /* ... */ } }
class CalculadoraImpuestos { double calcular(Pedido p) { /* ... */ } }
class ProcesadorPago { boolean procesar(Pago pago) { /* ... */ } }
class GestorInventario { void actualizar(Pedido p) { /* ... */ } }
class NotificadorPedido { void enviarConfirmacion(Pedido p) { /* ... */ } }
class GeneradorEtiqueta { String generar(Pedido p) { /* ... */ } }

class ServicioProcesamientoPedido {
    void procesar(Pedido p) {
        validador.validar(p);
        procesadorPago.procesar(p.getPago());
        gestorInventario.actualizar(p);
        notificador.enviarConfirmacion(p);
    }
}
```

### 3.5.5 Nombrar bien: el 80% del diseño

Nombrar es una de las decisiones de diseño más infravaloradas. Un buen nombre:
- Evita tener que leer la implementación para entender qué hace algo.
- Sirve como documentación viva.
- Revela si la clase/método tiene una responsabilidad clara.

#### Métodos con verbos

```java
// MAL: nombres ambiguos
void proceso();          // ¿qué proceso?
List<String> datos();    // ¿qué datos?
boolean check();         // ¿qué comprueba?

// BIEN: verbo + sustantivo
void enviarConfirmacionPedido();
List<String> obtenerNombresClientesActivos();
boolean tieneSaldoSuficiente(double cantidad);
```

#### Booleanos con "is", "has", "can", "should"

```java
// Métodos booleanos: deben leerse como pregunta
if (usuario.isActivo()) { /* ... */ }
if (pedido.hasDescuento()) { /* ... */ }
if (cuenta.canWithdraw(cantidad)) { /* ... */ }
if (archivo.shouldOverwrite()) { /* ... */ }

// Variables booleanas: la misma convención
private boolean isActive;
private boolean hasChildren;
private boolean canEdit;
```

#### Nombres que revelan intención

```java
// MAL: nombres que no revelan intención
int d;             // ¿días? ¿dólares?
String s;          // siempre mal nombre excepto en lambdas triviales
List<int[]> theList;  // ¿lista de qué?

// BIEN
int diasTranscurridosDesdeEnvio;
String codigoIsoPais;
List<Coordenada> puntosDeEntrega;

// Nombres pronunciables y buscables
// MAL:
String genymdhms;  // ¡generate year month day hour minute second?

// BIEN:
String timestampGeneracion;

// Evitar redundancia con el contexto
// Si ya estás en la clase Factura:
class Factura {
    // MAL                               // BIEN
    String facturaNumero;                String numero;
    Date facturaFechaEmision;            Date fechaEmision;
    BigDecimal facturaTotal;             BigDecimal total;
}
```
---

## 3.6 Encapsulamiento y Modificadores de Acceso

### 3.6.1 Los cuatro niveles de acceso

| Modificador    | Clase | Paquete | Subclase | Cualquiera |
|----------------|-------|---------|----------|------------|
| `private`      | Sí    | No      | No       | No         |
| (default)      | Sí    | Sí      | No       | No         |
| `protected`    | Sí    | Sí      | Sí       | No         |
| `public`       | Sí    | Sí      | Sí       | Sí         |

```java
package com.empresa.modelo;

public class Persona {
    private String dni;          // solo accesible dentro de esta clase
    String nombre;               // accesible dentro del paquete (default)
    protected String email;      // accesible en paquete + subclases
    public String nacionalidad;  // accesible desde cualquier lugar

    public Persona(String dni, String nombre) {
        this.dni = dni;
        this.nombre = nombre;
    }
}
```

```java
package com.empresa.servicio;

import com.empresa.modelo.Persona;

public class Cliente extends Persona {

    public Cliente(String dni, String nombre) {
        super(dni, nombre);
    }

    void mostrarDatos() {
        // System.out.println(dni);     // ERROR: private
        // System.out.println(nombre);  // ERROR: default (distinto paquete)
        System.out.println(email);      // OK: protected (es subclase)
        System.out.println(nacionalidad); // OK: public
    }
}
```

### 3.6.2 Getters y Setters

Para cumplir con el principio de encapsulamiento, los atributos se declaran `private` y se exponen mediante métodos públicos de acceso (_getters_) y modificación (_setters_).

```java
public class CuentaBancaria {
    private String titular;
    private double saldo;

    public CuentaBancaria(String titular, double saldoInicial) {
        this.titular = titular;
        if (saldoInicial >= 0) {
            this.saldo = saldoInicial;
        }
    }

    public String getTitular() {
        return titular;
    }

    public double getSaldo() {
        return saldo;
    }

    public void setTitular(String titular) {
        if (titular != null && !titular.isBlank()) {
            this.titular = titular;
        }
    }

    public void depositar(double cantidad) {
        if (cantidad > 0) {
            saldo += cantidad;
        }
    }

    public boolean retirar(double cantidad) {
        if (cantidad > 0 && cantidad <= saldo) {
            saldo -= cantidad;
            return true;
        }
        return false;
    }
}
```

### 3.6.3 ¿Por qué importa el encapsulamiento?

1. **Control de acceso:** los atributos no pueden ser modificados directamente desde fuera.
2. **Validación:** se pueden validar los valores antes de asignarlos.
3. **Flexibilidad:** se puede cambiar la implementación interna sin afectar al código cliente.
4. **Ocultación de complejidad:** el usuario de la clase no necesita saber cómo funciona internamente.

```java
// MAL: acceso directo sin control
persona.edad = -5;   // estado inválido

// BIEN: validación en el setter
public void setEdad(int edad) {
    if (edad >= 0 && edad <= 150) {
        this.edad = edad;
    } else {
        throw new IllegalArgumentException("Edad inválida: " + edad);
    }
}
```

### 3.6.4 La palabra clave `this`

`this` es una referencia al objeto actual. Tiene tres usos principales:

#### 1. Desambiguar campos de parámetros

```java
public class Rectangulo {
    private int ancho;
    private int alto;

    public Rectangulo(int ancho, int alto) {
        this.ancho = ancho;
        this.alto = alto;
    }
}
```

#### 2. Retornar la instancia actual (_method chaining_)

```java
public class ConstructorHTML {
    private StringBuilder html = new StringBuilder();

    public ConstructorHTML añadirEncabezado(String texto) {
        html.append("<h1>").append(texto).append("</h1>\n");
        return this;
    }

    public ConstructorHTML añadirParrafo(String texto) {
        html.append("<p>").append(texto).append("</p>\n");
        return this;
    }

    public String construir() {
        return html.toString();
    }
}

// Uso con encadenamiento:
String resultado = new ConstructorHTML()
    .añadirEncabezado("Título")
    .añadirParrafo("Contenido")
    .construir();
```

#### 3. Invocar otro constructor de la misma clase

```java
public class Configuracion {
    private String host;
    private int puerto;

    public Configuracion(String host, int puerto) {
        this.host = host;
        this.puerto = puerto;
    }

    public Configuracion() {
        this("localhost", 8080);
    }
}
```

### 3.6.5 Convención JavaBeans

Un JavaBean es una clase que sigue estas reglas:

- Constructor público sin argumentos.
- Atributos `private`.
- Getters y setters públicos con nombres `getPropiedad()` / `setPropiedad(valor)`.
- Para booleanos, el getter puede ser `isPropiedad()`.
- Implementa `java.io.Serializable` (opcional, pero común).

```java
import java.io.Serializable;

public class Alumno implements Serializable {
    private static final long serialVersionUID = 1L;

    private String matricula;
    private String nombre;
    private int edad;
    private boolean activo;

    public Alumno() { }

    public String getMatricula() { return matricula; }
    public void setMatricula(String matricula) { this.matricula = matricula; }

    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }

    public int getEdad() { return edad; }
    public void setEdad(int edad) { this.edad = edad; }

    public boolean isActivo() { return activo; }
    public void setActivo(boolean activo) { this.activo = activo; }
}
```

---

## 3.7 Herencia

### 3.7.1 `extends`: heredar de una clase

La herencia permite que una clase (subclase) adquiera los atributos y métodos de otra (superclase). Se usa la palabra clave `extends`.

```java
// Superclase
public class Animal {
    protected String nombre;
    protected int edad;

    public Animal(String nombre, int edad) {
        this.nombre = nombre;
        this.edad = edad;
    }

    public void comer() {
        System.out.println(nombre + " está comiendo");
    }

    public void dormir() {
        System.out.println(nombre + " está durmiendo");
    }
}
```

```java
// Subclase
public class Perro extends Animal {
    private String raza;

    public Perro(String nombre, int edad, String raza) {
        super(nombre, edad);   // llama al constructor de Animal
        this.raza = raza;
    }

    public void ladrar() {
        System.out.println(nombre + " dice: ¡Guau!");
    }
}
```

```java
// Otra subclase
public class Gato extends Animal {
    private boolean esIndependiente;

    public Gato(String nombre, int edad, boolean esIndependiente) {
        super(nombre, edad);
        this.esIndependiente = esIndependiente;
    }

    public void maullar() {
        System.out.println(nombre + " dice: ¡Miau!");
    }
}
```

```java
// Uso:
Perro p = new Perro("Rex", 3, "Pastor Alemán");
p.comer();   // heredado de Animal
p.dormir();  // heredado de Animal
p.ladrar();  // propio de Perro

Gato g = new Gato("Luna", 2, true);
g.comer();   // heredado
g.maullar(); // propio
```

### 3.7.2 ¿Qué se hereda?

| Se hereda                                         | No se hereda                                          |
|---------------------------------------------------|-------------------------------------------------------|
| Métodos `public` y `protected`                    | Constructores                                         |
| Atributos `public` y `protected`                  | Miembros `private` (existen pero no son accesibles)   |
| Miembros con acceso default (mismo paquete)       | Métodos `static` (se ocultan, no se heredan)          |

### 3.7.3 La palabra clave `super`

`super` hace referencia a la superclase inmediata y se usa para:

#### Llamar al constructor de la superclase

```java
public class Vehiculo {
    protected String marca;
    protected int año;

    public Vehiculo(String marca, int año) {
        this.marca = marca;
        this.año = año;
    }
}

public class Moto extends Vehiculo {
    private int cilindrada;

    public Moto(String marca, int año, int cilindrada) {
        super(marca, año);     // debe ser la primera instrucción
        this.cilindrada = cilindrada;
    }
}
```

Si no se llama a `super()` explícitamente, el compilador inserta `super()` (constructor sin argumentos). Si la superclase no tiene constructor sin argumentos, hay error de compilación.

#### Acceder a métodos de la superclase

```java
public class Ave {
    public void cantar() {
        System.out.println("El ave canta una melodía");
    }
}

public class Canario extends Ave {
    @Override
    public void cantar() {
        super.cantar();                              // llama al método original
        System.out.println("y añade un trino especial");
    }
}
```

### 3.7.4 El problema del frágil base class

Uno de los problemas más insidiosos de la herencia es el **frágil base class**: cambios aparentemente inocentes en la superclase pueden romper las subclases de forma impredecible.

#### Escenario 1: añadir un método a la superclase

```java
// Superclase original (v1)
public class InstrumentoMusical {
    protected String nombre;

    public InstrumentoMusical(String nombre) {
        this.nombre = nombre;
    }

    public void afinar() {
        System.out.println("Afinando " + nombre);
    }
}

// Subclase nuestra (v1)
public class Guitarra extends InstrumentoMusical {
    private int numeroCuerdas;

    public Guitarra(String nombre, int numeroCuerdas) {
        super(nombre);
        this.numeroCuerdas = numeroCuerdas;
    }

    @Override
    public void afinar() {
        ajustarTensionCuerdas();
        super.afinar();
    }

    public void preparar() {
        ajustarTensionCuerdas();
        System.out.println("Guitarra lista");
    }

    private void ajustarTensionCuerdas() {
        System.out.println("Ajustando tensión de " + numeroCuerdas + " cuerdas");
    }
}
```

```java
// Superclase — versión 2 (el autor añade un método nuevo)
public class InstrumentoMusical {
    // ... código existente ...

    // Nuevo método añadido en v2; el autor no conoce Guitarra
    public void preparar() {
        System.out.println("Preparando instrumento " + nombre);
        afinar();  // llama a afinar() que está sobrescrito en Guitarra
    }
}
// Ahora InstrumentoMusical.preparar() y Guitarra.preparar()
// colisionan. El despacho dinámico ejecutará Guitarra.preparar(),
// rompiendo la intención original de la superclase.
```

#### Escenario 2: self-use problem (método de superclase que llama a hooks sobrescritos)

```java
public class Contenedor {
    public void guardar(Object elemento) {
        validar(elemento);        // puede estar sobrescrito
        almacenarInternamente(elemento);
        incrementarContador();    // puede estar sobrescrito
    }

    protected void validar(Object elemento) {
        if (elemento == null) throw new NullPointerException();
    }

    protected void incrementarContador() {
        // lógica de conteo
    }

    private void almacenarInternamente(Object elemento) {
        // privado: seguro, no se puede sobrescribir
    }
}

public class ContenedorLimitado extends Contenedor {
    private final int capacidad;
    private int ocupados = 0;

    public ContenedorLimitado(int capacidad) {
        this.capacidad = capacidad;
    }

    @Override
    protected void incrementarContador() {
        ocupados++;
        if (ocupados > capacidad) {
            throw new IllegalStateException("Capacidad excedida");
        }
    }
}
```

Este problema es tan común que Josh Bloch, en _Effective Java_, dedica un ítem entero a advertir: **"Diseña y documenta la herencia, o prohíbela"**. La superclase debe documentar explícitamente qué métodos son "hooks" diseñados para ser sobrescritos:

```java
/**
 * Template method para guardar elementos.
 * Las subclases pueden sobrescribir {@link #validar} y
 * {@link #incrementarContador}, pero deben llamar a super
 * si necesitan la validación/conteo base.
 */
public void guardar(Object elemento) { /* ... */ }
```

### 3.7.5 Herencia vs Composición: La decisión más importante del diseño OO

Existe un principio que todo programador Java debería grabarse a fuego:

> **Prefiere composición sobre herencia.**
> — *Effective Java*, 3ª edición, Item 18

No es que la herencia sea mala. Es que la herencia **mal aplicada** es desastrosa. Y el umbral para aplicarla bien es alto: solo cuando existe una relación genuina de **IS-A (es-un)** verificable y estable.

#### Caso real 1: `java.util.Stack` extiende `Vector`

Este es probablemente el error de diseño más famoso de la biblioteca estándar de Java:

```java
public class Stack<E> extends Vector<E> {
    public E push(E item)  { addElement(item); return item; }
    public E pop()         { /* ... */ }
    public E peek()        { /* ... */ }
    public boolean empty() { return size() == 0; }
    public int search(Object o) { /* ... */ }
}
```

**Por qué fue un error:**

```java
Stack<String> pila = new Stack<>();
pila.push("A");
pila.push("B");
pila.push("C");

// Stack es un Vector, así que puedo hacer esto:
pila.add(0, "X");           // insertar en el medio — rompe LIFO
pila.remove(1);              // eliminar elemento arbitrario
pila.get(2);                 // acceso aleatorio — no es una pila
pila.forEach(System.out::println); // iterar — ¿orden LIFO o inserción?

// Stack expone TODA la API de Vector (insertElementAt, removeElementAt, set...)
// Destruyendo completamente la semántica LIFO que define a una pila.
```

La clase `Stack` **no ES-UN** `Vector` con operaciones extra. Es una estructura con semántica LIFO que **USA** una lista internamente. La solución con composición:

```java
public class PilaSegura<E> {
    private final List<E> elementos = new ArrayList<>();

    public void push(E item) {
        elementos.add(item);
    }

    public E pop() {
        if (elementos.isEmpty()) throw new EmptyStackException();
        return elementos.remove(elementos.size() - 1);
    }

    public E peek() {
        if (elementos.isEmpty()) throw new EmptyStackException();
        return elementos.get(elementos.size() - 1);
    }

    public boolean isEmpty() {
        return elementos.isEmpty();
    }

    public int size() {
        return elementos.size();
    }
}
```

#### Caso real 2: `java.util.Properties` extiende `Hashtable`

Otro error histórico de la JDK. `Properties` es un mapa de String → String para configuración, pero como extiende `Hashtable<Object, Object>`:

```java
Properties props = new Properties();
props.setProperty("database.url", "jdbc:mysql://localhost/db");
props.setProperty("database.user", "admin");

// Pero como Properties extiende Hashtable...
props.put("database.password", 12345);  // ¡valor no-String! compila sin error

// También:
// props.put(new Object(), new Object());  // compila, aunque no tiene sentido
```

**Composición bien aplicada:**

```java
public class ConfiguracionSegura {
    private final Map<String, String> propiedades = new HashMap<>();

    public void set(String clave, String valor) {
        Objects.requireNonNull(clave);
        Objects.requireNonNull(valor);
        propiedades.put(clave, valor);
    }

    public String get(String clave) {
        return propiedades.get(clave);
    }

    public String getOrDefault(String clave, String valorPorDefecto) {
        return propiedades.getOrDefault(clave, valorPorDefecto);
    }

    public Set<String> claves() {
        return Collections.unmodifiableSet(propiedades.keySet());
    }
}
```

#### Cómo aplicar composición: wrapper/decorator y forwarding methods

La composición consiste en:
1. Tu clase tiene un campo privado del tipo que quieres "extender".
2. Tu clase implementa la misma interfaz o expone sus propios métodos.
3. Cada método relevante "reenvía" (_forwards_) la llamada al objeto interno.

```java
// En lugar de: MiSet extiende HashSet
// Hacemos:

public class ConjuntoConEstadisticas<E> implements Set<E> {
    private final Set<E> delegado;
    private int contadorAdiciones = 0;

    public ConjuntoConEstadisticas(Set<E> delegado) {
        this.delegado = Objects.requireNonNull(delegado);
    }

    // Forwarding methods con lógica adicional
    @Override
    public boolean add(E e) {
        contadorAdiciones++;
        return delegado.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        contadorAdiciones += c.size();
        return delegado.addAll(c);
    }

    // Resto de forwarding methods puros
    @Override public int size() { return delegado.size(); }
    @Override public boolean isEmpty() { return delegado.isEmpty(); }
    @Override public boolean contains(Object o) { return delegado.contains(o); }
    @Override public Iterator<E> iterator() { return delegado.iterator(); }
    @Override public Object[] toArray() { return delegado.toArray(); }
    @Override public <T> T[] toArray(T[] a) { return delegado.toArray(a); }
    @Override public boolean remove(Object o) { return delegado.remove(o); }
    @Override public boolean containsAll(Collection<?> c) { return delegado.containsAll(c); }
    @Override public boolean removeAll(Collection<?> c) { return delegado.removeAll(c); }
    @Override public boolean retainAll(Collection<?> c) { return delegado.retainAll(c); }
    @Override public void clear() { delegado.clear(); }

    // Método propio
    public int getContadorAdiciones() {
        return contadorAdiciones;
    }
}
```

Si implementar 12 métodos de forwarding te parece tedioso, tienes razón. Pero:
1. Los IDEs generan automáticamente _delegate methods_.
2. Lombok ofrece `@Delegate` (aunque es experimental).
3. El costo de implementar forwarding se paga una vez; el costo de una herencia incorrecta se paga para siempre.

#### ¿Cuándo sí usar herencia?

La herencia es adecuada cuando se cumplen TODAS estas condiciones:
- Existe una relación IS-A genuina y natural: un `Perro` ES-UN `Animal`, un `Circulo` ES-UNA `Figura`.
- La superclase fue **diseñada explícitamente para ser extendida** (documenta sus hooks, protege su invariante interno).
- Puedes controlar ambas clases (están en tu código, no en una dependencia externa).
- No necesitas cambiar la implementación de la superclase en tiempo de ejecución (para eso está la composición con inyección de dependencias).

### 3.7.6 Sealed classes (Java 17) — Restringir la herencia

Las _sealed classes_ son la respuesta de Java al problema "cómo permitir herencia controlada sin exponer todo". Una clase sellada declara explícitamente qué clases pueden extenderla mediante la cláusula `permits`.

```java
// Solo estas tres clases pueden extender Forma
public sealed class Forma
    permits Circulo, Rectangulo, Triangulo {

    public abstract double area();
}

public final class Circulo extends Forma {
    private final double radio;

    public Circulo(double radio) { this.radio = radio; }

    @Override
    public double area() { return Math.PI * radio * radio; }
}

public non-sealed class Rectangulo extends Forma {
    private final double ancho, alto;

    public Rectangulo(double ancho, double alto) {
        this.ancho = ancho;
        this.alto = alto;
    }

    @Override
    public double area() { return ancho * alto; }
}

public final class Triangulo extends Forma {
    private final double base, altura;

    public Triangulo(double base, double altura) {
        this.base = base;
        this.altura = altura;
    }

    @Override
    public double area() { return (base * altura) / 2; }
}
```

**Reglas de las sealed classes:**
- La clase sellada y sus subclases permitidas deben estar en el mismo módulo (Java 9+) o paquete.
- Cada subclase debe declarar `final`, `sealed` o `non-sealed`.
- Las interfaces también pueden ser selladas.

**Casos de uso ideales:**

```java
// Patrón: modelado de estados finitos (máquina de estados)
public sealed interface EstadoPedido
    permits Pendiente, Procesando, Enviado, Entregado, Cancelado { }

public record Pendiente(LocalDateTime creado) implements EstadoPedido { }
public record Procesando(String operadorId) implements EstadoPedido { }
public record Enviado(String trackingNumber) implements EstadoPedido { }
public record Entregado(LocalDateTime fechaEntrega) implements EstadoPedido { }
public record Cancelado(String motivo) implements EstadoPedido { }

// El compilador garantiza que manejamos todos los estados:
public String describirEstado(EstadoPedido estado) {
    return switch (estado) {
        case Pendiente p -> "Pendiente desde " + p.creado();
        case Procesando pr -> "Procesando por " + pr.operadorId();
        case Enviado e -> "Enviado: tracking " + e.trackingNumber();
        case Entregado en -> "Entregado el " + en.fechaEntrega();
        case Cancelado c -> "Cancelado: " + c.motivo();
    };
    // ¡Sin default! El compilador sabe que cubrimos todas las opciones.
}
```

```java
// Patrón: Result monádico (éxito o error)
public sealed interface Resultado<T>
    permits Exito, Fallo { }

public record Exito<T>(T valor) implements Resultado<T> { }
public record Fallo<T>(String mensaje, Exception causa) implements Resultado<T> { }

// Uso con pattern matching exhaustivo
public String procesar(Resultado<String> resultado) {
    return switch (resultado) {
        case Exito<String> e -> "OK: " + e.valor();
        case Fallo<String> f -> "ERROR: " + f.mensaje();
    };
}
```

La diferencia con los enums es crucial: los enums tienen un conjunto fijo de instancias constantes, mientras que las sealed classes permiten **infinitas instancias** de un número **finito de tipos**.

### 3.7.7 Sobrescritura de métodos (_overriding_)

Una subclase puede redefinir un método heredado para proporcionar una implementación específica.

**Reglas:**
- Mismo nombre, mismos parámetros y mismo tipo de retorno (o subtipo — retorno covariante).
- El modificador de acceso no puede ser más restrictivo (sí más permisivo).
- No puede lanzar excepciones comprobables nuevas o más amplias.
- Usar la anotación `@Override` para que el compilador verifique la sobrescritura.

```java
public class Figura {
    public double calcularArea() {
        return 0.0;
    }

    public String getTipo() {
        return "Figura genérica";
    }
}

public class Circulo extends Figura {
    private double radio;

    public Circulo(double radio) {
        this.radio = radio;
    }

    @Override
    public double calcularArea() {
        return Math.PI * radio * radio;
    }

    @Override
    public String getTipo() {
        return "Círculo";
    }
}

public class Rectangulo extends Figura {
    private double ancho, alto;

    public Rectangulo(double ancho, double alto) {
        this.ancho = ancho;
        this.alto = alto;
    }

    @Override
    public double calcularArea() {
        return ancho * alto;
    }

    @Override
    public String getTipo() {
        return "Rectángulo";
    }
}
```

### 3.7.8 Sobrecarga vs. Sobrescritura

| Característica            | Sobrecarga (_overloading_)                | Sobrescritura (_overriding_)                 |
|---------------------------|-------------------------------------------|----------------------------------------------|
| ¿Dónde ocurre?            | Misma clase o jerarquía                   | Entre superclase y subclase                  |
| Parámetros                | Deben ser diferentes                      | Deben ser iguales                            |
| Tipo de retorno           | Puede variar                              | Igual o subtipo (covariante)                 |
| Resolución                | Tiempo de compilación                     | Tiempo de ejecución                          |
| Anotación                 | No hay                                    | `@Override` (recomendado)                    |
| Modificador de acceso     | Puede ser cualquiera                      | No puede ser más restrictivo                 |
| `static`                  | Pueden ser estáticos o de instancia       | No se pueden sobrescribir métodos estáticos  |
| Palabra clave             | Ninguna                                   | `super.metodo()` para invocar al padre       |

```java
public class EjemploSobrecargaSobrescritura {

    // SOBRECARGA: mismo nombre, distintos parámetros
    public int sumar(int a, int b) { return a + b; }
    public int sumar(int a, int b, int c) { return a + b + c; }
    public double sumar(double a, double b) { return a + b; }
}

class Vehiculo {
    public void mover() {
        System.out.println("El vehículo se mueve");
    }
}

class Bicicleta extends Vehiculo {
    // SOBRESCRITURA: misma firma, implementación específica
    @Override
    public void mover() {
        System.out.println("La bicicleta pedalea");
    }
}
```

### 3.7.9 Herencia simple en Java

Java solo permite **herencia simple** de clases (una clase solo puede extender una superclase). Esto evita problemas como el _diamante mortal_ de la herencia múltiple de C++. Para lograr funcionalidad similar, se usan **interfaces** (sección 3.8).

```
        Animal                ┌──────────┐
          ▲                   │  Animal  │
          │                   └────┬─────┘
   ┌──────┴──────┐            ┌────┴────┐
   │             │            │         │
 Perro         Gato        Perro      Gato
   ▲             ▲
   │             │
   └──► SI       └──► SI
       (Herencia simple, legal)

        Animal      Mascota     ┌──────────┐  ┌──────────┐
          ▲           ▲         │  Animal  │  │  Mascota │
          │           │         └────┬─────┘  └────┬─────┘
          └─────┬─────┘              └──────┬──────┘
                │                           │
             Perro                      ┌───┴───┐
           (Herencia múltiple           │ Perro │
            de clases: NO permitido)    └───────┘
                                        (No compila)
```

### 3.7.10 Clases y métodos `final`

- Una **clase `final`** no puede ser extendida.
- Un **método `final`** no puede ser sobrescrito.

```java
// Clase final: no se puede heredar de ella
public final class StringUtils {
    public static boolean esVacio(String s) {
        return s == null || s.isEmpty();
    }
}

// Esto no compilaría:
// public class StringUtilsExtendida extends StringUtils { }  // ERROR

public class OperacionMatematica {
    public final double PI() {
        return 3.141592653589793;
    }

    public double calcular(double a, double b) {
        return a + b;
    }
}

public class OperacionAvanzada extends OperacionMatematica {
    @Override
    public double calcular(double a, double b) {
        return a * b;  // OK: calcular() no es final
    }

    // @Override
    // public double PI() { return 3.14; }  // ERROR: PI() es final
}
```

---

## 3.8 Polimorfismo

### 3.8.1 Tipos de polimorfismo

| Tipo                    | Resolución          | También llamado         | Mecanismo                        |
|-------------------------|---------------------|-------------------------|----------------------------------|
| Polimorfismo estático   | Tiempo de compilación | _Compile-time_         | Sobrecarga de métodos            |
| Polimorfismo dinámico   | Tiempo de ejecución   | _Runtime_              | Sobrescritura de métodos         |

### 3.8.2 Polimorfismo estático (sobrecarga)

El compilador decide qué versión del método llamar basándose en el número y tipo de argumentos.

### 3.8.3 Polimorfismo dinámico (_Dynamic Method Dispatch_)

Es el mecanismo por el cual una llamada a un método sobrescrito se resuelve en **tiempo de ejecución** según el tipo real del objeto (no el tipo de la referencia).

```java
class Animal {
    public void hacerSonido() {
        System.out.println("Sonido genérico de animal");
    }
}

class Perro extends Animal {
    @Override
    public void hacerSonido() {
        System.out.println("¡Guau guau!");
    }
}

class Gato extends Animal {
    @Override
    public void hacerSonido() {
        System.out.println("¡Miau miau!");
    }
}

class Vaca extends Animal {
    @Override
    public void hacerSonido() {
        System.out.println("¡Muuu!");
    }
}

public class DemoPolimorfismo {
    public static void main(String[] args) {
        Animal[] granja = new Animal[3];
        granja[0] = new Perro();    // upcasting implícito
        granja[1] = new Gato();     // upcasting implícito
        granja[2] = new Vaca();     // upcasting implícito

        for (Animal a : granja) {
            a.hacerSonido();
        }
    }
}
// Salida: ¡Guau guau! / ¡Miau miau! / ¡Muuu!
```

**¿Cómo funciona?** La JVM mantiene una tabla de métodos virtuales (_vtable_) por cada clase. Cuando se invoca un método, la JVM busca en la vtable del tipo real del objeto, no del tipo de la referencia.

```
Referencia (tipo Animal)     Objeto real (tipo Perro)
┌──────────────────────┐     ┌──────────────────────┐
│ a ───────────────────┼────►│ hacerSonido() ->     │
│                      │     │   "¡Guau guau!"      │
│ a.hacerSonido()      │     │                      │
└──────────────────────┘     └──────────────────────┘

La JVM resuelve a.hacerSonido() usando el tipo real (Perro),
no el tipo declarado de la variable (Animal).
```

### 3.8.4 _Upcasting_ y _Downcasting_

#### Upcasting (implícito)

Convertir una referencia de subclase a una de superclase. Siempre seguro, implícito.

```java
Perro miPerro = new Perro("Rex", 3, "Labrador");
Animal animalRef = miPerro;   // upcasting: Perro → Animal (automático)

// animalRef.ladrar();  // ERROR: la referencia Animal no conoce ladrar()
animalRef.comer();       // OK: comer() está definido en Animal
```

#### Downcasting (explícito)

Convertir una referencia de superclase a una de subclase. Requiere cast explícito y puede fallar en tiempo de ejecución.

```java
Animal animalGenerico = new Perro("Rex", 3, "Labrador");

if (animalGenerico instanceof Perro) {
    Perro perroRecuperado = (Perro) animalGenerico;
    perroRecuperado.ladrar();
}

// Downcasting inseguro (lanza ClassCastException):
Animal otroAnimal = new Gato("Luna", 2, true);
// Perro peligro = (Perro) otroAnimal;   // ClassCastException en ejecución
```

### 3.8.5 `instanceof` y Pattern Matching (Java 16+)

Comprueba si un objeto es una instancia de un tipo o interfaz. A partir de Java 16, permite declarar la variable de patrón directamente:

```java
// Antes de Java 16 (verboso y propenso a error)
if (a instanceof Perro) {
    Perro p = (Perro) a;
    p.ladrar();
}

// Java 16+ (limpio y seguro)
if (a instanceof Perro p) {
    p.ladrar();   // p está en el ámbito aquí
}

// Con condición adicional
if (a instanceof Perro p && p.getEdad() > 5) {
    System.out.println("Perro mayor de 5 años");
}

// Uso con interfaces
public double calcularArea(Object forma) {
    if (forma instanceof Circulo c) {
        return Math.PI * c.getRadio() * c.getRadio();
    } else if (forma instanceof Rectangulo r) {
        return r.getAncho() * r.getAlto();
    } else if (forma instanceof Triangulo t) {
        return (t.getBase() * t.getAltura()) / 2;
    }
    throw new IllegalArgumentException("Forma no soportada: " + forma.getClass());
}
```

---

## 3.9 Clases Abstractas e Interfaces

### 3.9.1 Clases abstractas

Una clase abstracta se declara con `abstract` y **no puede ser instanciada** directamente. Puede contener tanto métodos abstractos (sin implementación) como métodos concretos (con implementación). Sirve como clase base que define una estructura común para sus subclases.

```java
public abstract class FiguraGeometrica {
    protected String color;

    public FiguraGeometrica(String color) {
        this.color = color;
    }

    public abstract double calcularArea();
    public abstract double calcularPerimetro();

    public void mostrarColor() {
        System.out.println("Color: " + color);
    }

    public String descripcion() {
        return "Soy una figura de color " + color
            + " con área " + calcularArea();
    }
}

public class Triangulo extends FiguraGeometrica {
    private double base;
    private double altura;

    public Triangulo(String color, double base, double altura) {
        super(color);
        this.base = base;
        this.altura = altura;
    }

    @Override
    public double calcularArea() {
        return (base * altura) / 2;
    }

    @Override
    public double calcularPerimetro() {
        double lado = Math.sqrt((base / 2) * (base / 2) + altura * altura);
        return base + 2 * lado;
    }
}
```

### 3.9.2 Interfaces

Una interfaz define un **contrato**: un conjunto de métodos que las clases que la implementan deben proporcionar. A partir de Java 8 y 9, las interfaces pueden incluir también métodos con implementación.

```java
public interface Volador {
    void volar();

    default void aterrizar() {
        System.out.println("Aterrizando de forma genérica...");
    }

    static String tipoDeVuelo() {
        return "Vuelo aéreo";
    }

    private void prepararVuelo() {
        System.out.println("Verificando condiciones de vuelo...");
    }

    default void despegar() {
        prepararVuelo();
        System.out.println("¡Despegando!");
    }
}

public interface Nadador {
    void nadar();

    default void flotar() {
        System.out.println("Flotando en el agua...");
    }
}

public class Pato implements Volador, Nadador {
    private String nombre;

    public Pato(String nombre) {
        this.nombre = nombre;
    }

    @Override
    public void volar() {
        System.out.println(nombre + " vuela batiendo las alas");
    }

    @Override
    public void nadar() {
        System.out.println(nombre + " nada en el estanque");
    }
}

public class Avion implements Volador {
    private String modelo;

    public Avion(String modelo) {
        this.modelo = modelo;
    }

    @Override
    public void volar() {
        System.out.println("El avión " + modelo + " vuela con turbinas");
    }
}

public class DemoInterfaces {
    public static void main(String[] args) {
        Volador[] voladores = new Volador[3];
        voladores[0] = new Pato("Donald");
        voladores[1] = new Avion("Boeing 747");
        voladores[2] = new Pato("Lucas");

        for (Volador v : voladores) {
            v.volar();
            v.despegar();   // método default
            v.aterrizar();  // método default
        }
    }
}
```

### 3.9.3 Principios de modelado: IS-A, HAS-A y CAN-DO

El diseño orientado a objetos gira en torno a tres tipos de relaciones fundamentales:

```
IS-A (herencia):         HAS-A (composición):        CAN-DO (interfaz):
Un Perro ES-UN Animal    Un Coche TIENE-UN Motor    Un Pato PUEDE-VOLAR
     ▲                            ◆                          ○
     │                            │                          │
  Animal                       Motor                      Volador
     ▲                            ▲                          ▲
     │                            │                          │
   Perro                       Coche                       Pato
```

#### IS-A (herencia) — Relación de identidad

Usa `extends` cuando el subtipo es una especialización del supertipo: puede ser tratado como el supertipo en cualquier contexto (Principio de Sustitución de Liskov).

```java
// Correcto: un Círculo ES-UNA Figura
public class Circulo extends Figura { }

// Incorrecto: una Pila NO ES-UN Vector
// public class Pila extends Vector { }  // no cumple Liskov
```

#### HAS-A (composición) — Relación de posesión

Usa un campo del tipo relacionado cuando tu clase contiene, usa o delega en otra.

```java
// Un Coche TIENE-UN Motor
public class Coche {
    private Motor motor;                      // composición
    private List<Rueda> ruedas;               // composición 1-a-muchos
}

// Un Pedido TIENE LineasDePedido
public class Pedido {
    private List<LineaPedido> lineas;         // agregación
}
```

#### CAN-DO (interfaz) — Relación de capacidad

Usa `implements` cuando tu clase adquiere una capacidad transversal que puede compartir con clases de jerarquías completamente distintas.

```java
// Tanto un Pato como un Avión PUEDEN-VOLAR, pero no comparten superclase
public class Pato extends Animal implements Volador { }
public class Avion extends Vehiculo implements Volador { }

// Comparable es el CAN-DO por excelencia en la JDK
public class Persona implements Comparable<Persona> { }
public class Archivo implements Comparable<Archivo> { }
// Persona y Archivo no comparten nada más que la capacidad de compararse
```

#### Cuándo usar clase abstracta vs interfaz

```
¿Necesitas compartir ESTADO (campos) entre las subclases?
  ├── SÍ → Clase abstracta (puede tener campos de instancia)
  └── NO  → Interfaz (solo constantes static final)

¿Necesitas compartir COMPORTAMIENTO (código) entre las subclases?
  ├── SÍ, y está ligado al estado compartido → Clase abstracta
  ├── SÍ, pero es transversal y no depende de estado → Interfaz con default methods
  └── NO → Interfaz con métodos abstractos

¿Varias clases de distintas jerarquías necesitan el mismo contrato?
  └── SÍ → Interfaz (única opción, Java no tiene herencia múltiple de clases)
```

```java
// Clase abstracta: comparte estado (nombre, id) y comportamiento (obtenerInformacion)
public abstract class Empleado {
    protected String nombre;
    protected String id;
    protected double salarioBase;

    public Empleado(String nombre, String id, double salarioBase) {
        this.nombre = nombre;
        this.id = id;
        this.salarioBase = salarioBase;
    }

    public abstract double calcularSalario();

    public String obtenerInformacion() {
        return "Empleado: " + nombre + " (ID: " + id + ")";
    }
}

// Interfaz: comparte contrato (capacidad de ser auditado)
// Puede ser implementada por cualquier clase, no solo Empleados
public interface Auditable {
    String getResponsable();
    LocalDateTime getFechaModificacion();
    String getDescripcionCambio();

    default String generarLogAuditoria() {
        return String.format("[%s] %s modificado por %s: %s",
            getFechaModificacion(),
            this.getClass().getSimpleName(),
            getResponsable(),
            getDescripcionCambio());
    }
}

public class EmpleadoTiempoCompleto extends Empleado implements Auditable {
    private double bono;
    private String responsableModificacion;
    private LocalDateTime fechaModificacion;
    private String descripcionCambio;

    public EmpleadoTiempoCompleto(String nombre, String id,
                                   double salarioBase, double bono) {
        super(nombre, id, salarioBase);
        this.bono = bono;
    }

    @Override
    public double calcularSalario() { return salarioBase + bono; }

    @Override
    public String getResponsable() { return responsableModificacion; }

    @Override
    public LocalDateTime getFechaModificacion() { return fechaModificacion; }

    @Override
    public String getDescripcionCambio() { return descripcionCambio; }

    public void actualizarBono(double nuevoBono, String responsable) {
        this.responsableModificacion = responsable;
        this.fechaModificacion = LocalDateTime.now();
        this.descripcionCambio = "Bono actualizado de " + bono + " a " + nuevoBono;
        this.bono = nuevoBono;
    }
}
```

### 3.9.4 Default methods: utilidad y peligros

Los default methods de Java 8 resolvieron un problema real: añadir métodos a interfaces existentes sin romper todas las implementaciones (por ejemplo, `Collection.stream()`).

**Cuándo son útiles:**

```java
public interface Cache<K, V> {
    V get(K key);
    void put(K key, V value);
    void invalidate(K key);
    int size();

    default V getOrCompute(K key, Function<K, V> computer) {
        V value = get(key);
        if (value == null) {
            value = computer.apply(key);
            put(key, value);
        }
        return value;
    }

    default boolean containsKey(K key) {
        return get(key) != null;
    }

    default boolean isEmpty() {
        return size() == 0;
    }
}
```

**Cuándo son peligrosos:**

1. **Falsa sensación de reutilización**: un default method no puede acceder a estado. Si tu "lógica compartida" necesita estado, usa una clase abstracta.

2. **Violación del contrato de la interfaz**: si una interfaz define 10 default methods que dependen de un solo método abstracto, estás creando una _abstract class_ disfrazada. El _template method pattern_ con clase abstracta es más apropiado.

3. **Evolución incontrolada**: cada default method que añades obliga a todas las implementaciones existentes a heredar ese comportamiento. Si el default es ineficiente para una implementación concreta, esa implementación debe sobrescribirlo — pero si su autor no sabe que existe, heredará comportamiento subóptimo.

```java
// MAL: abusar de default methods con lógica no trivial
public interface Repositorio<T> {
    void guardar(T entidad);
    Optional<T> buscarPorId(String id);

    // Excesivo: 15 default methods que implementan queries complejas
    default List<T> buscarTodos() {
        throw new UnsupportedOperationException("Implementar en subclase");
    }
    default List<T> buscarPorFiltro(Predicate<T> filtro) { /* ... */ }
    default Page<T> buscarPaginado(int pagina, int tamaño) { /* ... */ }
    // ... 12 más
}
// Esto debería ser una clase abstracta, no una interfaz.
```

### 3.9.5 El problema del diamante con default methods

Cuando una clase implementa dos interfaces que definen el mismo default method, hay un conflicto:

```java
interface Logger {
    default void log(String msg) {
        System.out.println("[LOG] " + msg);
    }
}

interface Auditor {
    default void log(String msg) {
        System.out.println("[AUDIT] " + msg);
    }
}

// Error de compilación: MiServicio hereda dos implementaciones de log()
// public class MiServicio implements Logger, Auditor { }
```

**Tres formas de resolverlo:**

```java
// Opción 1: Elegir una implementación
public class MiServicioV1 implements Logger, Auditor {
    @Override
    public void log(String msg) {
        Logger.super.log(msg);  // usa la versión de Logger
    }
}

// Opción 2: Implementación propia que combina ambas
public class MiServicioV2 implements Logger, Auditor {
    @Override
    public void log(String msg) {
        Logger.super.log(msg);
        Auditor.super.log(msg);
    }
}

// Opción 3: Implementación completamente nueva
public class MiServicioV3 implements Logger, Auditor {
    @Override
    public void log(String msg) {
        System.out.println("[SERVICIO] " + msg);
    }
}
```

**Regla de resolución de la JVM:**
1. Las implementaciones de clases siempre ganan sobre los default methods de interfaces.
2. Si dos interfaces colisionan y la clase no resuelve el conflicto: error de compilación.
3. Si una interfaz A extiende B, los default methods de A ganan sobre los de B.

### 3.9.6 Interfaces funcionales

Una **interfaz funcional** es aquella que tiene **exactamente un método abstracto**. Se marca con `@FunctionalInterface`.

```java
@FunctionalInterface
public interface Operacion {
    int ejecutar(int a, int b);

    default String describir() {
        return "Operación binaria sobre enteros";
    }
}

public class DemoFuncional {
    public static void main(String[] args) {
        Operacion suma = (a, b) -> a + b;
        Operacion resta = (a, b) -> a - b;
        Operacion multiplicacion = (a, b) -> a * b;

        System.out.println(suma.ejecutar(5, 3));           // 8
        System.out.println(resta.ejecutar(5, 3));          // 2
        System.out.println(multiplicacion.ejecutar(5, 3)); // 15
    }
}
```

Interfaces funcionales clave en `java.util.function`:

| Interfaz          | Método abstracto     | Descripción                              |
|-------------------|----------------------|------------------------------------------|
| `Predicate<T>`    | `boolean test(T t)`  | Evalúa una condición sobre `T`           |
| `Consumer<T>`     | `void accept(T t)`   | Opera sobre `T` sin devolver resultado   |
| `Function<T, R>`  | `R apply(T t)`       | Transforma `T` en `R`                    |
| `Supplier<T>`     | `T get()`            | Proporciona un valor de tipo `T`         |
| `UnaryOperator<T>`| `T apply(T t)`       | Operación unaria sobre `T`               |
| `BinaryOperator<T>`| `T apply(T a, T b)` | Operación binaria sobre `T`              |

### 3.9.7 Comparativa: Clase abstracta vs. Interfaz

| Característica                       | Clase abstracta                             | Interfaz                                       |
|--------------------------------------|---------------------------------------------|------------------------------------------------|
| Palabra clave                        | `abstract class`                            | `interface`                                    |
| Instanciación directa                | No                                          | No                                             |
| Constructores                        | Sí                                          | No                                             |
| Atributos de instancia               | Sí (cualquier modificador)                  | Solo `public static final` (constantes)        |
| Métodos concretos                    | Sí                                          | Sí: `default` (Java 8+), `static` (Java 8+)   |
| Métodos privados                     | Sí                                          | Sí (Java 9+)                                   |
| Herencia múltiple                    | No (solo una clase padre)                   | Sí (una clase puede implementar varias)        |
| Relación                             | _"es-un"_ (especialización de una entidad)   | _"puede-hacer"_ (capacidad o contrato)         |
| Modificadores de métodos             | Cualquiera                                  | Implícitamente `public` (abstract/default)     |
| `final` en métodos                   | Sí                                          | No (no tiene sentido en interfaces)            |
| Versatilidad evolutiva               | Añadir método concreto no rompe subclases   | Añadir método abstracto rompe implementaciones |

```java
// Ejemplo combinado: clase abstracta + múltiples interfaces
public abstract class DispositivoElectronico {
    protected String marca;
    protected boolean encendido;

    public DispositivoElectronico(String marca) {
        this.marca = marca;
        this.encendido = false;
    }

    public abstract void encender();
    public abstract void apagar();
}

public interface Conectable {
    void conectar();
    void desconectar();

    default boolean estaConectado() {
        return false;
    }
}

public interface Actualizable {
    void actualizarFirmware(String version);

    default String versionActual() {
        return "1.0.0";
    }
}

public class Smartphone extends DispositivoElectronico
                        implements Conectable, Actualizable {

    private String numeroTelefono;
    private boolean conectado;

    public Smartphone(String marca, String numeroTelefono) {
        super(marca);
        this.numeroTelefono = numeroTelefono;
    }

    @Override
    public void encender() {
        encendido = true;
        System.out.println(marca + " encendido. Bienvenido.");
    }

    @Override
    public void apagar() {
        encendido = false;
        System.out.println(marca + " apagado. Adiós.");
    }

    @Override
    public void conectar() {
        conectado = true;
        System.out.println("Conectado a la red");
    }

    @Override
    public void desconectar() {
        conectado = false;
        System.out.println("Desconectado de la red");
    }

    @Override
    public boolean estaConectado() {
        return conectado;
    }

    @Override
    public void actualizarFirmware(String version) {
        System.out.println("Firmware actualizado a v" + version);
    }
}
```

---

## 3.10 Nested Classes (Clases Anidadas)

Java permite declarar clases dentro de otras clases. Esto permite agrupar clases que solo se usan en un contexto concreto, aumentando la encapsulación y la legibilidad.

### 3.10.1 Tipos de nested classes

| Tipo                  | Declaración                                   | Acceso a miembros externos          |
|-----------------------|-----------------------------------------------|-------------------------------------|
| Static nested class   | `static class Interna`                        | Solo miembros estáticos             |
| Inner class           | `class Interna` (sin static)                  | Todos los miembros (incluye `private`) |
| Local class           | Declarada dentro de un método                 | Variables del método (solo `final` o effectively final) |
| Anonymous class       | Declarada sin nombre como expresión           | Variables del método (solo `final` o effectively final) |

### 3.10.2 Static nested classes

Son clases declaradas `static` dentro de otra clase. No tienen acceso al `this` de la clase externa (no tienen instancia asociada). Son el tipo más útil y recomendado.

```java
public class Calculadora {
    private static String version = "2.0";

    // Static nested class
    public static class Operacion {
        public static int sumar(int a, int b) {
            return a + b;
        }

        public static String getVersion() {
            return version;  // OK: variable estática de la clase externa
        }
    }
}

// Uso:
int resultado = Calculadora.Operacion.sumar(5, 3);
```

### 3.10.3 Inner classes (clases internas)

No son `static`, por lo que cada instancia de la inner class está ligada a una instancia de la outer class. Tienen acceso a todos los miembros (incluso `private`) de la instancia externa.

```java
public class Agenda {
    private List<Contacto> contactos = new ArrayList<>();

    private class Contacto {
        private String nombre;
        private String telefono;

        Contacto(String nombre, String telefono) {
            this.nombre = nombre;
            this.telefono = telefono;
        }

        void mostrarEnFormato() {
            // Acceso al miembro privado de la clase externa
            System.out.println(contactos.size() + " contactos. "
                + nombre + ": " + telefono);
        }
    }

    public void añadirContacto(String nombre, String telefono) {
        contactos.add(new Contacto(nombre, telefono));
    }
}
```

**Cuidado**: las inner classes retienen una referencia implícita a la instancia externa. Esto puede causar memory leaks si la instancia interna sobrevive a la externa. Si no necesitas acceso a la instancia externa, declara la clase como `static`.

### 3.10.4 Local classes

Se definen dentro de un bloque de código (método, bloque `if`, bucle). Solo son visibles dentro de ese bloque.

```java
public void validarEntrada(String entrada) {
    class ResultadoValidacion {
        boolean valido;
        String mensaje;

        ResultadoValidacion(boolean valido, String mensaje) {
            this.valido = valido;
            this.mensaje = mensaje;
        }
    }

    ResultadoValidacion resultado = new ResultadoValidacion(
        entrada != null && entrada.length() > 3,
        entrada == null ? "Entrada nula" : "Longitud: " + entrada.length()
    );

    if (!resultado.valido) {
        System.out.println(resultado.mensaje);
    }
}
```

### 3.10.5 Anonymous classes vs Lambdas

Las anonymous classes permiten declarar e instanciar una clase al mismo tiempo, sin darle nombre. Son útiles para implementaciones rápidas de interfaces o extensiones puntuales. Pero desde Java 8, las lambdas las han reemplazado en casi todos los casos de interfaces funcionales.

```java
// Anonymous class (antes de Java 8, y aún útil para interfaces con >1 método)
Runnable tareaAnonima = new Runnable() {
    @Override
    public void run() {
        System.out.println("Ejecutando desde anonymous class");
    }
};

// Lambda equivalente (solo si la interfaz es funcional)
Runnable tareaLambda = () -> System.out.println("Ejecutando desde lambda");
```

#### ¿Cuándo usar anonymous class en lugar de lambda?

```java
// 1. Cuando necesitas inicializar campos de instancia adicionales
Comparator<String> comp = new Comparator<>() {
    private int contador = 0;  // estado adicional

    @Override
    public int compare(String a, String b) {
        contador++;  // lambda no puede tener campos
        return a.length() - b.length();
    }
};

// 2. Cuando la interfaz tiene múltiples métodos abstractos
// (no es funcional — lambda no funciona aquí)
MouseListener listener = new MouseAdapter() {
    @Override
    public void mouseClicked(MouseEvent e) {
        System.out.println("Click en " + e.getPoint());
    }

    @Override
    public void mouseEntered(MouseEvent e) {
        System.out.println("Ratón entró");
    }
};

// 3. Cuando necesitas sobrescribir métodos de Object
// (las lambdas no pueden cambiar equals/hashCode/toString)
Object valor = new Object() {
    @Override
    public String toString() {
        return "valor personalizado";
    }
};

// 4. Cuando creas una subclase de una clase abstracta o concreta
// (las lambdas solo sirven para interfaces)
Thread hilo = new Thread("mi-hilo") {
    @Override
    public void run() {
        System.out.println("Ejecutando en: " + getName());
    }
};
hilo.start();
```

#### Diferencias semánticas importantes entre lambda y anonymous class

| Aspecto               | Lambda                                | Anonymous class                     |
|-----------------------|---------------------------------------|-------------------------------------|
| `this`                | Referencia al objeto externo          | Referencia a la propia anonymous class |
| `toString()`/`equals()`| No se pueden sobrescribir            | Sí se pueden sobrescribir           |
| Campos de instancia   | No permitidos                         | Permitidos                          |
| `invokedynamic`       | Sí (generación en runtime)            | No (compila a archivo .class)       |
| Interfaz              | Solo funcional (1 método abstracto)   | Cualquier interfaz o clase          |

### 3.10.6 Cuándo usar cada tipo de nested class

| Escenario                                                       | Tipo recomendado        |
|-----------------------------------------------------------------|-------------------------|
| Agrupar clases auxiliares sin dependencia de la clase externa   | Static nested class     |
| Acceso a miembros privados de la instancia externa              | Inner class             |
| Clase usada en un solo método, lógica pequeña                   | Local class             |
| Implementación puntual de interfaz no funcional                 | Anonymous class         |
| Implementación de interfaz funcional                            | Lambda                  |
---

## 3.11 La Clase `Object`

### 3.11.1 La raíz de la jerarquía

En Java, todas las clases heredan directa o indirectamente de `java.lang.Object`. Esto significa que cualquier objeto tiene a su disposición los métodos definidos en `Object`.

```
                         Object
                           ▲
              ┌────────────┼────────────┐
              │            │            │
           String      Throwable     Animal (tu clase)
              ▲            ▲            ▲
              │            │            │
      (no se extiende)  Exception     Perro
```

### 3.11.2 Métodos principales de `Object`

```java
public class Object {
    public String toString() { ... }            // Representación en texto
    public boolean equals(Object obj) { ... }    // Comparación de igualdad
    public int hashCode() { ... }               // Código hash del objeto
    public final Class<?> getClass() { ... }    // Metadatos de la clase
    protected Object clone() { ... }            // Clonación (poco recomendada)
    protected void finalize() { ... }           // Obsoleto desde Java 9
    public final void wait() { ... }            // Concurrencia
    public final void notify() { ... }
    public final void notifyAll() { ... }
}
```

### 3.11.3 `toString()`

Devuelve una representación en cadena del objeto. La implementación por defecto en `Object` devuelve algo como `NombreClase@hashcode_hex`. Casi siempre conviene sobrescribirlo.

```java
public class Producto {
    private int id;
    private String nombre;
    private double precio;

    public Producto(int id, String nombre, double precio) {
        this.id = id;
        this.nombre = nombre;
        this.precio = precio;
    }

    @Override
    public String toString() {
        return "Producto{id=" + id
            + ", nombre='" + nombre + '\''
            + ", precio=" + precio + '}';
    }
}

// Uso:
Producto p = new Producto(1, "Teclado", 49.99);
System.out.println(p);  // Producto{id=1, nombre='Teclado', precio=49.99}
```

### 3.11.4 `equals(Object o)` y `hashCode()`

#### El contrato equals/hashCode — explicado en detalle

El contrato entre `equals()` y `hashCode()` está definido en la documentación de `Object` y es vinculante para todo programa Java:

```
┌─────────────────────────────────────────────────────────────────┐
│                     CONTRATO EQUALS/HASHCODE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. REFLEXIVIDAD:  x.equals(x) siempre es true                  │
│  2. SIMETRÍA:      x.equals(y) ⇔ y.equals(x)                   │
│  3. TRANSITIVIDAD: x.equals(y) ∧ y.equals(z) ⇒ x.equals(z)     │
│  4. CONSISTENCIA:  x.equals(y) devuelve lo mismo mientras       │
│                    los objetos no cambien                        │
│  5. NULL:          x.equals(null) siempre es false              │
│                                                                  │
│  REGLA DE ORO:                                                  │
│  Si a.equals(b) == true  ➜  a.hashCode() == b.hashCode()      │
│  Si a.equals(b) == false ➜  hashCode puede o no ser igual      │
│  (pero si haces que siempre sean distintos, mejor rendimiento)  │
│                                                                  │
│  COROLARIO:                                                     │
│  Si sobrescribes equals(), DEBES sobrescribir hashCode().       │
│  Si sobrescribes hashCode(), probablemente debas sobrescribir   │
│  equals() también.                                              │
└─────────────────────────────────────────────────────────────────┘
```

**¿Por qué funciona así?** Las colecciones basadas en hash (`HashMap`, `HashSet`, `Hashtable`) usan `hashCode()` para encontrar el _bucket_ (compartimento) donde está o debería estar un objeto, y solo después usan `equals()` para comparar dentro del bucket.

```
Operación: conjunto.contains(b)

Paso 1: Calcular hashCode de b
        ┌────────────────────┐
        │ Bucket 0: [obj1]   │
        │ Bucket 1: []       │
        │ Bucket 2: [obj2]   │◄─── b.hashCode() % 8 = 2 → buscar en bucket 2
        │ Bucket 3: [obj3]   │
        │ ...                │
        └────────────────────┘

Paso 2: Dentro del bucket 2, para cada objeto o:
        Si b.equals(o) → encontrado (true)
        Si ningún objeto coincide → no encontrado (false)

Si hashCode() no está sobrescrito:
  - b tiene hashCode distinto al de a (basado en referencia)
  - Paso 1: b va a un bucket diferente al de a
  - Paso 2: no encuentra nada → contains() devuelve false
  - ¡Incluso si a.equals(b) es true!
```

#### Bug real en HashSet cuando equals/hashCode están rotos

```java
// Clase con equals sobrescrito pero hashCode NO
class PersonaRota {
    private String dni;
    private String nombre;

    PersonaRota(String dni, String nombre) {
        this.dni = dni;
        this.nombre = nombre;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof PersonaRota otra)) return false;
        return Objects.equals(dni, otra.dni);
    }
    // hashCode() NO sobrescrito — usa Object.hashCode() (basado en referencia)
}

// Demostración del bug:
Set<PersonaRota> conjunto = new HashSet<>();
PersonaRota a = new PersonaRota("123", "Ana");
PersonaRota b = new PersonaRota("123", "Ana");

conjunto.add(a);

System.out.println("a.equals(b) = " + a.equals(b));          // true
System.out.println("conjunto.contains(b) = " + conjunto.contains(b)); // false ← BUG
System.out.println("conjunto.size() = " + conjunto.size());  // 1

// Y ahora podemos añadir b, creando un duplicado lógico:
conjunto.add(b);
System.out.println("conjunto.size() = " + conjunto.size());  // 2 ← ¡duplicado!

// Esto es un bug real que ocurre en producción:
// clientes duplicados en un Set, registros que no se encuentran,
// comportamientos inconsistentes y no determinísticos.
```

#### Cómo escribir equals/hashCode correctamente

**Usando `java.util.Objects` (Java 7+):**

```java
public class PersonaCorrecta {
    private final String dni;
    private final String nombre;
    private final LocalDate fechaNacimiento;

    public PersonaCorrecta(String dni, String nombre, LocalDate fechaNacimiento) {
        this.dni = dni;
        this.nombre = nombre;
        this.fechaNacimiento = fechaNacimiento;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof PersonaCorrecta otra)) return false;
        return Objects.equals(dni, otra.dni)
            && Objects.equals(nombre, otra.nombre)
            && Objects.equals(fechaNacimiento, otra.fechaNacimiento);
    }

    @Override
    public int hashCode() {
        return Objects.hash(dni, nombre, fechaNacimiento);
    }
}
```

`Objects.equals(a, b)` maneja correctamente los nulls (si ambos son null, retorna true; si solo uno, false). `Objects.hash(...)` genera un hash combinado de forma correcta y consistente.

#### `instanceof` vs `getClass()` en equals

Hay un debate sobre qué usar en equals para comprobar el tipo:

```java
// Estilo 1: instanceof (permite que subclases sean equals)
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Punto punto)) return false;
    return x == punto.x && y == punto.y;
}
// Consecuencia: un Punto3D podría ser equals a un Punto
// si el instanceof y los campos coinciden.

// Estilo 2: getClass() (solo objetos de exactamente la misma clase)
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Punto punto = (Punto) o;
    return x == punto.x && y == punto.y;
}
// Consecuencia: un Punto3D nunca será equals a un Punto.
// Rompe el Principio de Sustitución de Liskov si la subclase
// no añade campos que afecten equals.
```

**Recomendación (Josh Bloch, Effective Java):**
- Usa `instanceof` si la clase es `final` o si equals está diseñado para funcionar entre superclase y subclases.
- Usa `getClass()` si quieres garantizar que objetos de distintas clases nunca sean iguales.
- Si la clase no es `final` y usas `instanceof`, asegúrate de que las subclases no añaden estado que rompa la simetría y transitividad.

#### Lombok `@EqualsAndHashCode`

Lombok genera equals/hashCode automáticamente:

```java
import lombok.EqualsAndHashCode;
import lombok.Getter;

@Getter
@EqualsAndHashCode
public class ProductoLombok {
    private String sku;
    private String nombre;
    private double precio;

    // @EqualsAndHashCode.Exclude  // para excluir un campo
    // private String descripcionInterna;
}
```

**Opciones útiles:**
- `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` — solo incluye campos marcados con `@EqualsAndHashCode.Include`.
- `@EqualsAndHashCode(callSuper = true)` — incluye el resultado de `super.equals()` y `super.hashCode()` en la comparación (necesario si tu clase extiende otra).
- `@EqualsAndHashCode.Exclude` — excluye un campo específico.

#### Records (Java 14+): equals/hashCode automáticos

Los records generan equals/hashCode basados en todos los campos del encabezado:

```java
public record PuntoRecord(int x, int y) { }

PuntoRecord p1 = new PuntoRecord(3, 5);
PuntoRecord p2 = new PuntoRecord(3, 5);

System.out.println(p1.equals(p2));   // true  — generado automáticamente
System.out.println(p1.hashCode());   // consistente, basado en x e y
System.out.println(p2.hashCode());   // mismo valor que p1.hashCode()
```

La implementación generada es equivalente a la manual con `Objects.equals` y `Objects.hash` para todos los componentes del record. Si necesitas una igualdad diferente (basada en un subconjunto de campos), deberías usar una clase normal, no un record.

### 3.11.5 `getClass()`

Devuelve el objeto `Class` que contiene metadatos sobre la clase del objeto. Es `final`, por lo que no se puede sobrescribir.

```java
Object obj = "Hola mundo";
System.out.println(obj.getClass());              // class java.lang.String
System.out.println(obj.getClass().getName());    // java.lang.String
System.out.println(obj.getClass().getSimpleName()); // String

// Comparar clases (útil en equals)
if (this.getClass() == otro.getClass()) { ... }
```

---

## 3.12 Inmutabilidad

### 3.12.1 Qué es la inmutabilidad y por qué es deseable

Un objeto es **inmutable** si su estado no puede modificarse después de su construcción. La inmutabilidad es una de las propiedades más deseables en el diseño OO porque elimina clases enteras de bugs:

- **Thread-safe automáticamente**: los objetos inmutables pueden compartirse entre hilos sin sincronización.
- **Caching seguro**: puedes almacenar y reutilizar instancias sin miedo a que cambien.
- **hashCode constante**: puedes calcular el hash una vez y cachearlo (como hace `String`).
- **Sin efectos secundarios**: pasar un objeto inmutable como parámetro es seguro; el método llamado no puede alterarlo.
- **Fallan pronto**: si el estado es inválido, el constructor lanza excepción en el momento de creación, no horas después.

### 3.12.2 Cómo diseñar una clase inmutable (5 reglas)

```java
// Ejemplo: una clase Dinero inmutable
public final class Dinero implements Comparable<Dinero> {
    private final BigDecimal cantidad;
    private final String moneda;

    // 1. Constructor que inicializa TODOS los campos
    public Dinero(BigDecimal cantidad, String moneda) {
        // 2. Validación en construcción (fail-fast)
        Objects.requireNonNull(cantidad, "cantidad no puede ser null");
        Objects.requireNonNull(moneda, "moneda no puede ser null");
        if (cantidad.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("cantidad no puede ser negativa");
        }

        // 3. Copia defensiva de parámetros mutables
        this.cantidad = cantidad;  // BigDecimal ya es inmutable
        this.moneda = moneda;      // String ya es inmutable
    }

    // 4. Sin setters — solo getters
    public BigDecimal getCantidad() { return cantidad; }
    public String getMoneda() { return moneda; }

    // 5. Métodos que "modifican" en realidad devuelven NUEVAS instancias
    public Dinero sumar(Dinero otro) {
        if (!this.moneda.equals(otro.moneda)) {
            throw new IllegalArgumentException("No se pueden sumar distintas monedas");
        }
        return new Dinero(this.cantidad.add(otro.cantidad), this.moneda);
    }

    public Dinero multiplicar(double factor) {
        return new Dinero(
            this.cantidad.multiply(BigDecimal.valueOf(factor)),
            this.moneda);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Dinero d)) return false;
        return cantidad.compareTo(d.cantidad) == 0
            && moneda.equals(d.moneda);
    }

    @Override
    public int hashCode() {
        return Objects.hash(cantidad, moneda);
    }

    @Override
    public String toString() {
        return cantidad + " " + moneda;
    }

    @Override
    public int compareTo(Dinero otro) {
        if (!this.moneda.equals(otro.moneda)) {
            throw new IllegalArgumentException("No se pueden comparar distintas monedas");
        }
        return this.cantidad.compareTo(otro.cantidad);
    }
}
```

#### Copias defensivas: el detalle crucial

Si tu clase inmutable contiene referencias a objetos mutables, debes hacer copias defensivas tanto al entrar (constructor) como al salir (getters):

```java
public final class InformeInmutable {
    private final String titulo;
    private final List<String> secciones;    // List es mutable
    private final Date fechaCreacion;        // Date es mutable y obsoleto

    public InformeInmutable(String titulo, List<String> secciones, Date fechaCreacion) {
        this.titulo = titulo;

        // Copia defensiva al entrar: si el llamador modifica su lista
        // después de construirnos, nosotros no nos vemos afectados
        this.secciones = List.copyOf(secciones);  // Java 10+: copia inmutable

        // Copia defensiva para Date (mutable)
        this.fechaCreacion = new Date(fechaCreacion.getTime());
    }

    public String getTitulo() { return titulo; }

    public List<String> getSecciones() {
        // Copia defensiva al salir (o devolver vista inmutable)
        return Collections.unmodifiableList(secciones);
    }

    public Date getFechaCreacion() {
        // Copia defensiva al salir
        return new Date(fechaCreacion.getTime());
    }
}
```

**Mejor aún: usar tipos inmutables en lugar de copias defensivas.**

```java
// En lugar de Date (mutable), usa:
// - Instant, LocalDate, LocalDateTime (java.time, inmutables desde Java 8)

// En lugar de List (mutable), declara internamente:
// - List.copyOf() (Java 10+, inmutable)
// - Collections.unmodifiableList() (vista de solo lectura)
```

### 3.12.3 Records (Java 14+): inmutabilidad por defecto

Los records son la forma más concisa de definir objetos inmutables portadores de datos:

```java
public record DineroRecord(BigDecimal cantidad, String moneda) {
    // Constructor compacto con validación
    public DineroRecord {
        Objects.requireNonNull(cantidad);
        Objects.requireNonNull(moneda);
        if (cantidad.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("No puede ser negativo");
        }
    }

    // Método que "modifica" — en realidad crea nueva instancia
    public DineroRecord sumar(DineroRecord otro) {
        if (!moneda.equals(otro.moneda)) {
            throw new IllegalArgumentException("Monedas distintas");
        }
        return new DineroRecord(cantidad.add(otro.cantidad), moneda);
    }
}
```

Los records proporcionan inmutabilidad superficial (_shallow immutability_). Si uno de los componentes es un objeto mutable (ej: `List<String>`), el record no impide que el contenido de esa lista sea modificado. Para inmutabilidad profunda, usa componentes inmutables o copias defensivas en el constructor compacto.

### 3.12.4 Clases de valor del mundo real

Las clases de valor (_value objects_) son objetos pequeños e inmutables definidos por sus atributos, no por su identidad. Son ubicuas en aplicaciones bien diseñadas:

```java
// Email: value object con validación en construcción
public final class Email {
    private final String valor;

    public Email(String valor) {
        Objects.requireNonNull(valor);
        if (!valor.matches("^[\\w.%+-]+@[\\w.-]+\\.[a-zA-Z]{2,}$")) {
            throw new IllegalArgumentException("Email inválido: " + valor);
        }
        this.valor = valor.toLowerCase();
    }

    public String getDominio() {
        return valor.substring(valor.indexOf('@') + 1);
    }

    public String getValor() { return valor; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Email e)) return false;
        return valor.equals(e.valor);
    }

    @Override
    public int hashCode() { return valor.hashCode(); }

    @Override
    public String toString() { return valor; }
}
```

```java
// Money: patrón de valor monetario
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount.setScale(currency.getDefaultFractionDigits(),
            RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public static Money usd(String amount) {
        return new Money(new BigDecimal(amount), Currency.getInstance("USD"));
    }

    public static Money eur(String amount) {
        return new Money(new BigDecimal(amount), Currency.getInstance("EUR"));
    }

    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }

    public Money subtract(Money other) {
        assertSameCurrency(other);
        return new Money(amount.subtract(other.amount), currency);
    }

    public Money multiply(double factor) {
        return new Money(amount.multiply(BigDecimal.valueOf(factor)), currency);
    }

    public Money[] allocate(int n) {
        BigDecimal[] parts = amount.divideAndRemainder(
            BigDecimal.valueOf(n), MathContext.DECIMAL128);
        Money[] result = new Money[n];
        // Algoritmo de asignación justa (centavos repartidos)
        for (int i = 0; i < n; i++) {
            result[i] = new Money(parts[0], currency);
        }
        for (int i = 0; i < parts[1].intValue(); i++) {
            result[i] = result[i].add(new Money(BigDecimal.valueOf(0.01), currency));
        }
        return result;
    }

    private void assertSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch: "
                + currency + " vs " + other.currency);
        }
    }

    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money m)) return false;
        return amount.compareTo(m.amount) == 0
            && currency.equals(m.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }

    @Override
    public String toString() {
        return currency.getSymbol() + " " + amount;
    }
}
```

```java
// Coordinate: value object para geolocalización
public final class Coordinate {
    private final double latitude;
    private final double longitude;

    public Coordinate(double latitude, double longitude) {
        if (latitude < -90 || latitude > 90) {
            throw new IllegalArgumentException("Latitud inválida: " + latitude);
        }
        if (longitude < -180 || longitude > 180) {
            throw new IllegalArgumentException("Longitud inválida: " + longitude);
        }
        this.latitude = latitude;
        this.longitude = longitude;
    }

    public double distanceTo(Coordinate other) {
        double dLat = Math.toRadians(other.latitude - this.latitude);
        double dLon = Math.toRadians(other.longitude - this.longitude);
        double a = Math.sin(dLat / 2) * Math.sin(dLat / 2)
            + Math.cos(Math.toRadians(this.latitude))
                * Math.cos(Math.toRadians(other.latitude))
                * Math.sin(dLon / 2) * Math.sin(dLon / 2);
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        return 6371 * c;  // Radio de la Tierra en km
    }

    public double getLatitude() { return latitude; }
    public double getLongitude() { return longitude; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Coordinate c)) return false;
        return Math.abs(c.latitude - latitude) < 0.000001
            && Math.abs(c.longitude - longitude) < 0.000001;
    }

    @Override
    public int hashCode() {
        return Objects.hash(Math.round(latitude * 1000000),
            Math.round(longitude * 1000000));
    }

    @Override
    public String toString() {
        return "(" + latitude + ", " + longitude + ")";
    }
}
```

---

## 3.13 Enumeraciones (`enum`) y Registros (`record`)

### 3.13.1 Enumeraciones

Un `enum` en Java es un tipo especial de clase que representa un conjunto fijo de constantes. Aunque parece una simple lista de valores, es mucho más potente: puede tener constructores, métodos, atributos e implementar interfaces.

#### Declaración básica

```java
public enum DiaSemana {
    LUNES, MARTES, MIERCOLES, JUEVES, VIERNES, SABADO, DOMINGO
}

// Uso:
DiaSemana hoy = DiaSemana.MARTES;
System.out.println(hoy);  // MARTES
```

#### Enums con atributos, constructores y métodos

```java
public enum EstadoPedido {
    PENDIENTE("El pedido está pendiente de procesar"),
    EN_PROCESO("El pedido se está preparando"),
    ENVIADO("El pedido está en camino"),
    ENTREGADO("El pedido ha sido entregado"),
    CANCELADO("El pedido ha sido cancelado");

    private final String descripcion;

    EstadoPedido(String descripcion) {
        this.descripcion = descripcion;
    }

    public String getDescripcion() {
        return descripcion;
    }

    public boolean esEstadoFinal() {
        return this == ENTREGADO || this == CANCELADO;
    }
}

// Uso:
EstadoPedido estado = EstadoPedido.ENVIADO;
System.out.println(estado.getDescripcion());  // El pedido está en camino
System.out.println(estado.esEstadoFinal());   // false
```

#### Enums con comportamiento polimórfico por constante

```java
public enum OperacionAritmetica {
    SUMA {
        @Override
        public double aplicar(double a, double b) { return a + b; }
    },
    RESTA {
        @Override
        public double aplicar(double a, double b) { return a - b; }
    },
    MULTIPLICACION {
        @Override
        public double aplicar(double a, double b) { return a * b; }
    },
    DIVISION {
        @Override
        public double aplicar(double a, double b) {
            if (b == 0) throw new ArithmeticException("División por cero");
            return a / b;
        }
    };

    public abstract double aplicar(double a, double b);
}

// Uso:
double resultado = OperacionAritmetica.SUMA.aplicar(10, 5);  // 15.0
```

#### Métodos útiles de `Enum`

```java
public enum Color {
    ROJO, VERDE, AZUL
}

// values(): array con todas las constantes en orden de declaración
for (Color c : Color.values()) {
    System.out.println(c + " (ordinal: " + c.ordinal() + ")");
}
// ROJO (ordinal: 0), VERDE (ordinal: 1), AZUL (ordinal: 2)

// valueOf(String): convierte un String en la constante enum
Color elegido = Color.valueOf("VERDE");  // Color.VERDE
// Color.valueOf("AMARILLO");           // IllegalArgumentException
```

| Método            | Descripción                                                |
|-------------------|------------------------------------------------------------|
| `values()`        | Devuelve un array con todas las constantes del enum        |
| `valueOf(String)` | Convierte el nombre de una constante en la instancia enum  |
| `ordinal()`       | Devuelve la posición (base 0) de la constante              |
| `name()`          | Devuelve el nombre exacto de la constante como String      |

### 3.13.2 Registros (_Records_, Java 14+)

Un `record` es un tipo especial de clase pensado para ser un **portador inmutable de datos**. Declara de forma concisa una clase cuyos atributos son `private final`, y automáticamente genera constructor canónico, `equals()`, `hashCode()` y `toString()`.

#### Declaración básica

```java
public record Punto(int x, int y) { }

// Uso:
Punto p1 = new Punto(3, 5);
Punto p2 = new Punto(3, 5);

System.out.println(p1.x());       // 3 (método de acceso, no getX())
System.out.println(p1.y());       // 5
System.out.println(p1);           // Punto[x=3, y=5]
System.out.println(p1.equals(p2)); // true
System.out.println(p1.hashCode() == p2.hashCode()); // true
```

#### Lo que el compilador genera automáticamente

```java
// Lo que escribirías manualmente...
public class PuntoManual {
    private final int x;
    private final int y;

    public PuntoManual(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int x() { return x; }
    public int y() { return y; }

    @Override
    public boolean equals(Object o) { /* ... */ }

    @Override
    public int hashCode() { /* ... */ }

    @Override
    public String toString() { /* ... */ }
}

// ...se reduce a:
public record Punto(int x, int y) { }
```

#### Constructor compacto

Permite añadir validaciones y lógica de inicialización sin repetir la lista de parámetros.

```java
public record PersonaRecord(String nombre, int edad) {

    // Constructor compacto: validación sin lista de parámetros
    public PersonaRecord {
        if (nombre == null || nombre.isBlank()) {
            throw new IllegalArgumentException("El nombre no puede estar vacío");
        }
        if (edad < 0 || edad > 150) {
            throw new IllegalArgumentException("Edad inválida: " + edad);
        }
        // La asignación a this.nombre y this.edad ocurre automáticamente al final
    }
}

// Uso:
PersonaRecord pr = new PersonaRecord("Ana", 28);   // OK
// PersonaRecord pr2 = new PersonaRecord("", -1);  // IllegalArgumentException
```

#### Records con métodos adicionales

Los records pueden tener métodos propios y static, e implementar interfaces.

```java
public record Rango(int minimo, int maximo) {

    // Constructor canónico personalizado
    public Rango {
        if (minimo > maximo) {
            throw new IllegalArgumentException(
                "mínimo (" + minimo + ") > máximo (" + maximo + ")");
        }
    }

    // Método derivado
    public int amplitud() {
        return maximo - minimo;
    }

    public boolean contiene(int valor) {
        return valor >= minimo && valor <= maximo;
    }

    // Método estático
    public static Rango de0a100() {
        return new Rango(0, 100);
    }
}
```

#### Records con interfaces

```java
public interface Identificable {
    String id();
    String etiqueta();
}

public record ProductoRecord(String sku, String nombre, double precio)
    implements Identificable {

    @Override
    public String id() {
        return sku;  // el método de acceso coincide con id() de la interfaz
    }

    @Override
    public String etiqueta() {
        return nombre + " ($" + precio + ")";
    }
}
```

### 3.13.3 ¿Cuándo usar record vs. clase regular?

| Usar `record` cuando...                                    | Usar `class` regular cuando...                               |
|------------------------------------------------------------|-------------------------------------------------------------|
| Los datos son inmutables por diseño                        | Los atributos necesitan cambiar después de la creación       |
| La clase es un simple portador de datos (DTO, value object)| La clase tiene lógica de negocio compleja                    |
| La igualdad se basa en todos los campos                    | La igualdad se basa en un subconjunto de campos              |
| Se quiere evitar código repetitivo (boilerplate)           | Se necesita herencia de clases (los records no pueden extender clases) |
| La transparencia es deseable (toString automático)         | Se necesita control fino sobre encapsulamiento               |

**Restricciones de los records:**

- Son implícitamente `final` (no se puede heredar de un record).
- No pueden extender otra clase (ya extienden `java.lang.Record`).
- Todos los campos son `private final`.
- No se pueden declarar campos de instancia adicionales (solo los del encabezado).

```java
// Ejemplo de comparación: mismo concepto como clase y como record

// Clase tradicional (más de 40 líneas)
public class DireccionClase {
    private final String calle;
    private final String ciudad;
    private final String codigoPostal;

    public DireccionClase(String calle, String ciudad, String codigoPostal) {
        this.calle = calle;
        this.ciudad = ciudad;
        this.codigoPostal = codigoPostal;
    }

    public String getCalle() { return calle; }
    public String getCiudad() { return ciudad; }
    public String getCodigoPostal() { return codigoPostal; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof DireccionClase d)) return false;
        return calle.equals(d.calle)
            && ciudad.equals(d.ciudad)
            && codigoPostal.equals(d.codigoPostal);
    }

    @Override
    public int hashCode() {
        return java.util.Objects.hash(calle, ciudad, codigoPostal);
    }

    @Override
    public String toString() {
        return "DireccionClase{calle='" + calle
            + "', ciudad='" + ciudad
            + "', codigoPostal='" + codigoPostal + "'}";
    }
}

// Record equivalente (1 línea)
public record DireccionRecord(String calle, String ciudad, String codigoPostal) { }
```

---

## 3.14 Resumen del capítulo

| Sección                          | Conceptos clave                                                                 |
|----------------------------------|---------------------------------------------------------------------------------|
| Clases y Objetos                 | Clase = plantilla; Objeto = instancia concreta; `new`; operador punto; estado y comportamiento; modelado de dominio; heap y stack |
| Atributos                        | Variables de instancia (por objeto) vs. `static` (compartidas); `final`; `volatile`; `AtomicReference`; reordenamiento JVM |
| Métodos                          | Firma, `void`, paso por valor (primitivos, referencias y arrays); bytecodes de invocación (`invokevirtual`, `invokeinterface`, `invokestatic`, `invokespecial`, `invokedynamic`); sobrecarga; varargs |
| Constructores                    | Sin tipo de retorno; constructor por defecto; sobrecarga; `this()`; bloques de inicialización; Builder vs telescópico vs setters |
| Diseño de clases efectivo        | Cohesión alta, acoplamiento bajo; Tell Don't Ask; Law of Demeter; SRP; naming conventions |
| Encapsulamiento                  | `private`, default, `protected`, `public`; getters/setters; `this`; JavaBeans   |
| Herencia                         | `extends`; `super`; `@Override`; frágil base class; Herencia vs Composición (Stack/Vector, Properties/Hashtable); forwarding; sealed classes (Java 17); `final` |
| Polimorfismo                     | Estático (sobrecarga) vs. dinámico (sobrescritura); upcasting/downcasting; `instanceof`; pattern matching (Java 16+) |
| Clases abstractas e interfaces   | `abstract class`; `interface` (default, static, private); IS-A vs HAS-A vs CAN-DO; diamante con default methods; interfaces funcionales |
| Nested classes                   | Static nested; inner; local; anonymous class vs lambda; `invokedynamic` |
| Clase `Object`                   | `toString()`, `equals()`, `hashCode()`; contrato equals-hashCode (con diagrama); `Objects.equals/hash`; `instanceof` vs `getClass()`; Lombok `@EqualsAndHashCode`; records |
| Inmutabilidad                    | 5 reglas para clases inmutables; copias defensivas; records; value objects: Email, Money, Coordinate |
| `enum` y `record`                | Enum con atributos/métodos y comportamiento polimórfico; record como DTO inmutable; constructor compacto; record vs clase regular |

---

**Este capítulo cubre los fundamentos de la Programación Orientada a Objetos en Java, desde la sintaxis básica hasta los principios de diseño avanzados. Dominar estos conceptos es esencial para escribir código Java mantenible, extensible y profesional.**

---

← [Capítulo anterior](capitulo-02-fundamentos.md) | [Inicio](../README.md) | [Capítulo siguiente →](capitulo-04-colecciones.md)
