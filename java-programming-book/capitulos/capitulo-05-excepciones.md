# Capítulo 5: Excepciones en Java

---

## 5.1 ¿Qué es una excepción?

Una **excepción** es un evento que ocurre durante la ejecución de un programa y que interrumpe el flujo normal de las instrucciones. Cuando el sistema encuentra una situación que no sabe manejar, crea un objeto de tipo excepción y lo «lanza» (*throws*) hacia arriba en la pila de llamadas.

### 5.1.1 La pila de llamadas (Call Stack)

Cada vez que un método invoca a otro, Java guarda el punto de retorno en una estructura llamada **pila de llamadas**. Por ejemplo:

```java
public class Main {
    public static void main(String[] args) {
        metodoA();               // (1) main llama a metodoA
    }

    static void metodoA() {
        metodoB();               // (2) metodoA llama a metodoB
    }

    static void metodoB() {
        metodoC();               // (3) metodoB llama a metodoC
    }

    static void metodoC() {
        int resultado = 10 / 0;  // (4) ¡Excepción!
    }
}
```

El resultado al ejecutar este programa es:

```
Exception in thread "main" java.lang.ArithmeticException: / by zero
    at Main.metodoC(Main.java:14)
    at Main.metodoB(Main.java:10)
    at Main.metodoA(Main.java:6)
    at Main.main(Main.java:3)
```

Este volcado se denomina **stack trace** (traza de pila) y muestra la secuencia inversa de llamadas: desde el punto donde ocurrió la excepción (`metodoC`, línea 14) hasta el origen (`main`, línea 3). Cuando una excepción no se captura en ningún nivel, el programa termina abruptamente y la JVM imprime la traza completa.

### 5.1.2 El costo real de las excepciones

Uno de los mitos más persistentes en Java es que «las excepciones son lentas». La realidad es más matizada: **lanzar una excepción no es caro; lo caro es llenar el stack trace**.

> [!NOTE]
> ### 🛩️ El Registro de Vuelo de una Caja Negra de Avión (fillInStackTrace)
>
> Imagina que un avión de pasajeros (tu aplicación Java) está volando sin problemas. De repente, ocurre una pequeña turbulencia esperada o un desvío menor en el plan de vuelo (un error de negocio controlado o validación de datos fallida).
> - **El enfoque tradicional de excepciones**: Para registrar este desvío menor, el sistema detiene el avión por completo en pleno aire, activa la **caja negra** para registrar la posición exacta de cada tornillo, la velocidad de cada engranaje, el historial completo de cada aeropuerto por el que pasó desde su despegue (recorrer los frames de la pila de llamadas) y redacta un informe oficial de 50 páginas en ese mismo instante. ¡Esto es sumamente pesado y consume una cantidad inmensa de recursos!
> - **El enfoque ligero (`fillInStackTrace` desactivado)**: Si sabes que es una situación controlada (por ejemplo, validando si un email es incorrecto), no necesitas la caja negra completa. Simplemente quieres reportar *"turbulencia detectada"* sin gastar combustible ni tiempo generando el pesado informe de vuelo de 50 páginas.
>
> **En resumen**: Capturar un *stack trace* es el equivalente a congelar el estado de ejecución y tomar una radiografía en 3D de todas las funciones que estaban activas en ese milisegundo. Si usas excepciones como parte del flujo normal de negocio, este comportamiento por defecto ahogará el rendimiento de tu aplicación.

Cuando se crea un objeto `Throwable`, la JVM invoca automáticamente `fillInStackTrace()`, un método nativo que recorre la pila de llamadas y captura cada frame. Esta operación es costosa porque:

1. Requiere navegar por toda la pila de la JVM, frame por frame.
2. Convierte cada frame en un objeto `StackTraceElement`, que contiene el nombre de la clase, el método, el archivo fuente y el número de línea.
3. Cuanto más profunda es la pila de llamadas, más tarda.

El siguiente benchmark ilustra la diferencia:

```java
import java.util.concurrent.TimeUnit;

public class ExcepcionBenchmark {

    public static void main(String[] args) {
        final int ITERACIONES = 100_000;

        // 1. Creación simple (sin stack trace)
        long inicio = System.nanoTime();
        for (int i = 0; i < ITERACIONES; i++) {
            ExcepcionLigera e = new ExcepcionLigera();
        }
        long fin = System.nanoTime();
        System.out.printf("Crear %d excepciones SIN stack trace: %d ms%n",
                ITERACIONES,
                TimeUnit.NANOSECONDS.toMillis(fin - inicio));

        // 2. Creación normal (con stack trace)
        inicio = System.nanoTime();
        for (int i = 0; i < ITERACIONES; i++) {
            RuntimeException e = new RuntimeException();
        }
        fin = System.nanoTime();
        System.out.printf("Crear %d excepciones CON stack trace: %d ms%n",
                ITERACIONES,
                TimeUnit.NANOSECONDS.toMillis(fin - inicio));

        // 3. Creación + lanzamiento + captura
        inicio = System.nanoTime();
        for (int i = 0; i < ITERACIONES; i++) {
            try {
                throw new RuntimeException();
            } catch (RuntimeException e) {
                // capturada
            }
        }
        fin = System.nanoTime();
        System.out.printf("Crear + lanzar + capturar %d excepciones: %d ms%n",
                ITERACIONES,
                TimeUnit.NANOSECONDS.toMillis(fin - inicio));
    }

    /**
     * Excepción que NO llena el stack trace para ahorrar coste.
     */
    static class ExcepcionLigera extends RuntimeException {
        @Override
        public synchronized Throwable fillInStackTrace() {
            return this; // no hace nada → sin stack trace
        }
    }
}
```

Resultado típico en una máquina moderna (Java 21):

```
Crear 100000 excepciones SIN stack trace: 3 ms
Crear 100000 excepciones CON stack trace: 120 ms
Crear + lanzar + capturar 100000 excepciones: 185 ms
```

La diferencia es abrumadora: **llenar el stack trace cuesta entre 30 y 60 veces más** que simplemente crear el objeto. En escenarios de alto rendimiento (sistemas de trading, procesamiento de millones de transacciones), esto es inaceptable.

### 5.1.3 Cómo se captura el stack trace

El mecanismo interno es el método `Throwable.fillInStackTrace()`:

```java
// En java.lang.Throwable (simplificado)
public class Throwable implements Serializable {

    // Este es el método clave: es nativo y captura toda la pila
    public synchronized Throwable fillInStackTrace() {
        // Implementación nativa que recorre la pila de la JVM
        // y crea un array de StackTraceElement
        // ...
        return this;
    }

    // Se invoca automáticamente en el constructor
    public Throwable() {
        fillInStackTrace(); // ← aquí ocurre la magia (y el coste)
    }
}
```

Cada `StackTraceElement` contiene:

```java
public final class StackTraceElement implements Serializable {
    private final String classLoaderName;
    private final String moduleName;
    private final String moduleVersion;
    private final String declaringClass;   // clase donde está el método
    private final String methodName;       // nombre del método
    private final String fileName;         // archivo fuente (o null)
    private final int    lineNumber;       // número de línea (o -1)
}
```

Puedes acceder al stack trace programáticamente:

```java
public class InspeccionarStack {

    public static void main(String[] args) {
        try {
            metodoA();
        } catch (RuntimeException e) {
            StackTraceElement[] frames = e.getStackTrace();

            System.out.println("Stack trace de la excepción:");
            for (int i = 0; i < frames.length; i++) {
                StackTraceElement frame = frames[i];
                System.out.printf("  [%d] %s.%s(%s:%d)%n",
                        i,
                        frame.getClassName(),
                        frame.getMethodName(),
                        frame.getFileName(),
                        frame.getLineNumber()
                );
            }

            System.out.println("\nTambién puedes obtenerlo del thread actual:");
            StackTraceElement[] currentStack = Thread.currentThread().getStackTrace();
            for (StackTraceElement frame : currentStack) {
                System.out.println("  " + frame);
            }
        }
    }

    static void metodoA() {
        metodoB();
    }

    static void metodoB() {
        throw new RuntimeException("Explosión controlada");
    }
}
```

### 5.1.4 Excepciones y rendimiento: por qué NO usar excepciones para control de flujo normal

Usar excepciones como mecanismo de control de flujo es uno de los antipatrones más dañinos en Java. Considera estos dos enfoques para comprobar si una cadena es un número:

```java
// ENFOQUE INCORRECTO: usar excepciones para control de flujo
public static boolean esNumero_Excepciones(String s) {
    try {
        Integer.parseInt(s);
        return true;
    } catch (NumberFormatException e) {
        return false;
    }
}

// ENFOQUE CORRECTO: validación predictiva
public static boolean esNumero_Correcto(String s) {
    if (s == null || s.isEmpty()) {
        return false;
    }
    for (char c : s.toCharArray()) {
        if (!Character.isDigit(c) && c != '-' && c != '+') {
            return false;
        }
    }
    return true;
}
```

Un benchmark con un millón de cadenas no numéricas:

```
esNumero_Excepciones: 320 ms  (crea un millón de NumberFormatException)
esNumero_Correcto:       8 ms  (iteración simple de caracteres)
```

Las diferencias clave:

| Aspecto | Excepciones para flujo | Control de flujo normal |
|---|---|---|
| **Coste** | Muy alto (stack trace) | Mínimo (saltos condicionales) |
| **Legibilidad** | Oscurece la intención | Claro y directo |
| **Depuración** | Los breakpoints se disparan constantemente | Comportamiento predecible |
| **Optimización JIT** | El JIT no optimiza caminos de excepción | El JIT optimiza agresivamente |

**Regla de oro**: las excepciones deben reservarse para condiciones **excepcionales** (errores, fallos imprevistos). Para condiciones **esperadas** y **frecuentes** (validación de entrada, búsqueda sin resultado, fin de archivo), usa estructuras de control normales, `Optional<T>`, o patrones como `Result<T, E>` que veremos en la sección 5.10.

---

## 5.2 Jerarquía de excepciones

Todas las excepciones en Java heredan de la clase base `Throwable`. Su jerarquía principal es:

```
java.lang.Throwable (implements Serializable)
├── java.lang.Error
│   ├── VirtualMachineError
│   │   ├── OutOfMemoryError
│   │   ├── StackOverflowError
│   │   └── InternalError
│   ├── LinkageError
│   │   ├── NoClassDefFoundError
│   │   ├── UnsatisfiedLinkError
│   │   └── ExceptionInInitializerError
│   ├── AssertionError
│   ├── ThreadDeath
│   └── ... (otros errores graves de la JVM)
└── java.lang.Exception
    ├── java.lang.RuntimeException
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ArithmeticException
    │   ├── IllegalArgumentException
    │   │   └── NumberFormatException
    │   ├── ClassCastException
    │   ├── IndexOutOfBoundsException
    │   │   ├── ArrayIndexOutOfBoundsException
    │   │   └── StringIndexOutOfBoundsException
    │   ├── IllegalStateException
    │   ├── ConcurrentModificationException
    │   ├── UnsupportedOperationException
    │   └── DateTimeException (java.time)
    ├── java.io.IOException
    │   ├── FileNotFoundException
    │   ├── EOFException
    │   ├── SocketException
    │   └── ... (más excepciones de E/S)
    ├── java.sql.SQLException
    │   ├── SQLIntegrityConstraintViolationException
    │   └── ... (más excepciones de BD)
    ├── java.lang.ReflectiveOperationException
    │   ├── ClassNotFoundException
    │   ├── NoSuchMethodException
    │   └── NoSuchFieldException
    ├── java.lang.InterruptedException
    ├── java.text.ParseException
    └── javax.naming.NamingException
```

### 5.2.1 `Error`

Representa problemas graves que ocurren en la JVM y que una aplicación normal **no debería intentar capturar**. Son errores de los que generalmente un programa no puede recuperarse. Ejemplos:

- **`OutOfMemoryError`**: el *heap* de la JVM se ha agotado. No hay memoria para nuevos objetos.
- **`StackOverflowError`**: la pila de llamadas ha superado su límite, normalmente por recursión infinita.
- **`NoClassDefFoundError`**: la JVM no puede encontrar una clase que estaba presente en tiempo de compilación.
- **`ExceptionInInitializerError`**: un inicializador estático ha lanzado una excepción.

```java
public static void recursionSinFin() {
    recursionSinFin(); // se llama a sí misma sin caso base
}
```

No se recomienda capturar `Error` con bloques `try-catch`, ya que el estado de la JVM suele ser irrecuperable.

### 5.2.2 `RuntimeException` (excepciones no comprobadas)

Son excepciones que ocurren durante la ejecución y que representan **errores de programación**. El compilador **no obliga** a manejarlas. Las más comunes son:

| Excepción | Se lanza cuando... |
|---|---|
| `NullPointerException` | Intentas usar una referencia `null` (invocar un método, acceder a un campo, etc.) |
| `ArrayIndexOutOfBoundsException` | Accedes a un índice inválido de un arreglo (negativo o >= longitud) |
| `ArithmeticException` | Ocurre una operación aritmética ilegal (ej: división entera por cero) |
| `IllegalArgumentException` | Un método recibe un argumento inapropiado |
| `ClassCastException` | Intentas convertir un objeto a una clase incompatible |
| `NumberFormatException` | Intentas convertir una cadena a número con formato inválido |
| `IllegalStateException` | Se invoca un método en un momento inapropiado |
| `UnsupportedOperationException` | Se invoca una operación no soportada (común en colecciones no modificables) |
| `ConcurrentModificationException` | Una colección se modifica mientras se itera sin usar el iterador |

```java
String texto = null;
texto.length();                      // NullPointerException

int[] numeros = {1, 2, 3};
int x = numeros[5];                  // ArrayIndexOutOfBoundsException

int division = 100 / 0;              // ArithmeticException

Object obj = "Hola";
Integer num = (Integer) obj;         // ClassCastException

int valor = Integer.parseInt("xyz"); // NumberFormatException

List<String> lista = List.of("a", "b");
lista.add("c");                      // UnsupportedOperationException
```

### 5.2.3 Excepciones comprobadas (Checked Exceptions)

Son excepciones que el compilador **obliga** a manejar: o las capturas con `try-catch` o las declaras con `throws` en el método. Representan condiciones recuperables previsibles, normalmente relacionadas con operaciones externas (E/S, red, base de datos).

Ejemplos comunes:

- **`IOException`**: error genérico de entrada/salida.
- **`FileNotFoundException`**: intento de abrir un archivo que no existe (hereda de `IOException`).
- **`SQLException`**: error al operar con una base de datos.
- **`InterruptedException`**: un hilo ha sido interrumpido mientras esperaba o dormía.
- **`ClassNotFoundException`**: no se encuentra una clase solicitada por nombre.

```java
// Esto NO compila: el compilador obliga a manejar IOException
FileReader lector = new FileReader("archivo.txt"); // error de compilación
```

### 5.2.4 Métodos clave de `Throwable`

La interfaz pública de `Throwable` es amplia y conviene conocerla:

```java
public class Throwable implements Serializable {

    // Constructores
    public Throwable()
    public Throwable(String message)
    public Throwable(String message, Throwable cause)
    public Throwable(Throwable cause)

    // Mensaje y causa
    public String   getMessage()           // mensaje descriptivo
    public String   getLocalizedMessage()  // versión localizada del mensaje
    public Throwable getCause()            // excepción que causó esta (encadenamiento)

    // Stack trace
    public StackTraceElement[] getStackTrace()
    public void setStackTrace(StackTraceElement[] stackTrace)
    public synchronized Throwable fillInStackTrace()

    // Excepciones suprimidas (try-with-resources)
    public final void addSuppressed(Throwable exception)
    public final Throwable[] getSuppressed()

    // Impresión
    public void printStackTrace()
    public void printStackTrace(PrintStream s)
    public void printStackTrace(PrintWriter w)
}
```

### 5.2.5 Encadenamiento de excepciones (Exception Chaining)

El encadenamiento permite preservar la causa raíz cuando una excepción se traduce a otra:

