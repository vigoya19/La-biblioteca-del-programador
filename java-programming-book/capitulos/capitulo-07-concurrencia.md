# Capítulo 7: Concurrencia y Multithreading

---

## 7.1 ¿Qué es un hilo?

Un **hilo** (thread) es la unidad más pequeña de ejecución dentro de un proceso. Mientras que un **proceso** es un programa en ejecución con su propio espacio de memoria (segmento de código, heap, stack), un hilo es un flujo de ejecución independiente que comparte el espacio de memoria del proceso que lo contiene: todos los hilos de un mismo proceso comparten el mismo **heap** y el mismo **código**, pero cada uno posee su propia **pila (stack)** para variables locales y llamadas a método.

```
┌──────────────────────────────────────────────┐
│                  PROCESO                     │
│  ┌────────────────────────────────────────┐  │
│  │              HEAP (compartido)          │  │
│  │    - Objetos                            │  │
│  │    - Atributos de instancia             │  │
│  │    - Atributos estáticos                │  │
│  └────────────────────────────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │  Stack   │ │  Stack   │ │  Stack   │      │
│  │ Hilo 1   │ │ Hilo 2   │ │ Hilo 3   │      │
│  │vars/llam │ │vars/llam │ │vars/llam │      │
│  └──────────┘ └──────────┘ └──────────┘      │
└──────────────────────────────────────────────┘
```

**¿Por qué concurrencia?** La ley de Moore sobre la duplicación del rendimiento cada 18 meses llegó a su límite físico en cuanto a velocidad de reloj. Hoy los procesadores incrementan su capacidad agregando más **núcleos (cores)**. Para aprovechar múltiples cores necesitamos **paralelismo real**, no solo concurrencia simulada. Incluso con un solo core, la concurrencia mejora la capacidad de respuesta (responsiveness) al evitar bloquear la interfaz durante operaciones largas como E/S.

| Concepto | Descripción |
|---|---|
| **Concurrencia** | Múltiples tareas avanzan en el tiempo (ejecución entrelazada) |
| **Paralelismo** | Múltiples tareas se ejecutan simultáneamente (cores diferentes) |
| **Hilo** | Flujo de ejecución secuencial; varios comparten el heap de un proceso |

### 7.1.1 Concurrencia vs. Paralelismo en detalle

Es crucial distinguir concurrencia de paralelismo. La **concurrencia** es una propiedad del programa: la capacidad de manejar múltiples tareas a la vez en el tiempo, donde cada tarea avanza, pero no necesariamente al mismo instante. Es posible tener programas concurrentes en máquinas con un solo core: el scheduler del sistema operativo entrelaza la ejecución de los hilos, dando a cada uno pequeños intervalos de tiempo (quantum). La ilusión es que "todo avanza a la vez", pero en realidad el CPU alterna entre tareas miles de veces por segundo.

El **paralelismo**, en cambio, es una propiedad de la ejecución: múltiples tareas se ejecutan literalmente al mismo tiempo en cores físicamente distintos. Para que haya paralelismo se necesita hardware con múltiples cores/CPUs. La concurrencia habilita el paralelismo: un programa concurrente bien diseñado puede escalar a múltiples cores, mientras que uno secuencial no.

```
CONCURRENCIA (1 core, time-slicing):

  Hilo A: ████····████····████····
  Hilo B: ····████····████····████
  Hilo C: ·████····████····████·····
          └───────────────────────┘
                tiempo →

PARALELISMO (3 cores, ejecución simultánea):

  Core 1 - Hilo A: ████████████████
  Core 2 - Hilo B: ████████████████
  Core 3 - Hilo C: ████████████████
          └───────────────────────┘
                tiempo →
```

### 7.1.2 El costo de la concurrencia

Aunque la concurrencia es poderosa, tiene costos que deben conocerse:

| Costo | Descripción |
|---|---|
| **Creación de hilos** | Un hilo de plataforma reserva ~1 MB de stack y recursos del SO. Crear y destruir hilos frecuentemente es costoso. |
| **Cambio de contexto** | El scheduler del SO debe guardar y restaurar el estado de cada hilo (registros, contador de programa, stack pointer). Esto ocurre constantemente. |
| **Sincronización** | Los locks, las barreras de memoria y las operaciones atómicas tienen un costo real en ciclos de CPU. |
| **Complejidad cognitiva** | El código concurrente es inherentemente más difícil de razonar, testear y depurar que el código secuencial. |
| **Consumo de memoria** | Cada hilo consume stack space; miles de hilos pueden agotar la memoria del sistema. |

En el modelo de memoria de Java (JMM), cada hilo guarda variables en su propia cache local (registros y cache de CPU) y las sincroniza con la memoria principal en momentos determinados. Esto hace necesario usar mecanismos de sincronización como `volatile`, `synchronized` o locks; de lo contrario, un hilo podría no ver cambios realizados por otro.

---

## 7.2 Crear hilos

Java ofrece dos vías principales para crear hilos:

### 7.2.1 Extender la clase `Thread`

```java
class MiHilo extends Thread {
    @Override
    public void run() {
        System.out.println("Ejecutando en: " + Thread.currentThread().getName());
    }
}

// Uso
MiHilo hilo = new MiHilo();
hilo.setName("hilo-personalizado");
hilo.start();   // ← start(), NO run()
```

### 7.2.2 Implementar la interfaz `Runnable`

```java
class MiTarea implements Runnable {
    @Override
    public void run() {
        System.out.println("Ejecutando en: " + Thread.currentThread().getName());
    }
}

// Uso
Thread hilo = new Thread(new MiTarea(), "tarea-1");
hilo.start();
```

Con lambda (Java 8+):

```java
Thread hilo = new Thread(() -> {
    System.out.println("Hilo lambda: " + Thread.currentThread().getName());
});
hilo.start();
```

### 7.2.3 `Runnable` vs `Thread` — ¿cuál usar?

| Criterio | `extends Thread` | `implements Runnable` |
|---|---|---|
| Reusabilidad | La clase ya no puede heredar de otra | La clase puede heredar de cualquier otra |
| Acoplamiento | Alta (la tarea está ligada a Thread) | Baja (la tarea es independiente) |
| Flexibilidad | Menor | Puede enviarse a `ExecutorService`, `Thread`, etc. |
| Recomendación | Evitar | **Preferir siempre** |

**`Thread.currentThread()`** devuelve una referencia al hilo que está ejecutando ese fragmento de código. Es especialmente útil para obtener el nombre del hilo, interrumpirlo o verificarlo:

```java
public void run() {
    Thread actual = Thread.currentThread();
    System.out.println("Soy " + actual.getName() + " (ID: " + actual.getId() + ")");
}
```

### 7.2.4 Prioridades de hilos

Java permite asignar prioridades a los hilos, lo que **sugiere** al scheduler del SO qué hilos son más importantes. La prioridad es un entero entre `Thread.MIN_PRIORITY` (1) y `Thread.MAX_PRIORITY` (10); la normal es `Thread.NORM_PRIORITY` (5).

```java
Thread hiloAlta = new Thread(() -> {
    System.out.println("Soy prioritario");
});
hiloAlta.setPriority(Thread.MAX_PRIORITY);  // 10
hiloAlta.start();
```

**Precaución:** La prioridad es una sugerencia, no una orden. El comportamiento depende enteramente del sistema operativo y del scheduler. En la práctica, basar la corrección del programa en prioridades es frágil y no portable. Para control real de ejecución, se usan locks, semáforos y otros mecanismos de sincronización, no prioridades.

---

## 7.3 Ciclo de vida de un hilo

Un hilo en Java pasa por los siguientes estados definidos en el `enum` `Thread.State`:

```
                    ┌─────────┐
                    │   NEW   │  ← se creó el objeto Thread
                    └────┬────┘
                         │ start()
                    ┌────▼────┐
              ┌─────│RUNNABLE │◄────────────────────────┐
              │     └────┬────┘                          │
              │          │ scheduler asigna CPU          │
              │          │ (running)                     │
              │     ┌────▼────┐                          │
              │     │RUNNING  │──► sleep()/wait(ms) ──►  │
              │     └────┬────┘    join(ms)          │   │
              │          │         Lock.tryLock(ms)  │   │
              │          │         ┌────────────┐    │   │
              │          ├────────►│ BLOCKED    │    │   │
              │          │ sync    │(espera     │    │   │
              │          │ bloqueo │ monitor)   │    │   │
              │          │         └────┬───────┘    │   │
              │          │              │ obtiene    │   │
              │          │              │ lock       │   │
              │          │         ┌────▼───────┐    │   │
              │          ├────────►│  WAITING   │    │   │
              │          │ wait()  │(wait,join, │    │   │
              │          │ join()  │ park)      │    │   │
              │          │ park()  └────┬───────┘    │   │
              │          │              │ notify/    │   │
              │          │              │ unpark     │   │
              │          │         ┌────▼──────────┐ │   │
              │          │         │TIMED_WAITING  │ │   │
              │          │         │(sleep(ms),   │─┘   │
              │          │         │wait(ms),...)  │     │
              │          │         └────┬──────────┘     │
              │          │              │ tiempo/señal   │
              │          └──────────────┘                │
              │                run() termina             │
              │          ┌────▼────┐                     │
              │          │TERMINATED│                     │
              │          └─────────┘                     │
              └──── synchronized lock adquirido ─────────┘
```

**Estados:**

| Estado | Significado |
|---|---|
| `NEW` | Objeto `Thread` creado; `start()` aún no invocado |
| `RUNNABLE` | Listo para ejecutarse; puede estar running o esperando al scheduler |
| `BLOCKED` | Esperando adquirir un monitor (lock intrínseco) para entrar a un bloque `synchronized` |
| `WAITING` | Espera indefinida hasta que otro hilo lo despierte (`wait()`, `join()`, `LockSupport.park()`) |
| `TIMED_WAITING` | Espera con tiempo límite (`sleep(ms)`, `wait(ms)`, `join(ms)`, `tryLock(ms)`) |
| `TERMINATED` | El método `run()` finalizó |

### Métodos clave del ciclo de vida

**`start()` vs `run()`:**
```java
hilo.start();  // Crea un nuevo hilo nativo; invoca run() en ese hilo
hilo.run();    // Ejecuta run() en el hilo actual — NO inicia un nuevo hilo
```

**`sleep(long millis)`:**
```java
// Pausa el hilo actual por 1.5 segundos
Thread.sleep(1500);  // lanza InterruptedException
```

**`join()`:**
```java
Thread hilo = new Thread(() -> {
    // tarea larga...
});
hilo.start();
hilo.join();  // El hilo actual espera a que 'hilo' termine
System.out.println("El hilo terminó");
```

**`join(long millis)`** espera como máximo el tiempo indicado.

**`yield()`:**
```java
Thread.yield();  // Sugerencia al scheduler: cede el CPU voluntariamente
// No hay garantía de que el scheduler haga caso
```

**`interrupt()`:**
```java
Thread hilo = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // trabajo...
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // restablecer flag
            break;
        }
    }
});
hilo.start();
// ... después
hilo.interrupt();  // pone flag interrupted = true; si está en sleep/wait, lanza InterruptedException
```

La interrupción es cooperativa: el hilo destino debe verificar `isInterrupted()` o manejar `InterruptedException`.

### Hilos daemon

Un **hilo daemon** es un hilo de servicio que no impide que la JVM termine. La JVM se cierra cuando todos los hilos no-daemon han finalizado.

```java
Thread daemon = new Thread(() -> {
    while (true) {
        // limpieza periódica
    }
});
daemon.setDaemon(true);  // debe llamarse antes de start()
daemon.start();
```

**Casos de uso de daemon threads:** garbage collection, monitoreo de estado, heartbeat, auto-guardado periódico.

**Precaución:** Cuando la JVM termina, los hilos daemon se abortan abruptamente; los bloques `finally` pueden no ejecutarse y los recursos pueden no liberarse. No usar hilos daemon para operaciones que requieran finalización ordenada (escritura en BD, cierre de archivos).

---

## 7.4 Sincronización con `synchronized`

### 7.4.1 El problema de la condición de carrera

Consideremos una operación aparentemente inocente: `contador++`.

```java
public class Contador {
    private int valor = 0;

    public void incrementar() {
        valor++;  // ¡NO es atómico!
    }

    public int getValor() {
        return valor;
    }
}
```

`valor++` son en realidad **tres operaciones** a nivel de bytecode/JVM:

```
1. LEER  el valor actual de la variable
2. SUMAR 1 al valor leído
3. ESCRIBIR el nuevo valor de vuelta
```

**Demostración del problema:**