```java
public class CuentaService {

    public void procesarPago(Long cuentaId, BigDecimal monto) throws PagoException {
        try {
            repositorio.debitar(cuentaId, monto); // puede lanzar SQLException
        } catch (SQLException e) {
            throw new PagoException(
                "Error al procesar pago para cuenta " + cuentaId, e
            );
        }
    }
}

// Al depurar:
try {
    cuentaService.procesarPago(42L, new BigDecimal("100"));
} catch (PagoException e) {
    System.out.println("Error principal: " + e.getMessage());
    System.out.println("Causa raíz: " + e.getCause().getClass().getSimpleName());
    System.out.println("Mensaje causa: " + e.getCause().getMessage());
    e.printStackTrace(); // muestra ambas trazas encadenadas
}
```

La salida muestra la cadena completa:

```
PagoException: Error al procesar pago para cuenta 42
    at com.banco.service.CuentaService.procesarPago(CuentaService.java:28)
    at com.banco.Main.main(Main.java:12)
Caused by: java.sql.SQLException: Deadlock detectado en tabla cuentas
    at com.banco.repository.CuentaRepository.debitar(CuentaRepository.java:45)
    at com.banco.service.CuentaService.procesarPago(CuentaService.java:25)
    ... 1 more
```

---

## 5.3 try-catch-finally

### 5.3.1 Estructura básica

```java
try {
    // Código que puede lanzar una excepción
    int resultado = 10 / 0;
} catch (ArithmeticException e) {
    // Código que maneja la excepción
    System.out.println("No se puede dividir por cero: " + e.getMessage());
}
```

### 5.3.2 Múltiples bloques catch

Cuando el bloque `try` puede lanzar varios tipos de excepción, se usan múltiples `catch`. **El orden importa**: deben ir de la más específica a la más general, porque Java evalúa los `catch` en secuencia y solo ejecuta el primero que coincida.

```java
try {
    String texto = null;
    System.out.println(texto.length());
    int[] arr = {1, 2};
    System.out.println(arr[5]);
} catch (NullPointerException e) {
    System.out.println("Referencia nula encontrada.");
} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println("Índice fuera del arreglo.");
} catch (RuntimeException e) {
    System.out.println("Otra RuntimeException: " + e);
}
```

Si pusiéramos `catch (RuntimeException e)` antes que `catch (NullPointerException e)`, este último sería inalcanzable y el compilador daría un error.

### 5.3.3 Multi-catch (Java 7+)

Cuando varios tipos de excepción se manejan de la misma forma:

```java
try {
    operacionRiesgosa();
} catch (IOException | SQLException e) {
    // Manejo común para ambos tipos
    logger.error("Error de E/S o base de datos: {}", e.getMessage(), e);
}

// Combinado con múltiples catch
try {
    procesar();
} catch (IOException | SQLException e) {
    logger.error("Error externo", e);
} catch (RuntimeException e) {
    logger.error("Error interno", e);
}
```

Restricciones del multi-catch:
- La variable `e` es implícitamente `final` (no puedes reasignarla).
- Los tipos no pueden tener relación de herencia entre sí (ej: `IOException | FileNotFoundException` no compila).

### 5.3.4 El bloque `finally`

El bloque `finally` se ejecuta **siempre**, independientemente de que se lance una excepción o no, e incluso si hay una instrucción `return` en el `try` o en un `catch`.

```java
public static int metodoConFinally() {
    try {
        System.out.println("Dentro del try");
        return 1;
    } catch (Exception e) {
        System.out.println("Dentro del catch");
        return 2;
    } finally {
        System.out.println("Dentro del finally (siempre se ejecuta)");
    }
}

// Salida:
// Dentro del try
// Dentro del finally (siempre se ejecuta)
// (el método retorna 1)
```

### 5.3.5 Casos especiales del finally

**¿Qué pasa si el finally tiene su propio return?**

```java
public static int casoPeligroso() {
    try {
        return 42;
    } finally {
        return 99;  // ¡PELIGROSO! Sobrescribe el valor de retorno del try
    }
}
// Retorna 99, no 42. El return del finally siempre prevalece.
```

**¿Y si tanto el try como el finally lanzan excepción?**

```java
public static void dobleExcepcion() {
    try {
        throw new RuntimeException("Error en try");
    } finally {
        throw new RuntimeException("Error en finally"); // ¡Oculta la primera!
    }
}
// La excepción del try se pierde. Solo se propaga la del finally.
```

**¿Y si se llama a System.exit() en el try?**

```java
public static void exitEnTry() {
    try {
        System.out.println("Saliendo...");
        System.exit(0); // Termina la JVM inmediatamente
    } finally {
        System.out.println("Esto NUNCA se imprimirá");
    }
}
```

`System.exit()` termina la JVM inmediatamente; el `finally` no se ejecuta.

### 5.3.6 try-catch anidado

```java
try {
    try {
        int[] arr = new int[3];
        arr[10] = 42;
    } catch (ArrayIndexOutOfBoundsException e) {
        System.out.println("Índice inválido en el bloque interno.");
        throw new RuntimeException("Excepción propagada al bloque externo.");
    }
} catch (RuntimeException e) {
    System.out.println("Capturada en el bloque externo: " + e.getMessage());
}
```

---

## 5.4 throw y throws

### 5.4.1 `throw`: lanzar una excepción explícitamente

```java
public static double dividir(int a, int b) {
    if (b == 0) {
        throw new IllegalArgumentException("El divisor no puede ser cero.");
    }
    return (double) a / b;
}
```

Puede lanzarse cualquier objeto que herede de `Throwable`.

### 5.4.2 `throws`: declarar excepciones en la firma del método

Cuando un método no maneja una excepción comprobada, debe declararla con `throws`:

```java
public static void leerArchivo(String ruta) throws IOException {
    FileReader fr = new FileReader(ruta);
    BufferedReader br = new BufferedReader(fr);
    System.out.println(br.readLine());
    br.close();
}
```

Múltiples excepciones separadas por comas:

```java
public void transaccionCompleja()
        throws IOException, SQLException, TimeoutException {
    // ...
}
```

### 5.4.3 Propagación de excepciones

Cuando un método lanza una excepción y no la captura, esta se propaga al método que lo invocó. Si ese método tampoco la captura, sigue subiendo hasta que alguien la maneje o el programa termine.

```java
public static void main(String[] args) {
    try {
        nivel1();
    } catch (Exception e) {
        System.out.println("Capturada en main: " + e);
        e.printStackTrace();
    }
}

static void nivel1() { nivel2(); }
static void nivel2() { nivel3(); }
static void nivel3() { throw new RuntimeException("Error en nivel 3"); }
```

Visualización de la propagación:

```
main()
  └─ nivel1()
       └─ nivel2()
            └─ nivel3()  ← throw RuntimeException
            ✗ (no captura → se sale de nivel3)
       ✗ (no captura → se sale de nivel2)
  ← catch (RuntimeException e)  ← capturada aquí
```

### 5.4.4 ¿Cuándo usar throws vs try-catch?

| Situación | Recomendación |
|---|---|
| Operación de E/S en biblioteca reutilizable | `throws IOException` (el cliente decide) |
| Operación de BD en un servicio | Traducir a excepción de dominio con `throw` |
| Validación de parámetros en setter | `throw new IllegalArgumentException(...)` |
| Tarea en segundo plano (hilo) | `try-catch` obligatorio (el hilo no propaga al principal) |

Usa `throws` cuando quieres que sea el método llamador quien decida cómo manejar la excepción. Usa `try-catch` cuando sabes cómo recuperarte de la excepción en ese punto. Como regla general: captura la excepción solo si puedes hacer algo útil con ella.

---

## 5.5 Excepciones checked vs unchecked

### 5.5.1 Tabla comparativa

| Característica | Checked Exceptions | Unchecked Exceptions |
|---|---|---|
| **Herencia** | Heredan de `Exception` pero no de `RuntimeException` | Heredan de `RuntimeException` (o de `Error`) |
| **Verificación del compilador** | Sí: obliga a capturarlas o declararlas con `throws` | No: el compilador no las verifica |
| **Representan** | Condiciones recuperables fuera del control del programador | Errores de programación (bugs) |
| **Ejemplos** | `IOException`, `SQLException`, `FileNotFoundException` | `NullPointerException`, `IllegalArgumentException` |
| **Manejo** | Deben manejarse explícitamente | Opcional, pero recomendable en puntos estratégicos |

### 5.5.2 ¿Por qué existen las excepciones comprobadas?

Java fue diseñado con la filosofía de que las condiciones de error recuperables deben ser parte explícita del contrato del método. James Gosling, creador de Java, argumentó que esto fuerza a los programadores a pensar en los fallos posibles y a escribir código más robusto.

### 5.5.3 La controversia de las checked exceptions

Con los años ha surgido debate sobre este mecanismo. Sus críticos argumentan que:

- Obligan a escribir código repetitivo (`catch` vacíos o re-lanzamientos en cadena).
- Rompen la transparencia de las lambdas y streams.
- Muchos programadores las capturan y las ignoran, lo que es peor que no tenerlas.
- Contaminan las firmas de los métodos en toda la cadena de llamadas.

Lenguajes posteriores como C#, Kotlin, Scala o Go optaron por no incluir excepciones comprobadas.

### 5.5.4 La técnica del "sneaky throw"

Una técnica avanzada (y controvertida) para lanzar checked exceptions sin declararlas:

> [!NOTE]
> ### 🦹 El Contrabandista con Camuflaje Holográfico (Sneaky Throw)
>
> Imagina una aduana súper estricta en la frontera (el compilador de Java) que revisa todo el equipaje. Tiene una lista negra de mercancías prohibidas llamadas **Checked Exceptions** (como `IOException` o `SQLException`). Si intentas pasar una sin una visa especial (`throws` en la firma de tu método), el guardia fronterizo de la aduana te arrestará de inmediato y no te dejará continuar (error de compilación).
>
> Sin embargo, un contrabandista inteligente inventa un dispositivo de **camuflaje holográfico (los Genéricos `<E extends Throwable>`)**:
> - Pone una **Checked Exception** real (por ejemplo, una maleta con dinamita `SQLException`) dentro del contenedor.
> - Activa el camuflaje holográfico para que el contenedor parezca una caja genérica e inocua de tipo `E`.
> - El guardia fronterizo (el compilador) mira el contenedor, ve la firma holográfica `E` (que bajo borrado de tipos se interpreta como si no tuviera restricciones o fuera una excepción no controlada / `RuntimeException`), y dice: *"Adelante, puedes pasar sin declarar"* (compila sin `throws`).
> - Una vez que el equipaje llega a su destino dentro del país en tiempo de ejecución (la JVM), **el camuflaje se disipa (borrado de tipos)**, revelando la maleta original con dinamita, la cual detona de inmediato en pleno vuelo de ejecución.
>
> **En resumen**: El *Sneaky Throw* burla los controles del compilador usando genéricos para lanzar excepciones comprobadas sin declararlas formalmente. Herramientas como Lombok (`@SneakyThrows`) lo hacen automáticamente por debajo para ahorrar líneas de código redundantes, pero debe usarse con precaución ya que el receptor de tu método no sabrá que debe prepararse para esa explosión.

```java
public class SneakyThrow {

    @SuppressWarnings("unchecked")
    public static <E extends Throwable> void lanzar(Throwable e) throws E {
        throw (E) e; // el compilador cree que es unchecked por el borrado de tipos
    }

    public static void main(String[] args) {
        lanzar(new SQLException("Error de base de datos")); // compila sin throws
    }
}
```

Esta técnica la usa Lombok con `@SneakyThrows` y también aparece en algunas bibliotecas. Úsala con moderación.

### 5.5.5 Buenas prácticas

- **Unchecked**: para errores de programación y condiciones de las que el código normalmente no se recupera.
- **Checked**: para condiciones recuperables previsibles que dependen de factores externos.

```java
// Uso correcto de checked: el llamador debe decidir qué hacer
public void cargarConfiguracion(String ruta) throws IOException {
    Properties props = new Properties();
    props.load(new FileReader(ruta));
}

// Uso correcto de unchecked: error de programación
public void setEdad(int edad) {
    if (edad < 0) {
        throw new IllegalArgumentException("La edad no puede ser negativa: " + edad);
    }
    this.edad = edad;
}
```

---

## 5.6 try-with-resources (Java 7+)

El bloque **try-with-resources** cierra automáticamente los recursos que implementan la interfaz `AutoCloseable` al finalizar el bloque, sin necesidad de un `finally` explícito.

### 5.6.1 La interfaz AutoCloseable

Cualquier clase que implemente `java.lang.AutoCloseable` (o su subinterfaz `java.io.Closeable`) puede usarse dentro de un try-with-resources. La jerarquía es:

```java
// java.lang.AutoCloseable (Java 7+)
public interface AutoCloseable {
    void close() throws Exception;
}

// java.io.Closeable (Java 5+, extiende AutoCloseable)
public interface Closeable extends AutoCloseable {
    void close() throws IOException;  // más específica: solo IOException
}
```

Esto incluye: `FileInputStream`, `FileReader`, `BufferedReader`, `Connection`, `Statement`, `ResultSet`, `Socket`, `ServerSocket`, `Scanner`, entre muchas otras.

### 5.6.2 Sintaxis y ejemplos

```java
// Con archivos
try (BufferedReader br = new BufferedReader(new FileReader("datos.txt"))) {
    String linea;
    while ((linea = br.readLine()) != null) {
        System.out.println(linea);
    }
} catch (IOException e) {
    System.err.println("Error al leer el archivo: " + e.getMessage());
}
// br.close() se invoca automáticamente, incluso si ocurre una excepción
```

```java
// Con conexión a base de datos (JDBC)
String sql = "SELECT id, nombre FROM usuarios";

try (Connection conn = DriverManager.getConnection(url, user, password);
     PreparedStatement stmt = conn.prepareStatement(sql);
     ResultSet rs = stmt.executeQuery()) {

    while (rs.next()) {
        System.out.println(rs.getInt("id") + ": " + rs.getString("nombre"));
    }
} catch (SQLException e) {
    System.err.println("Error de base de datos: " + e.getMessage());
}
// conn, stmt y rs se cierran automáticamente, en orden inverso al de declaración
```

### 5.6.3 Múltiples recursos y orden de cierre

Se pueden declarar varios recursos en el mismo `try`, separados por punto y coma. Los recursos se cierran en **orden inverso** al de su declaración. Esto es importante cuando un recurso depende de otro:

```java
try (
    FileInputStream fis = new FileInputStream("origen.txt");
    BufferedInputStream bis = new BufferedInputStream(fis);  // depende de fis
    DataInputStream dis = new DataInputStream(bis)           // depende de bis
) {
    int valor = dis.readInt();
} catch (IOException e) {
    System.err.println("Error de E/S: " + e.getMessage());
}
// Orden de cierre: dis → bis → fis (inverso al de declaración)
// Correcto: dis.close() necesita bis abierto, bis.close() necesita fis abierto
```

**¿Por qué el orden inverso?** Porque un recurso que envuelve a otro necesita que el recurso envuelto siga abierto durante su propio cierre. `DataInputStream.close()` puede necesitar escribir buffers pendientes a través del `BufferedInputStream` subyacente, que a su vez necesita el `FileInputStream` abierto.

### 5.6.4 Implementando AutoCloseable en tus propias clases

Crear un recurso propio que funcione con try-with-resources es tan simple como implementar `AutoCloseable`:

```java
/**
 * Simula una conexión a un servicio remoto.
 * Implementa AutoCloseable para usarse con try-with-resources.
 */
public class ConexionRemota implements AutoCloseable {

    private final String host;
    private final int puerto;
    private boolean conectado = false;
    private String idConexion;

    public ConexionRemota(String host, int puerto) {
        this.host = host;
        this.puerto = puerto;
    }

    /**
     * Establece la conexión con el servidor remoto.
     */
    public void conectar() throws ConnectionException {
        System.out.printf("[%s] Conectando a %s:%d...%n",
                Thread.currentThread().getName(), host, puerto);

        if (host == null || host.isEmpty()) {
            throw new ConnectionException("Host inválido: " + host);
        }

        this.idConexion = java.util.UUID.randomUUID().toString().substring(0, 8);
        this.conectado = true;
        System.out.printf("[%s] Conectado. ID: %s%n",
                Thread.currentThread().getName(), idConexion);
    }

    /**
     * Envía un mensaje a través de la conexión.
     */
    public String enviarMensaje(String mensaje) throws ConnectionException {
        if (!conectado) {
            throw new ConnectionException("No conectado. Llama a conectar() primero.");
        }
        System.out.printf("[%s] Enviando: \"%s\"%n",
                Thread.currentThread().getName(), mensaje);
        return "OK: " + mensaje.toUpperCase();
    }

    /**
     * Cierra la conexión. Invocado automáticamente por try-with-resources.
     */
    @Override
    public void close() throws ConnectionException {
        if (conectado) {
            System.out.printf("[%s] Cerrando conexión %s...%n",
                    Thread.currentThread().getName(), idConexion);
            this.conectado = false;
            System.out.printf("[%s] Conexión %s cerrada.%n",
                    Thread.currentThread().getName(), idConexion);
        }
    }

    public String getIdConexion() { return idConexion; }

    // Ejemplo de uso
    public static void main(String[] args) {
        try (ConexionRemota conn = new ConexionRemota("api.miservicio.com", 443)) {
            conn.conectar();
            String respuesta = conn.enviarMensaje("Hola, servidor");
            System.out.println("Respuesta: " + respuesta);
        } catch (ConnectionException e) {
            System.err.println("Error: " + e.getMessage());
        }
        // conn.close() se llama automáticamente al salir del try
    }
}

// Excepción personalizada para el ejemplo
class ConnectionException extends Exception {
    public ConnectionException(String mensaje) { super(mensaje); }
    public ConnectionException(String mensaje, Throwable causa) { super(mensaje, causa); }
}
```

Salida del ejemplo:

```
[main] Conectando a api.miservicio.com:443...
[main] Conectado. ID: a3f7b2c1
[main] Enviando: "Hola, servidor"
Respuesta: OK: HOLA, SERVIDOR
[main] Cerrando conexión a3f7b2c1...
[main] Conexión a3f7b2c1 cerrada.
```

### 5.6.5 Excepciones suprimidas (Suppressed Exceptions)

Si se lanza una excepción tanto en el bloque `try` como durante el cierre automático, la excepción del `close()` **no reemplaza** a la del `try`; se añade como **excepción suprimida**. Se recuperan con `getSuppressed()`:

> [!NOTE]
> ### 🚛 El Accidente del Camión Principal y los Retrasos de los Autos Traseros (Suppressed Exceptions)
>
> Imagina una autopista de un solo carril en donde viajan varios vehículos en fila. El primer vehículo grande es la **operación principal de tu negocio** (el código dentro del bloque `try`), y los autos que le siguen detrás son las **tareas de limpieza y cierre de recursos** (el método `close()` del try-with-resources).
>
> 1. **El Accidente del Camión (La Excepción Principal)**:
>    - Si el gran camión choca y bloquea por completo la autopista (se lanza una excepción dentro de tu bloque `try`), el tráfico se detiene de golpe. Este es el problema más grave y urgente que debes reportar a emergencias (la excepción capturada por el bloque `catch`).
> 2. **Los Choques Traseros de Limpieza (Las Excepciones Suprimidas)**:
>    - Dado que la autopista está bloqueada, el sistema intenta de forma desesperada pero ordenada apartar y apagar los motores de los autos de atrás (`close()`). Sin embargo, debido al impacto, dos de estos autos de limpieza también fallan y chocan (`RuntimeException` en el `close()`).
>    - **El dilema de Java pre-JDK 7**: En las versiones antiguas de Java, la grúa que venía a rescatar el camión ignoraba el choque principal y solo reportaba el último choquecito menor de los autos traseros, perdiendo el rastro del gran accidente original.
>    - **La solución actual (Excepciones Suprimidas)**: Ahora, Java mantiene al gran camión como el accidente principal en el reporte oficial. Y en la sección de "Notas adicionales" del mismo reporte (el array `getSuppressed()`), añade y adjunta de forma ordenada los pequeños choques traseros de limpieza.
>
> **En resumen**: Gracias a las excepciones suprimidas, si tu código de negocio falla y también falla la limpieza de archivos o bases de datos, nunca perderás el error original que causó la catástrofe, pudiendo auditar los fallos secundarios desde el mismo objeto de excepción principal.

```java
static class RecursoFallo implements AutoCloseable {
    private final String nombre;

    RecursoFallo(String nombre) { this.nombre = nombre; }

    @Override
    public void close() {
        throw new RuntimeException("Error al cerrar " + nombre);
    }
}

public static void main(String[] args) {
    // Caso 1: error en el try + error en el close
    try (RecursoFallo r = new RecursoFallo("recurso-A")) {
        throw new RuntimeException("Error dentro del try.");
    } catch (RuntimeException e) {
        System.out.println("Excepción principal: " + e.getMessage());
        for (Throwable suprimida : e.getSuppressed()) {
            System.out.println("  → Suprimida: " + suprimida.getMessage());
        }
    }
}
// Salida:
// Excepción principal: Error dentro del try.
//   → Suprimida: Error al cerrar recurso-A
```

```java
// Caso 2: múltiples recursos, ambos fallan al cerrar
try (
    RecursoFallo r1 = new RecursoFallo("recurso-A");
    RecursoFallo r2 = new RecursoFallo("recurso-B")
) {
    System.out.println("Operación exitosa.");
} catch (RuntimeException e) {
    System.out.println("Excepción principal: " + e.getMessage());
    for (Throwable suprimida : e.getSuppressed()) {
        System.out.println("  → Suprimida: " + suprimida.getMessage());
    }
}
// Salida:
// Excepción principal: Error al cerrar recurso-B
//   → Suprimida: Error al cerrar recurso-A
// Nota: r2 se cierra primero (orden inverso), su excepción es la principal
```

### 5.6.6 Cómo manejar casos donde close() también puede fallar

A veces `close()` puede lanzar excepciones que requieren manejo especial (flush pendiente, commit/rollback).

**Estrategia 1: Separar la operación del cierre**

```java
public void procesarArchivo(String ruta) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(ruta));
    try {
        String linea;
        while ((linea = br.readLine()) != null) {
            procesarLinea(linea);
        }
    } finally {
        try {
            br.close();
        } catch (IOException e) {
            logger.warn("Error no crítico al cerrar el reader: {}", e.getMessage());
        }
    }
}
```

**Estrategia 2: try-with-resources con catch adicional**

```java
try (OutputStream os = new FileOutputStream("salida.bin")) {
    os.write(datos);
    os.flush(); // forzar escritura antes del close implícito
} catch (IOException e) {
    logger.error("Error de E/S durante escritura o cierre", e);
    throw new ProcesamientoException("Error al escribir archivo de salida", e);
}
```

**Estrategia 3: Capturar suprimidas y decidir**

```java
try (RecursoComplejo r = new RecursoComplejo()) {
    r.operacion();
} catch (OperacionException e) {
    logger.error("Operación fallida", e);
    for (Throwable sup : e.getSuppressed()) {
        logger.error("Error durante cierre: {}", sup.getMessage());
    }
    throw e;
}
```

### 5.6.7 try-with-resources con recursos del mismo tipo

Cuando tienes varios recursos del mismo tipo, los declaras separados por punto y coma:

```java
try (
    FileInputStream origen1 = new FileInputStream("archivo1.txt");
    FileInputStream origen2 = new FileInputStream("archivo2.txt");
    FileOutputStream destino = new FileOutputStream("combinado.txt")
) {
    origen1.transferTo(destino);
    origen2.transferTo(destino);
} catch (IOException e) {
    logger.error("Error al combinar archivos", e);
}
```

Si necesitas muchos recursos del mismo tipo dinámicamente:

```java
/**
 * Permite cerrar múltiples recursos del mismo tipo de una vez.
 */
public class MultiRecurso<T extends AutoCloseable> implements AutoCloseable {

    private final List<T> recursos;

    @SafeVarargs
    public MultiRecurso(T... recursos) {
        this.recursos = new ArrayList<>(Arrays.asList(recursos));
    }

    public List<T> getRecursos() {
        return Collections.unmodifiableList(recursos);
    }

    @Override
    public void close() throws Exception {
        Exception primeraExcepcion = null;
        // Cerrar en orden inverso
        for (int i = recursos.size() - 1; i >= 0; i--) {
            try {
                recursos.get(i).close();
            } catch (Exception e) {
                if (primeraExcepcion == null) {
                    primeraExcepcion = e;
                } else {
                    primeraExcepcion.addSuppressed(e);
                }
            }
        }
        if (primeraExcepcion != null) {
            throw primeraExcepcion;
        }
    }
}

// Uso:
try (MultiRecurso<FileInputStream> recursos = new MultiRecurso<>(
        new FileInputStream("a.txt"),
        new FileInputStream("b.txt"),
        new FileInputStream("c.txt")
)) {
    for (FileInputStream fis : recursos.getRecursos()) {
        // procesar cada archivo...
    }
}
```

### 5.6.8 Estilo antiguo vs try-with-resources

**Sin try-with-resources (Java 6 o anterior)**:

```java
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("datos.txt"));
    String linea;
    while ((linea = br.readLine()) != null) {
        System.out.println(linea);
    }
} catch (IOException e) {
    System.err.println("Error: " + e.getMessage());
} finally {
    if (br != null) {
        try {
            br.close();
        } catch (IOException e) {
            System.err.println("Error al cerrar: " + e.getMessage());
        }
    }
}
```

Problemas de este estilo: código verboso, fácil olvidar el cierre, anidación excesiva, y si tanto el `try` como el `finally` lanzan excepción, la del `finally` sobrescribe a la del `try`, perdiendo información valiosa.

El try-with-resources resuelve todos estos problemas con menos de la mitad de líneas y mejor semántica.

---

## 5.7 Crear excepciones propias

Crear excepciones personalizadas permite expresar condiciones de error específicas del dominio de la aplicación.

### 5.7.1 Extendiendo Exception (checked)

```java
public class SaldoInsuficienteException extends Exception {

    private final String numeroCuenta;
    private final BigDecimal saldoDisponible;
    private final BigDecimal montoSolicitado;

    public SaldoInsuficienteException(String numeroCuenta,
                                       BigDecimal saldoDisponible,
                                       BigDecimal montoSolicitado) {
        super(String.format(
            "Saldo insuficiente en cuenta %s: disponible=%s, solicitado=%s",
            numeroCuenta, saldoDisponible, montoSolicitado
        ));
        this.numeroCuenta = numeroCuenta;
        this.saldoDisponible = saldoDisponible;
        this.montoSolicitado = montoSolicitado;
    }

    public SaldoInsuficienteException(String mensaje, Throwable causa) {
        super(mensaje, causa);
    }

    public String getNumeroCuenta() { return numeroCuenta; }
    public BigDecimal getSaldoDisponible() { return saldoDisponible; }
    public BigDecimal getMontoSolicitado() { return montoSolicitado; }
}
```

Quien invoque un método que lance esta excepción estará **obligado** a manejarla:

```java
public class CuentaBancaria {
    private String numero;
    private BigDecimal saldo;

    public CuentaBancaria(String numero, BigDecimal saldoInicial) {
        this.numero = numero;
        this.saldo = saldoInicial;
    }

    public void retirar(BigDecimal monto) throws SaldoInsuficienteException {
        if (monto.compareTo(saldo) > 0) {
            throw new SaldoInsuficienteException(numero, saldo, monto);
        }
        saldo = saldo.subtract(monto);
    }
}
```

### 5.7.2 Extendiendo RuntimeException (unchecked)

```java
public class ProductoNoEncontradoException extends RuntimeException {
    private final String codigoProducto;

    public ProductoNoEncontradoException(String codigoProducto) {
        super("Producto no encontrado: " + codigoProducto);
        this.codigoProducto = codigoProducto;
    }

    public ProductoNoEncontradoException(String codigoProducto, Throwable causa) {
        super("Producto no encontrado: " + codigoProducto, causa);
        this.codigoProducto = codigoProducto;
    }

    public String getCodigoProducto() { return codigoProducto; }
}
```

Al ser unchecked, no obliga al llamador a capturarla.

### 5.7.3 Constructores recomendados

Una buena excepción personalizada debe ofrecer al menos estos constructores:

```java
public class MiExcepcion extends Exception {

    // 1. Solo mensaje
    public MiExcepcion(String mensaje) { super(mensaje); }

    // 2. Mensaje y causa (el más importante para encadenamiento)
    public MiExcepcion(String mensaje, Throwable causa) { super(mensaje, causa); }

    // 3. Solo causa
    public MiExcepcion(Throwable causa) { super(causa); }

    // 4. Sin argumentos (menos común)
    public MiExcepcion() { super(); }
}
```

### 5.7.4 Campos adicionales para contexto

Añadir campos específicos del dominio hace que la excepción sea mucho más útil para depuración:

```java
public class PedidoRechazadoException extends RuntimeException {

    private final Long pedidoId;
    private final String motivo;
    private final LocalDateTime fecha;
    private final Map<String, Object> contexto;

    public PedidoRechazadoException(Long pedidoId, String motivo) {
        super(String.format("Pedido %d rechazado: %s", pedidoId, motivo));
        this.pedidoId = pedidoId;
        this.motivo = motivo;
        this.fecha = LocalDateTime.now();
        this.contexto = new HashMap<>();
    }

    public PedidoRechazadoException(Long pedidoId, String motivo, Throwable causa) {
        super(String.format("Pedido %d rechazado: %s", pedidoId, motivo), causa);
        this.pedidoId = pedidoId;
        this.motivo = motivo;
        this.fecha = LocalDateTime.now();
        this.contexto = new HashMap<>();
    }

    /**
     * Añade información contextual para depuración (contenido del carrito,
     * método de pago, dirección de envío). Soporta encadenamiento fluido.
     */
    public PedidoRechazadoException conContexto(String clave, Object valor) {
        this.contexto.put(clave, valor);
        return this;
    }

    public Long getPedidoId() { return pedidoId; }
    public String getMotivo() { return motivo; }
    public LocalDateTime getFecha() { return fecha; }
    public Map<String, Object> getContexto() {
        return Collections.unmodifiableMap(contexto);
    }

    // Uso:
    public static void main(String[] args) {
        throw new PedidoRechazadoException(12345L, "Stock insuficiente")
            .conContexto("productos", List.of("SKU-001", "SKU-002"))
            .conContexto("clienteId", 67890L)
            .conContexto("metodoPago", "TARJETA_CREDITO");
    }
}
```

### 5.7.5 ¿Cuándo crear excepciones personalizadas?

| Crear excepción personalizada cuando... | Usar excepción estándar cuando... |
|---|---|
| El error es específico del dominio de negocio | Una estándar describe bien el problema |
| Necesitas adjuntar datos adicionales (ID, monto, etc.) | Solo necesitas un mensaje descriptivo |
| Quieres que el código cliente reaccione a errores específicos | Es un error genérico de programación |
| Diferentes causas requieren distinto tratamiento | Basta con `IllegalArgumentException` o `IllegalStateException` |

No crees una excepción personalizada si una estándar como `IllegalArgumentException`, `IllegalStateException` o `UnsupportedOperationException` expresa adecuadamente lo ocurrido.

---

## 5.8 Buenas prácticas

### 5.8.1 No captures y ignores excepciones

El peor pecado en el manejo de excepciones es capturarlas y no hacer nada con ellas:

```java
// ¡TERRIBLE! Nunca hagas esto:
try {
    operacionRiesgosa();
} catch (Exception e) {
    // silencio...
}

// Mejor: al menos registra el error
try {
    operacionRiesgosa();
} catch (Exception e) {
    logger.error("Error en operacionRiesgosa", e);
}
```

### 5.8.2 No captures `Exception` de forma indiscriminada

Capturar `Exception` (o peor, `Throwable`) puede esconder errores graves como `NullPointerException` que deberían hacer que el programa falle:

```java
// Mal: captura todo indiscriminadamente
try {
    // ...
} catch (Exception e) {
    // ¿Esto captura NullPointerException? ¿Estás seguro?
}

// Bien: sé específico
try {
    // ...
} catch (IOException e) {
    logger.error("Error de E/S", e);
} catch (SQLException e) {
    logger.error("Error de base de datos", e);
}
```

### 5.8.3 Registra (log) las excepciones apropiadamente