```java
public class DemoCondicionCarrera {
    private static int contador = 0;

    public static void main(String[] args) throws InterruptedException {
        Runnable tarea = () -> {
            for (int i = 0; i < 10_000; i++) {
                contador++;  // condición de carrera
            }
        };

        Thread hilo1 = new Thread(tarea);
        Thread hilo2 = new Thread(tarea);

        hilo1.start();
        hilo2.start();
        hilo1.join();
        hilo2.join();

        System.out.println("Valor esperado: 20000");
        System.out.println("Valor obtenido: " + contador);
        // Típicamente < 20000 porque algunas lecturas/escrituras se pisan
    }
}
```

**Resultado típico:** ~15000-19000 (varía cada ejecución). Se "pierden" incrementos porque:
- Hilo A lee valor = 42
- Hilo B lee valor = 42 (antes de que A escriba)
- Hilo A escribe 43
- Hilo B escribe 43 (sobrescribe el incremento de A)

### 7.4.2 Métodos `synchronized`

```java
public class ContadorSeguro {
    private int valor = 0;

    public synchronized void incrementar() {  // lock en 'this'
        valor++;
    }

    public synchronized int getValor() {      // lock en 'this'
        return valor;
    }
}
```

Cuando un hilo invoca un método `synchronized` de instancia, adquiere el **monitor** del objeto (el lock intrínseco asociado a esa instancia). Otros hilos que intenten invocar cualquier método `synchronized` de la misma instancia se bloquearán hasta que el lock sea liberado.

Para métodos `static synchronized`, el lock es sobre el objeto `Class` asociado:

```java
public class Fabrica {
    private static int serie = 0;

    public static synchronized int siguienteNumero() {
        return ++serie;  // lock en Fabrica.class
    }
}
```

### 7.4.3 Bloques `synchronized`

Los bloques sincronizados ofrecen **granularidad más fina** y permiten elegir el objeto de lock:

```java
public class ContadorFino {
    private int valor = 0;
    private final Object lock = new Object();

    public void incrementar() {
        // código no sincronizado aquí
        synchronized (lock) {
            valor++;  // solo esta sección está protegida
        }
        // más código no sincronizado
    }
}
```

**Ventajas del bloque sobre el método:**
- Solo se sincroniza la sección crítica, mejor rendimiento
- Se puede elegir un lock privado (`new Object()`) para evitar que código externo adquiera `this`

### 7.4.4 Reentrada (Reentrancy)

Un lock intrínseco es **reentrante**: un hilo que ya posee el lock puede adquirirlo de nuevo sin bloquearse.

```java
public class Reentrante {
    public synchronized void metodoA() {
        System.out.println("En A");
        metodoB();  // mismo hilo, mismo lock — no se bloquea
    }

    public synchronized void metodoB() {
        System.out.println("En B");
    }
}
```

La JVM lleva un contador de adquisiciones; cada salida de bloque `synchronized` decrementa el contador. El lock se libera cuando llega a cero.

### 7.4.5 Deadlock

El **deadlock** (abrazo mortal) ocurre cuando dos o más hilos se bloquean mutuamente para siempre, cada uno esperando un recurso que el otro posee.

**Ejemplo clásico de deadlock con dos locks:**

```java
public class DeadlockDemo {
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (lockA) {
                System.out.println("Hilo 1 adquirió lockA");
                try { Thread.sleep(100); } catch (InterruptedException e) {}

                synchronized (lockB) {  // espera lockB (lo tiene Hilo 2)
                    System.out.println("Hilo 1 adquirió lockB");
                }
            }
        }, "Hilo-1").start();

        new Thread(() -> {
            synchronized (lockB) {
                System.out.println("Hilo 2 adquirió lockB");
                try { Thread.sleep(100); } catch (InterruptedException e) {}

                synchronized (lockA) {  // espera lockA (lo tiene Hilo 1)
                    System.out.println("Hilo 2 adquirió lockA");
                }
            }
        }, "Hilo-2").start();
    }
}
```

**La secuencia del desastre:**

```
Tiempo │ Hilo 1              │ Hilo 2
───────┼─────────────────────┼─────────────────────
   t0  │ lock.lockA          │
   t1  │ hold lockA          │ lock.lockB
   t2  │ espera lockB        │ hold lockB
   t3  │ ...                 │ espera lockA
   t4  │ BLOQUEADO PARA      │ BLOQUEADO PARA
       │ SIEMPRE             │ SIEMPRE
```

**Detección de deadlocks con `jstack`:**

```bash
# Obtener PID del proceso Java
jps -l
# 12345 DeadlockDemo

# Thread dump
jstack 12345
```

`jstack` detecta deadlocks automáticamente e imprime:

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f9e88006200 (object LockB)
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f9e88006100 (object LockA)
  which is held by "Thread-1"
```

También puedes usar `jcmd <pid> Thread.print` o enviar la señal `kill -3 <pid>` (en sistemas Unix) para generar un thread dump.

**Estrategias para prevenir deadlocks:**

| Estrategia | Descripción |
|---|---|
| **Orden consistente de locks** | Todos los hilos adquieren locks en el mismo orden. Si siempre adquieres lockA antes que lockB, no hay deadlock. |
| **tryLock con timeout** | Intentar adquirir el lock por un tiempo limitado; si no se consigue, liberar los locks y reintentar. |
| **Reducir la sección crítica** | Mantener los locks por el menor tiempo posible. Cuanto menos código sincronizado, menos probabilidad de deadlock. |
| **AtomicInteger / ConcurrentHashMap** | Reemplazar locks con clases lock-free que no pueden causar deadlock. |
| **No hacer llamadas externas con locks** | No invocar métodos alien (de otras clases/librerías) mientras se mantiene un lock; ese método podría intentar adquirir otro lock. |

### 7.4.6 Livelock

El **livelock** es una situación en la que los hilos no están bloqueados, pero tampoco progresan: reaccionan perpetuamente a las acciones de los demás sin llegar a completar su trabajo. Es como dos personas intentando pasar por un pasillo estrecho: ambas se mueven hacia un lado simultáneamente, luego al otro lado, y así indefinidamente.

**Ejemplo: transferencia bancaria con reintentos:**

```java
public class LivelockDemo {
    static class Cuenta {
        int saldo;
        final Lock lock = new ReentrantLock();

        boolean transferir(Cuenta destino, int monto) {
            while (true) {
                if (this.lock.tryLock()) {
                    try {
                        if (destino.lock.tryLock()) {
                            try {
                                if (saldo >= monto) {
                                    saldo -= monto;
                                    destino.saldo += monto;
                                    return true;
                                }
                                return false;
                            } finally {
                                destino.lock.unlock();
                            }
                        }
                    } finally {
                        this.lock.unlock();
                    }
                }
                // Ambos fallan, ambos reintentan, colisión perpetua
                try { Thread.sleep(10); } catch (InterruptedException e) {}
            }
        }
    }

    public static void main(String[] args) {
        Cuenta a = new Cuenta();
        Cuenta b = new Cuenta();
        a.saldo = 1000;
        b.saldo = 1000;

        new Thread(() -> a.transferir(b, 100)).start();
        new Thread(() -> b.transferir(a, 100)).start();
    }
}
```

**Solución al livelock:** introducir un backoff aleatorio (random backoff) en lugar de un sleep fijo, para que un hilo eventualmente gane la carrera. O usar orden consistente de locks (adquirirlos en orden de ID de cuenta, por ejemplo):

```java
boolean transferir(Cuenta destino, int monto) {
    Cuenta primero = this.id < destino.id ? this : destino;
    Cuenta segundo = this.id < destino.id ? destino : this;

    primero.lock.lock();
    try {
        segundo.lock.lock();
        try {
            if (saldo >= monto) {
                saldo -= monto;
                destino.saldo += monto;
                return true;
            }
            return false;
        } finally {
            segundo.lock.unlock();
        }
    } finally {
        primero.lock.unlock();
    }
}
```

### 7.4.7 Starvation (Inanición)

La **inanición** ocurre cuando un hilo de baja prioridad nunca obtiene acceso al recurso porque hilos de mayor prioridad lo acaparan continuamente. No está bloqueado ni en deadlock: simplemente nunca le toca el turno.

**Ejemplo con `ReentrantLock` no fair:**

```java
ReentrantLock lock = new ReentrantLock(); // default: no fair

// Hilo 1: adquiere y libera el lock rápidamente en bucle
new Thread(() -> {
    while (true) {
        lock.lock();
        try { Thread.sleep(1); } catch (InterruptedException e) {}
        finally { lock.unlock(); }
    }
}).start();

// Hilo 2: podría nunca obtener el lock (inanición)
new Thread(() -> {
    lock.lock();
    try {
        System.out.println("¡Por fin!");
    } finally { lock.unlock(); }
}).start();
```

**Solución:** usar `new ReentrantLock(true)` (fair lock) que encola a los hilos por orden de llegada, garantizando que todos eventualmente obtengan el lock. Sin embargo, los locks fair tienen peor rendimiento que los unfair.

### 7.4.8 ThreadLocal

`ThreadLocal` proporciona una variable donde **cada hilo tiene su propia copia independiente**, aislada de los demás hilos. Ideal para datos que deben ser accesibles desde cualquier parte del flujo de ejecución sin pasarlos explícitamente como parámetro.

**Caso de uso típico: contexto de usuario en aplicación web:**

```java
public class UserContext {
    private static final ThreadLocal<String> usuarioActual = new ThreadLocal<>();

    public static void setUsuario(String nombre) {
        usuarioActual.set(nombre);
    }

    public static String getUsuario() {
        return usuarioActual.get();
    }

    public static void limpiar() {
        usuarioActual.remove();  // ¡IMPRESCINDIBLE!
    }
}

// En el filtro/servlet (cada petición llega en un hilo distinto):
public class AuthFilter {
    public void doFilter(Solicitud req) {
        try {
            UserContext.setUsuario(req.getUsername());
            // ... toda la lógica de negocio ve UserContext.getUsuario()
        } finally {
            UserContext.limpiar();  // previene memory leaks
        }
    }
}
```

**Cómo funciona ThreadLocal internamente:**

```
Thread-1 ──────► Thread.threadLocals (ThreadLocalMap)
                  ├── key: UserContext.usuarioActual → value: "alice"
                  └── key: requestId               → value: "req-001"

Thread-2 ──────► Thread.threadLocals (ThreadLocalMap)
                  ├── key: UserContext.usuarioActual → value: "bob"
                  └── key: requestId               → value: "req-002"
```

Cada `Thread` tiene un campo interno `threadLocals` de tipo `ThreadLocalMap`, que es un mapa donde las claves son referencias débiles a los objetos `ThreadLocal` y los valores son las copias por hilo.

| Método | Descripción |
|---|---|
| `set(T value)` | Establece el valor para el hilo actual |
| `T get()` | Obtiene el valor para el hilo actual |
| `void remove()` | Elimina el valor. **Siempre llamar en finally** |
| `static withInitial(Supplier)` | (Java 8+) Crea ThreadLocal con valor inicial |

**Abuso y memory leaks con ThreadLocal:**

Si usas `ThreadLocal` en un pool de hilos (como Tomcat, Netty, o `ExecutorService`), los hilos se reutilizan y nunca mueren. Si no llamas a `remove()`, los valores persisten entre peticiones (¡fuga de datos entre usuarios!) y nunca son recolectados por el GC.

```java
// PELIGRO: memory leak en pool de hilos
ExecutorService pool = Executors.newFixedThreadPool(10);
pool.submit(() -> {
    UserContext.setUsuario("alice");
    // ... hace trabajo ...
    // ¡FALTA UserContext.limpiar()!
    // Este hilo se devuelve al pool con "alice" todavía en ThreadLocal
});
```

**Mejor práctica con ThreadLocal:**

```java
public static void ejecutarConContexto(String usuario, Runnable tarea) {
    UserContext.setUsuario(usuario);
    try {
        tarea.run();
    } finally {
        UserContext.limpiar();  // garantía de limpieza
    }
}
```

---

## 7.5 La API de Lock (`java.util.concurrent.locks`)

Los locks explícitos ofrecen ventajas sobre `synchronized`:

| Característica | `synchronized` | `Lock` |
|---|---|---|
| Interrumpible | No | `lockInterruptibly()` |
| Timeout al adquirir | No | `tryLock(ms)` |
| Fairness (orden FIFO) | No | `ReentrantLock(true)` |
| Múltiples condiciones | Solo una (con `wait/notify`) | Múltiples `Condition` |
| Estructura | Bloque | Objeto (más flexible) |
| tryLock no bloqueante | No | `tryLock()` retorna inmediatamente |
| Saber si el lock está tomado | No hay método | `isLocked()`, `getQueueLength()` |

### 7.5.1 `ReentrantLock`

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ContadorConLock {
    private int valor = 0;
    private final Lock lock = new ReentrantLock();

    public void incrementar() {
        lock.lock();
        try {
            valor++;
        } finally {
            lock.unlock();  // ¡IMPRESCINDIBLE! finally garantiza liberación
        }
    }

    public int getValor() {
        lock.lock();
        try {
            return valor;
        } finally {
            lock.unlock();
        }
    }
}
```