- Usa un framework de logging como SLF4J, Log4j o `java.util.logging`.
- Incluye siempre la excepción original como último parámetro para preservar la traza.
- No uses `e.printStackTrace()` en producción; usa logs que van a archivos o sistemas de monitoreo.
- Usa marcadores de posición `{}` en vez de concatenación de strings.

```java
private static final Logger logger = LoggerFactory.getLogger(MiClase.class);

try {
    procesarPedido(pedido);
} catch (PedidoRechazadoException e) {
    logger.error("Pedido {} rechazado. Motivo: {}. Contexto: {}",
                 e.getPedidoId(), e.getMotivo(), e.getContexto(), e);
    // La 'e' al final asegura que el stack trace se incluya en el log
}
```

### 5.8.4 No lances excepciones desde `finally`

Lanzar una excepción dentro del bloque `finally` puede sobrescribir la excepción original:

```java
// Mal: throw en finally oculta la excepción original
try {
    throw new RuntimeException("Error original");
} finally {
    throw new RuntimeException("Error en finally"); // ¡borra la original!
}

// Mejor: captura la excepción en el finally
} finally {
    try {
        recurso.close();
    } catch (Exception e) {
        logger.warn("Error al cerrar recurso", e);
    }
}
```

### 5.8.5 Usa tipos de excepción específicos

Lanza y captura los tipos más concretos posibles:

```java
// Mal: genérico
throw new RuntimeException("Algo salió mal");

// Bien: específico
throw new ProductoNoEncontradoException("SKU-12345");
```

### 5.8.6 Incluye mensajes descriptivos

El mensaje debe ayudar a entender qué ocurrió y por qué, incluyendo los valores relevantes:

```java
// Mal
throw new IllegalArgumentException("Argumento inválido");

// Bien
throw new IllegalArgumentException(
    "La cantidad no puede ser negativa. Valor recibido: " + cantidad
);
```

### 5.8.7 Encadena excepciones (exception chaining)

Preserva siempre la causa original al traducir excepciones:

```java
// Mal: la causa original se pierde
public void procesarArchivo(String ruta) throws ProcesamientoException {
    try {
        String contenido = Files.readString(Path.of(ruta));
    } catch (IOException e) {
        throw new ProcesamientoException("Error al leer el archivo"); // ¡causa perdida!
    }
}

// Bien: la causa original se preserva
public void procesarArchivo(String ruta) throws ProcesamientoException {
    try {
        String contenido = Files.readString(Path.of(ruta));
    } catch (IOException e) {
        throw new ProcesamientoException("Error al leer el archivo: " + ruta, e);
    }
}
```

Al pasar la excepción original como causa, el stack trace incluye ambas trazas:

```
ProcesamientoException: Error al leer el archivo: /ruta/config.xml
    at MiServicio.procesarArchivo(MiServicio.java:42)
    at MiServicio.main(MiServicio.java:10)
Caused by: java.nio.file.NoSuchFileException: /ruta/config.xml
    at java.base/sun.nio.fs.UnixException.translateToIOException(...)
    at ...
```

### 5.8.8 Otras buenas prácticas

- **No uses excepciones para control de flujo normal**: las excepciones son caras. Para situaciones esperadas, usa `Optional<T>` o el patrón `Result<T, E>`.
- **Lanza pronto, captura tarde**: lanza tan pronto como detectes el problema; captura en el nivel donde puedas tomar una decisión útil.
- **Documenta las excepciones** con `@throws` en el Javadoc.
- **No declares `throws Exception`** genérico en métodos públicos. Usa tipos concretos.
- **Favorece unchecked sobre checked** en APIs modernas, especialmente con lambdas y streams.
- **Nunca retornes null en lugar de lanzar una excepción**: desplaza el problema al llamador, que puede olvidar comprobar null.
- **Considera el contexto de concurrencia**: las excepciones en hilos secundarios no se propagan al principal (ver sección 5.13).

---

## 5.9 Patrones de manejo de excepciones

Los patrones de manejo de excepciones son soluciones probadas a problemas recurrentes en el diseño de software robusto.

### 5.9.1 Circuit Breaker

**Problema**: Un servicio remoto se cae y cada llamada falla tras un timeout de 30 segundos. Si cientos de peticiones llegan por segundo, los hilos se bloquean y el sistema colapsa.

**Solución**: El Circuit Breaker monitoriza los fallos consecutivos. Tras un umbral, «abre el circuito» y rechaza las llamadas inmediatamente (sin esperar al timeout). Periódicamente permite una «llamada de prueba» para verificar si el servicio se ha recuperado.

**Estados del Circuit Breaker**:

```
     ┌─────────────┐
     │   CERRADO    │ ← Funcionamiento normal. Las llamadas pasan.
     └──────┬──────┘
            │ fallos > umbral
            ▼
     ┌─────────────┐
     │   ABIERTO    │ ← Rechazo inmediato. No se llama al servicio.
     └──────┬──────┘
            │ timeout expira
            ▼
     ┌─────────────┐
     │ SEMI-ABIERTO │ ← Se permite una llamada de prueba.
     └──────┬──────┘
        ╱         ╲
   éxito           fallo
     ╱               ╲
    ▼                 ▼
┌─────────┐     ┌─────────┐
│ CERRADO │     │ ABIERTO │
└─────────┘     └─────────┘
```

**Implementación en Java**:

```java
import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.Supplier;

/**
 * Implementación simple del patrón Circuit Breaker.
 *
 * Estados:
 * - CLOSED: las llamadas pasan normalmente
 * - OPEN: las llamadas se rechazan inmediatamente
 * - HALF_OPEN: se permite una llamada de prueba
 */
public class CircuitBreaker {

    private enum Estado {
        CLOSED, OPEN, HALF_OPEN
    }

    private final String nombre;
    private final int umbralFallos;
    private final Duration timeoutAbierto;

    private final AtomicReference<Estado> estado =
            new AtomicReference<>(Estado.CLOSED);
    private final AtomicInteger contadorFallos = new AtomicInteger(0);
    private volatile Instant abiertoDesde;

    public CircuitBreaker(String nombre, int umbralFallos, Duration timeoutAbierto) {
        this.nombre = nombre;
        this.umbralFallos = umbralFallos;     // ej: 5 fallos consecutivos
        this.timeoutAbierto = timeoutAbierto; // ej: 30 segundos
    }

    /**
     * Ejecuta una operación protegida por el Circuit Breaker.
     * Si el circuito está abierto, lanza CircuitBreakerOpenException.
     */
    public <T> T ejecutar(Supplier<T> operacion) throws Exception {
        Estado estadoActual = estado.get();

        if (estadoActual == Estado.OPEN) {
            if (abiertoDesde != null &&
                Duration.between(abiertoDesde, Instant.now())
                    .compareTo(timeoutAbierto) > 0) {

                if (estado.compareAndSet(Estado.OPEN, Estado.HALF_OPEN)) {
                    System.out.printf("[%s] Circuito: OPEN → HALF_OPEN%n", nombre);
                    estadoActual = Estado.HALF_OPEN;
                } else {
                    estadoActual = estado.get();
                }
            }

            if (estadoActual == Estado.OPEN) {
                throw new CircuitBreakerOpenException(
                    String.format("Circuito [%s] ABIERTO. Operación rechazada.", nombre)
                );
            }
        }

        try {
            T resultado = operacion.get();
            onExito();
            return resultado;
        } catch (Exception e) {
            onFallo();
            throw e;
        }
    }

    private void onExito() {
        contadorFallos.set(0);
        estado.set(Estado.CLOSED);
        System.out.printf("[%s] Circuito: → CLOSED (éxito)%n", nombre);
    }

    private void onFallo() {
        int fallos = contadorFallos.incrementAndGet();
        System.out.printf("[%s] Fallo %d/%d%n", nombre, fallos, umbralFallos);

        if (fallos >= umbralFallos) {
            if (estado.compareAndSet(Estado.CLOSED, Estado.OPEN) ||
                estado.compareAndSet(Estado.HALF_OPEN, Estado.OPEN)) {
                abiertoDesde = Instant.now();
                System.out.printf("[%s] Circuito: → OPEN (abierto por %s)%n",
                        nombre, timeoutAbierto);
            }
        }
    }

    // ─── Excepción ───
    public static class CircuitBreakerOpenException extends RuntimeException {
        public CircuitBreakerOpenException(String mensaje) {
            super(mensaje);
        }
    }

    // ─── Demo ───
    public static void main(String[] args) throws InterruptedException {
        CircuitBreaker cb = new CircuitBreaker(
                "ServicioPagos", 3, Duration.ofSeconds(5));

        for (int i = 1; i <= 10; i++) {
            try {
                String resultado = cb.ejecutar(() -> {
                    throw new RuntimeException("Servicio no disponible");
                });
                System.out.println("Resultado: " + resultado);
            } catch (CircuitBreakerOpenException e) {
                System.out.println(">>> " + e.getMessage());
            } catch (Exception e) {
                System.out.println("Error: " + e.getMessage());
            }
            Thread.sleep(500);
        }
    }
}
```

Salida típica:

```
[ServicioPagos] Fallo 1/3
Error: Servicio no disponible
[ServicioPagos] Fallo 2/3
Error: Servicio no disponible
[ServicioPagos] Fallo 3/3
[ServicioPagos] Circuito: → OPEN (abierto por PT5S)
>>> Circuito [ServicioPagos] ABIERTO. Operación rechazada.
>>> Circuito [ServicioPagos] ABIERTO. Operación rechazada.
... (tras 5 segundos) ...
[ServicioPagos] Circuito: OPEN → HALF_OPEN
Error: Servicio no disponible       ← llamada de prueba falla
[ServicioPagos] Fallo 1/3
[ServicioPagos] Circuito: → OPEN (abierto por PT5S)
```

En producción, se recomienda usar **Resilience4j**, **Spring Cloud Circuit Breaker** o **Failsafe**, que implementan este patrón con métricas, configuración dinámica y estados adicionales.

Ejemplo con Resilience4j:

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .slowCallRateThreshold(50)
    .slowCallDurationThreshold(Duration.ofSeconds(2))
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .slidingWindowSize(10)
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("miServicio", config);

Supplier<String> operacionDecorada = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> llamarServicioExterno());

String resultado = Try.ofSupplier(operacionDecorada)
    .recover(throwable -> "Fallback: valor por defecto")
    .get();
```

### 5.9.2 Retry Pattern

**Problema**: Una operación de red falla esporádicamente por un pico de carga o timeout momentáneo. Reintentar inmediatamente suele tener éxito.

**Solución**: Reintentar la operación un número limitado de veces, con espera creciente entre intentos (**backoff exponencial**).

```
Intento 1: falla → esperar 100 ms
Intento 2: falla → esperar 200 ms
Intento 3: falla → esperar 400 ms
Intento 4: éxito ✓
```

**Implementación**:

```java
import java.time.Duration;
import java.util.function.Supplier;

/**
 * Implementa el patrón Retry con backoff exponencial y jitter.
 */
public class RetryTemplate {

    private final int maxIntentos;
    private final Duration backoffInicial;
    private final double multiplicadorBackoff;
    private final Duration backoffMaximo;

    public RetryTemplate(int maxIntentos, Duration backoffInicial) {
        this(maxIntentos, backoffInicial, 2.0, Duration.ofSeconds(30));
    }

    public RetryTemplate(int maxIntentos, Duration backoffInicial,
                         double multiplicadorBackoff, Duration backoffMaximo) {
        this.maxIntentos = maxIntentos;
        this.backoffInicial = backoffInicial;
        this.multiplicadorBackoff = multiplicadorBackoff;
        this.backoffMaximo = backoffMaximo;
    }

    /**
     * Ejecuta una operación con reintentos. Retorna el resultado o lanza
     * RetryExhaustedException si se agotan los intentos.
     */
    public <T> T ejecutar(Supplier<T> operacion) throws RetryExhaustedException {
        Exception ultimaExcepcion = null;
        Duration backoff = backoffInicial;

        for (int intento = 1; intento <= maxIntentos; intento++) {
            try {
                return operacion.get();
            } catch (Exception e) {
                ultimaExcepcion = e;

                if (intento == maxIntentos) {
                    throw new RetryExhaustedException(
                        String.format("Agotados %d intentos. Último error: %s",
                                      maxIntentos, e.getMessage()), e);
                }

                System.out.printf("  Intento %d fallido: %s. Reintentando en %s...%n",
                                  intento, e.getMessage(), backoff);

                try {
                    Thread.sleep(backoff.toMillis());
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RetryExhaustedException("Reintento interrumpido", ie);
                }

                // Calcular siguiente backoff con jitter (+/- 25% aleatorio)
                long backoffMs = backoff.toMillis();
                long jitter = (long) (backoffMs * (Math.random() * 0.5 - 0.25));
                long siguienteBackoff = Math.min(
                    (long) (backoffMs * multiplicadorBackoff) + jitter,
                    backoffMaximo.toMillis()
                );
                backoff = Duration.ofMillis(siguienteBackoff);
            }
        }

        throw new RetryExhaustedException("Error inesperado", ultimaExcepcion);
    }

    // ─── Excepción ───
    public static class RetryExhaustedException extends RuntimeException {
        public RetryExhaustedException(String mensaje, Throwable causa) {
            super(mensaje, causa);
        }
    }

    // ─── Demo ───
    public static void main(String[] args) {
        RetryTemplate retry = new RetryTemplate(
            5,
            Duration.ofMillis(100),     // empezar con 100ms
            2.0,                        // duplicar cada vez
            Duration.ofSeconds(10)      // máximo 10s
        );

        AtomicInteger contador = new AtomicInteger(0);

        try {
            String resultado = retry.ejecutar(() -> {
                int intento = contador.incrementAndGet();
                if (intento <= 3) {
                    throw new RuntimeException("Servicio temporalmente no disponible");
                }
                return "Éxito en el intento " + intento;
            });
            System.out.println("Resultado: " + resultado);
        } catch (RetryExhaustedException e) {
            System.err.println("No se pudo completar: " + e.getMessage());
        }
    }
}
```

Salida:

```
  Intento 1 fallido: Servicio temporalmente no disponible. Reintentando en PT0.1S...
  Intento 2 fallido: Servicio temporalmente no disponible. Reintentando en PT0.2S...
  Intento 3 fallido: Servicio temporalmente no disponible. Reintentando en PT0.4S...
Resultado: Éxito en el intento 4
```

**Combinar Circuit Breaker + Retry** es una práctica común en microservicios. Resilience4j y Spring Retry ofrecen ambas funcionalidades integradas.

### 5.9.3 Patrón de traducción (Exception Translation)

**Problema**: Las capas bajas (acceso a datos, HTTP, red) lanzan excepciones técnicas (`SQLException`, `IOException`, `HttpClientErrorException`) que no tienen significado para las capas de negocio.

**Solución**: Cada capa traduce las excepciones que recibe en excepciones significativas para la capa superior.

```
Capa de datos:       SQLException        → DataAccessException
Capa de servicio:    DataAccessException → UsuarioNoEncontradoException
Capa de presentación: UsuarioNoEncontradoException → HTTP 404
```

**Implementación**:

```java
// ─── Jerarquía de excepciones de dominio ───

public abstract class DominioException extends RuntimeException {
    private final String codigoError;

    public DominioException(String codigoError, String mensaje) {
        super(mensaje);
        this.codigoError = codigoError;
    }

    public DominioException(String codigoError, String mensaje, Throwable causa) {
        super(mensaje, causa);
        this.codigoError = codigoError;
    }

    public String getCodigoError() { return codigoError; }
}

public class UsuarioNoEncontradoException extends DominioException {
    private final Long usuarioId;

    public UsuarioNoEncontradoException(Long usuarioId) {
        super("USUARIO_NO_ENCONTRADO",
              "Usuario con ID " + usuarioId + " no encontrado");
        this.usuarioId = usuarioId;
    }

    public Long getUsuarioId() { return usuarioId; }
}

public class SaldoInsuficienteException extends DominioException {
    public SaldoInsuficienteException(
            String cuenta, BigDecimal saldo, BigDecimal monto) {
        super("SALDO_INSUFICIENTE",
              String.format("Saldo insuficiente en cuenta %s (disponible: %s, requerido: %s)",
                            cuenta, saldo, monto));
    }
}

// ─── Capa de repositorio: traduce SQLException a DataAccessException ───

public class UsuarioRepository {

    private final DataSource dataSource;

    public UsuarioRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Optional<Usuario> findById(Long id) {
        String sql = "SELECT id, nombre, email FROM usuarios WHERE id = ?";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setLong(1, id);
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(new Usuario(
                        rs.getLong("id"),
                        rs.getString("nombre"),
                        rs.getString("email")
                    ));
                }
                return Optional.empty();
            }
        } catch (SQLException e) {
            // Traducción: SQLException → DataAccessException
            throw new DataAccessException(
                "Error al buscar usuario por ID: " + id, e
            );
        }
    }
}

// ─── Capa de servicio: traduce DataAccessException a DominioException ───

public class UsuarioService {

    private final UsuarioRepository repository;

    public UsuarioService(UsuarioRepository repository) {
        this.repository = repository;
    }

    public Usuario obtenerUsuario(Long id) {
        try {
            return repository.findById(id)
                .orElseThrow(() -> new UsuarioNoEncontradoException(id));
        } catch (DataAccessException e) {
            // Traducción: error técnico → error de dominio
            throw new DominioException(
                "ERROR_INTERNO",
                "Error interno al recuperar usuario " + id,
                e
            );
        }
    }
}
```

### 5.9.4 Exception Shielding

**Problema**: Si una API pública expone los detalles internos en los mensajes de error (nombres de tablas, queries SQL, stack traces del framework), un atacante puede obtener información valiosa para explotar vulnerabilidades.

**Solución**: Las APIs públicas nunca deben exponer los detalles de la excepción original. Deben devolver mensajes genéricos y seguros, registrando los detalles solo internamente.

```java
@RestController
public class UsuarioController {

    private static final Logger logger =
        LoggerFactory.getLogger(UsuarioController.class);

    @GetMapping("/api/usuarios/{id}")
    public ResponseEntity<?> obtenerUsuario(@PathVariable Long id) {
        try {
            Usuario usuario = usuarioService.obtenerUsuario(id);
            return ResponseEntity.ok(usuario);
        } catch (UsuarioNoEncontradoException e) {
            // Seguro: mensaje controlado, no expone estructura interna
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(Map.of("error", "Usuario no encontrado"));
        } catch (DominioException e) {
            // Seguro: no exponemos detalles internos
            logger.error("Error de dominio al obtener usuario {}: {}",
                         id, e.getMessage(), e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error", "Error interno del servidor"));
        } catch (Exception e) {
            // Muy importante: nunca devolver e.getMessage() al cliente
            logger.error("Error inesperado al obtener usuario {}", id, e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error",
                    "Error inesperado. Contacte al administrador."));
        }
    }
}
```

### 5.9.5 Global Exception Handler

En lugar de esparcir `try-catch` por todos los controladores, centraliza el manejo de excepciones.

**En Spring Boot** (con `@ControllerAdvice`):

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger logger =
        LoggerFactory.getLogger(GlobalExceptionHandler.class);

    /**
     * Maneja excepciones de dominio de tipo "recurso no encontrado".
     */
    @ExceptionHandler(UsuarioNoEncontradoException.class)
    public ResponseEntity<ErrorResponse> handleUsuarioNoEncontrado(
            UsuarioNoEncontradoException e) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", e.getMessage()));
    }

    /**
     * Maneja errores de validación (Bean Validation / Jakarta Validation).
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidacion(
            MethodArgumentNotValidException e) {

        List<String> errores = e.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .collect(Collectors.toList());

        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("VALIDATION_ERROR",
                  "Error de validación", errores));
    }

    /**
     * Maneja cualquier DominioException no capturada por handlers específicos.
     */
    @ExceptionHandler(DominioException.class)
    public ResponseEntity<ErrorResponse> handleDominioException(DominioException e) {
        logger.error("Error de dominio [{}]: {}", e.getCodigoError(), e.getMessage(), e);
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(e.getCodigoError(), e.getMessage()));
    }

    /**
     * Captura cualquier excepción no manejada (último recurso).
     * NUNCA expone el mensaje original al cliente.
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception e) {
        logger.error("Error inesperado", e);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR",
                  "Error interno del servidor. Contacte al administrador."));
    }

    // ─── DTO de respuesta de error ───
    record ErrorResponse(String codigo, String mensaje, List<String> detalles) {
        public ErrorResponse(String codigo, String mensaje) {
            this(codigo, mensaje, List.of());
        }
    }
}
```

**Para aplicaciones standalone** (con `UncaughtExceptionHandler`):

```java
public class GlobalUncaughtExceptionHandler
        implements Thread.UncaughtExceptionHandler {

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.err.printf("[ERROR] Excepción no capturada en hilo '%s': %s%n",
                          t.getName(), e.getMessage());
        e.printStackTrace(System.err);
        // En producción: registrar en sistema de monitoreo, enviar alerta, etc.
    }

    public static void main(String[] args) {
        // Registrar para todos los hilos
        Thread.setDefaultUncaughtExceptionHandler(
            new GlobalUncaughtExceptionHandler()
        );

        Thread hilo = new Thread(() -> {
            throw new RuntimeException("Error en hilo secundario");
        }, "hilo-trabajador");
        hilo.start();

        System.out.println("Esperando...");
    }
}
```

---

## 5.10 Alternativas a las excepciones: el patrón Result/Either

### 5.10.1 ¿Por qué buscar alternativas?

Las excepciones tienen limitaciones importantes:

1. **Flujo de control oculto**: `throw` es un `goto` glorificado. Dificulta razonar sobre el flujo del programa.
2. **Coste de rendimiento**: Como vimos en 5.1.2, llenar el stack trace puede ser caro para operaciones frecuentes.
3. **Incompatibilidad funcional**: Las lambdas y streams no pueden lanzar checked exceptions limpiamente.
4. **Pérdida de información de tipo**: El compilador no te dice qué excepciones unchecked puede lanzar un método.
5. **Acoplamiento**: El llamador necesita conocer los tipos de excepción específicos, creando dependencias.

**El patrón Result/Either** aborda estos problemas haciendo que el resultado de una operación sea explícitamente un valor de éxito o un error, como parte del sistema de tipos.

### 5.10.2 Concepto

En lugar de:

```java
// Enfoque tradicional: excepciones
public Usuario buscarUsuario(Long id) throws UsuarioNoEncontradoException {
    // ...
}
```

Usamos un tipo que encapsula ambos casos:

```java
// Enfoque Result: el tipo lo dice todo
public Result<Usuario, ErrorBusqueda> buscarUsuario(Long id) {
    // ...
}
```

El tipo `Result<T, E>` tiene dos variantes:
- **`Success<T, E>`**: contiene un valor de tipo `T` (operación exitosa).
- **`Failure<T, E>`**: contiene un error de tipo `E` (operación fallida).

### 5.10.3 Implementación completa de Result<T, E>

```java
import java.util.NoSuchElementException;
import java.util.Objects;
import java.util.Optional;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Predicate;
import java.util.function.Supplier;

/**
 * Representa el resultado de una operación que puede tener éxito (Success)
 * o fallar (Failure). Inspirado en Result de Rust y Either de Scala/Vavr.
 *
 * @param <T> Tipo del valor en caso de éxito
 * @param <E> Tipo del error en caso de fallo
 */
public abstract class Result<T, E> {

    // ─── Constructores de fábrica ───

    public static <T, E> Result<T, E> success(T valor) {
        return new Success<>(valor);
    }

    public static <T, E> Result<T, E> failure(E error) {
        return new Failure<>(error);
    }

    /**
     * Ejecuta un Supplier y envuelve el resultado.
     * Si lanza excepción, se captura y se convierte en Failure.
     */
    public static <T, E> Result<T, E> of(
            Supplier<T> supplier,
            Function<Exception, E> errorMapper) {
        try {
            return success(supplier.get());
        } catch (Exception e) {
            return failure(errorMapper.apply(e));
        }
    }

    // ─── Consultas ───

    public abstract boolean isSuccess();
    public abstract boolean isFailure();

    // ─── Acceso a valores ───

    public abstract T get();
    public abstract E getError();

    public T getOrElse(T valorPorDefecto) {
        return isSuccess() ? get() : valorPorDefecto;
    }

    public T getOrElseGet(Supplier<? extends T> supplier) {
        return isSuccess() ? get() : supplier.get();
    }

    public <X extends Throwable> T getOrElseThrow(
            Function<? super E, ? extends X> exceptionMapper) throws X {
        if (isSuccess()) {
            return get();
        }
        throw exceptionMapper.apply(getError());
    }

    // ─── Transformaciones ───

    /**
     * Si es Success, aplica la función al valor.
     * Si es Failure, propaga el error sin cambios.
     */
    public abstract <U> Result<U, E> map(
            Function<? super T, ? extends U> mapper);

    /**
     * Si es Failure, aplica la función al error.
     * Si es Success, propaga el valor sin cambios.
     */
    public abstract <F> Result<T, F> mapError(
            Function<? super E, ? extends F> mapper);

    /**
     * Similar a map, pero la función devuelve otro Result.
     */
    public abstract <U> Result<U, E> flatMap(
            Function<? super T, Result<U, E>> mapper);

    // ─── Efectos secundarios ───

    public Result<T, E> ifSuccess(Consumer<? super T> consumer) {
        if (isSuccess()) consumer.accept(get());
        return this;
    }

    public Result<T, E> ifFailure(Consumer<? super E> consumer) {
        if (isFailure()) consumer.accept(getError());
        return this;
    }

    public Result<T, E> peek(Consumer<? super T> onSuccess,
                              Consumer<? super E> onFailure) {
        if (isSuccess()) onSuccess.accept(get());
        else onFailure.accept(getError());
        return this;
    }

    // ─── Filtrado ───

    public Result<T, E> filter(Predicate<? super T> predicado,
                                E errorSiNoCumple) {
        if (isSuccess() && !predicado.test(get())) {
            return failure(errorSiNoCumple);
        }
        return this;
    }

    // ─── Conversión a Optional ───

    public Optional<T> toOptional() {
        return isSuccess() ? Optional.of(get()) : Optional.empty();
    }

    // ─── Clases internas ───

    private static final class Success<T, E> extends Result<T, E> {
        private final T valor;

        Success(T valor) {
            this.valor = Objects.requireNonNull(valor,
                "Success no puede contener null");
        }

        @Override public boolean isSuccess() { return true; }
        @Override public boolean isFailure() { return false; }
        @Override public T get() { return valor; }

        @Override
        public E getError() {
            throw new NoSuchElementException("Success no contiene error");
        }

        @Override
        public <U> Result<U, E> map(
                Function<? super T, ? extends U> mapper) {
            return success(mapper.apply(valor));
        }

        @Override
        public <F> Result<T, F> mapError(
                Function<? super E, ? extends F> mapper) {
            @SuppressWarnings("unchecked")
            Result<T, F> result = (Result<T, F>) this;
            return result;
        }

        @Override
        public <U> Result<U, E> flatMap(
                Function<? super T, Result<U, E>> mapper) {
            return mapper.apply(valor);
        }

        @Override
        public String toString() {
            return "Success[" + valor + "]";
        }
    }

    private static final class Failure<T, E> extends Result<T, E> {
        private final E error;

        Failure(E error) {
            this.error = Objects.requireNonNull(error,
                "Failure no puede contener null");
        }

        @Override public boolean isSuccess() { return false; }
        @Override public boolean isFailure() { return true; }

        @Override
        public T get() {
            throw new NoSuchElementException(
                "Failure no contiene valor. Error: " + error);
        }

        @Override public E getError() { return error; }

        @Override
        public <U> Result<U, E> map(
                Function<? super T, ? extends U> mapper) {
            @SuppressWarnings("unchecked")
            Result<U, E> result = (Result<U, E>) this;
            return result;
        }

        @Override
        public <F> Result<T, F> mapError(
                Function<? super E, ? extends F> mapper) {
            return failure(mapper.apply(error));
        }

        @Override
        public <U> Result<U, E> flatMap(
                Function<? super T, Result<U, E>> mapper) {
            @SuppressWarnings("unchecked")
            Result<U, E> result = (Result<U, E>) this;
            return result;
        }

        @Override
        public String toString() {
            return "Failure[" + error + "]";
        }
    }
}
```

### 5.10.4 Comparación con Optional<T>

`Optional<T>` es insuficiente para representar operaciones que pueden fallar porque **pierde la información del error**:

```java
// Optional: sabes que falló, pero NO sabes por qué
public Optional<Usuario> buscarUsuario(Long id) {
    try {
        return Optional.of(repository.findById(id));
    } catch (SQLException e) {
        return Optional.empty(); // ← ¿Por qué falló? No lo sabes.
    }
}

// Result: sabes que falló Y por qué falló
public Result<Usuario, ErrorBusqueda> buscarUsuario(Long id) {
    try {
        return Result.success(repository.findById(id));
    } catch (SQLException e) {
        return Result.failure(new ErrorBusqueda("DB_ERROR", e.getMessage()));
    }
}
```

| Característica | `Optional<T>` | `Result<T, E>` |
|---|---|---|
| Representa éxito con valor | `Optional.of(valor)` | `Success(valor)` |
| Representa ausencia/fallo | `Optional.empty()` (sin razón) | `Failure(error)` (con razón) |
| Información del error | ❌ No disponible | ✅ Tipada y accesible |
| Transformaciones | `map`, `flatMap`, `filter` | `map`, `flatMap`, `mapError`, `filter` |
| Conversión a excepción | `orElseThrow(...)` | `getOrElseThrow(...)` |
| Efectos secundarios | `ifPresent(...)` | `ifSuccess(...)`, `ifFailure(...)` |
| Adecuado para | "Puede que no haya valor" | "La operación puede fallar por X razón" |

### 5.10.5 Ejemplo real: servicio de pago con Result