**Con fairness:**

```java
// Fair: el hilo que más tiempo lleva esperando obtiene el lock
Lock fairLock = new ReentrantLock(true);
// Unfair (default): puede haber inanición (starvation) pero mejor rendimiento
Lock unfairLock = new ReentrantLock();  // o ReentrantLock(false)
```

### 7.5.2 `tryLock()` con timeout

```java
public class ProcesadorConTimeout {
    private final Lock lock = new ReentrantLock();

    public boolean procesarSiSePuede() {
        if (lock.tryLock()) {  // no bloquea
            try {
                // sección crítica
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;  // no se pudo adquirir
    }

    public void procesarConTimeout() {
        try {
            if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
                try {
                    // sección crítica
                } finally {
                    lock.unlock();
                }
            } else {
                System.out.println("No se pudo adquirir el lock en 500 ms");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Caso de uso real: evitar deadlocks con `tryLock` + backoff aleatorio:**

```java
public boolean transferir(Cuenta origen, Cuenta destino, double monto) {
    while (true) {
        if (origen.lock.tryLock()) {
            try {
                if (destino.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                    try {
                        if (origen.saldo >= monto) {
                            origen.saldo -= monto;
                            destino.saldo += monto;
                            return true;
                        }
                        return false;
                    } finally {
                        destino.lock.unlock();
                    }
                }
            } finally {
                origen.lock.unlock();
            }
        }
        try {
            Thread.sleep(ThreadLocalRandom.current().nextLong(10, 100));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
}
```

### 7.5.3 `lockInterruptibly()`

Un hilo bloqueado en `synchronized` no puede ser interrumpido (se queda esperando el lock para siempre). Con `Lock`:

```java
public void operacionInterrumpible() throws InterruptedException {
    lock.lockInterruptibly();
    try {
        // sección crítica
    } finally {
        lock.unlock();
    }
}
```

### 7.5.4 `Condition` — Productor-Consumidor con Lock y múltiples condiciones

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class BufferLimitado {
    private final String[] buffer;
    private int contador = 0, ponerIdx = 0, tomarIdx = 0;

    private final Lock lock = new ReentrantLock();
    private final Condition noLleno = lock.newCondition();
    private final Condition noVacio = lock.newCondition();

    public BufferLimitado(int capacidad) {
        buffer = new String[capacidad];
    }

    public void poner(String item) throws InterruptedException {
        lock.lock();
        try {
            while (contador == buffer.length) {
                noLleno.await();   // espera hasta que haya espacio
            }
            buffer[ponerIdx] = item;
            ponerIdx = (ponerIdx + 1) % buffer.length;
            contador++;
            noVacio.signal();      // avisa que ya hay algo que consumir
        } finally {
            lock.unlock();
        }
    }

    public String tomar() throws InterruptedException {
        lock.lock();
        try {
            while (contador == 0) {
                noVacio.await();   // espera hasta que haya elementos
            }
            String item = buffer[tomarIdx];
            tomarIdx = (tomarIdx + 1) % buffer.length;
            contador--;
            noLleno.signal();      // avisa que hay espacio
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

La diferencia clave con `wait`/`notify`: puedes tener **múltiples condiciones**, lo que permite un diseño más claro y mejor rendimiento.

**Comparativa `wait/notify` vs `Condition`:**

| Aspecto | `wait/notify` | `Condition` |
|---|---|---|
| Nº de salas de espera | 1 por objeto | Ilimitadas |
| Señalización | `notify()` despierta uno aleatorio | `signal()` despierta en ESA condición |
| Interrupción | Soporta interrupción | Soporta `awaitUninterruptibly()` |
| Timeout | `wait(ms)` | `await(ms, TimeUnit)` |
| Precisión | Milisegundos | Nanosegundos con `awaitNanos()` |

### 7.5.5 `ReadWriteLock`/`ReentrantReadWriteLock`

Permite lecturas concurrentes pero bloquea la escritura:

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class CacheSeguro {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private Map<String, String> datos = new HashMap<>();

    public String leer(String clave) {
        rwLock.readLock().lock();
        try {
            return datos.get(clave);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void escribir(String clave, String valor) {
        rwLock.writeLock().lock();
        try {
            datos.put(clave, valor);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

Múltiples hilos pueden leer simultáneamente (readLock compartido), pero cuando uno escribe (writeLock exclusivo), nadie más puede leer ni escribir.

**Degradación de lock (downgrade):** Un hilo con writeLock puede adquirir readLock, liberar writeLock, y seguir con readLock:

```java
public void actualizarYCachear(String clave) {
    rwLock.writeLock().lock();
    try {
        datos.put(clave, calcular(clave));
        rwLock.readLock().lock();  // adquirir readLock antes de soltar writeLock
    } finally {
        rwLock.writeLock().unlock();  // libera escritura, mantiene lectura
    }
    try {
        System.out.println("Valor actualizado: " + datos.get(clave));
    } finally {
        rwLock.readLock().unlock();
    }
}
```

---

## 7.6 `wait()`, `notify()`, `notifyAll()`

Son los mecanismos de bajo nivel para coordinación entre hilos. Deben invocarse **siempre dentro de un bloque `synchronized`** sobre el objeto que actúa como monitor.

| Método | Efecto |
|---|---|
| `wait()` | El hilo actual libera el lock y se duerme hasta que otro hilo llame a `notify()`/`notifyAll()` sobre el mismo objeto |
| `wait(long ms)` | Como `wait()` pero se despierta automáticamente tras `ms` milisegundos |
| `notify()` | Despierta a **un** hilo que esté esperando en ese monitor (no hay control sobre cuál) |
| `notifyAll()` | Despierta a **todos** los hilos que esperan en ese monitor |

### Ejemplo: Productor-Consumidor

```java
import java.util.LinkedList;
import java.util.Queue;

public class ProductorConsumidor {
    private final Queue<Integer> cola = new LinkedList<>();
    private final int CAPACIDAD = 5;

    public void producir() throws InterruptedException {
        int valor = 0;
        while (true) {
            synchronized (cola) {
                while (cola.size() == CAPACIDAD) {
                    cola.wait();   // esperar espacio
                }
                cola.add(valor);
                System.out.println("Producido: " + valor);
                valor++;
                cola.notifyAll();  // avisar consumidores
            }
            Thread.sleep(500);
        }
    }

    public void consumir() throws InterruptedException {
        while (true) {
            synchronized (cola) {
                while (cola.isEmpty()) {
                    cola.wait();   // esperar elementos
                }
                int valor = cola.poll();
                System.out.println("Consumido: " + valor);
                cola.notifyAll();   // avisar productores
            }
            Thread.sleep(1000);
        }
    }

    public static void main(String[] args) {
        ProductorConsumidor pc = new ProductorConsumidor();
        new Thread(() -> { try { pc.producir(); } catch (InterruptedException e) {} }).start();
        new Thread(() -> { try { pc.consumir(); } catch (InterruptedException e) {} }).start();
    }
}
```

**Puntos importantes:**
- Usar siempre `while` (no `if`) para re-verificar la condición al despertar (protección contra spurious wakeups)
- Preferir `notifyAll()` sobre `notify()` excepto cuando se sabe exactamente qué hilo despertar
- Para código moderno, se recomienda usar `Lock` + `Condition` o `BlockingQueue`

### Spurious wakeups

Un **spurious wakeup** es un despertar de `wait()` sin que nadie haya llamado a `notify()`/`notifyAll()`. Por eso se usa `while` y no `if`:

```java
// INCORRECTO: vulnerable a spurious wakeups
synchronized (lock) {
    if (condicion == false) {
        lock.wait();  // si despierta espuriamente, sigue sin verificar
    }
    // asume que condicion == true — potencial bug
}

// CORRECTO: re-verifica la condición
synchronized (lock) {
    while (condicion == false) {
        lock.wait();  // si despierta espuriamente, vuelve a verificar
    }
    // ahora sí, condicion == true garantizado
}
```

---

## 7.7 `ExecutorService` y ThreadPool

Crear y destruir hilos es costoso. Un **thread pool** mantiene un conjunto de hilos reutilizables.

### 7.7.1 Fábrica `Executors`

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

// Pool con número fijo de hilos
ExecutorService poolFijo = Executors.newFixedThreadPool(4);

// Pool que crece según demanda, reutiliza hilos inactivos por 60s
ExecutorService poolCache = Executors.newCachedThreadPool();

// Un solo hilo (ejecución secuencial garantizada)
ExecutorService poolSimple = Executors.newSingleThreadExecutor();

// Pool para tareas programadas
ScheduledExecutorService poolProgramado = Executors.newScheduledThreadPool(2);
```

### 7.7.2 Uso de `ExecutorService`

```java
ExecutorService executor = Executors.newFixedThreadPool(3);

executor.submit(() -> {
    System.out.println("Tarea 1 en: " + Thread.currentThread().getName());
});

executor.submit(() -> {
    System.out.println("Tarea 2 en: " + Thread.currentThread().getName());
});

// Cierre ordenado: no acepta más tareas, termina las pendientes
executor.shutdown();

try {
    if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
        executor.shutdownNow();
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

**Cierre graceful idiomático:**

```java
void shutdownAndAwaitTermination(ExecutorService pool) {
    pool.shutdown();
    try {
        if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
            pool.shutdownNow();
            if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
                System.err.println("El pool no terminó");
            }
        }
    } catch (InterruptedException e) {
        pool.shutdownNow();
        Thread.currentThread().interrupt();
    }
}
```

### 7.7.3 `ThreadPoolExecutor` — parámetros

`Executors` es una fábrica de conveniencia. Por debajo crea `ThreadPoolExecutor`:

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    4,                     // corePoolSize
    10,                    // maximumPoolSize
    60L, TimeUnit.SECONDS, // keepAliveTime
    new LinkedBlockingQueue<>(100),  // cola de tareas pendientes
    new ThreadPoolExecutor.CallerRunsPolicy()  // política de rechazo
);
```

**Políticas de rechazo:**

| Política | Comportamiento |
|---|---|
| `AbortPolicy` (default) | Lanza `RejectedExecutionException` |
| `CallerRunsPolicy` | El hilo que envía ejecuta la tarea |
| `DiscardPolicy` | Descarta silenciosamente la tarea |
| `DiscardOldestPolicy` | Descarta la tarea más antigua de la cola |

**Estrategia de dimensionamiento del pool:**

Para tareas **CPU-bound**: `corePoolSize = Número de cores disponibles`.

```java
int cores = Runtime.getRuntime().availableProcessors();
ExecutorService pool = Executors.newFixedThreadPool(cores);
```

Para tareas **I/O-bound**: el número puede ser mayor. Heurística: `N_threads = N_cores * (1 + tiempo_espera / tiempo_cpu)`.

---

## 7.8 `Callable` y `Future`

`Runnable` no retorna valor ni lanza excepciones checked. `Callable<V>` resuelve ambas limitaciones.

```java
import java.util.concurrent.*;

Callable<Integer> tarea = () -> {
    Thread.sleep(2000);
    return 42;
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> futuro = executor.submit(tarea);

System.out.println("¿Terminó?: " + futuro.isDone());   // false

// get() bloquea hasta que el resultado esté disponible
Integer resultado = futuro.get();  // 42

// get() con timeout
try {
    Integer resultado2 = futuro.get(1, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    System.out.println("Timeout");
    futuro.cancel(true);
}

executor.shutdown();
```

**Métodos de `Future<V>`:**

| Método | Descripción |
|---|---|
| `V get()` | Bloquea hasta obtener el resultado |
| `V get(long, TimeUnit)` | Bloquea hasta obtener o timeout |
| `boolean cancel(boolean)` | Intenta cancelar la tarea |
| `boolean isCancelled()` | `true` si fue cancelada |
| `boolean isDone()` | `true` si terminó |


---

## 7.9 `CompletableFuture` (Java 8+)

`CompletableFuture<T>` implementa tanto `Future<T>` como `CompletionStage<T>`, permitiendo componer operaciones asíncronas de forma declarativa.

### 7.9.1 Creación