```java
import java.math.BigDecimal;
import java.util.*;

/**
 * Servicio de pago que usa Result en lugar de excepciones.
 * Cada operación devuelve explícitamente éxito o fallo con información detallada.
 */
public class ServicioPagos {

    // ─── Tipos de error (usando sealed interfaces de Java 17+) ───

    public sealed interface ErrorPago
            permits SaldoInsuficiente, CuentaBloqueada, LimiteExcedido,
                    CuentaNoEncontrada, ErrorConexion, FondosCongelados {
        String mensaje();
    }

    public record SaldoInsuficiente(BigDecimal disponible, BigDecimal requerido)
            implements ErrorPago {
        public String mensaje() {
            return String.format("Saldo insuficiente: disponible=%s, requerido=%s",
                                 disponible, requerido);
        }
    }

    public record CuentaBloqueada(String cuentaId, String motivo)
            implements ErrorPago {
        public String mensaje() {
            return "Cuenta " + cuentaId + " bloqueada: " + motivo;
        }
    }

    public record LimiteExcedido(BigDecimal limiteDiario, BigDecimal acumulado)
            implements ErrorPago {
        public String mensaje() {
            return String.format("Límite diario excedido: límite=%s, acumulado=%s",
                                 limiteDiario, acumulado);
        }
    }

    public record CuentaNoEncontrada(String cuentaId) implements ErrorPago {
        public String mensaje() {
            return "Cuenta no encontrada: " + cuentaId;
        }
    }

    public record ErrorConexion(String detalle) implements ErrorPago {
        public String mensaje() { return "Error de conexión: " + detalle; }
    }

    public record FondosCongelados(BigDecimal montoCongelado) implements ErrorPago {
        public String mensaje() {
            return "Fondos congelados por importe de " + montoCongelado;
        }
    }

    // ─── Tipo de éxito ───

    public record TransaccionExitosa(
            UUID transaccionId,
            String cuentaOrigen,
            String cuentaDestino,
            BigDecimal monto) { }

    // ─── Datos de ejemplo ───

    private final Map<String, Cuenta> cuentas = new HashMap<>();

    public ServicioPagos() {
        cuentas.put("ES-001", new Cuenta("ES-001",
            new BigDecimal("1000.00"), false, BigDecimal.ZERO));
        cuentas.put("ES-002", new Cuenta("ES-002",
            new BigDecimal("50.00"), true, BigDecimal.ZERO));
        cuentas.put("ES-003", new Cuenta("ES-003",
            new BigDecimal("5000.00"), false, new BigDecimal("4800.00")));
    }

    private record Cuenta(String id, BigDecimal saldo,
                          boolean bloqueada, BigDecimal acumuladoDiario) {}

    // ─── Método principal con Result ───

    public Result<TransaccionExitosa, ErrorPago> transferir(
            String cuentaOrigen, String cuentaDestino, BigDecimal monto) {

        if (monto.compareTo(BigDecimal.ZERO) <= 0) {
            return Result.failure(new ErrorPago() {
                public String mensaje() { return "El monto debe ser positivo"; }
            });
        }

        // Usar flatMap para encadenar validaciones
        return buscarCuenta(cuentaOrigen)
            .flatMap(origen ->
                buscarCuenta(cuentaDestino)
                    .flatMap(destino ->
                        validarTransferencia(origen, monto)
                            .map(ignored -> ejecutarTransferencia(
                                origen, destino, monto))
                    )
            );
    }

    private Result<Cuenta, ErrorPago> buscarCuenta(String id) {
        return Result.of(
            () -> {
                Cuenta cuenta = cuentas.get(id);
                if (cuenta == null) throw new IllegalArgumentException(id);
                return cuenta;
            },
            ex -> new CuentaNoEncontrada(id)
        );
    }

    private Result<Void, ErrorPago> validarTransferencia(
            Cuenta cuenta, BigDecimal monto) {

        if (cuenta.bloqueada) {
            return Result.failure(
                new CuentaBloqueada(cuenta.id, "Cuenta bloqueada por seguridad"));
        }
        if (cuenta.saldo.compareTo(monto) < 0) {
            return Result.failure(
                new SaldoInsuficiente(cuenta.saldo, monto));
        }
        BigDecimal limiteDiario = new BigDecimal("5000.00");
        BigDecimal nuevoAcumulado = cuenta.acumuladoDiario.add(monto);
        if (nuevoAcumulado.compareTo(limiteDiario) > 0) {
            return Result.failure(
                new LimiteExcedido(limiteDiario, cuenta.acumuladoDiario));
        }

        return Result.success(null);
    }

    private TransaccionExitosa ejecutarTransferencia(
            Cuenta origen, Cuenta destino, BigDecimal monto) {
        UUID transaccionId = UUID.randomUUID();
        System.out.printf("Transfiriendo %s de %s a %s (ID: %s)%n",
                          monto, origen.id, destino.id, transaccionId);
        return new TransaccionExitosa(transaccionId, origen.id, destino.id, monto);
    }

    // ─── Demo ───

    public static void main(String[] args) {
        ServicioPagos servicio = new ServicioPagos();

        // Caso 1: Transferencia exitosa
        System.out.println("=== Caso 1: Transferencia exitosa ===");
        servicio.transferir("ES-001", "ES-002", new BigDecimal("200"))
            .ifSuccess(tx -> System.out.println("✓ Éxito: " + tx))
            .ifFailure(err -> System.out.println("✗ Fallo: " + err.mensaje()));

        // Caso 2: Saldo insuficiente
        System.out.println("\n=== Caso 2: Saldo insuficiente ===");
        servicio.transferir("ES-002", "ES-001", new BigDecimal("1000"))
            .ifSuccess(tx -> System.out.println("✓ Éxito: " + tx))
            .ifFailure(err -> System.out.println("✗ Fallo: " + err.mensaje()));

        // Caso 3: Cuenta bloqueada
        System.out.println("\n=== Caso 3: Cuenta bloqueada ===");
        servicio.transferir("ES-002", "ES-001", new BigDecimal("10"))
            .ifSuccess(tx -> System.out.println("✓ Éxito: " + tx))
            .ifFailure(err -> System.out.println("✗ Fallo: " + err.mensaje()));

        // Caso 4: Transformar resultado a respuesta HTTP
        System.out.println("\n=== Caso 4: Mapear a HTTP status ===");
        Result<TransaccionExitosa, ErrorPago> resultado =
            servicio.transferir("ES-999", "ES-001", new BigDecimal("100"));

        if (resultado.isSuccess()) {
            System.out.println("HTTP 200: " + resultado.get());
        } else {
            ErrorPago error = resultado.getError();
            int httpStatus = error instanceof CuentaNoEncontrada ? 404
                : error instanceof SaldoInsuficiente ? 422
                : error instanceof CuentaBloqueada ? 423
                : error instanceof LimiteExcedido ? 429
                : 500;
            System.out.println("HTTP " + httpStatus + ": " + error.mensaje());
        }

        // Caso 5: Encadenamiento funcional con map y getOrElse
        System.out.println("\n=== Caso 5: Encadenamiento funcional ===");
        String resumen = servicio.transferir("ES-001", "ES-003",
                                             new BigDecimal("300"))
            .map(tx -> String.format("Transferencia %s: %s de %s a %s",
                   tx.transaccionId().toString().substring(0, 8),
                   tx.monto(), tx.cuentaOrigen(), tx.cuentaDestino()))
            .getOrElse("No se pudo completar la transferencia");
        System.out.println(resumen);

        // Caso 6: Re-lanzar como excepción cuando sea necesario
        System.out.println("\n=== Caso 6: Convertir a excepción ===");
        try {
            servicio.transferir("ES-002", "ES-001", new BigDecimal("1000"))
                .getOrElseThrow(err -> new RuntimeException(err.mensaje()));
        } catch (RuntimeException e) {
            System.out.println("Excepción capturada: " + e.getMessage());
        }
    }
}
```

### 5.10.6 Cuándo usar Result vs cuándo usar excepciones

| Usar **Result<T, E>** cuando... | Usar **Excepciones** cuando... |
|---|---|
| El fallo es una situación **esperada** y frecuente | El fallo es verdaderamente **excepcional** |
| El llamador debe manejar el error explícitamente | El error debe propagarse varias capas hacia arriba |
| Estás en un contexto funcional (streams, lambdas) | Estás en un contexto imperativo tradicional |
| Los errores son de dominio con significado para el negocio | El error es un bug de programación (NPE, cast incorrecto) |
| Quieres forzar al llamador a considerar el caso de error | No hay nada útil que el llamador pueda hacer |
| Necesitas componer múltiples operaciones que pueden fallar | Una sola operación puede fallar por razones muy distintas |

**En la práctica**, un enfoque híbrido funciona bien:
- Usa `Result` para errores de dominio esperados (validación, reglas de negocio).
- Usa excepciones unchecked para bugs de programación y fallos de infraestructura.
- Las excepciones checked son raramente necesarias en código moderno con Result.

---

## 5.11 Excepciones en programación funcional y streams

### 5.11.1 El problema: lambdas y checked exceptions

Las interfaces funcionales de Java (`Function<T,R>`, `Consumer<T>`, `Predicate<T>`, `Supplier<T>`) no declaran excepciones comprobadas. Esto crea fricción al usar código que lanza checked exceptions dentro de lambdas y streams:

```java
// ESTO NO COMPILA: IOException es checked y Function no la declara
List<String> lineas = Files.lines(Path.of("datos.txt"))
    .map(linea -> procesarLinea(linea))  // procesarLinea lanza IOException
    .collect(Collectors.toList());
```

```java
// Error de compilación: Unhandled exception: java.io.IOException
Function<String, String> procesador = linea -> {
    if (linea.startsWith("#")) {
        throw new IOException("Comentario inesperado"); // ¡No compila!
    }
    return linea.toUpperCase();
};
```

### 5.11.2 Solución 1: Wrapper que captura y relanza como RuntimeException

```java
/**
 * Interfaces funcionales que permiten lanzar excepciones checked.
 */
@FunctionalInterface
public interface ThrowingFunction<T, R, E extends Exception> {
    R apply(T t) throws E;

    static <T, R> Function<T, R> wrap(
            ThrowingFunction<T, R, Exception> f) {
        return t -> {
            try {
                return f.apply(t);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }

    static <T, R, E extends Exception> Function<T, R> wrap(
            ThrowingFunction<T, R, E> f,
            Function<Exception, ? extends RuntimeException> exceptionMapper) {
        return t -> {
            try {
                return f.apply(t);
            } catch (Exception e) {
                throw exceptionMapper.apply(e);
            }
        };
    }
}

@FunctionalInterface
public interface ThrowingConsumer<T, E extends Exception> {
    void accept(T t) throws E;

    static <T> Consumer<T> wrap(ThrowingConsumer<T, Exception> c) {
        return t -> {
            try {
                c.accept(t);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}

@FunctionalInterface
public interface ThrowingSupplier<T, E extends Exception> {
    T get() throws E;

    static <T> Supplier<T> wrap(ThrowingSupplier<T, Exception> s) {
        return () -> {
            try {
                return s.get();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}

@FunctionalInterface
public interface ThrowingRunnable<E extends Exception> {
    void run() throws E;

    static Runnable wrap(ThrowingRunnable<Exception> r) {
        return () -> {
            try {
                r.run();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}
```

**Uso con Streams**:

```java
// Ahora compila y funciona
List<String> lineas = Files.lines(Path.of("datos.txt"))
    .map(ThrowingFunction.wrap(linea -> {
        if (linea.startsWith("#")) {
            throw new IOException("Comentario inesperado");
        }
        return linea.toUpperCase();
    }))
    .collect(Collectors.toList());
```

### 5.11.3 Solución 2: @SneakyThrows de Lombok

Lombok ofrece la anotación `@SneakyThrows` que permite lanzar checked exceptions sin declararlas:

```java
import lombok.SneakyThrows;

public class ProcesadorArchivos {

    @SneakyThrows
    public String procesarLinea(String linea) {
        if (linea == null || linea.isEmpty()) {
            throw new IOException("Línea vacía"); // checked, pero compila
        }
        return linea.toUpperCase();
    }

    // Equivalente manual a lo que genera @SneakyThrows:
    public String procesarLineaManual(String linea) {
        try {
            if (linea == null || linea.isEmpty()) {
                throw new IOException("Línea vacía");
            }
            return linea.toUpperCase();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

**Uso en streams con @SneakyThrows** (se requiere un método separado para la anotación):

```java
public class Servicio {

    @SneakyThrows
    private String procesar(String linea) {
        if (linea.contains("ERROR")) {
            throw new SQLException("Línea con error: " + linea);
        }
        return linea.trim();
    }

    public List<String> procesarArchivo(String ruta) throws IOException {
        return Files.lines(Path.of(ruta))
            .map(this::procesar) // referencia a método con @SneakyThrows
            .collect(Collectors.toList());
    }
}
```

### 5.11.4 Solución 3: Stream con Result (elegante)

Combinando el patrón Result con streams, obtenemos un pipeline que maneja errores sin perder datos:

```java
/**
 * Procesa líneas de un archivo, capturando errores por línea
 * sin detener todo el proceso.
 */
public static void procesarArchivoConResultados(String ruta) throws IOException {

    List<Result<String, String>> resultados = Files.lines(Path.of(ruta))
        .map(linea -> Result.of(
            () -> procesarLinea(linea),
            ex -> "Error en línea: " + ex.getMessage()
        ))
        .collect(Collectors.toList());

    // Separar éxitos y fallos
    List<String> lineasProcesadas = resultados.stream()
        .filter(Result::isSuccess)
        .map(Result::get)
        .collect(Collectors.toList());

    List<String> errores = resultados.stream()
        .filter(Result::isFailure)
        .map(Result::getError)
        .collect(Collectors.toList());

    System.out.println("Procesadas: " + lineasProcesadas.size() + " líneas");
    System.out.println("Errores: " + errores.size() + " líneas");
    errores.forEach(e -> System.out.println("  - " + e));
    lineasProcesadas.forEach(System.out::println);
}

private static String procesarLinea(String linea) throws Exception {
    if (linea.startsWith("#")) return linea;
    if (linea.length() < 3)
        throw new Exception("Línea demasiado corta: '" + linea + "'");
    return linea.toUpperCase();
}
```

### 5.11.5 Try monádico (Vavr)

La biblioteca **Vavr** (antes Javaslang) proporciona `Try<T>` como alternativa monádica a `try-catch`:

```java
import io.vavr.control.Try;

// Encapsula una operación que puede fallar
Try<Integer> resultado = Try.of(() -> Integer.parseInt("123"));

// Patrón: success / failure
resultado.onSuccess(valor -> System.out.println("Éxito: " + valor))
         .onFailure(error -> System.err.println("Error: " + error.getMessage()));

// Recuperación con valor por defecto
Integer valor = Try.of(() -> Integer.parseInt("abc"))
    .recover(NumberFormatException.class, 0)
    .getOrElse(-1);

// Composición monádica
Try<Integer> compuesto = Try.of(() -> Integer.parseInt("100"))
    .map(x -> x * 2)
    .filter(x -> x > 50, () -> new IllegalArgumentException("Muy pequeño"))
    .flatMap(x -> Try.of(() -> x / 2));

// Conversión a Optional o excepción
Optional<Integer> opt = resultado.toJavaOptional();
Integer forzado = resultado.getOrElseThrow(e -> new RuntimeException(e));

// Reintento
Try<String> conReintento = Try.of(() -> llamarServicioExterno())
    .recoverWith(IOException.class,
        () -> Try.of(() -> llamarServicioExterno()));

// Try con recursos (equivalente a try-with-resources)
Try<String> contenido = Try.withResources(() -> new FileReader("datos.txt"))
    .of(reader -> new BufferedReader(reader).readLine());
```

### 5.11.6 Ejemplo completo: pipeline de procesamiento con manejo de errores

```java
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.*;

/**
 * Demo: procesamiento de datos con streams y manejo de errores elegante.
 */
public class PipelineFuncional {

    public static void main(String[] args) throws IOException {

        Path archivo = Path.of("transacciones.csv");
        // Formato: fecha,monto,descripcion

        // ── Pipeline con Result ──
        List<Result<Transaccion, ErrorLinea>> resultados = Files.lines(archivo)
            .skip(1) // saltar cabecera
            .filter(linea -> !linea.isBlank())
            .map(linea -> parsearTransaccion(linea))
            .collect(Collectors.toList());

        // ── Reporte ──
        List<Transaccion> exitosas = resultados.stream()
            .filter(Result::isSuccess)
            .map(Result::get)
            .collect(Collectors.toList());

        List<ErrorLinea> fallidas = resultados.stream()
            .filter(Result::isFailure)
            .map(Result::getError)
            .collect(Collectors.toList());

        System.out.println("=== Reporte de procesamiento ===");
        System.out.println("Transacciones exitosas: " + exitosas.size());
        System.out.println("Transacciones fallidas:  " + fallidas.size());

        double total = exitosas.stream()
            .mapToDouble(Transaccion::monto)
            .sum();
        System.out.printf("Total procesado: %.2f%n", total);

        if (!fallidas.isEmpty()) {
            System.out.println("\nErrores encontrados:");
            fallidas.forEach(e ->
                System.out.printf("  Línea %d: %s%n", e.numeroLinea, e.mensaje));
        }
    }

    record Transaccion(String fecha, double monto, String descripcion) {}
    record ErrorLinea(int numeroLinea, String mensaje) {}

    private static final AtomicInteger contadorLineas = new AtomicInteger(0);

    private static Result<Transaccion, ErrorLinea> parsearTransaccion(String linea) {
        int numLinea = contadorLineas.incrementAndGet();

        return Result.of(
            () -> {
                String[] partes = linea.split(",");
                if (partes.length != 3) {
                    throw new IllegalArgumentException("Formato inválido");
                }
                double monto = Double.parseDouble(partes[1].trim());
                if (monto <= 0) {
                    throw new IllegalArgumentException("Monto debe ser positivo");
                }
                return new Transaccion(partes[0].trim(), monto, partes[2].trim());
            },
            ex -> new ErrorLinea(numLinea, ex.getMessage())
        );
    }
}
```

---

## 5.12 Excepciones en aplicaciones multi-capa

### 5.12.1 Estrategia de manejo por capa

Una aplicación bien diseñada define una estrategia clara de excepciones para cada capa:

```
┌──────────────────────────────────────────────────────────┐
│  CAPA DE PRESENTACIÓN (Controllers, REST, UI)            │
│  - Traduce excepciones de dominio a códigos HTTP/UI      │
│  - Exception Shielding: nunca expone detalles internos   │
│  - Validación de entrada → 400 Bad Request               │
│  - Recurso no encontrado → 404 Not Found                 │
│  - Error de dominio → 422 Unprocessable Entity           │
│  - Error interno → 500 Internal Server Error             │
└────────────────────────┬─────────────────────────────────┘
                         │  DominioException
                         ▼