```java
import java.util.concurrent.CompletableFuture;

// Ejecutar asíncronamente en un hilo del ForkJoinPool común
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
    return "Hola";
});

// Con Runnable (no retorna valor)
CompletableFuture<Void> cf2 = CompletableFuture.runAsync(() -> {
    System.out.println("Tarea sin resultado");
});

// Completar manualmente (útil para callbacks)
CompletableFuture<String> cf3 = new CompletableFuture<>();
cf3.complete("Listo!");
```

### 7.9.2 Encadenamiento

```java
CompletableFuture<String> cadena = CompletableFuture
    .supplyAsync(() -> "usuario:123")
    .thenApply(usuario -> usuario.toUpperCase())
    .thenCompose(usuario -> buscarPermisosAsync(usuario))
    .thenAccept(permisos -> System.out.println("Permisos: " + permisos));
```

**Métodos de encadenamiento:**

| Método | Entrada | Salida | Uso |
|---|---|---|---|
| `thenApply(Function)` | T | U | Transformar resultado |
| `thenAccept(Consumer)` | T | Void | Consumir resultado |
| `thenRun(Runnable)` | — | Void | Ejecutar acción al terminar |
| `thenCompose(Function)` | T | CF<U> | Encadenar otro async (flatMap) |
| `thenCombine(CF, BiFunction)` | T, U | V | Combinar dos futuros |

### 7.9.3 Combinación

```java
CompletableFuture<Integer> futuro1 = CompletableFuture.supplyAsync(() -> {
    sleep(1000); return 10;
});
CompletableFuture<Integer> futuro2 = CompletableFuture.supplyAsync(() -> {
    sleep(2000); return 32;
});

CompletableFuture<Integer> suma = futuro1.thenCombine(futuro2, (a, b) -> a + b);
suma.thenAccept(resultado -> System.out.println("Suma: " + resultado)); // 42
```

### 7.9.4 Manejo de excepciones

```java
CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) throw new RuntimeException("¡Falló!");
    return "Éxito";
})
.exceptionally(ex -> {
    System.out.println("Error: " + ex.getMessage());
    return "Valor por defecto";
})
.handle((resultado, ex) -> {
    if (ex != null) return "Fallback: " + ex.getMessage();
    return "Resultado: " + resultado;
})
.thenAccept(System.out::println);
```

Diferencia entre `exceptionally` y `handle`:
- `exceptionally`: solo maneja error (recibe `Throwable`, retorna valor de recuperación)
- `handle`: maneja éxito y error (recibe `(resultado, excepcion)`, siempre se ejecuta)

**Manejo con `whenComplete` (side effect):**

```java
CompletableFuture.supplyAsync(() -> calcular())
    .whenComplete((resultado, error) -> {
        if (error != null) log.error("Falló", error);
        else log.info("Éxito: {}", resultado);
    });
```

### 7.9.5 Esperar múltiples futuros

```java
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> "B");
CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(() -> "C");

// Esperar a que TODOS terminen
CompletableFuture<Void> todos = CompletableFuture.allOf(cf1, cf2, cf3);
todos.join();

// Esperar a que CUALQUIERA termine
CompletableFuture<Object> cualquiera = CompletableFuture.anyOf(cf1, cf2, cf3);
Object resultado = cualquiera.get();  // "A", "B" o "C"
```

### 7.9.6 Ejecutor personalizado

Por defecto, los métodos `*Async` sin ejecutor usan `ForkJoinPool.commonPool()`. Para operaciones de bloqueo (E/S, BD), se debe usar un pool dedicado:

```java
ExecutorService miPool = Executors.newFixedThreadPool(4);

CompletableFuture.supplyAsync(() -> {
    return llamarApi();  // operación bloqueante
}, miPool);  // ← ejecutor propio

miPool.shutdown();
```

### 7.9.7 `CompletableFuture` avanzado: pipelines asíncronos

**Pipeline de procesamiento de datos:**

```java
public class PipelineDatos {
    public static void main(String[] args) {
        CompletableFuture<List<String>> resultado = CompletableFuture
            .supplyAsync(() -> obtenerIdsUsuarios())          // I/O
            .thenApplyAsync(ids -> ids.stream()               // CPU
                .map(String::toUpperCase)
                .toList())
            .thenComposeAsync(ids -> {                        // I/O paralelo
                List<CompletableFuture<String>> futuros = ids.stream()
                    .map(id -> CompletableFuture.supplyAsync(
                        () -> enriquecerUsuario(id)))
                    .toList();
                return CompletableFuture.allOf(
                    futuros.toArray(new CompletableFuture[0]))
                    .thenApply(v -> futuros.stream()
                        .map(CompletableFuture::join)
                        .toList());
            })
            .thenApplyAsync(datos -> datos.stream()           // CPU
                .filter(d -> d.length() > 5)
                .toList());

        List<String> datosFinales = resultado.join();
        System.out.println("Resultado: " + datosFinales);
    }
}
```

### 7.9.8 Timeout con `orTimeout` y `completeOnTimeout` (Java 9+)

```java
CompletableFuture<String> futuro = CompletableFuture.supplyAsync(() -> {
    sleep(5000);
    return "Resultado";
});

// Java 9+: lanzar TimeoutException
futuro.orTimeout(2, TimeUnit.SECONDS)
    .exceptionally(ex -> "Timeout!");

// Java 9+: proveer valor por defecto
futuro.completeOnTimeout("ValorDefault", 2, TimeUnit.SECONDS);
```

### 7.9.9 Ejemplo completo: agregación de precios

```java
public class AgregadorPrecios {
    public static void main(String[] args) {
        CompletableFuture<Double> precioTiendaA = CompletableFuture.supplyAsync(() -> {
            sleep(1200); return 99.99;
        });
        CompletableFuture<Double> precioTiendaB = CompletableFuture.supplyAsync(() -> {
            sleep(800); return 95.50;
        });
        CompletableFuture<Double> precioTiendaC = CompletableFuture.supplyAsync(() -> {
            sleep(1500); return 102.00;
        });

        CompletableFuture<Void> mejorPrecio = CompletableFuture
            .allOf(precioTiendaA, precioTiendaB, precioTiendaC)
            .thenRun(() -> {
                try {
                    double mejor = Math.min(
                        Math.min(precioTiendaA.get(), precioTiendaB.get()),
                        precioTiendaC.get()
                    );
                    System.out.println("Mejor precio: $" + mejor);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });

        mejorPrecio.join();
    }

    private static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }
}
```

---

## 7.10 Clases concurrentes

El paquete `java.util.concurrent` ofrece estructuras de datos y utilidades diseñadas para entornos multihilo sin necesidad de sincronización explícita.

### 7.10.1 `ConcurrentHashMap`

Mapa concurrente de alto rendimiento. Antes de Java 8, usaba bloqueo por segmentos (striped locking); desde Java 8 usa operaciones **CAS** (Compare-And-Swap) para actualizaciones sin bloqueo.

```java
import java.util.concurrent.ConcurrentHashMap;

ConcurrentHashMap<String, Integer> mapa = new ConcurrentHashMap<>();

mapa.put("clave", 1);

// Operaciones atómicas sin sincronización externa
mapa.putIfAbsent("clave", 2);
mapa.replace("clave", 1, 3);
mapa.remove("clave", 3);

// Computación atómica
mapa.compute("contador", (k, v) -> v == null ? 1 : v + 1);
mapa.computeIfAbsent("cache", k -> calcularCostoso(k));

// Operaciones atómicas compuestas
mapa.merge("visitas", 1L, Long::sum);      // incremento atómico
mapa.merge("maximo", nuevoValor, Math::max); // acumulador

mapa.compute("clave", (k, v) -> {
    if (v == null) return 1L;
    if (v > 1000) return null; // elimina entrada
    return v + 1;
});

// Búsqueda y reducción paralela
String resultado = mapa.search(10, (k, v) -> v > 100 ? k : null);
int suma = mapa.reduceValues(10, Integer::sum);

// Recorrido consistente sin ConcurrentModificationException
mapa.forEach((k, v) -> System.out.println(k + ": " + v));
```

### 7.10.2 `CopyOnWriteArrayList`

Lista thread-safe optimizada para **muchas lecturas y pocas escrituras**. Cada mutación crea una copia nueva del array interno.

```java
import java.util.concurrent.CopyOnWriteArrayList;

CopyOnWriteArrayList<String> lista = new CopyOnWriteArrayList<>();
lista.add("A");
lista.add("B");

// Iteración segura sin lock — itera sobre una snapshot
for (String s : lista) {
    lista.add("C");           // no ConcurrentModificationException
    System.out.println(s);    // "A", "B" (la iteración ve la snapshot)
}
```

```
Funcionamiento interno:

Antes de add("C"):
  array ──► [A, B]

add("C"):
  1. Copia: [A, B] → [A, B, C]
  2. Reemplaza referencia atómica: array ──► [A, B, C]

El iterador antiguo sigue viendo [A, B] (snapshot inmutable)
```

**Cuándo usar:** listeners/observers, catálogos de cambio lento, listas negras, configuraciones leídas frecuentemente.
**Cuándo NO usar:** muchas escrituras (cada escritura copia todo el array, O(n)). Usar `ConcurrentLinkedQueue` o `synchronizedList`.

### 7.10.3 `BlockingQueue`

Colas thread-safe que bloquean al productor si la cola está llena y al consumidor si está vacía.

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

BlockingQueue<String> colaArray = new ArrayBlockingQueue<>(10);
BlockingQueue<String> colaEnlazada = new LinkedBlockingQueue<>(100);
```

**Métodos principales:**

| Operación | Lanza excepción | Valor especial | Bloquea | Timeout |
|---|---|---|---|---|
| Insertar | `add(e)` | `offer(e)` | `put(e)` | `offer(e, time, unit)` |
| Eliminar | `remove()` | `poll()` | `take()` | `poll(time, unit)` |
| Examinar | `element()` | `peek()` | — | — |

**Comparativa de implementaciones:**

| Implementación | Capacidad | Orden | Uso típico |
|---|---|---|---|
| `ArrayBlockingQueue` | Fija | FIFO | Buffer limitado clásico |
| `LinkedBlockingQueue` | Opcional | FIFO | Cola de tareas de Executors |
| `PriorityBlockingQueue` | Ilimitada | Comparator | Tareas con prioridad |
| `DelayQueue` | Ilimitada | Tiempo retraso | Tareas programadas |
| `SynchronousQueue` | 0 | Transferencia directa | Handoff entre hilos |

**Ejemplo productor-consumidor con `BlockingQueue`:**

```java
public class ProductorConsumidorBQ {
    private static final BlockingQueue<Integer> cola = new LinkedBlockingQueue<>(5);

    public static void main(String[] args) {
        new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                try {
                    cola.put(i);
                    System.out.println("Producido: " + i);
                    Thread.sleep(300);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }).start();

        new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                try {
                    Integer valor = cola.take();
                    System.out.println("Consumido: " + valor);
                    Thread.sleep(600);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }).start();
    }
}
```

### 7.10.4 `CountDownLatch`

Permite que uno o más hilos esperen hasta que un conjunto de operaciones se complete. Cuenta regresiva de **un solo uso**.

```java
import java.util.concurrent.CountDownLatch;

public class DemoCountDownLatch {
    public static void main(String[] args) throws InterruptedException {
        int numServicios = 3;
        CountDownLatch latch = new CountDownLatch(numServicios);

        for (int i = 1; i <= numServicios; i++) {
            final int id = i;
            new Thread(() -> {
                System.out.println("Iniciando servicio " + id);
                try { Thread.sleep(id * 1000); } catch (InterruptedException e) {}
                System.out.println("Servicio " + id + " listo");
                latch.countDown();
            }).start();
        }

        latch.await();
        System.out.println("¡Todos los servicios listos! Arrancando aplicación...");
    }
}
```

### 7.10.5 `CyclicBarrier`

Punto de sincronización donde varios hilos se esperan mutuamente. **Reutilizable** (cíclico).

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class DemoCyclicBarrier {
    public static void main(String[] args) {
        final int NUM_JUGADORES = 3;
        CyclicBarrier barrera = new CyclicBarrier(NUM_JUGADORES, () -> {
            System.out.println("--- Todos listos, empieza la ronda ---");
        });

        for (int i = 1; i <= NUM_JUGADORES; i++) {
            final int jugador = i;
            new Thread(() -> {
                for (int ronda = 1; ronda <= 3; ronda++) {
                    System.out.println("Jugador " + jugador + " preparándose ronda " + ronda);
                    try { Thread.sleep((long)(Math.random() * 2000)); } catch (InterruptedException e) {}
                    System.out.println("Jugador " + jugador + " esperando en ronda " + ronda);
                    try {
                        barrera.await();
                    } catch (InterruptedException | BrokenBarrierException e) {
                        break;
                    }
                }
            }).start();
        }
    }
}
```