┌──────────────────────────────────────────────────────────┐
│  CAPA DE SERVICIO (Lógica de negocio)                    │
│  - Lanza excepciones de dominio específicas              │
│  - UsuarioNoEncontradoException                          │
│  - SaldoInsuficienteException                            │
│  - PedidoRechazadoException                              │
│  - Traduce excepciones técnicas a errores de dominio     │
└────────────────────────┬─────────────────────────────────┘
                         │  DataAccessException
                         ▼
┌──────────────────────────────────────────────────────────┐
│  CAPA DE DATOS (Repositorios, DAO)                       │
│  - Traduce SQLException a DataAccessException            │
│  - Traduce IOException de archivos                       │
│  - Nunca lanza excepciones de dominio hacia arriba       │
│    (no conoce el dominio de negocio)                     │
└────────────────────────┬─────────────────────────────────┘
                         │  SQLException, IOException
                         ▼
┌──────────────────────────────────────────────────────────┐
│  INFRAESTRUCTURA (BD, Archivos, Red, APIs externas)      │
│  - Lanza excepciones técnicas nativas                    │
│  - SQLException, IOException, SocketTimeoutException     │
└──────────────────────────────────────────────────────────┘
```

### 5.12.2 Implementación por capa

#### Capa de datos

```java
@Repository
public class PedidoRepository {

    private final JdbcTemplate jdbc;
    private final Logger logger = LoggerFactory.getLogger(PedidoRepository.class);

    public Optional<Pedido> findById(Long id) {
        String sql = """
            SELECT p.id, p.cliente_id, p.fecha, p.estado, p.total,
                   dp.producto_id, dp.cantidad, dp.precio_unitario
            FROM pedidos p
            LEFT JOIN detalle_pedido dp ON p.id = dp.pedido_id
            WHERE p.id = ?
            """;
        try {
            List<Pedido> resultados = jdbc.query(sql, new PedidoRowMapper(), id);
            if (resultados.isEmpty()) {
                return Optional.empty();
            }
            return Optional.of(agruparDetalles(resultados));

        } catch (DataAccessException e) {
            // Spring ya convierte SQLException a DataAccessException
            // Añadimos contexto para depuración
            logger.error("Error al buscar pedido ID={}", id, e);
            throw new DataAccessException(
                "Error al consultar pedido " + id + ": " + e.getMessage(), e
            ) {};
        }
    }

    public Pedido guardar(Pedido pedido) {
        try {
            // ... lógica de persistencia ...
            return pedido;
        } catch (DataAccessException e) {
            logger.error("Error al guardar pedido {}", pedido.getId(), e);
            throw new DataAccessException(
                "Error al persistir pedido " + pedido.getId(), e
            ) {};
        }
    }
}
```

#### Capa de servicio

```java
@Service
@Transactional
public class PedidoService {

    private final PedidoRepository repository;
    private final InventarioService inventario;
    private final Logger logger = LoggerFactory.getLogger(PedidoService.class);

    /**
     * Confirma un pedido verificando stock y realizando el cargo.
     *
     * @throws PedidoNoEncontradoException si el pedido no existe
     * @throws StockInsuficienteException si no hay stock suficiente
     * @throws PedidoYaConfirmadoException si ya estaba confirmado
     */
    public Pedido confirmarPedido(Long pedidoId) {
        Pedido pedido = repository.findById(pedidoId)
            .orElseThrow(() -> new PedidoNoEncontradoException(pedidoId));

        if (pedido.getEstado() == EstadoPedido.CONFIRMADO) {
            throw new PedidoYaConfirmadoException(pedidoId);
        }

        try {
            inventario.reservarStock(pedido.getDetalles());
        } catch (DataAccessException e) {
            // Error técnico → error de dominio (con causa preservada)
            logger.error("Error al verificar stock para pedido {}", pedidoId, e);
            throw new DominioException(
                "ERROR_INVENTARIO",
                "No se pudo verificar el inventario para el pedido " + pedidoId,
                e
            );
        }

        pedido.setEstado(EstadoPedido.CONFIRMADO);
        try {
            return repository.guardar(pedido);
        } catch (DataAccessException e) {
            throw new DominioException(
                "ERROR_PERSISTENCIA",
                "Error al confirmar pedido " + pedidoId, e
            );
        }
    }
}
```

#### Capa de presentación

```java
@RestController
@RequestMapping("/api/pedidos")
public class PedidoController {

    private final PedidoService service;
    private final Logger logger =
        LoggerFactory.getLogger(PedidoController.class);

    @PostMapping("/{id}/confirmar")
    public ResponseEntity<RespuestaPedido> confirmar(@PathVariable Long id) {
        // Sin try-catch: las excepciones las maneja @ControllerAdvice
        Pedido pedido = service.confirmarPedido(id);
        return ResponseEntity.ok(RespuestaPedido.from(pedido));
    }
}

/**
 * Manejador global: traduce cada tipo de excepción al código HTTP apropiado.
 */
@RestControllerAdvice
public class ApiExceptionHandler {

    private static final Logger logger =
        LoggerFactory.getLogger(ApiExceptionHandler.class);

    @ExceptionHandler(PedidoNoEncontradoException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            PedidoNoEncontradoException e) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("PEDIDO_NO_ENCONTRADO", e.getMessage()));
    }

    @ExceptionHandler(PedidoYaConfirmadoException.class)
    public ResponseEntity<ErrorResponse> handleYaConfirmado(
            PedidoYaConfirmadoException e) {
        return ResponseEntity.status(409) // Conflict
            .body(new ErrorResponse("PEDIDO_YA_CONFIRMADO", e.getMessage()));
    }

    @ExceptionHandler(StockInsuficienteException.class)
    public ResponseEntity<ErrorResponse> handleStock(
            StockInsuficienteException e) {
        return ResponseEntity.status(422) // Unprocessable Entity
            .body(new ErrorResponse("STOCK_INSUFICIENTE", e.getMessage(),
                 e.getProductosFaltantes()));
    }

    @ExceptionHandler(DominioException.class)
    public ResponseEntity<ErrorResponse> handleDominio(DominioException e) {
        logger.error("Error de dominio: [{}] {}", e.getCodigoError(),
                     e.getMessage(), e);
        return ResponseEntity.status(400)
            .body(new ErrorResponse(e.getCodigoError(), e.getMessage()));
    }

    @ExceptionHandler(DataAccessException.class)
    public ResponseEntity<ErrorResponse> handleDataAccess(
            DataAccessException e) {
        logger.error("Error de acceso a datos", e);
        // Exception Shielding: no exponemos detalles internos
        return ResponseEntity.status(500)
            .body(new ErrorResponse("ERROR_INTERNO",
                  "Error interno del servidor"));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception e) {
        logger.error("Error inesperado", e);
        return ResponseEntity.status(500)
            .body(new ErrorResponse("ERROR_INTERNO",
                  "Error inesperado. El equipo ha sido notificado."));
    }
}
```

### 5.12.3 Diagrama: viaje de una excepción a través de las capas

```
Cliente HTTP
  │  GET /api/pedidos/42/confirmar
  ▼
┌──────────────────────────────────────────────────┐
│ PedidoController.confirmar(42)                   │  PRESENTACIÓN
│   → service.confirmarPedido(42)                  │
└────────────────────┬─────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────┐
│ PedidoService.confirmarPedido(42)                │  SERVICIO
│   → repository.findById(42)                      │
│     → retorna Optional.empty()                   │
│   → orElseThrow(PedidoNoEncontradoException)     │
│                      │                           │
│    throw new PedidoNoEncontradoException(42)     │
└──────────────────────┬───────────────────────────┘
                       │  se propaga hacia arriba
                       ▼
┌──────────────────────────────────────────────────┐
│ PedidoController.confirmar(42)                   │  PRESENTACIÓN
│   ✗ No captura → se propaga al framework         │
└────────────────────┬─────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────┐
│ @RestControllerAdvice                            │
│ @ExceptionHandler(PedidoNoEncontradoException)   │
│   → ResponseEntity.status(404)                   │
│       .body({"codigo":"PEDIDO_NO_ENCONTRADO",    │
│              "mensaje":"Pedido 42 no encontrado"})│
└────────────────────┬─────────────────────────────┘
                     │
                     ▼
Cliente HTTP ← HTTP 404 Not Found
              {"codigo":"PEDIDO_NO_ENCONTRADO",
               "mensaje":"Pedido 42 no encontrado"}
```

### 5.12.4 Jerarquía de excepciones recomendada para aplicaciones multi-capa

```java
// ─── Base ───
public abstract class AplicacionException extends RuntimeException {
    private final String codigo;

    protected AplicacionException(String codigo, String mensaje) {
        super(mensaje);
        this.codigo = codigo;
    }

    protected AplicacionException(String codigo, String mensaje,
                                  Throwable causa) {
        super(mensaje, causa);
        this.codigo = codigo;
    }

    public String getCodigo() { return codigo; }
}

// ─── Recurso no encontrado ───
public abstract class RecursoNoEncontradoException
        extends AplicacionException {
    protected RecursoNoEncontradoException(String codigo, String mensaje) {
        super(codigo, mensaje);
    }
}

public class PedidoNoEncontradoException
        extends RecursoNoEncontradoException {
    private final Long pedidoId;

    public PedidoNoEncontradoException(Long pedidoId) {
        super("PEDIDO_NO_ENCONTRADO",
              "Pedido con ID " + pedidoId + " no encontrado");
        this.pedidoId = pedidoId;
    }
    public Long getPedidoId() { return pedidoId; }
}

// ─── Regla de negocio violada ───
public abstract class ReglaNegocioException extends AplicacionException {
    protected ReglaNegocioException(String codigo, String mensaje) {
        super(codigo, mensaje);
    }
}

public class SaldoInsuficienteException extends ReglaNegocioException {
    public SaldoInsuficienteException(BigDecimal disponible,
                                       BigDecimal requerido) {
        super("SALDO_INSUFICIENTE",
              String.format("Saldo insuficiente: disponible=%s, requerido=%s",
                            disponible, requerido));
    }
}

// ─── Infraestructura ───
public class InfraestructuraException extends AplicacionException {
    public InfraestructuraException(String codigo, String mensaje,
                                     Throwable causa) {
        super(codigo, mensaje, causa);
    }
}
```

---

## 5.13 Excepciones y concurrencia

### 5.13.1 El problema: excepciones en hilos secundarios no se propagan

Cuando una excepción ocurre en un hilo distinto al principal, **no se propaga automáticamente**. El hilo simplemente muere y la excepción se pierde si no se captura dentro del propio hilo:

```java
public class ExcepcionEnHilo {
    public static void main(String[] args) throws InterruptedException {
        Thread hilo = new Thread(() -> {
            System.out.println("Hilo iniciado...");
            throw new RuntimeException("¡Error en hilo secundario!");
            // Este código nunca se ejecuta
        }, "hilo-trabajador");

        hilo.start();

        Thread.sleep(1000); // dar tiempo al hilo para fallar
        System.out.println("Main sigue ejecutándose normalmente.");
        // La excepción del hilo NO afecta al hilo principal
    }
}
```

Salida:
```
Hilo iniciado...
Exception in thread "hilo-trabajador" java.lang.RuntimeException: ¡Error en hilo secundario!
    at ExcepcionEnHilo.lambda$main$0(ExcepcionEnHilo.java:7)
    at java.base/java.lang.Thread.run(Thread.java:1583)
Main sigue ejecutándose normalmente.
```

### 5.13.2 UncaughtExceptionHandler

Para capturar excepciones no manejadas en cualquier hilo, Java proporciona `UncaughtExceptionHandler`:

```java
public class CapturarExcepcionesHilos {

    public static void main(String[] args) throws InterruptedException {

        // Opción 1: Handler global para todos los hilos nuevos
        Thread.setDefaultUncaughtExceptionHandler((t, e) -> {
            System.err.printf("[GLOBAL] Excepción en hilo '%s': %s%n",
                              t.getName(), e.getMessage());
            // Registrar en log, enviar alerta, reiniciar el hilo...
        });

        // Opción 2: Handler específico para un hilo concreto
        Thread hilo = new Thread(() -> {
            System.out.println("Trabajando...");
            throw new RuntimeException("Fallo catastrófico en hilo");
        }, "hilo-especifico");

        hilo.setUncaughtExceptionHandler((t, e) -> {
            System.err.printf("[ESPECÍFICO] '%s' falló: %s%n",
                              t.getName(), e.getMessage());
            System.err.printf("[ESPECÍFICO] Estado: %s%n", t.getState());
        });

        hilo.start();
        hilo.join(); // esperar a que termine
        System.out.println("Estado final del hilo: " + hilo.getState());
    }
}
```

Salida:
```
Trabajando...
[ESPECÍFICO] 'hilo-especifico' falló: Fallo catastrófico en hilo
[ESPECÍFICO] Estado: RUNNABLE
Estado final del hilo: TERMINATED
```

### 5.13.3 Excepciones en CompletableFuture

`CompletableFuture` ofrece tres métodos para manejar excepciones de forma asíncrona: `exceptionally`, `handle` y `whenComplete`.

#### exceptionally

Se ejecuta solo si la etapa anterior falló. Permite proporcionar un valor de respaldo:

```java
CompletableFuture<String> futuro = CompletableFuture
    .supplyAsync(() -> {
        if (Math.random() > 0.5) {
            throw new RuntimeException("Servicio no disponible");
        }
        return "Resultado OK";
    })
    .exceptionally(ex -> {
        // Solo se ejecuta si falló
        System.err.println("Error: " + ex.getMessage());
        return "Valor de respaldo (fallback)";
    });

String resultado = futuro.join(); // no lanza excepción
System.out.println("Resultado: " + resultado);
```

#### handle (éxito o fallo)

Se ejecuta SIEMPRE, tanto si la etapa anterior tuvo éxito como si falló:

```java
CompletableFuture<String> futuro = CompletableFuture
    .supplyAsync(() -> {
        throw new RuntimeException("Conexión perdida");
    })
    .handle((resultado, excepcion) -> {
        // Se ejecuta SIEMPRE (éxito o fallo)
        if (excepcion != null) {
            System.err.println("Manejando error: " + excepcion.getMessage());
            return "Recuperado del error";
        }
        return "Éxito: " + resultado;
    });

System.out.println(futuro.join());
```

#### whenComplete (efecto secundario)

Similar a `finally`: se ejecuta siempre, pero **no modifica el resultado**. Si hubo excepción, se sigue propagando:

```java
CompletableFuture<String> futuro = CompletableFuture
    .supplyAsync(() -> {
        if (Math.random() > 0.3) {
            throw new RuntimeException("Fallo aleatorio");
        }
        return "Datos procesados";
    })
    .whenComplete((resultado, excepcion) -> {
        // Efecto secundario: logging, métricas, limpieza
        if (excepcion != null) {
            System.err.println("Operación falló: " + excepcion.getMessage());
        } else {
            System.out.println("Operación exitosa: " + resultado);
        }
    });
    // whenComplete NO modifica el resultado: si hubo excepción, se propaga

try {
    System.out.println("Resultado final: " + futuro.join());
} catch (CompletionException e) {
    System.err.println("CompletionException: " + e.getCause().getMessage());
}
```

#### Composición con manejo de errores en cada etapa

```java
/**
 * Pipeline asíncrono con manejo de errores en cada etapa.
 */
public class PipelineAsincrono {

    public static void main(String[] args) {

        CompletableFuture<ResultadoFinal> pipeline = CompletableFuture
            // Etapa 1: Buscar usuario (con fallback)
            .supplyAsync(() -> buscarUsuario(42L))
            .exceptionally(ex -> {
                System.err.println("Fallo en búsqueda: " + ex.getMessage());
                return new Usuario(42L, "Usuario por defecto", "");
            })

            // Etapa 2: Buscar pedidos (con fallback)
            .thenApplyAsync(usuario -> {
                List<Pedido> pedidos = buscarPedidos(usuario.id());
                return new UsuarioConPedidos(usuario, pedidos);
            })
            .exceptionally(ex -> {
                System.err.println("Fallo en pedidos: " + ex.getMessage());
                return new UsuarioConPedidos(
                    new Usuario(0L, "Error", ""), List.of());
            })

            // Etapa 3: Calcular total (no debería fallar con datos ya validados)
            .thenApplyAsync(up -> {
                BigDecimal total = up.pedidos().stream()
                    .map(Pedido::total)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);
                return new ResultadoFinal(up.usuario(), total);
            });

        ResultadoFinal resultado = pipeline.join();
        System.out.println("Pipeline completado: " + resultado);
    }

    record Usuario(Long id, String nombre, String email) {}
    record Pedido(Long id, BigDecimal total) {}
    record UsuarioConPedidos(Usuario usuario, List<Pedido> pedidos) {}
    record ResultadoFinal(Usuario usuario, BigDecimal totalPedidos) {}

    static Usuario buscarUsuario(Long id) {
        if (id == 99) throw new RuntimeException("Usuario no encontrado");
        return new Usuario(id, "Juan Pérez", "juan@example.com");
    }

    static List<Pedido> buscarPedidos(Long usuarioId) {
        if (usuarioId == 42) throw new RuntimeException("Timeout BD");
        return List.of(
            new Pedido(1L, new BigDecimal("150.00")),
            new Pedido(2L, new BigDecimal("89.99"))
        );
    }
}
```

### 5.13.4 Excepciones en ExecutorService

Cuando usas `ExecutorService.submit()`, las excepciones lanzadas por la tarea se envuelven en una `ExecutionException`:

```java
import java.util.concurrent.*;

public class ExcepcionesExecutorService {

    public static void main(String[] args) {

        ExecutorService executor = Executors.newFixedThreadPool(2);

        // ── submit(Callable) ──
        Future<Integer> futuro = executor.submit(() -> {
            System.out.println("Ejecutando tarea que fallará...");
            throw new RuntimeException("Error en la tarea Callable");
        });

        try {
            Integer resultado = futuro.get(); // bloquea hasta que termine
            System.out.println("Resultado: " + resultado);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.err.println("Tarea interrumpida");
        } catch (ExecutionException e) {
            // La excepción original está dentro de ExecutionException
            System.err.println("Excepción en la tarea:");
            System.err.println("  Tipo: " + e.getCause().getClass().getSimpleName());
            System.err.println("  Mensaje: " + e.getCause().getMessage());
        }

        // ── submit(Runnable) ──
        Future<?> futuroRunnable = executor.submit(() -> {
            throw new RuntimeException("Error en Runnable");
        });

        try {
            futuroRunnable.get(); // lanza ExecutionException
        } catch (ExecutionException e) {
            System.err.println("Error en Runnable: " + e.getCause().getMessage());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // ── execute(Runnable) ── ¡CUIDADO!
        // execute() NO devuelve Future. Las excepciones se pierden
        // a menos que uses UncaughtExceptionHandler
        executor.execute(() -> {
            throw new RuntimeException("Error en execute() - ¡se pierde!");
            // Esta excepción solo es visible con UncaughtExceptionHandler
        });

        executor.shutdown();
        try {
            executor.awaitTermination(2, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println("Programa finalizado.");
    }
}
```

**Resumen de propagación de excepciones en concurrencia**:

| Mecanismo | ¿Propaga excepciones al llamador? | Cómo acceder a la excepción |
|---|---|---|
| `new Thread().start()` | ❌ No | `UncaughtExceptionHandler` |
| `ExecutorService.execute()` | ❌ No | `UncaughtExceptionHandler` |
| `ExecutorService.submit(Callable)` | ✅ Sí (envuelta) | `Future.get()` → `ExecutionException.getCause()` |
| `ExecutorService.submit(Runnable)` | ✅ Sí (envuelta) | `Future.get()` → `ExecutionException.getCause()` |
| `CompletableFuture.supplyAsync()` | ✅ Sí (envuelta) | `join()` → `CompletionException.getCause()` |
| `CompletableFuture` con `.handle()` | ❌ (manejada) | `(resultado, excepcion)` en el callback |

### 5.13.5 Patrón: Supervisor de hilos

Un patrón útil en sistemas concurrentes es tener un hilo «supervisor» que monitoriza y reinicia hilos trabajadores caídos:

```java
public class SupervisorHilos {

    private static final Logger logger =
        LoggerFactory.getLogger(SupervisorHilos.class);

    public static void main(String[] args) {
        // Crear un hilo trabajador con supervisor
        crearHiloSupervisado("procesador-pagos", () -> {
            while (!Thread.currentThread().isInterrupted()) {
                procesarPago();
                Thread.sleep(2000);
            }
        });

        // El programa sigue vivo mientras los hilos supervisados existan
    }

    private static void crearHiloSupervisado(
            String nombre, ThrowingRunnable<Exception> tarea) {

        Runnable tareaConSupervision = () -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    tarea.run();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                } catch (Exception e) {
                    logger.error("Hilo '{}' falló. Reiniciando en 5s...: {}",
                                 nombre, e.getMessage());
                    try {
                        Thread.sleep(5000); // esperar antes de reiniciar
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                    logger.info("Reiniciando hilo '{}'...", nombre);
                    // El bucle continúa → reinicio del hilo lógico
                }
            }
            logger.info("Hilo '{}' finalizado.", nombre);
        };

        Thread hilo = new Thread(tareaConSupervision, nombre);
        hilo.setUncaughtExceptionHandler((t, e) ->
            logger.error("Hilo '{}' murió inesperadamente: {}",
                         t.getName(), e.getMessage())
        );
        hilo.start();
    }

    private static void procesarPago() {
        if (Math.random() < 0.3) { // 30% de probabilidad de fallo
            throw new RuntimeException("Error de conexión con pasarela de pago");
        }
        System.out.println("Pago procesado correctamente.");
    }
}
```

---

## 5.14 Aserciones (assert)

### 5.14.1 ¿Qué es una aserción?

Una **aserción** es una declaración booleana que el programador **cree** que siempre es verdadera en un punto concreto del código. Si resulta falsa, la JVM lanza un `AssertionError`.

Las aserciones son una herramienta de **desarrollo y depuración**, no de control de errores en producción. Sirven para documentar y verificar invariantes internos.

### 5.14.2 Sintaxis

```java
// Forma simple
assert condicion;

// Forma con mensaje
assert condicion : "Mensaje descriptivo si falla";
```

Ejemplos:

```java
public class Calculadora {

    public static double raizCuadrada(double x) {
        // Invariante de entrada: no podemos calcular raíces negativas
        assert x >= 0 : "x no puede ser negativo: " + x;

        double resultado = Math.sqrt(x);

        // Invariante de salida: el resultado al cuadrado debe ser ≈ x
        assert Math.abs(resultado * resultado - x) < 0.0001
            : "Error de precisión en raíz cuadrada";

        return resultado;
    }

    public static int dividir(int a, int b) {
        assert b != 0 : "El divisor no puede ser cero";
        return a / b;
    }

    // Aserción como invariante de clase
    private double saldo;
    private static final double SALDO_MAXIMO = 1_000_000_000;

    public void depositar(double monto) {
        saldo += monto;
        assert saldo <= SALDO_MAXIMO
            : "Saldo excede el máximo permitido: " + saldo;
    }
}
```

### 5.14.3 Habilitar y deshabilitar aserciones

Por defecto, las aserciones están **deshabilitadas** en tiempo de ejecución. La JVM las ignora completamente, por lo que no tienen coste en producción.

Para habilitarlas:

```bash
# Habilitar para todas las clases
java -ea com.miempresa.MiClase

# Habilitar para un paquete específico
java -ea:com.miempresa.servicio... com.miempresa.Main

# Habilitar para una clase específica
java -ea:com.miempresa.Calculadora com.miempresa.Main

# Deshabilitar para un paquete (mientras otros están habilitados)
java -ea -da:com.miempresa.externo... com.miempresa.Main
```

En código, puedes comprobar si las aserciones están habilitadas:

```java
boolean asercionesHabilitadas = false;
assert asercionesHabilitadas = true; // efecto secundario intencionado

if (asercionesHabilitadas) {
    System.out.println("Aserciones habilitadas");
}
```

### 5.14.4 ¿Cuándo usar assert vs excepciones?

| Usar **assert** | Usar **excepciones** |
|---|---|
| Para verificar **invariantes internos** que siempre deben ser verdad | Para validar **entradas del usuario** o condiciones externas |
| Para documentar suposiciones del programador | Para manejar condiciones de error recuperables |
| Para detectar **bugs durante el desarrollo** | Para gestionar fallos en **producción** |
| En **métodos privados** (tú controlas las entradas) | En **métodos públicos** (no controlas quién llama) |
| Se desactivan en producción (sin coste) | Siempre activas |

```java
// BIEN: aserción para invariante interno
private void ordenarInternamente(int[] arr) {
    // Este método es privado, nosotros controlamos las llamadas
    assert arr != null : "arr no debería ser null aquí";
    Arrays.sort(arr);
}

// MAL: aserción para validar entrada pública
public void procesarPago(BigDecimal monto) {
    assert monto.compareTo(BigDecimal.ZERO) > 0; // ¡MAL!
    // Si las aserciones están desactivadas, esta validación no existe.
    // Cualquier usuario podría pasar un monto negativo.
}

// BIEN: excepción para validar entrada pública
public void procesarPago(BigDecimal monto) {
    if (monto == null || monto.compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException(
            "El monto debe ser positivo. Recibido: " + monto);
    }
    // ... lógica de negocio ...
}

// BIEN: aserción para postcondición en método privado
private String formatearInternamente(double valor) {
    String resultado = String.format("%.2f", valor);
    assert resultado != null && !resultado.isEmpty()
        : "formatearInternamente produjo resultado vacío";
    return resultado;
}
```

### 5.14.5 Ejemplo: invariantes con aserciones en una estructura de datos

```java
/**
 * Implementación de una cola circular con invariantes verificados por aserciones.
 */
public class ColaCircular<T> {

    private final T[] elementos;
    private int cabeza;
    private int cola;
    private int size;

    @SuppressWarnings("unchecked")
    public ColaCircular(int capacidad) {
        if (capacidad <= 0) {
            throw new IllegalArgumentException(
                "Capacidad debe ser positiva: " + capacidad);
        }
        this.elementos = (T[]) new Object[capacidad];
        this.cabeza = 0;
        this.cola = 0;
        this.size = 0;
        verificarInvariante();
    }

    public void encolar(T elemento) {
        if (size == elementos.length) {
            throw new IllegalStateException("Cola llena");
        }
        elementos[cola] = elemento;
        cola = (cola + 1) % elementos.length;
        size++;
        verificarInvariante();
    }

    public T desencolar() {
        if (size == 0) {
            throw new NoSuchElementException("Cola vacía");
        }
        T elemento = elementos[cabeza];
        elementos[cabeza] = null; // evitar fuga de memoria
        cabeza = (cabeza + 1) % elementos.length;
        size--;
        verificarInvariante();
        return elemento;
    }

    public int size() { return size; }
    public boolean estaVacia() { return size == 0; }
    public boolean estaLlena() { return size == elementos.length; }

    /**
     * Verifica los invariantes de la estructura.
     * Se ejecuta SOLO cuando las aserciones están habilitadas (-ea).
     */
    private void verificarInvariante() {
        assert size >= 0 && size <= elementos.length
            : "Size inválido: " + size;
        assert cabeza >= 0 && cabeza < elementos.length
            : "Cabeza fuera de rango: " + cabeza;
        assert cola >= 0 && cola < elementos.length
            : "Cola fuera de rango: " + cola;

        // Si la cola está vacía, cabeza y cola deben coincidir
        if (size == 0) {
            assert cabeza == cola
                : "Cola vacía pero cabeza != cola: " + cabeza + " vs " + cola;
        }

        // Verificar que no hay elementos nulos en posiciones ocupadas
        int contador = 0;
        int idx = cabeza;
        while (contador < size) {
            assert elementos[idx] != null
                : "Elemento nulo en posición ocupada: " + idx;
            idx = (idx + 1) % elementos.length;
            contador++;
        }
    }

    // Demo
    public static void main(String[] args) {
        ColaCircular<String> cola = new ColaCircular<>(5);

        cola.encolar("A");
        cola.encolar("B");
        cola.encolar("C");

        System.out.println("Desencolado: " + cola.desencolar()); // A
        System.out.println("Desencolado: " + cola.desencolar()); // B

        cola.encolar("D");
        cola.encolar("E");
        cola.encolar("F");

        while (!cola.estaVacia()) {
            System.out.println("Desencolado: " + cola.desencolar());
        }

        System.out.println("Todas las aserciones pasaron correctamente.");
    }
}
```

Para ejecutar con aserciones habilitadas:

```bash
java -ea ColaCircular
```

### 5.14.6 Aserciones en pruebas unitarias

En pruebas unitarias, las aserciones son la herramienta principal. JUnit 5 usa `assertEquals`, `assertTrue`, `assertThrows`, etc. Pero las aserciones de Java (`assert`) también pueden usarse en tests si se configura el runner adecuadamente:

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculadoraTest {

    @Test
    void testDivision() {
        // Aserciones de JUnit (preferidas en tests)
        assertEquals(2, Calculadora.dividir(10, 5));
        assertThrows(ArithmeticException.class,
            () -> Calculadora.dividir(10, 0));
    }

    @Test
    void testRaizCuadrada() {
        double resultado = Calculadora.raizCuadrada(25);
        // assert de Java también funciona si -ea está habilitado
        assert resultado >= 0 : "La raíz cuadrada debe ser no negativa";
        assertEquals(5.0, resultado, 0.0001);
    }
}
```

---

## Resumen del capítulo

En este capítulo hemos cubierto en profundidad el manejo de excepciones en Java:

1. **Fundamentos**: qué son las excepciones, la pila de llamadas y el coste real de llenar el stack trace.
2. **Jerarquía**: `Throwable`, `Error`, `Exception`, `RuntimeException` y las principales excepciones del JDK.
3. **Mecanismos**: `try-catch-finally`, multi-catch, `throw`, `throws`, try-with-resources y excepciones suprimidas.
4. **Checked vs Unchecked**: diferencias, controversia y cuándo usar cada tipo.
5. **Recursos**: implementación de `AutoCloseable` propio, orden de cierre inverso, manejo de fallos en `close()`.
6. **Excepciones propias**: cuándo y cómo crearlas, campos de contexto, constructores recomendados.
7. **Buenas prácticas**: no ignorar, no capturar indiscriminadamente, logging adecuado, encadenamiento, mensajes descriptivos.
8. **Patrones**: Circuit Breaker, Retry con backoff exponencial, traducción de excepciones, Exception Shielding, Global Exception Handler.
9. **Alternativa Result/Either**: implementación completa, comparación con Optional, servicio de pago con Result.
10. **Programación funcional**: ThrowingFunction, @SneakyThrows, Stream con Result, Try monádico de Vavr.
11. **Aplicaciones multi-capa**: estrategia por capa, jerarquía de excepciones de dominio, diagrama de propagación.
12. **Concurrencia**: UncaughtExceptionHandler, CompletableFuture (exceptionally, handle, whenComplete), ExecutorService, supervisor de hilos.
13. **Aserciones**: sintaxis, habilitación con -ea, cuándo usar assert vs excepciones, invariantes verificables.

Las excepciones son una herramienta poderosa, pero como cualquier herramienta, requieren criterio. El abuso de excepciones (para control de flujo, sin información de contexto, ignoradas) conduce a sistemas frágiles y difíciles de depurar. El uso juicioso, combinado con patrones modernos como Result, Circuit Breaker y manejo centralizado, produce sistemas robustos, mantenibles y resilientes.

---

---

← [Capítulo anterior](capitulo-04-colecciones.md) | [Inicio](../README.md) | [Capítulo siguiente →](capitulo-06-io-nio.md)