**Comparativa CountDownLatch vs CyclicBarrier:**

| Característica | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| Reutilización | Un solo uso | Reutilizable |
| Propósito | Esperar a que N hilos terminen | N hilos se esperan entre sí en un punto |
| Acción al llegar | No | Runnable opcional al llegar todos |
| Conteo | Cuenta regresiva (countDown) | Cuenta ascendente hasta parties |

### 7.10.6 `Semaphore`

Controla el número de hilos que pueden acceder a un recurso simultáneamente (permisos).

```java
import java.util.concurrent.Semaphore;

public class DemoPoolConexiones {
    private static final Semaphore semaforo = new Semaphore(3);

    static class Tarea implements Runnable {
        private final int id;
        Tarea(int id) { this.id = id; }

        @Override
        public void run() {
            try {
                semaforo.acquire();
                System.out.println("Tarea " + id + " usando conexión");
                Thread.sleep(2000);
                System.out.println("Tarea " + id + " libera conexión");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                semaforo.release();
            }
        }
    }

    public static void main(String[] args) {
        for (int i = 1; i <= 10; i++) {
            new Thread(new Tarea(i)).start();
        }
    }
}
```

**Semaphore vs ReentrantLock:**

| Aspecto | `Semaphore` | `ReentrantLock` |
|---|---|---|
| Propósito | N accesos concurrentes | Exclusión mutua (1 acceso) |
| Liberación | Desde otro hilo | Desde el mismo hilo |
| Reentrante | No | Sí |
| Dueño | Sin concepto de dueño | `isHeldByCurrentThread()` |

### 7.10.7 Tabla comparativa de colecciones thread-safe

| Uso | Pre-Java 5 | Java 5+ (`concurrent`) |
|---|---|---|
| Map sincronizado | `Collections.synchronizedMap(new HashMap<>())` | `ConcurrentHashMap` |
| List sincronizada | `Collections.synchronizedList(new ArrayList<>())` | `CopyOnWriteArrayList` |
| Cola bloqueante | Manual con `wait/notify` | `BlockingQueue` |
| Contador atómico | `synchronized` sobre `int` | `AtomicInteger` |

**Regla general:** Preferir las clases de `java.util.concurrent` sobre envolturas sincronizadas de `Collections`.

---

## 7.11 El Modelo de Memoria de Java (JMM) en profundidad

### 7.11.1 ¿Qué es la JMM y por qué existe?

El **Java Memory Model (JMM)** es la especificación formal que define cómo los hilos interactúan a través de la memoria: cuándo un hilo ve las escrituras de otro, cuándo las operaciones son atómicas y qué garantías de visibilidad y ordenamiento existen. Sin la JMM, escribir código concurrente correcto sería imposible, porque el comportamiento variaría entre arquitecturas de hardware.

**El problema físico: cachés de CPU y reordenamiento**

Un procesador moderno tiene varios niveles de caché entre los cores y la memoria principal (RAM):

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│    Core 0    │  │    Core 1    │  │    Core 2    │  │    Core 3    │
│  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
│  │L1 Cache│  │  │  │L1 Cache│  │  │  │L1 Cache│  │  │  │L1 Cache│  │
│  │(32 KB) │  │  │  │(32 KB) │  │  │  │(32 KB) │  │  │  │(32 KB) │  │
│  └────┬───┘  │  │  └────┬───┘  │  │  └────┬───┘  │  │  └────┬───┘  │
│  ┌────▼───┐  │  │  ┌────▼───┐  │  │  ┌────▼───┐  │  │  ┌────▼───┐  │
│  │L2 Cache│  │  │  │L2 Cache│  │  │  │L2 Cache│  │  │  │L2 Cache│  │
│  │(256 KB)│  │  │  │(256 KB)│  │  │  │(256 KB)│  │  │  │(256 KB)│  │
│  └────┬───┘  │  │  └────┬───┘  │  │  └────┬───┘  │  │  └────┬───┘  │
└───────┼──────┘  └───────┼──────┘  └───────┼──────┘  └───────┼──────┘
        │                 │                 │                 │
        └─────────┬───────┴────────┬────────┴────────────────┘
                  │      L3 Cache (compartida, ~8-32 MB)     │
                  └──────────────┬───────────────────────────┘
                                 │
                      ┌──────────▼──────────┐
                      │  Memoria Principal   │
                      │       (RAM)          │
                      └─────────────────────┘
```

Cada core tiene sus cachés L1 y L2 privadas. Cuando un core escribe en una variable, la escritura va a su caché local y no llega inmediatamente a la RAM (ni los otros cores la ven). Cuando otro core lee la misma variable, ve el valor de **su propia caché** (que puede estar desactualizado). La JMM define las reglas de cuándo y cómo los datos se mueven entre cachés y memoria principal.

**Reordenamiento de instrucciones:**

Tanto el compilador JIT como la CPU pueden reordenar instrucciones para optimizar el rendimiento, siempre que el resultado final sea el mismo **desde la perspectiva de un solo hilo**. En entornos multihilo, ese reordenamiento puede causar bugs sutiles:

```java
// Código fuente
int a = 1;          // (1)
flag = true;        // (2)

// El compilador/CPU podría ejecutarlo como:
flag = true;        // (2) primero — reordenamiento
int a = 1;          // (1) después

// Hilo 2 ve flag==true, lee a, pero a todavía no es 1 = BUG
```

### 7.11.2 La relación Happens-Before

La JMM define la relación **happens-before** (hb) como su concepto fundamental. Si una acción A "happens-before" B, entonces B **garantiza** ver los efectos de A (y todo lo que fue visible para A).

#### Regla 1: Program Order Rule

Dentro de un mismo hilo, las acciones ocurren en el orden del programa. Si en el código la línea X está antes que Y, X happens-before Y.

```java
int a = 1;    // A
int b = 2;    // B: A hb B
```

#### Regla 2: Monitor Lock Rule

Un unlock en un monitor happens-before cualquier lock subsiguiente en ese mismo monitor.

```
Hilo A:                      Hilo B:
synchronized(lock) {         synchronized(lock) {  ← ve todo
    x = 42;                      int y = x; // 42
} // unlock                     }
  unlock(A) ─── hb ──→ lock(B)
```

```java
public class MonitorLockRule {
    private static int x = 0;
    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread escritor = new Thread(() -> {
            synchronized (lock) {
                x = 42;
            } // unlock hb el lock del lector
        });

        Thread lector = new Thread(() -> {
            synchronized (lock) {
                System.out.println(x); // garantizado: 42
            }
        });

        escritor.start();
        escritor.join();
        lector.start();
        lector.join();
    }
}
```

#### Regla 3: Volatile Variable Rule

Una escritura en una variable `volatile` happens-before cualquier lectura subsiguiente de esa misma variable. La escritura volatile actúa como barrera de memoria (memory fence): vacía todas las escrituras previas a la memoria principal; la lectura volatile invalida las cachés locales.

```java
volatile boolean listo = false;
int dato = 0;

// Hilo A:
dato = 42;       // (1)
listo = true;    // (2) volatile write — hb → (3)

// Hilo B:
if (listo) {     // (3) volatile read — ve (2)
    // dato == 42 garantizado, porque (1) hb (2) hb (3)
    System.out.println(dato); // 42
}
```

#### Regla 4: Thread Start Rule

`thread.start()` happens-before cualquier acción dentro del hilo iniciado.

```java
x = 42;                    // (1) hb...
new Thread(() -> {
    System.out.println(x); // (2) ...esto. Garantizado: ve 42
}).start();
```

#### Regla 5: Thread Join Rule

Cualquier acción dentro de un hilo happens-before `thread.join()` que retorna exitosamente.

```java
Thread hilo = new Thread(() -> {
    x = 42;                // (1) hb...
});
hilo.start();
hilo.join();               // (2) ...esto
System.out.println(x);     // garantizado: ve 42
```

#### Regla 6: Transitivity

Si A happens-before B, y B happens-before C, entonces A happens-before C.

```java
// Hilo A: escrebe x=42, luego volatileFlag=true
// Hilo B: lee volatileFlag==true, luego synchronized(lock) { lee x }
// Transitivity: x=42 hb volatileFlag=true hb lock hb leer x
// Resultado: Hilo B ve x=42
```

### 7.11.3 `volatile` en profundidad

`volatile` es una de las palabras clave más malentendidas de Java. **No reemplaza a `synchronized`**, no hace atómicas las operaciones compuestas y no protege secciones críticas. Solo garantiza:

| Garantía | Explicación |
|---|---|
| **Visibilidad** | Una escritura en una variable volatile es inmediatamente visible a todos los hilos. |
| **Ordenamiento** | Las escrituras/lecturas volatile no se reordenan con otras operaciones de memoria (barrera). |

**Lo que `volatile` NO garantiza: atomicidad**

```java
volatile int contador = 0;

// INCORRECTO: esto NO es atómico aunque contador sea volatile
contador++;  // leer + sumar + escribir — tres operaciones, no atómicas

// CORRECTO: usar AtomicInteger
AtomicInteger contador = new AtomicInteger(0);
contador.incrementAndGet();  // operación atómica real
```

**Cuándo usar `volatile`:**

1. **Flags de estado** (el caso más común y correcto):

```java
public class Worker implements Runnable {
    private volatile boolean running = true;

    public void run() {
        while (running) {  // siempre ve el valor actual
            // trabajo...
        }
    }

    public void detener() {
        running = false;  // inmediatamente visible para todos los hilos
    }
}
```

Sin `volatile`, el hilo podría optimizar `running` en un registro/caché y nunca ver el cambio (bucle infinito).

2. **Double-checked locking (Singleton thread-safe)**:

```java
public class Singleton {
    private static volatile Singleton instancia;  // ← volatile es CLAVE

    private Singleton() {}

    public static Singleton getInstancia() {
        if (instancia == null) {                // (1) primera comprobación (sin lock)
            synchronized (Singleton.class) {
                if (instancia == null) {        // (2) segunda comprobación (con lock)
                    instancia = new Singleton();
                }
            }
        }
        return instancia;
    }
}
```

**¿Por qué volatile es necesario en double-checked locking?**

La instrucción `instancia = new Singleton()` se descompone en:

```
1. Asignar memoria para el objeto
2. Inicializar el objeto (ejecutar constructor)
3. Asignar la referencia a la variable 'instancia'
```

El compilador puede reordenar (2) y (3):

```
1. Asignar memoria
3. Asignar referencia a 'instancia' (el objeto aún NO está inicializado) ← ¡REORDENADO!
2. Inicializar objeto
```

Si otro hilo ve `instancia != null` entre (3) y (2), usará un objeto parcialmente construido. `volatile` impide este reordenamiento.

**Bug de visibilidad — demostración:**

```java
public class BugVisibilidad {
    private static boolean flag = false;  // sin volatile
    private static int valor = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread escritor = new Thread(() -> {
            valor = 42;
            flag = true;    // podría reordenarse: flag=true primero
        });

        Thread lector = new Thread(() -> {
            while (!flag) {  // podría leer flag=false para siempre
                // busy-wait
            }
            System.out.println("Valor: " + valor);  // podría imprimir 0
        });

        lector.start();
        Thread.sleep(100);
        escritor.start();

        escritor.join();
        lector.join(2000);
        System.out.println("¿Terminó el lector? " + !lector.isAlive());
    }
}
```

Ejecuta este código varias veces en modo servidor (JIT agresivo) y verás que a veces el lector nunca sale del bucle, o sale pero ve `valor=0`. La solución: `private static volatile boolean flag = false;`.

### 7.11.4 `final` y la garantía de inicialización segura

Los campos `final` tienen una garantía especial en la JMM: un objeto correctamente construido (sin escapar `this` del constructor) tiene todos sus campos `final` visibles para cualquier hilo que obtenga una referencia a él, **sin necesidad de sincronización adicional**.

```java
public class Inmutable {
    private final int x;
    private final String nombre;

    public Inmutable(int x, String nombre) {
        this.x = x;
        this.nombre = nombre;
        // Al salir del constructor: "freeze" de campos final
        // (store-store barrier) → flush a memoria principal
    }
}

// Hilo A:
Inmutable obj = new Inmutable(42, "Hola");
mapa.put("clave", obj);  // publicar

// Hilo B:
Inmutable obj = mapa.get("clave");
if (obj != null) {
    System.out.println(obj.getX());  // garantizado: 42 (sin synchronized)
}
```

**Condición clave:** la referencia al objeto no debe "escapar" del constructor antes de que termine:

```java
// PELIGRO: this escapa del constructor
public class Peligroso {
    private final int x;

    public Peligroso() {
        MapaGlobal.registrar(this);  // ← ¡this escapa antes del freeze!
        this.x = 42;
    }
}
// Otro hilo podría ver x=0 porque el objeto se publicó antes del freeze de final
```

**Objetos inmutables y thread-safety:**

Un objeto es inmutable si:
1. Todos sus campos son `final`
2. La clase es `final` o los métodos no pueden ser sobrescritos
3. No expone referencias mutables (defensive copies)
4. `this` no escapa del constructor

Los objetos inmutables son inherentemente thread-safe. Con Java 16+, los `record` son inmutables por definición:

```java
record Punto(int x, int y) {}  // inmutable, thread-safe automáticamente
```

---

## 7.12 Virtual Threads (Project Loom, Java 21+)

### 7.12.1 El problema de los threads de plataforma

Históricamente, en Java cada hilo es un **wrapper** alrededor de un hilo del sistema operativo:

```
Platform thread:  1 Thread Java = 1 OS Thread
                ┌─────────────┐
 Thread Java ──►│ OS Thread   │──► Núcleo CPU
                │ (~1 MB stack)│
                └─────────────┘
```

| Limitación | Impacto |
|---|---|
| **Memoria** | Cada thread de plataforma reserva ~1 MB de stack. 10,000 threads = 10 GB solo en stacks. |
| **Creación costosa** | El SO asigna stack, inicializa estructuras de kernel. ~1 ms por thread. |
| **Cambio de contexto** | El scheduler del SO interrumpe y reanuda threads; cambio de espacio de kernel costoso. |
| **Límite del SO** | La mayoría de SO limitan threads a ~10,000-30,000. |

**Consecuencia:** no es viable "un thread por petición" en alta concurrencia. La alternativa ha sido programación reactiva (CompletableFuture, WebFlux, RxJava), pero es compleja y difícil de depurar.

### 7.12.2 Virtual Threads: la solución

Los **virtual threads** (JEP 444, final en Java 21) desacoplan el hilo Java del hilo del SO:

```
Virtual threads:   Muchos Threads Java → Pocos Carrier Threads (OS Threads)
                 ┌─────────────┐
 Virtual Thread ─┤              │
 Virtual Thread ─┤  Carrier     │──► OS Thread ──► Núcleo CPU
 Virtual Thread ─┤  Thread      │
 Virtual Thread ─┤  (ForkJoin)  │
 Virtual Thread ─┤              │
 Virtual Thread ─┴─────────────┘
```

Cientos de miles (incluso millones) de virtual threads comparten un pequeño pool de **carrier threads** (OS threads). Cuando un virtual thread se bloquea (I/O, sleep, lock), el carrier thread se desacopla y ejecuta otro virtual thread.

| Característica | Platform Thread | Virtual Thread |
|---|---|---|
| Costo de creación | ~1 ms | ~1 µs |
| Memoria por thread | ~1 MB (stack fijo) | ~200-300 bytes (stack en heap) |
| Máximo práctico | ~10,000 | ~1,000,000+ |
| Bloqueo | El OS thread se bloquea | El carrier se desacopla |
| Stack | Fijo, memoria nativa | Dinámico, en el heap |
| Pool | Necesitas thread pools | No necesitas pools |

### 7.12.3 Crear Virtual Threads

```java
// Método 1: Thread.ofVirtual()
Thread vThread = Thread.ofVirtual()
    .name("mi-virtual-thread")
    .start(() -> {
        System.out.println("Virtual: " + Thread.currentThread());
        // Thread[#21,mi-virtual-thread,5,main]
    });
vThread.join();

// Método 2: Thread.startVirtualThread()
Thread vThread2 = Thread.startVirtualThread(() -> {
    System.out.println("Virtual thread rápido");
});

// Método 3: ExecutorService virtual (recomendado)
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> System.out.println("Tarea 1"));
    executor.submit(() -> System.out.println("Tarea 2"));
} // auto-close espera que todas las tareas terminen
```

**Demostración: 100,000 virtual threads vs platform threads**

```java
public class VirtualThreadDemo {
    public static void main(String[] args) throws Exception {
        // ─── VIRTUAL THREADS: 100,000 en segundos ───
        long inicio = System.currentTimeMillis();
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 100_000; i++) {
                final int id = i;
                executor.submit(() -> {
                    try {
                        Thread.sleep(1000);  // simula I/O
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
            }
        }
        long fin = System.currentTimeMillis();
        System.out.println("100,000 virtual threads: " + (fin - inicio) + " ms");
        // Típicamente ~300-800 ms

        // ─── PLATFORM THREADS ───
        // Esto probablemente lanzará OutOfMemoryError o colgará el sistema:
        // ExecutorService pool = Executors.newCachedThreadPool();
        // for (int i = 0; i < 100_000; i++) {
        //     pool.submit(() -> {
        //         try { Thread.sleep(1000); } catch (InterruptedException e) {}
        //     });
        // }
        // ¡Descomentar bajo propio riesgo — consumirá ~100 GB de RAM!
    }
}
```

### 7.12.4 Pinned Threads

Un virtual thread se **pinned** (ancla) al carrier thread cuando ejecuta código que no puede desacoplarse:

**Situaciones que pinnean un virtual thread:**
1. Bloque `synchronized` (el lock intrínseco pinnea al carrier)
2. Métodos nativos (JNI)

```java
// Pinned: el lock intrínseco pinnea al carrier
synchronized (lock) {
    socket.read();  // ¡el carrier se bloquea con el virtual thread!
}
```

**Solución 1: usar ReentrantLock en lugar de synchronized**

```java
// En lugar de:
synchronized (lock) { operacionBloqueante(); }

// Usar:
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    operacionBloqueante();  // el virtual thread se desacopla del carrier
} finally {
    lock.unlock();
}
```

**Detección de pinned threads:**

```bash
java -Djdk.tracePinnedThreads=full MiAplicacion
# Thread[#23,virtual-thread-1,5,main] pinned during 2ms
```

### 7.12.5 Structured Concurrency (Java 21+)

**Structured Concurrency** (JEP 453, preview en Java 21/22) trata las tareas concurrentes como un bloque estructurado: si el bloque termina, todas las subtareas terminan.

**Problema con ExecutorService tradicional:**

```java
ExecutorService pool = Executors.newFixedThreadPool(3);
Future<User> futuroUser = pool.submit(() -> buscarUsuario(id));
Future<Order> futuroOrder = pool.submit(() -> buscarPedido(id));

User user = futuroUser.get();
Order order = futuroOrder.get();
// PROBLEMA: si buscarUsuario falla, buscarPedido sigue ejecutándose
// (hilo huérfano — thread leak)
```

**Solución con StructuredTaskScope:**

```java
import java.util.concurrent.StructuredTaskScope;

Response handleRequest(int userId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<User> userTask = scope.fork(() -> buscarUsuario(userId));
        Subtask<Order> orderTask = scope.fork(() -> buscarPedido(userId));

        scope.join();           // espera a que todas terminen O alguna falle
        scope.throwIfFailed();  // si alguna falló, lanza la excepción

        return new Response(userTask.get(), orderTask.get());
    } // auto-close: cancela todas las subtareas pendientes
}
```

**ShutdownOnSuccess:** se detiene cuando CUALQUIER subtarea tiene éxito:

```java
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
    scope.fork(() -> consultarServicioA());
    scope.fork(() -> consultarServicioB());
    scope.fork(() -> consultarServicioC());

    String resultado = scope.join().result();  // el primero que tenga éxito
}
```

### 7.12.6 Ejemplo real: servidor web con virtual threads

```java
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.Executors;

public class ServidorWebVirtual {
    public static void main(String[] args) throws Exception {
        int puerto = 8080;
        try (ServerSocket server = new ServerSocket(puerto);
             var executor = Executors.newVirtualThreadPerTaskExecutor()) {

            System.out.println("Servidor escuchando en puerto " + puerto);

            while (true) {
                Socket cliente = server.accept();
                executor.submit(() -> manejarCliente(cliente));
            }
        }
    }

    private static void manejarCliente(Socket cliente) {
        try (cliente) {
            var in = cliente.getInputStream();
            var out = cliente.getOutputStream();

            Thread.sleep(200); // simula I/O

            String respuesta = """
                HTTP/1.1 200 OK\r
                Content-Type: text/plain\r
                \r
                Hola desde Virtual Thread!""";

            out.write(respuesta.getBytes());
            out.flush();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 7.12.7 Comparativa de rendimiento

Prueba de servidor HTTP con ~100ms de latencia simulada:

| Configuración | Conexiones | CPU | Memoria | Timeouts |
|---|---|---|---|---|
| FixedThreadPool(200) | 200 | 5% | 400 MB | A partir de 200 |
| FixedThreadPool(1000) | 1,000 | 15% | 1.5 GB | A partir de 1,000 |
| Virtual Threads | 100,000 | 30% | 300 MB | ~0 |
| Virtual Threads | 1,000,000 | 35% | 450 MB | ~0 |

**Cuándo usar Virtual Threads:**
- Servidores web con muchas conexiones concurrentes
- Microservicios que llaman a otros servicios
- Procesamiento de mensajes (colas, eventos)
- Cualquier escenario I/O-bound con alta concurrencia

**Cuándo NO usar Virtual Threads:**
- Tareas CPU-bound puras. Usa ForkJoinPool.
- Código con secciones `synchronized` muy largas
- Bibliotecas que requieran hilos con identidad fija (ThreadLocal excesivo)

---

## 7.13 El paquete `java.util.concurrent.atomic`

### 7.13.1 CAS (Compare-And-Swap)

El fundamento de todas las clases atómicas es la operación **CAS**, implementada por el hardware mediante la instrucción `CMPXCHG` (x86). CAS es atómico y lock-free.

```
CAS(M, E, N):
  1. Dirección de memoria (M)
  2. Valor esperado    (E)
  3. Nuevo valor        (N)

Algoritmo en hardware:
  if (valor en M == E) {
      escribir N en M;
      return true;
  } else {
      return false;  // otro hilo cambió el valor
  }
```

Todo ocurre en **una sola instrucción de CPU**: el hardware garantiza que la lectura, comparación y escritura son indivisibles.

```
┌──────────────────────────────────────────┐
│   Hilo A          Hilo B         Memoria │
│                                          │
│ CAS(M, 5, 10)                   M = 5   │
│   ├─ lee 5                               │
│   ├─ 5==5 ✓                              │
│   └─ M=10                        M = 10  │
│                                          │
│            CAS(M, 5, 42)         M = 10  │
│              ├─ lee 10                   │
│              ├─ 10==5 ✗                  │
│              └─ false            M = 10  │
└──────────────────────────────────────────┘
```

### 7.13.2 `AtomicInteger`, `AtomicLong`, `AtomicReference`

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.AtomicReference;

AtomicInteger contadorAtomico = new AtomicInteger(0);

contadorAtomico.incrementAndGet();       // ++i (atómico)
contadorAtomico.getAndIncrement();       // i++
contadorAtomico.decrementAndGet();       // --i
contadorAtomico.addAndGet(5);            // i += 5
int viejo = contadorAtomico.getAndSet(100);
boolean ok = contadorAtomico.compareAndSet(100, 200);  // CAS manual

// Operaciones lambda atómicas (Java 8+)
contadorAtomico.updateAndGet(x -> Math.max(x, 50));
contadorAtomico.accumulateAndGet(3, (x, delta) -> x * delta);

// AtomicLong: contadores de 64 bits (misma API)
AtomicLong secuencia = new AtomicLong(0);
long siguiente = secuencia.incrementAndGet();  // ID único global

// AtomicReference: referencias atómicas
AtomicReference<String> ref = new AtomicReference<>("inicial");
ref.compareAndSet("inicial", "modificado");
ref.updateAndGet(String::toUpperCase);
```

**Ejemplo: contador de visitas thread-safe SIN synchronized:**

```java
public class ContadorVisitas {
    private static final AtomicLong visitas = new AtomicLong(0);

    public static void incrementar() {
        visitas.incrementAndGet();  // thread-safe sin locks
    }

    public static long getVisitas() {
        return visitas.get();
    }

    public static void main(String[] args) throws InterruptedException {
        int hilos = 100;
        int incrementosPorHilo = 10_000;
        Thread[] threads = new Thread[hilos];

        for (int i = 0; i < hilos; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < incrementosPorHilo; j++) {
                    incrementar();
                }
            });
            threads[i].start();
        }

        for (Thread t : threads) t.join();

        System.out.println("Visitas totales: " + getVisitas());
        System.out.println("Esperado: " + (hilos * incrementosPorHilo));
        // Resultado exacto garantizado: 1,000,000
    }
}
```

### 7.13.3 `LongAdder` y `DoubleAdder`

Bajo alta contención (muchos hilos actualizando el mismo contador), `AtomicLong` sufre de **contención CAS**: cuando muchos cores intentan CAS simultáneamente, la mayoría falla y tiene que reintentar.

```java
import java.util.concurrent.atomic.LongAdder;

LongAdder contadorAdder = new LongAdder();

contadorAdder.increment();   // más rápido que AtomicLong bajo contención
contadorAdder.add(10);
long total = contadorAdder.sum();   // suma todas las celdas
contadorAdder.reset();              // pone a cero
```

**Funcionamiento interno de LongAdder:**

```
┌─────────────────────────────────┐
│          LongAdder              │
│  ┌───────┐ ┌───────┐ ┌───────┐  │
│  │Cell 0 │ │Cell 1 │ │Cell 2 │  │
│  │  42   │ │  37   │ │  51   │  │
│  └───────┘ └───────┘ └───────┘  │
│    ↑          ↑          ↑       │
│  Hilo 1     Hilo 2     Hilo 3   │
│                                 │
│  sum() = 42 + 37 + 51 + base    │
└─────────────────────────────────┘

// El array de cells crece bajo contención:
//  - Pocos hilos: solo actualiza variable base
//  - Muchos hilos: distribuye entre cells (striping)
//  - sum() combina todas las celdas (instantáneo, no atómico)
```

**Comparativa de rendimiento:**

```java
public class BenchmarkAtomics {
    static final int HILOS = 64;
    static final int ITERACIONES = 1_000_000;

    public static void main(String[] args) throws Exception {
        // AtomicLong
        AtomicLong atomicLong = new AtomicLong(0);
        long t1 = System.currentTimeMillis();
        Thread[] threads = new Thread[HILOS];
        for (int i = 0; i < HILOS; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITERACIONES; j++)
                    atomicLong.incrementAndGet();
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        long t2 = System.currentTimeMillis();
        System.out.println("AtomicLong: " + (t2 - t1) + " ms");

        // LongAdder (mucho más rápido bajo contención)
        LongAdder longAdder = new LongAdder();
        t1 = System.currentTimeMillis();
        for (int i = 0; i < HILOS; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITERACIONES; j++)
                    longAdder.increment();
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        t2 = System.currentTimeMillis();
        System.out.println("LongAdder:  " + (t2 - t1) + " ms");

        // Resultado típico:
        //   AtomicLong: ~5000 ms
        //   LongAdder:  ~800 ms   (6x más rápido)
    }
}
```

### 7.13.4 El problema ABA y `AtomicStampedReference`

El **problema ABA** es una limitación de CAS: si entre la lectura y el CAS, el valor cambió A → B → A, el CAS ve A y asume que nada cambió.

```
Tiempo │ Hilo A                 │ Hilo B               │ Memoria
───────┼────────────────────────┼──────────────────────┼────────
  t0   │ lee valor: "A"         │                      │ "A"
  t1   │                        │ CAS: "A" → "B"       │ "B"
  t2   │                        │ CAS: "B" → "A"       │ "A"
  t3   │ CAS: esperaba "A",     │                      │
       │   ve "A", OK pero      │                      │ "C"
       │   ¡hubo cambios!       │                      │
```

**Ejemplo crítico: pila lock-free**

```java
class Nodo { String valor; Nodo siguiente; }
AtomicReference<Nodo> cabeza = new AtomicReference<>();

// Problema ABA en pop:
// 1. Leo cabeza → Nodo A
// 2. Otro hilo hace pop de A, pop de B, push de A
// 3. CAS piensa que la cabeza sigue igual, pero la estructura es diferente
```

**Solución: `AtomicStampedReference`** añade un "sello" (stamp) — número de versión que se incrementa con cada modificación:

```java
import java.util.concurrent.atomic.AtomicStampedReference;

AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
int[] stamp = new int[1];
String valor = ref.get(stamp);
boolean ok = ref.compareAndSet(valor, "C", stamp[0], stamp[0] + 1);
```

**¿Cuándo preocuparse por ABA?** Pilas/colas lock-free, gestión de memoria lock-free. En la mayoría de aplicaciones normales (contadores, flags), ABA no es un problema.

### 7.13.5 Otras clases atómicas

| Clase | Propósito |
|---|---|
| `AtomicBoolean` | Flag booleano atómico |
| `AtomicIntegerArray` | Array de int con operaciones atómicas por índice |
| `AtomicLongArray` | Array de long con operaciones atómicas por índice |
| `AtomicReferenceArray` | Array de referencias con operaciones atómicas |
| `AtomicIntegerFieldUpdater` | Actualización atómica de campos volatile de una clase |
| `AtomicMarkableReference` | Referencia + flag booleano (útil para "borrado lógico") |

```java
// AtomicIntegerArray
AtomicIntegerArray arrayAtomico = new AtomicIntegerArray(10);
arrayAtomico.set(0, 42);
arrayAtomico.incrementAndGet(0);       // 43
arrayAtomico.compareAndSet(0, 43, 100); // true

// AtomicIntegerFieldUpdater (reflection-based, sin crear objetos extra)
class MiClase {
    volatile int contador;  // debe ser volatile
}
AtomicIntegerFieldUpdater<MiClase> updater =
    AtomicIntegerFieldUpdater.newUpdater(MiClase.class, "contador");
```

---

## 7.14 Patrones de concurrencia avanzados

### 7.14.1 Producer-Consumer con BlockingQueue

El clásico patrón productor-consumidor es la base de muchos sistemas de procesamiento.

**Sistema de procesamiento de pedidos:**

```java
public class SistemaPedidos {
    static class Pedido {
        final long id;
        final String cliente;
        final double monto;

        Pedido(long id, String cliente, double monto) {
            this.id = id;
            this.cliente = cliente;
            this.monto = monto;
        }
    }

    static final Pedido POISON_PILL = new Pedido(-1, "", 0.0);

    static class GeneradorPedidos implements Runnable {
        private final BlockingQueue<Pedido> cola;
        private final int numPedidos;

        GeneradorPedidos(BlockingQueue<Pedido> cola, int numPedidos) {
            this.cola = cola;
            this.numPedidos = numPedidos;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < numPedidos; i++) {
                    Pedido p = new Pedido(i, "Cliente-" + i, Math.random() * 1000);
                    cola.put(p);
                    System.out.println("Nuevo pedido: " + p.id);
                    Thread.sleep((long)(Math.random() * 100));
                }
                cola.put(POISON_PILL);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    static class ProcesadorPedidos implements Runnable {
        private final BlockingQueue<Pedido> cola;
        private final String nombre;

        ProcesadorPedidos(BlockingQueue<Pedido> cola, String nombre) {
            this.cola = cola;
            this.nombre = nombre;
        }

        @Override
        public void run() {
            try {
                while (true) {
                    Pedido p = cola.take();
                    if (p == POISON_PILL) {
                        cola.put(POISON_PILL); // re-encolar para otros consumidores
                        break;
                    }
                    System.out.println(nombre + " procesa pedido #" + p.id
                        + " por $" + String.format("%.2f", p.monto));
                    Thread.sleep((long)(Math.random() * 500));
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Pedido> cola = new LinkedBlockingQueue<>(20);

        Thread productor = new Thread(new GeneradorPedidos(cola, 50));
        Thread c1 = new Thread(new ProcesadorPedidos(cola, "Consumidor-1"));
        Thread c2 = new Thread(new ProcesadorPedidos(cola, "Consumidor-2"));
        Thread c3 = new Thread(new ProcesadorPedidos(cola, "Consumidor-3"));

        productor.start(); c1.start(); c2.start(); c3.start();
        productor.join();  c1.join();  c2.join();  c3.join();

        System.out.println("Todos los pedidos procesados.");
    }
}
```

### 7.14.2 Read-Write Lock: cache con estadísticas

```java
public class CacheConfiguracion {
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Map<String, String> datos = new HashMap<>();
    private final AtomicLong lecturas = new AtomicLong(0);
    private final AtomicLong escrituras = new AtomicLong(0);
    private final AtomicLong fallosCache = new AtomicLong(0);

    public String get(String clave) {
        lock.readLock().lock();
        try {
            lecturas.incrementAndGet();
            String valor = datos.get(clave);
            if (valor == null) fallosCache.incrementAndGet();
            return valor;
        } finally {
            lock.readLock().unlock();
        }
    }

    public void put(String clave, String valor) {
        lock.writeLock().lock();
        try {
            escrituras.incrementAndGet();
            datos.put(clave, valor);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public void imprimirEstadisticas() {
        lock.readLock().lock();
        try {
            System.out.printf("Lecturas: %d | Escrituras: %d | "
                + "Tamaño: %d | Fallos: %d%n",
                lecturas.get(), escrituras.get(), datos.size(), fallosCache.get());
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

### 7.14.3 MapReduce local con ForkJoinPool

`ForkJoinPool` está diseñado para tareas que pueden dividirse recursivamente. Usa **work-stealing**: si un worker termina, roba trabajo de la cola de otros workers.

```
Arquitectura ForkJoinPool (Work Stealing):

  Worker 0: [Task1, Task2, ...]──┐   (deque propio, LIFO)
  Worker 1: [Task3, ...]     <──┘ roba de la otra cola (FIFO)
  Worker 2: [Task4, Task5, ...]
  Worker 3: [Task6, ...]
```

**Ejemplo: suma de array gigante con divide y vencerás:**

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class SumaArrayForkJoin extends RecursiveTask<Long> {
    private static final int UMBRAL = 10_000;
    private final long[] array;
    private final int inicio, fin;

    public SumaArrayForkJoin(long[] array, int inicio, int fin) {
        this.array = array;
        this.inicio = inicio;
        this.fin = fin;
    }

    @Override
    protected Long compute() {
        int longitud = fin - inicio;

        if (longitud <= UMBRAL) {
            // Caso base: calcular secuencialmente
            long suma = 0;
            for (int i = inicio; i < fin; i++) {
                suma += array[i];
            }
            return suma;
        }

        // Dividir el problema
        int medio = inicio + longitud / 2;
        SumaArrayForkJoin izquierda = new SumaArrayForkJoin(array, inicio, medio);
        SumaArrayForkJoin derecha = new SumaArrayForkJoin(array, medio, fin);

        // Fork: ejecutar subtareas en paralelo
        izquierda.fork();
        long resultadoDerecha = derecha.compute();
        long resultadoIzquierda = izquierda.join();

        return resultadoIzquierda + resultadoDerecha;
    }

    public static void main(String[] args) {
        int N = 100_000_000;
        long[] array = new long[N];
        for (int i = 0; i < N; i++) array[i] = 1;

        ForkJoinPool pool = new ForkJoinPool();
        SumaArrayForkJoin tarea = new SumaArrayForkJoin(array, 0, N);

        long inicio = System.currentTimeMillis();
        long suma = pool.invoke(tarea);
        long fin = System.currentTimeMillis();

        System.out.println("Suma: " + suma + " (esperado: " + N + ")");
        System.out.println("Tiempo ForkJoin: " + (fin - inicio) + " ms");
    }
}
```

**`RecursiveAction` para tareas sin retorno (QuickSort paralelo):**

```java
import java.util.concurrent.RecursiveAction;

public class QuickSortForkJoin extends RecursiveAction {
    private final int[] array;
    private final int low, high;

    public QuickSortForkJoin(int[] array, int low, int high) {
        this.array = array;
        this.low = low;
        this.high = high;
    }

    @Override
    protected void compute() {
        if (low < high) {
            int pivot = partition(array, low, high);
            invokeAll(
                new QuickSortForkJoin(array, low, pivot - 1),
                new QuickSortForkJoin(array, pivot + 1, high)
            );
        }
    }

    private int partition(int[] arr, int low, int high) {
        int pivot = arr[high], i = low - 1;
        for (int j = low; j < high; j++) {
            if (arr[j] <= pivot) {
                i++;
                int tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
            }
        }
        int tmp = arr[i + 1]; arr[i + 1] = arr[high]; arr[high] = tmp;
        return i + 1;
    }
}
```

### 7.14.4 Estrategias de diseño de objetos thread-safe

| Estrategia | Descripción | Ejemplo |
|---|---|---|
| **Confinamiento** | No compartir datos. Cada hilo tiene sus objetos. | ThreadLocal, variables locales |
| **Inmutabilidad** | Objetos inmutables no necesitan sincronización. | `record Punto(int x, int y)` |
| **Sincronización** | Proteger datos compartidos con locks. | `synchronized`, `Lock` |
| **Clases concurrentes** | Usar estructuras del JDK en vez de construir propias. | `ConcurrentHashMap`, `BlockingQueue` |
| **Confinamiento al stack** | Variables locales en el stack son automáticamente thread-safe. | `int local = 0;` |

---

## 7.15 Testing de código concurrente

### 7.15.1 Por qué es difícil testear concurrencia

| Desafío | Descripción |
|---|---|
| **No determinismo** | El scheduler del SO decide el orden. Un bug puede aparecer 1 de cada 10,000 ejecuciones. |
| **Dependencia del hardware** | Un bug puede manifestarse solo en CPUs con 8+ cores, o solo en ARM. |
| **Efecto observador** | Añadir logs o breakpoints cambia el timing y puede ocultar condiciones de carrera. |
| **Falsos positivos** | `Thread.sleep(1000)` hace el test lento pero no garantiza orden, solo retrasa. |

### 7.15.2 CountDownLatch y CyclicBarrier en tests

**Forzar entrelazado específico para testear una race condition:**

```java
@Test
public void testCondicionCarreraConLatch() throws Exception {
    CountDownLatch inicio = new CountDownLatch(1);
    CountDownLatch fin = new CountDownLatch(2);
    AtomicInteger contador = new AtomicInteger(0);

    // Hilo 1: espera señal, incrementa, señala fin
    new Thread(() -> {
        try { inicio.await(); } catch (InterruptedException e) {}
        contador.incrementAndGet();
        fin.countDown();
    }).start();

    // Hilo 2: espera señal, incrementa, señala fin
    new Thread(() -> {
        try { inicio.await(); } catch (InterruptedException e) {}
        contador.incrementAndGet();
        fin.countDown();
    }).start();

    inicio.countDown();  // ¡disparo simultáneo!
    fin.await(2, TimeUnit.SECONDS);

    assertEquals(2, contador.get());  // AtomicInteger garantiza 2
}
```

### 7.15.3 Herramientas especializadas

**jcstress (Java Concurrency Stress Tests):** Framework oficial de OpenJDK para micro-benchmarks de concurrencia.

```java
@JCStressTest
@Outcome(id = "1", expect = Expect.ACCEPTABLE, desc = "Correcto")
@Outcome(id = "0", expect = Expect.FORBIDDEN,  desc = "Se perdió actualización")
@State
public class AtomicityTest {
    AtomicInteger ai = new AtomicInteger(0);

    @Actor
    public void actor1() { ai.incrementAndGet(); }

    @Actor
    public void actor2() { ai.incrementAndGet(); }

    @Arbiter
    public void arbiter(IntResult1 r) { r.r1 = ai.get(); }
}
```

**vmlens:** Ejecuta tests unitarios con múltiples schedules para detectar condiciones de carrera.

**ThreadSanitizer (TSan):** Sanitizer de Clang/GCC para detectar data races en código nativo (JNI).

### 7.15.4 Estrategia de tests para concurrencia

1. **Testear secuencialmente primero.** Si no funciona en single-thread, no funcionará en multi.
2. **Ejecutar muchas veces:** usar `@RepeatedTest(1000)` o un bucle.
3. **Variar el número de hilos:** testear con 1, 2, 4, 8, 16, 32 hilos.
4. **Usar timeouts:** un deadlock se manifiesta como test que nunca termina.
5. **Stress test con múltiples iteraciones:**

```java
@Test
public void stressTestConcurrencia() throws Exception {
    for (int iteracion = 0; iteracion < 10_000; iteracion++) {
        AtomicInteger contador = new AtomicInteger(0);
        int hilos = 4;
        CountDownLatch latch = new CountDownLatch(hilos);

        for (int i = 0; i < hilos; i++) {
            new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    contador.incrementAndGet();
                }
                latch.countDown();
            }).start();
        }

        latch.await(5, TimeUnit.SECONDS);
        assertEquals(4000, contador.get(),
            "Fallo en iteración " + iteracion);
    }
}
```

---

## 7.16 Depuración de problemas de concurrencia

### 7.16.1 Thread dumps

Un **thread dump** es una instantánea del estado de todos los hilos vivos en la JVM.

**Cómo obtener un thread dump:**

```bash
# Método 1: jstack
jstack <PID>

# Método 2: jcmd
jcmd <PID> Thread.print

# Método 3: Señal del SO (Unix/Linux/macOS)
kill -3 <PID>   # envía QUIT; el dump va a stdout

# Método 4: Desde dentro de la aplicación
Thread.getAllStackTraces().forEach((thread, stack) -> {
    System.out.println(thread);
    for (StackTraceElement frame : stack) {
        System.out.println("\t" + frame);
    }
});

# Método 5: Encontrar el PID primero
jps -l
```

### 7.16.2 Cómo leer un thread dump

Un thread dump típico contiene entradas como:

```
"pool-1-thread-3" #13 prio=5 os_prio=0 tid=0x00007f9e8800a000 nid=0x5a3c
   waiting on condition [0x00007f9e6c1f8000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000006c2e85540>
          (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
        at com.miempresa.BufferLimitado.tomar(BufferLimitado.java:47)
```

**Anatomía de una entrada:**

| Campo | Significado |
|---|---|
| `"pool-1-thread-3"` | Nombre del hilo |
| `#13` | Número interno JVM |
| `prio=5` | Prioridad Java (1-10) |
| `tid=0x...` | Thread ID interno JVM |
| `nid=0x5a3c` | Native thread ID (OS thread) |
| `TIMED_WAITING` | Estado actual |
| `- parking to wait for` | Lock/condición que espera |

**Estados clave:**

| Estado | Significado | Acción |
|---|---|---|
| `BLOCKED` | Esperando `synchronized` que otro tiene | Buscar qué hilo tiene ese lock |
| `WAITING` | Espera indefinida (wait, join, park) | ¿Falta un `notify()`? |
| `TIMED_WAITING` | Esperando con timeout | ¿Timeout muy largo? |
| `RUNNABLE` | Ejecutándose o esperando CPU | Si muchos RUNNABLE, ¿saturación? |

**Deadlock en thread dump:**

Al final del thread dump, `jstack` incluye una sección de deadlock:

```
Found one Java-level deadlock:
=============================
"Hilo-2":
  waiting to lock monitor 0x... (object LockB)
  which is held by "Hilo-1"
"Hilo-1":
  waiting to lock monitor 0x... (object LockA)
  which is held by "Hilo-2"
```

**¿Qué buscar en múltiples thread dumps?**

- **Muchos hilos BLOCKED** sobre el mismo lock → contención, cuello de botella.
- **Muchos hilos WAITING** en el mismo pool → pool sin trabajo, o tareas bloqueadas.
- **Un solo hilo RUNNABLE**, los demás esperando → tarea secuencial.
- **Hilos que nunca cambian de estado** entre dumps → posible deadlock o livelock.
- **Cantidad creciente de hilos** entre dumps consecutivos → thread leak.

### 7.16.3 Java Flight Recorder (JFR) y JDK Mission Control (JMC)

JFR es un perfilador de baja sobrecarga (< 2%) integrado en el JDK.

```bash
# Al arrancar la JVM
java -XX:StartFlightRecording=duration=60s,filename=miperfil.jfr MiApp

# Attach a proceso en ejecución
jcmd <PID> JFR.start duration=60s filename=perfil.jfr
```

**Eventos de concurrencia que captura JFR:**

| Evento | Qué mide |
|---|---|
| `Java Monitor Enter` | Tiempo esperando `synchronized` |
| `Java Monitor Wait` | Tiempo en `wait()` |
| `Thread Park` | Tiempo en `LockSupport.park()` |
| `Thread Sleep` | Duración de `Thread.sleep()` |
| `Java Thread Start/End` | Creación y terminación de hilos |

### 7.16.4 Diagnóstico de aplicaciones lentas por contención

**Síntomas de contención de locks:**
- El throughput no escala con el número de cores.
- Añadir más hilos empeora el rendimiento.
- Hilos mayoritariamente en BLOCKED en los thread dumps.
- Tiempo de CPU alto pero poco trabajo útil.

**Diagnóstico paso a paso:**

```bash
# 1. Ver uso de CPU por hilo
top -H -p <PID>

# 2. Convertir hilo top a hexadecimal y buscar en thread dump
thread_id_hex=$(printf "%x" <HILO_ID_TOP>)
jstack <PID> | grep -A 20 "nid=0x$thread_id_hex"

# 3. Múltiples thread dumps (cada 5 segundos)
for i in 1 2 3; do
    jstack <PID> > "threaddump-$i.txt"
    sleep 5
done

# 4. Buscar el lock más contencioso
grep "waiting to lock" threaddump-*.txt | sort | uniq -c | sort -rn
# El lock con más ocurrencias = cuello de botella principal

# 5. Para locks de java.util.concurrent
jstack <PID> | grep -E "parking to wait for|waiting to lock"
```

### 7.16.5 VisualVM

VisualVM (incluido en JDK hasta Java 8, descargable para versiones posteriores) proporciona interfaz gráfica para monitorear hilos:

- Pestaña Threads: ver hilos, estado, timeline de actividad
- Thread dump con un clic
- Timeline visual de estados de hilos (patrones: ráfagas de BLOCKED, hilos siempre WAITING)
- Heap dump para ver objetos relacionados con threads y locks

---

## Resumen del capítulo

- Un **hilo** es una unidad de ejecución ligera que comparte heap con otros hilos del mismo proceso pero posee stack propio.
- Crear hilos: implementar `Runnable` (preferido) o extender `Thread`. Invocar `start()`, no `run()`.
- El ciclo de vida recorre estados `NEW → RUNNABLE → (BLOCKED|WAITING|TIMED_WAITING) → TERMINATED`.
- `synchronized` protege secciones críticas. El lock intrínseco es reentrante. Se usa a nivel de método o bloque.
- **Deadlock** ocurre cuando dos hilos esperan locks del otro. **Livelock**: hilos no bloqueados pero sin progreso. **Starvation**: hilos que nunca obtienen el recurso.
- **ThreadLocal** aísla datos por hilo (contexto de usuario en web apps). Siempre llamar `remove()` para evitar memory leaks.
- La API `Lock` (`ReentrantLock`, `ReadWriteLock`, `Condition`) ofrece más control: timeout, interrupción, fairness y múltiples condiciones.
- El **JMM** define las reglas de visibilidad entre hilos. La relación **happens-before** es el concepto fundamental (6 reglas).
- `volatile` garantiza visibilidad y orden, pero NO atomicidad. Clave en flags y double-checked locking.
- Los campos `final` tienen garantía de inicialización segura (freeze al salir del constructor).
- `wait`/`notify`/`notifyAll` son mecanismos de bajo nivel; usar `while` (no `if`) y `notifyAll()`.
- `ExecutorService` gestiona pools de hilos. `shutdown()` + `awaitTermination()` al finalizar.
- `Callable<V>` permite retornar valores. `Future<V>` representa un resultado pendiente.
- `CompletableFuture` permite encadenar, combinar y manejar errores en operaciones asíncronas.
- **Virtual threads** (Java 21+): millones de hilos ligeros que comparten pocos carrier threads. Ideales para I/O-bound masivo.
- **Structured Concurrency** (Java 21+): `StructuredTaskScope` — cancelación automática, manejo de errores, sin fugas de hilos.
- **Pinned threads**: un virtual thread se ancla al carrier en `synchronized` o JNI. Usar `ReentrantLock` como alternativa.
- Las clases `java.util.concurrent` evitan sincronización manual: `ConcurrentHashMap`, `BlockingQueue`, `CountDownLatch`, `CyclicBarrier`, `Semaphore`.
- **CAS** (Compare-And-Swap) es la base de las clases atómicas: operación hardware indivisible sin locks.
- `AtomicInteger`/`AtomicLong` para operaciones atómicas sin `synchronized`. `LongAdder` es más rápido bajo alta contención.
- El **problema ABA** ocurre cuando un valor cambia A→B→A durante CAS. `AtomicStampedReference` lo resuelve con número de versión.
- **Patrones**: Producer-Consumer con `BlockingQueue`, Read-Write Lock para muchas lecturas, MapReduce con `ForkJoinPool`.
- **ForkJoinPool** usa work-stealing: workers ociosos roban trabajo de otras colas. Ideal para divide y vencerás.
- **Testing:** usar CountDownLatch para coordinar hilos en tests, ejecutar miles de iteraciones, variar número de hilos, usar jcstress.
- **Depuración:** `jstack`/`jcmd` para thread dumps, JFR para profiling de baja sobrecarga, VisualVM para monitoreo visual.

